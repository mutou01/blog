一个agent产品，没有Harness，就只剩一个模型。
模型其实有很多种，AlphaGo、OpenAI Five、AlphaStar、腾讯 绝悟...
而如今我们口头常说的模型，一般指向LLM，大语言模型，其带来的影响是直观且深刻的。
文本所描述的agent产品，指的也是LLM+Harness组合的产物。
关于LLM，本文并不展开，它是核心，但门槛更高，而本文关注更容易落地的，Harness。

Harness，翻译为马具，人是骑手，通过马具驾驭马匹，也就是AI，去到达终点。
没有马具也能跑，但跑往那个方向，跑多快，新手能不能骑，会不会把人颠下来，这些问题，马具来解决。
所以Harness要做的，是以下五个部分。

```
Harness = Tools + Knowledge + Observation + Action Interfaces + Permissions

    Tools:          文件读写、Shell、网络、数据库、浏览器
    Knowledge:      产品文档、领域资料、API 规范、风格指南
    Observation:    git diff、错误日志、浏览器状态、传感器数据
    Action:         CLI 命令、API 调用、UI 交互
    Permissions:    沙箱隔离、审批流程、信任边界
```
当我们要做一个agent产品，也就需要从以下五个角度，去实现一个Harness。

- **实现工具。** 给 agent 一双手。文件读写、Shell 执行、API 调用、浏览器控制、数据库查询。每个工具都是 agent 在环境中可以采取的一个行动。设计它们时要原子化、可组合、描述清晰。
- **策划知识。** 给 agent 领域专长。产品文档、架构决策记录、风格指南、合规要求。按需加载（s05），不要前置塞入。Agent 应该知道有什么可用，然后自己拉取所需。
- **管理上下文。** 给 agent 干净的记忆。子 agent 隔离（s04）防止噪声泄露。上下文压缩（s06）防止历史淹没。任务系统（s07）让目标持久化到单次对话之外。
- **控制权限。** 给 agent 边界。沙箱化文件访问。对破坏性操作要求审批。在 agent 和外部系统之间实施信任边界。这是安全工程与 harness 工程的交汇点。
- **收集任务过程数据。** Agent 在你的 harness 中执行的每一条行动序列都是训练信号。真实部署中的感知-推理-行动轨迹是微调下一代 agent 模型的原材料。你的 harness 不仅服务于 agent -- 它还可以帮助进化 agent。


参考文章：
[learn-claude-code/README-zh.md at main · shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code/blob/main/README-zh.md)