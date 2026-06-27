# Codex Sandbox 与权限边界分析

本文回答三个问题：Codex 中的 `sandbox` 是什么、起什么作用、以及当前源码用什么技术实现。分析以本 topic submodule 的 `openai/codex` 源码为准，官方 Codex 手册用于补充产品语义。

## 核心结论

Codex 的 `sandbox` 不是一个单独的“安全模式开关”，而是一套围绕 agent 动作边界的执行系统：

- 从产品语义看，sandbox 是 Codex 运行本地命令时的技术边界：哪些文件可读写、命令能否访问网络、是否由外部环境负责隔离。
- 从源码模型看，sandbox 的核心输入是 `PermissionProfile`，它被编译成 `FileSystemSandboxPolicy` 与 `NetworkSandboxPolicy` 两类运行时权限。
- 从执行链路看，工具调用先经过 approval/exec policy，再由 `ToolOrchestrator` 选择是否启用平台 sandbox，最后由 `SandboxManager` 把原始命令转换成 macOS / Linux / Windows 对应的 sandbox 启动命令。
- 从实现技术看，macOS 走 Seatbelt / `sandbox-exec`，Linux/WSL2 主要走 `codex-linux-sandbox` + bubblewrap + seccomp，Windows 走 restricted token / capability SID / ACL / WFP 组合，必要时使用 elevated helper。
- approval 和 sandbox 是两层控制：sandbox 定义“技术上能不能做”，approval policy 定义“越界时要不要问人”。默认本地自动化姿态接近 `workspace-write + on-request`。

一句话概括：sandbox 让 Codex 可以在明确、可执行的边界内自主行动；approval 则是它尝试跨出边界时的闸门。

## 官方语境

官方 Codex 手册把 sandbox 定义为让 Codex 在不获得整机无限制访问的情况下自主工作的边界；它作用于 Codex 运行的本地命令，而不仅是内置文件操作。手册同时明确区分：

- `sandbox`：技术边界。
- `approval policy`：何时必须暂停并请求确认。

常见用户可见模式包括：

| 模式 | 用户语义 | 源码倾向 |
| --- | --- | --- |
| `read-only` | 可检查文件，不能自动修改 | `PermissionProfile::read_only()` |
| `workspace-write` | 可在 workspace 和临时目录内工作，网络默认关 | `PermissionProfile::workspace_write()` |
| `danger-full-access` | 去掉文件系统和网络 sandbox 边界 | `PermissionProfile::Disabled` |

官方参考入口见 [`ref/INDEX.md`](./ref/INDEX.md) 中的 Codex 官方文档；本报告的源码入口集中在文末。

## 概念模型

### `PermissionProfile`

源码里当前最核心的权限抽象是 [`PermissionProfile`](./codex/codex-rs/protocol/src/models.rs)，它有三类：

- `Managed { file_system, network }`：Codex 自己负责构造 sandbox，是普通本地执行的主路径。
- `Disabled`：不套外层 sandbox，对应 `danger-full-access`；运行时文件系统不受限，网络视为 enabled。
- `External { network }`：文件系统隔离由外部调用者提供，Codex 只保留网络策略信息。

`PermissionProfile::to_runtime_permissions()` 会拆出两个运行时策略：

- [`FileSystemSandboxPolicy`](./codex/codex-rs/protocol/src/permissions.rs)
- [`NetworkSandboxPolicy`](./codex/codex-rs/protocol/src/permissions.rs)

这说明源码里的 sandbox 不是先有“某个 OS 后端”，而是先有可序列化、可合并、可从配置/请求派生的权限模型。

### 文件系统策略

`FileSystemSandboxPolicy` 由三部分组成：

- `kind`：`Restricted` / `Unrestricted` / `ExternalSandbox`。
- `entries`：一组路径权限规则。
- `glob_scan_max_depth`：Linux/Windows 等平台预展开 deny-read glob 时的深度限制。

每条 entry 是：

- 路径：绝对路径、glob，或特殊路径 `:root`、`:minimal`、`:workspace_roots`、`:tmpdir`、`:slash_tmp`。
- 权限：`read`、`write`、`deny`。

`workspace-write` 的源码语义很具体：

1. `:root = read`，先给全盘读基线。
2. `:workspace_roots = write`，当前 workspace 可写。
3. `/tmp` 和 `$TMPDIR` 通常可写，除非配置排除。
4. 额外 writable roots 可加入。
5. `.git`、`.agents`、`.codex` 在可写根下默认重新收紧为只读。

