Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agent-linux（AirymaxOS）CI 强制点与合并门禁
> **文档定位**：120-development-process 模块第 8 卷。把 [01 卷补丁生命周期](01-patch-lifecycle.md) 中的「必须通过 CI」翻译成可执行的阻断规则——哪个阶段、必须过哪些门禁、失败如何处置。\
> **文档版本**：v1.1.0\
> **最后更新**：2026-10-29\
> **上级文档**：[120-development-process README](README.md)\
> **同源映射**：Linux 6.6 内核 CI（kernelci.org + Intel 0-Day）\
> **核心约束**：workflow 是薄壳；判定落在 `ci/gates/`；8 个 workflow 协同，不存在独立的 `[SC]` 契约工作流

---

## 1. 定位与权威源边界

### 1.1 三段式职责划分

同一件事不允许在两个地方各写一遍。CI/CD 主题按下列边界拆分，本文只负责最后一行：

| 主题 | 唯一权威源 | 本文是否复制 |
|------|-----------|-------------|
| workflow 文件清单、job 结构、触发路径 | [`.github/workflows/README.md`](../../../agent-linux/.github/workflows/README.md) | 否 |
| 门禁注册表（id / severity / authority / executor） | [`tools/os-chk/gates.yaml`](../../../agent-linux/tools/os-chk/gates.yaml) | 否 |
| 编译矩阵、静态分析、测试矩阵、SBOM、签名、缓存 | [70-build-system/03-ci-cd-pipeline.md](../70-build-system/03-ci-cd-pipeline.md) | 否 |
| 生命周期阶段 ↔ 门禁的阻断关系、分支保护、失败处置、预算棘轮 | **本文** | — |

### 1.2 判定逻辑的落点

`.github/workflows/*.yml` 里的每个 step 只有三种形态：`actions/checkout`、`ci/ci.sh <gate>`、`actions/upload-artifact`。只要出现第四种形态（内联 shell 断言），就违反薄壳原则，由 `ci/check-workflows.py` 在 `run-lint` 中拦下。

因此排障不需要重跑云端 job：同一条 `ci/ci.sh <gate>` 可以在本地对遗留的构建目录复放，这是本文与 70-build-system/03 共同的技术前提。

### 1.3 适用范围

agent-linux 管理仓的 8 个子仓（kernel / distro / system / security / services / memory / cognition / cloudnative）的语言级 CI 由各叶子仓自己的 `.github/workflows/` 承担；管理仓只做编排与门禁聚合。本文描述的是管理仓这一层。

---

## 2. 八个 workflow 与生命周期阶段的对应

| workflow | 触发 | 阻断级别 | 承接的生命周期阶段 |
|----------|------|---------|------------------|
| `kernel-verify.yml` | push / PR（`kernel` gitlink、`ci/gates/kernel-*.sh`、`ci/lib/**`） | 阻断 | 补丁进入评审前：编译矩阵、静态分析、KUnit、kselftest |
| `airy-verify.yml` | push / PR（同上） | 阻断 | Airymax 专属代码（`security/airy/`、`kernel/superv/`）的 CONFIG / BTF / 符号完整性 |
| `os-chk-verify.yml` | push / PR（`tools/**`、`sdk/**`、`ci/**`、四个受审 gitlink） | 阻断 | 门禁表执行 + 全量 Python 工具链测试 |
| `mgmt-verify.yml` | PR（治理文件、`.github/**`、`kernel`）；push `main` | 阻断 | 治理文件完整性、12 个 `[SC]` 头唯一副本、子仓 CI 聚合 |
| `ssot-verify.yml` | push / PR（`**.md`、`ssot-registry.yaml`） | 阻断 | SSoT 规则 ID 注册、跨文档引用、相对路径链接 |
| `image-build.yml` | `workflow_dispatch` | 阻断（发布门，非 PR 门） | 发行镜像组装 + QEMU 安装/冷启动验收 |
| `nightly.yml` | cron `0 2 * * *` / `workflow_dispatch` | **不阻断** | 回归哨兵：形式化验证、soak、chaos、自动回退 |
| `release.yml` | tag `v*` / `v*-rc*` / `workflow_dispatch` | 阻断（影响发布） | SBOM、内核 RPM、签名、ghcr 推送、Release 发布 |

