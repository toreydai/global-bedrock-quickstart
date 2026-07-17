# Demo12 — Tool Use / Function Calling 工具调用

## 实验简介

使用 Converse API 的 tool use 能力，让模型在需要时请求调用一个本地工具，再把工具结果回传给模型生成最终答案。

**实验目标：**
- 定义 tool schema
- 识别模型返回的 `toolUse`
- 回传 `toolResult`
- 完成一轮工具增强回答

**预计 AI 执行时长：** 15-20 分钟

---

## 前提条件

- 已完成 Demo02 和 Demo06
- 所选模型支持 tool use

```bash
export AWS_REGION=us-east-1
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
mkdir -p tmp
```

---

## 步骤

### 1. 发送带工具定义的请求

```bash
cat > tmp/tool-request.json <<'EOF'
{
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "text": "查询 bedrock-quickstart 项目的当前预算状态，并给出一句建议。"
        }
      ]
    }
  ],
  "toolConfig": {
    "tools": [
      {
        "toolSpec": {
          "name": "get_budget_status",
          "description": "Return the current workshop budget status.",
          "inputSchema": {
            "json": {
              "type": "object",
              "properties": {
                "project": {
                  "type": "string"
                }
              },
              "required": ["project"]
            }
          }
        }
      }
    ]
  },
  "inferenceConfig": {
    "maxTokens": 400,
    "temperature": 0
  }
}
EOF

aws bedrock-runtime converse \
  --model-id "${BEDROCK_TEXT_MODEL_ID}" \
  --cli-input-json file://tmp/tool-request.json \
  --region ${AWS_REGION} \
  > tmp/tool-step1-output.json
```

### 2. 提取 toolUse

```bash
jq '.output.message.content[] | select(.toolUse)' tmp/tool-step1-output.json
export TOOL_USE_ID=$(jq -r '.output.message.content[] | select(.toolUse) | .toolUse.toolUseId' tmp/tool-step1-output.json)
```

### 3. 回传工具结果

```bash
jq -n --arg toolUseId "${TOOL_USE_ID}" '{
  messages: [
    {
      role: "user",
      content: [{text: "查询 bedrock-quickstart 项目的当前预算状态，并给出一句建议。"}]
    },
    {
      role: "assistant",
      content: [
        {
          toolUse: {
            toolUseId: $toolUseId,
            name: "get_budget_status",
            input: {project: "bedrock-quickstart"}
          }
        }
      ]
    },
    {
      role: "user",
      content: [
        {
          toolResult: {
            toolUseId: $toolUseId,
            content: [
              {
                json: {
                  project: "bedrock-quickstart",
                  monthlyBudgetUsd: 50,
                  currentSpendUsd: 7.25,
                  status: "OK"
                }
              }
            ]
          }
        }
      ]
    }
  ],
  inferenceConfig: {maxTokens: 300, temperature: 0}
}' > tmp/tool-step2-request.json

aws bedrock-runtime converse \
  --model-id "${BEDROCK_TEXT_MODEL_ID}" \
  --cli-input-json file://tmp/tool-step2-request.json \
  --region ${AWS_REGION} \
  > tmp/tool-step2-output.json

jq -r '.output.message.content[]?.text' tmp/tool-step2-output.json
```

---

## 验收标准

- 第一次响应包含 `toolUse`
- 第二次响应能引用工具返回的预算状态
- 模型没有伪造未提供的预算数据

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|----------|----------|
| 1 | `jq -e '.output.message.content[] \| select(.toolUse) \| .toolUse.name == "get_budget_status"' tmp/tool-step1-output.json` | `true` |
| 2 | `jq -r '.output.message.content[]?.text' tmp/tool-step2-output.json \| grep -E "7.25\|OK"` | 输出中包含工具回传的预算数值或状态 |

## 实验总结

本实验验证了 Converse API tool use 的完整闭环：模型判断需要调用工具并返回结构化 `toolUse` 请求，客户端执行工具逻辑后把 `toolResult` 回传，模型基于真实工具结果生成最终回答，且没有凭空编造未提供的数据。这是构建"轻量级工具增强问答"应用（区别于 Demo15 完整 Agent）的最小可行模式，也是理解 Demo15 Action Group 底层机制的基础。

## 清理

```bash
rm -f tmp/tool-*.json
```
