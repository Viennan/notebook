# Codex 研究知识库索引

本 topic 针对 [`openai/codex`](https://github.com/openai/codex) 做代码分析。该仓库是 OpenAI Codex CLI、SDK 与 app-server 等开源组件的主要源码入口；分析以本 topic 内的源码 submodule 为准，官方文档用于补充产品语境和用户可见行为。

## 仓库信息

- 上游仓库：<https://github.com/openai/codex>
- submodule 路径：[`./codex`](./codex)
- 初始化分支：`main`
- 初始化 commit：`6d2168f06ae275d5e1f73cabf935d2bcc8549998`
- 递归子模块：无
- 依赖初始化：未执行
- 技术栈线索：Rust workspace（`codex-rs/Cargo.toml`、`codex-rs/Cargo.lock`）、pnpm workspace（`package.json`、`pnpm-lock.yaml`）、Python SDK/runtime（`sdk/python/pyproject.toml`、`sdk/python-runtime/pyproject.toml`）

## 初始化状态

当前已完成：

- 创建分析目录和权威参考资料索引。
- 引入源码 submodule。
- 记录本地 submodule commit、分支和递归子模块状态。
- 将 `notes/code-ana_codex/codex/**` 加入 `.vscode/settings.json` 的 `files.watcherExclude`。

当前暂不初始化依赖。后续若为了类型跳转或本地构建需要依赖，应优先使用冻结、只读、不开安装脚本的命令，例如在源码目录中执行 `pnpm install --frozen-lockfile --ignore-scripts`，或在 `codex-rs/` 中执行 `cargo fetch --locked`。

## 文档列表

### [`codex-agent-core-report.md`](./codex-agent-core-report.md)

Codex 交互式 agent core 主链路报告。覆盖默认 `codex` 入口到 TUI/app-server、`thread/start` 与 `turn/start`、core SQ/EQ、`submission_loop`、`RegularTask`/`run_turn`、Responses stream、工具执行、exec policy、approval/sandbox、以及 core event 回传到 TUI 的端到端链路。

### [`ref/INDEX.md`](./ref/INDEX.md)

官方或权威资料入口。当前记录上游仓库、官方 Codex 文档、Codex CLI 文档、开源组件说明、README、贡献与安装资料等，供后续分析按需加载。

## 后续分析入口

源码引入后，可优先补充：

1. `codex-rs-architecture.md`：`codex-rs` workspace、core/TUI/CLI/app-server/protocol 等 crate 的职责边界。
2. `codex-tool-runtime.md`：shell、patch、file read/write、MCP、browser/app 等工具能力如何注册、审批、执行和回传。
3. `codex-customization-surface.md`：`AGENTS.md`、skills、plugins、MCP、hooks、config 与 memory 在运行时上下文中的位置。
4. `codex-sandbox-approvals.md`：sandbox、approval policy、权限提示、网络限制和安全边界的实现。
