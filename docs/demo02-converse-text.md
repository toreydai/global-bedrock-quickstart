# Demo02 — 使用 Converse API 完成首次文本推理

## 实验简介

使用 Bedrock Runtime 的 Converse API 调用一个文本模型，理解标准 messages 结构、推理参数和响应解析方式。

**实验目标：**
- 使用 `aws bedrock-runtime converse` 完成一次文本推理
- 解析响应文本、token 使用量和 stopReason
- 验证模型访问、Region 和权限链路可用

**预计 AI 执行时长：** 5 分钟

---

## 前提条件

- 已完成 Demo01
- 已设置 `BEDROCK_TEXT_MODEL_ID`
- 权限包含 `bedrock:InvokeModel`

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
```

---

## 步骤

### 1. 构造请求

```bash
mkdir -p tmp
cat > tmp/converse-text.json <<'EOF'
{
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "text": "用三句话解释 Amazon Bedrock 是什么，面向已经熟悉 AWS 的工程师。"
        }
      ]
    }
  ],
  "inferenceConfig": {
    "maxTokens": 300,
    "temperature": 0.2,
    "topP": 0.9
  }
}
EOF
```

> **已知限制/复测结论：** 早期一次执行怀疑当前 AWS CLI 在 `--cli-input-json` 请求体中处理非 ASCII（中文）文本会报错，曾临时改用英文样例规避。后续用真实账号复测：只要像上一步一样用**带引号的 heredoc**（`cat > file <<'EOF' ... EOF`）把请求体写成 UTF-8 文件，再通过 `--cli-input-json file://...` 引用，AWS CLI v2.33.x 和 v2.34.x（含 Python 3.9 和 3.14 两种运行时）在 `LANG=C.UTF-8`、`LANG=C` 甚至强制 ASCII locale 下都能正确处理中文请求体，未复现编码错误。因此本步骤保留中文示例 prompt。真正的风险点在于**不要把 JSON 直接拼进命令行参数**（例如 `--cli-input-json "$(cat file)"`）或使用不带引号的 heredoc（`<<EOF`，会做 shell 变量展开），这类写法在部分 shell/locale 组合下更容易把多字节字符处理错误；请始终使用本文档展示的“带引号 heredoc 写文件 + `file://` 引用”模式。

### 2. 调用 Converse API

```bash
aws bedrock-runtime converse \
  --model-id "${BEDROCK_TEXT_MODEL_ID}" \
  --cli-input-json file://tmp/converse-text.json \
  --region ${AWS_REGION} \
  > tmp/converse-text-output.json
```

### 3. 解析结果

```bash
jq -r '.output.message.content[]?.text' tmp/converse-text-output.json
jq '{stopReason,usage,metrics}' tmp/converse-text-output.json
```

---

## 验收标准

- 命令返回成功
- 输出包含中文解释
- `usage.inputTokens` 和 `usage.outputTokens` 非空
- `stopReason` 是正常结束状态

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|----------|----------|
| 1 | `jq -r '.output.message.role' tmp/converse-text-output.json` | `assistant` |
| 2 | `jq -e '.usage.inputTokens > 0 and .usage.outputTokens > 0' tmp/converse-text-output.json` | `true` |

## 清理

```bash
rm -f tmp/converse-text.json tmp/converse-text-output.json
```