这也解释了为什么 `workspace-write` 不是“workspace 全部无限写”：Git 元数据、agent 配置和 Codex 元数据被作为 protected metadata 特别保护。

### 网络策略

`NetworkSandboxPolicy` 本身只有两态：

- `Restricted`
- `Enabled`

细粒度的域名 allow/deny、本地地址保护、Unix socket 规则、SOCKS/HTTP 代理、MITM hook 等，不直接放在这个 enum 里，而是由 [`codex-network-proxy`](./codex/codex-rs/network-proxy/src) 及配置编译链处理。

因此要区分两件事：

- `network.enabled = true`：允许命令层面有网络能力。
- `network_proxy` / profile network config：在已允许网络时，用代理规则进一步约束目的地。

源码里也保留了 `AdditionalPermissionProfile`，用于单次命令或审批后追加临时权限。它会和当前 `PermissionProfile` 合并，但 deny-read 规则有特殊保护：如果当前策略包含 deny-read，Codex 不会因为“提权/无 sandbox 重试”而静默丢掉这些读限制。

## 运行链路

Sandbox 的主执行链路可以简化成：

```text
模型发起工具调用
  -> shell/unified_exec runtime 生成 SandboxCommand
  -> ToolOrchestrator 判定 approval requirement
  -> SandboxManager 判断是否需要平台 sandbox
  -> SandboxManager::transform 生成平台启动命令
  -> ExecRequest / execute_exec_request 启动进程
  -> 若 sandbox deny 且策略允许，approval 后重试
```

关键文件：

- [`core/src/tools/orchestrator.rs`](./codex/codex-rs/core/src/tools/orchestrator.rs)：集中处理 approval、sandbox 选择、首次尝试、失败后升级重试。
- [`core/src/tools/sandboxing.rs`](./codex/codex-rs/core/src/tools/sandboxing.rs)：定义 `SandboxAttempt`、`ToolRuntime`、approval/sandbox helper。
- [`core/src/tools/runtimes/shell.rs`](./codex/codex-rs/core/src/tools/runtimes/shell.rs)：shell 工具把命令、cwd、env、additional permissions 包成 `SandboxCommand`，再调用 `SandboxAttempt::env_for()`。
- [`sandboxing/src/manager.rs`](./codex/codex-rs/sandboxing/src/manager.rs)：`SandboxManager` 根据策略选择 OS 后端并完成命令转换。

`ToolOrchestrator` 的关键行为：

- 先看 `AskForApproval` 和 exec policy，决定是否必须先审批。
- 再按 `SandboxManager::should_sandbox()` 计算本次是否请求 sandbox。
- 首次尝试如果 sandbox 内成功，直接返回。
- 如果收到 sandbox deny，且工具允许升级、approval policy 允许，再请求 approval 后重试。
- 如果当前文件系统策略含 deny-read，不能直接无 sandbox 重试，因为那会丢掉“不可读”的安全语义。

这部分和 [`codex-agent-core-report.md`](./codex-agent-core-report.md) 的工具执行主链路相衔接；本文只聚焦 sandbox 专题。

## 是否需要平台 sandbox

[`policy_transforms.rs`](./codex/codex-rs/sandboxing/src/policy_transforms.rs) 中的 `should_require_platform_sandbox()` 是判断核心：

- 有 managed network requirements 时必须 sandbox。
- 网络是 `Restricted` 时，除 `ExternalSandbox` 外需要平台 sandbox。
- 网络已 enabled 时，如果文件系统不是 full-disk write，仍需要平台 sandbox。
- 文件系统 unrestricted 且网络 enabled 时，可以不加平台 sandbox。

这意味着 `danger-full-access` 通常不会套平台 sandbox；但“网络代理强制模式”这类 managed network 要求可能仍要求一个可控执行环境。

## 平台实现

### macOS：Seatbelt / `sandbox-exec`

macOS 后端在 [`seatbelt.rs`](./codex/codex-rs/sandboxing/src/seatbelt.rs)：

- Codex 固定使用 `/usr/bin/sandbox-exec`，避免 PATH 中被注入同名程序。
- `create_seatbelt_command_args()` 把文件系统策略、网络策略、代理信息拼成 SBPL policy，并传给 `sandbox-exec -p <policy> -- <command>`。
- 文件读写策略会被编译为 Seatbelt 的 `file-read*` / `file-write*` allow 规则。
- deny-read glob 会转换成 anchored regex deny 规则，同时限制 `file-read*` 和部分 unlink 写操作，避免用破坏性操作探测路径。
- `.git`、`.agents`、`.codex` 的 protected metadata 会转成针对可写根的排除或 regex 保护。
- 网络策略会按 `NetworkSandboxPolicy` 和 proxy 状态动态生成：无网络、全网络、只允许 loopback proxy、允许 Unix socket 等。

