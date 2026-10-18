Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Airymax Lockdown 阶梯设计
> **文档定位**：AirymaxOS 单调递增 lockdown 阶梯（none/early/late/full）的唯一权威详细设计，定义等级状态机、接口规则表、内核设施映射、接入点与错误语义\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-10-18\
> **上级文档**：[AirymaxOS 总览](../README.md) + [110-security/README.md](README.md)\
> **设计依据**：本文件为 SSoT（Single Source of Truth），`security/airy/airy_lockdown.c` 的等级语义、规则表内容、gate 接入点、boot 参数与 securityfs 接口均以本文件为唯一权威定义

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Airymax lockdown 阶梯** 的唯一权威源。四级等级定义、单调递增约束、`airy_lock_gate()` / `airy_lock_down()` 的判定方向、三个 agent 接口规则、`enum lockdown_reason` 到等级的映射、`airy.lockdown=` 与 `securityfs` 两个触发入口、`-AIRY_ELOCKDOWN` 与 `-EPERM` 的错误分工、两个 ftrace 事件均以本文件为唯一权威定义。其他文档只能引用本文件，禁止重新定义等级划分或 gate 判定方向。
>
> [SC] 侧对应物：`kernel/include/airymax/airy_lockdown.h`（类型与原型）、`kernel/include/uapi/linux/airymax/error.h`（`AIRY_ELOCKDOWN`）、`kernel/include/trace/events/airy.h`（事件）。文档与头文件出现分歧时，以头文件为编译期事实，并同步修订本文件。

---

## 文档信息卡

- **目标读者**：安全模块开发者、发行版集成者、运维工程师
- **前置知识**：理解 [01-lsm-framework.md](01-lsm-framework.md) 的 LSM 框架与 early LSM 机制、[07-airy-lsm-design.md](07-airy-lsm-design.md) 的 `airy_uring_cmd_check()` 钩子、Linux `Documentation/admin-guide/lockdown.rst`
- **预计阅读时间**：25 分钟
- **核心概念**：单调阶梯、early LSM、接口收敛、策略拒绝与安全违规的区分
- **复杂度标识**：中级

---

## §1 设计目标

### 1.1 为什么在 Linux lockdown 之外再造一层

Linux 6.6 内置 lockdown LSM 只有 integrity 与 confidentiality 两个档位，且档位只能通过命令行或 securityfs 在启动早期设定一次。它覆盖的是内核自身设施的收敛（`/dev/mem`、`kexec`、debugfs、MSR 等），对 AirymaxOS 新增的 agent 接口（capability 授予、RoVol 写、调度改写）没有任何约束。

AirymaxOS 需要在同一套机制里同时收敛**两类**面：

| 面 | 例子 | Linux lockdown 覆盖 | Airymax 阶梯覆盖 |
|----|------|--------------------|-----------------|
| 内核设施面 | `/dev/mem`、`kexec_load`、debugfs、perf、kprobes | 是（integrity/confidentiality） | 是（复用 `security_locked_down()` 钩子） |
| Agent 接口面 | capability 授予、RoVol 写、调度策略改写 | 否 | 是（`airy_lock_gate()`） |

选择复用 `security_locked_down()` 钩子而非旁路实现，是为了让内核中已有的数百个 `security_locked_down()` 调用点自动获得 Airymax 等级语义，无需逐个修改调用点。

### 1.2 为什么是四档而不是两档

两档无法表达"启动期必须关闭、运行期可以短暂开放、配置冻结后彻底关闭"的三段式部署节奏。四档把收敛过程拆成可观测的台阶：

- **none**：开发与调试镜像的默认值，所有接口开放。
- **early**：boot 阶段即生效，关闭"能改写内核本体或绕过模块签名"的路径。
- **late**：Macro-Supervisor 就绪后，关闭"能观测或干预内核内部状态"的路径。
- **full**：配置冻结后，关闭所有已知设施，仅保留收缩方向的操作。

---

## §2 等级与状态机

### 2.1 等级定义