workflow 的文件名、职责与 job 明细见 [`.github/workflows/README.md`](../../../agent-linux/.github/workflows/README.md)，此处不重复。

---

## 3. 门禁注册表与阻断语义

### 3.1 单入口

所有门禁只有一个入口：`ci/ci.sh <gate> [args]`，其实现是 `exec bash ci/gates/<gate>.sh "$@"`。`ci/ci.sh --list` 是门禁清单的可执行答案——文档与 `--list` 不一致时，以 `--list` 为准并修文档。

### 3.2 注册门禁（`gates.yaml` v1.9.0，15 项，全部 P0）

| id | 名称 | executor |
|----|------|----------|
| `OS-CHK-DOC-01` | Copyright 头部 | `tools/check-copyright.py --root .` |
| `OS-CHK-DOC-04` | `[SC]` 头文件清单一致性（五路编译） | `tools/sc-tri-compile/sc_tri_compile.py --check` |
| `OS-IRON-001` | syscall ABI 稳定性 | `tools/abi-check/abi_check.py --check` |
| `OS-IRON-015` | SSoT 规则 ID 注册一致性 | `tools/validate-ssot.py . ssot-registry.yaml` |
| `OS-IRON-009-CODEGEN` | codegen 产物零漂移 | `tools/codegen/run_codegen.py --check` |
| `OS-IRON-009-DSL` | 策略 DSL 决策表零漂移 | `tools/dsl/dsl_compile.py --check` |
| `OS-TEST-066` | Airymax 定制 checkpatch 扩展规则 | `kernel/scripts/airy_checkpatch_ext.pl` |
| `OS-BUILD-MATRIX` | defconfig 构建矩阵一致性 | `tools/defconfig-matrix/defconfig_matrix.py --check` |
| `OS-BUILD-CI-MATRIX` | CI 编译矩阵一致性 | `tools/ci-matrix/ci_matrix.py --check` |
| `OS-BUILD-BTF` | BTF / eBPF 基线一致性 | `tools/btf-integration/btf_integration.py --check` |
| `OS-BUILD-KBUILD-VERSION` | Kbuild 版本注入一致性 | `tools/kbuild-version/kbuild_version.py --check` |
| `OS-BUILD-RELEASE` | 发布物料清单一致性 | `tools/release-engineering/release_manifest.py --check` |
| `OS-BUILD-NIGHTLY` | nightly 契约一致性 | `tools/nightly/nightly.py --check` |
| `OS-BUILD-REPO-SCAFFOLD` | 八子仓脚手架一致性 | `tools/repo-scaffold/repo_scaffold.py --check` |
| `OS-OPS-ROLLBACK` | 版本回滚契约一致性 | `tools/airy-rollback/airy_rollback.py check` |

不在注册表内、由 workflow 直接调用的辅助门禁（`mgmt-files`、`mgmt-leaves`、`distro-lint`、`kernel-*` 系列、`assemble`、`accept`、`iso-publish` 等）属于构建与集成载荷，其契约由上面的注册表门禁反向约束：例如 `assemble` 的产物必须被 `OS-BUILD-RELEASE` 的物料清单登记。

### 3.3 阻断语义

- **P0 阻断**：门禁失败 ⇒ 该 PR 不可合并，`main` 上的 push 会留下红色状态。
- **不阻断**：只有 `nightly.yml` 属于此类，理由是它跑的是小时级全量与形式化载荷，放进 PR 路径会把反馈延迟拉到不可接受（见 §6.1）。
- **不得降级**：没有任何一条运行路径允许把 P0 门禁改成 `continue-on-error`。紧急处置走 §4.2 的回退通道，而不是关掉门禁。

---

## 4. 合并阻断矩阵

### 4.1 分支保护（目标态）

| 分支 | 必须通过的 workflow | 必须审查 | 禁止 force push |
|------|-------------------|---------|----------------|
| `main` | `kernel-verify`、`airy-verify`、`os-chk-verify`、`mgmt-verify`、`ssot-verify` | ✓ | ✓ |
| `release/*` | 同上 | ✓ | ✓ |
| `stable-*` / `lts-*` | 同上 | ✓ | ✓ |
| `docs-*` | `ssot-verify` | ✓ | ✓ |

