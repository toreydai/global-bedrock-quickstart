# Demo10 — Guardrails 内容安全与敏感信息过滤

## 实验简介

创建一个最小 Guardrail，并通过 Converse API 应用到请求，验证内容过滤、PII 处理和拒答行为。

**实验目标：**
- 创建 Guardrail 和版本
- 在 Converse 请求中引用 Guardrail
- 用安全测试样例验证拦截效果

**预计 AI 执行时长：** 15-20 分钟

---

## 前提条件

- 已完成 Demo02 和 Demo06
- 权限包含 Bedrock Guardrails 管理和 `bedrock:InvokeModel`

```bash
export AWS_REGION=us-east-1
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
export GUARDRAIL_NAME=bedrock-quickstart-guardrail
mkdir -p tmp
```

---

## 步骤

### 1. 创建 Guardrail

```bash
aws bedrock create-guardrail \
  --name ${GUARDRAIL_NAME} \
  --description "QuickStart guardrail for basic safety and PII tests" \
  --blocked-input-messaging "I cannot help with that request." \
  --blocked-outputs-messaging "I cannot provide that response." \
  --content-policy-config 'filtersConfig=[{type=HATE,inputStrength=MEDIUM,outputStrength=MEDIUM},{type=VIOLENCE,inputStrength=MEDIUM,outputStrength=MEDIUM},{type=MISCONDUCT,inputStrength=MEDIUM,outputStrength=MEDIUM}]' \
  --sensitive-information-policy-config 'piiEntitiesConfig=[{type=EMAIL,action=ANONYMIZE},{type=PHONE,action=ANONYMIZE}]' \
  --region ${AWS_REGION} \
  > tmp/guardrail-create.json

export GUARDRAIL_ID=$(jq -r '.guardrailId' tmp/guardrail-create.json)
```

### 2. 创建版本

```bash
aws bedrock create-guardrail-version \
  --guardrail-identifier ${GUARDRAIL_ID} \
  --description "v1" \
  --region ${AWS_REGION} \
  > tmp/guardrail-version.json

export GUARDRAIL_VERSION=$(jq -r '.version' tmp/guardrail-version.json)
```

### 3. 应用到 Converse 请求

```bash
jq -n --arg gid "${GUARDRAIL_ID}" --arg gv "${GUARDRAIL_VERSION}" '{
  messages: [
    {
      role: "user",
      content: [
        {text: "请把这个联系方式改写成一句客服备注：alice@example.com, +1-202-555-0100"}
      ]
    }
  ],
  guardrailConfig: {
    guardrailIdentifier: $gid,
    guardrailVersion: $gv,
    trace: "enabled"
  },
  inferenceConfig: {maxTokens: 300, temperature: 0}
}' > tmp/guardrail-converse.json

aws bedrock-runtime converse \
  --model-id "${BEDROCK_TEXT_MODEL_ID}" \
  --cli-input-json file://tmp/guardrail-converse.json \
  --region ${AWS_REGION} \
  > tmp/guardrail-output.json

jq -r '.output.message.content[]?.text' tmp/guardrail-output.json
jq '.trace // empty' tmp/guardrail-output.json
```

---

## 验收标准

- Guardrail 创建成功并有可用版本
- Converse 响应应用了 guardrailConfig
- PII 测试样例被匿名化或触发 trace 记录

## 清理

```bash
aws bedrock delete-guardrail \
  --guardrail-identifier ${GUARDRAIL_ID} \
  --region ${AWS_REGION}
rm -f tmp/guardrail-*.json
```
