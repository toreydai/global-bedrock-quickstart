# Demo04 — ConverseStream 流式输出

## 实验简介

使用 `converse-stream` 获取增量输出，理解 Bedrock streaming 响应事件和客户端消费方式。

**实验目标：**
- 调用 `bedrock-runtime converse-stream`
- 提取 contentBlockDelta 中的文本增量
- 理解流式输出适合聊天 UI 和长文本生成

**预计 AI 执行时长：** 10 分钟

---

## 前提条件

- 已完成 Demo02
- 权限包含 `bedrock:InvokeModelWithResponseStream`

```bash
export AWS_REGION=us-east-1
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
mkdir -p tmp
```

---

## 步骤

### 1. 准备流式请求

```bash
cat > tmp/stream.json <<'EOF'
{
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "text": "写一个 6 行以内的 Bedrock 应用上线 checklist，面向平台工程团队。"
        }
      ]
    }
  ],
  "inferenceConfig": {
    "maxTokens": 500,
    "temperature": 0.2
  }
}
EOF
```

### 2. 执行 streaming 调用

```bash
aws bedrock-runtime converse-stream \
  --model-id "${BEDROCK_TEXT_MODEL_ID}" \
  --cli-input-json file://tmp/stream.json \
  --region ${AWS_REGION} \
  > tmp/stream-output.json
```

> **已知限制：** 当前 AWS CLI 版本未暴露 `bedrock-runtime converse-stream` 子命令（`Found invalid choice: 'converse-stream'`）。CLI 不支持时，改用下面的 boto3 方法作为可用路径 —— boto3 的 `bedrock-runtime` 客户端支持 `converse_stream`，即使 CLI 不支持。

**boto3 替代方法（已验证可用）：**

```bash
cat > tmp/stream_boto3.py <<'PY'
import json
import boto3

client = boto3.client("bedrock-runtime", region_name="us-east-1")
model_id = "amazon.nova-lite-v1:0"

response = client.converse_stream(
    modelId=model_id,
    messages=[{
        "role": "user",
        "content": [{"text": "写一个 6 行以内的 Bedrock 应用上线 checklist，面向平台工程团队。"}]
    }],
    inferenceConfig={"maxTokens": 500, "temperature": 0.2},
)

event_types = []
text_chunks = []
for event in response["stream"]:
    key = next(iter(event.keys()))
    event_types.append(key)
    if key == "contentBlockDelta":
        delta_text = event["contentBlockDelta"].get("delta", {}).get("text")
        if delta_text:
            text_chunks.append(delta_text)

print("event types seen:", event_types)
print("concatenated text:", "".join(text_chunks))
PY

python3 tmp/stream_boto3.py
```

预期能看到 `messageStart`、多个 `contentBlockDelta`、`contentBlockStop`、`messageStop`、`metadata` 事件类型，以及拼接后的完整文本。

### 3. 提取文本片段

```bash
jq -r '.stream[]? | .contentBlockDelta.delta.text? // empty' tmp/stream-output.json
jq -r '[.stream[]? | .contentBlockDelta.delta.text? // empty] | join("")' tmp/stream-output.json
```

---

## 验收标准

- streaming 调用成功（当前 CLI 不支持时，以 boto3 `converse_stream` 验证）
- 能从事件流中拼接出完整文本
- 能看到 messageStart、contentBlockDelta、messageStop 或 metadata 类事件

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|----------|----------|
| 1 | `python3 tmp/stream_boto3.py 2>&1 \| grep "event types seen:"` | 输出包含 `contentBlockDelta` |
| 2 | `python3 tmp/stream_boto3.py 2>&1 \| grep "concatenated text:"` | 输出非空文本 |

## 实验总结

本实验验证了 Bedrock 流式输出的事件结构（`messageStart` → 多个 `contentBlockDelta` → `contentBlockStop` → `messageStop`/`metadata`），并确认当前 AWS CLI 未暴露 `converse-stream` 子命令时，boto3 的 `converse_stream` 是可靠的替代路径。流式输出通过降低首字节等待时间显著改善聊天类 UI 的交互体验，是生产应用相比一次性返回更常用的调用方式。

## 清理

```bash
rm -f tmp/stream.json tmp/stream-output.json tmp/stream_boto3.py
```