`image-build` 与 `release` 不参与 PR 阻断：前者只由发布责任人 `workflow_dispatch` 触发，后者由 tag 触发。

### 4.2 回退通道

回归已经进入分支时，唯一合法的快速路径是**回退**而不是豁免：`git revert` 产生的新提交同样要通过全部门禁，但因为它恢复了已被验证过的状态，通常一次即绿。直接 push 绕过门禁在审计上等同于未评审提交。

### 4.3 提交侧处置

门禁红了的处置顺序固定为：本地复放 `ci/ci.sh <gate>` → 定位到具体断言 → 修根因 → 重推。禁止通过缩小受检范围、增加排除项、调整阈值来「让门禁变绿」；这类改动本身需要评审，且会触发对应的清单一致性门禁（`OS-BUILD-CI-MATRIX` 校验矩阵格数与排除项）。

---

## 5. 代码侧校验的落点

| workflow | job 数 | 内容 |
|----------|-------|------|
| `kernel-verify.yml` | 2 | `kernel-build`：4 config × 4 arch × 2 compiler − 4 排除格 = 28 格，`max-parallel: 8`，`W=2` 零告警门；`kernel-verify`：checkpatch `--strict`、sparse `C=2`、Coccinelle、KUnit（UML）、kselftest（QEMU） |
| `airy-verify.yml` | 2 | `airy-build`：`airy_defconfig` 四架构编译 + `kernel-airy --kind assert`（Airymax CONFIG、BTF 内嵌、六个 `built-in.a`、`security/airy/` 与 `kernel/superv/` 符号冲突）；`airy-static`：SPDX 标识审计 |
| `os-chk-verify.yml` | 1 | 解析 `gates.yaml` 派发全部注册 executors，聚合成报告并按 P0/P1 阻断；同时承载全量 Python 工具链的依赖安装与 pytest |

矩阵维度、排除项、格数与超时的唯一事实源是 [`tools/ci-matrix/manifest.yaml`](../../../agent-linux/tools/ci-matrix/manifest.yaml)，与 `kernel-verify.yml` 的偏差由 `OS-BUILD-CI-MATRIX` 拦截——「清单写了 28 格、流水线只跑 20 格」是明确的阻断项。

---

## 6. nightly.yml

### 6.1 nightly 结果不阻断 PR

nightly 的载荷是形式化验证、72h soak、chaos 注入，单轮以小时计。把它挂到 PR 路径上会让每个补丁的反馈时间失去上界，从而诱发「先合并再修」的实践，反而降低质量。因此 nightly **只**跑在 cron 与手动触发上，其结果作为回归哨兵与回退依据，不参与合并判定。

同理，`pull_request` / `pull_request_target` / `push` 三种触发在 nightly 中是**禁止项**，由 `OS-BUILD-NIGHTLY` 显式校验，防止后来者「顺手加上 PR 触发」。

### 6.2 触发条件

- `schedule`：唯一 cron `0 2 * * *`（每日 02:00 UTC）。
- `workflow_dispatch`：手动补跑。

### 6.3 job 拓扑与超时预算

| job | 超时 | needs | 说明 |
|-----|------|-------|------|
| `nightly-test-suite` | 360 分钟 | — | formal-verify / soak-runner / chaos-runner |
| `nightly-revert-or-budget` | 30 分钟 | `nightly-test-suite` | `always: true`，回归时自动回退，否则做预算检查 |

CI 总时长预算 60 分钟（OS-STD-TEST-011），由 `ci-budget-check --max-minutes 60` 在第二个 job 中强制；内核冷缓存回退场景的上限放宽到 90 分钟。

### 6.4 helper 契约

`tools/nightly/manifest.yaml` 登记了 6 个 helper，每个都有 `status` 与调用参数契约：

- `deferred`：脚本尚未落地，workflow 必须带存在性守卫并输出 SKIP 标记，不得静默执行。
- `available`：脚本已落地，守卫必须移除。

