Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agent-linux（AirymaxOS）静态分析
> **文档定位**：agent-linux（AirymaxOS）测试工程体系第 5 卷——编译时静态分析（Static Analysis）。本卷规定 Sparse（内核静态分析工具）、Smatch（语义匹配）、Coccinelle（语义补丁检测）、Clang Static Analyzer、Coverity 扫描集成、checkpatch.pl 扩展、[SC] 头文件编译器无关性检查（`check-uapi-compiler-agnostic.sh`）的启用模型、规则配置与 `kernel-verify` workflow 集成。\
> **文档版本**：v1.1.0\
> **最后更新**： 2026-10-19\
> **上级文档**：[80-testing README](README.md)\
> **同源映射**：agentrt 7 层验证 L5（静态分析）+ Linux 6.6 内核基线 `scripts/sparse/`、`scripts/coccinelle/`、`scripts/checkpatch.pl`\
> **理论根基**：Linux 6.6 内核基线静态分析思想 + Airymax 五维正交 24 原则（E-8 可测试性 / S-1 反馈闭环 / A-4 完美主义）\
> **核心约束**：IRON-9 v3 [SC] 共享契约层——`check-uapi-compiler-agnostic.sh` 强制校验 12 个 [SC] 头文件编译器无关性，禁止使用 GCC/Clang 专属扩展。

---

## 0. 章节定位

本卷是 agent-linux 测试工程 10 主题文档中的第 5 卷，回答"编译时代码错误怎么发现"。它在 04-dynamic-analysis（运行时检测）与 06-coverage-metrics（覆盖率度量）之间形成编译时分析层：

- **上游依赖**：README 定义"测试体系分层"——L5 静态分析由本卷展开；50-engineering-standards/06-toolchain-and-automation 定义"7 层验证"——本卷对应第 11 层（编译时分析层）。
- **下游依赖**：06-coverage-metrics 定义"代码覆盖率怎么测"；10-formal-verification 定义"形式化验证怎么做"——本卷的静态分析结果与形式化验证互补。

本卷所有强制规则均赋予 **OS-TEST** / **OS-KER** / **OS-STD** 编号，与 50-engineering-standards/09-ssot-registry.md 的"SSoT 规则编号注册表"对齐。

### 0.1 关键术语

| 术语 | 定义 |
|------|------|
| Sparse | Linux 内核官方静态分析工具，由 Linus Torvalds 创建 |
| Smatch | 语义匹配静态分析工具，由 Dan Carpenter 维护 |
| Coccinelle | 语义补丁工具，检测代码模式（如 `NULL` 解引用） |
| Clang Static Analyzer | Clang 编译器内置的静态分析器 |
| Coverity | 商业静态分析服务，扫描内核代码 |
| `checkpatch.pl` | 内核代码风格检查脚本 |
| `check-uapi-compiler-agnostic.sh` | agent-linux 专属脚本，校验 [SC] 头文件编译器无关性 |
| `kernel-verify` workflow | CI 流水线的静态分析阶段 |

---

## 1. 静态分析模型总览

### 1.1 起源与定位

内核静态分析是 Linux 6.6 内核基线中"在编译时（或独立阶段）通过工具分析代码模式发现错误"的机制。其设计目标有三：**编译时反馈**（无需运行代码即可发现错误）、**零运行开销**（不影响生产部署）、**模式覆盖**（检测人眼易遗漏的代码模式）。

agent-linux 完整继承 Linux 6.6 内核基线的静态分析工具链（Sparse/Smatch/Coccinelle/checkpatch.pl），不修改任何上游源文件。agent-linux 专属静态分析以独立 `scripts/airy_*.sh` 脚本形式驻留于 `scripts/`，遵循 IRON-9 v3 [IND] 独立实现层原则。

```mermaid
flowchart TB
    subgraph "Linux 6.6 静态分析框架（继承层）"
        A["源码 + 配置<br/>kernel/ + airy_defconfig"]
        B1["Sparse<br/>scripts/sparse/"]
        B2["Smatch<br/>scripts/smatch/"]
        B3["Coccinelle<br/>scripts/coccinelle/"]
        B4["checkpatch.pl<br/>scripts/checkpatch.pl"]
        B5["Clang Static Analyzer<br/>clang --analyze"]
        C["报告<br/>warnings.txt / html"]
    end
    A --> B1
    A --> B2
    A --> B3
    A --> B4
    A --> B5
    B1 --> C
    B2 --> C
    B3 --> C
    B4 --> C
    B5 --> C
    subgraph "agent-linux 扩展层（[IND]）"
        D["airy_checkpatch_ext.pl<br/>checkpatch 扩展规则"]
        E["check-uapi-compiler-agnostic.sh<br/>[SC] 头文件编译器无关性校验"]
        F["airy_sparse_rules<br/>Agent 专属 Sparse 规则"]
    end
    B4 -.->|扩展| D
    B1 -.->|规则注入| F
    A -.->|12 个 [SC] 头文件| E
    D --> C
    E --> C
    F --> C
    subgraph "外部商业服务"
        G["Coverity Scan<br/>coverity.com"]
    end
    C -.->|每周同步| G
```

