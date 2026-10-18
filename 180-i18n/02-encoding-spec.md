Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agent-linux（AirymaxOS）编码规范详细设计
> **文档定位**：agent-linux（AirymaxOS，极境智能体操作系统）字符编码规范与 UTF-8 处理详细设计\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[agent-linux 设计文档](README.md)\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## 1. 设计目标与范围

### 1.1 设计目标

agent-linux（AirymaxOS）编码规范旨在为内核态与用户态代码统一字符编码处理标准。本设计聚焦三大目标：

1. **内核态 UTF-8 一等公民支持**：内核态禁用 `wchar_t`，所有字符串以 UTF-8 字节序列表达，提供 UTF-8 序列判定、解码与整串校验原语（落地位于 `kernel/corekern/charset/`）
2. **用户态 Unicode 规范一致性**：12 daemon 与 9 协议适配层统一使用 UTF-8 编码，禁止 GB2312/GBK/Shift-JIS 等非 Unicode 编码流入系统
3. **Agent 多语言 prompt 编码契约**：Agent 提示词与 LLM 推理输入/输出严格 UTF-8，Token 计数按 Unicode 码点（而非字节）计算

### 1.2 适用范围

- Linux 6.6 内核基线内核态 UTF-8 字符串处理（`include/airymax/airy_utf8.h`、`kernel/corekern/charset/airy_utf8.c`）
- 用户态 glibc / musl 的 UTF-8 处理（iconv / mbsrtowcs 等）
- AgentsIPC 消息体的字符串负载编码（消息头 `trace_id` 为 `__u64` 数值，见 §5.1）
- Agent 系统提示词、用户输入、LLM 输出的统一 UTF-8 处理
- 文档源码注释编码规范（所有 .md / .c / .h / .py 文件 UTF-8）

### 1.3 术语规范

本设计严格遵守 agent-linux 术语规范：agentrt（用户态）称为**微核心**（micro-core），agent-linux（OS 发行版）称为**微内核**（micro-kernel）。编码契约在 agentrt 与 agent-linux 之间属于 IRON-9 v3 [SC] 共享契约层。

### 1.4 编码规范矩阵

| 范围 | 编码 | 字节序 | BOM | 备注 |
|------|------|--------|-----|------|
| 内核源码 | UTF-8 | N/A | 禁止 | .c/.h 文件 |
| 用户态源码 | UTF-8 | N/A | 禁止 | .py/.rs/.go 文件 |
| 文档 | UTF-8 | N/A | 禁止 | .md 文件 |
| IPC 消息 | UTF-8 | LE/BE 无关 | 禁止 | 字节流 |
| JSON-RPC | UTF-8 | N/A | 禁止 | RFC 8259 |
| 日志 | UTF-8 | N/A | 禁止 | syslog / journald |
| 配置文件 | UTF-8 | N/A | 禁止 | .yaml / .conf / .ini |
| 内核日志 | UTF-8 | N/A | 禁止 | printk 输出 |

---

## 2. 内核态 UTF-8 处理规范

### 2.1 禁止 wchar_t 规则

agent-linux 内核态严格禁止使用 `wchar_t` 类型，原因有三：

1. **Linux 6.6 内核不提供 wchar_t 运行时支持**：内核态没有 `wprintf`、`wcslen`、`wcscpy` 等 C 标准库宽字符函数
2. **wchar_t 大小不可移植**：Linux 上 4 字节，Windows 上 2 字节，会导致跨平台数据不兼容
3. **UTF-8 字节序列更紧凑**：CJK 字符在 UTF-8 中占 3 字节，比 wchar_t 的 4 字节省 25% 内存

替代方案：所有内核字符串以 `char *` 表达 UTF-8 字节序列，长度以字节数（`size_t`）记录。

### 2.2 UTF-8 编码规则速查

UTF-8 编码规则（RFC 3629）：

| Unicode 码点范围 | UTF-8 字节序列 | 字节数 |
|------------------|---------------|--------|
| U+0000 ~ U+007F | 0xxxxxxx | 1 |
| U+0080 ~ U+07FF | 110xxxxx 10xxxxxx | 2 |
| U+0800 ~ U+FFFF | 1110xxxx 10xxxxxx 10xxxxxx | 3 |
| U+10000 ~ U+10FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx | 4 |