| 值 | 名称 | 语义 | 典型部署 |
|----|------|------|---------|
| 0 | `AIRY_LOCK_NONE` | 不收敛任何接口 | 开发镜像 |
| 1 | `AIRY_LOCK_EARLY` | 关闭内核本体改写与 agent 扩张路径 | 生产镜像默认 |
| 2 | `AIRY_LOCK_LATE` | 追加关闭内核内部观测路径 | 业务就绪后 |
| 3 | `AIRY_LOCK_FULL` | 关闭全部已知设施 | 配置冻结后 |

等级由 `atomic_t` 承载，`airy_lock_get()` 读取，`airy_lock_set()` 提升。

### 2.2 单调性

`airy_lock_set()` 的判定只有三种结果：

| 当前 `cur` 与请求 `level` 的关系 | 结果 | 原因 |
|--------------------------------|------|------|
| `cur < level` | `atomic_try_cmpxchg` 提升，记 `pr_info` 与 `airy_lockdown_change`，返回 0 | 正常提升 |
| `cur == level` | 返回 0，不记事件 | 幂等，重复设置不算状态变化 |
| `cur > level` | 返回 `-EPERM`，状态不变 | 禁止降级 |

禁止降级是安全属性而非便利性取舍：一旦某个接口被关闭，说明系统已进入更严格的信任阶段；允许降级等价于给攻击者一个"先触发降级再改写"的窗口。需要重新开放的场景只能重启并修改启动参数。

`level >= AIRY_LOCK_MAX` 或 `by >= AIRY_LOCK_TRIGGER_MAX` 一律返回 `-EPERM`，防止越界索引规则表。

### 2.3 初值来源：Kconfig 而非命令行

启动顺序决定了初值只能来自 Kconfig：

```
start_kernel()
  └─ early_security_init()      → 遍历 __start_early_lsm_info，调用 airy_lock_init()
  └─ setup_arch()
  └─ parse_early_param()        → 解析 airy.lockdown=
  └─ ...
  └─ security_init()            → ordered_lsm_init()，普通 LSM 生效
```

`early_param` 的处理晚于 early LSM 的 `.init`。如果初值依赖命令行，那么在 `early_security_init()` 到 `parse_early_param()` 之间的窗口里阶梯恒为 none，early 等级的钩子无法生效。因此：

- **初值**：`CONFIG_AIRY_LOCKDOWN_DEFAULT_LEVEL`（0-3），在 `airy_lock_init()` 中通过 `airy_lock_set(..., AIRY_LOCK_BY_CONFIG)` 写入。
- **命令行**：`airy.lockdown=none|early|late|full`，只能提升，不能低于 Kconfig 默认值；请求更低等级时 `airy_lock_set()` 返回 `-EPERM`，参数解析失败并在启动日志中可见。

### 2.4 状态机

```
          config default
                │
                ▼
   ┌───────► none ──► early ──► late ──► full
   │          │        │         │        │
   │          └────────┴─────────┴────────┘
   │                  只能向右
   └── 降级请求一律 -EPERM，状态不变
```

---

## §3 Agent 接口规则表

`enum airy_lock_iface` 三个入口，规则表 `airy_lock_rules[]` 在编译期确定，运行时只读：

| 接口 | 名称 | 关闭起点 | 被关闭的操作 | 始终保留的操作 |
|------|------|---------|-------------|--------------|
| `AIRY_LOCK_IF_CAP_GRANT` | cap-grant | early | `AIRY_CAP_OP_REQUEST` 的 bootstrap 分支（`required_perms == 0`）、`airy_badge_install()` | `airy_badge_revoke()`、FREEZE、所有校验型操作 |
| `AIRY_LOCK_IF_ROVOL` | rovol | early | `AIRY_ROVOL_TIER_SET`、`AIRY_ROVOL_MGLRU_CONFIG` 等写操作 | `AIRY_ROVOL_LIST`、`AIRY_ROVOL_TIER_GET` |
| `AIRY_LOCK_IF_SCHED` | sched | early | `AIRY_SCHED_SET_POLICY`、`AIRY_SCHED_SET_BUDGET`、`AIRY_SCHED_SET_PRIORITY` | 所有 `GET_*` 与 `YIELD` |

判定方向：

```c
need = airy_lock_rules[iface].level;
cur  = airy_lock_get();
if (cur < need)
	return 0;              /* 尚未到达关闭该接口的等级 */
return -AIRY_ELOCKDOWN;    /* 已到达，拒绝 */
```