### 1.2 静态分析运行载体

| 载体 | 工具 | 触发时机 | 适用场景 |
|------|------|---------|---------|
| 开发者本地 | Sparse + Smatch | `make C=1` / `make CHECK=smatch` | 即时反馈 |
| CI PR | checkpatch + Sparse + Coccinelle | 每次 push | 强制阻断 |
| CI nightly | Smatch + Coverity | 每日 | 大规模扫描 |
| Coverity Scan | Coverity | 每周同步 | 商业工具深度扫描 |

**OS-TEST-060**：CI PR 阶段必须运行 checkpatch + Sparse + Coccinelle 三项静态分析；任一工具报告 `error` 级别问题即阻断 PR。

**OS-KER-120**：agent-linux 专属模块（`security/airy/`、`kernel/superv/`、`kernel/syscalls/`、`kernel/corekern/`、`kernel/log/`、`kernel/ipc/`、`include/airymax/`、`include/uapi/linux/airymax/`，以及 `init/airy_core.c`、`include/trace/events/airy.h`）必须通过 `airy_checkpatch_ext.pl` 扩展规则；扩展规则比上游 checkpatch 更严格（如禁止 `printk` 必须用 `pr_*`）。

---

## 2. Sparse：内核静态分析工具

### 2.1 Sparse 工作原理

Sparse 由 Linus Torvalds 于 2003 年创建，是 Linux 内核官方静态分析工具。其核心能力：

| 检测项 | 修饰符 / 注解 | 说明 |
|--------|--------------|------|
| 类型限定符 | `__bitwise` | 防止不同 typedef 之间的赋值 |
| 地址空间 | `__user` / `__iomem` / `__percpu` | 检测地址空间混淆 |
| RCU 注解 | `__rcu` | 检测 RCU 指针的解引用时机 |
| 锁注解 | `__must_hold` / `__acquires` / `__releases` | 检测锁获取/释放上下文 |
| 上下文注解 | `__must_hold` / `__cond_locks` | 检测上下文条件 |
| NULL 解引用 | （自动） | 跟踪指针 NULL 状态 |

### 2.2 Sparse 启用方式

```bash
# 编译时启用 Sparse（每个 .c 文件单独分析）
make C=1                    # 仅重新编译的文件
make C=2                    # 所有文件

# 指定 Sparse 路径
make CHECK=/usr/bin/sparse

# 启用额外警告
make C=1 CF="-Wbitwise -Wno-bitwise-pointer -Wcontext -Wdecl -Wno-transparent-union"
```

### 2.3 Sparse 在 agent-linux 的应用

agent-linux 在 `airy_defconfig` 中启用 Sparse 注解：

```c
/* include/uapi/linux/airymax/airy_types.h */
#ifndef _AIRY_TYPES_H
#define _AIRY_TYPES_H

#include <linux/types.h>
#include <linux/compiler_types.h>

/* __user 指针注解：检测 user/kernel 指针混淆 */
struct airy_ipc_msg __user *airy_ipc_user_msg;

/* __iomem 指针注解：检测 I/O 内存误用 */
void __iomem *airy_dev_mmio_base;

/* __rcu 指针注解：检测 RCU 临界区外的解引用 */
struct airy_agent_state __rcu *airy_agent_rcu;

/* __bitwise 类型：防止 typedef 之间赋值 */
typedef struct { int val; } __bitwise airy_agent_id_t;
typedef struct { int val; } __bitwise airy_cap_id_t;
typedef struct { int val; } __bitwise airy_token_t;

/* 锁注解：检测锁获取上下文 */
void airy_ipc_send(struct airy_ipc_msg *msg)
    __must_hold(&airy_ipc_ring_lock);

void airy_agent_state_set(int agent_id, int state)
    __acquires(&airy_agent_state_lock)
    __releases(&airy_agent_state_lock);

#endif /* _AIRY_TYPES_H */
```

**OS-TEST-061**：agent-linux 专属模块必须使用 `__user`/`__iomem`/`__rcu`/`__bitwise` 注解；CI PR 阶段的 Sparse 报告中，`warning: incorrect type in assignment` 类警告即阻断 PR。

