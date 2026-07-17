# Demo15 — Agents for Bedrock + Lambda Action Group

## 实验简介

创建一个最小 Bedrock Agent，让它根据用户问题调用 Lambda Action Group 查询“预算状态”，再把工具结果整合成最终回答。

**实验目标：**
- 创建 Agent 执行角色
- 创建 Lambda 工具函数和 resource policy
- 定义 OpenAPI schema
- 创建 Action Group
- Prepare Agent、创建 Alias 并调用

**预计 AI 执行时长：** 40-60 分钟

---

## 前提条件

- 已完成 Demo07 和 Demo12
- 权限包含 Bedrock Agent、IAM、Lambda、Logs

```bash
export AWS_REGION=us-east-1
export AWS_PARTITION=aws
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
export AGENT_NAME=bedrock-quickstart-agent
export AGENT_ROLE_NAME=BedrockQuickStartAgentRole
export TOOL_FUNCTION_NAME=bedrock-quickstart-budget-tool
mkdir -p tmp
```

---

## 步骤

### 1. 创建 Agent 执行角色

```bash
cat > tmp/agent-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "bedrock.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ${AGENT_ROLE_NAME} \
  --assume-role-policy-document file://tmp/agent-trust.json

cat > tmp/agent-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:${AWS_PARTITION}:bedrock:${AWS_REGION}::foundation-model/${BEDROCK_TEXT_MODEL_ID}"
    },
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:${AWS_PARTITION}:lambda:${AWS_REGION}:${ACCOUNT_ID}:function:${TOOL_FUNCTION_NAME}"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name ${AGENT_ROLE_NAME} \
  --policy-name ${AGENT_ROLE_NAME}-policy \
  --policy-document file://tmp/agent-policy.json

export AGENT_ROLE_ARN=arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:role/${AGENT_ROLE_NAME}
```

### 2. 创建 Lambda 工具

```bash
cat > tmp/tool_lambda.py <<'PY'
import json


def lambda_handler(event, context):
    # Bedrock Agent action group event shape may include apiPath/httpMethod/requestBody.
    response_body = {
        "project": "bedrock-global-quickstart",
        "monthlyBudgetUsd": 50,
        "currentSpendUsd": 7.25,
        "status": "OK",
        "recommendation": "当前预算正常；继续保留日志 7 天，并在完成 RAG 实验后删除 OpenSearch Serverless collection。"
    }
    return {
        "messageVersion": "1.0",
        "response": {
            "actionGroup": event.get("actionGroup", "BudgetTools"),
            "apiPath": event.get("apiPath", "/budget/status"),
            "httpMethod": event.get("httpMethod", "GET"),
            "httpStatusCode": 200,
            "responseBody": {
                "application/json": {
                    "body": json.dumps(response_body, ensure_ascii=False)
                }
            }
        }
    }
PY

cd tmp
zip tool_lambda.zip tool_lambda.py
cd -
```

创建 Lambda execution role：

```bash
cat > tmp/tool-lambda-trust.json <<'EOF'
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
  --role-name ${TOOL_FUNCTION_NAME}-role \
  --assume-role-policy-document file://tmp/tool-lambda-trust.json

aws iam attach-role-policy \
  --role-name ${TOOL_FUNCTION_NAME}-role \
  --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

export TOOL_LAMBDA_ROLE_ARN=arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:role/${TOOL_FUNCTION_NAME}-role

aws lambda create-function \
  --function-name ${TOOL_FUNCTION_NAME} \
  --runtime python3.12 \
  --role ${TOOL_LAMBDA_ROLE_ARN} \
  --handler tool_lambda.lambda_handler \
  --zip-file fileb://tmp/tool_lambda.zip \
  --timeout 20 \
  --region ${AWS_REGION}

aws lambda wait function-active --function-name ${TOOL_FUNCTION_NAME} --region ${AWS_REGION}
export TOOL_FUNCTION_ARN=arn:${AWS_PARTITION}:lambda:${AWS_REGION}:${ACCOUNT_ID}:function:${TOOL_FUNCTION_NAME}
```

### 3. 创建 Agent

