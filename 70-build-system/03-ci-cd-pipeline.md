Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agent-linux（AirymaxOS）CI/CD 管道完整设计
> **文档定位**：agent-linux（AirymaxOS）CI/CD 管道完整设计——覆盖 7 层自动化验证体系（OS-STD-TOOL-001~161）的工程实现，定义 8 子仓 + 管理仓的全量 CI 覆盖矩阵、编译验证矩阵、代码质量门禁、测试覆盖矩阵、SSoT 校验流水线、[SC] 共享契约层双向 CI、发布流水线、CI 基础设施与失败回滚策略。本卷是 1.0.1 开发阶段的设计基线，所有设计必须在本阶段完成。\
> **文档版本**：v1.0.2\
> **最后更新**： 2026-10-19\
> **上级文档**：[agent-linux 设计文档](README.md)\
> **同源映射**：agentrt `cmake/` + `.github/workflows/`（伞仓 CI）+ Linux 6.6 内核 CI 思想（KUnit/kselftest/checkpatch/sparse/Coccinelle）\
> **理论根基**：Linux 6.6 内核基线 CI 工程 + Airymax 五维正交 24 原则（S/K/C/E/A 五维）+ 7 层自动化验证体系（OS-STD-TOOL-001~161）\
> **核心约束**：IRON-9 v3 同源且部分代码共享——agent-linux CI 沿用 Linux 内核 CI 思想但与上游独立演进；[SC] 共享契约层双向 CI 由 OS-IRON-008 强制\
> **SSoT 依赖声明**：本文档引用的全部规则编号（`OS-STD-TOOL-*` / `OS-IRON-*` / `OS-STD-FMT-*` / `OS-TEST-*` / `OS-STD-TEST-*`）的权威定义与登记状态以 [`50-engineering-standards/09-ssot-registry.md`](../50-engineering-standards/09-ssot-registry.md)（Markdown 权威源，文档体系）与 [`agent-linux/ssot-registry.yaml`](../../../agent-linux/ssot-registry.yaml)（机器可读 Schema，管理仓根）为唯一来源（SSoT）。本文档仅引用已登记编号，不私自定义新编号。当本文档与 SSoT 注册表冲突时，以注册表为准。

---

## 0. 章节定位

本卷是 agent-linux 构建系统在 1.0.1 开发阶段的 CI/CD 管道设计基线。它在 `70-build-system/README.md`（构建系统主索引）与 `80-testing/`（测试工程卷）之间形成自动化验证执行层：

- **上游依赖**：README 定义构建系统的机制总览；`01-kbuild-system.md`/`02-kconfig-system.md` 定义"代码如何被编译、配置如何驱动构建"；本卷定义"编译与配置产物如何被自动化验证拦截/放行"。
- **下游依赖**：`80-testing/01-kunit-framework.md`（OS-TEST-001~012）与 `80-testing/02-kselftest.md`（OS-TEST-013~022）定义测试套件编写规范；本卷定义这些测试如何在 CI 矩阵中被调度执行并形成门禁。

本卷所有强制规则均引用 [`09-ssot-registry.md`](../50-engineering-standards/09-ssot-registry.md) 中已登记的 **OS-STD-TOOL** / **OS-IRON** / **OS-STD-FMT** / **OS-TEST** / **OS-STD-TEST** 编号，禁止在本文档私自定义新编号。

### 0.1 关键术语

| 术语 | 定义 |
|------|------|
| 7 层自动化验证体系 | OS-STD-TOOL-001~161 定义的端到端验证链：L1 编译期检查 / L2 静态分析 / L3 预提交 / L4 CI 门禁 / L5 连接验证 / L6 协议验证 / L7 发布验证 |
| CI 矩阵 | 配置 × 架构 × 编译器的笛卡尔积构建矩阵（4×4×2−4=28，loongarch+clang 组合整体排除） |
| [SC] 共享契约层 | kernel 与用户态共用的 12 个头文件契约（OS-IRON-014），权威副本位于 `kernel/include/uapi/linux/airymax/` |
| [SC] 强制 | kernel 侧权威副本 + `mgmt-files` / OS-CHK-DOC-04 / OS-IRON-001 三道门禁（OS-IRON-008） |
| SSoT | Single Source of Truth，规则编号唯一权威来源（`09-ssot-registry.md`/`.yaml`） |
| 门禁（Gate） | CI 中阻断 PR 合并的硬性检查点，失败即禁止合并 |
| SBOM | Software Bill of Materials，软件物料清单（本设计采用 SPDX 格式） |

---

## 1. 设计目标

agent-linux CI/CD 管道的设计目标是把"代码进入主线"的工程契约（OS-STD-PROD 系列）转化为可机器执行、可审计、可回滚的自动化验证流水线。核心承载是 **7 层自动化验证体系（OS-STD-TOOL-001~161）**。

### 1.1 7 层自动化验证体系

7 层验证覆盖编号区间 **OS-STD-TOOL-001~161**。SSoT 注册表（[`ssot-registry.yaml`](../../../agent-linux/ssot-registry.yaml) Ch 4.6）将编号区间划分为两大段：`OS-STD-TOOL-001~099`（L1-3 段）与 `OS-STD-TOOL-101~161`（L4-7 段）。具体规则到层级的归属由各规则在权威定义文档中的语义决定，而非单纯按编号段机械切分。下表给出层级的执行时机、阻断语义与已登记的代表规则：

| 层级 | 名称 | 编号归属 | 代表规则（已登记） | 执行时机 | 阻断合并 |
|------|------|---------|------------------|---------|---------|
| **L1** | 编译期检查 | 001~099 段（L1-3） | OS-STD-TOOL-001（编译期检查段起点，权威定义见注册表） | 本地 + CI | 是 |
| **L2** | 静态分析 | 001~099 段（L1-3） | OS-STD-TOOL-081（sparse C=2 + W 分级） | 本地 + CI | 是 |
| **L3** | 预提交（pre-commit） | 001~099 段（L1-3） | OS-STD-FMT-001/002（clang-format） | 本地（git hook） | 否（建议） |
| **L4** | CI 门禁 | 101~161 段（L4-7） | OS-STD-TOOL-081-CI（CI 强制 sparse W=2）、OS-STD-TOOL-160（checkpatch --strict）、OS-STD-TOOL-161（多配置构建） | CI（PR） | 是 |
| **L5** | 连接验证 | 101~161 段（L4-7） | OS-STD-TOOL-121/122（覆盖率 ≥80%）、OS-STD-TOOL-124（关键路径 100%） | CI（develop nightly） | 是 |
| **L6** | 协议验证 | 101~161 段（L4-7） | ABI/协议契约一致性校验 | CI（develop nightly） | 是 |
| **L7** | 发布验证 | 101~161 段（L4-7） | OS-STD-TOOL-081-REL（发布前 sparse 全量审计）、OS-STD-TOOL-160/161 | CI（release tag） | 是（阻塞发版） |

> **SSoT 对齐说明**：上表"编号归属"列严格遵循注册表的两段划分（001~099 / 101~161），不为本卷私自定义子段边界。个别规则（如 OS-STD-TOOL-081 基线属 L2 静态分析，但其 `-CI`/`-REL` 子编号分别在 L4/L7 执行）按规则语义跨层生效，其权威定义见注册表"权威定义"列指向的源文档。本卷禁止新增或重新切分编号段；新增层级映射须通过 RFC 流程在 [`09-ssot-registry.md`](../50-engineering-standards/09-ssot-registry.md) 登记后方可使用。

设计要点：

- **L1~L4 前 4 层** 是每个 PR 必须通过的"合并门禁"（对齐 `05-development-process.md`：补丁进入 `develop` 前必须通过 7 层验证的前 4 层——编译期 / 静态分析 / 预提交 / CI 门禁）。
- **L5~L6** 在 `develop` 分支 nightly build 中执行，检测跨子系统符号冲突、ABI 漂移、协议契约违反。
- **L7** 仅在 release tag 推送时触发，对应 SBOM 生成、签名构建、发布渠道分发。
- **OS-STD-TOOL-001 与 OS-STD-TOOL-161 是端到端边界**：001 是编译期检查段（L1-3 段，001~099）起点，161 定义多配置构建（defconfig + allnoconfig + allmodconfig）三态皆绿（161 权威定义见注册表）。

### 1.2 全量 CI 覆盖

CI 覆盖范围遵循 OS-IRON-013（8 架构子仓 + distro 发行版子仓共 9 个独立 git 仓库 + submodule 管理）：

- **9 子仓**：每个子仓独立维护 `.github/workflows/`，CI 互相隔离、独立失败、独立回滚。
- **管理仓（agent-linux）**：维护 SSoT 校验、文件完整性、子仓状态聚合、文档格式校验，串行执行。
- **kernel 子仓**：CI 最重，承载 Kbuild 编译、checkpatch、sparse、Coccinelle、KUnit、kselftest 全套内核验证。
- **[SC] 共享契约层**：kernel 侧权威副本 + `mgmt-files` / OS-CHK-DOC-04 / OS-IRON-001 门禁强制（§7）。

### 1.3 设计原则映射

| 原则 | 在本卷的体现 |
|------|------------|
| **A-4 完美主义** | 7 层验证全绿方可合并；零警告编译；多配置矩阵全绿 |
| **S-1 反馈闭环** | L3 本地预提交毫秒级反馈 + L4 CI 分钟级阻断 |
| **S-2 层次分解** | 7 层按"局部 → 全局 → 跨仓 → 发布"递进 |
| **E-3 资源确定性** | ccache/cargo/pip 三级缓存 + CI 总时长 ≤60 分钟，kernel 矩阵冷缓存兜底 90 分钟（OS-STD-TEST-011） |
| **E-7 文档即代码** | SSoT YAML Schema 机器可读 + .md 文档规则 ID 自动校验 |
| **K-4 可插拔策略** | 子仓按构建系统分类（Meson/Cargo/Go/tsc），CI job 可独立增删 |