**OS-KER-121**：`airy_agent_id_t` / `airy_cap_id_t` / `airy_token_t` 必须为 `__bitwise` 类型；CI 通过 Sparse 验证三者之间无赋值，违反即 PR 驳回。

### 2.4 `airy_sparse_rules`：Agent 专属 Sparse 规则

agent-linux 扩展 Sparse 规则集，位于 `scripts/airy_sparse_rules/`：

| 规则 ID | 检测内容 | 严重性 |
|--------|---------|--------|
| `AIRY-SPARSE-001` | `airy_agent_state` 字段未通过 `READ_ONCE()`/`WRITE_ONCE()` 访问 | warning |
| `AIRY-SPARSE-002` | `airy_ipc_ring` 操作未在 `airy_ipc_ring_lock` 保护下 | error |
| `AIRY-SPARSE-003` | `airy_token_budget` 字段未通过原子操作访问 | error |
| `AIRY-SPARSE-004` | `airy_cap_check()` 调用未在 `airy_cap_cache_lock` 保护下 | warning |
| `AIRY-SPARSE-005` | `airy_lsm_hook` 注册未使用 `security_hook_list` | error |

---

## 3. Smatch：语义匹配静态分析

### 3.1 Smatch 工作原理

Smatch 由 Dan Carpenter 维护，扩展 Sparse 的能力，进行过程间分析（interprocedural analysis）与跨函数跟踪。其核心能力：

- **NULL 解引用跟踪**：跨函数跟踪指针 NULL 状态
- **缓冲区溢出**：跟踪数组索引与缓冲区大小
- **锁平衡检查**：检测锁获取/释放不平衡
- **未初始化变量**：检测未初始化变量的使用
- **范围检查**：检测整数范围越界

### 3.2 Smatch 启用方式

```bash
# 安装 Smatch
git clone https://github.com/error27/smatch.git
cd smatch && make && make install

# 编译时启用 Smatch
make CHECK=smatch C=1

# 仅运行 Smatch（不编译）
smatch_scripts/build_kernel_data.sh
smatch_scripts/check_kernel.sh
```

### 3.3 Smatch 报告解析

```
kernel/corekern/ipc/airy_ipc_ring.c:234 airy_ipc_send() error: potential NULL dereference 'msg->payload'.
kernel/corekern/taskflow/airy_task_lifecycle.c:456 airy_task_alloc() warn: variable 'task->state' uninitialized.
security/airy/airy_cap_check.c:789 airy_cap_check() error: buffer overflow 'cap_array' 41 <= 41.
```

**OS-TEST-062**：CI nightly 必须运行 Smatch 全量扫描；`error: ` 级别报告即标记 nightly 失败，`warn: ` 级别报告汇总至 issue 但不阻断。

---

## 4. Coccinelle：语义补丁检测

### 4.1 Coccinelle 工作原理

Coccinelle 由 Julia Lawall 维护，通过语义补丁（semantic patch，`.cocci` 文件）检测代码模式。其优势在于"模式即规则"——开发者无需理解编译器内部，编写类似 diff 的语法即可检测模式。

### 4.2 Coccinelle 启用方式

```bash
# 运行所有 Coccinelle 规则
make coccicheck

# 指定模式
make coccicheck MODE=report     # 仅报告（默认）
make coccicheck MODE=patch      # 生成补丁
make coccicheck MODE=context    # 上下文模式
make coccicheck M=security/airy/  # 仅扫描指定目录

# 指定规则
make coccicheck COCCI=scripts/coccinelle/null/deref_null.cocci
```

### 4.3 上游 Coccinelle 规则集

Linux 6.6 在 `scripts/coccinelle/` 下提供 80+ 规则，覆盖以下类别：

| 类别 | 目录 | 示例规则 |
|------|------|---------|
| `null` | `scripts/coccinelle/null/` | `deref_null.cocci`（NULL 解引用） |
| `locks` | `scripts/coccinelle/locks/` | `double_lock.cocci`（双加锁） |
| `api` | `scripts/coccinelle/api/` | `alloc/alloc_cast.cocci`（多余的类型转换） |
| `iterators` | `scripts/coccinelle/iterators/` | `forone.cocci`（迭代器使用错误） |
| `misc` | `scripts/coccinelle/misc/` | `boolinit.cocci`（bool 初始化） |
| `free` | `scripts/coccinelle/free/` | `kfree.cocci`（kfree 后访问） |

### 4.4 agent-linux 专属 Coccinelle 规则

`scripts/coccinelle/airy/` 下定义 agent-linux 专属规则：

