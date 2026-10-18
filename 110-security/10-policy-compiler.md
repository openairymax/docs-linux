Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Airymax 安全策略编译器与决策表运行时设计
> **文档定位**：AirymaxOS 声明式安全策略（`.adl` → 纯 C 决策表）的唯一权威详细设计，定义编译期与运行时的职责边界、决策表数据模型、钩子字段契约、四值裁决语义、注册校验、可观测性与测试矩阵\
> **文档版本**：v1.0.2\
> **最后更新**： 2026-10-18\
> **上级文档**：[AirymaxOS 总览](../README.md) + [110-security/README.md](README.md)\
> **设计依据**：本文件为 SSoT（Single Source of Truth），`tools/dsl/` 的生成契约、`security/airy/airy_policy.c` 的注册与裁决语义、`security/airy/airy_lsm.c` 的字段采集、`securityfs` 的 `policy` 接口均以本文件为唯一权威定义

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Airymax 安全策略编译器与决策表运行时** 的唯一权威源。`.adl` 语言契约、生成头文件的机制数据模型、七个 LSM 钩子的字段向量、四值裁决的合并顺序、注册校验规则、`AIRY_EPOLICY` 错误语义、`/sys/kernel/security/airy/policy` 的输出格式、`airy:airy_policy_decision` 事件均以本文件为唯一权威定义。其他文档只能引用本文件，禁止重新定义字段名或裁决合并规则。
>
> [SC] 侧对应物：`kernel/include/airymax/airy_policy.h`（机制类型、钩子枚举、运行时原型）、`tools/dsl/templates/dsl_decision_table.h.j2`（生成器模板）、`kernel/include/uapi/linux/airymax/security_types.h`（`enum airy_verdict`）。文档与头文件出现分歧时，以头文件为编译期事实，并同步修订本文件。

---

## 文档信息卡

- **目标读者**：安全模块开发者、策略作者、发行版集成者
- **前置知识**：理解 [07-airy-lsm-design.md](07-airy-lsm-design.md) 的七个 LSM 钩子、[01-lsm-framework.md](01-lsm-framework.md) 的 `security_hook_list` 注册、[30-interfaces/06-codegen-pipeline.md](../30-interfaces/06-codegen-pipeline.md) 的"机制与策略分离（K-1）"
- **预计阅读时间**：30 分钟
- **核心概念**：编译期生成、运行时机制、首匹配决策表、字段向量、四值裁决、策略违规与策略拒绝的区分
- **复杂度标识**：高级

---

## §1 设计目标

### 1.1 为什么需要声明式策略

AirymaxOS 的 agent 安全语义散落在七个 LSM 钩子里：哪些 `uring_cmd` 操作码非法、哪些 `task_alloc` 组合是 capability 伪造、哪些 `file_open` 路径类需要拒绝。若把这些判断全部写成 C 分支，每增加一条策略都要改内核代码、重新审计控制流，策略与机制无法分别演进。

声明式策略把判断从代码中剥离：策略作者只写"当什么条件成立时给出什么裁决"，编译器把它变成常量决策表，内核只保留一张通用的表驱动执行机制。这样做的收益是可审计（每条规则是一个数据行）、可验证（无循环、无动态分配、无间接跳转）、可回滚（策略版本即数据版本）。

### 1.2 为什么不用 BPF

上游的 BPF LSM 允许把策略编译成 eBPF 程序挂到钩子上。AirymaxOS 明确不采用该方案，原因与 [110-security/README.md](README.md) §2.1 的纯 C LSM 选型一致：

| 维度 | BPF LSM | 纯 C 决策表 |
|------|---------|------------|
| 验证器语义 | 依赖版本，行为随内核演进 | 无验证器，控制流在编译期确定 |
| 审计 | 需要额外工具反汇编 | 表行即策略，可直接打印 |
| 启动依赖 | 需要 `CONFIG_BPF`、JIT、map 生命周期 | 无额外子系统依赖 |
| 故障模式 | 程序加载失败、map 缺页 | 注册期一次校验，运行期无分配 |

因此本设计把"编译"的目标从 eBPF 字节码改为**纯 C 常量数组**，由内核在启动期注册、在钩子内执行。

### 1.3 与 L-03、B-08 的边界

