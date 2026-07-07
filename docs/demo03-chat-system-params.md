# Demo03 — 多轮对话、系统提示词与推理参数

## 实验简介

用 Converse API 实现一个两轮对话，加入 system prompt 和可控推理参数，观察上下文需要由客户端显式传回。

**实验目标：**
- 使用 `system` 定义回答风格和边界
- 维护多轮 messages
- 对比低温度和高温度输出差异

**预计 AI 执行时长：** 10 分钟

---

## 前提条件

- 已完成 Demo02
- 权限包含 `bedrock:InvokeModel`

```bash
export AWS_REGION=us-east-1
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
mkdir -p tmp
```

---

## 步骤

### 1. 第一轮对话

```bash
cat > tmp/chat-turn1.json <<'EOF'
{
  "system": [
    {
      "text": "你是严谨的 AWS 架构师。回答必须简洁、可执行，不编造服务能力。"
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "text": "我要给一个内部知识库做 RAG，先列出最小 AWS 组件。"
        }
      ]
    }
  ],
  "inferenceConfig": {
    "maxTokens": 400,
    "temperature": 0.1
  }
}
EOF

aws bedrock-runtime converse \
  --model-id "${BEDROCK_TEXT_MODEL_ID}" \
  --cli-input-json file://tmp/chat-turn1.json \
  --region ${AWS_REGION} \
  > tmp/chat-turn1-output.json

jq -r '.output.message.content[]?.text' tmp/chat-turn1-output.json
```

### 2. 第二轮带上历史上下文

把第一轮 assistant 回复写回 messages，再追问：

```bash
ASSISTANT_TEXT=$(jq -r '.output.message.content[]?.text' tmp/chat-turn1-output.json)
jq -n --arg assistant "$ASSISTANT_TEXT" '{
  system: [{text: "你是严谨的 AWS 架构师。回答必须简洁、可执行，不编造服务能力。"}],
  messages: [
    {role: "user", content: [{text: "我要给一个内部知识库做 RAG，先列出最小 AWS 组件。"}]},
    {role: "assistant", content: [{text: $assistant}]},
    {role: "user", content: [{text: "把这些组件按创建顺序排序，并说明哪个最容易产生持续费用。"}]}
  ],
  inferenceConfig: {maxTokens: 500, temperature: 0.1}
}' > tmp/chat-turn2.json

aws bedrock-runtime converse \
  --model-id "${BEDROCK_TEXT_MODEL_ID}" \
  --cli-input-json file://tmp/chat-turn2.json \
  --region ${AWS_REGION} \
  > tmp/chat-turn2-output.json

jq -r '.output.message.content[]?.text' tmp/chat-turn2-output.json
```

### 3. 参数对比

```bash
jq '.inferenceConfig.temperature=0.9' tmp/chat-turn2.json > tmp/chat-turn2-hot.json
aws bedrock-runtime converse \
  --model-id "${BEDROCK_TEXT_MODEL_ID}" \
  --cli-input-json file://tmp/chat-turn2-hot.json \
  --region ${AWS_REGION} \
  > tmp/chat-turn2-hot-output.json
```

---

## 验收标准

- 第二轮回答能引用第一轮上下文
- 低温度输出更稳定、结构更一致
- 输出 JSON 中 usage 字段完整

## 清理

```bash
rm -f tmp/chat-turn*.json
```