```cocci
// scripts/coccinelle/airy/airy_agent_state_access.cocci
// 检测 airy_agent_state 字段未通过 READ_ONCE/WRITE_ONCE 访问
///
// AIRY-COCCI-001: airy_agent_state fields must use READ_ONCE/WRITE_ONCE
//

@bad_access@
struct airy_agent_state *S;
identifier field;
position p;
@@
  S->field@p

@script:ocaml depends on bad_access@
field << bad_access.field;
@@
if List.mem field ["state"; "agent_id"; "token_budget"; "sched_policy"]
then begin
  Printf.printf "AIRY-COCCI-001: %s must use READ_ONCE/WRITE_ONCE\n" field
end
```

```cocci
// scripts/coccinelle/airy/airy_ipc_ring_lock.cocci
// 检测 airy_ipc_ring 操作未在锁保护下
///
// AIRY-COCCI-002: airy_ipc_ring operations must hold airy_ipc_ring_lock
//

@locked_context@
@@

@bad_unlocked@
expression E;
position p != locked_context.p;
@@
  airy_ipc_ring_*@p(E)
```

**OS-TEST-063**：agent-linux 专属 Coccinelle 规则（`AIRY-COCCI-001` ~ `AIRY-COCCI-010`）必须在 CI PR 阶段运行；任一规则触发即阻断 PR。

---

## 5. Clang Static Analyzer

### 5.1 Clang 静态分析器

Clang 编译器内置静态分析器（`clang --analyze`），可检测：

- 死代码（dead code）
- 死存储（dead store）
- 内存泄漏（malloc 后未 free）
- NULL 解引用
- 路径敏感分析

### 5.2 启用方式

```bash
# 编译时启用 Clang 静态分析
make CC=clang CFLAGS="-analyze" 

# 通过 scan-build 工具
scan-build make ARCH=um -j$(nproc)

# 查看报告
scan-view /tmp/scan-build-*/
```

### 5.3 在 agent-linux 的应用

agent-linux 在 CI nightly 阶段运行 Clang 静态分析器，重点扫描 `security/airy/`、
`kernel/superv/`、`kernel/syscalls/`、`kernel/corekern/`、`kernel/log/`、
`kernel/ipc/` 等 AirymaxOS 目录：

```bash
# CI nightly 步骤
scan-build -o /tmp/scan-build \
  --status-bugs \
  --html-title="agent-linux Clang Static Analysis" \
  make ARCH=um CC=clang -j$(nproc) security/airy/
```

**OS-TEST-064**：CI nightly 必须运行 Clang 静态分析器扫描上述 AirymaxOS 目录；任一 bug 报告即标记 nightly 失败。

---

## 6. Coverity 扫描集成

### 6.1 Coverity 工作原理

Coverity 是 Synopsys 公司的商业静态分析服务，提供深度跨函数分析与漏洞检测。Linux 内核社区通过 `coverity.com` 免费扫描内核代码。

### 6.2 agent-linux Coverity 集成

agent-linux 每周自动同步代码至 Coverity Scan：

```yaml
# .github/workflows/coverity-scan.yml
name: coverity-scan
on:
  schedule:
    - cron: "0 0 * * 0"  # 每周日 UTC 00:00
  workflow_dispatch: {}

jobs:
  coverity:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Install Coverity Build Tool
        run: |
          wget -q https://scan.coverity.com/download/linux64 \
            --post-data "token=${{ secrets.COVERITY_TOKEN }}&project=agent-linux" \
            -O cov-analysis-linux64.tgz
          tar xzf cov-analysis-linux64.tgz
      - name: Build with cov-build
        run: |
          export PATH=$PWD/cov-analysis-linux64/bin:$PATH
          # UML 无 airy_defconfig 变体：defconfig + merge_config 叠加 x86 airy_defconfig
          make ARCH=um defconfig
          ./scripts/kconfig/merge_config.sh -m .config arch/x86/configs/airy_defconfig
          make ARCH=um olddefconfig
          cov-build --dir cov-int make ARCH=um -j$(nproc)
      - name: Submit to Coverity Scan
        run: |
          tar czf agent-linux.tgz cov-int
          curl --form token=${{ secrets.COVERITY_TOKEN }} \
               --form email=ci@spharx.com \
               --form file=@agent-linux.tgz \
               --form version="$(git rev-parse --short HEAD)" \
               --form description="agent-linux weekly scan" \
               https://scan.coverity.com/builds?project=agent-linux
```

**OS-TEST-065**：Coverity 报告的高优先级缺陷（High impact）必须在 72 小时内修复；中优先级（Medium）在 1 周内修复；低优先级（Low）在 1 个月内修复。

---

## 7. checkpatch.pl 扩展

### 7.1 上游 checkpatch.pl

Linux 6.6 的 `scripts/checkpatch.pl` 是内核代码风格检查脚本，检测：

