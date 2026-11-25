Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 会话入口与模式切换
> **文档定位**：agent-linux（AirymaxOS）发行版管理——引导期会话选择、运行期 TUI ↔ GUI 双向切换与注销回退的工程设计\
> **文档版本**：v1.0.3\
> **最后更新**： 2026-11-25\
> **上级文档**：[190-distribution README](README.md)\
> **同源映射**：[20-modules/07-system.md](../20-modules/07-system.md) 会话入口、[20-modules/02-services.md](../20-modules/02-services.md) 单元归属\
> **核心约束**：会话状态只有一处事实源；切换只由 root 的 oneshot 单元执行；任何失败都必须回到可用会话\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0\
> **SSoT 依赖声明**：本文档所引用的全部规则编号（`OS-IRON-*`、`OS-BAN-*`、`OS-STD-*`、`OS-ACC-*` 等）权威定义见 `50-engineering-standards/09-ssot-registry.md`，本文档不私自定义新规则编号。\
> **IRON-9 v3 层次**：[IND] 完全独立层（会话入口为 agent-linux 发行版专属）

---

## 目录

- [第 1 章 定位与边界](#第-1-章-定位与边界)
- [第 2 章 状态面：desired 与 mode](#第-2-章-状态面desired-与-mode)
- [第 3 章 引导期会话选择](#第-3-章-引导期会话选择)
- [第 4 章 运行期模式切换](#第-4-章-运行期模式切换)
- [第 5 章 注销回退与兜底](#第-5-章-注销回退与兜底)
- [第 6 章 失败形态与可观测性](#第-6-章-失败形态与可观测性)
- [第 7 章 打包归属与安装面](#第-7-章-打包归属与安装面)
- [第 8 章 验收判据](#第-8-章-验收判据)
- [第 9 章 相关文档](#第-9-章-相关文档)

---

## 第 1 章 定位与边界

### 1.1 为什么属于发行版管理

同一份镜像要同时服务两类装机：工作站需要图形桌面，无人值守节点只需要字符界面。
两者不是两套镜像，而是一次安装后的**两种会话入口**。入口归谁选择、切换时谁有权写状态、
打包时哪些文件归哪个包，都是发行层面的事实，不能留给某个程序在运行期临时约定。

本章定义三个交付件：

| 交付件 | 作用 | 运行时机 |
|--------|------|----------|
| 引导期选择器 | 在一次 boot 内决定进 TUI 还是 GUI | `graphical.target` 之前 |
| 切换单元模板 | 不重启机器地在两种会话间往返 | 会话就绪之后 |
| 用户入口命令 | 用户侧发起切换的唯一接口 | 任意已就绪会话内 |

### 1.2 边界

- 本文只覆盖**入口与切换**，不覆盖会话内部实现：TUI 的渲染、输入与退出语义见
  [20-modules/07-system.md](../20-modules/07-system.md)；图形桌面自身的登录、座位与
  就绪探针见 [03-installer-design.md](03-installer-design.md)。
- 不新增调度、不引入 D-Bus 依赖、不改 `default.target`。安装器的默认目标断言是既有
  契约，改它等于让安装门禁判负。
- 不新增「切换就绪」标记：两种会话各自的既有就绪信号就是判据，切换层只读不造。

---

## 第 2 章 状态面：desired 与 mode

### 2.1 两个文件，两种语义

| 文件 | 语义 | 写入时机 | 读取者 |
|------|------|----------|--------|
| `/run/airymaxos/session/desired` | 意图态：本次 boot 想要哪个会话 | 启停单元**之前** | 只有 systemd 的 `ExecCondition` |
| `/run/airymaxos/session/mode` | 实际态：已经就绪的会话 | 就绪**之后** | 程序与 CLI 一律只读此文件 |
| `/run/airymaxos/session/switch-reason` | 最近一次切换原因 | 发起切换时 | 排障与验收脚本 |

程序不得读取 `desired`：意图与实际不一致是切换进行中的正常中间态，程序按意图渲染
会在切换失败时显示与屏幕相反的状态。两个文件都用「同目录临时文件 + `rename`」原子替换，
不允许先截断再写。

### 2.2 写者集合封闭

- 运行期：只有 `airy-session-switch@.service` 写 `mode` 与 `desired`。
- 引导期：只有 `airy-session-state` 这个 root helper 在两条就绪路径的 `ExecStartPost`
  中写初始 `mode`，并在同一事务内把 `desired` 对齐为同值。

任何程序（含 CLI）都不写状态文件。状态目录 `/run/airymaxos/session` 属 root、模式 0755，
由写者 `create_dir_all` 自建——`/run` 是 tmpfs，不能假设目录存在，也不引入新的
`tmpfiles.d` 条目来制造第二处真相。

### 2.3 对齐规则

每次写 `mode` 都在同一事务里把 `desired` 写成同值，因此 `desired ≠ mode` 只出现在
「切换已发起、尚未就绪」的窗口内。回滚路径同样先写 `desired=<旧模式>`，再启回旧单元，
最后写 `mode`；只启回单元而留下残留 `desired` 判负。

---

## 第 3 章 引导期会话选择

### 3.1 选择器

`airy-session-choice.service` 在选择窗口内独占 `tty1`，与 `getty@tty1.service` 以
`Conflicts=` + `After=` 排序，输入输出走 `StandardInput=tty` / `StandardOutput=tty`。
选择器不链渲染栈、不进备用屏，只维护一行倒计时，符合「无动画」的引导期原则。

| 项 | 设计 |
|----|------|
| 窗口 | 120 秒；超时取默认值 GUI |
| 输入 | 方向键选择 + 回车确认；超时自动确认 |
| 输出 | 一行倒计时，随剩余秒数重绘 |
| 写入 | 仅写 `desired`；`tui` 分支显式 `systemctl start boot-tui.service` |

选择器不把 `boot-tui` 写成 `Wants=`：`Wants=` 无法按选择结果条件生效，会产生
「选了 GUI 也起 TUI」。

结论只由一条判据链得出，顺序固定：命令行旁路 → 无键盘 → tty1 交互倒计时，
都拿不到就取 `--default`。每次结论落一行 `AIRYMAX-SESSION choice=<gui|tui>
reason=<…>`，取值如下：

| reason | 含义 |
|--------|------|
| `cmdline` | 命令行显式关闭会话选择（`airymax.session=off`） |
| `no-keyboard` | 输入类可读且没有任何字母键设备，直接进桌面 |
| `key` | 倒计时内按键确认 |
| `timeout` | 窗口到期，取默认值 |
| `no-tty` / `no-termios` / `no-raw` / `poll` | 拿不到可交互终端，回落默认值 |

无键盘判据取 `/sys/class/input/input*/capabilities/key` 位图中的 KEY_A，与验收侧
同一口径：电源键、休眠键不构成键盘。输入类读不到时照常等待——判不了有没有
键盘就不替用户做决定，裁剪过的宿主也不会因此被误跳过。

窗口内没人按键是合法等待，不是卡死。单元用 `TimeoutStartSec=150` 封住自身时长，
开机看门狗遇到「作业挂起 + 单元 activating + 界有限」时只落一行
`AIRYMAX-BOOT-WAIT unit=airy-session-choice.service cap=<界> reason=bounded`，
不计入停滞；界到期后 systemd 收尾该单元，看门狗回到常规判定。界不可判定
（`infinity` 或读数缺失）时不放行——对无界阻塞保持沉默比误判更糟。

### 3.2 条件跳过的落点

`lightdm.service` 与 `airymax-desktop-ready.service` 各挂一个同名 drop-in，内容是一条
`ExecCondition=/usr/lib/airymaxos/desktop/airy-session-cond.sh`。判据为：

- `desired` 缺失 → 放行（失败安全）；
- `desired=gui` → 放行；
- `desired=tui` → 退出码 1，单元被**条件跳过**而非失败。

跳过必须写 journal `AIRYMAX-SESSION-SKIP unit=…`；无痕迹的静默跳过判负。
目标基底 systemd 低于 243 时 `ExecCondition` 不可用，退回「`ConditionPathExists` +
派生标记文件」，并在技术债台账显式登记，不得静默降级。

`default.target` 保持 `graphical.target`：TUI 分支下目标照常到达，只是两个桌面单元被
条件跳过，`Wants=` 作业因此不带降级标记；运行期切 GUI 时先写 `desired=gui`，同一条件
立即放行。

---

## 第 4 章 运行期模式切换

### 4.1 唯一入口

用户侧只有 `airy-session-mode <gui|tui>`：

| 情形 | 退出码 | 行为 |
|------|--------|------|
| 参数不是 `gui`/`tui` | 2 | 打印用法，不动作 |
| 系统未就绪（`is-system-running` 非 `running`/`degraded`） | 4 | 不动作 |
| 目标等于当前 `mode` | 0 | 不重启任何单元 |
| 其余 | 0 | 提交切换作业后立即返回 |

入口以 `--no-block` 提交作业：TUI 侧调用后自身会被切换单元停掉，CLI 作为子进程随即
被回收，「等 `systemctl` 返回再打印」在切换路径上永远等不到结果。

### 4.2 切换单元

切换只由 root 的 oneshot 模板 `airy-session-switch@.service` 执行，实例名只接受
`gui` 与 `tui`。顺序固定，任一步失败即进入回滚：

1. 校验目标与当前状态；
2. 写 `desired`；
3. 停旧会话并等待其 `inactive`；
4. 等待 VT 释放；
5. 启动新会话；
6. 等待就绪；
7. 写 `mode` 并在同一事务内对齐 `desired`；
8. 写 journal `AIRYMAX-SESSION-MODE from=… to=… reason=…`。

先写 `mode` 再启会话判负：失败后状态与事实相反，下一次切换会把已运行的会话「再切一遍」。

### 4.3 就绪判据

| 方向 | 判据 |
|------|------|
| GUI | 本次 boot 新出现 `AIRYMAX-DESKTOP-READY`，且 `systemctl is-active lightdm` 为 active |
| TUI | `systemctl is-active boot-tui` 为 active，`Type=notify` 的 `READY=1` 已上报，且 `tty1` 前台进程组是 `boot-tui` |

GUI 方向必须同时启动就绪探针 `airymax-desktop-ready.service`：该单元只被
`graphical.target` 拉取，只 `start lightdm` 不会有进程打印标记，等待必然超时。
探针文件属既有交付，切换层只挂 drop-in、只启动，不修改。

### 4.4 超时与回滚

| 方向 | 就绪窗口 | 超时行为 |
|------|----------|----------|
| GUI | 90 秒 | `desired` 改回旧模式 → 启回旧单元 → 写回 `mode` |
| TUI | 15 秒 | 同上 |

回滚后写 `switch=failed:<cause>` 到 journal 与状态目录，不得停在黑屏。任何超时后
既不回滚也不报错的行为判负。

### 4.5 前台 VT

`TTYPath=` 只决定进程打开哪个终端，不保证该 VT 在前台。

- TUI 方向：切换单元显式 `chvt 1`；
- GUI 方向：优先依赖 X 自行激活所在 VT；lightdm 就绪 30 秒后 `fgconsole` 仍未变化时，
  兜底 `chvt <n>`，`n` 取 `loginctl list-sessions --no-legend` 的 TTY 列。

禁止解析 `seat-status`：那是面向人的画板输出，格式随版本漂移，不能作为机读判据。
验收同时看 `fgconsole` 与单元状态，否则会把「切了但看不见」判成成功。

---

## 第 5 章 注销回退与兜底

图形会话注销后不应停在 greeter：无人登录的机器回到字符入口更符合「可诊断优先」。

`session-cleanup-script` 只在系统处于 `running` 或 `degraded` 时，把 `logout` 写入
`switch-reason` 并启 `airy-session-switch@tui.service`。脚本本身不写 `mode`——写状态
只归单元，权限模型不因省一个分支而破例。

TUI 侧异常退出不自动重启：崩溃后回到 `getty@tty1`，可登录查看 journal。
`boot-tui` 的 `ExecStopPost` 以 `+-` 前缀回落 getty（`+` 保证 root 执行、`-` 忽略失败），
崩溃与主动退出都执行；被 `Conflicts=` 停掉的 getty 不会自行复活。

---

## 第 6 章 失败形态与可观测性

| 失败形态 | 可观测点 | 结论 |
|----------|----------|------|
| 选择器未启动 | tty1 直接出现登录提示符 | 引导期选择未生效 |
| 条件脚本静默跳过 | 无 `AIRYMAX-SESSION-SKIP` journal 行 | 判负 |
| 切换后单元全 active 但屏幕未变 | `fgconsole` 与目标 VT 不一致 | 切换未完成 |
| GUI 就绪超时 | `switch=failed:gui-timeout` | 已回滚到 TUI |
| `desired` 残留旧值 | `desired ≠ mode` 且无进行中作业 | 回滚不完整 |
| 状态目录缺失 | 写者报错，`mode` 不出现 | 写者未自建目录 |

所有状态变更必须落 journal，格式固定为 `AIRYMAX-SESSION-<事件> key=value`，供验收脚本
逐行断言；不接受「只写文件不写日志」的静默路径。

---

## 第 7 章 打包归属与安装面

会话入口的文件归属必须写进 spec，否则目标机上文件不存在：

| 文件 | 归属包 | 说明 |
|------|--------|------|
| `boot-tui.service` | `airymaxos-tui` | `Type=notify`、`Conflicts=getty@tty1.service`、`Restart=no` |
| `airy-session-choice.service` | `airymaxos-tui` | 引导期选择器，独占 tty1 |
| `airy-session-switch@.service` | `airymaxos-tui` | 运行期切换 oneshot 模板 |
| `airy-session-mode` | `airymaxos-tui` | 用户入口命令，可执行 |
| `airy-session-state` | `airymaxos-tui` | root helper，只由 `ExecStartPost` 调用 |
| `airy-session-cond.sh` | `airymaxos-tui` | `ExecCondition` 脚本，可执行 |
| `10-airymax-session-choice.conf` | `airymaxos-tui` | 两个桌面单元的 drop-in |
| polkit 规则 | `airymaxos-tui` | 仅授权 `amuser` 发起切换作业 |
| `/run/airymaxos/session` | 运行期自建 | 不打包、不建 tmpfiles 条目 |

发布门禁逐条断言上述路径在包内为**实体文件**且入口为可执行位；软链判负。

---

## 第 8 章 验收判据

| 编号 | 场景 | 通过条件 |
|------|------|----------|
| AC-A3 | 引导期选择 | 选 TUI 后 5 秒内出现 TUI 首帧，桌面单元被跳过且 journal 有记录；选 GUI 后到 `AIRYMAX-DESKTOP-READY` |
| AC-A10 | 运行期双向切换 | TUI→GUI→TUI 各一次，每次 `mode` 与实际屏幕一致，`fgconsole` 指向目标 VT，无残留 `desired` |
| AC-A10-b | 切换回滚 | 人为阻断目标会话启动，超时后回到原会话，`switch=failed:<cause>` 可检索 |
| AC-A10-c | 注销回退 | GUI 注销后 15 秒内回到 TUI，`switch-reason=logout` |

三架构（x86_64 / aarch64 / riscv64）执行同一判据；字符入口不依赖图形栈，riscv64 上
必须与 x86_64 表现一致。

---

## 第 9 章 相关文档

- [190-distribution README](README.md) —— 发行版管理主索引
- [20-modules/07-system.md](../20-modules/07-system.md) —— 系统模块与会话入口
- [20-modules/02-services.md](../20-modules/02-services.md) —— 单元与守护进程归属
- [03-installer-design.md](03-installer-design.md) —— 安装、座位与桌面就绪探针
- [100-operations/README.md](../100-operations/README.md) —— 部署与排障
