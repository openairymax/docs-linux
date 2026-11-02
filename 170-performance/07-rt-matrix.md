Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agent-linux（AirymaxOS）实时能力矩阵
> **文档定位**：agent-linux（AirymaxOS，极境智能体操作系统）实时能力探测与适用性判定设计\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-11-03\
> **上级文档**：[agent-linux 设计文档](README.md)\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## 1. 设计目标与范围

实时能力不是一个布尔值。同一台机器上，PREEMPT_RT 是否启用、实时带宽是否被
置零、cgroup 是否记账 RT、调度器是否支持 SCHED_DEADLINE、中断是否线程化、
CPU 是否隔离、调频是否被钉在 performance，这些条件各自独立，缺一条对应一种
不同的现场症状：可能是 `sched_setscheduler` 返回 `EPERM`，可能是延迟尖峰，
也可能是频率被省电策略拉走。

若把上述条件压成一个「实时可用/不可用」，现场只能得到一句无信息量的结论。
因此本设计把每项能力单列，并给出整机等级，供安装器、验收套件与运维共同判读。

**适用范围**：装机自检、升级后能力回归、实时工作负载准入判定、现场故障定位。
**不在范围**：修改任何内核参数（由 X-05 共存策略负责）、实时性压测（由 X-04 负责）。

---

## 2. 探测项与判据

探测全部只读文件系统与 `/proc/cmdline`，不写任何接口，也不依赖 debugfs——
现场常常没有挂载 debugfs，依赖它会让探测在救援环境里失效。

| 项 | 探测来源 | yes 判据 | 缺失时的含义 |
|----|---------|---------|-------------|
| `realtime` | `/sys/kernel/realtime` | 值为 `1` | 内核非 PREEMPT_RT |
| `budget` | `/proc/sys/kernel/sched_rt_runtime_us` | 可读且非 `0` | 接口不可读，本次无法判定 |
| `group` | cgroup 挂载根 | 找到 v1 记账点或 v2 暴露 `cpu.rt.max` | 未找到记账树，RT 组调度未启用 |
| `deadline` | `/proc/sys/kernel/sched_deadline_period_max_us` | 文件存在 | 未编入 SCHED_DEADLINE |
| `threadirqs` | `/proc/cmdline` | 含 `threadirqs` | 中断未线程化 |
| `isolated` | `/proc/cmdline` | 含 `isolcpus=` 或 `nohz_full=` | 未做 CPU 隔离 |
| `governor` | `cpu0/cpufreq/scaling_governor` | 值为 `performance` | 无 cpufreq，无法判定 |
| `clocksource` | `clocksource0/current_clocksource` | 值为 `tsc` | 无 clocksource，无法判定 |

### 2.1 三态原则

每项结论只有 `yes` / `no` / `unknown` 三种：

- `no`：读到了，但条件不满足（例如 `runtime=0`）。
- `unknown`：这次读不到（无权限、节点不存在、文件内容非法）。

**unknown 不等于 no**。把读不到说成不支持，会让运维去换内核或重编，而实际
问题在挂载与权限上（例如 tracefs 需要 root 时把「没权限看」报成「内核没实现」）。
因此只有 `no` 计入 `pending`，`unknown` 在报告中单列。

---

## 3. 等级判定

| 等级 | 条件 | 含义 |
|------|------|------|
| `ready` | `realtime`、`budget`、`group` 全为 yes | 完整实时：PREEMPT_RT + 带宽 + 组记账 |
| `soft` | `budget` 为 yes，其余不全 | 软实时：实时类可用，但内核非 PREEMPT_RT 或缺组记账 |
| `none` | `budget` 非 yes | 不具备实时能力 |

`ready` 是硬实时工作负载的准入条件；`soft` 允许音频、合成器等软实时场景，
但验收套件必须按软实时基线判定（见 06-benchmark-suite.md）。`none` 时
不应部署依赖 SCHED_FIFO/SCHED_DEADLINE 的服务。

---

## 4. 命令接口

`airy-rt-cap matrix` 输出能力矩阵，与 `status`/`apply`/`config` 共用同一入口。

```text
airy-rt-cap matrix [--json] [--sysroot DIR] [--procroot DIR]
                   [--cgroup DIR] [--cmdline FILE]
```

| 退出码 | 含义 |
|-------|------|
| 0 | `ready` |
| 3 | `soft`（降级可用） |
| 1 | `none` |
| 2 | 用法错误 |

`--json` 与 `Matrix.dump()` 同形，字段为 `grade`、`pending`、`caps`；门禁与
装机脚本据此直接判读，不解析人类可读输出。`--sysroot`/`--procroot`/`--cmdline`
只服务于测试与救援环境，产品路径使用缺省值。

---

## 5. 与其他任务的关系

- **X-01 实时带宽**：`matrix` 的 `budget` 与 `group` 直接复用带宽包的布局识别，
  避免同一份 cgroup 判定出现两套口径。
- **X-04 实时性验收**：验收前先跑 `matrix`，按 `ready`/`soft` 选择对应基线；
  等级下降即判定环境回归，不再继续压测。
- **X-05 桌面与 RT 共存**：矩阵中的 `governor`、`isolated`、`threadirqs`
  是共存策略的输入；策略只读矩阵结果，不重复探测。

---

## 6. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0.1 | 2026-11-03 | 初版：八项能力、三态判据、三级等级与命令接口 |