- 代码风格（缩进、空格、行宽）
- 提交消息格式
- 类型使用（如禁止 `typedef struct`）
- 错误处理模式
- 内存管理（`kmalloc` 返回值检查）

### 7.2 `airy_checkpatch_ext.pl`：agent-linux 扩展

`kernel/scripts/airy_checkpatch_ext.pl` 随 kernel 子模块（`ALK-6.6-dev`）维护，
在上游 checkpatch 之外补充 AirymaxOS 架构约束。规则集由引擎内的单一注册表
维护，`--help` 与 `--list-rules` 均由其派生；本卷记录契约，规则实现以引擎为
唯一事实源（SSoT），集成测试位于 `tools/checkpatch/tests/`。

| 规则 ID | 检测内容 | 严重性 |
|--------|---------|--------|
| `AIRY-CHECKPATCH-001` | 必须使用 `pr_*` 而非 `printk` | error |
| `AIRY-CHECKPATCH-002` | 非 static 函数必须以 `airy_`/`stc_` 前缀命名 | error |
| `AIRY-CHECKPATCH-003` | `airy_agent_state` 访问必须用 `READ_ONCE`/`WRITE_ONCE` | error |
| `AIRY-CHECKPATCH-004` | `airy_ipc_send` 调用必须检查返回值（含多行语句） | error |
| `AIRY-CHECKPATCH-005` | `airy_cap_check` 调用必须显式处理 `-EPERM` | warning |
| `AIRY-CHECKPATCH-006` | 禁止在 `airy_*`/`stc_*` 函数中直接调用 `printk` | error |
| `AIRY-CHECKPATCH-007` | LSM 注册必须使用 `security_hook_list` | error |
| `AIRY-CHECKPATCH-008` | 禁止在 `airy_ipc`/`airy_mem` 路径调用 `dma_alloc_coherent` | error |
| `AIRY-CHECKPATCH-009` | 禁止在 AirymaxOS 路径启用 `CONFIG_BPF_LSM` | error |
| `AIRY-CHECKPATCH-010` | 禁止在 AirymaxOS 路径启用 `CONFIG_SCHED_EXT` | error |
| `AIRY-CHECKPATCH-011` | 文件头：SPDX 首行 + 前导注释区 ≤ 20 行（OS-STD-STY-007） | error |

扫描落点（相对 kernel 根，省略路径时使用）：

- 目录：`security/airy/`、`kernel/superv/`、`kernel/syscalls/`、
  `kernel/corekern/`、`kernel/log/`、`kernel/ipc/`、`include/airymax/`、
  `include/uapi/linux/airymax/`
- 独立文件：`init/airy_core.c`、`include/trace/events/airy.h`
- 配置：自动追加 `arch/*/configs/airy*_defconfig`

分析语义：

- 先剥离注释与字符串字面量，再按语句分析；多行调用与宏实参内的调用均
  参与规则 004/005 判定。
- 函数体范围由花括号配平推导，`static` 辅助函数的归属与其调用点一致。
- 规则 004/005 仅在返回值被丢弃时报告：裸调用与 `(void)` 显式丢弃均视为
  未检查；`if (...)`、赋值、断言宏等消费结果的调用不报告。
- 规则 009/010 同时拒绝 `=y` 与 `=m`；`# CONFIG_X is not set` 视为关闭。
- 目录遍历剪枝 `.git`、`.kunit`、`__pycache__`；显式目标先解析为绝对路径。

CLI 契约：

```bash
LC_ALL=C perl kernel/scripts/airy_checkpatch_ext.pl              # 默认落点
LC_ALL=C perl kernel/scripts/airy_checkpatch_ext.pl kernel/security/airy
perl kernel/scripts/airy_checkpatch_ext.pl --list-rules          # TSV 注册表
LC_ALL=C perl kernel/scripts/airy_checkpatch_ext.pl --quiet      # 仅违规项
```

| 退出码 | 含义 |
|--------|------|
| `0` | 无 `error` 级违规（允许 `warning`） |
| `1` | 存在 `error` 级违规 |
| `2` | 用法或 IO 错误（显式目标不存在、文件不可读），优先于 `1` |

输出格式：`<相对路径>:<行号>: <ERROR|WARNING> <规则号>: <说明>`，末行汇总为
`airy_checkpatch_ext: N files checked, N errors, N warnings`；`--quiet` 省略
汇总行，错误信息仍写标准错误。

**OS-TEST-066**：CI PR 阶段必须运行 `airy_checkpatch_ext.pl` 扫描上述全部默认
落点与 `airy*_defconfig`；任一 `error` 级别报告即阻断 PR。

---

## 8. [SC] 头文件编译器无关性检查

### 8.1 检查目标