---

## 2. CI 管道架构

### 2.1 workflow 文件布局

所有 GitHub Actions workflow 文件位于 `agent-linux/.github/workflows/`：

```
agent-linux/.github/workflows/
├── mgmt-verify.yml        # 管理仓：文件完整性（含 [SC] 头文件存在性与仓内唯一副本）+ 子仓 CI 聚合
├── ssot-verify.yml        # SSoT 规则 ID 校验（全仓）
├── kernel-verify.yml      # kernel 子仓：Kbuild 矩阵 + checkpatch + sparse + Coccinelle + KUnit + kselftest
├── airy-verify.yml        # Airymax 内核代码：4 架构构建 + kernel-airy（CONFIG/BTF/产物归属/符号冲突/SPDX）
├── os-chk-verify.yml      # OS-CHK 统一门禁编排（gates.yaml，P0/P1 阻断）
├── image-build.yml        # 发行前镜像验收：desktop ISO 组装 + QEMU 装机/冷启动（手动触发，非 PR 门禁）
├── nightly.yml            # develop nightly build（L5/L6 连接/协议验证；formal/soak/chaos 工具未实现，降级 SKIP）
└── release.yml            # release tag 流水线（L7 发布验证）
```

> **薄壳原则**：8 个 workflow 只做触发、矩阵与环境准备，判定逻辑一律落在 `ci/ci.sh <gate>` 对应的 `ci/gates/*.sh`，这样同一份检查在本地与 CI 完全同源。workflow 内联 `run:` 块与 job 数受 `ci/workflow-budgets.txt` 双向棘轮约束（只许下调），由 `ci/check-workflows.py` 在每次 PR 强制。

> **规划说明**：codegen 一致性不再单列 workflow，作为 OS-CHK 门禁 `OS-IRON-009-CODEGEN` 在 `os-chk-verify.yml` 内执行（设计定义见 [`30-interfaces/06-codegen-pipeline.md`](../30-interfaces/06-codegen-pipeline.md) §7）。coverity-scan / nightly-dynamic-analysis / formal-verification 等独立 workflow 尚未实现，属规划项，不在上述 8 个 workflow 之列；形式化验证相关能力见 §5.4 的"未实现"标注。`image-build.yml` 是发行前门禁而非 PR 门禁，按 `release.yml` 先例申请发布级例外：`assemble` 240 分钟、`accept` 360 分钟；`accept` 的时限由装机腿数推导（双固件两腿 × 每腿装机 7200 秒 + 冷启动 900 秒），而非拍定值，设计与验收接线见 [`0.0.3-极境OS任务方案.md`](../../docs-closed/agent-linux-dev/0.0.3-极境OS任务方案.md) §14 §3。

> **内核单次构建**：`image-build.yml` 的 `release_tag` 输入非空时，内核包不再就地构建，而是从该 Release 下载 x86_64 内核包、按发行 `.sha256sum` 逐条校验后经 `run-image-ci.sh --kernel-rpms` 交给组装，组装段把内核记为 `reuse`。组装完成后再把载荷清单里的内核包哈希与发行校验和逐条比对，任一不符即判红。镜像与发行资产的内核只允许有一次构建，否则同名包哈希分叉，装机载荷无法用发行校验和核验；下载不全或载荷内核包少于三个同样直接判失败，不回落就地构建。

### 2.2 管理仓 CI（mgmt-verify.yml）

管理仓 CI 由 **2 个 job** 组成：`file-integrity`（治理文件与子仓完整性）→ `orchestrate-leaf-ci`（子仓 CI 编排），任一失败即阻断：

1. **file-integrity（治理文件完整性）**：校验 `.gitmodules` 中声明的 9 子仓 submodule 目录实际存在（kernel/services/security/memory/cognition/cloudnative/system/tests-linux/distro）；校验 `kernel/include/uapi/linux/airymax/` 下 12 个头文件物理存在（OS-IRON-014）；校验治理文件（README/LICENSE/NOTICE/MAINTAINERS/CODEOWNERS 等）齐备；校验无 `airymaxos-` 前缀残留。
2. **orchestrate-leaf-ci（子仓 CI 编排）**：通过 `aggregate-subrepo-ci.py` 聚合 9 子仓最近一次 CI 状态（任一子仓主干红则管理仓红）；markdownlint 校验 `docs/`；校验文档版权头（`check-copyright.py`）。

> **职责划分**：SSoT 规则 ID 校验不在此 workflow 内，由独立的 `ssot-verify.yml` 承担（§6）。本 workflow 通过 `needs: file-integrity` 保证 job 2 串行等待 job 1。

```yaml
# agent-linux/.github/workflows/mgmt-verify.yml
name: mgmt-verify
on:
  pull_request:
    paths:
      - 'docs/**'
      - '.gitmodules'
      - 'ssot-registry.yaml'
      - 'MAINTAINERS'
      - 'CODEOWNERS'
  push:
    branches: [main, develop]

jobs:
  file-integrity:
    name: Governance File & Submodule Integrity
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { submodules: recursive }
      - name: Verify 8 submodule directories exist
        run: |
          set -e
          for sub in kernel services security memory cognition cloudnative system tests-linux; do
            test -d "$sub" || { echo "FAIL: $sub/ missing"; exit 1; }
          done
      - name: Verify [SC] 12 shared headers exist
        run: |
          set -e
          for h in error.h log_types.h ipc.h sched.h memory_types.h \
                   security_types.h cognition_types.h syscalls.h syscall_gen.h \
                   uapi_compat.h lsm_types.h bpf_struct_ops.h; do
            test -f "kernel/include/uapi/linux/airymax/$h" || { echo "FAIL: $h missing"; exit 1; }
          done
      - name: Verify required governance files
        run: |
          set -e
          for f in README.md README_zh.md LICENSE NOTICE CONTRIBUTING.md \
                   MAINTAINERS SECURITY.md CODEOWNERS .gitignore .gitmodules; do
            test -f "$f" || { echo "FAIL: $f missing"; exit 1; }
          done

  orchestrate-leaf-ci:
    name: Orchestrate Leaf Submodule CI
    runs-on: ubuntu-latest
    needs: file-integrity
    steps:
      - uses: actions/checkout@v4
        with: { submodules: recursive }
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - name: Aggregate 8 subrepo CI status
        run: python3 agent-linux/tools/aggregate-subrepo-ci.py --fail-on-red
      - name: markdownlint (docs only)
        run: markdownlint-cli2 'docs/**/*.md'
      - name: Verify copyright header (line 1)
        run: python3 agent-linux/tools/check-copyright.py --root docs
```

### 2.3 kernel 子仓 CI（kernel-verify.yml）

kernel 子仓 CI 是全仓最重的管道，承载 Linux 内核基线全套验证：

| 阶段 | 工具 | 规则编号 | 阻断 |
|------|------|---------|------|
| Kbuild 编译 | `make` + 4 配置矩阵（28 格） | OS-STD-TOOL-161 | 是 |
| checkpatch | `scripts/checkpatch.pl --strict` | OS-STD-TOOL-160 | 是 |
| sparse | `make C=2` + W=2 | OS-STD-TOOL-081-CI | 是 |
| Coccinelle | `make coccicheck` | L2 静态分析段（OS-STD-TOOL-001~099） | 是 |
| KUnit | UML + `kunit.py run` | OS-TEST-001~012 | 是 |
| kselftest | QEMU + TAP 解析 | OS-TEST-013~022 | 是 |

> **job 结构**：kernel-verify.yml 固定 **2 个 job**——`kernel-build`（28 格 Kbuild 编译矩阵）+ `kernel-verify`（静态分析 + 运行时测试），`kernel-verify` 通过 `needs: kernel-build` 串行等待编译矩阵全绿。

#### 门禁分层（2026-10 系统化改造）

workflow 文件不再承载任何判断逻辑，每个检查点都是一条可在本地对着源码树重放的命令：

```bash
ci/ci.sh kernel-build --config airy_defconfig --arch arm64 --compiler gcc
ci/ci.sh kernel-analyze --kind sparse --log /tmp/sparse.log
```

| 门禁 | 职责 | 覆盖规则 |
|------|------|---------|
| `kernel-deps` | 工具链准备。`--kind build` 按架构与编译器取交叉工具链，`--kind verify` 取静态分析与 virtme-ng；Debian 宿主直接安装，其它发行版只做存在性校验并报出缺哪个命令 | — |
| `kernel-build` | 矩阵单格：配置求解 → 编译 → `error:` 一律红、`warning:` 仅 Airymax 路径红 | OS-STD-TOOL-161 |
| `kernel-consistency` | 任务书矩阵 / ABI 黄金样本 / codegen 三条零漂移 | OS-STD-TOOL-161、OS-IRON-001 |
| `kernel-kunit` | UML 上跑 KUnit，并与基线 TAP 比对 | OS-TEST-001~012 |
| `kernel-refbuild` | x86_64 airy_defconfig 参考构建：10 条关键 CONFIG 精确断言、`CONFIG_LSM` 含 airy、vmlinux 带 `.BTF` 段 | OS-OBS-012 |
| `kernel-analyze` | `--kind checkpatch\|sparse\|coccinelle`，只扫 Airymax 目录 | OS-STD-TOOL-160、OS-STD-TOOL-081-CI、L2 |
| `kernel-kselftest` | 子集构建安装 → virtme-ng 运行 → TAP 解析 | OS-TEST-013~022 |