简化后像这样：

```text
PermissionProfile
  -> FileSystemSandboxPolicy + NetworkSandboxPolicy
  -> SBPL policy text
  -> /usr/bin/sandbox-exec -p <policy> -- <command>
```

### Linux / WSL2：`codex-linux-sandbox` + bubblewrap + seccomp

Linux 后端容易被名字误导：`SandboxType::LinuxSeccomp` 并不表示只有 seccomp。当前实现是组合式的：

- [`sandboxing/src/landlock.rs`](./codex/codex-rs/sandboxing/src/landlock.rs) 负责把命令转换为 `codex-linux-sandbox --permission-profile <json> -- <command>`。
- [`linux-sandbox`](./codex/codex-rs/linux-sandbox/README.md) 是真正的 helper crate。
- 默认文件系统隔离走 bubblewrap。
- 进程内再应用 `PR_SET_NO_NEW_PRIVS` 和 seccomp 网络过滤。
- Landlock 保留为 legacy fallback，不是默认文件系统后端。

Linux helper 的执行阶段在 [`linux_run_main.rs`](./codex/codex-rs/linux-sandbox/src/linux_run_main.rs)：

1. 解析 `--permission-profile`，得到 runtime filesystem/network policies。
2. 如果是 full disk write 且不需要 proxy-only network，可能跳过 bubblewrap，只应用必要的 seccomp/no_new_privs。
3. 默认先用 bubblewrap 建立文件系统视图。
4. 再 re-enter helper 的 inner stage，应用 seccomp/no_new_privs。
5. 最后 `execvp` 到用户命令。

[`bwrap.rs`](./codex/codex-rs/linux-sandbox/src/bwrap.rs) 的文件系统语义很接近 macOS：

- 默认只读根：`--ro-bind / /`。
- 受限读策略可以从 `--tmpfs /` 开始，只挂载批准的 readable roots。
- writable roots 用 `--bind <root> <root>` 重新开放写权限。
- protected subpaths 用 `--ro-bind` 再收紧。
- deny-read roots / glob 匹配路径会被挂载遮蔽。
- `--unshare-user`、`--unshare-pid` 总是参与，网络 restricted 或 proxy-only 时加 `--unshare-net`。
- 默认挂载新的 `/proc`，失败时有 fallback。

网络 restricted 时，seccomp 会阻止 `connect`、`accept`、`bind`、`listen` 等网络相关 syscall，并限制 socket domain。managed proxy mode 下，bubblewrap 使用隔离网络 namespace，helper 通过 TCP/UDS/TCP bridge 把代理环境变量改写到 sandbox 内可达的本地端口，再用 seccomp 防止绕过。

Linux 依赖 bubblewrap 和 user namespace；WSL2 走正常 Linux 路径，WSL1 因缺少所需 user namespace 能力而不支持 bubblewrap sandboxing。

### Windows：restricted token / capability SID / ACL / WFP

Windows 后端在 [`windows-sandbox-rs`](./codex/codex-rs/windows-sandbox-rs/src)。

用户可见配置里有 [`WindowsSandboxLevel`](./codex/codex-rs/protocol/src/config_types.rs)：

- `Disabled`
- `RestrictedToken`
- `Elevated`

当 Windows sandbox 启用时，`SandboxManager::transform_for_direct_spawn()` 会把命令包装成：

```text
codex.exe --run-as-windows-sandbox ... -- <command>
```

包装参数由 [`wrapper.rs`](./codex/codex-rs/windows-sandbox-rs/src/wrapper.rs) 生成，包含：

- permission profile JSON
- cwd 和 workspace roots
- env JSON
- sandbox level
- read/write roots override
- deny-read / deny-write override
- proxy enforced 标志

Windows 权限落地大致分两条后端：

- legacy restricted-token backend：直接创建 restricted token，并用 capability SID / ACL 控制可读写区域。
- elevated backend：当配置要求 elevated 或 proxy enforced 时，走 dedicated sandbox account / command runner IPC 路径。

文件系统侧：

