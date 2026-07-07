# Demo18 — Prompt 管理、模型评估与成本清理审计

## 实验简介

把前面 Demo 中的临时 prompt 收敛成可管理资产，并建立基础评估和成本治理动作。这个 Demo 是整个 QuickStart 的收尾。

**实验目标：**
- 梳理 prompt 版本管理策略
- 建立最小离线评估集
- 对比两个模型或两版 prompt 的输出质量与 token 成本
- 检查并清理所有可能持续计费的资源

**预计 AI 执行时长：** 30-45 分钟

---

## 前提条件

- 已完成 Demo02 / Demo06 / Demo13
- 权限包含 Bedrock、S3、Budgets（可选）、CloudWatch Logs

```bash
export AWS_REGION=us-east-1
export BEDROCK_BUCKET=${BEDROCK_BUCKET:?set this from Demo01}
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
mkdir -p tmp
```

---

## 步骤

### 1. 创建最小评估集

```bash
cat > tmp/eval-set.jsonl <<'EOF'
{"id":"q1","input":"用两句话解释 Bedrock Converse API。","expected_keywords":["Converse","messages","模型"]}
{"id":"q2","input":"列出 Bedrock RAG 最小组件。","expected_keywords":["Knowledge Base","S3","embedding"]}
{"id":"q3","input":"员工可以把客户合同截图发到外部翻译工具吗？","expected_keywords":["不能","敏感","授权"]}
{"id":"q4","input":"资料里没有海外差旅报销上限时应该怎么回答？","expected_keywords":["资料不足","不能猜测"]}
EOF

aws s3 cp tmp/eval-set.jsonl s3://${BEDROCK_BUCKET}/eval/eval-set.jsonl
```

### 2. 运行轻量评估

```bash
while read -r line; do
  ID=$(echo "$line" | jq -r '.id')
  INPUT=$(echo "$line" | jq -r '.input')
  jq -n --arg input "$INPUT" '{
    system: [{text: "你是企业内部政策助手。回答必须简洁，不能编造资料中没有的信息。"}],
    messages: [{role:"user", content:[{text:$input}]}],
    inferenceConfig: {maxTokens: 300, temperature: 0.1}
  }' > tmp/eval-request-${ID}.json
  aws bedrock-runtime converse \
    --model-id "${BEDROCK_TEXT_MODEL_ID}" \
    --cli-input-json file://tmp/eval-request-${ID}.json \
    --region ${AWS_REGION} \
    > tmp/eval-output-${ID}.json
  jq -r --arg id "$ID" '{id:$id,text:.output.message.content[0].text,usage:.usage}' tmp/eval-output-${ID}.json
done < tmp/eval-set.jsonl > tmp/eval-results.jsonl
```

> **已验证发现：** 本步骤只带了一句 system prompt（"不能编造资料中没有的信息"），没有接入任何 RAG 检索或 Guardrail。真实执行时 `q3`（客户合同截图能否发外部翻译工具）模型实际回答"可以"，未命中 `expected_keywords` 里的"不能"——这**是预期内的失败样例**，不是评估脚本或模型的缺陷。它恰好证明：仅靠 system prompt 约束不足以保证策略合规，必须叠加 Demo10/11 的 Guardrails（拦截/改写不合规回答）或 Demo13/14 的 RAG 检索（把回答锚定到真实政策文档）才能让这类问题稳定给出正确答案。汇总评估结果时，应把 `q3` 标记为"基线预期失败、待接入 Guardrail/RAG 后复测"，而不是当作脚本报错处理。

### 3. 汇总 token 使用

```bash
jq -s '{
  cases: length,
  inputTokens: map(.usage.inputTokens) | add,
  outputTokens: map(.usage.outputTokens) | add,
  totalTokens: ((map(.usage.inputTokens) | add) + (map(.usage.outputTokens) | add))
}' tmp/eval-results.jsonl
```

### 4. 正式 Bedrock model evaluation 选做

如果当前账号支持 Bedrock model evaluation job：
- 上传评估数据集到 S3
- 创建 evaluation job
- 选择 automatic 或 human evaluation
- 记录指标、模型版本、prompt 版本和费用

### 5. 成本清理审计

必须检查：

```bash
aws logs describe-log-groups --log-group-name-prefix /aws/bedrock --region ${AWS_REGION}
aws s3 ls s3://${BEDROCK_BUCKET}/ --recursive
aws bedrock-agent list-knowledge-bases --region ${AWS_REGION}
aws bedrock-agent list-agents --region ${AWS_REGION}
aws opensearchserverless list-collections --region ${AWS_REGION} || true
aws lambda list-functions --region ${AWS_REGION} --query "Functions[?contains(FunctionName, 'bedrock')].FunctionName"
```

持续费用重点：
- OpenSearch Serverless collection
- Provisioned throughput
- Lambda/API Gateway 日志
- CloudWatch Logs 存储
- S3 对象
- 高量模型调用

---

## 验收标准

- 评估集已上传 S3
- 每条评估问题都有响应和 token usage
- 输出 token 汇总
- 列出当前仍会持续计费的资源
- 明确哪些资源已清理、哪些保留给后续实验

## 清理

```bash
aws s3 rm s3://${BEDROCK_BUCKET}/eval/ --recursive
rm -f tmp/eval-*.json tmp/eval-*.jsonl tmp/eval-results.jsonl
```

按执行过的 Demo 逐项清理 Knowledge Base、Agent、Guardrail、Lambda、OpenSearch Serverless、CloudWatch Logs 和 S3 数据。
