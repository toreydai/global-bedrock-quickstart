# Demo05 — 多模态输入：图片与文档理解

## 实验简介

通过 Converse API 发送图片或文档内容，验证多模态模型的输入格式和基础解析能力。

**实验目标：**
- 选择支持 image/document 输入的模型
- 用本地小样例测试图片理解
- 用短文档测试摘要和结构化提取

**预计 AI 执行时长：** 10-15 分钟

---

## 前提条件

- 已完成 Demo01
- 已设置 `BEDROCK_VISION_MODEL_ID`
- 权限包含 `bedrock:InvokeModel`

```bash
export AWS_REGION=us-east-1
export BEDROCK_VISION_MODEL_ID=${BEDROCK_VISION_MODEL_ID:?set this from Demo01}
mkdir -p tmp
```

---

## 步骤

### 1. 确认模型支持多模态

```bash
aws bedrock list-foundation-models \
  --region ${AWS_REGION} \
  --query "modelSummaries[?modelId=='${BEDROCK_VISION_MODEL_ID}']"
```

### 2. 创建一个短文档请求

```bash
printf 'Amazon Bedrock QuickStart\n\n目标：用最少步骤验证模型调用、RAG、安全护栏和成本治理。\n负责人：平台工程团队。\n' > tmp/sample.txt

jq -n --rawfile doc tmp/sample.txt '{
  messages: [
    {
      role: "user",
      content: [
        {text: "请提取这份文档的目标和负责人，返回 JSON。"},
        {
          document: {
            format: "txt",
            name: "QuickStartSample",
            source: {bytes: ($doc | @base64)}
          }
        }
      ]
    }
  ],
  inferenceConfig: {maxTokens: 300, temperature: 0.1}
}' > tmp/document-request.json
```

### 3. 调用 Converse

```bash
aws bedrock-runtime converse \
  --model-id "${BEDROCK_VISION_MODEL_ID}" \
  --cli-input-json file://tmp/document-request.json \
  --region ${AWS_REGION} \
  > tmp/document-output.json

jq -r '.output.message.content[]?.text' tmp/document-output.json
```

### 4. 图片输入选做

如果本地有小于模型限制的 PNG/JPEG 文件：

```bash
export IMAGE_PATH=tmp/sample.png
export IMAGE_FORMAT=png
IMAGE_B64=$(base64 -w 0 "${IMAGE_PATH}")
jq -n --arg img "${IMAGE_B64}" --arg fmt "${IMAGE_FORMAT}" '{
  messages: [
    {
      role: "user",
      content: [
        {text: "请描述这张图片中的主要对象和潜在风险。"},
        {image: {format: $fmt, source: {bytes: $img}}}
      ]
    }
  ],
  inferenceConfig: {maxTokens: 400, temperature: 0.1}
}' > tmp/image-request.json
```

---

## 验收标准

- 文档请求返回结构化摘要
- 响应能正确识别文档中的目标和负责人
- 如执行图片选做，模型能描述图片内容

## 清理

```bash
rm -f tmp/sample.txt tmp/document-request.json tmp/document-output.json tmp/image-request.json
```