即"当前等级 **不低于** 接口要求的关闭等级"时拒绝。规则表当前三项均为 early，意味着系统一旦进入 early 及以上，agent 的扩张路径全部关闭；late 与 full 阶段新增关闭的是内核设施面（§4），不是 agent 接口面。

### 3.1 为什么把三个接口都定在 early

按威胁模型，capability 授予、RoVol 写、调度改写是 agent 获得新能力的三个入口。任何一次成功的扩张都可以绕过后续所有静态检查，因此它们的关闭时机必须早于其他面。把它们放在 late 或 full 会让系统在 early 阶段处于"内核已收紧、agent 仍可自我扩张"的矛盾状态。

### 3.2 为什么反向路径不设 gate

`airy_badge_revoke()`、RoVol 读取、调度读取在任何等级都必须可用。原因有两条：

1. **可恢复性**：错误授予的 capability 必须能被立即撤销，否则运维在 sealed 系统上无法止损。
2. **可观测性**：监控与排障依赖读取调度与 RoVol 状态，关闭读路径会让故障不可诊断。

因此规则表只描述"扩张方向"，收缩方向天然放行。

---

## §4 内核设施映射表

`security_locked_down(what)` 由 LSM 钩子 `locked_down` 分发到 `airy_lock_down()`。映射表 `airy_lock_reason[]` 以 `enum lockdown_reason` 为下标，值为关闭该设施所需的最低等级；0 表示任何等级都不关闭（保留给 `LOCKDOWN_NONE` 与未列出的值）。

| 等级 | `enum lockdown_reason` |
|------|-----------------------|
| early | `MODULE_SIGNATURE`、`DEV_MEM`、`EFI_TEST`、`KEXEC`、`BPF_WRITE_USER` |
| late | `MODULE_PARAMETERS`、`DEBUGFS`、`KCORE`、`KPROBES`、`BPF_READ_KERNEL`、`PERF`、`TRACEFS` |
| full | `HIBERNATION`、`PCI_ACCESS`、`IOPORT`、`MSR`、`ACPI_TABLES`、`DEVICE_TREE`、`PCMCIA_CIS`、`TIOCSSERIAL`、`MMIOTRACE`、`XMON_WR`、`DBG_WRITE_KERNEL`、`RTAS_ERROR_INJECTION`、`XEN_USER_ACTIONS`、`DBG_READ_KERNEL`、`XMON_RW`、`XFRM_SECRET` |

分级依据：

- **early**：能改写内核本体、绕过模块签名或替换运行镜像的设施。这些一旦被利用，后续所有等级的约束都不再有意义。
- **late**：能读取或干预内核内部状态的设施。它们在启动期可能被合法的初始化流程使用，业务就绪后不再需要。
- **full**：与具体硬件平台绑定的维护性设施（MSR、IOPORT、PCI、ACPI、设备树、串口等）。只在配置冻结后关闭，避免影响硬件热插拔与运维窗口。

`LOCKDOWN_NONE` 与未显式列出的枚举值映射为 0，语义为"不关闭"。

### 4.1 与 Linux lockdown LSM 的关系

两者是**并存**关系，不是替代关系：

| 维度 | Linux lockdown LSM | Airymax lockdown |
|------|-------------------|------------------|
| 等级 | integrity / confidentiality | none / early / late / full |
| 触发 | `lockdown=` 命令行、securityfs | `airy.lockdown=`、`/sys/kernel/security/airy/lockdown`、Kconfig 初值 |
| 可降级 | 否 | 否 |
| 钩子 | `locked_down` | `locked_down` |
| 启用方式 | `CONFIG_SECURITY_LOCKDOWN_LSM` + `CONFIG_LSM` 列表 | `CONFIG_SECURITY_AIRY` + `DEFINE_EARLY_LSM` |

两者都注册 `locked_down` 钩子时，LSM 框架按顺序调用，任一返回非 0 即拒绝。Airymax 阶梯通过 early LSM 注册，天然不受 `CONFIG_LSM` 名单过滤，因此即使发行版没有在 `CONFIG_LSM` 中列出 `airy_lockdown`，只要 `CONFIG_SECURITY_AIRY=y`，阶梯就生效。