特殊范围禁止：
- U+D800 ~ U+DFFF：UTF-16 代理对，UTF-8 中非法
- U+FDD0 ~ U+FDEF：非字符，禁止
- U+nFFFE / U+nFFFF（n 为 0-16，即每个平面末尾两个码点）：非字符，禁止
- U+FEFF：BOM，任意位置禁止（不限于串首）
- 过长编码：同一码点的最短编码之外的形式一律拒绝（含以 0xC0/0xC1 为首字节的 2 字节形式）

### 2.3 内核 UTF-8 原语库

内核侧原语已落地，SSoT 为以下三处：

| 文件 | 角色 |
|------|------|
| `kernel/include/airymax/airy_utf8.h` | 公共头，仅暴露三个原语 |
| `kernel/kernel/corekern/charset/airy_utf8.c` | 判定实现 |
| `kernel/kernel/corekern/charset/airy_utf8_test.c` | KUnit 边界矩阵 |

编译由 `CONFIG_AIRY_COREKERN` 门控（`kernel/kernel/corekern/Kbuild` 挂载 `charset/`），不为此新增 Kconfig。API 只保留消费者真实需要的三个原语：

| 原语 | 形态 | 职责 |
|------|------|------|
| `airy_u8_len()` | `static inline unsigned int` | 由首字节判定序列长度，非法首字节返回 0 |
| `airy_u8_dec()` | `int` | 解码并判定单个序列，返回消耗字节数或负 `AIRY_EENC_*` |
| `airy_u8_scan()` | `int` | 扫描整串并以码点计数，空串合法 |

```c
/* kernel/include/airymax/airy_utf8.h —— 已落地 */
#define AIRY_U8_MAX	4
#define AIRY_U8_MAXCP	0x10ffffU

typedef u32 airy_ucp_t;

static inline unsigned int airy_u8_len(u8 lead);

__must_check int airy_u8_dec(const u8 *s, size_t len, airy_ucp_t *cp);
__must_check int airy_u8_scan(const u8 *s, size_t len);
```

判定顺序固定：首字节结构 → 续字节数量 → 续字节掩码 → 过长编码 → 代理对 → 码点上界 → 非字符 → BOM。`airy_u8_dec()` 返回负值时**不写** `*cp`，调用方无需预置输出参数；`airy_u8_len()` 只做结构判定、不校验续字节，纯内联且无符号。

不提供编码方向与子串截取原语：内核侧当前没有生成或裁剪 UTF-8 的调用点，凭空实现只会成为无消费者死代码。后续若出现真实调用点，按本文件 §7 的错误码契约新增原语，不在调用点就地展开判定。

当前唯一消费者是 syscall 550（`airy_sys_sched_ctl`）。`kernel/kernel/syscalls/airy_sched.c` 中 `airy_sched_arg()` 的处理链为：`strndup_user()` 有界复制（上限 `AIRY_SCHED_ARG_MAX`）→ `airy_sched_chk()` 校验 → `kfree()`。校验策略与复制分离，使策略本身可在不持有用户指针的情况下被 KUnit 覆盖。

### 2.4 安全字符串操作规范

agent-linux 内核态严格禁止 `strcpy`、`strcat`、`sprintf`、`gets` 等不安全函数，统一使用 `strscpy`、`strscpy_pad`、`scnprintf`：

```c
/* 安全字符串操作示例（K&R 风格，Tab=8） */
#include <linux/string.h>
#include <linux/slab.h>
#include <airymax/airy_utf8.h>

/* 错误示例：禁止使用 strcpy（缓冲区溢出风险） */
/* int bad_example(char *dst) { strcpy(dst, "agent-linux"); } */

/* 正确示例：使用 strscpy（自动截断并保证 NUL 终止） */
int good_example(char *dst, size_t dst_size)
{
	int ret;

	ret = strscpy(dst, "agent-linux", dst_size);
	if (ret < 0)
		return -E2BIG;
	return 0;
}
```

