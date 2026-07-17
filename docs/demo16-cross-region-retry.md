# Demo16 — Cross-Region Inference、限流、重试与超时

## 实验简介

验证 Bedrock 应用在生产环境中的可靠性基础：跨区域推理、模型限流、客户端重试、超时和降级。

**实验目标：**
- 理解 inference profile / cross-region inference 的使用场景
- 识别 `ThrottlingException`、`ModelTimeoutException`、`ServiceUnavailableException`
- 在 boto3 客户端配置重试和超时
- 设计应用降级策略

**预计 AI 执行时长：** 20-30 分钟

---

## 前提条件

- 已完成 Demo07
- 当前账号支持目标模型或 inference profile

```bash
export AWS_REGION=us-east-1
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
mkdir -p tmp
```

---

## 步骤

### 1. 查询 inference profiles

```bash
aws bedrock list-inference-profiles \
  --region ${AWS_REGION} \
  --query 'inferenceProfileSummaries[].{Name:inferenceProfileName,Id:inferenceProfileId,Status:status,Models:models}' \
  --output table
```

如果有可用 profile，设置：

```bash
export BEDROCK_INFERENCE_PROFILE_ID=<available-inference-profile-id>
```

### 2. boto3 重试与超时样例

```bash
cat > tmp/retry_client.py <<'PY'
import os
import time

import boto3
from botocore.config import Config
from botocore.exceptions import ClientError, ReadTimeoutError

config = Config(
    retries={"max_attempts": 5, "mode": "standard"},
    connect_timeout=5,
    read_timeout=60,
)

client = boto3.client("bedrock-runtime", region_name=os.environ["AWS_REGION"], config=config)
model_id = os.environ.get("BEDROCK_INFERENCE_PROFILE_ID") or os.environ["BEDROCK_TEXT_MODEL_ID"]

start = time.time()
try:
    response = client.converse(
        modelId=model_id,
        messages=[{"role": "user", "content": [{"text": "用一句话说明为什么生产应用需要 Bedrock 重试策略。"}]}],
        inferenceConfig={"maxTokens": 200, "temperature": 0},
        requestMetadata={"demo": "demo16"},
    )
    print(response["output"]["message"]["content"][0]["text"])
    print({"latencySeconds": round(time.time() - start, 3), "usage": response.get("usage")})
except (ClientError, ReadTimeoutError) as exc:
    print({"error": type(exc).__name__, "detail": str(exc)})
    raise
PY

python3 tmp/retry_client.py
```

### 3. 限流与降级策略

记录应用策略：

| 场景 | 推荐处理 |
|------|----------|
| `ThrottlingException` | 指数退避重试，限制并发，必要时切换 inference profile |
| `ModelTimeoutException` | 缩短 prompt / maxTokens，异步处理长任务 |
| `ValidationException` | 不重试，修复模型 ID、schema 或输入 |
| `AccessDeniedException` | 不重试，检查模型访问和 IAM |
| 长文本任务 | 使用 batch 或队列异步化 |

---

## 验收标准

- 能列出当前账号的 inference profiles 或明确记录不支持
- Python 样例成功调用模型
- 客户端包含明确重试和超时配置
- 形成异常分类处理表

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|----------|----------|
| 1 | `python3 tmp/retry_client.py` | 打印模型回答文本和 `{'latencySeconds': ..., 'usage': ...}` |
| 2 | `python3 -c "from botocore.config import Config; c=Config(retries={'max_attempts':5,'mode':'standard'}); print(c.retries)"` | 输出确认 `max_attempts=5`、`mode=standard` |

## 实验总结

本实验验证了生产级 Bedrock 客户端应具备的可靠性基础：查询 inference profile 了解跨区域推理能力、在 boto3 客户端配置显式重试和超时（而非依赖 SDK 默认值）、并对不同异常类型（限流可重试、校验类不可重试）形成分类处理表。这些配置是 Demo07 那类应用封装在真正上生产前必须补齐的健壮性细节。

## 清理

```bash
rm -f tmp/retry_client.py
```
