# Demo06 — Prompt 模板与结构化 JSON 输出

## 实验简介

把前面自由文本问答收敛成可复用 prompt 模板，并要求模型返回稳定 JSON。这个 Demo 是后续评估、工具调用和应用封装的基础。

**实验目标：**
- 编写面向企业政策助手的 system prompt
- 要求模型返回固定 JSON schema
- 用 `jq` 验证输出是否可解析
- 观察 prompt 约束不足时的失败模式

**预计 AI 执行时长：** 10-15 分钟

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

### 1. 创建结构化输出请求

```bash
cat > tmp/structured-request.json <<'EOF'
{
  "system": [
    {
      "text": "你是企业内部政策助手。你只能根据用户问题提取意图和风险等级。必须只返回 JSON，不要返回 Markdown。JSON 字段固定为 intent、risk_level、answer_style、follow_up_needed。risk_level 只能是 low、medium、high。"
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "text": "员工问：我能不能把客户合同截图发到外部翻译工具里？"
        }
      ]
    }
  ],
  "inferenceConfig": {
    "maxTokens": 300,
    "temperature": 0
  }
}
EOF
```

### 2. 调用模型并提取文本

```bash
aws bedrock-runtime converse \
  --model-id "${BEDROCK_TEXT_MODEL_ID}" \
  --cli-input-json file://tmp/structured-request.json \
  --region ${AWS_REGION} \
  > tmp/structured-output.json

jq -r '.output.message.content[]?.text' tmp/structured-output.json > tmp/structured-text.json
cat tmp/structured-text.json
```

### 3. 验证 JSON 和字段

```bash
jq -e '.intent and .risk_level and .answer_style and (.follow_up_needed|type=="boolean")' tmp/structured-text.json
jq -e '.risk_level == "low" or .risk_level == "medium" or .risk_level == "high"' tmp/structured-text.json
```

### 4. 失败路径测试

把 system prompt 中“必须只返回 JSON”删掉再调用一次，观察输出是否混入解释文本。记录是否需要更强约束或应用侧 JSON 修复逻辑。

---

## 验收标准

- 模型输出能被 `jq` 解析
- 四个字段都存在
- `risk_level` 落在枚举范围内
- 失败路径测试能说明 prompt 约束对应用稳定性的影响

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|----------|----------|
| 1 | `jq -e '.intent and .risk_level and .answer_style and (.follow_up_needed\|type=="boolean")' tmp/structured-text.json` | `true` |
| 2 | `jq -r '.risk_level' tmp/structured-text.json` | `low`、`medium` 或 `high` 三者之一 |

## 常见失败与处理

| 现象 | 原因 | 处理 |
|------|------|------|
| 输出包含 Markdown fenced block | prompt 约束不足 | 强化 system prompt，应用侧去除代码块后再解析 |
| `jq` 解析失败 | 模型返回自然语言 | 降低 temperature，加入字段示例 |
| 字段缺失 | schema 描述不够明确 | 把字段和枚举写进 system prompt |

## 实验总结

本实验把自由文本问答收敛为可解析的结构化 JSON 输出，验证了 `temperature=0` + 明确 schema 约束下模型输出的稳定性；同时通过对比"去掉强约束"的失败路径，证明了 prompt 工程本身无法 100% 保证格式正确，应用侧仍需具备 JSON 解析容错能力。这里建立的结构化输出模式是后续 Demo10/11 Guardrails 校验和 Demo18 评估打分的基础。

## 清理

```bash
rm -f tmp/structured-*.json
```
