Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 输入接入（Input Ingress）契约
> **文档定位**：定义 AirymaxOS 全部用户意图的统一接入契约，覆盖开机 TUI、桌面 dock、语音、
> 外接结构化 API 与批处理五类来源，并规定两条互不链接的运行时通路。\
> **文档版本**：v1.0.0\
> **最后更新**：2026-11-17\
> **上级文档**：[接口设计总览](README.md)\
> **相关文档**：[01-syscalls.md](01-syscalls.md)、[08-sc-error-contract.md](08-sc-error-contract.md)、
> [02-ipc-protocol.md](02-ipc-protocol.md)、`docs-closed/agent-linux-dev/10-references/base-definition-and-replacement.md`

---

## 1. 定位与边界

### 1.1 解决什么问题

同一句用户意图，可能来自键盘、语音、外部程序或脚本。若每个入口各自解析、各自审批、
各自执行，就会出现四种行为、四套日志、四种失败语义。本契约把「意图」抽象为一等对象，
所有入口只负责**产生 Intent**，执行与审批由通路自行完成。

### 1.2 边界

- 本契约**只定义接入面**：Intent 结构、通道职责、信任级别、审批记录、错误语义。
- 不定义智能体内部计划格式，不定义工具注册表，不定义模型提示词。
- 不新增内核 ABI：不新增 syscall、不新增 ioctl、不新增 Fault 码。
- 不规定 UI 布局；布局属各通路自己的实现。

### 1.3 一个契约层、两条独立通路（权威表述）

| 项 | 内容 |
|---|---|
| 契约层 | Intent schema、trace_id、信任级别、审批记录、错误语义 |
| 通路 A | **boot TUI**：独立 Rust 会话层，本地解析、本地执行，**不链接 agentrt** |
| 通路 B | **桌面 dock**：AirymaxRT gateway（HTTP/JSON-RPC + SSE），智能体计划与工具执行在 runtime 内 |

两条通路共享契约，不共享运行时、不共享渲染器、不共享进程。任何「抽公共渲染器/公共执行器」
的实现都违反本契约，评审直接判负。

---

## 2. Intent 结构

### 2.1 字段表

| 字段 | 类型 | 必填 | 语义 |
|---|---|---|---|
| `schema` | string | 是 | 固定 `airy.intent/1`；MAJOR 变更才改此值 |
| `trace_id` | string(32) | 是 | 十六进制，随执行链路透传；重试必须复用 |
| `ts` | string | 是 | RFC 3339，产生时刻，用于乱序与超时判定 |
| `source.kind` | enum | 是 | `keyboard` / `voice` / `api` / `batch` |
| `source.device` | string | 否 | 设备标识；无设备时为 `local` |
| `trust.level` | enum | 是 | `T0`–`T3`，见 §4 |
| `trust.principal` | string | 是 | 发起者标识：`console` / `uid:1000` / `svc:<name>` |
| `session.mode` | enum | 是 | `tui` / `gui` / `headless` |
| `session.id` | string | 是 | 会话标识，用于审计关联 |
| `payload.kind` | enum | 是 | `text` / `structured` / `audio-ref` |
| `payload.text` | string | 条件 | `kind=text` 必填；UTF-8，已含 IME 提交结果 |
| `payload.lang` | string | 否 | BCP 47；缺省 `zh-CN` |
| `payload.structured` | object | 条件 | `kind=structured` 必填；见 §2.3 |
| `payload.audio_ref` | object | 条件 | `kind=audio-ref` 必填；见 §3.2 |
| `constraints.deadline_ms` | integer | 否 | 端到端预算；超时即取消，不给部分结果 |
| `constraints.allow` | array | 否 | 允许的动作类别白名单 |
| `constraints.deny` | array | 否 | 禁止的动作类别黑名单，优先级高于 allow |
| `approval` | object | 否 | 高危动作的审批凭据，见 §4.3 |

### 2.2 文本意图示例

```json
{
  "schema": "airy.intent/1",
  "trace_id": "9f2c1b7e4d6a4c0e8b3f5a1d2c7e9b40",
  "ts": "2026-11-17T10:02:11+08:00",
  "source": {"kind": "keyboard", "device": "tty1"},
  "trust": {"level": "T0", "principal": "console"},
  "session": {"mode": "tui", "id": "s-0f31"},
  "payload": {"kind": "text", "text": "查看磁盘使用情况", "lang": "zh-CN"},
  "constraints": {"deadline_ms": 5000, "deny": ["disk.format"]}
}
```

