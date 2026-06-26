# hermes-agent 研究知识库索引

本 topic 针对 [`NousResearch/hermes-agent`](https://github.com/NousResearch/hermes-agent) 做代码分析，源码作为 submodule 放在 [`./hermes-agent`](./hermes-agent)。当前已完成 topic 初始化、权威参考资料索引、core report、memory system 专题报告和 skill system 专题报告。

## 仓库信息

- 上游仓库：<https://github.com/NousResearch/hermes-agent>
- submodule 路径：[`./hermes-agent`](./hermes-agent)
- 初始化分支：`main`
- 初始化 commit：`bb7ff7dc302cbcbe41cf6bc09424ffc9fb2d062f`
- 递归子模块：无
- 依赖初始化：未执行

## 文档列表

### [`hermes-agent-core-report.md`](./hermes-agent-core-report.md)

主研究报告，重点覆盖：

- CLI、messaging gateway、ACP、cron 等入口如何收敛到 `AIAgent.run_conversation`
- `AIAgent` 如何作为 facade，把初始化、turn prologue、provider loop、工具执行、prompt、session 和 memory/skills 拆到 `agent/` 内部模块
- system prompt 的 `stable / context / volatile` 分层，以及为什么 Hermes 尽量保持 prompt cache prefix 稳定
- provider runtime 如何解析 `chat_completions`、`codex_responses`、`anthropic_messages` 等 API mode
- 工具系统如何通过 registry、toolset、availability、agent-level interception、middleware 和并发执行组成行动能力
- SQLite session store、FTS5 search、memory、skills、delegation、context compression 如何支撑跨 turn 持续性

适合先读，用来建立 Hermes Agent 的主干执行链路和源码地图。

### [`hermes-agent-memory-system.md`](./hermes-agent-memory-system.md)

记忆系统专题报告，重点覆盖：

- Hermes memory 的六条通道：内建 curated memory、provider static block、provider dynamic recall、tool result、post-turn sync 和 SQLite transcript archive
- 各类 memory 在 prompt / user message / tool result / external backend / SessionDB 中的位置，以及何时进入模型上下文
- 内建 `MEMORY.md` / `USER.md` 的冻结 system prompt snapshot、读写时机、容量限制、漂移防护和注入安全扫描
- built-in `memory` tool 相关 prompt 路径：`MEMORY_GUIDANCE`、`MEMORY_SCHEMA`、background review、onboarding profile build、clarify 和 write approval
- `MemoryProvider` / `MemoryManager` 外部 provider 生命周期：static prompt block、turn-start prefetch、post-turn sync、session hooks、provider tools 和内建写入镜像
- 8 个 bundled external provider 的工具清单、自动 recall/写入机制、生成算法取向和优缺点
- Honcho provider 的 peer/session/user/AI 建模、`context/tools/hybrid` recall mode、base context + dialectic supplement 双层自动注入
- 后台 background review、`write_approval`、pending store 如何让主动记忆具备治理边界
- `session_search` / SQLite FTS5 作为 transcript archive recall，与 memory provider 的分工

### [`hermes-agent-skill-system.md`](./hermes-agent-skill-system.md)

技能系统专题报告，重点覆盖：

- skill 作为 procedural memory 的目录包模型，以及 `SKILL.md`、`references/`、`templates/`、`scripts/`、`assets/` 的分工
- skill 在 system prompt 中的索引位置、全文按需加载方式，以及 `skill_view` / `/skill-name` / `--skills` 的渐进披露机制
- 任务执行中如何通过 `skill_manage(create|patch|write_file|edit)` 自动创建、迭代和修补 skill
- `background_review` 如何从一次对话中提炼可复用 procedure，并优先更新现有 skill 或 umbrella skill
- `curator` 如何对 agent-created skill 做长期维护、收敛、归档和反熵治理
- `skill_usage` / `skill_provenance` 如何记录 skill 的使用、来源和生命周期状态

适合先读，用来建立 Hermes Agent 如何把“经验”外化为 skill 并持续自我改进的主线。

## 参考资料

- [`ref/INDEX.md`](./ref/INDEX.md)：官方文档、上游仓库和源码内权威资料入口。

## 后续分析入口

后续开始分析时，可优先补充：

1. `hermes-agent-tool-runtime.md`：工具 registry、toolsets、approval、MCP、terminal backend 与 managed gateway 专题。
2. `hermes-agent-gateway-session.md`：gateway session key、agent cache、platform callbacks、reset/resume 专题。
