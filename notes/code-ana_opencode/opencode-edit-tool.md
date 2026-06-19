# opencode edit 工具研究报告

本文专题研究 opencode 的 `edit` 工具。重点不是只看 V1/V2 差异，而是把 `edit` 当作一个 coding agent 的核心变更能力来分析：它解决什么问题、如何设计、在执行链路中扮演什么角色、和 `read` / `write` / `apply_patch` 的边界是什么，以及在真实编码任务中会怎样被使用。

核心文件：

- [`packages/opencode/src/tool/edit.ts`](./opencode/packages/opencode/src/tool/edit.ts)
- [`packages/opencode/src/tool/edit.txt`](./opencode/packages/opencode/src/tool/edit.txt)
- [`packages/opencode/src/tool/registry.ts`](./opencode/packages/opencode/src/tool/registry.ts)
- [`packages/opencode/src/tool/external-directory.ts`](./opencode/packages/opencode/src/tool/external-directory.ts)
- [`packages/core/src/tool/edit.ts`](./opencode/packages/core/src/tool/edit.ts)
- [`packages/core/src/file-mutation.ts`](./opencode/packages/core/src/file-mutation.ts)
- [`packages/core/src/location-mutation.ts`](./opencode/packages/core/src/location-mutation.ts)
- [`packages/opencode/test/tool/edit.test.ts`](./opencode/packages/opencode/test/tool/edit.test.ts)
- [`packages/core/test/tool-edit.test.ts`](./opencode/packages/core/test/tool-edit.test.ts)

## 总览：`edit` 是 opencode 最“代理化”的写文件能力之一

从表面看，`edit` 只是“把 `oldString` 替换成 `newString`”。但在 opencode 里，它远不止一个字符串替换器。

它同时承担了几层职责：

- **模型侧可控修改**：让 LLM 用结构化参数描述一次局部修改，而不是直接生成整个文件。
- **权限审批载体**：在真正改文件前，把差异组织成 diff，交给权限系统和用户确认。
- **稳定写入机制**：处理 BOM、CRLF/LF、并发修改、外部目录访问。
- **反馈回路节点**：把 diff、诊断、变更元数据回灌给模型和 UI。
- **编码工作流中间层**：连接 `read -> decide -> edit -> diagnostics -> next step` 这条最常见的 coding loop。

因此 `edit` 在 opencode 中并不是一个“辅助工具”，而是 coding agent 进行精细代码修改时最核心的手段之一。

## 一、为什么需要 `edit`

### 1. 避免全文件重写

对于 LLM 来说，直接 `write` 整个文件有几个天然问题：

- 模型需要重放大量未修改内容，token 成本高。
- 容易破坏文件中与任务无关的部分。
- 当文件较大时，更容易引入格式、缩进、局部遗漏等错误。

`edit` 让模型只描述“我想把这段文本替换成那段文本”，从而把改动压缩到局部。

### 2. 让权限审批更可解释

opencode 的权限系统不是只问“允许编辑文件吗”，而是会把即将发生的修改差异一起带给用户。在旧路径里，`edit` 会先生成 unified diff，再把它放进 permission metadata：

```ts
yield* ctx.ask({
  permission: "edit",
  patterns: [path.relative(instance.worktree, filePath)],
  always: ["*"],
  metadata: {
    filepath: filePath,
    diff,
  },
})
```

见 [`packages/opencode/src/tool/edit.ts`](./opencode/packages/opencode/src/tool/edit.ts)。

也就是说，`edit` 的参数不仅仅驱动文件写入，也驱动“用户是否理解并批准这次修改”。

### 3. 形成一种适合 LLM 的“局部补丁语言”

`edit(path, oldString, newString, replaceAll?)` 本质上是一种比自由 diff 更简单、比整文件写入更精细的变更表达方式：

- 比 `write` 更局部。
- 比 `apply_patch` 更容易生成。
- 比 shell `sed` / `perl -pi` 更结构化、更可控。

它非常适合模型在阅读代码后做“针对性、小范围、可验证”的修改。

## 二、模型视角：`edit` 在 coding loop 中怎么被使用

从模型角度，一次典型的编码循环通常是：

```text
read / grep / glob
-> 理解上下文
-> 选择目标文件和修改点
-> 调用 edit
-> 观察 tool result / diagnostics
-> 再读、再改或转向其他工具
```