> **次序约束已变成断言**：此前「KUnit 必须早于所有 in-tree 构建」「sparse 依赖参考构建的 `.o`」只存在于 YAML 注释里，步骤一重排就会静默退化成假绿。现在 `kernel-kunit` 检测到源码树已有 `.config` 或 `vmlinux` 即拒绝启动；`kernel-analyze --kind sparse` 找不到已编译的 Airymax 目标文件即判红，而不是「零对象零告警」地通过。
>
> **为什么拆成七个门禁而不是一个大脚本**：单格失败要能只重跑那一格（矩阵 28 格）；静态检查各自的红线口径不同（checkpatch 只能解析 `total:` 行，sparse 必须限定目标文件，coccinelle 必须收窄 `--dir`），混在一起迟早互相掩盖。
>
> **棘轮约束**：`ci/workflow-budgets.txt` 记录每个 workflow 允许的内联 `run` 块数上限，只许减不许加，由 `ci/check-workflows.py` 在 lint 阶段强制。kernel-verify.yml 已从 10 降到 0，全仓合计 78 → 53。

> **路径过滤器语义**：kernel 在管理仓中是 **gitlink（submodule 指针）**，父仓无法枚举子仓内部文件，子模块提交上推只会改动 `kernel` 这一条路径本身。因此 `paths` 必须同时列出 `'kernel'` 与 `'kernel/**'`，否则内核更新不会触发任何流水线；同时列出 `ci/gates/kernel-*.sh`，门禁本身改动也要重跑。

```yaml
# agent-linux/.github/workflows/kernel-verify.yml（节选，完整文件 147 行）
name: kernel-verify
on:
  pull_request:
    paths:
      - 'kernel'
      - 'kernel/**'
      - 'tools/ci-matrix/**'
      - '.github/workflows/kernel-verify.yml'
      - 'ci/gates/kernel-*.sh'

jobs:
  kernel-build:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    strategy:
      fail-fast: false
      max-parallel: 8
      matrix:
        config: [defconfig, allnoconfig, allmodconfig, airy_defconfig]
        arch: [x86_64, arm64, riscv, loongarch]
        compiler: [gcc, clang]
        exclude:
          - arch: loongarch      # 上游 clang 尚不支持 loongarch
            compiler: clang
    steps:
      - uses: actions/checkout@v4
        with: { submodules: recursive }
      - name: Install build dependencies and toolchain
        run: ci/ci.sh kernel-deps --kind build --arch ${{ matrix.arch }} --compiler ${{ matrix.compiler }}
      - uses: hendrikmuhs/ccache-action@v1
        with:
          key: kernel-${{ matrix.config }}-${{ matrix.arch }}-${{ matrix.compiler }}
          max-size: 2G
      - name: Kbuild ${{ matrix.config }} / ${{ matrix.arch }} / ${{ matrix.compiler }}
        run: ci/ci.sh kernel-build --config ${{ matrix.config }} --arch ${{ matrix.arch }} --compiler ${{ matrix.compiler }} --log "$RUNNER_TEMP/build.log"

  kernel-verify:
    runs-on: ubuntu-latest
    needs: kernel-build
    timeout-minutes: 60
    env:
      KSFT_INSTALL: ${{ github.workspace }}/kselftest-install
    steps:
      - uses: actions/checkout@v4
        with: { submodules: recursive }
      - name: Install analysis and runtime test tools
        run: ci/ci.sh kernel-deps --kind verify
      - name: Consistency (matrix / ABI / codegen)
        run: ci/ci.sh kernel-consistency --kind all
      - name: KUnit on UML + baseline diff
        run: ci/ci.sh kernel-kunit --tap "$RUNNER_TEMP/kunit.tap"
      - name: Reference build (airy_defconfig / x86_64)
        run: ci/ci.sh kernel-refbuild --log "$RUNNER_TEMP/build.log"
      - name: checkpatch.pl --strict
        run: ci/ci.sh kernel-analyze --kind checkpatch --log "$RUNNER_TEMP/checkpatch.log"
      - name: sparse C=2 W=2 on Airymax objects
        run: ci/ci.sh kernel-analyze --kind sparse --log "$RUNNER_TEMP/sparse.log"
      - name: Coccinelle on Airymax directories
        run: ci/ci.sh kernel-analyze --kind coccinelle --log "$RUNNER_TEMP/cocci.log"
      - name: kselftest subset in virtme-ng
        run: ci/ci.sh kernel-kselftest --tap "$RUNNER_TEMP/kselftest.tap"
```

### 2.4 其他子仓 CI（按构建系统分类，子仓级 workflow）

> **归属说明**：下列 workflow 模板位于**各子仓自身的** `.github/workflows/` 目录中，不在管理仓 `agent-linux/.github/workflows/` 的 8 个 workflow 之内（§2.1）。每个子仓按自身构建系统选用对应模板，管理仓通过 `mgmt-verify.yml` 聚合各子仓 CI 状态（§2.2）。

非 kernel 子仓按构建系统分类，每类一个 workflow 模板，子仓按自身归属选用：

| 构建系统 | workflow（子仓级） | 典型语言 | 关键 job |
|---------|------------------|---------|---------|
| **Meson** | `ci-meson.yml` | C/C++ 用户态 | meson setup + ninja + meson test + clang-tidy |
| **Cargo** | `ci-cargo.yml` | Rust | cargo fmt --check + clippy --deny warnings + cargo test |
| **Go** | `ci-go.yml` | Go | go vet + golangci-lint + go test -race |
| **tsc** | `ci-tsc.yml` | TypeScript | tsc --noEmit + eslint + jest |

```yaml
# <subrepo>/.github/workflows/ci-cargo.yml（节选，子仓级 workflow）
jobs:
  cargo-build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with: { components: clippy, rustfmt }
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt -- --check
      - run: cargo clippy --all-targets --all-features -- -D warnings
      - run: cargo test --all-features --no-fail-fast
```

### 2.5 [SC] 共享契约层门禁（OS-IRON-008）

[SC] 层是 kernel 与用户态共用的 12 个头文件契约（OS-IRON-014）。权威副本为 `kernel/include/uapi/linux/airymax/`，变更由 kernel 子仓 CI 与 os-chk 三道契约门禁共同验证；用户态消费副本的跨仓一致性尚未纳入门禁，详见 §7。

---

## 3. 编译验证矩阵

### 3.1 矩阵定义（OS-STD-TOOL-161）

每个 PR 必须通过 **4 配置 × 4 架构 × 2 编译器 − 4（loongarch 暂不支持 clang，该架构的 4 个配置组合整体排除）= 28 配置矩阵** 的 Kbuild 构建。配置/架构/编译器三者皆绿方可合并——`allnoconfig` 暴露 `#ifdef` 错配、`allmodconfig` 暴露符号冲突、`defconfig` 验证默认路径、`airy_defconfig` 验证 Airymax 专属配置闭包。

| 维度 | 取值 | 数量 | 验证目标 |
|------|------|------|---------|
| **配置** | defconfig / allnoconfig / allmodconfig / airy_defconfig | 4 | `#ifdef` 错配 / 符号冲突 / 默认路径 / Airymax 配置闭包 |
| **架构** | x86_64 / arm64 / riscv / loongarch | 4 | 四架构 ABI 一致性 |
| **编译器** | GCC / Clang | 2 | 编译器无关性（loongarch+clang 组合暂不支持，见 exclude） |
| **合计** | — | **28** | — |

### 3.2 矩阵生成与调度

```yaml
# agent-linux/.github/workflows/kernel-verify.yml（矩阵段）
strategy:
  fail-fast: false          # 单格失败不取消其他格，便于一次性收集全部错误
  max-parallel: 8           # 28 格分批运行，避免超出 GitHub Actions 并发上限
  matrix:
    config: [defconfig, allnoconfig, allmodconfig, airy_defconfig]
    arch: [x86_64, arm64, riscv, loongarch]
    compiler: [gcc, clang]
    exclude:
      - arch: loongarch
        compiler: clang     # 上游工具链暂不支持，28 = 4×4×2−4
```

### 3.3 矩阵编译命令模板

```bash
# 通用模板：$ARCH / $CONFIG 由 matrix 注入；工具链前缀由编译器分支决定
#   gcc   → CROSS_COMPILE=<arch 前缀>（x86_64 为空）
#   clang → LLVM=1，且不得同时设置 CROSS_COMPILE（跨架构目标由 ARCH 推导）
mkargs=("CROSS_COMPILE=$CROSS")
[ "$COMPILER" = clang ] && mkargs=(LLVM=1)

# 配置生成：airy_defconfig 是叠加在架构 base defconfig 上的配置片段，不是 make 目标，
# 必须走 defconfig → merge_config.sh -m → olddefconfig 三步法；其余配置名可直接作为目标。
if [ "$CONFIG" = "airy_defconfig" ]; then
  make ARCH=$ARCH "${mkargs[@]}" defconfig
  ./scripts/kconfig/merge_config.sh -m .config "arch/$KDIR/configs/airy_defconfig"
  make ARCH=$ARCH "${mkargs[@]}" olddefconfig
else
  make ARCH=$ARCH "${mkargs[@]}" "$CONFIG"
fi

make ARCH=$ARCH "${mkargs[@]}" -j"$(nproc)" 2>&1 | tee build.log

# 模块构建（allmodconfig 跳过，已全部内建）
if [ "$CONFIG" != "allmodconfig" ]; then
  make ARCH=$ARCH "${mkargs[@]}" modules -j"$(nproc)" 2>&1 | tee -a build.log
fi

# 编译期门禁（对齐 A-4 完美主义）：error 一律阻断；
# warning 只对 Airymax 路径收敛阻断，避免上游既有 warning 把矩阵永久染红。
grep -E "error:" build.log && { echo "::error::compiler errors"; exit 1; }
airy_re='security/airy/|kernel/(superv|syscalls|corekern|log|ipc)/|/airymax/'
grep -E "warning:" build.log > warnings.log && grep -E "${airy_re}" warnings.log
```