IRON-9 v3 [SC] 共享契约层的 12 个头文件（`error.h` / `log_types.h` / `ipc.h` / `sched.h` / `memory_types.h` / `security_types.h` / `cognition_types.h` / `syscalls.h` / `syscall_gen.h` / `uapi_compat.h` / `lsm_types.h` / `bpf_struct_ops.h`）必须**编译器无关**——即用 GCC、Clang、TinyCC（TCC）等任意 C 编译器均能编译通过，且语义一致。这保证 [SC] 头文件可在 agentrt（用户态）与 agent-linux（内核态）之间无障碍共享。

### 8.2 `check-uapi-compiler-agnostic.sh` 脚本

```bash
#!/bin/bash
# scripts/check-uapi-compiler-agnostic.sh
# 校验 12 个 [SC] 头文件的编译器无关性

set -euo pipefail

SC_HEADERS=(
    "error.h" "log_types.h" "ipc.h" "sched.h" "memory_types.h"
    "security_types.h" "cognition_types.h" "syscalls.h" "syscall_gen.h"
    "uapi_compat.h" "lsm_types.h" "bpf_struct_ops.h"
)

SC_DIR="include/uapi/linux/airymax"

# 使用的编译器列表
COMPILERS=("gcc" "clang" "tcc")

# 使用的标准列表
STANDARDS=("c89" "c99" "c11" "c17")

# 使用的架构列表
ARCHS=("x86_64" "arm64" "riscv" "loongarch")

failures=0

for header in "${SC_HEADERS[@]}"; do
    for compiler in "${COMPILERS[@]}"; do
        for std in "${STANDARDS[@]}"; do
            # 构造最小测试程序
            test_c=$(mktemp --suffix=.c)
            cat > "$test_c" <<EOF
#include <$header>
int main(void) { return 0; }
EOF

            # 编译（仅预处理 + 编译，不链接）
            extra_flags=""
            case "$compiler" in
                gcc)   extra_flags="-Wall -Wextra -Wpedantic -Werror" ;;
                clang) extra_flags="-Wall -Wextra -Wpedantic -Werror" ;;
                tcc)   extra_flags="-Wall -Werror" ;;
            esac

            if ! $compiler -std=$std $extra_flags \
                    -I"$SC_DIR" -I"$SC_DIR/.." \
                    -c "$test_c" -o /dev/null 2>&1; then
                echo "::error::[SC] $header failed with $compiler -std=$std"
                failures=$((failures + 1))
            fi

            rm -f "$test_c"
        done
    done
done

if [ "$failures" -gt 0 ]; then
    echo "::error::[SC] headers compiler-agnostic check failed ($failures failures)"
    exit 1
fi

echo "[SC] headers compiler-agnostic check: PASS (12 headers × 3 compilers × 4 standards = 144 combinations)"
exit 0
```

### 8.3 编译器无关性禁用清单

[SC] 头文件**禁止**使用以下编译器扩展：

| 禁用项 | 原因 | 替代方案 |
|--------|------|---------|
| `__attribute__((packed))` | GCC/Clang 专属 | C11 `_Alignas` 或手动对齐 |
| `__builtin_*` | GCC/Clang 专属 | 标准库函数 |
| `__asm__` / `asm()` | 架构相关 | 内联汇编隔离到 [IND] 层 |
| `__sync_*` / `__atomic_*` | GCC 内建 | C11 `<stdatomic.h>` |
| `typeof` / `__typeof__` | GCC/Clang 专属 | 显式类型声明 |
| 语句表达式 `({ ... })` | GCC 专属 | 内联函数 |
| 三元运算符 `?:` 在初始化器 | 部分编译器不支持 | 预处理条件 |
| 可变长度数组 VLA | C99 可选，C11 可选 | 固定大小数组 |
| `#pragma pack` | 编译器专属 | 手动填充 |

**OS-TEST-067**：CI PR 阶段必须运行 `check-uapi-compiler-agnostic.sh`；任一头文件在任一编译器/标准组合下编译失败即阻断 PR。

**OS-KER-122**：[SC] 头文件禁止使用 §8.3 清单中的编译器扩展；CI 通过正则 `grep -E "__attribute__|__builtin_|__asm__|__sync_|__atomic_|typeof|\\(\\{"` 检测，违规即 PR 驳回。

**OS-STD-081**：`check-uapi-compiler-agnostic.sh` 必须覆盖 3 编译器 × 4 标准 = 12 种组合；CI 在 PR 阶段执行完整 12 种组合，nightly 阶段额外覆盖 3 架构（共 36 种组合）。

---

## 9. CI 集成：`kernel-verify` workflow

### 9.1 `kernel-verify` workflow 静态分析阶段

