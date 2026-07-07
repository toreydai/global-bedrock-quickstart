# Demo09 — IAM 最小权限与应用角色

## 实验简介

为一个示例应用创建只允许调用指定模型的 IAM Role / Policy，验证 Bedrock Runtime 最小权限。

**实验目标：**
- 理解 Bedrock control plane 与 runtime 权限差异
- 创建最小化 `bedrock:InvokeModel` / `bedrock:InvokeModelWithResponseStream` policy
- 验证允许模型可调用，未授权模型被拒绝

**预计 AI 执行时长：** 10-15 分钟

---

## 前提条件

- 已完成 Demo02 和 Demo07
- 权限包含 IAM 创建/附加策略

```bash
export AWS_REGION=us-east-1
export AWS_PARTITION=aws
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BEDROCK_TEXT_MODEL_ID=${BEDROCK_TEXT_MODEL_ID:?set this from Demo01}
export APP_ROLE_NAME=BedrockQuickStartAppRole
export APP_POLICY_NAME=BedrockQuickStartInvokePolicy
mkdir -p tmp
```

---

## 步骤

### 1. 创建权限策略

```bash
cat > tmp/bedrock-app-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:${AWS_PARTITION}:bedrock:${AWS_REGION}::foundation-model/${BEDROCK_TEXT_MODEL_ID}"
      ]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ${APP_POLICY_NAME} \
  --policy-document file://tmp/bedrock-app-policy.json
```

### 2. 创建示例角色

按操作机类型选择 trust policy。EC2 操作机可使用：

```bash
cat > tmp/ec2-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ${APP_ROLE_NAME} \
  --assume-role-policy-document file://tmp/ec2-trust.json

aws iam attach-role-policy \
  --role-name ${APP_ROLE_NAME} \
  --policy-arn arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/${APP_POLICY_NAME}
```

### 3. 验证策略内容

```bash
aws iam get-policy-version \
  --policy-arn arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/${APP_POLICY_NAME} \
  --version-id v1 \
  --query 'PolicyVersion.Document'
```

---

## 验收标准

- Policy 只包含 runtime 调用权限
- Resource 限定到指定 foundation model ARN
- Role trust principal 与实际计算环境一致

## 清理

```bash
aws iam detach-role-policy \
  --role-name ${APP_ROLE_NAME} \
  --policy-arn arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/${APP_POLICY_NAME}
aws iam delete-role --role-name ${APP_ROLE_NAME}
aws iam delete-policy --policy-arn arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/${APP_POLICY_NAME}
rm -f tmp/bedrock-app-policy.json tmp/ec2-trust.json
```