### 3.4 矩阵失败语义

- **defconfig 失败**：默认路径损坏，P0 级阻塞——禁止任何绕过。
- **allnoconfig 失败**：`#ifdef CONFIG_*` 错配，遗漏 `depends on` 或 `select` 依赖——必须修复 Kconfig 依赖闭包。
- **allmodconfig 失败**：符号冲突 / 重复定义 / 模块间循环依赖——必须修复符号导出。
- **arm64/riscv/loongarch 失败但 x86_64 通过**：架构相关代码未做抽象——必须在 PR 描述中说明架构差异并经 L2 维护者 ACK。
- **Clang 失败但 GCC 通过**：GCC 扩展语法（如 `__attribute__` 非标准用法）——必须改为标准 C 或加 `#ifdef __clang__` 兼容。

### 3.5 矩阵优化

28 格全量构建在 GitHub Actions 上热缓存耗时约 40 分钟，冷缓存实测最慢格
62 分钟（airy_defconfig / arm64 / clang，2026-10-26）。优化策略：

- **定向 ccache**：GitHub 每仓缓存上限 10G，28 格各 2G 的名义总量 56G 必然
  被 LRU 淘汰（表现为 `Restore cache` → `No cache found`），因此只给最慢的
  `airy_defconfig` 七格建缓存（7×800M≈5.6G），其余格冷构建也在 30 分钟内。
- **路径过滤**：PR 仅触及 `Documentation/` 或 `tools/` 时跳过矩阵，仅跑文档格式校验。
- **并行分批**：28 格 `max-parallel: 8` 分批运行，每批 ≤20 分钟。

### 3.6 矩阵清单化与门禁（SSoT）

矩阵定义、排除项、调度约束、超时预算、编译门禁语义与失败归因收敛在
`agent-linux/tools/ci-matrix/manifest.yaml`，作为**唯一事实源**：

- `tools/ci-matrix/ci_matrix.py --check` 逐项交叉校验清单与
  `.github/workflows/kernel-verify.yml` 的 `kernel-build` job（维度、格数、
  `exclude`、`fail-fast`、`max-parallel`、超时、job 数、门禁 token），
  并以 OS-CHK 门禁 `OS-BUILD-CI-MATRIX`（P0）在 PR 中阻断漂移。
- 架构维度不在该清单内重复定义，统一取自
  `tools/defconfig-matrix/manifest.yaml` 中 `build: true` 的条目。
- `--summary <result.json>` 按 §3.4 语义对逐格结果归因，输出
  `ci-fail-compile` 标签，供 §10.1 失败分类复用；缺失格、重复格、
  未知维度取值与 `skipped` / `cancelled` 一律判失败。

---

## 4. 代码质量门禁

### 4.1 checkpatch.pl --strict（OS-STD-TOOL-160）

每个 PR 提交前必须通过 `scripts/checkpatch.pl --strict`。checkpatch 检查 Linux 6.6 内核基线规则 + agent-linux 扩展规则。

**门禁定级**：`--strict` 下只要有 ERROR/WARNING/CHECK 之一，checkpatch 就返回退出码 1，因此门禁**不能依赖退出码**，必须解析输出末行 `total: N errors, N warnings, N checks, N lines checked`。定级为 **ERROR 阻断合并 / WARNING 与 CHECK 仅报告**（L-11 归一化，避免历史遗留风格问题把门禁永久染红）。

```bash
# 对 Airymax 目录下的每个 .c/.h 文件单独检查
LC_ALL=C scripts/checkpatch.pl --strict --no-tree -f security/airy/airy_ipc.c
```

CI 门禁实现（文件清单来自 `find`，见 §2.3 说明）：

```yaml
- name: checkpatch --strict (OS-STD-TOOL-160)
  run: |
    mapfile -t files < <(find "${dirs[@]}" -type f \( -name '*.c' -o -name '*.h' \) | LC_ALL=C sort)
    err_total=0
    for f in "${files[@]}"; do
      out="$(LC_ALL=C scripts/checkpatch.pl --strict --no-tree -f "$f" 2>&1 || true)"
      total="$(printf '%s\n' "$out" | grep -m1 '^total:' || true)"
      errs="$(printf '%s\n' "$total" | sed -n 's/^total: \([0-9]\{1,\}\) errors.*/\1/p')"
      err_total=$((err_total + ${errs:-0}))
    done
    [ "$err_total" -gt 0 ] && exit 1    # 仅 ERROR 阻断，WARNING/CHECK 报告不阻断
```

### 4.2 sparse make C=2 + W 分级（OS-STD-TOOL-081）

sparse 是 Linux 内核基线的类型检查器，通过 `make C=2` 启用。W 分级体系区分执行环境：

| 子编号 | 环境 | 命令 | 语义 |
|--------|------|------|------|
| OS-STD-TOOL-081 | 基线 | `make C=2` | sparse 类型检查 + W 分级体系定义 |
| OS-STD-TOOL-081-CI | CI | `make C=2 W=2` | CI 强制检查新增代码，零 warning |
| OS-STD-TOOL-081-REL | 发布 | `make C=2 W=2` 全量 | 发布前全量审计（含历史代码） |

| W 级别 | 执行环境 | 触发条件 |
|--------|---------|---------|
| W=0 | — | 关闭额外警告 |
| W=1 | 本地开发 | 开发者本地 `make W=1`，建议性 |
| W=2 | CI（PR） | `make W=2 C=2`，强制零 warning（OS-STD-TOOL-081-CI） |
| W=2 | 发布 | `make W=2 C=2` 全量审计（OS-STD-TOOL-081-REL） |

```yaml
- name: sparse make C=2 W=2 on Airymax objects (OS-STD-TOOL-081-CI)
  run: |
    # 只对 Airymax 自己的目标文件触发 sparse，避免整树 C=2 超时；
    # 对象清单由构建产物推导，保证与编译配置一致。
    objs=()
    for c in $(find security/airy kernel/superv kernel/syscalls \
                 kernel/corekern kernel/log kernel/ipc -name '*.c'); do
      [ -f "${c%.c}.o" ] && objs+=("${c%.c}.o")
    done
    [ "${#objs[@]}" -eq 0 ] && { echo "::error::no Airymax objects built"; exit 1; }
    make ARCH=x86_64 C=2 W=2 -j"$(nproc)" "${objs[@]}" 2>&1 | tee sparse.log
    # 门禁只匹配 Airymax 路径，避免上游既有 sparse 报告把门禁永久染红
    pat='^(security/airy|kernel/(superv|syscalls|corekern|log|ipc))/[^:]+:[0-9]+:[0-9]+: (warning|error):'
    grep -E "$pat" sparse.log && exit 1
```

### 4.3 Coccinelle 语义补丁检查

Coccinelle（`make coccicheck`）使用语义补丁（`.cocci`）检测常见错误模式：NULL 解引用、双重锁、错误路径资源泄漏。CI 以 `MODE=report` 运行，落在 Airymax 路径上的任何报告即失败。

整树 `coccicheck` 在 CI 上必然超时，因此收敛为 **10 个脚本 × 6 个 Airymax 目录 = 60 次扫描**，每次只扫一个目录：

```yaml
- name: Coccinelle on Airymax directories
  run: |
    cocci_scripts=(
      scripts/coccinelle/misc/doubleinit.cocci
      scripts/coccinelle/misc/ifcol.cocci
      scripts/coccinelle/misc/minmax.cocci
      scripts/coccinelle/misc/returnvar.cocci
      scripts/coccinelle/misc/semicolon.cocci
      scripts/coccinelle/misc/uninitialized_var.cocci
      scripts/coccinelle/free/ifnullfree.cocci
      scripts/coccinelle/locks/double_lock.cocci
      scripts/coccinelle/null/deref_null.cocci
      scripts/coccinelle/null/kmerr.cocci
    )
    cocci_dirs=(security/airy kernel/superv kernel/syscalls \
                kernel/corekern kernel/log kernel/ipc)
    for script in "${cocci_scripts[@]}"; do
      for dir in "${cocci_dirs[@]}"; do
        # coccicheck 把 SPFLAGS 追加在 OPTIONS 末尾，spatch 冲突时取最后一个选项，
        # 由此把默认的 --dir $srctree 收窄到单个 Airymax 目录。
        make -s ARCH=x86_64 coccicheck MODE=report J=1 \
          COCCI="$script" SPFLAGS="--dir $dir" >> cocci.log 2>&1 || true
      done
    done
    grep -q '^coccicheck failed' cocci.log && exit 1
    pat='^(security/airy|kernel/(superv|syscalls|corekern|log|ipc))/[^:]+:[0-9]+:[0-9]+'
    grep -E "$pat" cocci.log && exit 1
```

> **规划项标识**：agent-linux 自有的语义补丁（AgentsIPC ring buffer 越界、share_pool 引用计数等）**尚未落地**——`kernel/scripts/coccinelle/` 下现有 71 个 `.cocci` 全部来自 Linux 6.6 上游，无 Airymax 专属脚本。上述 CI 阶段仅运行上游脚本，自定义规则列入后续迭代，落地后按同一 `SPFLAGS="--dir <dir>"` 收敛方式并入脚本清单。