这意味着 `edit` 几乎总是和 `read` 成对出现。

旧版工具说明 [`edit.txt`](./opencode/packages/opencode/src/tool/edit.txt) 也明确写了这一点：

- 先 `Read` 再 `Edit`
- 不要把 `Read` 输出中的行号前缀带进 `oldString`
- 优先编辑已有文件，而不是新建文件
- 多重匹配时要补更多上下文或用 `replaceAll`

不过从当前旧版 [`edit.ts`](./opencode/packages/opencode/src/tool/edit.ts) 看，没有看到“必须已 Read 才允许 Edit”的直接运行时检查。这条更像 prompt-level guidance：它通过工具说明训练模型先读再改，而不是由 `edit` 执行函数强制验证读取历史。

这里体现出一个重要设计思想：**`edit` 不是独立能力，而是读取后的局部落地能力**。

## 三、旧版 `edit`：一个“高功能、强工作流”的编辑工具

### 参数形式

旧版 `edit` 的参数很简单：

- `filePath`
- `oldString`
- `newString`
- `replaceAll`

见 [`packages/opencode/src/tool/edit.ts`](./opencode/packages/opencode/src/tool/edit.ts)。

但执行逻辑并不简单。

### 1. 支持“空 oldString 创建新文件”

旧版 `edit` 有一个很有意思的设计：如果 `oldString === ""`，且目标文件不存在，它允许直接创建新文件。

这意味着旧版 `edit` 不只是“修改现有文件”，也能兼任一种受限写入：

- 如果文件已存在，空 `oldString` 会被拒绝。
- 如果文件不存在，则把 `newString` 作为内容写入，并走同样的 diff/permission/event 流程。

测试里也覆盖了这个行为，见 [`packages/opencode/test/tool/edit.test.ts`](./opencode/packages/opencode/test/tool/edit.test.ts) 中 “creates new file when oldString is empty”。

这说明旧版 `edit` 的定位其实更接近“文本级 mutation tool”，而不是纯粹的 existing-file exact editor。

### 2. 先算 diff，再 ask permission，再写入

旧版 `edit` 的一个核心思路是：

```text
读源文件
-> 计算目标内容
-> 生成 diff
-> 用户审批
-> 真正写盘
```

这里的 diff 不只是给用户看，也会进入 tool metadata、session summary、UI 展示。

### 3. 写入后还有 formatter、watcher、LSP

旧版 `edit` 写入之后不会立刻结束，而会继续串联几个副作用：

- `format.file(filePath)`
- 发布 `FileSystem.Event.Edited`
- 发布 `Watcher.Event.Updated`
- `lsp.touchFile(filePath, "document")`
- 收集 `lsp.diagnostics()`

最后如果当前文件出现诊断问题，还会把诊断文本直接拼到 tool 输出里：

```text
LSP errors detected in this file, please fix:
...
```

这说明旧版 `edit` 不是“写完就算了”，而是把“写入后的静态/语义反馈”继续压回模型上下文，逼迫 agent 进入自我修复循环。

### 4. tool metadata 是一等产物

旧版 `edit` 还会产出：

- `diff`
- `filediff`
- `diagnostics`

其中 `filediff` 是 [`Snapshot.FileDiff`](./opencode/packages/opencode/src/snapshot/index.ts) 兼容结构，包含：

- `file`
- `patch`
- `additions`
- `deletions`

这些元数据会被：

- CLI/TUI 工具视图消费
- permission 提示消费
- session diff summary 消费
- 导出/展示系统消费

所以 `edit` 不是只返回一段“成功/失败”的文本；它是 session 中变更语义的重要生产者。

## 四、旧版替换算法：不是 exact match，而是“渐进放宽”的匹配器链

旧版 `edit` 最值得深入研究的部分，是它的 `replace(...)` 实现。

### 1. 设计目标

LLM 经常会出现这样的问题：

- 少带了一点缩进
- 行尾换行符风格不一致
- 从 `Read` 结果中复制时丢了一点空白
- 多行块中间有小幅偏差
- 文本被转义后再反向传回

如果 `edit` 只接受死板的 exact match，会非常脆弱，模型会频繁失败。旧版因此采用了一个 **replacer pipeline**，从最严格的匹配开始，逐步尝试更宽松但仍受保护的匹配。

