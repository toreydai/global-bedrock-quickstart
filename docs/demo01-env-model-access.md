# Demo01 — 准备实验环境与模型访问

## 实验简介

建立本系列所有 Demo 依赖的基础环境：CLI、Python SDK、S3 桶、CloudWatch 日志组，并确认当前账号在 `us-east-1` 可调用的 Bedrock 模型。

**实验目标：**
- 验证 AWS CLI、boto3、jq 可用
- 创建 QuickStart S3 桶和调用日志组
- 枚举当前 Region 可用模型
- 设置后续 Demo 使用的模型环境变量

**实验流程：**
1. 初始化环境变量
2. 检查调用身份和工具版本
3. 创建 S3 桶与日志组
4. 枚举 foundation models 并选择文本/多模态/embedding 模型
5. 记录模型访问状态和后续变量

**预计 AI 执行时长：** 5-10 分钟

---

## 前提条件

- **工具**：AWS CLI 2.x、Python 3.10+、jq
- **权限**：STS、S3、Logs、Bedrock `ListFoundationModels`
- **前提**：账号已开通 Amazon Bedrock；如模型需要显式授权，请先在 Bedrock Console 申请模型访问

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export AWS_PARTITION=aws
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BEDROCK_BUCKET=bedrock-quickstart-${ACCOUNT_ID}-${AWS_REGION}
export BEDROCK_LOG_GROUP=/aws/bedrock/quickstart/invocations
```

---

## 步骤

### 1. 检查身份与工具

```bash
aws sts get-caller-identity --query '{Arn:Arn,Account:Account}' --output table
aws --version
python3 --version
jq --version
```

### 2. 创建 S3 桶和日志组

```bash
aws s3api create-bucket --bucket ${BEDROCK_BUCKET} --region ${AWS_REGION}
aws s3api put-bucket-encryption \
  --bucket ${BEDROCK_BUCKET} \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
aws logs create-log-group --log-group-name ${BEDROCK_LOG_GROUP} --region ${AWS_REGION} || true
aws logs put-retention-policy --log-group-name ${BEDROCK_LOG_GROUP} --retention-in-days 7
```

### 3. 枚举可用模型

```bash
aws bedrock list-foundation-models \
  --region ${AWS_REGION} \
  --query 'modelSummaries[].{ModelId:modelId,Provider:providerName,Input:inputModalities,Output:outputModalities}' \
  --output table
```

选择当前账号可访问的模型并导出变量：

```bash
export BEDROCK_TEXT_MODEL_ID=<available-text-model-id>
export BEDROCK_VISION_MODEL_ID=<available-vision-model-id>
export BEDROCK_EMBED_MODEL_ID=<available-embedding-model-id>
```

### 4. 保存本地环境片段

```bash
mkdir -p tmp
cat > tmp/env.example <<EOF
export AWS_REGION=${AWS_REGION}
export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}
export BEDROCK_BUCKET=${BEDROCK_BUCKET}
export BEDROCK_LOG_GROUP=${BEDROCK_LOG_GROUP}
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID}
export BEDROCK_VISION_MODEL_ID=${BEDROCK_VISION_MODEL_ID}
export BEDROCK_EMBED_MODEL_ID=${BEDROCK_EMBED_MODEL_ID}
EOF
```

---

## 验收标准

- `aws sts get-caller-identity` 返回当前执行身份
- S3 桶存在且开启 SSE-S3 加密
- CloudWatch Logs 日志组存在且保留期为 7 天
- 至少选出一个可调用文本模型

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|----------|----------|
| 1 | `aws s3api head-bucket --bucket ${BEDROCK_BUCKET}` | 退出码为 0 |
| 2 | `aws logs describe-log-groups --log-group-name-prefix ${BEDROCK_LOG_GROUP}` | 返回目标日志组 |
| 3 | `test -n "${BEDROCK_TEXT_MODEL_ID}" && echo ok` | `ok` |

## 实验总结

本实验搭建了整个 QuickStart 系列复用的基础设施：统一的 S3 桶、调用日志组，以及经过枚举确认的可用模型 ID（文本/视觉/embedding）。后续所有 Demo 都依赖这里导出的 `BEDROCK_TEXT_MODEL_ID` 等环境变量，模型访问权限和 Region 选择上的任何偏差都会在这一步先暴露出来，避免在后面的 Demo 中才发现基础环境问题。

## 清理

如果只执行 Demo01 且不继续后续实验：

```bash
aws logs delete-log-group --log-group-name ${BEDROCK_LOG_GROUP} --region ${AWS_REGION}
aws s3 rm s3://${BEDROCK_BUCKET} --recursive
aws s3api delete-bucket --bucket ${BEDROCK_BUCKET} --region ${AWS_REGION}
```