### 4.4 clang-analyzer 静态分析

对用户态子仓（Meson/Cargo/Go/tsc）启用 clang-analyzer / clang-tidy（C/C++）与对应语言的静态分析器，作为 L2 静态分析的补充：

| 语言 | 工具 | 配置 |
|------|------|------|
| C/C++ | clang-tidy | `.clang-tidy`（继承 Linux 6.6 检查集 + agent-linux 扩展） |
| Rust | clippy | `clippy.toml`，`--deny warnings` |
| Go | golangci-lint | `.golangci.yml` |
| TypeScript | eslint | `.eslintrc.json` |

### 4.5 .clang-format 格式验证（OS-STD-FMT-001/002）

C/C++ 代码必须通过 `.clang-format`（同源 Linux 6.6 内核基线 689 行配置 + 560 ForEachMacros）格式验证：

- **OS-STD-FMT-001**：所有 `.c`/`.h` 文件必须通过 `clang-format --dry-run -Werror`，无格式差异。
- **OS-STD-FMT-002**：禁止提交格式修复与功能变更混合的 PR；格式修复必须独立 commit。

```yaml
- name: clang-format check (OS-STD-FMT-001/002)
  run: |
    git diff --name-only origin/${{ github.base_ref }} HEAD -- 'kernel/**/*.c' 'kernel/**/*.h' > files.txt
    while read f; do
      clang-format --dry-run --Werror "$f"
    done < files.txt
```

---

## 5. 测试覆盖矩阵

### 5.1 测试类型矩阵

| 测试类型 | 规则编号 | 执行环境 | 频率 | 阻断合并 |
|---------|---------|---------|------|---------|
| KUnit 单元测试 | OS-TEST-001~012 | UML | 每 PR | 是 |
| kselftest 系统测试 | OS-TEST-013~022 | QEMU | 每 PR | 是 |
| 形式化验证（seL4 风格） | — | CI（nightly） | 每日 | 否（工具未实现，nightly 降级 SKIP） |
| Soak 长时间运行 | — | CI（nightly） | 每日 | 否（工具未实现，nightly 降级 SKIP） |
| Chaos 混沌测试 | — | CI（nightly） | 每日 | 否（工具未实现，nightly 降级 SKIP） |

### 5.2 KUnit 单元测试（OS-TEST-001~012）

KUnit 是 agent-linux 内核态白盒单元测试框架，默认在 UML（User Mode Linux）上运行，毫秒级反馈：

- **OS-TEST-001**：所有 agent-linux 内核模块必须提供至少一个 KUnit 测试套件；无 KUnit 测试的模块禁止合入 `kernel` 主分支。
- **OS-TEST-010**：CI 必须解析 TAP 输出并上传至测试报告系统；`not ok` 与 `# SKIP` 必须分别计数。
- **OS-TEST-011**：所有 agent-linux Agent SDK 接口必须有 KUnit 契约测试（正常路径 + ≥1 异常路径 + ≥1 边界路径）。
- **OS-TEST-012**：PR 引入新 KUnit 套件时，CI 必须对比 PR 前后 TAP 用例数，总用例数必须单调递增。

```yaml
- name: KUnit on UML (OS-TEST-001~012)
  working-directory: kernel
  run: |
    ./tools/testing/kunit/kunit.py run --arch=um --raw_output=kunit \
      --kunitconfig=security/airy 2>&1 | tee "$GITHUB_WORKSPACE/kunit.tap"
- name: TAP case count monotonic (OS-TEST-012)
  run: python3 tools/kunit-tap-diff.py --current kunit.tap
```

> **次序约束**：`kunit.py` 使用独立的 `O=.kunit` 构建目录，但 kbuild 在检测到源码树存在 in-tree 构建残留时会报 `The source tree is not clean, please run 'make mrproper'`。因此 KUnit 步骤**必须排在 in-tree 内核构建之前**；若反过来，CI 必红。
>
> **参数说明**：`kunit.py` 的 `--raw_output` 仅在取值为 `all`（原样输出全部内容）或 `kunit`（只抽取 KUnit TAP 行）时有明确语义；其余取值会原样打印完整内核启动日志，导致下游 TAP 解析失败——旧文档的 `--raw_output=TAP` 即属此列。`--kunitconfig=security/airy` 直接指向唯一配置目录 `kernel/security/airy/.kunitconfig`，无需先 `make ARCH=um defconfig`。
>
> **基线比对**：`kunit-tap-diff.py` 的 `--baseline-file` 默认为 `tools/kunit-tap-baseline.tap`（版本受控的基线，8 套件 / 49 叶子），不再依赖 `origin/develop`——shallow clone 下该 ref 不可解析。PR 引入新套件时用 `--update` 刷新基线。

### 5.3 kselftest 系统测试（OS-TEST-013~022）

kselftest 是 agent-linux 系统级测试框架，默认在 QEMU 上运行（CI 容器无真实硬件）：

- **OS-TEST-013**：所有 agent-linux 内核特性必须有对应的 kselftest 子系统测试；无覆盖时 PR 评审必须显式标注"kselftest 豁免理由"。
- **OS-TEST-014**：kselftest 默认在 QEMU 运行；依赖真实硬件的测试必须用 `ksft_test_result_skip()` 显式跳过并标注原因。
- **OS-TEST-022**：flaky 测试必须在 `airy_flaky_baseline.md` 登记并限期修复；未修复的 flaky 测试由 CI 标记为 `KSFT_XFAIL`。

```yaml
      - name: kselftest subset in virtme-ng
        run: ci/ci.sh kernel-kselftest --tap "$RUNNER_TEMP/kselftest.tap"
```

> **执行载体**：构建安装、运行、TAP 解析三段合成一条门禁 `ci/gates/kernel-kselftest.sh`。理由不是少写 YAML，而是 install 目录是「构建」与「运行」之间的唯一契约：三段分开写时，任一段单独重跑都会在 install 目录过期的情况下照样通过。`run_kselftest.sh` 从自身位置推导 `BASE_DIR` 并读取同目录的 `kselftest-list.txt`，因此**必须从安装目录运行**；Linux 6.6 的该脚本只接受 `-s|--summary`、`-t`、`-c`、`-l`、`-d`、`-o`、`-h`，**不存在 `--tap` 参数**，旧文档的 `./tools/testing/kselftest/run_kselftest.sh --tap` 既路径错误又参数非法。
>
> **预算按目录落，不做全局覆盖**：上游预算有两层——裸机缺省 `kselftest_default_timeout=45`，目录可用 `settings` 改预算且 `timeout=0` 表示无界。`run_kselftest.sh -o` 是全局覆盖，会同时抹掉这两层：0.0.3 把 pidfd 从 45s 提到 240s 后，失败项立刻平移到 `rseq/run_param_test.sh`，因为该目录声明 `timeout=0`（上游把它当浸泡用）。门禁改为把预算写进**安装目录**的 `<dir>/settings`（不碰源码树，install 会覆盖回去），上游已声明有限预算的目录一律尊重（`seccomp` 保持 120s），并加整批墙钟上界（缺省 1200s）防止死锁吃掉 job 预算。
>
> **浸泡用例显式剔除且可审计**：`rseq/run_param_test.sh` 一轮约 608 次进程启动（8 组配置 + 9 个注入点 + 9 个阻塞相位 × 4 二进制 × 1000 reps × 6×ncpu 线程），不是回归信号。门禁在运行前按 `dir:name` 从清单剔除，并**先断言该名字确实在清单里**——名字写错则硬失败，避免「静默少跑一批用例」；剔除项与理由登记在验收报告，实机浸泡另立任务。
>
> **运行环境**：CI 容器无 `/dev/kvm` 时门禁自动追加 `--disable-kvm` 退回 TCG（本地有 KVM 则不做模拟）；x86_64 `defconfig` 未启用 `CONFIG_VIRTIO_FS`，故必须 `--force-9p` 走 9p 共享。**不加 `--summary`**：它把 logfile 指向 `$BASE_DIR/output.log`，而 virtme-ng 把安装目录只读挂载，整批用例会因写日志失败而一律 `exit=2`（0.0.3 实测 19/19 命中）。所选子集 `seccomp pidfd membarrier rseq` 均为自包含测试，缺 CONFIG 的用例以 `SKIP` 上报，而 `parse-tap.py` 将 `SKIP` 记为 `ok`，不误伤门禁。

### 5.4 形式化验证（seL4 风格）

借鉴 seL4 形式化验证思想（OS-IRON-012：seL4 借鉴仅限架构层），规划在 nightly build 中对关键内核子系统（调度器、IPC ring buffer、share_pool 引用计数）运行形式化验证。

> **未实现标注**：`formal-verify.py` 目前**不存在**（`tools/` 下未提供），`nightly.yml` 已降级为 `SKIP` 分支——脚本存在时执行并仅告警，不存在时打印 `SKIP: formal-verify.py not yet implemented (1.0.1+ verification path)`。本小节为规划描述，不构成已落地能力。

```yaml
# agent-linux/.github/workflows/nightly.yml（节选，实际结构）
nightly-test-suite:
  runs-on: ubuntu-latest
  timeout-minutes: 4320   # 72h max
  steps:
    - uses: actions/checkout@v4
        with: { submodules: recursive }
    - name: Formal verification (seL4-style)
      run: |
        if [ -f agent-linux/tools/formal-verify.py ]; then
          python3 agent-linux/tools/formal-verify.py --target kernel/sched/core.c || \
            echo "WARN: formal verification incomplete"
        else
          echo "SKIP: formal-verify.py not yet implemented (1.0.1+ verification path)"
        fi
```