### 2. replacer 链

旧版 `replace(...)` 会依次尝试：

1. `SimpleReplacer`
2. `LineTrimmedReplacer`
3. `BlockAnchorReplacer`
4. `WhitespaceNormalizedReplacer`
5. `IndentationFlexibleReplacer`
6. `EscapeNormalizedReplacer`
7. `TrimmedBoundaryReplacer`
8. `ContextAwareReplacer`
9. `MultiOccurrenceReplacer`

见 [`packages/opencode/src/tool/edit.ts`](./opencode/packages/opencode/src/tool/edit.ts) 中 `replace(...)`。

这条链的思想非常明确：

- 先尽量 exact
- exact 不行，再尝试“模型最常犯的轻微偏差”
- 但又不能无限宽松，否则会改错位置

### 3. 几个关键 replacer 的设计意义

#### `LineTrimmedReplacer`

按行比较时忽略前后空白，适合模型因为缩进或复制问题导致的轻微不一致。

#### `BlockAnchorReplacer`

对多行块，抓第一行和最后一行作为锚点，再对中间内容做相似度判断。这是一种很典型的“结构锚定”策略：即使中间局部有偏差，只要边界和整体相似度足够高，就仍可定位块。

#### `IndentationFlexibleReplacer`

对多行块先做公共缩进去除再比较，适合嵌套代码块因缩进层级变化导致的错位。

#### `EscapeNormalizedReplacer`

处理 `\n`、`\t`、引号、反斜杠等转义差异，适合模型把代码字符串或模板文本重新编码之后再回写。

#### `ContextAwareReplacer`

如果一大块文本中只有首尾行和部分中间行能对上，也会尝试在上下文上判断是否属于同一块。这体现出旧版 `edit` 的一个思路：**不仅匹配文本，也匹配局部结构上下文**。

### 4. 为什么还要设防

旧版 `edit` 虽然宽松，但也不是无限放宽。它有一个关键保护：

```ts
isDisproportionateMatch(search, oldString)
```

如果匹配到的真实块远大于 `oldString`，会直接拒绝：

```text
Refusing replacement because the matched span is much larger than oldString...
```

这很重要。因为一旦 fuzzy matching 过宽，LLM 可能本来想改一小段，结果误匹配到一大块相似代码。这个保护就是在压制“宽松匹配带来的误伤”。

### 5. 旧版 `edit` 的本质

可以把旧版 `edit` 理解成：

```text
exact-replace first
+ model-error correction layer
+ human approval layer
+ post-write diagnostics layer
```

这是一种非常 agent-oriented 的设计。它不是单纯追求文本纯度，而是在为“不稳定的 LLM 局部编辑”提供容错和护栏。

## 五、V2 `edit`：缩回到 exact-edit leaf

V2 的 [`packages/core/src/tool/edit.ts`](./opencode/packages/core/src/tool/edit.ts) 明显更窄。

### 1. 明确退回 exact text replace

V2 的工具描述就是：

```text
Replace exact text in one file.
```

执行逻辑也很简单：

- resolve path
- external_directory permission
- edit permission
- 读取文件
- 统一行尾
- 统计 exact occurrences
- 0 次匹配报错
- 多次匹配且未 `replaceAll` 报错
- `writeIfUnchanged(...)` 条件提交

不再带 fuzzy replacer pipeline。

### 2. 通过 TODO 明确承认“功能回退”

V2 文件里直接写了 TODO：

- Port V1 fuzzy correction strategies
- Add formatter integration
- Publish watcher/file-edit events
- Add snapshots / undo
- Add LSP notification and diagnostics

这说明 V2 目前不是“更强版本”，而是**更小、更干净、更 typed 的核心叶子实现**。

换句话说，V2 的目标不是先保留所有旧行为，而是先把最核心的不变式固定下来：

- 输入 schema 明确
- 权限边界明确
- 文件提交语义明确
- tool result 结构明确

然后再逐项把旧版富功能接回去。

### 3. 为什么要收缩

这是一个很典型的 runtime-first 重构策略：

- 先定义一个小而确定的语义核心
- 把模糊、UI、格式化、LSP、副作用延后
- 避免一开始把过多隐式行为带入新的 durable runtime