```yaml
# .github/workflows/kernel-verify.yml
name: kernel-verify
on: [push, pull_request]

jobs:
  checkpatch:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Run checkpatch.pl + airy_checkpatch_ext.pl
        run: |
          # 对每个 .patch 文件运行 checkpatch（失败即阻断，对齐 kernel-verify.yml）
          for patch in $(git format-patch -1 HEAD); do
            scripts/checkpatch.pl --strict --no-tree -f "$patch" || exit 1
          done
          # 对 AirymaxOS 全部默认落点运行 airy_checkpatch_ext（省略路径即默认落点）
          LC_ALL=C perl kernel/scripts/airy_checkpatch_ext.pl || exit 1
  
  sparse:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Install Sparse
        run: sudo apt-get install -y sparse
      - name: Build with Sparse
        run: |
          # UML 无 airy_defconfig 变体：defconfig + merge_config 叠加 x86 airy_defconfig
          make ARCH=um defconfig
          ./scripts/kconfig/merge_config.sh -m .config arch/x86/configs/airy_defconfig
          make ARCH=um olddefconfig
          make ARCH=um C=2 CHECK=sparse \
            CF="-Wbitwise -Wcontext -Wdecl -Wno-transparent-union" \
            -j$(nproc) 2>&1 | tee sparse.log
      - name: Parse Sparse warnings
        run: |
          errors=$(grep -c "^.*: error:" sparse.log || true)
          if [ "$errors" -gt 0 ]; then
            echo "::error::Sparse reported $errors errors"
            grep "error:" sparse.log
            exit 1
          fi

  coccinelle:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Install Coccinelle
        run: sudo apt-get install -y coccinelle
      - name: Run Coccinelle
        run: |
          make coccicheck MODE=report M=security/airy/ 2>&1 | tee coccinelle.log
          # 检查 agent-linux 专属规则
          make coccicheck MODE=report \
            COCCI=scripts/coccinelle/airy/airy_agent_state_access.cocci
          make coccicheck MODE=report \
            COCCI=scripts/coccinelle/airy/airy_ipc_ring_lock.cocci

  uapi-compiler-agnostic:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Install compilers
        run: |
          sudo apt-get install -y gcc clang
          # TCC 需要从源码安装
          git clone https://github.com/TinyCC/tinycc.git /tmp/tcc
          cd /tmp/tcc && ./configure && make && sudo make install
      - name: Run check-uapi-compiler-agnostic.sh
        run: scripts/check-uapi-compiler-agnostic.sh

  smatch:
    runs-on: ubuntu-24.04
    if: github.event_name == 'schedule'  # 仅 nightly 运行
    steps:
      - uses: actions/checkout@v4
      - name: Install Smatch
        run: |
          git clone https://github.com/error27/smatch.git /tmp/smatch
          cd /tmp/smatch && make && sudo make install
      - name: Run Smatch
        run: |
          # UML 无 airy_defconfig 变体：defconfig + merge_config 叠加 x86 airy_defconfig
          make ARCH=um defconfig
          ./scripts/kconfig/merge_config.sh -m .config arch/x86/configs/airy_defconfig
          make ARCH=um olddefconfig
          make ARCH=um CHECK=smatch C=1 -j$(nproc) 2>&1 | tee smatch.log
      - name: Parse Smatch reports
        run: |
          errors=$(grep -c "^.*: error:" smatch.log || true)
          if [ "$errors" -gt 0 ]; then
            echo "::error::Smatch reported $errors errors"
            grep "error:" smatch.log
            exit 1
          fi
```

### 9.2 CI 各阶段的工具组合

| 阶段 | 工具 | 阻断条件 |
|------|------|---------|
| PR 阶段（push/pull_request） | checkpatch + airy_checkpatch_ext + Sparse + Coccinelle + check-uapi-compiler-agnostic | 任一 error |
| nightly | Smatch + Coverity | 任一 error |
| 每周 | Coverity Scan | 高优先级缺陷 72 小时修复 |

---

## 10. 与上下游测试层的协作

### 10.1 与 04-dynamic-analysis 的关系

04 卷动态分析检测"运行时行为"，本卷静态分析检测"代码模式"。二者互补：

- **静态分析**：检测 NULL 解引用模式、未初始化变量、锁不平衡（无需运行）。
- **动态分析**：检测实际越界访问、实际数据竞争（需要运行触发）。

CI 必须同时启用两者，静态分析阻断 PR，动态分析阻断 nightly。

### 10.2 与 03-kernel-selftests 的关系

03 卷的 [SC] 头文件 SHA-256 自检（运行时）与本卷的 `check-uapi-compiler-agnostic.sh`（编译时）共同守护 [SC] 契约：

