# AirymaxOS 整体方案评审与 LOC 估算（终版）

> 日期：2026-10-03 · 方法：docs-linux 全 185 篇逐文档技术点提取（166 行矩阵）+ airymaxhub 全仓实测校准
> 口径：预估新增 = 全功能设计净新增；上游 Linux 6.6（2,348 万行）、UKUI 上游、agentrt 可复用（~33 万行）均不计入自研

---

## 1. 四问结论

| 问题 | 结论 |
|---|---|
| 逻辑闭环？ | **是**。核心链路（4 syscall → Capability Folding → io_uring fastpath → LSM 执法 → 双层 Supervisor 冻结/裁决 → ring buffer 日志）每环有详设 + 代码锚点 |
| 研究、证伪过？ | **是**。`_review_v3.6` 实存 13 份、68 项改进/12 项闭环；M0✅/M1✅/M2⚠️条件 GO；10 类已知缺口在 09-known-caveats.md 自我声明 |
| 可进入正式开发？ | **内核侧可以**（A/B/C/D 模块 ~9 万行全部有详设，Phase 0 已完成，corekern/UAPI/syscall 表已落地）；**全面开发前须解决第 5 节阻碍** |
| GUI（UKUI）？ | **设计齐备、可开发**。决策 2026-09-15 已入 distro 仓（六条铁律即设计规格），集成层部分落地；仅冻结点 TODO 表 6 项待核对 |

## 2. UKUI 在设计方案的哪里（定位答案）

| # | 位置 | 内容 |
|---|---|---|
| 1 | `agent-linux/distro/desktop/README.md`（主设计文档） | 决策 2026-09-15「OS 侧系统桌面 = UKUI 4.x（openKylin/openEuler 线）」；六条集成铁律：①RPM化不vendor ②agent入口唯一=Gateway（kylin-ai* 排除屏蔽）③调度隔离 airymax-desktop.slice ④X11起步Wayland二期 ⑤GPLv3合规 ⑥滚动与内核解耦；冻结点 TODO 表 6 项 |
| 2 | `agent-linux/distro/ukui-patches/README.md` | 零补丁策略 + 允许补丁三条件 + 硬规则 |
| 3 | `agent-linux/distro/desktop/70-airymax-ukui-policy.sh`（59 行，已实现） | 屏蔽 kylin-ai* 4 服务 + fcitx5 环境 + graphical.target |
| 4 | `agent-linux/distro/desktop/airymax-desktop.slice`（26 行，已实现） | CPUWeight=50/IOWeight=50 低干扰 slice |
| 5 | `agent-linux/distro/`（README L23-35 / kickstart 83 行 / comps 99 行） | UKUI 作为默认桌面组（airymax-desktop* → UKUI set） |
| 6 | `REPOSITORIES.md` L57/L255 | 2026-09-15 变更历史：新增 distro 叶仓，"决策：OS 侧系统桌面采用 UKUI 4.x，以 RPM 消费、与内核仓单向依赖" |
| 7 | `agent-workload/products/desktop/`（Tauri v2+React 客户端，实测 42,799 行） | Airymax Desktop Client——GUI 应用实物 |

> 注：docs-linux 文档树内无 UKUI 内容属正常分工（docs-linux 为内核态工程文档树，桌面集成归 distro 叶仓）。
> `190-distribution/03-installer-design.md:103` 的 `@desktop（GNOME）` 为旧口径，建议随下次修订对齐 UKUI。

## 3. 分模块 LOC 预测

| 模块 | 已存在 | 需新增 | 合计 | 置信度 |
|---|---|---|---|---|
| A 内核核心与 IPC fastpath（corekern/syscall 548-551/Capability Folding/io_uring/[SC] UAPI/sched_tac/trace events） | ~3,800 | 40,734 | 44,534 | 中高 |
| B 安全（airy LSM/capability CNode/沙箱/机密计算/国密） | ~2,706 | 19,500 | 22,206 | 中 |
| C 双层 Supervisor + A-UCS 统一配置 | 533 | 18,500 | 19,033 | 高 |
| D 日志与可观测（A-ULP/logger_d/审计哈希链/panic 生存） | 534 | 12,760 | 13,294 | 高 |
| E 记忆子系统（MemoryRovol L1-L4/mem_d/CXL/PMEM/MGLRU） | 238 | 17,880 | 18,118 | 中 |
| F 认知子系统（CLT kthread/Thinkdual/Wasm 3.0/GPU-NPU/cogn_d） | 377 | 24,670 | 25,047 | 中 |
| G 云原生（K8s CRD/shim/CNI/CSI/supernode/gateway_d/agentctl） | 186 | 29,350 | 29,536 | 中 |
| H 系统用户态与运维（config_d/monitoring/DevStation/告警/部署/备份） | 295 | 51,650 | 51,945 | 低-中 |
| I 驱动模型（agent_bus/devm_airy_* 5 资源族/三类 Agent 驱动/VFIO/IOMMU） | 0 | 15,400 | 15,400 | 高 |
| J 测试与形式化验证（KUnit/kselftest/契约/fuzz/TLA+Coq/CBMC） | 301 | 19,200 | 19,501 | 中 |
| K 服务用户态化（用户态 VFS/网络栈/dev_d/12 daemons 编排） | 0 | 27,000 | 27,000 | 低 |
| L 构建/CI/codegen（OS-CHK 21 检查/6 workflow/缺 3 生成器/验收 runner） | 0 | 29,180 | 29,180 | 中 |
| M 发行版（RPM 51 包/dnf/installer/livepatch/rpm-ostree/可重现/SBOM） | 473 | 25,300 | 25,773 | 中 |
| N 兼容性（ABI 四层/POSIX/上游跟踪/IPC 版本化/DKMS） | 0 | 9,400 | 9,400 | 中 |
| O 性能基准（8 基准+registry/SLO/efficiency） | 0 | 7,400 | 7,400 | 中 |
| P i18n（locale 四层/UTF-8 原语/错误码 gettext/CJK 宽度库/双语 CI） | 0 | 13,400 | 13,400 | 中 |
| Q 应用开发支撑（8 态状态机/SDK 适配/编排/token 令牌桶/部署） | 0 | 7,700 | 7,700 | 中高 |
| **OS 侧小计（A-Q）** | **9,443** | **369,024** | **378,467** | — |
| R1 UKUI 桌面集成层（distro/desktop + kickstart/comps 桌面组完整化） | （含在 M 的 distro 473） | 1,900 | — | 中 |
| R2 products/desktop GUI 客户端（Tauri v2 + React 18 全功能补全） | **42,799** | 12,800 | 55,599 | 中 |
| **GUI 小计（R）** | **42,799** | **14,700** | **57,499** | — |
| **总计** | **52,242** | **383,724** | **435,966** | — |

