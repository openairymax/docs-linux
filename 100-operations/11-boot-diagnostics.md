Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 开机打印与卡点归因
> **文档定位**：agent-linux（AirymaxOS）发行版运维——九阶段开机契约、卡点自证报告与会话内健康探针的工程设计\
> **文档版本**：v1.0.0\
> **最后更新**： 2026-11-25\
> **上级文档**：[100-operations README](README.md)\
> **同源映射**：[190-distribution/07-session-entry.md](../190-distribution/07-session-entry.md) 有界等待、[20-modules/07-system.md](../20-modules/07-system.md) 系统单元、[190-distribution/03-installer-design.md](../190-distribution/03-installer-design.md) 安装期标记\
> **核心约束**：打印只报事实、不做判定；观测者自身必须有终点，且不得成为新的卡点\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0\
> **SSoT 依赖声明**：本文档所引用的全部规则编号权威定义见 `50-engineering-standards/09-ssot-registry.md`，本文档不私自定义新规则编号。\
> **IRON-9 v3 层次**：[IND] 完全独立层（开机打印与看门狗为发行版专属）

---

## 目录

- [第 1 章 定位与边界](#第-1-章-定位与边界)
- [第 2 章 九阶段契约](#第-2-章-九阶段契约)
- [第 3 章 卡点自证](#第-3-章-卡点自证)
- [第 4 章 会话内健康探针](#第-4-章-会话内健康探针)
- [第 5 章 现场判读](#第-5-章-现场判读)
- [第 6 章 验收判据](#第-6-章-验收判据)
- [第 7 章 相关文档](#第-7-章-相关文档)

---

## 第 1 章 定位与边界

开机失败时现场只有一句「没起来」。要让它变成可归因的事实，必须能回答三个问题：

| 问题 | 载体 | 回答不了时的形态 |
|------|------|------------------|
| 走到哪一步 | 九阶段行（序号 / 总数 / 代号） | 屏幕停在最后一条 `Starting` |
| 这一步等多久算卡 | 心跳行 `used=` / `limit=` | 不知道是在跑还是已经死 |
| 卡在谁身上 | 卡点报告（作业表 + 单元事实） | 只知道目标没到，不知道哪条作业没回 |

三个问题分别由打印契约（`ui.sh` + `stage_run.sh`）、看门狗（`boot_watch.sh`）
和会话内探针回答。它们只产出事实，不参与启动排序、不判负、不写任何状态文件：
判定权归 systemd 的生命周期与交付判据，观测者越权会让「失败单元为 0」这条
交付判据失去意义。

边界：

- 不替代 systemd：不杀进程、不改单元、不提前收工。慢与死的区分由心跳和
  作业表承担，超时收尾仍由单元的 `TimeoutStartSec`。
- 不覆盖内核早期：initramfs 与根切换之前没有本项目的单元，只能由
  `seen=0` 说明「还没走到第一阶段」。
- 不新增第二处措辞：阶段名、序号、列宽只登记在阶段表一处，调用方只报阶段名。

---

## 第 2 章 九阶段契约

### 2.1 阶段表

阶段表的顺序即执行顺序；序号与总数一律从表里取，不由各脚本自报——自报的
总数一旦与表不符，「7/8」与「7/9」会同时出现在同一台机器的日志里。

| 序号 | 阶段名（journal） | 代号（console / 串口） | 生产单元 |
|------|-------------------|------------------------|----------|
| 1 | 基础挂载 | `mount` | `airy-boot-mount.service` |
| 2 | 首启判定 | `firstboot` | `agentrt-firstboot.service` |
| 3 | 数据分区 | `data` | `airy-disk-mount.service` |
| 4 | 运行时带宽 | `bandwidth` | `airy-rt-bandwidth.service` |
| 5 | 换页上限 | `swaplimit` | `airy-rt-memcap.service` |
| 6 | 电源与热治理 | `power` | `airy-power.service` |
| 7 | 桌面 RT 共存 | `desktoprt` | `airy-rt-coexist.service` |
| 8 | 桌面就绪 | `desktop` | `airymax-desktop-ready.service` |
| 9 | 自检与取证 | `health` | `agentrt-health-check.service` |

阶段名不含空白：单元以 `ExecStart` 传名，systemd 按空白切分参数，带空格的
名字会被拆成多个参数，运行期查不到序号，现场只剩一行 `FAILED`。阶段名的
显示宽度按 CJK 记 2 列，与代号列一起由 `units_lint` 校验。

### 2.2 一行怎么读

```text
[ 3/9 ] data            ...   up=8s
[ 3/9 ] data            ok    ms=42 mounted=/data dev=/dev/nvme0n1p4
[ 5/9 ] swaplimit       wait  used=30s limit=60s up=38s
```

| 记号 | 含义 | 出现位置 |
|------|------|----------|
| `...` | 阶段已开始，命令未返回 | 起始行 |
| `ok` | 命令以 0 退出，关键值随行 | 结论行 |
| `note` | 以登记过的宽限码退出（数据分区未启用、实时上限部分落档、机器仍热） | 结论行 |
| `FAIL` | 其余非零退出，附 `rc=` | 结论行 |
| `wait` | 已跑过心跳周期仍未返回 | 心跳行 |
| `up=` | 开机秒数，取自 `/proc/uptime` | 起始行、心跳行 |
| `used=` / `limit=` | 心跳已用秒数 / 预期上限，限制来自 `TimeoutStartSec` | 心跳行 |
| `ms=` | 阶段实测耗时（毫秒） | 结论行 |

`ms=` 取不到时不写，也不写 0：「没有耗时这个事实」与「耗时是 0 毫秒」在
验收现场是两件事。关键值只取工具给出的事实，缺项就不出现；键的顺序由调用
参数决定，工具内部字段顺序变化不会让阶段行的列漂移。

### 2.3 双通道

console 侧只出 ASCII 代号，journal 侧出中文阶段名：

| 通道 | 内容 | 理由 |
|------|------|------|
| `/dev/console` | `[ k/9 ] 代号 token 关键值` | 本地屏是 Linux VT，镜像字体无 CJK 字形；中文写进去只剩方块 |
| 串口 | 同 console 行 | 现场常只有串口可读 |
| journal | `[ k/9 ] 中文名 token 关键值` | 系统起来之后由运维读取，终端编码正确 |

两行的序号、总数、token 与关键值逐字相同，判读脚本只读这些，不看人类措辞。
通道归属由 `con.sh` 统一判定：命令行末位是 `tty0` 时 `/dev/console` 不指向
串口，只写 `/dev/console` 会让串口现场一条阶段行都读不到；串口优先、
`/dev/console` 次之，两者同源时只写一份。

### 2.4 一次开机一条

阶段结论每开机只出一次，避免 timer 重跑的单元把屏幕与 journal 刷满：

- 闸门：`ui_gate` 首次返回 0，之后返回 1；标记落固定路径 `/run/airymax-ui`，
  重启即清空。目录不可写时放行——宁可重复一行，也不能把结论吞掉。
- 沙箱：`/run` 在 `ProtectSystem=strict` 下只读，凡跑 `stage_run.sh` 且启用该
  选项的单元必须声明 `RuntimeDirectory=airymax-ui`，由 systemd 保证可写；
  这一条由 `units_lint` 断言，不靠记忆。
- 快照：工具每轮输出的明细以「分档 + 关键值」组成指纹，指纹相同且未过
  `AIRY_STAGE_REPEAT`（默认 1800 秒）就不转发，指纹变化立即转发，指纹不变
  但到期也转发一次当心跳。温度、节流计数、负载等逐轮都变的明细不进指纹。

### 2.5 契约错误

阶段名不在表内、序号与总数不符，属实现错误，不是运行时故障：`ui_gate` 以 5
拒绝，`stage_run.sh` 用法错误以 64 退出，打印契约库缺失以 6 退出。拒绝时
console 侧落一行纯 ASCII 事实 `airymax-boot: <fact>`——触发契约错误的往往
正是中文阶段名，原样打出去在 VT 上仍是乱码，等于没有诊断。

---

## 第 3 章 卡点自证

九阶段行只覆盖本项目自己的单元，而它们都排在某个 target 前后。上游作业
（登录管理器等 D-Bus 名、文件系统检查、plymouth 等待单元）不返回时，systemd
不再写新行，目标达不到，第九阶段永远不运行。

### 3.1 触发与排序

`airy-boot-watch.timer` 在开机 60 秒拉起 `airy-boot-watch.service`。两者都
必须关掉 `DefaultDependencies`：缺省依赖自带 `Requires/After=sysinit.target`，
只看显式那一行而留着缺省，看门狗仍排在目标之后，在唯一需要它的场景里永远
拿不到执行机会。

服务只保留 `After=systemd-journald.socket` 一条排序（结论要写 journal 与
串口），`TimeoutStartSec=660` 严格大于脚本自身上限 600 秒，并容下两段限时
取证的余量；两者必须成对改动。服务不参与启动排序，`StandardOutput` 只取
journal，console 与串口由脚本按设备实际归属自己写——交给 systemd 再复制
一遍，会在 `/dev/console` 归属串口的机器上打出两行。

### 3.2 判定

判定只认观测，不认意图：

| 参数 | 默认 | 含义 |
|------|------|------|
| `AIRY_WATCH_POLL` | 10 秒 | 采样周期 |
| `AIRY_WATCH_WIN` | 40 秒 | 停滞窗口：连续这么久没有推进才算停滞 |
| `AIRY_WATCH_MAX` | 600 秒 | 观测者硬上限，与 `TimeoutStartSec` 成对 |
| `AIRY_WATCH_JOBS` | 8 | 报告里列出的未完成作业上限 |
| `AIRY_WATCH_TAIL` | 8 | journal 末尾行数 |
| `AIRY_WATCH_JTIME` | 5 秒 | 单次取证命令限时 |

推进的判据是「作业表或 journal 游标前移」：只看作业表会把「慢而有序」判成
卡死（aarch64 TCG 上 333 秒与 373 秒的作业表逐字相同，系统仍在写日志，图形
目标 407 秒才达成）。作业表为空不算推进——目标未达成却没有任何作业，说明
systemd 已停在别处。反复重启的单元让作业表每轮都变，这一事实记进 `churn=`
而不当作健康：最需要证据的卡点恰恰长这样。

三种收场：

| 条件 | 结论 |
|------|------|
| 目标已达成，登录管理器在跑 | 静默，健康路径不留一行 |
| 目标已达成，但 `graphical.target` 下没有登录管理器 | `STUCK`，`reason=nodm` |
| 窗口走满，目标仍未达成 | `STUCK`，`reason=unreached` |
| 撞上 `MAX` 仍在推进 | `SLOW`，`reason=cap` |

`SLOW` 不按卡死判负，等待时长由开机腿自己的上限决定。退出码恒为 0：观测者
以非零退出会在 `systemctl --failed` 里多出一条由自己造成的失败，掩盖真正的
失败者。

### 3.3 有界等待

窗口内没人按键是合法等待，不是卡死。会话选择器在 `tty1` 上等人最多 120 秒
（单元 `TimeoutStartSec=150`），这期间目标当然达不到，作业表也逐字不变。

故判定先读单元事实：挂起作业的单元处于 `activating` 且 `TimeoutStartUSec`
有限，说明 systemd 保证会在界内收尾，这一轮只落一行
`AIRYMAX-BOOT-WAIT unit=<单元> cap=<界> reason=bounded`，不推进停滞计数；
界到期后 systemd 自己收尾，作业表随之改变，判定自然恢复。界不可判定
（`infinity`、读数缺失）时不放行——对无界阻塞保持沉默比误判更糟。同一单元
持续等待只落一行，避免把串口刷成重复行。

### 3.4 报告行族

卡点报告按固定顺序落盘，每行一个事实，可逐行断言：

| 行 | 关键字段 | 回答什么 |
|----|----------|----------|
| `AIRYMAX-BOOT-STUCK` | `target=` `active=` `sub=` `uptime=` `idle=` `churn=` `reason=` | 哪个目标没到、停了多久 |
| `AIRYMAX-BOOT-SLOW` | `target=` `active=` `sub=` `uptime=` `idle=` `jobs=` `reason=cap cap=` | 慢而有序还是真的停了 |
| `AIRYMAX-BOOT-WAIT` | `unit=` `cap=` `reason=bounded` | 这是合法等待 |
| `AIRYMAX-BOOT-JOBS` / `-JOB` | `n=` / `unit=` `type=` `state=` `job=` | 还有哪些作业没回 |
| `AIRYMAX-BOOT-UNIT` | `unit=` `active=` `sub=` `pid=` `result=` | 首个未完成作业与目标的单元事实 |
| `AIRYMAX-BOOT-DM` | `unit=` `active=` `sub=` `result=` `restarts=`；`lightdm` / `greeter` / `xserver` / `xwrap` / `core` / `frame` | 桌面四种收场的证据 |
| `AIRYMAX-BOOT-STAGE` | `seen=n/9` `last=k` `token=` `up=` `stall=` | 九阶段走到第几步、停了多久 |
| `AIRYMAX-BOOT-LAST` | journal 末尾若干行 | 阻塞前的最后一句 |
| `AIRYMAX-BOOT-FAILED` | `n=` | 失败单元数 |
| `AIRYMAX-BOOT-CONSOLE` | `active=` | 输出到底落在哪 |
| `AIRYMAX-BOOT-CMDLINE` | `argv="..."` | 通道归属与 `console=` 顺序 |
| `AIRYMAX-BOOT-HINT` | `cmd="..."` | 下一步该敲什么 |

`AIRYMAX-BOOT-STAGE` 取 journal 里的中文行而不是各单元 stdout：中文行恰好
每阶段每开机一条，不会被单元输出重复计入。`seen=0` 说明开机在第一个阶段
之前就停了（initramfs 或根切换），这比只看 journal 末尾更能钉住卡在哪一层。
`stall=` 与 `up=` 同源，都按开机时钟计算，NTP 校时不会把停滞时长算成负数。

证据段允许缺省：文件读不到、`coredumpctl` 缺席、journald 未起时跳过该段。
观测者不得因取证而变成新的卡点。

---

## 第 4 章 会话内健康探针

启动完成之后，`boot_state.sh` 在会话内补一次失败单元盘点：

| 行 | 语义 |
|----|------|
| `AIRYMAX-BOOT-HEALTH failed=0` | 无失败单元 |
| `AIRYMAX-BOOT-DEGRADED failed=<n> units=<列表>` | 有失败单元，附每个单元的 `AIRYMAX-UNIT-STATE|` 与 `AIRYMAX-UNIT-JOURNAL|` |
| `AIRYMAX-BOOT-DEGRADED failed=unknown units=none` | 盘点本身失败，按不得放行处理 |

这两行是观测通道，不参与任何判定；判定只看 `HEALTH` 与 `DEGRADED`。安装
验收另取 `AIRYMAX-BOOT-CHAIN|` 转发 `systemd-analyze critical-chain` 原文，
只在取证夹具里出现，不是每次开机都打印。

---

## 第 5 章 现场判读

| 现象 | 先看 | 结论 |
|------|------|------|
| 屏幕停在 `[ k/9 ] 代号 ...` 且有心跳 | 心跳 `used=` 与单元 `TimeoutStartSec` | 该阶段还在跑，超过上限才由 systemd 收尾 |
| 有心跳但 `used=` 卡住不涨 | `AIRYMAX-BOOT-UNIT` | 工具阻塞在系统调用，心跳每周期仍会重绘 |
| 没有任何阶段行 | `AIRYMAX-BOOT-STAGE seen=0` | 第一个阶段之前就停了 |
| `STUCK reason=unreached` | `AIRYMAX-BOOT-JOB` 第一条 | 首个未完成作业即卡点 |
| `STUCK reason=nodm` | `AIRYMAX-BOOT-DM unit=` | 目标到了但登录管理器没跑 |
| `SLOW reason=cap` | `AIRYMAX-BOOT-STAGE` 与 critical-chain | 慢而有序，查磁盘与单元依赖链 |
| `WAIT reason=bounded` | 单元的 `TimeoutStartUSec` | 合法等待，不要按卡死处理 |
| 屏幕有行、串口没有 | `AIRYMAX-BOOT-CONSOLE` 与 `CMDLINE` | `console=` 顺序；通道判据见 `con.sh` |
| `FAILED n>0` | `systemctl --failed` | 失败单元与该阶段的工具同源 |

---

## 第 6 章 验收判据

| 编号 | 场景 | 通过条件 |
|------|------|----------|
| AC-B1 | 九阶段完整性 | 正常启动后 journal 中每个阶段恰好一条结论行，序号与总数来自阶段表 |
| AC-B2 | 阶段重跑不刷屏 | timer 重跑的单元在结论不变时不重复落阶段行与快照 |
| AC-B3 | 卡点归因 | 人为让某作业不返回，开机 60 秒后 40 秒窗口内出现 `AIRYMAX-BOOT-STUCK` 与作业表，退出码为 0 |
| AC-B4 | 有界等待 | 会话选择窗口内只出现 `AIRYMAX-BOOT-WAIT`，无 `STUCK` |
| AC-B5 | 健康静默 | 正常启动的串口与 journal 中没有 `STUCK` / `SLOW` |
| AC-B6 | 契约错误可见 | 传入不在表内的阶段名，console 出现 `airymax-boot:` 事实行，退出码 5 |
| AC-B7 | 三架构同形 | x86_64 / aarch64 / riscv64 上阶段代号、序号、token 与关键值格式一致 |

---

## 第 7 章 相关文档

- [100-operations README](README.md) —— 运维体系主索引
- [03-monitoring.md](03-monitoring.md) —— 运行时监控数据源
- [07-systemd-integration.md](07-systemd-integration.md) —— 单元与守护进程集成
- [190-distribution/07-session-entry.md](../190-distribution/07-session-entry.md) —— 会话选择与有界等待
- [190-distribution/03-installer-design.md](../190-distribution/03-installer-design.md) —— 安装期标记与桌面就绪探针
- [20-modules/07-system.md](../20-modules/07-system.md) —— 系统模块与会话入口