- **编译时**：校验 [SC] 头文件编译器无关性（§8）。
- **运行时**：校验 [SC] 头文件 SHA-256 逐字节一致（03 卷 §3.5）。

---

## 11. 维护者制度与版本演进

### 11.1 规则编号注册表

本卷强制规则编号 `OS-TEST-060` ~ `OS-TEST-067`、`OS-KER-120` ~ `OS-KER-122`、`OS-STD-081`，已注册至 50-engineering-standards/09-ssot-registry.md 的"SSoT 规则编号注册表"。

### 11.2 v1.1.0 变更内容

1. §7.2 与内核引擎 `kernel/scripts/airy_checkpatch_ext.pl` 对齐：规则集扩展为
   `AIRY-CHECKPATCH-001` ~ `011`（新增文件头规则 011），规则注册表为唯一事实源，
   `--help` 与 `--list-rules` 由其派生。
2. 明确默认扫描落点：8 个 Airy 目录 + 2 个独立文件 + 自动追加的
   `arch/*/configs/airy*_defconfig`。
3. 记录分析语义：注释/字符串剥离后的语句级分析、花括号配平推导函数范围、
   规则 004/005 仅对丢弃返回值报告、009/010 同时拒绝 `=y` 与 `=m`、
   VCS/构建目录剪枝与显式目标绝对化。
4. 明确 CLI 契约与退出码：`--quiet` 仅输出违规项；`2`（用法/IO 错误）
   优先于 `1`（存在 error）。
5. 集成测试扩充至 32 个用例，覆盖规则命中、严重级语义、默认落点、剪枝与
   错误路径。
6. 全文路径校正为实际工程落点（`security/airy/`、`kernel/corekern/` 等），
   删除与实际引擎不一致的规则伪代码，避免 SSoT 漂移。

### 11.3 后续版本规划

- v1.0.1：新增 `airy_sparse_rules` 5 项规则，覆盖 IPC Ring 锁序、Token 预算原子性。
- 下一版本：将 `check-uapi-compiler-agnostic.sh` 扩展至 5 编译器 × 5 标准 × 5 架构 = 125 组合。
- 后续版本：与 10-formal-verification 联动，将形式化验证的属性检查集成至静态分析阶段。

---

## 12. 相关文档

- [80-testing README](README.md)：测试体系主索引（v1.0），定义 L5 静态分析分层
- [03-kernel-selftests.md](03-kernel-selftests.md)：内核自测试（[SC] 头文件运行时校验）
- [04-dynamic-analysis.md](04-dynamic-analysis.md)：动态分析（与本卷互补）
- [06-coverage-metrics.md](06-coverage-metrics.md)：覆盖率度量
- [10-formal-verification.md](10-formal-verification.md)：形式化验证（与本卷联动）
- [../10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)：IRON-9 v3 四层模型（[SC] 契约层）
- [../30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md)：A-UEF [SC] error.h 契约
- [../50-engineering-standards/05-development-process.md](../50-engineering-standards/05-development-process.md)：工具链与自动化
- [../70-build-system/03-ci-cd-pipeline.md](../70-build-system/03-ci-cd-pipeline.md)：CI/CD 流水线（`kernel-verify.yml`）

---

## 13. 参考材料

- Linux 6.6 `Documentation/dev-tools/sparse.rst`（Sparse 文档）
- Linux 6.6 `Documentation/dev-tools/coccinelle.rst`（Coccinelle 文档）
- Linux 6.6 `scripts/checkpatch.pl`（代码风格检查）
- Linux 6.6 `scripts/coccinelle/`（80+ 语义补丁规则）
- Smatch 项目：<https://github.com/error27/smatch>
- Coverity Scan：<https://scan.coverity.com>
- Clang Static Analyzer：<https://clang-analyzer.llvm.org>

---

## 14. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0.1 | 2026-07-18 | 初始版本：定义 6 大静态分析工具（Sparse/Smatch/Coccinelle/Clang Static Analyzer/Coverity/checkpatch.pl）的启用模型；新增 `airy_checkpatch_ext.pl` 扩展规则 10 项；新增 `check-uapi-compiler-agnostic.sh` [SC] 头文件编译器无关性校验；定义 `kernel-verify` workflow 静态分析阶段 |
| v1.1.0 | 2026-10-19 | §7.2 与内核引擎实际实现对齐：规则集 001~011、默认扫描落点、分析语义、CLI 与退出码契约；全文路径校正至实际工程落点；删除与引擎不一致的伪代码；集成测试 32 用例 |

---

> **文档结束** | agent-linux 测试工程体系 v1.0.1 第 5 卷 | 维护者：开源极境工程与规范委员会 | "From data intelligence emerges."