### 2.3 结构化意图

`payload.structured` 只接受**已注册动作**，不接受自由形式的工具调用：

```json
{
  "action": "disk.status",
  "args": {"unit": "human", "paths": ["/", "/data"]},
  "expect": {"schema": "airy.disk.status/1"}
}
```

未知 `action` 一律拒绝（`-EINVAL`），不得「尽力猜一个相近动作」。注册表由执行侧持有，
接入侧只做语法校验。

### 2.4 校验顺序

1. `schema` 与必填字段（不通过 → `-EINVAL`，`ingress_status=invalid`）。
2. `trace_id` 格式与唯一性（重试复用允许；新请求复用 → `-EINVAL`）。
3. `trust.level` 与 `source.kind` 的匹配（见 §4.1，不匹配 → `-EACCES`）。
4. `payload` 与 `kind` 的一致性（不通过 → `-EINVAL`）。
5. 白/黑名单（命中 deny → `-EPERM`）。
6. 执行侧的动作注册与参数校验（不通过 → `-EINVAL`）。

---

## 3. 五个来源（四通道 + 批处理）

### 3.1 C1 键盘与输入法

| 项 | 约定 |
|---|---|
| 原始按键 | 由终端读取，**不**产生 Intent |
| 中文输入 | 由独立 IME 进程处理，提交结果作为 `payload.text` 进入 Intent |
| IME 依赖 | 独立 Rust crate；只共享 `AIRYIME1` 词典格式与数据资产，**不链接 agentrt** |
| 分派规则 | 输入法激活态下的按键归 IME；提交（Enter/空格选词）后才形成 Intent |
| 失败判据 | IME 进程退出 → 文本仍可输入（直通），并在状态栏标 `ime=down`；不得静默丢字 |

GUI 会话的中文输入走 IBus（系统既有闭包），boot TUI 不复用 IBus 的 daemon 形态；
两者的共同点只有词典数据与「提交即文本」这一语义。

### 3.2 C2 语音

| 项 | 约定 |
|---|---|
| 采集 | 由 ASR 网关进程独占录音设备；TUI 主进程不直接读 PCM |
| 识别 | 本地离线模型；断网可用是硬要求 |
| 传输 | 识别结果以 `payload.kind=text` 注入；仅在需要回放时保留 `audio-ref` |
| 音频引用 | `{"path": "...", "digest": "sha256:...", "duration_ms": 3120}` |
| 失败判据 | 模型缺失/设备忙/识别为空 → 明确状态 `asr=unavailable|busy|empty`，不伪造文本 |

`audio-ref` 只引用本地受管目录，接入侧不解析路径；digest 与实物不符即拒绝（`-EINVAL`）。

### 3.3 C3 外接结构化 API

| 项 | 约定 |
|---|---|
| 传输 | Unix domain socket，JSON Lines；单连接单会话 |
| 认证 | `SO_PEERCRED` 取 uid/gid/pid；不在消息里自报身份 |
| 信任 | 认证通过 → `T1`；`root` 且显式声明 → `T2`，仍需审批高危动作 |
| 背压 | 单连接未完成请求上限 8；超出 → `-EAGAIN`，不排队到无限 |
| 幂等 | 相同 `trace_id` 的重试返回同一结果引用，不重复执行 |

### 3.4 C4 批处理（stdin / 文件）

| 项 | 约定 |
|---|---|
| 用途 | CI、验收脚本、无人值守任务 |
| 形态 | 每行一个 Intent JSON；读到 EOF 结束 |
| 信任 | `T3`，**默认禁止**高危动作，且不可通过参数提升 |
| 输出 | 每行一个结果 JSON，与输入按 `trace_id` 对应 |
| 失败判据 | 单行失败不终止整批；末行汇总必须给出成功/失败计数与非零退出码 |

### 3.5 非来源：指针与触摸

指针事件属于 UI 内部交互，不直接形成 Intent；只有 UI 把一次交互解释为「用户意图」时
（例如点击 dock 的发送按钮）才按 C1 的规则产生 Intent。