---

## 3. 用户态 Unicode 规范

### 3.1 用户态字符串处理

用户态 daemon 使用 glibc 的 mbsrtowcs / iconv 进行编码转换，所有外源输入必须经过编码校验：

```c
/* services/common/airy_unicode.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <wchar.h>
#include <locale.h>
#include <iconv.h>

/* 校验输入字符串为合法 UTF-8（用户态版本） */
int airy_user_utf8_validate(const char *s, size_t len)
{
	mbstate_t state;
	wchar_t wc;
	size_t consumed = 0;
	size_t n;

	memset(&state, 0, sizeof(state));
	while (consumed < len) {
		n = mbrtowc(&wc, s + consumed, len - consumed, &state);
		if (n == (size_t)-1)
			return -EILSEQ;       /* 非法字节序列 */
		if (n == (size_t)-2)
			return -EILSEQ;       /* 不完整序列（len 已耗尽） */
		if (n == 0)
			break;
		consumed += n;
	}
	return 0;
}

/* 从其他编码转换为 UTF-8（如 GB18030 → UTF-8） */
int airy_user_convert_to_utf8(const char *from_codec,
				const char *in, size_t in_len,
				char *out, size_t out_size)
{
	iconv_t cd;
	char *inbuf = (char *)in;
	char *outbuf = out;
	size_t inbytesleft = in_len;
	size_t outbytesleft = out_size;
	int ret;

	cd = iconv_open("UTF-8", from_codec);
	if (cd == (iconv_t)-1)
		return -EINVAL;

	ret = iconv(cd, &inbuf, &inbytesleft, &outbuf, &outbytesleft);
	if (ret == (size_t)-1) {
		int err = errno;
		iconv_close(cd);
		switch (err) {
		case EILSEQ:
			return -EILSEQ;
		case EINVAL:
			return -EINVAL;
		case E2BIG:
			return -E2BIG;
		default:
			return -EINVAL;
		}
	}

	/* NUL 终止 */
	if (outbytesleft == 0) {
		iconv_close(cd);
		return -E2BIG;
	}
	*outbuf = '\0';
	iconv_close(cd);
	return out_size - outbytesleft;
}
```

### 3.2 JSON-RPC 编码契约

所有 JSON-RPC 请求/响应必须使用 UTF-8 编码（RFC 8259 强制要求）：

```c
/* JSON-RPC 响应构造（确保 UTF-8） */
#include <jansson.h>
#include <string.h>

int airy_jsonrpc_build_response(int code, const char *message,
				   const char *locale, char *out, size_t out_size)
{
	json_t *root, *error;
	char *json_str;
	int ret;

	/* 校验 message 为合法 UTF-8 */
	if (airy_user_utf8_validate(message, strlen(message)) < 0)
		return -EILSEQ;

	root = json_object();
	error = json_object();
	json_object_set_new(error, "code", json_integer(code));
	json_object_set_new(error, "message", json_string(message));
	json_object_set_new(error, "locale", json_string(locale));
	json_object_set_new(root, "error", error);

	json_str = json_dumps(root, JSON_ENSURE_ASCII | JSON_COMPACT);
	if (!json_str) {
		ret = -ENOMEM;
		goto out_free_root;
	}

	ret = strscpy(out, json_str, out_size);
	if (ret < 0) {
		ret = -E2BIG;
		goto out_free_json;
	}

out_free_json:
	free(json_str);
out_free_root:
	json_decref(root);
	return ret;
}
```

---

## 4. Agent 多语言 prompt 编码

### 4.1 prompt 编码契约

Agent 系统提示词（system prompt）与用户输入必须为 UTF-8 编码。Token 计数按 Unicode 码点（而非字节）计算，确保 CJK 用户与英文用户公平。