- [`resolved_permissions.rs`](./codex/codex-rs/windows-sandbox-rs/src/resolved_permissions.rs) 把 `PermissionProfile` 解析成 Windows-local 权限视图，并判断 read-only capability 还是 writable roots capability。
- [`token.rs`](./codex/codex-rs/windows-sandbox-rs/src/token.rs) 使用 `CreateRestrictedToken`，开启 `DISABLE_MAX_PRIVILEGE | LUA_TOKEN | WRITE_RESTRICTED`，并加入 capability SID、logon SID、Everyone 等 restricting SID。
- [`spawn_prep.rs`](./codex/codex-rs/windows-sandbox-rs/src/spawn_prep.rs) 负责准备 ACL：对 allow paths 添加 allow ACE，对 deny paths 添加 deny-write ACE，并同步 deny-read ACL 状态。
- workspace 下 `.codex` / `.agents` 等也会被保护。

网络侧：

- legacy backend 会通过环境变量尽量关闭常见工具的网络入口，例如设置 proxy 到 `127.0.0.1:9`、`PIP_NO_INDEX`、`NPM_CONFIG_OFFLINE`、`CARGO_NET_OFFLINE`、禁用 `GIT_SSH_COMMAND` 等。
- elevated/proxy 场景下还会走 Windows Filtering Platform：[`wfp.rs`](./codex/codex-rs/windows-sandbox-rs/src/wfp.rs) 安装 Codex-owned persistent provider/sublayer/filter，对指定 sandbox account 添加 blocking filters。

所以 Windows 的 sandbox 不是“Windows Sandbox 虚拟机”那类产品，而是围绕 Windows 安全 token、ACL、能力 SID、专用账户、WFP 和 wrapper 进程组合出的命令隔离层。

## 网络 sandbox 与网络 approval

网络有三层容易混淆：

1. `NetworkSandboxPolicy`：命令是否允许网络。
2. `codex-network-proxy`：在允许网络时，是否通过代理执行细粒度策略。
3. network approval flow：网络代理遇到 allowlist miss 时，是否可以触发审批。

[`network-proxy/src/config.rs`](./codex/codex-rs/network-proxy/src/config.rs) 定义了代理配置：HTTP/SOCKS 地址、domain allow/deny、Unix socket allow/deny、`allow_local_binding`、MITM hooks 等。

[`network-proxy/src/runtime.rs`](./codex/codex-rs/network-proxy/src/runtime.rs) 中 host 判断顺序是：

1. 显式 deny 优先。
2. 默认阻止 local/private 地址，防 DNS rebinding 和误连本机/内网服务。
3. 配置 allowlist 时，必须命中 allowlist。

如果 `NetworkProxySpec::start_proxy()` 启用了 approval flow，并且不是 managed hard deny，allowlist miss 可以通过 `NetworkPolicyDecider` 变成 “ask”。[`core/src/tools/network_approval.rs`](./codex/codex-rs/core/src/tools/network_approval.rs) 管理这个审批生命周期，包括一次允许、session 允许、拒绝，以及 guardian/auto-review 路由。

## 与 approval 的关系

可以用这个表理解两者分工：

| 层 | 解决什么问题 | 典型源码 |
| --- | --- | --- |
| sandbox | 技术上限制命令能做什么 | `PermissionProfile`、`SandboxManager`、OS backend |
| approval policy | 何时需要人或 reviewer 放行 | `AskForApproval`、`ToolOrchestrator`、`request_command_approval` |
| exec/network policy | 哪些命令或网络目的地按规则 allow/deny/ask | exec policy、network proxy、network approval |

常见行为：

- `workspace-write + on-request`：优先在 sandbox 内自动执行；越界或网络被拦时请求 approval。
- `read-only + on-request`：适合只读审查；写入和多数命令需要审批或被拦。
- `danger-full-access + never`：文件系统和网络边界都撤掉，也不弹 approval，是最大授权。
- `approval_policy = never` 不等于 full access；如果 sandbox 仍是 `workspace-write` 或 `read-only`，Codex 仍只能在 sandbox 约束内尽力执行。

一个很重要的源码细节：如果存在 deny-read 规则，Codex 不允许把“提权”简单变成无 sandbox 执行，因为 deny-read 只能由 sandbox 后端执行；无 sandbox 重试会把本应不可读的路径暴露出来。

## 易混淆点

- `sandbox_mode` 是旧用户配置入口；当前更通用的源码模型是 `PermissionProfile` 和 `[permissions.<profile>]`。
- Linux 的 `SandboxType::LinuxSeccomp` 名字偏窄；当前默认文件系统后端是 bubblewrap，seccomp 主要收紧网络和危险 syscall。
- `network.enabled = true` 只表示命令可有网络；如果 managed proxy 启用，实际目的地仍可能被 allowlist、local/private guard、DNS 检查和 network approval 限制。
- `ExternalSandbox` 不是无 sandbox，而是 Codex 不负责文件系统 sandbox，外部执行环境负责。
- Windows 的“sandbox”不是启动一台 Windows Sandbox VM，而是受限 token、ACL、capability SID、专用账户和 WFP 的组合。
- `danger-full-access` 只是撤掉 Codex 的本地 sandbox 边界；它不等于跳过所有更高层组织/系统策略。