| 工作包 | 职责 | 产物 |
|--------|------|------|
| L-03（已完成） | 编译期：`.adl` 词法/语法/语义分析、代码生成 | `tools/dsl/*.py` + 生成头 |
| **B-06（本文件）** | **运行期：注册、钩子绑定、字段采集、裁决聚合、fault、内省** | `security/airy/airy_policy.c` + `airy_policy.h` |
| B-08（后继） | 策略下发：把生成头与 agentrt 策略包接入发行版构建 | 构建集成与策略清单 |

L-03 只负责"把策略写成合法的 C 数据"，不理解内核钩子的运行时语义；B-06 只负责"执行任何合法的表"，不解析 `.adl`。两者之间唯一的契约是 `struct airy_policy_table`。

---

## §2 分层架构

### 2.1 编译期（L-03）

```
policy.adl ──► dsl_lexer ──► dsl_parser ──► dsl_semantics ──► dsl_codegen ──► policy_tables.h
                                                                  │
                                                     dsl_decision_table.h.j2
```

生成头在 `__KERNEL__` 下引入 `airy_policy.h` 取得机制类型；在用户态（pytest、工具链自检）下自带一份等价的机制块，以便脱离内核头文件编译。两条分支共用同一组结构体定义，`AIRY_POLICY_MECHANISM_DEFINED` 守卫保证同一翻译单元内只定义一次。

### 2.2 运行期（B-06）

```
LSM hook (airy_lsm.c)
   │  采集字段向量 fields[AIRY_POLICY_MAX_FIELDS]
   ▼
airy_policy_check(hook, fields, &fault)
   │  遍历注册槽位，按 hook 过滤
   │  槽位内 field_map 把表字段下标映射到钩子字段下标
   ▼
首匹配行 → 四值裁决 → 跨表合并 → 统计 / trace
   │
   ▼
0 或 -EACCES（fault 经出参返回）
```

运行时只做四件事：**注册校验**、**字段重排**、**首匹配求值**、**聚合与记账**。它不包含任何具体策略判断。

### 2.3 机制与策略分离（K-1）

- 生成头是**策略**：常量数组，随策略版本变化。
- `airy_policy.c` 是**机制**：不引用任何具体表名、不假设任何字段值。
- 内核不内建默认策略。没有注册任何表的系统等价于"全部 ALLOW"，这是显式的空策略语义，不是桩实现；策略由 B-08 的发行版构建注入。

---

## §3 DSL 契约摘要

完整语言定义见 `tools/dsl/README` 与 `dsl_types.py`，此处只固定与运行时相关的部分。

### 3.1 顶层结构

```
version "1.0.1";
table <name> {
	hook = <hook_name>;
	input <field> : <u8|u16|u32|u64>;
	rule <number> { when <cond>; verdict <VERDICT>; fault <code>; }
	default verdict <VERDICT>;
}
```

### 3.2 钩子名

| `.adl` 钩子名 | 运行时枚举 | 可否阻断 |
|---------------|-----------|---------|
| `uring_cmd` | `AIRY_POLICY_HOOK_URING_CMD` | 是 |
| `task_alloc` | `AIRY_POLICY_HOOK_TASK_ALLOC` | 是 |
| `task_free` | `AIRY_POLICY_HOOK_TASK_FREE` | 否（返回 `void`） |
| `task_kill` | `AIRY_POLICY_HOOK_TASK_KILL` | 是 |
| `file_open` | `AIRY_POLICY_HOOK_FILE_OPEN` | 是 |
| `inode_alloc_security` | `AIRY_POLICY_HOOK_INODE_ALLOC` | 是（分配失败） |
| `inode_free_security` | `AIRY_POLICY_HOOK_INODE_FREE` | 否（返回 `void`） |

`task_free` 与 `inode_free_security` 是观测钩子：`DENY` 无法阻止已经发生的释放，运行时把该裁决记为策略缺陷（计入 `deny` 与 `fault` 统计并发出 trace），调用方忽略返回值。策略作者应在这两个钩子上只使用 `ALLOW`/`AUDIT`/`COMPLAIN`。

### 3.3 比较算子

`==`(0) `!=`(1) `<`(2) `<=`(3) `>`(4) `>=`(5) `&`(6)。`&` 的语义是"`value` 的全部置位在 `lhs` 中都为 1"，即 `(lhs & value) == value`。未知算子求值为 `false`。

