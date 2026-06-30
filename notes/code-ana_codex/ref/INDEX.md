# Codex 参考资料索引

本索引用于集中记录 `openai/codex` 代码分析所需的官方或权威资料。分析结论以本 topic 内 submodule 的源码 commit 为准；在线资料用于补充产品语境、用户可见行为、安装方式和贡献约束。

## 官方入口

- [openai/codex GitHub 仓库](https://github.com/openai/codex)：上游源码、issue、discussion、release 与仓库 README 入口。
- [Codex 官方文档](https://developers.openai.com/codex)：OpenAI Codex 产品、CLI、IDE、app、cloud、配置、安全和自定义能力的官方文档入口。
- [Codex CLI 文档](https://developers.openai.com/codex/cli)：本地终端 Codex CLI 的安装、使用和行为说明。
- [Codex 开源组件说明](https://developers.openai.com/codex/open-source)：官方列出的开源组件边界；其中 `openai/codex` 是 Codex CLI、SDK 和 app-server 的主要源码位置。
- [Codex Subagents](https://developers.openai.com/codex/subagents)：官方 subagent workflow、可用性、管理方式、自定义 agents、`[agents]` 配置和示例入口。
- [Codex Subagent concepts](https://developers.openai.com/codex/concepts/subagents)：官方 subagent 概念、适用场景、上下文污染/腐化权衡、显式触发原则和模型/推理选择语境。
- [Codex GitHub Releases](https://github.com/openai/codex/releases/latest)：发布版本、二进制包和版本节奏核对入口。

## 上游仓库内资料

这些资料使用本地 submodule 相对链接，便于分析时固定到当前源码 commit。

- [README](../codex/README.md)：仓库定位、安装方式、文档入口和许可证说明。
- [codex-rs README](../codex/codex-rs/README.md)：Rust CLI 源码目录的官方入口。
- [AGENTS.md](../codex/AGENTS.md)：上游维护约定、Rust 编码规则、测试规则、TUI 风格、app-server API 规则等。
- [Contributing](../codex/docs/contributing.md)：贡献流程和本地开发约束。
- [Installing & building](../codex/docs/install.md)：源码安装、构建和开发环境入口。
- [LICENSE](../codex/LICENSE)：Apache-2.0 许可证。

## 当前核对记录

- 核对日期：2026-06-30
- 上游分支：`main`
- `refs/heads/main` 参考 commit：`6d2168f06ae275d5e1f73cabf935d2bcc8549998`
- 源码 submodule：[`../codex`](../codex)
- 本地依赖：尚未初始化

## 使用原则

- 不把在线文档内容整段搬入 notebook；保留 URL、用途和适用范围即可。
- 报告中的架构结论必须回到当前 submodule commit 的源码、测试或上游仓库内文档。
- 官方文档若与源码行为不一致，应明确标注“产品文档语境”和“当前源码实现”的差异。
- 非官方资料只作为辅助材料，加入时需注明来源性质和为什么值得参考。
