# Demo17 — Batch Inference、Prompt Caching 与吞吐优化

## 实验简介

面向离线批处理和高并发应用，比较实时调用、batch inference、prompt caching 和 provisioned throughput 的适用场景。

**实验目标：**
- 判断任务适合实时、异步还是 batch
- 准备 batch inference 输入输出 S3 路径
- 理解 prompt caching 对重复长上下文的价值
- 形成吞吐优化决策表

**预计 AI 执行时长：** 30-45 分钟

---

## 前提条件

- 已完成 Demo01 和 Demo12
- 权限包含 Bedrock batch/model invocation job、S3、IAM

```bash
export AWS_REGION=us-east-1
export AWS_PARTITION=aws
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BEDROCK_BUCKET=${BEDROCK_BUCKET:?set this from Demo01}
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
export BATCH_JOB_NAME=bedrock-quickstart-batch
mkdir -p tmp
```

---

## 步骤

### 1. 选择推理模式

| 模式 | 适合场景 | 不适合场景 |
|------|----------|------------|
| Realtime Converse | 低延迟交互 | 大批量离线任务 |
| Streaming | 聊天 UI、长回答体验优化 | 后台批处理 |
| Batch inference | 离线分类、摘要、数据标注 | 用户实时请求 |
| Prompt caching | 多请求共享长上下文 | 短 prompt、上下文变化大 |
| Provisioned throughput | 稳定高吞吐、低延迟 | 流量很小或高度波动 |

### 2. 准备 batch 输入

具体 JSONL 格式会随模型和 API 形态变化，执行前用当前 Bedrock batch inference 文档确认。建议用 3 条企业政策助手分类任务：

```bash
cat > tmp/batch-input.jsonl <<'EOF'
{"recordId":"1","modelInput":{"messages":[{"role":"user","content":[{"text":"判断风险等级：员工要把客户合同发到外部翻译工具。"}]}],"inferenceConfig":{"maxTokens":200,"temperature":0}}}
{"recordId":"2","modelInput":{"messages":[{"role":"user","content":[{"text":"判断风险等级：员工询问年假政策。"}]}],"inferenceConfig":{"maxTokens":200,"temperature":0}}}
{"recordId":"3","modelInput":{"messages":[{"role":"user","content":[{"text":"判断风险等级：员工要把 AWS access key 发给供应商。"}]}],"inferenceConfig":{"maxTokens":200,"temperature":0}}}
EOF

aws s3 cp tmp/batch-input.jsonl s3://${BEDROCK_BUCKET}/batch/input/batch-input.jsonl
```

### 3. 创建 batch job

创建 batch job 需要一个 Bedrock 可承担的 role，允许读写指定 S3 prefix 并调用模型。按当前 CLI schema 执行：

```bash
aws bedrock create-model-invocation-job \
  --job-name ${BATCH_JOB_NAME} \
  --role-arn ${BEDROCK_BATCH_ROLE_ARN} \
  --model-id ${BEDROCK_TEXT_MODEL_ID} \
  --input-data-config "s3InputDataConfig={s3Uri=s3://${BEDROCK_BUCKET}/batch/input/}" \
  --output-data-config "s3OutputDataConfig={s3Uri=s3://${BEDROCK_BUCKET}/batch/output/}" \
  --region ${AWS_REGION}
```

轮询：

```bash
aws bedrock get-model-invocation-job \
  --job-identifier ${BATCH_JOB_NAME} \
  --region ${AWS_REGION}
```

### 4. Prompt caching 设计

如果当前模型支持 prompt caching：
- 把企业政策长上下文作为可缓存前缀
- 对同一前缀运行多条问题
- 对比 cache read/write token、延迟和费用

记录：

| 指标 | 无缓存 | 有缓存 |
|------|--------|--------|
| 首次延迟 | | |
| 后续延迟 | | |
| input tokens | | |
| cache read tokens | | |
| 成本变化 | | |

---

## 验收标准

- batch 输入上传成功
- batch job 创建成功或明确记录当前账号/模型不支持原因
- 输出结果可从 S3 下载并解析
- prompt caching 形成适用/不适用判断

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|----------|----------|
| 1 | `aws s3 ls s3://${BEDROCK_BUCKET}/batch/input/batch-input.jsonl` | 返回文件列表，非空 |
| 2 | `aws bedrock get-model-invocation-job --job-identifier ${BATCH_JOB_NAME} --region ${AWS_REGION} --query 'status' --output text` | `Completed`（或已记录当前账号/模型不支持的明确原因） |

## 实验总结

本实验对比了实时调用、streaming、batch inference、prompt caching 和 provisioned throughput 五种推理模式的适用场景，并实际跑通了一次 batch inference 任务（离线批量风险分类）。这类决策表是把 Bedrock 应用从"能跑通单次调用"推进到"针对不同流量特征选择合适吞吐/成本方案"的关键一步，是 Demo18 成本审计的前置输入。

## 清理

```bash
aws s3 rm s3://${BEDROCK_BUCKET}/batch/ --recursive
rm -f tmp/batch-input.jsonl
```

如创建了 batch role，删除对应 inline policy 和 IAM role。