### 3.4 四值裁决

| 值 | 名称 | 运行期行为 |
|----|------|-----------|
| 0 | `AIRY_VERDICT_ALLOW` | 放行，不记录 |
| 1 | `AIRY_VERDICT_DENY` | 拒绝，返回 `-EACCES` |
| 2 | `AIRY_VERDICT_AUDIT` | 放行，计审计并发出 trace |
| 3 | `AIRY_VERDICT_COMPLAIN` | 放行，记录"本应拒绝"（对齐 AppArmor complain 学习模式） |

裁决枚举唯一权威定义在 [security_types.h](../../agent-linux/kernel/include/uapi/linux/airymax/security_types.h) 的 `enum airy_verdict`；生成头与运行时都**不得重新定义**这些常量（OS-IRON-014）。

---

## §4 决策表数据模型

```c
struct airy_policy_cond {
	u8  field;	/* 表内字段下标，0 .. field_count-1 */
	u8  op;		/* enum airy_policy_cmp_op */
	u64 value;
};

struct airy_policy_row {
	const struct airy_policy_cond *conds;
	u16 cond_count;
	u8  verdict;	/* enum airy_verdict */
	u16 fault;	/* AIRY_FAULT_* 或 0 */
};

struct airy_policy_table {
	const char *name;
	const char *hook;
	const struct airy_policy_row *rows;
	u16 row_count;
	u8  field_count;
	const char *const *field_names;
	u8  default_verdict;
};
```

约束：

- 行按数组顺序**首匹配**：第一条 `conds` 全部成立的行使该表给出裁决；没有行匹配时使用 `default_verdict`。
- 一张表只绑定一个钩子；同一钩子可以注册多张表，按注册顺序求值。
- 表是只读常量数据，注册后内核不再修改；运行时状态（统计）与表分离存放。
- `fault` 只在该行胜出时随裁决上报，取第一个产生 `DENY`/`COMPLAIN` 的行的非零值。

---

## §5 钩子字段契约

字段名是策略与内核之间的**唯一**字符串契约。注册时按钩子解析；解析不到的字段名使注册失败，不会静默变成 0。

| 钩子 | 下标 | 字段名 | 类型 | 来源 |
|------|------|--------|------|------|
| `uring_cmd` | 0 | `op` | u32 | `ioucmd->cmd_op` |
| | 1 | `cmd_flags` | u32 | `ioucmd->flags` |
| | 2 | `agent_id` | u32 | 调用者 `airy_task_sec.agent_id` |
| `task_alloc` | 0 | `clone_flags` | u64 | 钩子入参 `clone_flags` |
| | 1 | `task_type` | u8 | 0 = 内核线程（`PF_KTHREAD`），1 = 用户任务 |
| | 2 | `agent_id` | u32 | 新任务 `airy_task_sec.agent_id`（分配时通常为 0） |
| `task_free` | 0 | `agent_id` | u32 | 被释放任务 |
| | 1 | `agent_state` | u8 | `enum airy_agent_state` |
| `task_kill` | 0 | `sig` | u32 | 信号编号 |
| | 1 | `target_agent` | u32 | 目标 `agent_id` |
| | 2 | `target_state` | u8 | 目标 `agent_state` |
| | 3 | `caller_agent` | u32 | 调用者 `agent_id` |
| `file_open` | 0 | `open_flags` | u32 | `file->f_flags` |
| | 1 | `acc_mode` | u32 | `file->f_mode & FMODE_*` |
| | 2 | `agent_id` | u32 | 调用者 `agent_id` |
| `inode_alloc_security` | 0 | `inode_type` | u16 | `inode->i_mode & S_IFMT` |
| `inode_free_security` | 0 | `inode_type` | u16 | `inode->i_mode & S_IFMT` |

规则：

- 一张表的 `field_count` 不得超过 `AIRY_POLICY_MAX_FIELDS`（8），字段名不得在同一张表内重复。
- 未采集的字段在向量中恒为 0；策略作者不应依赖未列出的字段。
- 字段契约变更属于 ABI 变更：必须同时修改本文件 §5、`airy_policy.c` 的 `airy_pol_fields[][]` 与采集点。

---

## §6 裁决语义与合并

### 6.1 单表求值