从系统演化角度看，这样做是合理的。因为旧版 `edit` 的 fuzzy matching、watcher、formatter、diagnostics 都会影响 tool settlement 和 event 语义；V2 先把“什么叫一次安全的条件编辑”钉死，再慢慢恢复高级体验。

## 六、权限与安全：`edit` 不只是改文件，还要决定“何时读取文件”

### 1. 旧版：外部目录先审批，edit diff 后审批

旧版有两类权限边界：

- `external_directory`：如果目标在 worktree 外，先审批外部目录访问。
- `edit`：为了给用户展示 diff，需要先读文件、生成修改结果，再带着 diff ask permission。

这带来一个特点：即便最后 `edit` 被拒绝，工具在 `edit` ask 之前其实已经读过目标文件并推导出了 diff。外部目录审批可以挡住 worktree 外访问，但 worktree 内的 edit diff 审批不是“读前审批”。

### 2. V2：权限优先于内容披露

V2 更强调“先权限，后内容”。

测试里专门覆盖了一个场景：当 `edit` 权限被拒绝时，不应读取目标文件内容，也不应向模型暴露“`oldString` 到底匹不匹配”。见 [`packages/core/test/tool-edit.test.ts`](./opencode/packages/core/test/tool-edit.test.ts) 中 “denied edit reads no target content and does not disclose whether oldString matches”。

这背后对应 V2 `edit` 的实际顺序：

1. resolve 路径
2. external_directory assert
3. edit assert
4. 只有通过后，才读取文件并检查 `oldString`

这比旧版更严格，也更符合 durable permission runtime 的思路。

### 3. 外部目录边界

无论 V1 还是 V2，`edit` 都不是简单地拿路径就写。

旧版靠 [`external-directory.ts`](./opencode/packages/opencode/src/tool/external-directory.ts)：

- 如果目标不在当前实例 worktree 中，
- 先申请 `external_directory`
- 资源模式通常是父目录下的 `*`

V2 则通过 [`location-mutation.ts`](./opencode/packages/core/src/location-mutation.ts) 把路径 canonicalize，并显式生成：

- location 内 resource
- external directory boundary

这意味着 `edit` 在权限系统中的真正资源不是原始路径字符串，而是规范化后的 permission resource。

## 七、并发与一致性：为什么 `edit` 需要锁和条件写

### 1. 旧版：进程内文件锁

旧版 `edit` 用一个 `Map<string, Semaphore>` 按 canonical file path 给文件加锁，避免多个 edit 同时踩同一个文件。

测试也验证了并发修改同一文件不同区域时，最终应能保住两个改动，见 [`packages/opencode/test/tool/edit.test.ts`](./opencode/packages/opencode/test/tool/edit.test.ts) 中 “preserves concurrent edits to different sections of the same file”。

### 2. V2：`FileMutation.writeIfUnchanged`

V2 没有把并发逻辑埋在 `edit` 自己内部，而是下沉到 [`file-mutation.ts`](./opencode/packages/core/src/file-mutation.ts)：

- 按 canonical target 加 keyed mutex
- `writeIfUnchanged` 在锁内比较当前 bytes 与 `expected`
- 如果文件已变，则抛 `StaleContentError`

`edit` 把这个错误映射成：

```text
File changed after permission approval. Read it again before editing.
```

这是一种比旧版更 runtime-native 的一致性策略：

- 不保证“计算 diff 时世界静止”
- 但保证“提交时如果世界变了，就不把旧假设覆盖上去”

对 coding agent 来说，这很关键，因为长任务中用户、别的 agent、格式化器、文件同步器都可能同时动文件。

## 八、`edit`、`write`、`apply_patch`、`read` 的关系

### 1. 和 `read`

`read` 是 `edit` 的上游。模型通常要先用 `read` 获取局部文本，再把其中一段送入 `oldString`。旧版 `edit.txt` 明确告诫模型：

- 必须先 read
- 不要带 `1: ` 这种行号前缀

这说明 `edit` 的成功率高度依赖读取结果的格式与模型如何复制该结果。

### 2. 和 `write`

`write` 更适合：

- 创建全新文件
- 整个文件需要重写
- `oldString` 很难稳定定位

`edit` 更适合：

- 小范围修改已有文件
- 保留周围上下文不变
- 用户或模型希望 diff 更可解释

旧版其实允许 `edit(oldString="")` 创建文件，这让两者边界有些重叠；但从方法论上，`write` 是 whole-file mutation，`edit` 是 localized mutation。

