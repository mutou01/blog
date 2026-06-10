# GitHub 热点日报 2026-05-22

## 今日重点推荐

### 1. LangGraph：Agent 工作流继续补齐“可恢复执行”细节

- **项目名**：[langchain-ai/langgraph](https://github.com/langchain-ai/langgraph)
- **发生了什么**：[LangGraph 1.2.1](https://github.com/langchain-ai/langgraph/releases) 于 5 月 21 日发布；近期 1.2.0 还补进 durable error-handler resume、StateGraph 默认节点配置、checkpoint delta snapshot 等能力。
- **为什么值得关注**：这不是“又一个 Agent demo”，而是在补长流程 Agent 最容易出事故的地方：工具结果污染消息、宿主崩溃后的恢复、checkpoint 体积与频率控制。
- **我可以从中学到什么**：把 Agent 当作有状态工作流系统设计，核心不是会不会调用模型，而是节点状态、错误恢复、消息隔离、checkpoint 策略和可观测事件能否形成稳定闭环。

### 2. LiteLLM：模型网关进入“供应链 + 权限 + MCP 路由”阶段

- **项目名**：[BerriAI/litellm](https://github.com/BerriAI/litellm)
- **发生了什么**：[LiteLLM v1.85.1](https://github.com/BerriAI/litellm/releases) 于 5 月 21 日发布，发布页强调所有 Docker 镜像使用 cosign 签名校验；近期 dev 版还继续推进 proxy 权限、Responses bridge cache、OTel span、Bedrock/DeepSeek/Gemini 适配。
- **为什么值得关注**：企业 Agent 系统接入多个模型、向量存储和工具网关后，LiteLLM 这类组件会变成“模型控制面”，发布重点也从 API 适配转向镜像可信、权限边界、审计与成本治理。
- **我可以从中学到什么**：模型网关要分清推理面和管理面；MCP/向量接口也要继承租户、团队、对象级权限；生产镜像签名和成本归因应当是基础能力。

### 3. vLLM：推理服务继续向大规模、工具调用和 Reasoning 模型靠拢

- **项目名**：[vllm-project/vllm](https://github.com/vllm-project/vllm)
- **发生了什么**：[vLLM v0.21.0](https://github.com/vllm-project/vllm/releases) 近期发布，重点包括 KV cache transfer、HMA/KV offload、OpenAI compatibility、Responses API 流式工具调用、reasoning item 修复和多硬件后端优化。
- **为什么值得关注**：Agent 系统上线后，瓶颈常常从“会不会调用模型”转为“推理吞吐、KV cache、并发工具调用、reasoning 模型预算和 OpenAI 兼容接口是否稳定”。
- **我可以从中学到什么**：推理服务不是单纯起一个 OpenAI-compatible server；要把 KV offload、调度、spec decode、tool parser、Responses API 兼容、镜像体积和硬件后端一起纳入容量规划。

### 4. Open Multi-Agent：TypeScript 后端里的“目标到任务 DAG”轻量编排

- **项目名**：[open-multi-agent/open-multi-agent](https://github.com/open-multi-agent/open-multi-agent)
- **发生了什么**：[v1.4.1](https://github.com/open-multi-agent/open-multi-agent/releases) 于 5 月 18 日发布，修复 OpenAI-family adapter 空 choices 崩溃、token budget 命中时孤立 `tool_use` block 的问题，并新增可选 Vercel AI SDK adapter；v1.4.0 还引入 PlanOnly、reasoning block 保留和 shared memory TTL。
- **为什么值得关注**：它把“目标 -> 动态任务 DAG -> 并行执行 -> 汇总结果”做成 TypeScript-native 库，适合 Node.js 后端团队研究轻量多 Agent 编排。
- **我可以从中学到什么**：动态编排要提供 PlanOnly、预算上限、任务依赖、并行 fan-out、shared memory TTL、MCP 工具接入和运行轨迹；这些控制面比“多几个 Agent 角色名”更重要。

### 5. MCP Registry：MCP 生态开始需要“注册中心级”的治理能力

- **项目名**：[modelcontextprotocol/registry](https://github.com/modelcontextprotocol/registry)
- **发生了什么**：[MCP Registry v1.7.9](https://github.com/modelcontextprotocol/registry/releases) 于 5 月 12 日发布，包含发布 CLI、SBOM、sigstore 资产，并继续修复验证器、发布认证文档和依赖安全。
- **为什么值得关注**：MCP server 数量增长后，真正的问题会变成发现、命名空间归属、版本可信、发布认证和客户端选择。Registry 说明 MCP 已经从“协议热度”进入“生态治理”阶段。
- **我可以从中学到什么**：做内部 MCP 市场时，不要只维护一个链接列表；要设计 namespace ownership、发布者身份、schema 校验、审计、兼容版本和废弃策略。

## 今日破圈高 Star 项目

### 1. Supabase：Postgres 平台继续向企业权限与 Agent 友好文档演进

- **项目名**：[supabase/supabase](https://github.com/supabase/supabase)
- **发生了什么**：[Supabase 5 月开发者更新](https://supabase.com/changelog/45702-developer-update-may-2026) 包含自定义 OAuth/OIDC provider、public schema 新表不再自动暴露到 Data API、`@supabase/server` SDK、Data API 暴露开关、Markdown/RSS changelog，以及 Supabase Agent Skills。
- **为什么值得关注**：它虽然是破圈高 star 项目，但这轮更新对后端和 Agent 系统很直接：默认最小权限、跨运行时服务端 auth、数据库 API 暴露治理、文档转 Markdown 和面向 AI coding agent 的技能包，都是 Agent 落地会遇到的基础设施问题。
- **我可以从中学到什么**：数据库平台给 Agent 使用时，默认公开 API 是风险；应当显式授权、显式暴露、显式审计。把产品文档变成 Agent 可消费的 Markdown/skills，也正在成为开发者平台的新标配。

### 2. Immich：高 star 应用项目里的后端工程样板

- **项目名**：[immich-app/immich](https://github.com/immich-app/immich)
- **发生了什么**：[Immich v2.6.0](https://github.com/immich-app/immich/releases) release 包含 350+ commits，重点包括服务端 deduplication 逻辑迁移、元数据同步、`immich-admin schema-check`、OAuth idToken claims、移动端原生 HTTP client、搜索和媒体浏览性能改进。
- **为什么值得关注**：它不是 Agent 框架，但非常适合后端开发者学习真实产品如何处理大规模媒体、后台任务、数据库 schema 校验、权限修复、客户端/服务端职责迁移和重大版本前的兼容管理。
- **我可以从中学到什么**：复杂应用的后端演进通常不是重写，而是把逻辑从客户端迁回服务端、补 schema 健康检查、收紧共享权限、优化队列与查询，再用清晰 release note 管理用户预期。

## 其他值得扫一眼

- [microsoft/markitdown](https://github.com/microsoft/markitdown)：文档转 Markdown 工具，RAG/Agent ingestion 前处理值得参考，尤其是它对不可信输入和 URI 访问权限的安全提醒。
- [qdrant/qdrant](https://github.com/qdrant/qdrant)：向量数据库方向继续关注 snapshot 恢复、key rotation、独立 metrics 端口和集群可观测性，而不只是 ANN 搜索性能。
- [weaviate/weaviate](https://github.com/weaviate/weaviate)：近期 release 重点在 RAFT、backup、vector cache 和 export 修复，适合关注向量数据库生产稳定性的团队扫一眼。
- [langfuse/langfuse](https://github.com/langfuse/langfuse)：LLM observability/evals/prompt 管理仍是 Agent 落地基础设施，和 LiteLLM、OpenTelemetry、LangChain 生态集成值得持续跟踪。
- [modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk)：官方 MCP SDK 生态仍在快速增长；如果你做内部工具接入，优先跟官方 SDK 的接口与安全模型。
