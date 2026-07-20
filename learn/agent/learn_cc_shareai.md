## s01 Agent循环
消息历史不是聊天记录展示层，而是模型下一轮要读的工作上下文；
message在叠加，100次对话后的message还会保留多少第一次对话的content？

## s02 工具使用
工具能力靠一层清晰的路由面增长；
一个工具，一个handler（函数），工具注册进dispatch map，在发送请求的时候给LLM，由模型决定调用工具；
```
TOOLS = [    {"name": "bash", "description": "Run a shell command.",     "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}}]
```
定义为**schema**，很像Skills，要着重区分哦！

## s03  代办写入
防止模型在一次多步骤任务会话中产生漂移；
添加todo作为模型计划的入口（维护任务表），用active step来防止计划失焦，‘提醒’负责告知模型更新计划（状态）；

## s04 子智能体
不是为了多一个角色，而是多一个干净的上下文，聚焦于局部任务；

给父智能体一个task工具，让模型能够主动创建子智能体；
子智能体要用自己的消息列表，实现父子上下文隔离；
只给子智能体必要的工具，不给子智能体派生子智能体的能力，防止无限递归；
带回父智能体的只有最终结果或者总结；

子智能体根据任务，从父智能体的上下文中fork必要的消息历史，使其更好完成局部任务；

## s05 技能系统
skill是一份围绕某类任务可复用的说明书=使用场景+固定步骤+注意事项；
skill系统的核心，不是“多一个工具”，而是“把可选知识从常驻prompt中拆出来，改成按需加载”；

## s06 上下文压缩
上下文压缩的核心，不是删历史，而是移走细节，让模型在更短的活跃上下文中，仍然保住继续工作的连续性；
移走细节通过大结果落盘，旧结果压缩，整体过长的摘要来实现；
为了保证连续性，摘要一定要保留；
当前目标任务，已完成的关键动作，已修改或重点查看过的文件，关键决定与约束，未完成事项；

## s07 权限系统——**模型的意图，先过安全闸门，再行动**
权限系统不是为了让agent更笨，而是为了让agent的行动先经过一次可靠的安全判断；

tool_call->deny rules->mode check->allow roles->ask user->tool_run；
先deny危险操作（sudo、rm、越界路劲、命令替换、重定向、shell元字符拼接）；
再根据**权限模式**（mode，default、plan、auto）决定**权限结果**（behavior）；
若该工具的执行命令符合allow rules，通过；
否则，behavior = ask，去询问用户；
behavior = allow，该次工具才可执行；

bash工具应该被单独审查，它不是普通文本，而是可执行动作描述（权限太大，危险）；

## s08 Hook系统
Hook让系统围绕主循环生长，而不是不断重写主循环本身；

请注意区分tool、子agent、skill；-hook包含tool，比子agent代价更小，比skill更灵活；
最大区别，**hook不要求主循环理解每个拓展需求；**（不作为上下文进入LLM）
主循环知道事件名，hook runner知道怎么调拓展逻辑；
hook最少包含事件名、payload（tool_name、input）、返回结果；

## s09 记忆系统
只有跨会话，无法从当前工作重新推导的知识，才值得进入memory；

四类基础memory；
user，用户偏好，代码风格、回答风格、偏好工具链；
feedback，用户反馈，纠正错误、鼓励执行顺序、频繁错误；
project，不易从代码中重新看出的项目约定或背景，项目合规设计、短期不能动的旧目录、故意旧规则；
reference，外部资源指针，资料库url、问题单看板位置、监控面板位置；

不可存进memory的边界；
可通过读代码获取内容，如文件结构、函数签名、目录布局；
task/plan，如任务进度；
易过时内容，临时分支名；
代码细节；
有安全风险的密钥、凭证；

单条memory的frontmatter，name、description、**type**、content，用索引文件MEMORY.md快速确定可用memory；
memory存储长久内容，但非永恒规则，故**适合用来提供方向，而非替代当前观察**，即若遇代码冲突，优先**验证**后的真实状态；

memory也分作用域，private与team（project类与reference类记忆）；
让llm追问用户要求存储的内容，而非盲目存为memory；

## s10 系统提示词
系统提示词不是一整块大字符串，而是一条可维护的组装流水线，把不同来源的信息按清晰边界组装起来；

```text
core
+ tools
+ skills
+ memory
+ claude_md
+ dynamic_context
= final system prompt
```

务必理清system prompt、system reminder、最终模型输入三者的关系；
```
广义 prompt / 最终模型输入
├─ 主 system prompt：稳定规则、身份、工具、长期约束
├─ system reminder：当前轮临时提醒、动态状态
├─ user message
├─ tool result
└─ memory / attachment 等
```

## s11 错误恢复
错误不是例外，而是主循环必须预留出来的一条正常分支，让系统清楚自己此刻是在继续、重试、还是恢复流程；

先从三种错误开始：
```
1. 输出被截断
   模型还没说完，但 token 用完了

2. 上下文太长
   请求装不进模型窗口了

3. 临时连接失败
   网络、超时、限流、服务抖动
```


错误先分类，恢复并执行，最后才把错误结果反馈给用户；

## s12 任务系统
大目标拆成多个小任务，生成有排序、持久化的DAG任务图，是多agent协作的基础；
重点在于目标的持久化、进度的可恢复；

接到任务会有执行清单，但串行执行是不可靠的，agent执行子任务时，发现缺了东西，回去加，相关子任务全要修改，执行一下就全乱了；
所以任务要有先后，用DAG表现依赖，blockedBy实现；

请注意，执行清单保存在会话内存中，而任务系统中的每个任务都是一个json文件，任务之间通过blockedBy相互依赖，跨会话持久化在磁盘中，.tasks/目录下；

关于task_id生成规则，顺序id + HW（High Watermark，高水位标记，一个偏移值，维护数据一致性）文件；

### 必要流程：
创建任务（保存至.tasks/{id}.json、blockedBy声明依赖task_ids）-认领任务（校验处于pending状态、can_start依赖task皆completed，设置owner记录执行agent防止多agent场景重复认领）-完成与解锁任务（status = completed，扫描所有其他任务，找出解锁的下游任务）-认领任务（循环开始，直到所有任务completed）；

跨会话恢复任务时，get_task返回完整任务json（description、blockedBy），list_tasks只显示一行摘要；

任务teammate终止或shutdow，未完成任务unassign（清除owner），status = pending，其他agent认领；