### 3. 和 `apply_patch`

`apply_patch` 是更高表达力的多文件变更工具：

- 可以一次改多个文件
- 支持 add/update/delete/move
- 更适合批量修改和结构性补丁

旧版 registry 里还会基于模型动态切换：

```ts
const usePatch =
  input.modelID.includes("gpt-") && !input.modelID.includes("oss") && !input.modelID.includes("gpt-4")
if (tool.id === ApplyPatchTool.id) return usePatch
if (tool.id === EditTool.id || tool.id === WriteTool.id) return !usePatch
```

见 [`packages/opencode/src/tool/registry.ts`](./opencode/packages/opencode/src/tool/registry.ts)。

这意味着对某些模型，opencode 故意让它更多使用 `apply_patch`，而不是 `edit/write`。这里背后的判断是：

- 有些模型更擅长生成 structured patch
- 有些模型更擅长逐文件 `oldString/newString`

所以 `edit` 不是唯一编辑工具，而是 opencode 针对模型能力做出来的某一类 mutation interface。

## 九、`edit` 在 coding 任务中的典型使用情形

### 1. 精确修一处 bug

比如：

- 条件判断改一个布尔分支
- 调整一个函数调用参数
- 替换一段错误消息
- 改一个 import / type annotation / config key

这类场景最适合 `edit`：局部、可读、风险低。

### 2. 在读完上下文后修改一个代码块

先 `read` 函数，再 `edit` 替换函数体中的一段。这是 coding agent 最经典的模式。

### 3. rename / 批量字符串替换

当同一个精确文本在单个文件内多次出现时，用 `replaceAll` 很自然。旧版 `edit.txt` 也明确把变量 rename 作为典型场景。

### 4. 生成对用户友好的审批 diff

在 ask 模式下，`edit` 因为天然生成 diff，很适合在人类确认回路里使用。相较之下，shell 或一些“黑箱变换”更不透明。

### 5. 配合 LSP 迭代修错

旧版 `edit` 会在写入后把当前文件诊断直接反馈给模型，这使它很适合：

```text
小改一处
-> 看诊断
-> 再 edit 修诊断
```

这其实是一个非常强的 agent 自修复环。

## 十、`edit` 的设计哲学

### 1. 用局部文本替换来约束模型

opencode 没让模型随意说“去改那个文件”，而是逼它给出 `oldString/newString`。这是一种强约束：

- 迫使模型阅读具体上下文
- 迫使模型描述“我要改哪一段”
- 使审批、diff、回滚、诊断都更有锚点

### 2. 工具本身对模型误差有容错

旧版 `edit` 的 replacer 链体现出一个非常现实的哲学：**要适应 LLM，而不是假设 LLM 会完美复制文本。**

### 3. 但容错不能突破安全边界

所以又有：

- 多匹配报错
- disproportionate match 拒绝
- external_directory 审批
- stale content 拒绝提交

换句话说，`edit` 的容错是“帮助模型成功”，不是“让模型可以模糊地乱改”。

### 4. 从 prompt-heavy 迁移到 runtime-heavy

旧版靠：

- `edit.txt`
- fuzzy replacer pipeline
- post-write formatter/LSP/watcher

形成一个很厚的“工作流型工具”。

V2 则先收缩为：

- typed schema
- exact semantics
- permission-first
- conditional commit

这反映出 opencode 的整体演进：把以前散落在 prompt 和工具副作用里的东西，逐渐重新分配到更稳定的 runtime abstraction 中。

## 十一、当前结论

`edit` 是 opencode coding agent 最关键的局部修改能力之一。它的重要性不在于“能替换字符串”，而在于它把一次代码修改组织成了一个可代理、可审批、可诊断、可继续迭代的闭环。

如果用一句话概括：

```text
`edit` 是 opencode 把“读懂代码后做一个小而具体的修改”这件事，工程化为工具协议的代表作。
```

旧版 `edit` 更像一个高功能、懂模型误差、懂 diff、懂 LSP 的智能编辑器叶子；V2 `edit` 则更像一个语义更干净、权限更前置、结果更可验证的 exact-edit core。两者的差别不只是功能多寡，而是系统设计重心从“工具自带丰富工作流”转向“runtime 主导工作流、工具保持小而稳”。