### 5.5 Soak 与 Chaos 测试

- **Soak 测试**：规划 72 小时持续运行 agent-linux 工作负载，检测内存泄漏、引用计数漂移、timer 累积误差。
- **Chaos 测试**：规划在 QEMU 中注入故障（CPU 热插拔、内存 hotremove、磁盘 I/O 错误、网络分区），验证 agent-linux 的故障恢复路径。

> **未实现标注**：`soak-runner.py` / `chaos-runner.py` 目前**均不存在**，`nightly.yml` 已将 formal/soak/chaos 合并为单一 `nightly-test-suite` job 并降级为 `SKIP`（脚本不存在时打印 SKIP，存在时仅告警不阻断）。

```yaml
# agent-linux/.github/workflows/nightly.yml（节选，实际结构）
    - name: 72h soak test (memory leak / refcount drift)
      run: |
        if [ -f agent-linux/tools/soak-runner.py ]; then
          python3 agent-linux/tools/soak-runner.py --duration 72h --workload agent-busy-loop
        else
          echo "SKIP: soak-runner.py not yet implemented"
        fi

    - name: Chaos injection (CPU hotplug / mem hotremove / I/O error / net partition)
      run: |
        if [ -f agent-linux/tools/chaos-runner.py ]; then
          python3 agent-linux/tools/chaos-runner.py --profile full
        else
          echo "SKIP: chaos-runner.py not yet implemented"
        fi
```

### 5.6 覆盖率门槛

> **数值统一说明**：覆盖率门槛数值以 [`80-testing/06-coverage-metrics.md`](../80-testing/06-coverage-metrics.md) 的 4 级门槛为唯一权威（kernel/core ≥90%、security ≥95%、ipc ≥90%、关键路径 100%），并在 [`80-testing/README.md`](../80-testing/README.md) 汇总互引。下表与本卷其他章节出现覆盖率数值时，一律以 06 卷为准。

| 规则编号 | 范围 | 门槛 |
|---------|------|------|
| **OS-STD-TOOL-121** | 内核子系统（kernel/core） | 行覆盖率 ≥90% |
| **OS-STD-TOOL-122** | Agent SDK / security 层 | 行覆盖率 ≥95% |
| **OS-STD-TOOL-124** | 关键路径（fastpath 等 4 项） | 覆盖率 100% |

> **未实现标注**：`agent-linux/tools/coverage-gate.py` 与 `critical-paths.txt` 目前**均不存在**，覆盖率门禁尚未在 CI 落地；落地时按 06 卷 `airy_coverage_gate.sh` 方案执行（夜间/PR 门槛校验，任一模块跌破门槛即阻断）。以下为规划命令，非已运行流水线：

```yaml
- name: Coverage gate (OS-STD-TOOL-121/122/124)
  run: |
    make ARCH=x86_64 defconfig
    ./tools/testing/kunit/kunit.py run --arch=um --make_options=GCOV=1
    lcov --capture --directory . --output-file coverage.info
    python3 agent-linux/tools/coverage-gate.py \
      --info coverage.info \
      --kernel-subsystem 90 \
      --agent-sdk 95 \
      --critical-paths agent-linux/tools/critical-paths.txt:100
```

"关键路径"清单由维护者维护（规划路径 `agent-linux/tools/critical-paths.txt`），覆盖：调度器上下文切换、IPC 消息拷贝、share_pool 引用计数增减、LSM 钩子决策、capability 检查。

---

## 6. SSoT 校验流水线

### 6.1 流水线组成

SSoT 校验流水线确保全仓 `.md` 文档引用的规则编号与 SSoT 注册表一致，根除"私自定义编号"与"引用已废弃编号"两类违规：

| 组件 | 路径 | 职责 |
|------|------|------|
| YAML Schema | `agent-linux/ssot-registry.yaml`（管理仓根） | 机器可读注册表（全部已登记编号） |
| Python 校验脚本 | `agent-linux/tools/validate-ssot.py` | 扫描 .md + 比对 YAML |
| CI workflow | `agent-linux/.github/workflows/ssot-verify.yml` | 每 PR + 每 push 触发 |

### 6.2 校验逻辑

`validate-ssot.py` 的校验逻辑：

1. 加载 `ssot-registry.yaml`（管理仓根），构建已登记编号集合 `registered`（含 `status: active` 与 `status: deprecated`）。
2. 递归扫描仓库内所有 `.md` 文件，正则提取 `OS-[A-Z]+(-[A-Z]+)?-\d+` 形式的规则 ID，构建 `referenced` 集合。
3. 比对：
   - **未登记引用**：`referenced - registered` → 文档引用了注册表中不存在的编号，P0 级违规（私自定义编号）。
   - **已废弃引用**：引用 `status: deprecated` 的编号 → P1 级违规（应迁移到替代编号）。
   - **格式错误**：不符合 `OS-<前缀>(-<子域>)-<NNN>` 格式的编号 → P1 级违规。

```yaml
# agent-linux/.github/workflows/ssot-verify.yml
name: ssot-verify
on:
  pull_request:
    paths:
      - '**/*.md'
      - 'ssot-registry.yaml'
      - 'agent-linux/tools/validate-ssot.py'
  push:
    branches: [main, develop]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - name: Install PyYAML
        run: pip install pyyaml
      - name: Validate rule IDs against SSoT registry
        run: |
          python3 agent-linux/tools/validate-ssot.py \
            --md-root . \
            --yaml ssot-registry.yaml \
            --strict
```

### 6.3 校验脚本核心实现（伪代码）

```python
# agent-linux/tools/validate-ssot.py（核心逻辑节选）
import re, sys, yaml
from pathlib import Path

RULE_ID_RE = re.compile(r'OS-[A-Z]+(?:-[A-Z]+)?-\d+')
# 已登记前缀：OS-IRON / OS-KER / OS-STD-CODE / OS-STD-FMT / OS-STD-STY /
#             OS-STD-GOV / OS-STD-TOOL / OS-STD-PROD / OS-BAN / OS-ACC /
#             OS-ABI / OS-SEC / OS-ARCH / OS-IFACE / OS-TEST / OS-STD-TEST

def load_registered(yaml_path):
    data = yaml.safe_load(Path(yaml_path).read_text())
    registered = {}  # id -> status
    # 解析 individual_entries 与 ranges，展开为完整 id 集合
    for section in data['registry']:
        for entry in section.get('entries', []):
            registered[entry['id']] = entry.get('status', 'active')
        for entry in section.get('individual_entries', []):
            registered[entry['id']] = entry.get('status', 'active')
        for rng in section.get('ranges', []):
            # 展开 OS-STD-TOOL-001~099 / 101~161 等 range
            ...
    return registered

def scan_references(md_root):
    referenced = {}  # id -> [files]
    for md in Path(md_root).rglob('*.md'):
        for match in RULE_ID_RE.finditer(md.read_text()):
            referenced.setdefault(match.group(0), []).append(str(md))
    return referenced

def main():
    registered = load_registered(sys.argv['--yaml'])
    referenced = scan_references(sys.argv['--md-root'])
    violations = []
    for rule_id, files in referenced.items():
        if rule_id not in registered:
            violations.append(f"[P0] 未登记编号 {rule_id} 引用于 {files}")
        elif registered[rule_id] == 'deprecated':
            violations.append(f"[P1] 已废弃编号 {rule_id} 引用于 {files}")
    if violations:
        print('\n'.join(violations), file=sys.stderr)
        sys.exit(1)
```

### 6.4 SSoT 校验与 7 层验证的关系

SSoT 校验独立于 7 层验证体系，是横切关注点：它不验证代码质量，而验证"文档与注册表的一致性"。SSoT 校验失败属于 §10 中的"SSoT 违规"类别，阻断合并。

---

## 7. [SC] 共享契约层（OS-IRON-008）

### 7.1 双仓副本模型

[SC] 共享契约层是 kernel 与用户态共同遵守的头文件契约（OS-IRON-014）。权威副本位于 kernel 子仓 `kernel/include/uapi/linux/airymax/`；用户态仓 `agent-workload` 在 `agentrt/commons/include/airymax/` 维护消费副本，随 SDK 分发。两份副本的许可证头必然不同（内核侧 `GPL-2.0-only WITH Linux-syscall-note`，用户态侧 `BSD-3-Clause OR GPL-2.0`），故「逐字节相同」不是可达目标，契约一致性以声明与常量取值是否一致为准。

权威副本目录下实际共 18 个文件：14 个 `.h`（12 个 [SC] 头文件 + 2 个 codegen 派生头 `structures_32_gen.h`/`structures_64_gen.h`）+ `syscall.xml`（R-01 SSoT）、`structures_32.bf`/`structures_64.bf`（R-02 SSoT）、`Kbuild`。

`syscall_gen.h`/`bpf_struct_ops.h` 是两个补充共享文件，与 10 个核心头同等强制：