## 源码入口

权限模型：

- [`protocol/src/models.rs`](./codex/codex-rs/protocol/src/models.rs)：`PermissionProfile`、`SandboxEnforcement`、built-in profiles、legacy bridge。
- [`protocol/src/permissions.rs`](./codex/codex-rs/protocol/src/permissions.rs)：filesystem/network runtime policy、protected metadata、workspace-write 构造、deny-read matcher。
- [`config/src/config_toml.rs`](./codex/codex-rs/config/src/config_toml.rs)：旧 `sandbox_mode` 到 `PermissionProfile` 的派生。
- [`core/src/config/permissions.rs`](./codex/codex-rs/core/src/config/permissions.rs)：`[permissions]` profile 编译。

执行链路：

- [`core/src/tools/orchestrator.rs`](./codex/codex-rs/core/src/tools/orchestrator.rs)：approval、sandbox selection、retry。
- [`core/src/tools/sandboxing.rs`](./codex/codex-rs/core/src/tools/sandboxing.rs)：`SandboxAttempt` 和 transform 入口。
- [`core/src/sandboxing/mod.rs`](./codex/codex-rs/core/src/sandboxing/mod.rs)：`SandboxExecRequest` 到 `ExecRequest`。
- [`sandboxing/src/manager.rs`](./codex/codex-rs/sandboxing/src/manager.rs)：平台后端选择与命令包装。
- [`sandboxing/src/policy_transforms.rs`](./codex/codex-rs/sandboxing/src/policy_transforms.rs)：additional permissions 合并、是否需要平台 sandbox。

平台实现：

- [`sandboxing/src/seatbelt.rs`](./codex/codex-rs/sandboxing/src/seatbelt.rs)：macOS Seatbelt policy 生成。
- [`sandboxing/src/landlock.rs`](./codex/codex-rs/sandboxing/src/landlock.rs)：Linux helper 参数生成。
- [`sandboxing/src/bwrap.rs`](./codex/codex-rs/sandboxing/src/bwrap.rs)：bubblewrap 检测与启动警告。
- [`linux-sandbox/README.md`](./codex/codex-rs/linux-sandbox/README.md)：Linux sandbox 当前行为说明。
- [`linux-sandbox/src/linux_run_main.rs`](./codex/codex-rs/linux-sandbox/src/linux_run_main.rs)：Linux helper 主流程。
- [`linux-sandbox/src/bwrap.rs`](./codex/codex-rs/linux-sandbox/src/bwrap.rs)：bubblewrap 文件系统视图构造。
- [`linux-sandbox/src/landlock.rs`](./codex/codex-rs/linux-sandbox/src/landlock.rs)：no_new_privs、seccomp、legacy Landlock。
- [`windows-sandbox-rs/src/wrapper.rs`](./codex/codex-rs/windows-sandbox-rs/src/wrapper.rs)：Windows wrapper 参数协议。
- [`windows-sandbox-rs/src/resolved_permissions.rs`](./codex/codex-rs/windows-sandbox-rs/src/resolved_permissions.rs)：Windows 权限解析。
- [`windows-sandbox-rs/src/token.rs`](./codex/codex-rs/windows-sandbox-rs/src/token.rs)：restricted token 和 capability SID。
- [`windows-sandbox-rs/src/spawn_prep.rs`](./codex/codex-rs/windows-sandbox-rs/src/spawn_prep.rs)：Windows spawn 前 ACL/环境准备。
- [`windows-sandbox-rs/src/wfp.rs`](./codex/codex-rs/windows-sandbox-rs/src/wfp.rs)：Windows Filtering Platform 过滤器。

网络：

- [`core/src/config/network_proxy_spec.rs`](./codex/codex-rs/core/src/config/network_proxy_spec.rs)：network proxy 启动与 managed constraints。
- [`network-proxy/src/config.rs`](./codex/codex-rs/network-proxy/src/config.rs)：domain / socket / proxy 配置模型。
- [`network-proxy/src/runtime.rs`](./codex/codex-rs/network-proxy/src/runtime.rs)：host allow/deny/local guard。
- [`core/src/tools/network_approval.rs`](./codex/codex-rs/core/src/tools/network_approval.rs)：network approval 生命周期。
