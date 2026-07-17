# Demo13 — Knowledge Bases 构建最小 RAG Golden Path

## 实验简介

用 Amazon Bedrock Knowledge Bases 构建一个最小 RAG：S3 放入企业政策文档，OpenSearch Serverless 存储向量，Knowledge Base 同步数据，再用 `retrieve-and-generate` 回答问题并返回 citation。

**实验目标：**
- 理解 Knowledge Base、Data Source、Embedding Model、Vector Store 的关系
- 创建 Bedrock service role
- 创建 OpenSearch Serverless vector store
- 创建 Knowledge Base 和 S3 data source
- 同步数据并执行 `retrieve-and-generate`

**预计 AI 执行时长：** 35-60 分钟

---

## 前提条件

- 已完成 Demo01
- 已设置 `BEDROCK_TEXT_MODEL_ID` 和 `BEDROCK_EMBED_MODEL_ID`
- 权限包含 Bedrock Agent、S3、IAM、OpenSearch Serverless

```bash
export AWS_REGION=us-east-1
export AWS_PARTITION=aws
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BEDROCK_BUCKET=${BEDROCK_BUCKET:?set this from Demo01}
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
export BEDROCK_EMBED_MODEL_ID=${BEDROCK_EMBED_MODEL_ID:?set this from Demo01}
export KB_NAME=bedrock-quickstart-kb
export KB_ROLE_NAME=BedrockQuickStartKnowledgeBaseRole
export AOSS_COLLECTION_NAME=bedrock-quickstart-kb
export AOSS_INDEX_NAME=bedrock-quickstart-index
mkdir -p tmp
```

---

## 步骤

### 1. 上传企业政策样例文档

```bash
cat > tmp/policy-faq.txt <<'EOF'
企业政策助手知识库

主题：客户合同与外部工具
政策：客户合同、报价单、源代码、访问凭证和客户个人信息不得上传到未授权外部 AI 工具或翻译工具。
处理建议：如需翻译或摘要，应使用公司批准的内部工具，并确保开启审计日志。

主题：默认云区域
政策：Bedrock QuickStart 默认使用 AWS Global us-east-1 区域。

主题：持续费用
政策：OpenSearch Serverless collection、CloudWatch Logs、S3 对象、Provisioned Throughput 和高频模型调用可能产生持续费用。
EOF

aws s3 cp tmp/policy-faq.txt s3://${BEDROCK_BUCKET}/kb/policy-faq.txt
```

### 2. 创建 Knowledge Base service role

```bash
cat > tmp/kb-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "bedrock.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ${KB_ROLE_NAME} \
  --assume-role-policy-document file://tmp/kb-trust.json

cat > tmp/kb-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:${AWS_PARTITION}:s3:::${BEDROCK_BUCKET}",
        "arn:${AWS_PARTITION}:s3:::${BEDROCK_BUCKET}/kb/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel"],
      "Resource": "arn:${AWS_PARTITION}:bedrock:${AWS_REGION}::foundation-model/${BEDROCK_EMBED_MODEL_ID}"
    },
    {
      "Effect": "Allow",
      "Action": ["aoss:APIAccessAll"],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name ${KB_ROLE_NAME} \
  --policy-name ${KB_ROLE_NAME}-policy \
  --policy-document file://tmp/kb-policy.json

export KB_ROLE_ARN=arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:role/${KB_ROLE_NAME}
```

### 3. 创建 OpenSearch Serverless collection

```bash
cat > tmp/aoss-encryption-policy.json <<EOF
{
  "Rules": [
    {
      "ResourceType": "collection",
      "Resource": ["collection/${AOSS_COLLECTION_NAME}"]
    }
  ],
  "AWSOwnedKey": true
}
EOF

aws opensearchserverless create-security-policy \
  --name ${AOSS_COLLECTION_NAME}-enc \
  --type encryption \
  --policy file://tmp/aoss-encryption-policy.json \
  --region ${AWS_REGION}

cat > tmp/aoss-network-policy.json <<EOF
[
  {
    "Rules": [
      {
        "ResourceType": "collection",
        "Resource": ["collection/${AOSS_COLLECTION_NAME}"]
      }
    ],
    "AllowFromPublic": true
  }
]
EOF

aws opensearchserverless create-security-policy \
  --name ${AOSS_COLLECTION_NAME}-net \
  --type network \
  --policy file://tmp/aoss-network-policy.json \
  --region ${AWS_REGION}

aws opensearchserverless create-collection \
  --name ${AOSS_COLLECTION_NAME} \
  --type VECTORSEARCH \
  --region ${AWS_REGION} \
  > tmp/aoss-collection.json

export AOSS_COLLECTION_ID=$(jq -r '.createCollectionDetail.id' tmp/aoss-collection.json)
```