| 序号 | 文件 | 共享内容 |
|------|------|---------|
| 1 | `kernel/include/uapi/linux/airymax/error.h` | 错误码 SSoT |
| 2 | `kernel/include/uapi/linux/airymax/log_types.h` | 统一日志 5 级枚举/128B 记录 |
| 3 | `kernel/include/uapi/linux/airymax/ipc.h` | AgentsIPC 消息协议 |
| 4 | `kernel/include/uapi/linux/airymax/sched.h` | 调度器策略枚举 |
| 5 | `kernel/include/uapi/linux/airymax/memory_types.h` | share_pool 内存语义 |
| 6 | `kernel/include/uapi/linux/airymax/security_types.h` | LSM 钩子/capability 类型 |
| 7 | `kernel/include/uapi/linux/airymax/cognition_types.h` | 认知模块类型 |
| 8 | `kernel/include/uapi/linux/airymax/syscalls.h` | syscall 编号声明 |
| 9 | `kernel/include/uapi/linux/airymax/syscall_gen.h` | syscall 编号宏（codegen 产物，`__NR_airy_sys_*` 548-551） |
| 10 | `kernel/include/uapi/linux/airymax/uapi_compat.h` | 用户态兼容桥接 |
| 11 | `kernel/include/uapi/linux/airymax/lsm_types.h` | LSM 专用类型 |
| 12 | `kernel/include/uapi/linux/airymax/bpf_struct_ops.h` | BPF 结构操作 |

> 注：`syscall_gen.h`（`__NR_airy_sys_*` 548-551）内容由 `syscall.xml`（R-01 SSoT）经 `tools/codegen/syscall_gen.py` 生成并提交入库，禁止手工修改；漂移由 OS-IRON-009-CODEGEN 门禁拦截。

### 7.2 三道强制检查

| 检查 | 承载 | 拦什么 |
|------|------|--------|
| 头文件齐备 + 仓内唯一副本 | `mgmt-files` 门禁（`mgmt-verify.yml`） | 12 个 [SC] 头缺失；`ipc.h` 在本仓内被复制出第二份导致语义分叉 |
| 五路编译与布局一致 | OS-CHK-DOC-04（`sc_tri_compile.py --check`，`os-chk-verify.yml`） | 头文件在 kernel / 用户态 / 第三方 / [DSL] 两条降级路径下编译失败或类型宽度、字段偏移漂移；`syscalls.h` ↔ `syscall.xml` ↔ `syscall_gen.h` 双向不一致 |
| syscall ABI 稳定 | OS-IRON-001（`abi_check.py --check`，`os-chk-verify.yml`） | 已发布编号被改动、基线快照漂移 |

上表三道检查只覆盖权威副本。头文件位于 `kernel` 子仓内部，其变更由该子仓的 `kernel-verify.yml` 与管理仓上述门禁共同覆盖；用户态 `agentrt/commons/include/airymax/` 副本不在本仓 CI 覆盖范围内，跨仓一致性在 0.0.3 尚无门禁，已登记为 0.0.4 的 `SC-01`。

### 7.3 已退役的镜像 PR 机制

0.0.2 及更早版本用 `sc-verify.yml` 在用户态镜像仓创建同名 PR 并轮询对端 CI。0.0.3 起该机制移除，原因有二：

- 镜像 PR 的触发路径写在管理仓 `pull_request.paths` 上，而这些路径全在 `kernel` gitlink 内部，父仓的 glob 永不命中，workflow 实际不可达；
- 用户态副本随 `agent-workload` 独立发版后，两端许可证头必然不同，「逐字节 diff」的判定本身失效。

未来若要恢复跨仓校验，做法不是加回镜像 PR，而是在权威侧生成声明归一化后的哈希账本（`tools/sc-ledger`），由消费侧在自己的 CI 里比对同一份账本，既不需要跨仓 token，也不受 gitlink 路径限制。

### 7.4 回归路径

门禁红即阻断：`mgmt-files` 失败说明契约头缺失或被复制出第二份，先删副本再谈变更；OS-CHK-DOC-04 失败说明改动破坏了五路编译或双向编号一致，必须改契约源（`syscall.xml`）后重新生成，不允许手改生成物；OS-IRON-001 失败说明动了已发布的 syscall 编号，属于破坏性变更，按 `05-development-process.md` 走 L3 审批并升主版本。

---

## 8. 发布流水线

### 8.1 7 层验证对应关系

发布流水线对应 7 层验证的第 4~7 层：

| 层级 | 发布阶段 | 执行内容 |
|------|---------|---------|
| L4 | CI 门禁 | release 候选分支通过 28 矩阵 + 全套质量门禁 |
| L5 | 连接验证 | ABI 兼容性回归（与上一 release 对比） |
| L6 | 协议验证 | AgentsIPC 协议契约一致性 + syscall 语义映射校验 |
| L7 | 发布验证 | SBOM 生成 + 签名构建 + 发布渠道分发 |

### 8.2 触发条件

```yaml
# agent-linux/.github/workflows/release.yml
on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'        # 稳定版 v0.2.8
      - 'v[0-9]+.[0-9]+.[0-9]+-rc[0-9]+' # 候选版 v0.2.8-rc1
  workflow_dispatch:
    inputs:
      tag:
        description: 'Release tag'
        required: true
```

### 8.3 SBOM 生成（SPDX 格式）

```yaml
sbom:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
      with: { submodules: recursive }
    - name: Generate SPDX SBOM
      run: |
        # 内核态：扫描 vmlinux + 模块符号
        syft kernel/ -o spdx-json > sbom-kernel.spdx.json
        # 用户态：扫描 8 子仓依赖
        for repo in agentrt sdk ecosystem products ...; do
          syft "$repo/" -o spdx-json > "sbom-${repo}.spdx.json"
        done
        # 合并为统一 SBOM
        python3 agent-linux/tools/merge-sbom.py \
          --output sbom-airymaxos-${{ github.ref_name }}.spdx.json \
          sbom-*.spdx.json
    - name: Upload SBOM artifact
      uses: actions/upload-artifact@v4
      with:
        name: sbom-${{ github.ref_name }}
        path: sbom-airymaxos-${{ github.ref_name }}.spdx.json
```

### 8.4 签名构建（GPG / cosign）

发布产物必须双重签名：内核镜像用 GPG 签名，OCI 镜像用 cosign 签名。

```yaml
sign:
  runs-on: ubuntu-latest
  needs: sbom
  steps:
    - uses: actions/checkout@v4
    - name: Import GPG signing key
      uses: crazy-max/ghaction-import-gpg@v6
      with:
        gpg_private_key: ${{ secrets.GPG_PRIVATE_KEY }}
        passphrase: ${{ secrets.GPG_PASSPHRASE }}
    - name: Sign kernel image + modules (GPG)
      run: |
        make ARCH=x86_64 -j$(nproc) binrpm-pkg
        gpg --batch --yes --detach-sign --armor \
          -o airymaxos-kernel-${{ github.ref_name }}.rpm.asc \
          airymaxos-kernel-${{ github.ref_name }}.rpm
    - name: Sign OCI image (cosign)
      env:
        COSIGN_KEY: ${{ secrets.COSIGN_KEY }}
        COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
      run: |
        cosign sign --key env://COSIGN_KEY \
          ghcr.io/openairymax/airymaxos:${{ github.ref_name }}
```

### 8.5 发布渠道

| 渠道 | 产物 | 工具 | 目标 |
|------|------|------|------|
| **dnf repo** | kernel RPM + 模块 RPM | createrepo + 推送至 dnf 仓库 | Fedora/RHEL 系发行版 |
| **OCI image** | airymaxos 容器镜像 | docker build + push ghcr.io | 云原生部署 |
| **SDK tarball** | Agent SDK（多语言绑定） | tar + gpg 签名 | 开发者本地集成 |

```yaml
publish:
  runs-on: ubuntu-latest
  needs: sign
  steps:
    - uses: actions/checkout@v4
    - name: Publish dnf repo
      run: |
        createrepo airymaxos-repo/${{ github.ref_name }}/
        rsync -avz airymaxos-repo/ ${{ secrets.DNF_REPO_HOST }}:/var/www/airymaxos-repo/
    - name: Publish OCI image
      run: |
        docker build -t ghcr.io/openairymax/airymaxos:${{ github.ref_name }} .
        docker push ghcr.io/openairymax/airymaxos:${{ github.ref_name }}
    - name: Publish SDK tarball
      run: |
        tar czf airymaxos-sdk-${{ github.ref_name }}.tar.gz sdk/
        gpg --batch --yes --detach-sign --armor \
          -o airymaxos-sdk-${{ github.ref_name }}.tar.gz.asc \
          airymaxos-sdk-${{ github.ref_name }}.tar.gz
    - name: Create GitHub Release
      uses: softprops/action-gh-release@v2
      with:
        files: |
          airymaxos-kernel-*.rpm
          airymaxos-kernel-*.rpm.asc
          airymaxos-sdk-*.tar.gz
          airymaxos-sdk-*.tar.gz.asc
          sbom-airymaxos-*.spdx.json
        generate_release_notes: true
```

---

## 9. CI 基础设施

### 9.1 运行环境

所有 CI job 运行于 GitHub Actions `ubuntu-latest`（22.04）。kernel 子仓因矩阵规模最大，是 CI 总时长的主要约束对象。

| 子仓 | 运行器 | 备注 |
|------|--------|------|
| kernel | ubuntu-latest（2 核 7G） | 矩阵 28 格，需 ccache 加速 |
| agentrt | ubuntu-latest | 用户态全量 |
| sdk | ubuntu-latest | 多语言绑定矩阵 |
| 其余 5 子仓 | ubuntu-latest | 按构建系统分类 |

### 9.2 缓存策略

三级缓存减少 70%+ 重复构建时间：

| 缓存 | 工具 | key 策略 | 容量 |
|------|------|---------|------|
| **ccache** | hendrikmuhs/ccache-action | `kernel-{config}-{arch}-{compiler}` | 2G/格 |
| **cargo cache** | Swatinem/rust-cache | `cargo-{hash(Cargo.lock)}` | 1G |
| **pip cache** | actions/setup-python cache | `pip-{hash(requirements.txt)}` | 500M |