```python
# agentrt SDK 示例：多语言 prompt UTF-8 处理
from agentrt import CognitionClient
from agentrt.encoding import validate_utf8, count_codepoints

client = CognitionClient(language="zh-CN")

# 中文 prompt（UTF-8 编码，自动校验）
prompt = "你是一个智能助手，请用中文回答用户问题。"
if not validate_utf8(prompt.encode('utf-8')):
    raise ValueError("prompt 不是合法 UTF-8")

# Token 计数按码点（3 字节 CJK 字符算 1 码点）
token_count = count_codepoints(prompt)
print(f"prompt 码点数: {token_count}")

# 提交推理
result = client.complete(
    system_prompt=prompt,
    user_input="解释什么是微内核"
)
```

### 4.2 Token 计数规范

agent-linux 的 Token 计数遵循"按 Unicode 码点计数"原则：

| 字符类型 | 字节数 | 码点数 | Token 计数 |
|----------|--------|--------|------------|
| ASCII（A-Z, 0-9） | 1 | 1 | 1 |
| 拉丁扩展（é, ñ） | 2 | 1 | 1 |
| CJK（中、日、韩） | 3 | 1 | 1 |
| Emoji（😀） | 4 | 1 | 1 |

```c
/* Token 计数按码点：校验与计数复用 airy_u8_scan() 的同一次扫描 */
#include <airymax/airy_utf8.h>

int airy_token_count(const char *prompt, size_t byte_len)
{
	return airy_u8_scan((const u8 *)prompt, byte_len);
}
```

---

## 5. 内核态 UTF-8 字符串处理完整示例

### 5.1 IPC 消息头 TraceID 数值处理

> **修正说明**：[SC] `ipc.h` 中 `trace_id` 为 **`__u64`（offset 8，8 字节）**，**不是** UTF-8 字符串，也不存在 16B UUID 布局。旧文档将其当作定长字符串处理（strlen/memcpy/UTF-8 校验）与头结构不符，已重写为纯数值读写；字符串型追踪信息应放入 payload。

```c
/* IPC 消息头中 trace_id 字段为 __u64（offset 8，8 字节，见 [SC] ipc.h），
 * 非 UTF-8 字符串；分布式追踪 ID 以数值形式写入。 */
#include <linux/types.h>
#include <linux/airymax/ipc.h>

/* 设置 IPC 消息头的 trace_id（__u64 数值写入） */
static inline void airy_ipc_set_trace_id(struct airy_ipc_msg_hdr *hdr,
					  __u64 trace_id)
{
	hdr->trace_id = trace_id;
}

/* 读取 IPC 消息头的 trace_id（__u64） */
static inline __u64 airy_ipc_get_trace_id(const struct airy_ipc_msg_hdr *hdr)
{
	return hdr->trace_id;
}
```

### 5.2 printk 多语言消息的 UTF-8 安全长度限制

printk 单条记录上限为 1024 字节（`PRINTK_SAFE_LOG_BUF_LEN`）。多语言消息在进入 printk 之前必须先通过 `airy_u8_scan()` 校验，否则截断位置可能落在多字节序列中间，日志里出现半个字符：

```c
/* 先校验后输出：截断只允许发生在已校验的序列边界 */
#include <airymax/airy_utf8.h>

int airy_log_msg(const char *level, const char *msg, size_t len)
{
	int ncp;

	ncp = airy_u8_scan(msg, len);
	if (ncp < 0)
		return ncp;

	printk("%s%.*s\n", level, (int)len, msg);
	return ncp;
}
```

内核侧不提供截断封装：printk 调用点均使用常量格式串，超长记录由 printk 自身截断；需要控制长度时应在构造消息阶段按码点预算，而不是在输出路径反向裁剪半条序列。校验与码点计数由 `airy_u8_scan()` 一次扫描完成，不需要额外的 `strlen` 预扫描。

---

## 6. 性能考量

### 6.1 UTF-8 处理开销

| 操作 | 开销（ns） | 备注 |
|------|-----------|------|
| `airy_u8_scan()` 校验并计数（1KB） | 200-500 | 线性扫描，一次完成 |
| `airy_u8_dec()` 解码（单序列） | 5-15 | 分支判断 |
| iconv 转换（GB18030→UTF-8，1KB） | 5,000-10,000 | 库调用 |

### 6.2 优化策略