---

## §5 接入点

代码中只有三处扩张路径调用 `airy_lock_gate()`，一处内核设施面注册 `locked_down` 钩子：

| 位置 | 调用 | 说明 |
|------|------|------|
| `security/airy/airy_cap_check.c` → `airy_uring_cmd_check()` | `airy_lock_gate(AIRY_LOCK_IF_CAP_GRANT)` | 仅 CAP_REQUEST 且 `required_perms == 0` 的 bootstrap 分支 |
| `kernel/syscalls/airy_syscalls.c` → `airy_badge_install()` | `airy_lock_gate(AIRY_LOCK_IF_CAP_GRANT)` | 参数校验之后、写入 capability 表之前 |
| `kernel/syscalls/airy_rovol.c` → `airy_rovol_serve()` | `airy_lock_gate(AIRY_LOCK_IF_ROVOL)` | 非只读操作 |
| `kernel/syscalls/airy_sched.c` → `airy_sched_serve()` | `airy_lock_gate(AIRY_LOCK_IF_SCHED)` | 仅三个改写型操作 |
| `security/airy/airy_lockdown.c` | `LSM_HOOK_INIT(locked_down, airy_lock_down)` | 内核设施面 |

### 5.1 gate 位置的选择原则

gate 必须放在**参数校验之后、状态写入之前**：

- 放在参数校验之前，非法参数会先得到 `-AIRY_ELOCKDOWN`，掩盖调用方真正的错误，使排障困难。
- 放在状态写入之后，拒绝时状态已被修改，等于没有拒绝。

`airy_badge_install()` 中 gate 在 `msg` 空指针检查之后、`dst_task` 范围检查之前，因为后者依赖 `msg` 已非空；`airy_uring_cmd_check()` 中 gate 在 `required_perms` 判定之后，因为只有 bootstrap 分支需要关闭。

### 5.2 拒绝路径不得进入 Fault 处理

`airy_uring_cmd_check()` 的常规失败路径会 `goto fault` 并触发 `AIRY_FAULT_CAP_FORGED`。lockdown 拒绝必须直接 `return ret`，不得走 `goto fault`：

| 类别 | 返回 | 是否触发 Fault | 是否进入 DSL 降级 |
|------|------|---------------|------------------|
| 安全违规（伪造 Badge、越权） | `AIRY_FAULT_*` | 是 | 是 |
| 策略拒绝（lockdown gate） | `-AIRY_ELOCKDOWN` | 否 | 否 |

混淆两者会把运维的合法收敛动作误报为攻击，污染审计日志，并触发不必要的 Ring 冻结。

---

## §6 配置、命令行与 securityfs

### 6.1 Kconfig

| 选项 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `CONFIG_SECURITY_AIRY` | bool | n | 总开关；为 n 时 `airy_lock_gate()` 以内联返回 0 降级 |
| `CONFIG_AIRY_LOCKDOWN_DEFAULT_LEVEL` | int (0-3) | 0 | 阶梯初值；发行版镜像应设为 1 |

### 6.2 命令行

```
airy.lockdown=early
```

在 `parse_early_param()` 阶段解析。合法值为 `none`、`early`、`late`、`full`；非法值返回 `-EINVAL`；低于 Kconfig 初值时返回 `-EPERM`，两者都会在启动日志中显示为无法识别的参数。

### 6.3 securityfs

```
/sys/kernel/security/airy/lockdown
```

- 权限 0600，仅 root 可读写。
- 读：返回当前等级名（`none`/`early`/`late`/`full`），带换行。
- 写：接受等级名，允许尾部单个换行；非法名返回 `-EINVAL`；降级返回 `-EPERM`。
- 注册时机：`late_initcall(airy_lock_fs)`。选择 late 而非 `core_initcall`，是为了免疫 `security/inode.o` 与 `security/airy/` 之间的链接顺序，同时仍早于任何用户态任务。
- `CONFIG_SECURITYFS=n` 时整个注册函数在编译期短路并返回 0，`securityfs_create_*` 的 `ERR_PTR(-ENODEV)` 桩不会被触达。

