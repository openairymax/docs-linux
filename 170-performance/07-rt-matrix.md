Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agent-linux（AirymaxOS）实时能力矩阵
> **文档定位**：agent-linux（AirymaxOS，极境智能体操作系统）实时能力探测与适用性判定设计\
> **文档版本**：v1.1.0\
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
**不在范围**：修改内核启动参数（由 X-05 共存策略只读使用，不写 `cmdline`）、
实时性压测（由 X-04 负责）。

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

## 5. 桌面与实时共存

实时带宽只约束 SCHED_FIFO/SCHED_RR 的总量，不约束 agent 侧的 CFS 负载。合成器、
输入线程与 agent 进程同为 SCHED_OTHER，按 slice 权重分 CPU；重负载下桌面失响应
不是因为实时预算超标，而是普通负载抢走了 CPU。共存动作因此只有一个：在基线与
保护值之间切换桌面 slice 的 `CPUWeight`。

| 项 | 取值 | 说明 |
|----|------|------|
| 触发信号 | `/proc/pressure/cpu` 的 `some avg10` | 有任务可运行却在等 CPU 的时间占比 |
| 回退信号 | `/proc/loadavg` 除核数 | 没有编入 PSI 的内核（如 285H 的 6.6.157-alk） |
| 阈值 | 20%（PSI） | 达到即进入保护档 |
| 回退阈值 | 100%（负载） | 整机平均可运行任务数达到核数才进入保护档 |
| 基线权重 | 50 | 与 `airymax-desktop.slice` 的静态值一致 |
| 保护权重 | 200 | 运行时抬升，桌面在被抢占时获得更大份额 |
| 落地方式 | `systemctl set-property --runtime` | 只在本启动内生效，重启回落静态基线 |

首选 PSI：它直接度量「有任务可运行却在等 CPU」，正是桌面失响应的直接指标。
但 285H 的 6.6.157-alk 没有编入 PSI，策略在唯一可验收的机器上恒为基线，等于
没装。因此增加负载回退：`loadavg` 除以核数，单位同样是「占满整机的比例」。
两者语义不同——`loadavg` 把不可中断睡眠算进来，等 IO 时会虚高——所以不共用
阈值：PSI 用 20% 的时间占比，负载回退要求平均可运行任务数达到核数（100%），
用更严的门槛吸收 IO 等待的虚高。两条都读不到时保持基线，报告里写明「无 CPU
争用读数」，不把「没有读数」当成「没有争用」。

`--json` 因此增加 `origin`（`psi`/`loadavg`/`none`）与 `cores` 两个字段，
现场据此判断这次判定用的是哪条信号。

写完后回读 `CPUWeight`，只有回读值等于目标值才算落地，因此「命令返回 0」与
「属性真的改了」在报告里是两件事。

矩阵等级是准入而不是动作：`none` 时实时负载本来就不会被调度，桌面无须让路；
`governor`/`isolated`/`threadirqs` 作为建议输出，用于解释「实时延迟为什么降不
下来」。调频归 `airy-power` 独占，本策略只读不写，避免两处同时写
`scaling_governor`。

```text
airy-rt-cap coexist [--json] [--apply] [--desktop UNIT]
                    [--systemctl FILE] [--pressure FILE] [--loadavg FILE]
```

| 退出码 | 含义 |
|-------|------|
| 0 | 已一致，或桌面 slice 未加载（`absent`） |
| 3 | 漂移：只读评估与目标权重不一致，或 `--apply` 的写入未被回读确认 |
| 1 | `systemctl` 查询失败，未做任何调整 |

`--json` 与 `Plan.dump()` 同形，字段为 `mode`、`unit`、`weight`、`current`、
`pressure`、`origin`、`cores`、`grade`、`present`、`admit`、`applied`、`notes`、
`detail`。

落地单元为 `airy-rt-coexist.service` 与 `airy-rt-coexist.timer`：开机 90 秒后
首次评估，之后每 60 秒重评。桌面 slice 只在登录后存在，未登录时单元报 `absent`
并以 0 退出，不产生 failed 与 incident。两个单元默认关闭，桌面档位由 kickstart
显式启用；服务器档位不启用时不产生任何运行时写入，桌面权重恒为静态基线。

