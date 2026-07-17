# Demo07 — Lambda 与 API Gateway 封装最小 Bedrock 应用

## 实验简介

把 Converse API 封装成一个最小 HTTP 应用：API Gateway 接收问题，Lambda 调用 Bedrock，返回企业政策助手答案。

**实验目标：**
- 创建 Lambda execution role
- 部署 Python Lambda 调用 Bedrock Runtime
- 用 Function URL 或 API Gateway 暴露 HTTP 接口
- 验证应用层错误处理和超时设置

**预计 AI 执行时长：** 20-30 分钟

---

## 前提条件

- 已完成 Demo02 和 Demo06
- 权限包含 IAM、Lambda、Logs、API Gateway 或 Lambda Function URL、Bedrock Runtime

```bash
export AWS_REGION=us-east-1
export AWS_PARTITION=aws
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
export APP_NAME=bedrock-policy-assistant
mkdir -p tmp
```

---

## 步骤

### 1. 创建 Lambda 执行角色

```bash
cat > tmp/lambda-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ${APP_NAME}-role \
  --assume-role-policy-document file://tmp/lambda-trust.json

cat > tmp/lambda-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:${AWS_PARTITION}:logs:${AWS_REGION}:${ACCOUNT_ID}:*"
    },
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:${AWS_PARTITION}:bedrock:${AWS_REGION}::foundation-model/${BEDROCK_TEXT_MODEL_ID}"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name ${APP_NAME}-role \
  --policy-name ${APP_NAME}-policy \
  --policy-document file://tmp/lambda-policy.json

export LAMBDA_ROLE_ARN=arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:role/${APP_NAME}-role
```

### 2. 创建 Lambda 代码

```bash
cat > tmp/lambda_function.py <<'PY'
import json
import os

import boto3

bedrock = boto3.client("bedrock-runtime")
MODEL_ID = os.environ["MODEL_ID"]


def handler(event, context):
    try:
        body = json.loads(event.get("body") or "{}")
        question = body.get("question", "").strip()
        if not question:
            return {"statusCode": 400, "body": json.dumps({"error": "question is required"})}

        response = bedrock.converse(
            modelId=MODEL_ID,
            system=[{"text": "你是企业内部政策助手。回答必须简洁，并提醒不要提交敏感数据。"}],
            messages=[{"role": "user", "content": [{"text": question}]}],
            inferenceConfig={"maxTokens": 500, "temperature": 0.1},
            requestMetadata={"app": "bedrock-policy-assistant"},
        )
        text = "".join(part.get("text", "") for part in response["output"]["message"]["content"])
        return {"statusCode": 200, "body": json.dumps({"answer": text, "usage": response.get("usage", {})}, ensure_ascii=False)}
    except Exception as exc:
        return {"statusCode": 500, "body": json.dumps({"error": type(exc).__name__, "message": str(exc)})}
PY

cd tmp
zip lambda.zip lambda_function.py
cd -
```

### 3. 部署 Lambda

```bash
aws lambda create-function \
  --function-name ${APP_NAME} \
  --runtime python3.12 \
  --role ${LAMBDA_ROLE_ARN} \
  --handler lambda_function.handler \
  --zip-file fileb://tmp/lambda.zip \
  --timeout 30 \
  --memory-size 256 \
  --environment Variables="{MODEL_ID=${BEDROCK_TEXT_MODEL_ID}}" \
  --region ${AWS_REGION}

aws lambda wait function-active --function-name ${APP_NAME} --region ${AWS_REGION}
```

### 4. 本地 invoke 验证

```bash
cat > tmp/lambda-event.json <<'EOF'
{
  "body": "{\"question\":\"员工可以把客户合同截图发给外部 AI 工具翻译吗？\"}"
}
EOF

aws lambda invoke \
  --function-name ${APP_NAME} \
  --payload fileb://tmp/lambda-event.json \
  --region ${AWS_REGION} \
  tmp/lambda-response.json

cat tmp/lambda-response.json | jq
```

### 5. Function URL 选做

```bash
aws lambda create-function-url-config \
  --function-name ${APP_NAME} \
  --auth-type AWS_IAM \
  --region ${AWS_REGION}
```

生产环境建议接入 API Gateway、WAF、认证和调用配额。本 Demo 的目标是最小可用应用封装。

---

## 验收标准

- Lambda 调用成功
- 响应包含 `answer` 和 `usage`
- 空问题返回 400
- Lambda execution role 只允许调用指定模型

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|----------|----------|
| 1 | `cat tmp/lambda-response.json \| jq -r '.statusCode'` | `200` |
| 2 | `cat tmp/lambda-response.json \| jq -r '.body' \| jq -e '.answer and .usage'` | `true` |
| 3 | `aws iam get-role-policy --role-name ${APP_NAME}-role --policy-name ${APP_NAME}-policy --query 'PolicyDocument.Statement[?Action==`bedrock:InvokeModel`].Resource' --output text` | 输出为具体模型 ARN，不是 `*` |

## 常见失败与处理

| 现象 | 原因 | 处理 |
|------|------|------|
| Lambda `AccessDeniedException` | execution role 缺少模型 ARN 权限 | 检查 policy 中模型 ID 和 Region |
| Lambda import boto3 失败 | runtime 异常或打包问题 | 使用 Lambda Python runtime 自带 boto3 或打包依赖 |
| Lambda timeout | 模型响应慢或 timeout 太低 | timeout 提高到 30-60 秒，应用侧设置重试策略 |

## 实验总结

本实验把 Demo02 的 Converse API 调用封装成一个端到端 HTTP 应用：API Gateway/Function URL 接收请求、Lambda 调用 Bedrock、返回结构化 JSON，并验证了空输入校验、异常兜底和最小权限执行角色（Resource 限定到具体模型 ARN）。这是本系列中唯一一个演示"如何把 Bedrock 能力包装成可对外提供服务的应用"的 Demo，为 Demo08 的调用日志和 Demo09 的 IAM 权限收敛提供了被测对象。

## 清理

```bash
aws lambda delete-function-url-config --function-name ${APP_NAME} --region ${AWS_REGION} || true
aws lambda delete-function --function-name ${APP_NAME} --region ${AWS_REGION}
aws iam delete-role-policy --role-name ${APP_NAME}-role --policy-name ${APP_NAME}-policy
aws iam delete-role --role-name ${APP_NAME}-role
rm -f tmp/lambda-*.json tmp/lambda_function.py tmp/lambda.zip
```
