# Demo08 — 调用日志、CloudWatch 与 S3 归档

## 实验简介

启用 Bedrock 模型调用日志，把请求/响应元数据送到 CloudWatch Logs 或 S3，用于审计、排障和成本分析。

**实验目标：**
- 配置模型调用日志
- 发起一次带 `requestMetadata` 的调用
- 在 CloudWatch Logs 中查询调用记录

**预计 AI 执行时长：** 10-15 分钟

---

## 前提条件

- 已完成 Demo01、Demo02 和 Demo07
- 权限包含 Bedrock logging configuration、CloudWatch Logs、S3

```bash
export AWS_REGION=us-east-1
export BEDROCK_BUCKET=${BEDROCK_BUCKET:?set this from Demo01}
export BEDROCK_LOG_GROUP=${BEDROCK_LOG_GROUP:-/aws/bedrock/quickstart/invocations}
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
mkdir -p tmp
```

---

## 步骤

### 1. 确认日志目标

```bash
aws logs describe-log-groups \
  --log-group-name-prefix "${BEDROCK_LOG_GROUP}" \
  --region ${AWS_REGION}
```

### 2. 配置调用日志

根据当前 AWS CLI 版本支持的参数配置模型调用日志。优先使用 CloudWatch Logs，必要时增加 S3 归档。

```bash
aws bedrock put-model-invocation-logging-configuration \
  --region ${AWS_REGION} \
  --logging-config "cloudWatchConfig={logGroupName=${BEDROCK_LOG_GROUP},roleArn=<bedrock-logging-role-arn>},textDataDeliveryEnabled=true,embeddingDataDeliveryEnabled=true,imageDataDeliveryEnabled=false"
```

> 说明：如果命令提示需要服务角色，创建一个信任 `bedrock.amazonaws.com`、允许写入目标日志组的 IAM Role，再重试。**实测发现**：只给 `logs:CreateLogStream` + `logs:PutLogEvents` 且 Resource 只写 log-group 级 ARN（`arn:aws:logs:<region>:<account>:log-group:<name>`）第一次会校验失败；必须**同时**包含 log-group 级和 log-stream 级两种 ARN，并加上 `logs:DescribeLogStreams`，`put-model-invocation-logging-configuration` 才能一次成功。角色 inline policy 示例：
>
> ```json
> {
>   "Version": "2012-10-17",
>   "Statement": [
>     {
>       "Effect": "Allow",
>       "Action": [
>         "logs:CreateLogStream",
>         "logs:PutLogEvents",
>         "logs:DescribeLogStreams"
>       ],
>       "Resource": [
>         "arn:aws:logs:<region>:<account-id>:log-group:<log-group-name>",
>         "arn:aws:logs:<region>:<account-id>:log-group:<log-group-name>:log-stream:*"
>       ]
>     }
>   ]
> }
> ```

### 3. 发送带元数据的调用

```bash
cat > tmp/logged-converse.json <<'EOF'
{
  "messages": [
    {
      "role": "user",
      "content": [
        {"text": "返回一句话：Bedrock logging test"}
      ]
    }
  ],
  "requestMetadata": {
    "workshop": "global-bedrock-quickstart",
    "demo": "demo06"
  },
  "inferenceConfig": {
    "maxTokens": 100,
    "temperature": 0
  }
}
EOF

aws bedrock-runtime converse \
  --model-id "${BEDROCK_TEXT_MODEL_ID}" \
  --cli-input-json file://tmp/logged-converse.json \
  --region ${AWS_REGION} \
  > tmp/logged-converse-output.json
```

### 4. 查询日志

```bash
aws logs start-query \
  --log-group-name "${BEDROCK_LOG_GROUP}" \
  --start-time $(date -u -d '15 minutes ago' +%s) \
  --end-time $(date -u +%s) \
  --query-string 'fields @timestamp, @message | sort @timestamp desc | limit 20' \
  --region ${AWS_REGION}
```

---

## 验收标准

- 模型调用成功
- CloudWatch Logs 中能查到近期 Bedrock invocation 记录
- 日志中不得包含真实敏感数据

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|----------|----------|
| 1 | `aws bedrock get-model-invocation-logging-configuration --region ${AWS_REGION} --query 'loggingConfig.cloudWatchConfig.logGroupName' --output text` | `${BEDROCK_LOG_GROUP}` |
| 2 | `aws logs start-query --log-group-name "${BEDROCK_LOG_GROUP}" --start-time $(date -u -d '15 minutes ago' +%s) --end-time $(date -u +%s) --query-string 'fields @message \| filter @message like /demo06/' --region ${AWS_REGION}` | 返回的 query 结果中能找到本次调用的 `requestMetadata` |

## 实验总结

本实验打通了 Bedrock 模型调用的审计闭环：开启 CloudWatch Logs 调用日志后，请求/响应元数据（含自定义 `requestMetadata`）可在 CloudWatch Logs Insights 中检索，为生产环境的调用审计、异常排查和使用量分析提供了基础设施。需要注意日志内容可能包含用户输入原文，涉及敏感信息的场景应评估是否需要脱敏或关闭 `textDataDeliveryEnabled`。

## 清理

如后续不再需要调用日志：

```bash
aws bedrock delete-model-invocation-logging-configuration --region ${AWS_REGION}
```

按需删除日志组和 S3 对象：

```bash
aws logs delete-log-group --log-group-name "${BEDROCK_LOG_GROUP}" --region ${AWS_REGION}
```
