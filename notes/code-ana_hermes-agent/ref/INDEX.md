# hermes-agent 权威参考资料索引

本目录用于收录 `hermes-agent` 代码分析时可优先参考的官方或权威资料入口。资料正文默认不搬运到本仓库；除非后续分析确有必要，优先保存 URL、用途说明和本地源码内对应位置。

## 官方资料

- Hermes Agent 官方文档：<https://hermes-agent.nousresearch.com/docs>
  - 用途：理解产品能力、安装配置、用户指南、开发者指南和参考文档。
  - 维护方式：作为在线文档总入口，分析前优先核对相关页面。
- GitHub 仓库：<https://github.com/NousResearch/hermes-agent>
  - 用途：确认上游源码、README、issue/release 入口和项目元信息。
  - 本地镜像：[`../hermes-agent`](../hermes-agent)

## 技能系统参考

- [`../hermes-agent/website/docs/user-guide/features/skills.md`](../hermes-agent/website/docs/user-guide/features/skills.md)
  - 用途：理解 skill 的用户可见行为、加载方式和 agent-managed skills。
- [`../hermes-agent/website/docs/developer-guide/creating-skills.md`](../hermes-agent/website/docs/developer-guide/creating-skills.md)
  - 用途：理解 skill 的目录结构、frontmatter、最佳实践和 authoring 约束。
- [`../hermes-agent/website/docs/developer-guide/prompt-assembly.md`](../hermes-agent/website/docs/developer-guide/prompt-assembly.md)
  - 用途：理解 skills system 如何向 prompt 贡献紧凑索引，以及与其他 prompt 层的装配顺序。
- [`../hermes-agent/website/docs/user-guide/features/curator.md`](../hermes-agent/website/docs/user-guide/features/curator.md)
  - 用途：理解 curator 的自动维护、归档和 consolidation 行为。
- [`../hermes-agent/website/docs/reference/tools-reference.md`](../hermes-agent/website/docs/reference/tools-reference.md)
  - 用途：核对 `skill_manage`、`skill_view`、`skills_list` 等工具入口在参考文档中的定义。

## 本地源码内资料入口

- [`../hermes-agent/README.md`](../hermes-agent/README.md)：项目定位、核心能力、安装入口和文档导航。
- [`../hermes-agent/CONTRIBUTING.md`](../hermes-agent/CONTRIBUTING.md)：开发环境、贡献流程和代码约定。
- [`../hermes-agent/SECURITY.md`](../hermes-agent/SECURITY.md)：安全策略与漏洞报告入口。
- [`../hermes-agent/docs`](../hermes-agent/docs)：源码仓库内的设计、契约、生命周期、安全和 RCA 文档。
- [`../hermes-agent/website/docs`](../hermes-agent/website/docs)：官方文档站点的英文文档源文件。

## 使用约定

- 先从官方文档和本地源码内资料建立产品语境，再回到当前 submodule commit 的源码验证实现。
- 代码分析结论以本地 submodule 当前 commit 为准；在线资料用于补充意图、概念和用户可见行为。
- 若新增外部参考，优先选择官方文档、上游仓库、标准规范、论文或一手发布资料，并记录其用途。