```
for row in table.rows:
	if all(cond in row.conds: cmp(fields[cond.field], cond.op, cond.value)):
		return row.verdict, row.fault
return table.default_verdict, 0
```

### 6.2 跨表合并

同一钩子的多张表各自给出裁决，按严格度合并：

```
DENY > COMPLAIN > AUDIT > ALLOW
```

- 任一表给出 `DENY` → 最终 `DENY`，返回 `-EACCES`。
- 无 `DENY` 且有 `COMPLAIN` → 最终 `COMPLAIN`，放行并记录。
- 无 `DENY`/`COMPLAIN` 且有 `AUDIT` → 最终 `AUDIT`，放行并记录。
- 全部 `ALLOW` → 放行。

合并采用"悲观收敛"：任何一张表都不能放宽另一张表的拒绝，避免策略叠加时出现权限放大。

### 6.3 与既有钩子逻辑的次序

现有钩子（agent 生命周期、Badge 校验、Ring 冻结）是**机制硬约束**，先执行；决策表是**部署策略**，后执行：

1. 机制检查失败 → 直接返回既有错误码，不求值策略。
2. 机制检查通过 → 求值策略；`DENY` 返回 `-EACCES`。
3. `COMPLAIN`/`AUDIT` 不改变返回值，只记账与 trace。

因此策略只能收紧、不能放宽机制；这也是"策略不可绕过内核不变量"的形式化表述。

### 6.4 策略拒绝与安全违规

| 情况 | 返回 | fault | 语义 |
|------|------|-------|------|
| 策略 `DENY`，行带 fault | `-EACCES` | 行 fault | 策略拒绝，同时是安全违规信号 |
| 策略 `DENY`，行无 fault | `-EACCES` | 0 | 纯策略拒绝 |
| 机制检查失败 | 既有错误码 | 由机制决定 | 安全违规 |
| `COMPLAIN` | 0 | 行 fault（如非零） | 学习模式，记录本应拒绝 |

`-EACCES` 的选择与既有 `file_open` 拒绝一致；`fault` 由调用方决定是否上报 Micro-Supervisor，运行时不自行升级为 `-EPERM`。

---

## §7 注册与校验

`airy_policy_register()` 在 LSM 初始化期调用，注册后表不可变。校验全部在注册期完成，运行期不再做合法性检查。

| 检查 | 失败返回 |
|------|---------|
| `table`/`name`/`hook`/`rows`/`field_names` 为空 | `-EINVAL` |
| 钩子名无法解析 | `-EINVAL` |
| `field_count` 为 0 或 > `AIRY_POLICY_MAX_FIELDS` | `-EINVAL` |
| 字段名在该钩子契约中不存在 | `-EINVAL` |
| 同一表内字段名重复 | `-EINVAL` |
| `row_count` 为 0 | `-EINVAL` |
| 行 `cond_count > 0` 但 `conds` 为空 | `-EINVAL` |
| 条件 `field >= field_count` 或 `op > AIRY_CMP_MASK` | `-EINVAL` |
| 行裁决或默认裁决 > `AIRY_VERDICT_COMPLAIN` | `-EINVAL` |
| 同名表已注册 | `-EEXIST` |
| 槽位已满（`AIRY_POLICY_MAX_TABLES` = 16） | `-ENOSPC` |

注册顺序即求值顺序，必须稳定：同一构建产物在每次启动时按相同顺序注册，统计与 trace 才可复现。

---

## §8 可观测性

### 8.1 securityfs

`/sys/kernel/security/airy/policy`（`0400`，root 只读）输出一张表：

```
# name hook rows evals allow deny audit complain fault
uring_guard uring_cmd 2 1024 1021 2 1 0 0
```

- `evals` 为该表被求值的次数；`allow/deny/audit/complain` 为胜出裁决计数；`fault` 为带非零 fault 的胜出次数。
- 目录 `airy` 由 `airy_fs.c` 统一创建（与 `lockdown` 共用），避免两个 `late_initcall` 争抢同一目录项。

### 8.2 trace

`airy:airy_policy_decision` 只在裁决非 `ALLOW` 或 fault 非零时发出：

| 字段 | 含义 |
|------|------|
| `hook` | `enum airy_policy_hook` |
| `table` | 注册槽位下标（对应 securityfs 行序） |
| `verdict` | `enum airy_verdict` |
| `fault` | 行 fault |