**单一总数：需新增 ≈ 383,724 行（38.4 万行）；全功能完成后自研总规模 ≈ 435,966 行（43.6 万行）**

## 4. 口径说明

1. **净新增（主口径）**：**≈ 38.4 万行** = OS 侧 369,024（docs-linux 全功能 A-Q）+ GUI 14,700（UKUI 集成层 1,900 + desktop 补全 12,800）
2. **全功能自研总规模**：**≈ 43.6 万行** = 净新增 38.4 万 + 已存在 5.2 万（OS 侧实测 9,443 + desktop 实测 42,799）
3. **完整交付面**：≈ 76 万行（43.6 万 + agentrt 可复用 ~33 万：daemons/cupolas/heapstore/gateway/protocols/sdk/tools 等）；上游 Linux 6.6（2,348 万行）与 UKUI 上游作为 RPM/基线消费，不计

**已存在实测构成**：kernel 仓 54 个 airy 文件 5,375 行 + [SC] UAPI 12 头 1,459 行 + include/airymax 78 行 + syscall 表（548-571 全 24 槽位已注册）≈ 6,972；7 叶仓骨架 1,985（memory 238/security 242/cognition 377/services 346/system 295/cloudnative 186/tests-linux 301）；distro 473；desktop 42,799。

## 5. 阻碍"全面开发"的问题（按优先级）

1. **资源-体量错配**：130-roadmap/03-resource-estimation.md 实施编码仅 580h（kernel 180h），vs 需新增 38.4 万行（差 ~60 倍），且未含 GUI/distro 工时——须重做
2. **LOC 条款自相矛盾**：00-alk-kernel-overview.md 改造点①-⑨合计 ~30,514 行 > 同文 L189 自述"≤2 万行核心"；[SC] 头文件数 12 vs 10 不一致（08-ci-cd-pipeline.md L285 vs L308）
3. **执行停滞**：4 核心 syscall（548-551）全 -ENOSYS 占位；trace events 零落地；codegen 4 生成器缺 3；WS-8 通电计划停在阶段 3
4. **设计空洞 ≈ 1.4 万行**：机密计算（SGX/TDX/SEV-SNP/VirtCCA）与国密（SM2/3/4/9）仅组件清单无详设；Wasm/llm-scheduler 概要级
5. **低置信度 ≈ 9 万行**：DevStation 19.5K、服务用户态化 27K、IOMMU/NUMA 仅提及级——直接开工风险高
6. **冻结点 TODO 表 6 项**（distro/desktop）：分级冻结包名/输入法/终端/登录管理器/kylin-ai* 显式清单/SELinux 叠加待首个冻结点核对

## 6. 与设计自设预算交叉验证

| 设计条款 | 实测 | 判定 |
|---|---|---|
| 内核核心 5-10 万行（03-microkernel-strategy.md:59） | A+I+J 内核态 ≈ 5.4 万行新增 | ✓ 落在区间 |
| 微内核化补丁 ≤2 万行（00-alk:110/189） | 改造点①-⑨ 3.05 万行 | ✗ 自相矛盾，须修订 |
| 单驱动 ≤2000 行（OS-DRV-082） | 最大 mem 驱动 1,500 | ✓ |
| 零补丁 RPM 消费 UKUI（distro/ukui-patches） | UKUI 本体 0 行，集成层 1,900 | ✓ |
| 580h 编码工时（03-resource-estimation.md:127） | 38.4 万行净新增 | ✗ 须重做 |

## 7. 方法说明

- **需新增**：8 组并行子代理逐文档提取 docs-linux 全 185 篇的 ~240 个技术点，聚合为 166 行矩阵（技术点|文档:章节|层次|设计深度|现有|新增 LOC|置信度），按 A-Q 模块汇总
- **已存在**：airymaxhub 全仓实测（wc -l 统计 kernel 仓 airy 文件、7 叶仓、distro、products/desktop）
- **校准教训**：前轮"docs-linux 无 UKUI"结论源于搜索范围过窄（仅搜文档树未搜 distro 叶仓），已修正