```bash
aws bedrock-agent create-agent \
  --agent-name ${AGENT_NAME} \
  --foundation-model ${BEDROCK_TEXT_MODEL_ID} \
  --instruction "你是企业内部政策助手。用户询问预算或成本状态时，必须调用预算工具。回答必须简洁，并说明是否需要清理资源。" \
  --agent-resource-role-arn ${AGENT_ROLE_ARN} \
  --region ${AWS_REGION} \
  > tmp/agent-create.json

export AGENT_ID=$(jq -r '.agent.agentId' tmp/agent-create.json)
```

### 4. 允许 Bedrock 调用 Lambda

```bash
aws lambda add-permission \
  --function-name ${TOOL_FUNCTION_NAME} \
  --statement-id allow-bedrock-agent \
  --action lambda:InvokeFunction \
  --principal bedrock.amazonaws.com \
  --source-arn arn:${AWS_PARTITION}:bedrock:${AWS_REGION}:${ACCOUNT_ID}:agent/${AGENT_ID} \
  --region ${AWS_REGION}
```

### 5. 定义 OpenAPI schema

```bash
cat > tmp/budget-openapi.json <<'EOF'
{
  "openapi": "3.0.0",
  "info": {
    "title": "Budget status API",
    "version": "1.0.0"
  },
  "paths": {
    "/budget/status": {
      "get": {
        "description": "Get current budget status for the Bedrock QuickStart project.",
        "operationId": "getBudgetStatus",
        "responses": {
          "200": {
            "description": "Budget status",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "project": {"type": "string"},
                    "monthlyBudgetUsd": {"type": "number"},
                    "currentSpendUsd": {"type": "number"},
                    "status": {"type": "string"},
                    "recommendation": {"type": "string"}
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
EOF
```

### 6. 创建 Action Group

```bash
aws bedrock-agent create-agent-action-group \
  --agent-id ${AGENT_ID} \
  --agent-version DRAFT \
  --action-group-name BudgetTools \
  --description "Budget and cost status tools" \
  --action-group-executor lambda=${TOOL_FUNCTION_ARN} \
  --api-schema payload="$(cat tmp/budget-openapi.json)" \
  --region ${AWS_REGION} \
  > tmp/action-group-create.json
```

如果当前 CLI 不接受 `payload="$(cat ...)"`，改用当前版本支持的 `s3` 或 `payload` 结构，并记录偏离。

### 7. Prepare Agent 并创建 Alias

```bash
aws bedrock-agent prepare-agent \
  --agent-id ${AGENT_ID} \
  --region ${AWS_REGION}

aws bedrock-agent create-agent-alias \
  --agent-id ${AGENT_ID} \
  --agent-alias-name quickstart \
  --region ${AWS_REGION} \
  > tmp/agent-alias.json

export AGENT_ALIAS_ID=$(jq -r '.agentAlias.agentAliasId' tmp/agent-alias.json)
```

### 8. 调用 Agent

```bash
aws bedrock-agent-runtime invoke-agent \
  --agent-id ${AGENT_ID} \
  --agent-alias-id ${AGENT_ALIAS_ID} \
  --session-id quickstart-session-1 \
  --input-text "当前 Bedrock QuickStart 预算状态怎么样？需要清理什么资源？" \
  --region ${AWS_REGION} \
  > tmp/agent-invoke-output.json
```

> **已知限制：** 当前 AWS CLI 版本的 `bedrock-agent-runtime help` 中没有 `invoke-agent` 子命令（`Found invalid choice: 'invoke-agent'`）。CLI 不支持时，改用下面的 boto3 方法 —— `bedrock-agent-runtime` 的 boto3 客户端支持 `invoke_agent`，返回值是一个需要迭代的 `EventStream`（`chunk` / `trace` 等事件）。

**boto3 替代方法（已验证可用，并确认触发了 Lambda Action Group）：**

```bash
cat > tmp/agent_invoke.py <<PY
import json
import boto3

agent_id = "${AGENT_ID}"
agent_alias_id = "${AGENT_ALIAS_ID}"

client = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

response = client.invoke_agent(
    agentId=agent_id,
    agentAliasId=agent_alias_id,
    sessionId="quickstart-session-1",
    inputText="当前 Bedrock QuickStart 预算状态怎么样？需要清理什么资源？",
    enableTrace=True,
)

final_text_parts = []
lambda_evidence = []
for event in response["completion"]:
    key = next(iter(event.keys()))
    if key == "chunk" and "bytes" in event["chunk"]:
        final_text_parts.append(event["chunk"]["bytes"].decode("utf-8"))
    elif key == "trace":
        trace = event["trace"].get("trace", {})
        trace_str = json.dumps(trace, default=str)
        if "BudgetTools" in trace_str or "getBudgetStatus" in trace_str or "actionGroupInvocationOutput" in trace_str:
            lambda_evidence.append(trace)

print("final text:", "".join(final_text_parts))
print("lambda action-group trace evidence count:", len(lambda_evidence))
PY

python3 tmp/agent_invoke.py
```