---

## 4. 信任与审批

### 4.1 来源与信任级别匹配

| 来源 | 允许的最低/最高级别 | 说明 |
|---|---|---|
| C1 键盘 | T0 | 物理在场 |
| C2 语音 | T0 | 与键盘同权；识别结果不降低信任 |
| C3 API | T1–T2 | 由 `SO_PEERCRED` 决定 |
| C4 批处理 | T3 | 不可提升 |

### 4.2 级别语义

| 级别 | 语义 | 高危动作 |
|---|---|---|
| T0 | 本地控制台用户 | 允许，但需二次确认 |
| T1 | 已认证会话用户 | 允许，但需审批凭据 |
| T2 | 服务主体 | 允许，审批凭据 + 审计 |
| T3 | 自动化 | 默认拒绝，仅白名单动作 |

### 4.3 审批记录

审批不是布尔值，而是一条可审计记录：

```json
{
  "approval": {
    "id": "ap-7d1e",
    "by": "console",
    "ts": "2026-11-17T10:02:14+08:00",
    "scope": ["disk.format"],
    "trace_id": "9f2c1b7e4d6a4c0e8b3f5a1d2c7e9b40",
    "expires_ms": 60000
  }
}
```

约束：

1. 审批 `scope` 必须覆盖实际动作，且只对同一 `trace_id` 有效。
2. 审批不可由被审批方自行生成；boot TUI 由本地确认对话框产生，dock 由 gateway 的确认流程产生。
3. 审批过期即作废，不因重试而续期。
4. 审批记录与执行记录同源落审计，缺任一半视为未审批。

### 4.4 判定权归属

最终判定不在接入侧，也不在 UI。策略判定由既有安全通道完成：
`/sys/kernel/security/airy/policy` 与 LSM 钩子。接入侧只做「语法 + 信任级别 + 白黑名单」
三级预检；预检通过不代表允许执行。

---

## 5. 两条通路的执行语义

### 5.1 通路 A：boot TUI（0.0.5）

| 项 | 约定 |
|---|---|
| 进程 | 独立 Rust 会话层；不链接 agentrt，不依赖 gateway |
| 意图解析 | 本地规则 + 结构化动作；不做自由文本的工具规划 |
| 执行 | 调用既有 `airy-*` CLI 或已冻结的 `--json` 接口 |
| 审批 | 本地 TUI 对话框；确认结果写入审批记录 |
| 失败 | 明确错误码 + 可执行修复建议；不得吞错 |
| 离线 | 全路径离线可用；不发起任何网络请求 |

### 5.2 通路 B：桌面 dock（0.0.6）

| 项 | 约定 |
|---|---|
| 进程 | Slint dock（客户端）+ AirymaxRT gateway（服务端） |
| 传输 | HTTP/JSON-RPC + SSE（`agent.run_stream`） |
| 意图 | dock 把 Intent 提交给 gateway；计划与工具执行在 runtime 内 |
| 审批 | gateway 发起确认，dock 呈现；审批记录回传 runtime |
| 失败 | SSE 断流必须给出 `stream=closed` 与最后事件序号，便于续传 |
| 复用边界 | 复用 AirymaxRT 的智能体能力，不复用 boot TUI 的进程与渲染 |

### 5.3 契约一致性

两条通路的 Intent 字段、错误语义、审批记录结构必须逐字段一致。差异只允许出现在
「执行器是谁」与「UI 长什么样」。每季度以同一组语料各跑一次，结果差异必须为 0。

---

## 6. schema 版本与兼容

| 规则 | 内容 |
|---|---|
| 版本形态 | `airy.intent/MAJOR`；MINOR 不进 `schema` 字段 |
| MAJOR 内 | 只允许新增可选字段；禁止删除字段、禁止改变语义、禁止收紧校验 |
| 消费方 | 必须忽略未知字段；必须拒绝未知 `payload.kind` |
| 生产方 | 必须填全部必填字段；不得依赖消费方的默认值补全 |
| 弃用 | 字段弃用先标记一个 MINOR 周期，再于下一 MAJOR 删除 |
| 破坏性变更 | 升 MAJOR，并同时提供旧版接受期（至少一个预览版本） |

---

## 7. 错误语义