`ALLOW` 路径不产生 trace，避免在文件打开等高频钩子上引入无界开销。

### 8.3 统计的并发语义

统计是 `atomic64_t`，允许近似：读侧用 `atomic64_read()` 取得快照，不保证跨字段一致。统计不参与任何判定，只用于观测。

---

## §9 并发与生命周期

- 注册只在启动期（LSM `init`）发生；`airy_policy_check()` 用 `smp_load_acquire()` 读取槽位数，注册用 `smp_store_release()` 发布，保证读者看到完整槽位。
- 槽位数组是静态存储，不分配、不释放，不存在 use-after-free 与内存压力路径。
- 求值路径无锁、无睡眠、无分配；最坏情况是 `MAX_TABLES × row_count × cond_count` 次比较，全部为常量上界。
- KUnit 通过 `airy_policy_reset()` 与 `airy_pol_stat()`（仅 `CONFIG_AIRY_KUNIT`）清空注册表并读取单表计数快照，生产路径不存在这两个符号。`airy_pol_stat()` 的计数字段与 §8.1 的 securityfs 行一一对应，测试因此可以在不读取 `seq_file` 的前提下断言裁决分类与 fault 归属。

---

## §10 测试矩阵

`security/airy/airy_policy_test.c`（KUnit）覆盖：

| 用例 | 断言 |
|------|------|
| `policy_cmp` | 七种算子的真值表；未知算子为 false |
| `policy_match` | 多条件合取 |
| `policy_first` | 首匹配优先于后续行 |
| `policy_default` | 无匹配时回落默认裁决 |
| `policy_deny` | `DENY` 返回 `-EACCES` 且 fault 出参正确 |
| `policy_allow` | `ALLOW` 返回 0 且不产生 fault |
| `policy_audit` | `AUDIT` 放行并计入 audit |
| `policy_complain` | `COMPLAIN` 放行并计入 complain |
| `policy_merge` | 多表合并取最严格裁决 |
| `policy_fault` | 首个 `DENY` 的 fault 被保留 |
| `policy_field` | 未知字段名注册失败 |
| `policy_hook` | 未知钩子名注册失败 |
| `policy_verdict` | 越界裁决注册失败 |
| `policy_dup` | 同名表重复注册返回 `-EEXIST` |
| `policy_full` | 槽位耗尽返回 `-ENOSPC` |
| `policy_reset` | 测试复位清空注册表与统计 |

集成验证：`tools/dsl/Makefile` 的 `check` 目标保证生成头零漂移；`make O=/tmp/airy-kbuild` 在源码区外编译；`kunit.py` 运行完整套件。

---

## §11 错误码

| 符号 | 值 | 含义 |
|------|----|------|
| `-EACCES` | -13 | 策略 `DENY` |
| `-EINVAL` | -22 | 表结构或字段契约非法 |
| `-EEXIST` | -17 | 同名表已注册 |
| `-ENOSPC` | -28 | 注册槽位耗尽 |

策略编译期的错误码（`AIRY_DSL_SOURCE_ERROR` 等）由 L-03 定义，不进入内核。

---

## §12 相关文档

- [07-airy-lsm-design.md](07-airy-lsm-design.md)：七个 LSM 钩子与 Badge 机制
- [09-lockdown.md](09-lockdown.md)：与 `airy` securityfs 目录共用的收敛机制
- [01-lsm-framework.md](01-lsm-framework.md)：`security_hook_list` 与钩子排序
- [30-interfaces/06-codegen-pipeline.md](../30-interfaces/06-codegen-pipeline.md)：机制与策略分离（K-1）、`@generated` 约束
- [50-engineering-standards/90-terminology.md](../50-engineering-standards/90-terminology.md)：`AIRY_VERDICT_*` 术语唯一来源
- [90-observability/README.md](../90-observability/README.md)：trace 与审计通道

---

## §13 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0.1 | 2026-10-18 | 初始版本：确立 L-03 编译期 / B-06 运行期边界、决策表数据模型、七钩子字段契约、四值裁决合并规则、注册校验、securityfs 与 trace |
| v1.0.2 | 2026-10-18 | B-06 收口：七钩子接入策略求值、生成头复用机制头、补充 `airy_pol_stat()` 测试接口说明与 KUnit 16 用例矩阵 |