用 `trace` 事件中出现 `BudgetTools` / `getBudgetStatus` / `actionGroupInvocationOutput` 来确认 Lambda Action Group 被真正调用；也可以用 `aws logs filter-log-events --log-group-name /aws/lambda/${TOOL_FUNCTION_NAME}` 核对 Lambda 的调用时间戳是否落在本次 invoke 调用的时间窗口内，作为独立证据。

---

## 验收标准

- Lambda 工具可独立 invoke
- Agent 创建成功并 prepare
- Action Group 创建成功
- Agent runtime 调用触发 Lambda
- 最终回答包含预算状态和清理建议

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|----------|----------|
| 1 | `python3 tmp/agent_invoke.py 2>&1 \| grep "lambda action-group trace evidence count:"` | 数字大于 `0` |
| 2 | `aws logs filter-log-events --log-group-name /aws/lambda/${TOOL_FUNCTION_NAME} --region ${AWS_REGION} --query 'events[-1].message' --output text` | 能查到近期一条调用记录，时间戳落在本次 invoke 窗口内 |
| 3 | `python3 tmp/agent_invoke.py 2>&1 \| grep "final text:"` | 输出中包含预算状态与清理建议 |

## 常见失败与处理

| 现象 | 原因 | 处理 |
|------|------|------|
| Action Group 创建失败 | OpenAPI schema 或 CLI 参数格式不匹配 | 用当前 CLI skeleton 校验 schema |
| Lambda 未被调用 | Agent instruction 不明确或 permission 缺失 | 检查 `lambda add-permission` 和 trace |
| Agent prepare 失败 | role 权限不足 | 检查 `bedrock:InvokeModel` 和 `lambda:InvokeFunction` |
| invoke-agent 输出难解析 | event stream 格式 | 保存原始 JSON，按 `chunk.bytes` 或 trace 字段解析 |
| CLI 无 `invoke-agent` 子命令 | Agent Runtime CLI schema 未暴露该命令 | 改用 boto3 `bedrock-agent-runtime` 客户端的 `invoke_agent`，迭代返回的 `EventStream` |

## 实验总结

本实验创建了一个最小 Bedrock Agent，验证了 Agent 判断调用工具、触发 Lambda Action Group、把工具结果整合进最终回答的完整调用链。整条链路依赖两层授权拼接：Agent 执行角色允许它调用模型和 Lambda，Lambda 的 resource policy 反过来允许 Bedrock 服务调用它。用 `trace` 事件确认 Lambda 被真正触发（而非模型编造答案），是验证 Agent 应用正确性的关键手段，也是本系列中最接近生产级 Agent 应用架构的 Demo。

## 清理

```bash
aws bedrock-agent delete-agent-alias --agent-id ${AGENT_ID} --agent-alias-id ${AGENT_ALIAS_ID} --region ${AWS_REGION}
aws bedrock-agent delete-agent --agent-id ${AGENT_ID} --skip-resource-in-use-check --region ${AWS_REGION}

aws lambda remove-permission --function-name ${TOOL_FUNCTION_NAME} --statement-id allow-bedrock-agent --region ${AWS_REGION} || true
aws lambda delete-function --function-name ${TOOL_FUNCTION_NAME} --region ${AWS_REGION}

aws iam delete-role-policy --role-name ${AGENT_ROLE_NAME} --policy-name ${AGENT_ROLE_NAME}-policy
aws iam delete-role --role-name ${AGENT_ROLE_NAME}
aws iam detach-role-policy --role-name ${TOOL_FUNCTION_NAME}-role --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name ${TOOL_FUNCTION_NAME}-role

rm -f tmp/agent-*.json tmp/action-group-create.json tmp/budget-openapi.json tmp/tool_lambda.py tmp/tool_lambda.zip tmp/*trust.json tmp/*policy.json tmp/agent_invoke.py
```