轮询 collection 到 `ACTIVE`：

```bash
aws opensearchserverless batch-get-collection \
  --ids ${AOSS_COLLECTION_ID} \
  --region ${AWS_REGION}
```

### 4. 创建 AOSS access policy

```bash
cat > tmp/aoss-access-policy.json <<EOF
[
  {
    "Rules": [
      {
        "ResourceType": "collection",
        "Resource": ["collection/${AOSS_COLLECTION_NAME}"],
        "Permission": ["aoss:*"]
      },
      {
        "ResourceType": "index",
        "Resource": ["index/${AOSS_COLLECTION_NAME}/*"],
        "Permission": ["aoss:*"]
      }
    ],
    "Principal": ["${KB_ROLE_ARN}"]
  }
]
EOF

aws opensearchserverless create-access-policy \
  --name ${AOSS_COLLECTION_NAME}-access \
  --type data \
  --policy file://tmp/aoss-access-policy.json \
  --region ${AWS_REGION}
```

### 5. 创建向量 index

OpenSearch Serverless index 创建可通过 OpenSearch API 完成。执行时使用当前 collection endpoint 和支持的认证方式创建一个包含 vector field、text field、metadata field 的 index。字段约定：

| 字段 | 用途 |
|------|------|
| `bedrock-quickstart-vector` | embedding vector |
| `AMAZON_BEDROCK_TEXT_CHUNK` | 文本 chunk |
| `AMAZON_BEDROCK_METADATA` | metadata |

> 如果当前 AWS CLI / 环境已提供自动创建 index 的 Knowledge Bases 配置，可跳过手工建 index，直接在下一步使用对应 storageConfiguration。

### 6. 创建 Knowledge Base

按当前 AWS CLI schema 创建 Knowledge Base。核心参数必须包含：
- `roleArn=${KB_ROLE_ARN}`
- embedding model ARN
- OpenSearch Serverless collection ARN
- vector index name
- text/vector/metadata field mapping

命令模板：

```bash
aws bedrock-agent create-knowledge-base \
  --name ${KB_NAME} \
  --role-arn ${KB_ROLE_ARN} \
  --knowledge-base-configuration file://tmp/kb-configuration.json \
  --storage-configuration file://tmp/kb-storage-configuration.json \
  --region ${AWS_REGION} \
  > tmp/kb-create.json

export KNOWLEDGE_BASE_ID=$(jq -r '.knowledgeBase.knowledgeBaseId' tmp/kb-create.json)
```

### 7. 创建 S3 data source

```bash
jq -n --arg bucketArn "arn:${AWS_PARTITION}:s3:::${BEDROCK_BUCKET}" '{
  type: "S3",
  s3Configuration: {
    bucketArn: $bucketArn,
    inclusionPrefixes: ["kb/"]
  }
}' > tmp/kb-data-source-config.json

aws bedrock-agent create-data-source \
  --knowledge-base-id ${KNOWLEDGE_BASE_ID} \
  --name ${KB_NAME}-s3 \
  --data-source-configuration file://tmp/kb-data-source-config.json \
  --region ${AWS_REGION} \
  > tmp/kb-data-source.json

export DATA_SOURCE_ID=$(jq -r '.dataSource.dataSourceId' tmp/kb-data-source.json)
```

### 8. 启动 ingestion job

```bash
aws bedrock-agent start-ingestion-job \
  --knowledge-base-id ${KNOWLEDGE_BASE_ID} \
  --data-source-id ${DATA_SOURCE_ID} \
  --region ${AWS_REGION} \
  > tmp/ingestion-job.json

export INGESTION_JOB_ID=$(jq -r '.ingestionJob.ingestionJobId' tmp/ingestion-job.json)
```

轮询到 `COMPLETE`：

```bash
aws bedrock-agent get-ingestion-job \
  --knowledge-base-id ${KNOWLEDGE_BASE_ID} \
  --data-source-id ${DATA_SOURCE_ID} \
  --ingestion-job-id ${INGESTION_JOB_ID} \
  --region ${AWS_REGION}
```

### 9. Retrieve and Generate