```yaml
# 统一缓存配置示例
- uses: hendrikmuhs/ccache-action@v1
  with:
    key: kernel-${{ matrix.config }}-${{ matrix.arch }}-${{ matrix.compiler }}
    max-size: 2G
- uses: Swatinem/rust-cache@v2
  if: matrix.compiler == 'cargo'
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'
```

### 9.3 并行度

| 维度 | 策略 |
|------|------|
| **8 子仓** | 并行（各子仓独立 workflow，互不阻塞） |
| **kernel 28 矩阵** | `max-parallel: 8`（分批运行） |
| **管理仓 2 job** | 串行（file-integrity → orchestrate-leaf-ci） |
| **[SC] 契约门禁** | `mgmt-files` → os-chk 三道检查串行（清单 → 五路编译 → ABI） |

### 9.4 超时约束（OS-STD-TEST-011）

CI 总时长稳态不得超过 60 分钟；唯一例外是 kernel 28 矩阵 job 的**冷缓存兜底
90 分钟**（ccache 未命中时最慢格实测 62 分钟，见 §3.5）。各阶段预算：

| 阶段 | 预算 | 备注 |
|------|------|------|
| kernel 28 矩阵 | 40 分钟 | ccache 命中后 ≤25 分钟 |
| kernel 矩阵（冷缓存兜底） | 90 分钟 | 硬上限，仅该 job 适用 |
| kernel 质量门禁（checkpatch/sparse/Coccinelle） | 8 分钟 | 串行 |
| kernel KUnit + kselftest | 10 分钟 | UML + QEMU |
| 其他 7 子仓（并行） | 15 分钟 | 与 kernel 并行，不占 kernel 预算 |
| 管理仓 2 job（串行） | 5 分钟 | 与子仓并行 |
| **总计（关键路径）** | **≤60 分钟** | kernel 为关键路径，冷缓存轮次 ≤90 分钟 |

```yaml
# 每个 job 显式声明 timeout-minutes，避免悬挂 job 拖垮总时长
kernel-build:
  runs-on: ubuntu-latest
  timeout-minutes: 90   # 取值由 tools/ci-matrix/manifest.yaml 唯一承载
```

超时处理：单 job 超时计入失败；CI 总时长超 60 分钟由 `nightly.yml` 的
`ci-budget` job 检测并告警，超时子集必须拆分或缓存优化（OS-STD-TEST-011）。
矩阵 job 的预算值以 `tools/ci-matrix/manifest.yaml` 的 `budget.timeout_minutes`
为唯一事实源，`tools/ci-matrix/ci_matrix.py --check` 校验 workflow 与清单一致。

---

## 10. 失败处理与回滚

### 10.1 CI 失败分类

| 类别 | 触发条件 | 严重级别 | 处理 |
|------|---------|---------|------|
| **编译错误** | 28 矩阵任一格 `make` 非零退出 | P0 | 阻断合并，PR 标记 `ci-fail-compile` |
| **测试失败** | KUnit `not ok` / kselftest 非 0 退出 / 覆盖率未达门槛 | P0 | 阻断合并，PR 标记 `ci-fail-test` |
| **静态分析警告** | sparse W=2 warning / Coccinelle report / clang-tidy warning | P0 | 阻断合并，PR 标记 `ci-fail-static` |
| **格式违规** | checkpatch ERROR/WARNING / clang-format 差异 | P0 | 阻断合并，PR 标记 `ci-fail-format` |
| **SSoT 违规** | 引用未登记/已废弃编号 / 格式错误 | P0 | 阻断合并，PR 标记 `ci-fail-ssot` |
| **[SC] 契约门禁失败** | 头文件缺失或出现第二份物理副本 / 五路编译或编号双向不一致 | P0 | 阻断合并 |
| **flaky 测试** | 同一 PR 重复运行结果不一致 | P1 | 登记 `airy_flaky_baseline.md`，限期修复 |

### 10.2 自动回滚策略

- **PR 合并前失败**：GitHub branch protection 自动阻止合并，无需手动回滚。PR 作者修复后重新推送触发 CI。
- **PR 合并后 develop 红型（nightly 检出）**：nightly build 检出回归时，自动创建 revert PR 并 @ 原作者；revert PR 走快速通道（仅跑 L1~L4，跳过 L5~L6）。
- **release tag 发布后检出 P0 缺陷**：发布流水线不自动回滚已发布产物，而是发布 `*.errata.md` 安全公告 + 在下一 patch release 修复；若缺陷为安全漏洞，触发紧急 hotfix 流水线（绕过 nightly，直接 L1~L4 + L7）。
- **[SC] 头文件破坏向后兼容**：自动阻止 kernel PR 合并（§7.4）；若已合并，nightly 的 ABI 兼容性回归（L5）会检出并阻断后续 develop 合并。

```yaml
# nightly 自动 revert 逻辑（节选）
auto-revert-on-regression:
  runs-on: ubuntu-latest
  if: failure()
  steps:
    - uses: actions/checkout@v4
    - name: Identify regressing commit (git bisect)
      run: python3 agent-linux/tools/auto-bisect.py --baseline last-green
    - name: Create revert PR
      run: python3 agent-linux/tools/create-revert-pr.py --regression-commit $COMMIT
```

### 10.3 通知机制

CI 失败通过 GitHub PR review comment 精准通知，避免全局噪音：

| 场景 | 通知对象 | 形式 |
|------|---------|------|
| 编译/测试/静态分析失败 | PR 作者 + 子系统维护者（基于 `CODEOWNERS`） | PR review comment + check annotation |
| SSoT 违规 | PR 作者 + 文档维护者（`50-engineering-standards/` CODEOWNERS） | PR review comment，列出违规编号与文件 |
| [SC] 契约门禁失败 | PR 作者 + kernel L3 维护者 | PR review comment + 门禁日志 |
| nightly 回归 | 引入回归的 commit 作者 + 顶级维护者 | revert PR comment + 邮件列表 |
| CI 总时长超 60 分钟（kernel 矩阵冷缓存轮次超 90 分钟） | CI 基础设施维护者 | issue 自动创建 |

```yaml
- name: Notify on failure (PR review comment)
  if: failure()
  uses: actions/github-script@v7
  with:
    script: |
      const fs = require('fs');
      const log = fs.readFileSync('sparse.log', 'utf8');
      await github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `:x: **CI 失败（静态分析警告，OS-STD-TOOL-081-CI）**\n\n\`\`\`\n${log.slice(0, 4000)}\n\`\`\`\n\n请修复 sparse warning 后重新推送。`
      });
```

---

## 11. 五维原则映射

| 原则 | 在本卷的体现 |
|------|------------|
| **A-4 完美主义** | 7 层验证全绿方可合并；28 矩阵零警告；关键路径 100% 覆盖（OS-STD-TOOL-124） |
| **S-1 反馈闭环** | L3 本地预提交毫秒级 + L4 CI 分钟级 + nightly 回归自动 revert |
| **S-2 层次分解** | 7 层按"局部→全局→跨仓→发布"递进；8 子仓 CI 隔离 |
| **E-3 资源确定性** | 三级缓存 + CI ≤60 分钟、kernel 矩阵冷缓存兜底 90 分钟（OS-STD-TEST-011）+ 可重现发布 |
| **E-7 文档即代码** | SSoT YAML Schema 机器可读 + .md 规则 ID 自动校验 |
| **K-4 可插拔策略** | 子仓按构建系统分类，CI job 独立增删 |
| **E-1 安全内生** | 发布双重签名（GPG + cosign）+ SBOM 全链路追溯 |

---

## 12. 相关文档

- [`70-build-system/README.md`](README.md)（父文档：构建系统主索引）
- [`70-build-system/01-kbuild-system.md`](01-kbuild-system.md)（Kbuild 递归构建）
- [`70-build-system/02-kconfig-system.md`](02-kconfig-system.md)（Kconfig 配置系统）
- [`50-engineering-standards/09-ssot-registry.md`](../50-engineering-standards/09-ssot-registry.md)（SSoT 规则编号权威注册表，文档体系 Markdown 源）
- [`agent-linux/ssot-registry.yaml`](../../../agent-linux/ssot-registry.yaml)（SSoT 机器可读 Schema，管理仓根）
- [`50-engineering-standards/05-development-process.md`](../50-engineering-standards/05-development-process.md)（补丁生命周期 + 7 层验证前 4 层强制）
- [`50-engineering-standards/120-cross-project-code-sharing.md`](../50-engineering-standards/120-cross-project-code-sharing.md)（[SC] 共享契约层 12 个头文件，OS-IRON-014）
- [`80-testing/01-kunit-framework.md`](../80-testing/01-kunit-framework.md)（KUnit，OS-TEST-001~012）
- [`80-testing/02-kselftest.md`](../80-testing/02-kselftest.md)（kselftest，OS-TEST-013~022 + OS-STD-TEST-011）

---

## 13. 参考材料

- Linux 6.6 `scripts/checkpatch.pl`（编码规范检查）
- Linux 6.6 `scripts/coccinelle/`（语义补丁）
- Linux 6.6 `Documentation/dev-tools/sparse.rst`（sparse 类型检查）
- Linux 6.6 `tools/testing/kunit/`（KUnit 框架）
- Linux 6.6 `tools/testing/kselftest/`（kselftest 框架）
- Linux 6.6 `.clang-format`（689 行 + 560 ForEachMacros）
- SPDX Specification 2.3（SBOM 格式）
- cosign（OCI 镜像签名）
- GitHub Actions Documentation（workflow 语法 + branch protection）

---

> **文档结束** | agent-linux CI/CD 管道完整设计 v1.0.1 | 1.0.1 开发阶段基线