用户态错误沿用既有错误契约的 errno 负值，不新增 Fault 码（Fault 表示内核不可恢复语义，
接入错误不属此类）。

| 情形 | 返回值 | `ingress_status` | 附加 |
|---|---|---|---|
| JSON 不合法 | `-EINVAL` | `invalid` | 解析位置 |
| 必填字段缺失 | `-EINVAL` | `invalid` | 字段名 |
| 未知动作 | `-EINVAL` | `unsupported` | 动作名 |
| 信任级别不足 | `-EACCES` | `denied` | 所需级别 |
| 白黑名单拒绝 | `-EPERM` | `denied` | 命中项 |
| 未审批/审批过期 | `-EACCES` | `approval-required` | 动作类别 |
| 超时 | `-ETIMEDOUT` | `timeout` | 预算毫秒 |
| 背压 | `-EAGAIN` | `busy` | 未完成数 |
| 依赖不可用（IME/ASR/模型） | `-ENOTSUP` | `unavailable` | 组件名 |
| 结果不可序列化 | `-EIO` | `internal` | 事件序号 |

所有错误必须同时满足三条：有稳定状态串、有可执行建议、有 `trace_id`。
三者缺一即视为实现不合格。

---

## 8. 中文与显示

1. 输入：IME 提交的文本按 UTF-8 原样进入 `payload.text`，接入侧不做拼音/字形转换。
2. 显示：TUI 必须能在无 GUI 的 console 上显示中文；字形缺失时显式降级为方块并报
   `font=missing`，不得输出乱码或静默替换。
3. 宽度：按东亚宽度计算列宽；不得按字节或码点计宽。
4. 排序与检索：不做依赖 locale 的隐式排序；需要排序时显式指定。

---

## 9. 验收判据

| # | 判据 | 方式 |
|---|---|---|
| V-1 | 四通道各有端到端成功路径 | 键盘（中文）、语音、API、批处理各一条 |
| V-2 | 四通道各有可判定失败路径 | 上表 §7 每类至少一条 |
| V-3 | 中文输入与显示在无 GUI 环境下可用 | tty1 实机/伪终端 |
| V-4 | IME 进程退出不丢字 | 杀进程后继续输入 |
| V-5 | 未审批的高危动作必被拒绝 | 构造无 `approval` 的 `disk.format` |
| V-6 | trace_id 全链路可追 | 从接入日志追到执行记录 |
| V-7 | 批处理单行失败不终止整批 | 混合成功/失败输入 |
| V-8 | 断网下语音与 TUI 全路径可用 | 隔离网络 |
| V-9 | 两条通路契约字段一致 | 同语料双跑，schema diff 为 0 |

---

## 10. 输入抽象层（`boot-tui/src/input/`）

前九章定义「进来的东西长什么样」，本章定义「怎么把四种完全不同的东西变成同一个样子」。
抽象边界选错会让语音结果被拼音引擎二次转换、让 API 输入被迫走键盘路径——两者都是
本设计要排除的形态。

### 10.1 三层，边界的名字叫 RawEvent

| 层 | 类型 | 产生者 | 消费者 |
|---|---|---|---|
| L1 事件层 | `RawEvent { trace_id, source, trust, at, body }` | 四个适配器 | 组合层 |
| L2 组合层 | `Composition`（preedit / candidates / commit） | 仅 `Key` 源进 IME；`Voice` / `Structured` / `Line` 直接产文本或结构 | 契约层 |
| L3 契约层 | `airy.intent/1`（§2） | 组合层 | `intent` crate → `airy-tui-exec` |

关键设计：**抽象的单位是事件，不是文本**。文本只是 `Key` 源经过 IME 之后的产物；
把语音和结构化输入也塞进「文本 -> 输入法」的管线，是抽象错误的典型症状。

### 10.2 trait 与四个适配器

```rust
pub enum SourceKind { Key, Voice, Api, Batch }

pub enum RawBody {
    Key(KeyEvent),                                      // 键码 + 修饰键，不含文本
    Transcript { text: String, confidence: f32, audio_ref: AudioRef },
    Structured(serde_json::Value),                      // 已结构化，待 schema 校验
    Line(String),                                       // 批处理一行
}

pub struct RawEvent {
    pub trace_id: TraceId,
    pub source: SourceKind,
    pub trust: Trust,        // T0–T3，见 §4.1
    pub at: MonoTime,
    pub body: RawBody,
}

pub trait InputSource: Send {
    fn kind(&self) -> SourceKind;
    fn drain(&mut self, max: usize) -> Result<Vec<RawEvent>, SourceError>;
    fn close(&mut self) -> Result<(), SourceError>;
}
```

