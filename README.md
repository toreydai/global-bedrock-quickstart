# Global Bedrock QuickStart
## Amazon Bedrock Hands-on Lab Collection

基于 Amazon Bedrock（Converse API / Knowledge Bases / Guardrails / Agents）· 全球区（us-east-1）· Prompt 驱动执行

---

## Demo 列表

本 QuickStart 以一个贯穿场景展开：**企业内部政策助手**。从最简单的模型调用开始，逐步加入结构化输出、流式 UI、文档理解、应用封装、安全护栏、RAG、工具调用、Agent、评估、吞吐和成本治理。

建议按难度分层学习：

| 难度 | 目标 | 适合对象 |
|------|------|----------|
| 基础 | 会调用模型并理解请求/响应结构 | 初次接触 Bedrock 的开发者 |
| 进阶 | 能把模型接入应用并加上权限、日志、安全护栏 | 应用工程师、平台工程师 |
| 生产化 | 能搭建 RAG / Agent / 评估 / 生产治理闭环 | 架构师、平台负责人、AI 应用负责人 |

## 基础：模型调用与输入输出

### 1. 基础环境

- [Demo01 — 准备实验环境与模型访问](docs/demo01-env-model-access.md)

### 2. Converse API 主线

- [Demo02 — 使用 Converse API 完成首次文本推理](docs/demo02-converse-text.md)
- [Demo03 — 多轮对话、系统提示词与推理参数](docs/demo03-chat-system-params.md)
- [Demo04 — ConverseStream 流式输出](docs/demo04-streaming.md)
- [Demo05 — 多模态输入：图片与文档理解](docs/demo05-multimodal-document.md)
- [Demo06 — Prompt 模板与结构化 JSON 输出](docs/demo06-prompt-structured-output.md)

## 进阶：封装、权限、安全与工具

### 3. 最小应用与运行治理

- [Demo07 — Lambda 与 API Gateway 封装最小 Bedrock 应用](docs/demo07-lambda-api-app.md)
- [Demo08 — 调用日志、CloudWatch 与 S3 归档](docs/demo08-invocation-logging.md)
- [Demo09 — IAM 最小权限与应用角色](docs/demo09-iam-least-privilege.md)

### 4. 安全护栏与工具调用

- [Demo10 — Guardrails 内容安全与敏感信息过滤](docs/demo10-guardrails.md)
- [Demo11 — Guardrails 深入：Denied Topics、PII、Grounding 验证](docs/demo11-guardrails-advanced.md)
- [Demo12 — Tool Use / Function Calling 工具调用](docs/demo12-tool-use.md)

## 生产化：RAG、Agent、评估、吞吐与成本

### 5. RAG 与 Agent

- [Demo13 — Knowledge Bases 构建最小 RAG Golden Path](docs/demo13-knowledge-base-rag.md)
- [Demo14 — Knowledge Bases 深入：Metadata Filter、Rerank 与 Citation 校验](docs/demo14-knowledge-base-advanced.md)
- [Demo15 — Agents for Bedrock + Lambda Action Group](docs/demo15-agents.md)

### 6. 可靠性、评估与成本

- [Demo16 — Cross-Region Inference、限流、重试与超时](docs/demo16-cross-region-retry.md)
- [Demo17 — Batch Inference、Prompt Caching 与吞吐优化](docs/demo17-batch-cache-throughput.md)
- [Demo18 — Prompt 管理、模型评估与成本清理审计](docs/demo18-prompt-eval-cost.md)

---

## 使用方式

1. 在此目录下打开 Claude Code，`CLAUDE.md` 自动加载全球区 Bedrock 配置
2. 将对应 [`docs/`](docs/) 目录中的 Demo 文件内容粘贴到对话框，由 AI 自主执行
3. 每个 Demo 末尾均有**清理**步骤，实验结束后执行以避免持续计费

> **当前状态**：本仓库是 Bedrock QuickStart 课程版骨架。基础/进阶 Demo 以可直接执行的 CLI / Python 命令为主；生产化部分涉及 Knowledge Bases、Agents、Batch、Prompt Caching、Inference Profile 等能力时，执行前必须先用当前 AWS CLI schema 和账号功能可用性确认参数，因为 Bedrock 模型访问、模型 ID、Region 支持、配额和功能开放状态会变化。

---

## 环境要求

| 工具 | 版本 |
|------|------|
| AWS CLI | 2.x |
| Python | 3.10+ |
| boto3 | latest |
| jq | latest |
| git | latest |

操作机建议使用 Amazon Linux 2023 EC2，绑定具备 Bedrock、IAM、S3、CloudWatch Logs、OpenSearch Serverless（Knowledge Bases Demo）、Lambda（Tool/Agent Demo）操作权限的 IAM Role。

## 命名与资源约定

| 项目 | 约定 |
|------|------|
| Region | `us-east-1` |
| AWS partition | `aws` |
| 项目 tag | `Project=bedrock-global-quickstart` |
| S3 桶 | `s3://bedrock-quickstart-${ACCOUNT_ID}-${AWS_REGION}/` |
| 调用日志组 | `/aws/bedrock/quickstart/invocations` |
| 默认模型变量 | `BEDROCK_TEXT_MODEL_ID`，执行时动态选择 |
| 默认 embedding 模型变量 | `BEDROCK_EMBED_MODEL_ID`，执行时动态选择 |

## License

MIT - see the [LICENSE](LICENSE) file for details.

## 免责声明

- 本 Workshop 仅供学习和测试用途。
- 执行过程中会调用模型并创建 AWS 资源，可能产生费用，请在实验完成后及时清理资源。
- 作者不对因使用本 Workshop 产生的任何费用或损失承担责任。
- 所有命令和配置仅作为示例参考，生产环境使用前请根据实际需求进行安全评估和调整。