1. **缓存校验结果**：daemon 启动时一次性校验配置文件，运行时不再校验
2. **SIMD 加速**：用户态可使用 SIMD 指令加速 UTF-8 校验（如 `simdutf` 库）
3. **批量处理**：IPC 批量消息的 UTF-8 校验合并为一次扫描

---

## 7. 错误码体系对接

编码错误码已在 SSoT `error.h` 的 Encoding 子空间注册，幅值 241-246（返回负空间 `[-241, -246]`）：

| 错误码 | 幅值 | 触发条件 |
|--------|------|---------|
| `AIRY_EENC_INVALID` | 241 | 非法首字节、续字节缺失或掩码不符、序列被截断 |
| `AIRY_EENC_OVERLONG` | 242 | 过长编码（含以 0xC0/0xC1 为首字节的 2 字节形式） |
| `AIRY_EENC_SURROGATE` | 243 | UTF-16 代理对 U+D800~U+DFFF |
| `AIRY_EENC_NONCHAR` | 244 | Unicode 非字符：U+FDD0~U+FDEF、每平面末尾 U+nFFFE/U+nFFFF |
| `AIRY_EENC_RANGE` | 245 | 码点超出 U+10FFFF |
| `AIRY_EENC_BOM` | 246 | U+FEFF，任意位置（不限于串首） |

宏为正数幅值，调用方返回 `-AIRY_EENC_*`。子空间 `[247, 260]` 保留给编码相关扩展，`[261, 300]` 为跨子系统预留；再次分配前必须同步更新 `30-interfaces/08-sc-error-contract.md` 与物理头文件。

集中错误处理示例（与 `kernel/kernel/syscalls/airy_sched.c` 的实际形态一致）：

```c
#include <airymax/airy_utf8.h>
#include <linux/airymax/error.h>

__must_check int airy_sched_chk(const char *s)
{
	int ret;

	if (!s)
		return -AIRY_EINVAL;

	ret = airy_u8_scan(s, strlen(s));
	if (ret < 0)
		return ret;

	return AIRY_EOK;
}
```

---

## 8. 五维原则映射

| 原则 | 在本设计的体现 |
|------|---------------|
| **K-2 接口契约化** | UTF-8 编码作为 [SC] 共享契约 |
| **E-1 安全内生** | 禁止 strcpy/wchar_t，从源头消除安全风险 |
| **E-4 跨平台一致性** | UTF-8 跨架构一致，无字节序问题 |
| **E-5 命名语义化** | `airy_u8_*` 函数名即语义（len/dec/scan） |
| **E-6 错误可追溯** | `AIRY_EENC_*`（241-246，已注册于 SSoT error.h）区分结构错误与策略拒绝 |
| **A-3 人文关怀** | CJK 用户与英文用户公平的 Token 计数 |

---

## 9. IRON-9 v3 同源映射

| 组件 | agent-linux（[SC]） | agentrt（[SC]） | 共享 |
|------|------------------------|------------------|------|
| UTF-8 头文件 | `include/airymax/airy_utf8.h` | 同源镜像 | [SC] 定义，SSoT 在 agent-linux 内核树 |
| 错误码 | `AIRY_EENC_*` 241-246 | 同左 | 已注册于 SSoT `error.h` |
| Token 计数规则 | 按码点 | 按码点 | 共享契约 |
| 编码规范文档 | 本文件 | 同源镜像 | [SC] |

---

## 10. 相关文档

- `180-i18n/README.md`（国际化主索引）
- `180-i18n/01-locale-design.md`（Locale 设计）
- `180-i18n/04-cjk-support.md`（CJK 支持设计）
- `50-engineering-standards/01-coding-standards.md`（编码规范）
- `170-performance/03-ipc-performance.md`（IPC 消息头）

---

## 11. 参考材料

- RFC 3629（UTF-8 编码规范）
- RFC 8259（JSON 规范，强制 UTF-8）
- Linux 6.6 `lib/charset.c`
- Linux 6.6 `include/linux/string.h`
- Unicode 标准（Unicode 15.1）
- POSIX locale 与 iconv 规范

---

> **文档结束** | agent-linux（AirymaxOS）编码规范详细设计 