| 适配器 | 数据来源 | 必须遵守 |
|---|---|---|
| `KeySource` | `/dev/tty` 原始键事件 | **只产键码，不产文本**；文本只能由 `ImeSession` commit 或显式 `ascii-passthrough` 产生 |
| `VoiceSource` | 只与 `asr-gateway` 的 socket 通信 | 不打开录音设备；`Transcript` 先进可编辑缓冲区，用户确认后才成为 Intent |
| `ApiSource` | Unix socket + JSON Lines | 直接收 `airy.intent/1`，只做 schema 校验与背压 |
| `BatchSource` | stdin / 文件 | 逐行独立 `trace_id`；单行失败不终止整批（§7） |

`VoiceSource` 的确认环节不是可选项：语音不绕过审批（§4.1），也不允许「识别即执行」。

### 10.3 IME 会话协议

- 传输：`/run/airy/ime.sock`，JSON Lines，与 Intent 入口同一 framing——同一套日志与回放工具
  可同时处理输入与意图。
- 消息：`open` / `key` / `select` / `commit` / `reset` / `close`，每条带 `session` 与 `seq`。
- 状态机：`Idle → Composing(preedit) → Selecting(n) → Commit`；`Esc` 回 `Idle` 并清 preedit。
- 崩溃恢复：`boot-tui` 保留未提交键的环形缓冲（上限 256 事件），`boot-ime` 重启后按 `seq`
  重放；恢复期状态栏 `ime=recovering`。
- 超时：单次 `key` 回复 50 ms 未到标记 `ime=slow`，**不回落为英文**；词典缺失报
  `ime=no-dict`，只允许用户显式切 ASCII 直通（§7 的失败语义表）。

### 10.4 为什么抽象层不单独成 crate

`boot-tui` 已持有键盘与页面状态机；把 trait 与四个适配器放 `boot-tui/src/input/`，
热路径少一次进程边界。`boot-ime` 与 `asr-gateway` 独立成进程的理由是它们独占设备或持有
大状态（词典、模型），不是因为「输入抽象」需要隔离。新增第六个 crate 会违反
`distro/lockfile/lang-policy.yaml` 的「每个 crate 必须有当期消费者」判据。

### 10.5 验收

| # | 判据 | 方式 |
|---|---|---|
| V-10 | 键盘不产文本直通（显式 ASCII 模式除外） | 关闭 IME 后键入中文键码，不得出现字母 |
| V-11 | IME 崩溃不丢未提交按键 | `kill -9` 后 200 ms 内 `ime=recovering` 并恢复 preedit |
| V-12 | 语音无麦克风时显式不可用 | 撤销权限后 `asr=unavailable`，不得静默转键盘替代 |
| V-13 | API 背压不丢不重 | 并发 64 条，超限返回 `-EAGAIN`，trace_id 去重后计数相等 |
| V-14 | 四通道 schema 一致 | 同语义四路输入，`airy.intent/1` diff 仅 `source` 与 `trace_id` |

---

## 11. 不做什么

| 不做 | 理由 |
|---|---|
| 为输入新增 syscall | 违反「只消费既有 ABI」；接入是用户态问题 |
| 定义第二套 Intent | 两条通路共用一个契约层 |
| 在 TUI 内嵌 agentrt | 0.0.5 裁定：独立 Rust IME 与会话层，不链接 agentrt |
| 用 IBus 承担 boot TUI 输入 | IBus 依赖 GUI 会话与 D-Bus，开机 TUI 不可用 |
| 把审批做成布尔值 | 无法审计、无法限域、无法过期 |
| 让接入侧决定策略 | 判定权在安全通道，接入侧只做预检 |
| 把语音结果直接当文本指令执行 | 识别不可避免有误；必须经可编辑缓冲区确认 |
| 为输入抽象新增 crate | 会话热路径不需要多一层进程边界；四个适配器在 `boot-tui` 内即可 |