---

## §7 错误语义

| 场景 | 返回 | 说明 |
|------|------|------|
| agent 接口被 gate 拒绝 | `-AIRY_ELOCKDOWN`（261） | 新增注册码，位于 Lockdown 子空间 |
| 内核设施被 `locked_down` 拒绝 | `-EPERM` | 遵循 Linux `security_locked_down()` 既有约定，调用点普遍按 `-EPERM` 处理 |
| 降级请求 | `-EPERM` | 状态不变 |
| 非法等级名 | `-EINVAL` | 参数解析失败 |
| 越界 iface / reason | `-AIRY_ELOCKDOWN` / `-EPERM` + `WARN_ON_ONCE` | 属于编码错误，不应在正常运行中出现 |

`AIRY_ELOCKDOWN` 的注册见 [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) §2.13。

---

## §8 可观测性

`CONFIG_AIRY_AGENT_TRACING=y` 时提供两个事件：

| 事件 | 参数 | 触发点 |
|------|------|--------|
| `airy:lockdown_change` | `prev`、`level`、`trigger` | 等级实际提升时；`trigger` 取 0=config、1=command line、2=securityfs |
| `airy:lockdown_reject` | `domain`、`id`、`required`、`active` | agent 接口被 gate 拒绝（`domain=0`）、内核设施被拒绝（`domain=1`） |

设计要点：

- 只在**实际状态变化**时发 `change`，幂等设置不发，避免日志噪声。
- `reject` 的 `required` 与 `active` 同时携带，便于直接判断是规则配置错误还是调用方越界。
- `reject` 不携带调用者身份，身份信息由上层 syscall 审计事件关联，避免在安全关键路径做额外查表。

---

## §9 测试

`security/airy/airy_lockdown_test.c` 提供 10 个 KUnit 用例，套件名 `airy_lockdown`，每个用例前调用 `airy_lock_clr()` 复位到 none：

| 用例 | 覆盖点 |
|------|--------|
| `lock_none` | 初值为 none，所有接口放行 |
| `lock_raise` | none → early 提升成功，`get()` 反映新值 |
| `lock_no_lower` | early → none 返回 `-EPERM`，状态不变 |
| `lock_same` | 重复设置同等级返回 0 |
| `lock_bad` | 越界 level / trigger 返回 `-EPERM` |
| `lock_names` | 四个合法名解析、`sealed` 拒绝、NULL 参数拒绝 |
| `lock_gate` | none 下三个接口全放行；early 下三个接口全部 `-AIRY_ELOCKDOWN` |
| `lock_early` | `KEXEC` 拒绝；`DEBUGFS`、`XFRM_SECRET`、`NONE` 放行 |
| `lock_late` | `KEXEC`、`DEBUGFS` 拒绝；`XFRM_SECRET`、`NONE` 放行 |
| `lock_full` | `KEXEC`、`DEBUGFS`、`XFRM_SECRET` 拒绝；`NONE` 放行 |

`airy_lock_clr()` 仅在 `CONFIG_AIRY_KUNIT=y` 时编译，生产内核不暴露复位入口。

---

## §10 相关文档

- [01-lsm-framework.md](01-lsm-framework.md)：LSM 框架与 `early_security_init()`
- [03-capability-model.md](03-capability-model.md)：capability 授予与撤销
- [07-airy-lsm-design.md](07-airy-lsm-design.md)：`airy_uring_cmd_check()` 钩子
- [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md)：`AIRY_ELOCKDOWN` 注册
- [130-roadmap/02-milestones-and-timeline.md](../130-roadmap/02-milestones-and-timeline.md) §3.5.1：分阶段策略的里程碑归属
- [10-architecture/08-threat-model.md](../10-architecture/08-threat-model.md)：锁定对象与攻击面

---

## §11 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0.1 | 2026-10-18 | 初始版本：四级阶梯、三接口规则表、`enum lockdown_reason` 映射、四处 gate 接入、命令行与 securityfs 入口、两个 ftrace 事件、10 个 KUnit 用例 |

---

> **文档结束** | Airymax Lockdown 阶梯设计 v1.0.1 | 维护者：开源极境工程与规范委员会 | "From data intelligence emerges."
