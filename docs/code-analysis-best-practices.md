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
├── ${repo}/                 # 待分析仓库，使用 git submodule 引入
├── ${repo}-core-report.md   # 主报告，可按实际命名
└── ${repo}-${topic}.md      # 专题报告，可多篇
```

命名建议：

- topic 目录使用 `notes/code-ana_${RepoName}`。
- submodule 目录使用上游仓库名，保持短小稳定。
- `INDEX.md` 只做摘要、阅读顺序和文件入口，不承载长分析。

### submodule 引入

优先把待分析仓库作为 submodule 放在对应 topic 内：

```bash
mkdir -p notes/code-ana_${RepoName}
git submodule add ${RepositoryURL} notes/code-ana_${RepoName}/${repo}
```

如果 `.gitmodules` 已经手动配置好，则使用：

```bash
git submodule update --init --recursive notes/code-ana_${RepoName}/${repo}
```

初始化后记录：

- 上游仓库地址。
- 当前 commit。
- 是否有子模块。
- 分析时采用的分支或 tag。

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
- 创建 topic `INDEX.md`。
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
- 用流程图或文本结构表达主链路。
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