「未实现即静默执行」与「已实现仍被跳过」都会让 nightly 变成安慰剂，因此两者都由 `OS-BUILD-NIGHTLY` 判为漂移并阻断。

---

## 7. release.yml 与发布物料一致性

`release.yml` 只有两个 job：`build-and-sign`（SPDX SBOM、内核 RPM、GPG + cosign 签名、SDK tarball）与 `publish-release`（校验签名产物、推送 OCI 镜像到 ghcr、清理旧资产、创建 GitHub Release）。

流水线声明的产物与 [`tools/release-engineering/manifest.yaml`](../../../agent-linux/tools/release-engineering/manifest.yaml) 必须双向对齐：清单里的每条 active 物料都要在流水线里有真实的 evidence 声明，流水线声明的每个产物 glob 都要在清单里登记。单向对齐会分别产生「文档写了、流水线没产」与「流水线产了、清单没登记」两类漂移，都由 `OS-BUILD-RELEASE` 阻断。

---

## 8. 镜像流水线与实机验收

`image-build.yml` 是发布门而非 PR 门，两个 job 各有明确判据：

| job | 判据 |
|-----|------|
| `assemble` | 在固定版本 openEuler netinst 基础上于特权官方容器内组装桌面 ISO；装配清单与 ISO 校验和必须与登记值一致 |
| `accept` | QEMU 下分别以 BIOS 与 UEFI 安装并冷启动，直到出现 `AIRYMAX-DESKTOP-READY`（或 `acceptance=true` 时的 `AIRYMAX-SMOKE-PASS`），导出 TAP 与串口日志 |

产物是分块 OCI 制品（独立包 `ghcr.io/openairymax/airymaxos-iso`，192 MiB/块，账本 `iso-oci.txt`），与基础镜像 `ghcr.io/openairymax/airymaxos` 分离发布——ISO 体积会独立增长，混进同一个包会让基础镜像的拉取成本被 ISO 绑架。

实机验收（U 盘写入、真实硬件启动、桌面登录）不在 CI 能力范围内，由 [190-distribution/03-installer-design.md](../190-distribution/03-installer-design.md) 规定的字段与 `airy-flash` 的回读校验承担，CI 只负责让该 ISO 具备可验收的前提。

---

## 9. 资源与预算

### 9.1 runner

托管 `ubuntu-latest` 为主，`kernel-build` 因 28 格矩阵需要 `max-parallel: 8` 的自限。`image-build` 需要特权容器与 QEMU，单 job 硬上限 360 分钟。

### 9.2 并发限制

除 nightly 外，各 workflow 按分支分组并发，新推送取消进行中的同组运行。nightly 例外：`concurrency.group: nightly` 且 `cancel_in_progress: false`——夜里被打断的 soak 等于没跑，重跑要再等一天。

### 9.3 内联预算棘轮

[`ci/workflow-budgets.txt`](../../../agent-linux/ci/workflow-budgets.txt) 为每个 workflow 记录「允许的内联 `run` 块数」与「允许的 job 数」，只许减不许加。当前 8 个文件中 6 个已归零，`nightly.yml`（6 段）与 `release.yml`（10 段）是迁移中的欠账。内联预算为 0 的文件还会被检查禁用模式（`docker load`、`sha256sum -c`、`rm -rf /usr/share/dotnet`、`apt-get install qemu`、TAP 解析），防止逻辑被抄回 YAML。

### 9.4 超时约束

CI 总时长 ≤60 分钟（OS-STD-TEST-011）。矩阵类 workflow 的单格超时与总预算必须能对上：28 格 × 单格耗时 ÷ `max-parallel` 不得突破总预算，否则 `OS-BUILD-CI-MATRIX` 拒绝该配置。

---

## 10. 失败处理

### 10.1 分类

| 类别 | 判据 | 处置 |
|------|------|------|
| 环境失败 | checkout / 依赖安装 / runner 镜像异常 | 重试一次；连续两次同类失败升级为基础设施问题 |
| 门禁失败 | `ci/ci.sh <gate>` 非零退出 | 不改配置，修根因（§4.3） |
| 超时 | 命中 §9.4 的预算 | 先判断是否矩阵格数或缓存失效导致，再决定拆分而非放宽 |
| flaky | 同一提交重跑结果不一致 | 立即登记并定位，禁止用重试掩盖（§10.2） |

