# Demo14 — Knowledge Bases 深入：Metadata Filter、Rerank 与 Citation 校验

## 实验简介

在 Demo13 的最小 RAG 上增加生产常见需求：按部门/文档类型过滤、对检索结果重排、校验回答是否有引用来源。

**实验目标：**
- 为文档设计 metadata
- 用 metadata filter 限制检索范围
- 理解 reranking 对召回质量的影响
- 在应用侧强制 citation 校验

**预计 AI 执行时长：** 25-35 分钟

---

## 前提条件

- 已完成 Demo13
- 已有 `KNOWLEDGE_BASE_ID`

```bash
export AWS_REGION=us-east-1
export KNOWLEDGE_BASE_ID=${KNOWLEDGE_BASE_ID:?set this from Demo13}
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
mkdir -p tmp
```

---

## 步骤

### 1. 准备带 metadata 的样例

为 HR、Legal、IT 三类政策分别创建文档和 metadata。metadata 文件格式需与当前 Knowledge Bases S3 data source 支持的格式对齐；执行前用当前 AWS 文档或 CLI schema 确认。

建议 metadata 字段：

| 字段 | 示例 | 用途 |
|------|------|------|
| `department` | `legal` | 部门过滤 |
| `doc_type` | `policy` | 文档类型过滤 |
| `effective_year` | `2026` | 时间过滤 |

### 2. 同步 data source

```bash
aws bedrock-agent start-ingestion-job \
  --knowledge-base-id ${KNOWLEDGE_BASE_ID} \
  --data-source-id ${DATA_SOURCE_ID} \
  --region ${AWS_REGION} \
  > tmp/kb-advanced-ingestion.json
```

轮询到 `COMPLETE` 后继续。

### 3. Metadata filter 检索

```bash
jq -n --arg kb "${KNOWLEDGE_BASE_ID}" '{
  knowledgeBaseId: $kb,
  retrievalQuery: {text: "客户合同能不能发到外部翻译工具？"},
  retrievalConfiguration: {
    vectorSearchConfiguration: {
      numberOfResults: 5,
      filter: {
        equals: {
          key: "department",
          value: "legal"
        }
      }
    }
  }
}' > tmp/kb-filter-retrieve.json

aws bedrock-agent-runtime retrieve \
  --cli-input-json file://tmp/kb-filter-retrieve.json \
  --region ${AWS_REGION} \
  > tmp/kb-filter-retrieve-output.json

jq '.retrievalResults[].metadata' tmp/kb-filter-retrieve-output.json
```

### 4. Citation 校验

应用侧规则：

```bash
jq -e '.citations | length > 0' tmp/retrieve-generate-output.json
jq -e '[.citations[]?.retrievedReferences[]?] | length > 0' tmp/retrieve-generate-output.json
```

如果没有 citation，应用返回“资料不足，无法基于知识库回答”，不要直接展示模型猜测。

### 5. Rerank 选做

如果当前账号和 Region 支持 reranking：
- 设置 reranking configuration
- 对比 rerank 前后的 top results
- 记录 reranker 模型、延迟和 token/费用影响

---

## 验收标准

- metadata filter 后结果只来自目标部门
- retrieve-and-generate 输出包含 citation
- 无 citation 的回答被应用侧拒绝
- 如执行 rerank，记录 top result 顺序变化和延迟变化

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|----------|----------|
| 1 | `jq -e '[.retrievalResults[]?.metadata.department] \| all(. == "legal")' tmp/kb-filter-retrieve-output.json` | `true` |
| 2 | `jq -e '.citations \| length > 0' tmp/retrieve-generate-output.json` | `true` |

## 实验总结

本实验在 Demo13 最小 RAG 基础上补齐了生产环境的常见需求：用 `metadata filter` 把检索范围收窄到指定部门，验证了过滤后的结果不会跨部门泄露；同时在应用侧强制要求 citation 非空，没有引用来源的回答不直接展示给用户，而是明确告知"资料不足"。这一组约束是把 Demo13 的 RAG Demo 升级为可用于企业内部合规场景的关键一步。

## 清理

本 Demo 通常复用 Demo13 资源。只删除新增对象：

```bash
rm -f tmp/kb-advanced-*.json tmp/kb-filter-*.json
```
