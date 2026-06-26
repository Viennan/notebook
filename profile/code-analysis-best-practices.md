# Code Analysis Best Practices

本文总结本次 opencode 代码分析形成的经验，用于以后分析其他代码仓库。

## 目标

代码分析不是按目录复述源码，而是回答一个核心问题：

> 这个项目的关键能力是如何被组织、执行和约束的？

报告应帮助读者快速建立主干理解，并能按需跳转到源码细节。

## 初始化工作流

新增代码分析 topic 时，先完成目录和仓库初始化，再开始分析。

### 目录模板

```text
notes/code-ana_${RepoName}/
├── INDEX.md
├── ref/
│   └── INDEX.md             # 官方或权威参考资料索引
├── ${repo}/                 # 待分析仓库，使用 git submodule 引入
├── ${repo}-core-report.md   # 主报告，可按实际命名
└── ${repo}-${topic}.md      # 专题报告，可多篇
```

命名建议：

- topic 目录使用 `notes/code-ana_${RepoName}`。
- submodule 目录使用上游仓库名，保持短小稳定。
- `ref/INDEX.md` 用于索引官方或权威参考资料，避免权威 URL 散落在报告正文里。
- `INDEX.md` 只做摘要、阅读顺序和文件入口，不承载长分析。

### 权威资料参考机制

代码分析 topic 可以在 `ref/` 目录维护官方或权威参考资料索引。推荐从初始化阶段就建立 `ref/INDEX.md`，后续分析时按需扩充。

`ref/INDEX.md` 建议记录：

- 官方文档入口、上游仓库、release、规范、论文或一手发布资料。
- 本地 submodule 内的 README、CONTRIBUTING、SECURITY、docs、website docs 等资料入口。
- 每个资料的用途，例如产品语境、开发者指南、API reference、设计文档或安全边界。
- 必要时记录最后核对日期、对应版本、适用范围和已知过时风险。

使用原则：

- 资料正文默认不搬运到 notebook；优先保存 URL、用途说明和本地相对链接。
- 分析结论以当前 submodule commit 的源码为准，在线资料用于补充意图、概念和用户可见行为。
- topic `INDEX.md` 应索引 `ref/INDEX.md`，但不把所有参考 URL 展开到 topic 首页。
- 新增非官方资料时，优先选择一手来源；社区文章、博客和讨论串要标明其辅助性质。

### submodule 引入

优先把待分析仓库作为 submodule 放在对应 topic 内。submodule 引入命令由用户在本地执行；agent 只提供命令和目标路径，等待用户确认完成后，再继续创建索引、记录 commit 和做后续检查。

```bash
mkdir -p notes/code-ana_${RepoName}
git submodule add ${RepositoryURL} notes/code-ana_${RepoName}/${repo}
```

如果 `.gitmodules` 已经手动配置好，也应把初始化命令交给用户执行：

```bash
git submodule update --init --recursive notes/code-ana_${RepoName}/${repo}
```

初始化后记录：

- 上游仓库地址。
- 当前 commit。
- 是否有子模块。
- 分析时采用的分支或 tag。

同时维护 VS Code 文件监听排除，避免大型 submodule 占满 watcher 导致新输出的 Markdown 无法被正常跟踪：

- 在 `.vscode/settings.json` 的 `files.watcherExclude` 中为 submodule 路径加入 `true`。
- 排除粒度只到 submodule 目录，例如 `notes/code-ana_${RepoName}/${repo}/**`；不要排除整个 `notes/code-ana_${RepoName}/**`，否则 topic 内新生成的 Markdown 也可能被编辑器忽略。
- 新增、移动或移除 submodule 时，同步增删对应 watcher 排除项，保留用户已有的其他 VS Code 设置。

常用检查：

```bash
git -C notes/code-ana_${RepoName}/${repo} status --short
git -C notes/code-ana_${RepoName}/${repo} rev-parse HEAD
git -C notes/code-ana_${RepoName}/${repo} submodule status --recursive
```

### 源仓库只读原则

分析过程中默认不得改动 submodule 源仓库内容。允许的例外是：写入源仓库自身 `.gitignore` 已经忽略的目录或文件，用于安装依赖、生成索引、运行本地工具。

例如：

- 可以在已忽略的 `node_modules/`、`.turbo/`、`target/`、`.venv/`、`.gradle/` 中生成内容。
- 不要修改源码、lockfile、配置文件、测试快照或格式化结果。
- 运行任何可能写源码的命令前，先确认输出路径是否被忽略。

检查某个路径是否被源仓库忽略：

```bash
git -C notes/code-ana_${RepoName}/${repo} check-ignore -v node_modules/
git -C notes/code-ana_${RepoName}/${repo} check-ignore -v target/
```

分析结束前检查 submodule 是否保持干净：

```bash
git -C notes/code-ana_${RepoName}/${repo} status --short
```

若出现非忽略文件改动，应暂停并确认来源，不要随手 revert 用户改动。

### 依赖初始化原则

初始化依赖的目标是提升代码阅读、跳转和类型理解能力，不是构建或修改项目。默认选择不会运行安装脚本、不会更新 lockfile 的命令。

先识别仓库类型：