### 10.2 flaky 管理

测试一旦被观察到在同一提交上不稳定，就有两条路：修掉，或删除。不允许第三条路——无限重试。重试会把不确定性从 CI 转移到用户，代价高一个数量级。

### 10.3 通知

阻断类失败通过 PR 状态回写；`main` 变红额外在 issue 上留痕。nightly 失败按 §6.3 的第二个 job 自动回退，不依赖人工值守。

---

## 11. 可观测与报表

### 11.1 门禁与镜像的观测标记

判定结果以固定标记落入日志，便于 grep 与机器汇总：

| 标记 | 来源 | 含义 |
|------|------|------|
| `AIRY-FLASH-PASS` / `AIRY-FLASH-FAIL` | `airy-flash` 写入与回读 | 介质写入是否逐字节验证通过 |
| `AIRYMAX-SMOKE-PASS` / `AIRYMAX-SMOKE-FAIL` | `run-smoke.sh` | 安装后冒烟检查 |
| `AIRYMAX-DESKTOP-READY` | `desktop_ready.sh` | 桌面会话可达 |
| `AIRYMAX-RT-BUDGET` | `rt_budget.sh` | RT 运行时预算的分配与落地情况 |
| `AIRYMAX-OBSERVE` | `airy_observe.sh` | 运行期观测点快照 |
| `AIRYMAX-RECORD` | `airy_record.sh` | 录屏产物路径、帧数与校验和 |

标记格式统一为 `MARK key=value ...`，值含空格时用双引号包裹，保证单行可解析。

### 11.2 周报

周报只汇总三件事：门禁红过的次数与原因分布、迁移欠账（§9.3）的净减少量、flaky 登记与清零数。其余指标不进门禁决策。

---

## 12. 与 Airymax Unify Design 的关系

IRON-9 v3 四层模型对 CI 的唯一硬要求是 `[SC]` 共享契约层不被单端改动。0.0.3 的落地方式是把「唯一性」与「可编译性」拆成两道门禁，而不是再造一条逐字节 diff 流水线：

- 唯一性：`ci/ci.sh mgmt-files`（OS-IRON-008）确认 12 个 `[SC]` 头在源码树内只有一份物理副本。
- 可编译性：`OS-CHK-DOC-04` 用 `sc_tri_compile.py --check` 在五路（kernel-space / linux-user-space / third-party / dsl-kernel-space / dsl-linux-user-space）下编译同一份头文件。

跨仓（用户态副本 `agentrt/commons/include/airymax/`）的一致性在 0.0.3 **尚无门禁**，登记为 0.0.4 的 `SC-01`。这是已知缺口，不是已解决问题。

---

## 13. 相关文档

- [01-patch-lifecycle.md](01-patch-lifecycle.md)：补丁生命周期（本文的上游）
- [03-pull-requests.md](03-pull-requests.md)：PR 评审规则
- [09-release-process.md](09-release-process.md)：发布流程
- [70-build-system/03-ci-cd-pipeline.md](../70-build-system/03-ci-cd-pipeline.md)：构建、矩阵、静态分析、签名、缓存的技术细节
- [190-distribution/03-installer-design.md](../190-distribution/03-installer-design.md)：烧录与安装的判据
- [`.github/workflows/README.md`](../../../agent-linux/.github/workflows/README.md)：workflow 清单（SSoT）

---

## 14. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0 | 2026-06-20 | 初始版本：6 个 workflow 逐一定义 job 结构 |
| v1.0.1 | 2026-07-21 | 补充 nightly / release 的 job 细节 |
| v1.1.0 | 2026-10-29 | 重写为流程层文档：workflow 清单与 job 结构交由 `.github/workflows/README.md` 与 70-build-system/03 承载，本文只保留阻断关系、分支保护、失败处置与预算棘轮；删除已退役的 `sc-verify.yml` 契约工作流章节，`[SC]` 校验改为指向 `mgmt-files`（OS-IRON-008）与 `OS-CHK-DOC-04` |