```bash
jq -n --arg kb "${KNOWLEDGE_BASE_ID}" --arg model "arn:${AWS_PARTITION}:bedrock:${AWS_REGION}::foundation-model/${BEDROCK_TEXT_MODEL_ID}" '{
  input: {text: "员工能把客户合同发到外部翻译工具吗？哪些资源可能产生持续费用？"},
  retrieveAndGenerateConfiguration: {
    type: "KNOWLEDGE_BASE",
    knowledgeBaseConfiguration: {
      knowledgeBaseId: $kb,
      modelArn: $model
    }
  }
}' > tmp/retrieve-generate.json

aws bedrock-agent-runtime retrieve-and-generate \
  --cli-input-json file://tmp/retrieve-generate.json \
  --region ${AWS_REGION} \
  > tmp/retrieve-generate-output.json

jq -r '.output.text' tmp/retrieve-generate-output.json
jq '.citations' tmp/retrieve-generate-output.json
```

---

## 验收标准

- S3 样例文档上传成功
- OpenSearch Serverless collection 状态为 `ACTIVE`
- Knowledge Base 状态为 `ACTIVE`
- Ingestion Job 状态为 `COMPLETE`
- 回答能基于政策文档拒绝外部工具上传
- citations 中能看到 S3 文档来源

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|----------|----------|
| 1 | `aws opensearchserverless batch-get-collection --ids ${AOSS_COLLECTION_ID} --region ${AWS_REGION} --query 'collectionDetails[0].status' --output text` | `ACTIVE` |
| 2 | `aws bedrock-agent get-knowledge-base --knowledge-base-id ${KNOWLEDGE_BASE_ID} --region ${AWS_REGION} --query 'knowledgeBase.status' --output text` | `ACTIVE` |
| 3 | `aws bedrock-agent get-ingestion-job --knowledge-base-id ${KNOWLEDGE_BASE_ID} --data-source-id ${DATA_SOURCE_ID} --ingestion-job-id ${INGESTION_JOB_ID} --region ${AWS_REGION} --query 'ingestionJob.status' --output text` | `COMPLETE` |
| 4 | `jq '.citations \| length' tmp/retrieve-generate-output.json` | 大于 `0` |

## 常见失败与处理

| 现象 | 原因 | 处理 |
|------|------|------|
| KB 创建失败 | AOSS policy 或 field mapping 不匹配 | 检查 collection ARN、index name、field mapping |
| Ingestion 失败 | service role 不能读 S3 或调用 embedding model | 检查 IAM inline policy 和 bucket prefix |
| retrieve 无结果 | 文档未同步或 index 为空 | 查看 ingestion job failure reasons |
| 持续计费 | AOSS collection 未删除 | 清理时必须删除 collection 和 policies |

## 实验总结

本实验搭建了一条最小但完整的 RAG 链路：S3 文档经 Knowledge Base 的 Data Source 分块并调用 Embedding 模型生成向量，写入 OpenSearch Serverless 索引，`retrieve-and-generate` 先做向量检索再交给文本模型生成带 citation 的回答。验证结果显示模型能正确依据私有政策文档拒绝"客户合同发外部工具"这类违规请求，citation 机制让回答可追溯到具体源文档，是企业级 RAG 应用满足合规审计要求的关键能力，也是 Demo14 深化功能（metadata filter、rerank）的基础。

## 清理

按依赖反向清理：

```bash
aws bedrock-agent delete-data-source \
  --knowledge-base-id ${KNOWLEDGE_BASE_ID} \
  --data-source-id ${DATA_SOURCE_ID} \
  --region ${AWS_REGION}

aws bedrock-agent delete-knowledge-base \
  --knowledge-base-id ${KNOWLEDGE_BASE_ID} \
  --region ${AWS_REGION}

aws s3 rm s3://${BEDROCK_BUCKET}/kb/ --recursive

aws opensearchserverless delete-collection \
  --id ${AOSS_COLLECTION_ID} \
  --region ${AWS_REGION}

aws opensearchserverless delete-access-policy --name ${AOSS_COLLECTION_NAME}-access --type data --region ${AWS_REGION}
aws opensearchserverless delete-security-policy --name ${AOSS_COLLECTION_NAME}-net --type network --region ${AWS_REGION}
aws opensearchserverless delete-security-policy --name ${AOSS_COLLECTION_NAME}-enc --type encryption --region ${AWS_REGION}

aws iam delete-role-policy --role-name ${KB_ROLE_NAME} --policy-name ${KB_ROLE_NAME}-policy
aws iam delete-role --role-name ${KB_ROLE_NAME}
rm -f tmp/kb-*.json tmp/aoss-*.json tmp/ingestion-job.json tmp/retrieve-generate*.json tmp/policy-faq.txt
```