```bash
rg --files notes/code-ana_${RepoName}/${repo} -g 'package.json' -g 'bun.lock' -g 'pnpm-lock.yaml' -g 'package-lock.json' -g 'yarn.lock' -g 'Cargo.toml' -g 'go.mod' -g 'pyproject.toml'
```

推荐命令：

| 仓库类型 | 优先命令 | 说明 |
| --- | --- | --- |
| Bun / TypeScript | `bun install --frozen-lockfile --ignore-scripts` | 适合有 `bun.lock` 的仓库，避免改 lockfile 和执行 postinstall |
| pnpm / TypeScript | `pnpm install --frozen-lockfile --ignore-scripts` | 适合有 `pnpm-lock.yaml` 的仓库 |
| npm / TypeScript | `npm ci --ignore-scripts` | 适合有 `package-lock.json` 的仓库 |
| Yarn classic | `yarn install --frozen-lockfile --ignore-scripts` | 适合 Yarn v1 |
| Rust | `cargo fetch --locked` | 只拉依赖，避免改 `Cargo.lock` |
| Go | `go mod download` | 下载模块缓存，通常不改源码 |
| Python | `python -m venv .venv` 后按 lock/requirements 安装 | 仅当 `.venv/` 已被忽略时使用 |
| Java / Maven | `mvn -q -DskipTests dependency:go-offline` | 预取依赖，不运行测试 |
| Java / Gradle | `./gradlew --no-daemon dependencies` | 可能写 `.gradle/`，先确认其被忽略 |

如果项目依赖安装脚本生成类型文件或 native binding，先记录原因，再决定是否运行更完整的安装命令。不要默认运行会修改源码树的生成命令。

### topic 初始化清单

- 创建 `notes/code-ana_${RepoName}/`。
- 引入 submodule。
- 将 submodule 路径加入 `.vscode/settings.json` 的 `files.watcherExclude`，只排除 `${repo}/**`。
- 创建 topic `INDEX.md`。
- 创建 `ref/INDEX.md`，记录官方或权威参考资料入口。
- 更新 `notes/INDEX.md`。
- 记录仓库 commit 和分析范围。
- 判断是否需要依赖初始化；若需要，选择对应技术栈的只读/冻结命令。
- 确认 submodule 没有非忽略改动。

## 推荐流程

1. 先定分析范围
   - 明确关注的产品能力或技术问题。
   - 主动排除暂不相关模块，例如 UI、企业版、配置边角等。
   - 对“当前实现”和“迁移中架构”保持区分。

2. 先找主执行链路
   - 优先定位入口、核心 loop、数据模型、状态流转和关键服务边界。
   - 不急着细读所有文件，先画出系统如何从输入走到结果。
   - 找到“真正做事”的模块，而不是只看 API 或文档表层。

3. 用问题驱动深挖
   - 将用户疑问整理成具体问题。
   - 每轮只深挖一个抽象层或机制。
   - 对容易混淆的点用 FAQ 记录，例如角色、状态、权限、上下文、消息格式。

4. 源码、测试、提示词一起读
   - 源码说明机制。
   - 测试确认边界和异常行为。
   - prompt/tool description 说明模型能看到什么、被如何引导。
   - 文档可辅助理解，但结论以当前源码为准。

5. 区分事实与解释
   - 事实：代码在哪里、流程怎样、字段如何变化。
   - 解释：为什么这样设计、权衡是什么、可能风险是什么。
   - 高层判断要能回到具体源码或行为证据。

6. 保持文档可导航
   - 总报告建立地图。
   - 专题报告聚焦单一机制。
   - FAQ 收敛反复讨论的问题。
   - `INDEX.md` 用摘要和阅读顺序组织，不堆满细节。

## 报告写法

- 开头先给核心结论，再列源码地图。
- 报告应先解释系统完成什么功能、为什么需要这些抽象、模型如何建立、运行路径如何串起来，最后才进入必要的实现代码。
- 推荐按 why / what / how / evidence 组织：先讲设计目标和用户问题，再讲功能边界、核心对象、状态与关系，然后讲读写时机、调用链、生命周期和异常路径，最后用源码、测试、prompt 或配置片段作为证据。
- 避免写成逐文件代码笔记。文件路径、类名和函数名应服务于架构理解，不应替代功能模型和运行路径说明。
- 适当使用流程图、时序图、数据流图、状态表、对比表等表达主链路；当系统有多种 memory、provider、tool、prompt 注入位置或生命周期时，图表通常比长段文字更清晰。
- 代码片段只放必要部分，较长代码用源码链接。
- 对迁移中项目，明确标注旧路径、目标架构和双轨边界。
- 结尾总结设计哲学、限制和风险。

## 常用检查清单

- 是否回答了“主干如何工作”？
- 是否说明了关键抽象之间的关系？
- 是否避免把配置、UI、文档表层误认为核心实现？
- 是否核对了测试或调用方来确认行为？
- 是否说明模型实际能看到什么，而不是只看内部数据结构？
- 是否更新了 topic `INDEX.md`？

## opencode 分析中的有效模式

- 先建立 session loop、tool call、permission、event/projection 等主线，再做专题。
- 对 V1/V2 这类双轨实现，先比较抽象变化，再逐层研究 V2 部件。
- 对 prompt、tool、compaction、multi-agent 等模型相关机制，要追问“provider 最终看到什么”。
- 对用户连续追问的问题，及时整理成 FAQ，避免知识散落在对话中。