### 5.1 推理负载的内存边界

CPU 权重只在争用时重新分配时间片，管不住内存。285H 现场三个推理实例 RSS 合计
29.5 G / 30 G 物理内存，swap 已用 11 G，`MemFree` 只剩 250~380 MB：此时任何
一次分配都要先唤醒 kswapd 或走 direct reclaim，实时延迟出现毫秒级尖峰，桌面与
SSH 会话一起失去响应。根因不是某个进程申请得太多，而是推理负载没有任何 cgroup
边界，峰值得以吃掉留给内核、桌面与页缓存的那部分内存。

边界声明在 `airymax-agent.slice` 上，`airy-llm.service` 只声明 `Slice=` 归属：

| 属性 | 取值 | 作用 |
|------|------|------|
| `MemoryMax` | 90% | 硬上限，为内核与交互路径留出一成；越界由 OOM killer 在本组内选 victim |
| `CPUWeight` | 100 | 与桌面基线 50 保持 2:1，跑满核不被压 |
| `IOWeight` | 100 | 读权重时不被桌面压缩到最低档 |

用比例而不是绝对字节：模型尺寸随机型变化，钉死字节数会在换机型时把本来能跑的
模型 OOM 掉；「内核与桌面按机器规模预留一成」则与机型无关。上限只在 slice 上
声明一处，推理单元不重复声明，否则「哪一层在生效」无从判断。

**只声明 hybrid 层级上真正会落地的属性。** 285H 的 `/sys/fs/cgroup` 是 tmpfs，
memory 控制器走 cgroup v1（`systemd.unified_cgroup_hierarchy` 未置位、无
`cgroup2` 挂载）。实测结论：

- `MemoryMax` 写入 `memory.limit_in_bytes`：`systemctl set-property --runtime`
  后读到 29324836864（90% × 32.6 GB），有效；
- `MemoryHigh` 在 v1 上**不写** `memory.soft_limit_in_bytes`，值停在默认；
  该属性是 cgroup v2 专有，写了只会制造「有软限」的假象；
- `MemorySwapMax` 依赖 memsw 记账，285H 的 cmdline 没有 `swapaccount=1`，
  `memory.memsw.limit_in_bytes` 停在上限值。

因此 0.0.4 只声明 `MemoryMax`。要拿到「组内软限 + 换页上限」需要镜像改成
unified cgroup v2 并在 cmdline 打开 `swapaccount=1`；这两项属于镜像层决策，
不由策略模块在运行时假装生效。

slice 的 cgroup 路径是嵌套的：`airymax-agent.slice` 的父 slice 是
`airymax.slice`，现场读取必须走
`/sys/fs/cgroup/memory/airymax.slice/airymax-agent.slice/`；只看
`/sys/fs/cgroup/memory/airymax-agent.slice` 会误判为「限制没落地」。

内存边界与档位、空闲深度是三个独立控制点：档位决定频率与功耗墙，空闲深度决定
唤醒成本，内存边界决定超卖时谁先付出代价。

---

## 6. 与其他任务的关系

- **X-01 实时带宽**：`matrix` 的 `budget` 与 `group` 直接复用带宽包的布局识别，
  避免同一份 cgroup 判定出现两套口径。
- **X-04 实时性验收**：验收前先跑 `matrix`，按 `ready`/`soft` 选择对应基线；
  等级下降即判定环境回归，不再继续压测。
- **X-05 桌面与 RT 共存**：矩阵中的 `governor`、`isolated`、`threadirqs`
  是共存策略的输入；策略只读矩阵结果，不重复探测。

---

## 7. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.3.0 | 2026-11-03 | §5.1 增加推理负载的内存边界：`airymax-agent.slice` 的 `MemoryMax` 与 cgroup v1 实测边界 |
| v1.2.0 | 2026-11-03 | §5 增加无 PSI 内核的负载回退：独立阈值、`origin`/`cores` 字段、重写「无读数」口径 |
| v1.1.0 | 2026-11-03 | 增加 §5 桌面与实时共存：PSI 阈值、权重切换、`coexist` 接口与落地单元 |
| v1.0.1 | 2026-11-03 | 初版：八项能力、三态判据、三级等级与命令接口 |
