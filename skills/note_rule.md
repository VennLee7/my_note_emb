---
name: programmer-knowledge-note-rule
description: 当 Codex 需要把真实开发过程、AI 对话、代码、Git diff、日志、构建错误、Crash、调试证据、Datasheet 或工程上下文沉淀为程序员长期知识笔记时使用。本 Skill 负责在 note/ 中创建或更新结构化知识笔记，在 history-session/ 中保存有价值的调查历史，并处理查重、链接、粒度、事实可信度、不确定信息和敏感信息脱敏。
---

# Programmer Knowledge Note Rule

本 Skill 用于维护程序员工程知识库。目标不是保存 AI 聊天记录，而是从真实开发活动中持续提炼可复用知识。

核心区分：

```text
History 保存：我是怎么走到这里的。
Note 保存：以后真正值得知道什么。
Debug Case 保存：我是如何证明根因的。
Concept Note 保存：这个东西本质上是什么。
Implementation Note 保存：它在工程里应该怎么落地。
System Note 保存：这些组件怎样组成一个系统。
```

## 1. 仓库结构

当前仓库结构应保持轻量：

```text
my_note_emb/
|-- README.md
|-- note/
|-- history-session/
`-- skills/
    |-- create_rule.md
    `-- note_rule.md
```

目录职责：

- `note/`：结构化、可复用的工程知识。
- `history-session/`：有价值的调查过程、AI 对话、原始上下文。
- `skills/`：Agent 规则文件，不放普通笔记。

不要为了形式大规模重构目录。只有当笔记数量和检索需求真正出现时，才在 `note/` 下创建浅层领域目录，例如：

```text
note/embedded/
note/os/
note/linux/
note/cpp/
note/python/
note/rust/
note/web/
note/debug/
note/build-system/
note/network/
note/devops/
note/architecture/
```

避免过早建立深层分类。

## 2. 触发条件

当用户要求以下任务时使用本 Skill：

- 整理、保存、归档、沉淀技术笔记；
- 保存有价值的 AI 对话或调试历史；
- 从代码、Git diff、日志、Crash、构建错误、寄存器、波形、抓包、Datasheet、Device Tree、CMake、Kconfig、README 中提炼知识；
- 把已解决的问题整理成 Debug Case；
- 把概念、系统、实现方案整理成长期可复用的程序员知识；
- 跨项目、跨语言、跨平台维护该知识库。

不要为没有长期工程价值的临时聊天创建知识笔记。

## 3. 写文件规则

本 Skill 的知识模型和笔记格式是跨项目、跨语言、跨平台的；但当前 Writer Backend 明确是：

```text
Windows PowerShell
```

当前不要为了 Linux/macOS 重写 Writer Backend。未来如有需要，再扩展为：

```text
Writer Backend
|-- Windows PowerShell
`-- POSIX Shell
```

创建或编辑本仓库文件时，必须遵循 `skills/create_rule.md`，防止文件被异常加密或损坏：

1. 先 `Resolve-Path` 目标目录。
2. 用 `New-Item -ItemType Directory -Path <dir> -Force` 创建目录。
3. 用 `New-Item -ItemType File -Path <file> -Force` 创建文件。
4. 用终端 `Set-Content -Path <file> -Encoding UTF8` 写入。
5. 不要用 `apply_patch`、编辑器保存、Python 直接写文件或其他非终端写入路径。
6. 写入后用 `Get-Content -LiteralPath <file> -TotalCount 20` 验证可读。

## 4. 定位笔记仓库

在任意工程中被调用时，先定位笔记仓库：

1. 优先使用用户明确给出的路径。
2. 如果当前 workspace 内存在 `my_note_emb/`，使用它。
3. 否则向上或在邻近 workspace 查找同时包含以下路径的目录：
   - `note/`
   - `history-session/`
   - `skills/note_rule.md`
4. 如果存在多个候选，优先选择用户提到的路径；无法判断时简短询问。
5. 除非用户明确要求，不要把知识笔记写入当前代码工程。

## 5. 判断当前工程

写笔记前，尽量从可访问上下文中判断当前工程：

- Git 根目录、仓库名、remote、branch、status、diff；
- 构建文件：`CMakeLists.txt`、`Kconfig`、`package.json`、`pyproject.toml`、`Cargo.toml`、`Makefile`；
- OS、SDK、Compiler、Framework、Library、MCU、Board、Hardware revision；
- 用户显式给出的环境信息。

只能记录实际看到或用户明确提供的信息。未知字段省略或标记 `unknown`，不要编造版本、commit、日志、硬件信息或历史对话。

## 6. 读取上下文

可用输入包括：

- 当前用户请求和可见 AI 对话上下文；
- 当前工程代码、README、配置和文档；
- Git diff、本地修改、构建错误；
- 日志、Crash dump、寄存器、波形、抓包、终端输出；
- Datasheet、原理图、Device Tree、Kconfig、CMake；
- 用户明确给出的分析结论；
- 当前会话中完成的推理、实验和结果。

没有看到的信息不能假装看到。无法访问完整历史对话时，不得凭空补造。若无真实 History 可保存，可只生成 Note，并在 frontmatter 中标记：

```yaml
history_available: false
```

## 7. 写入前判断

创建或更新前，先内部判断：

```text
这是否值得长期保存？
这是知识，还是临时状态？
这是已有知识的补充，还是新知识单元？
属于 concept、system、implementation 还是 debug-case？
哪些是事实？哪些是假设？
证据来自日志、源码、实验、文档还是用户结论？
是否已有相关 Note？
能否从具体问题抽象出可复用知识？
```

如果只有过程价值，优先保存 History。若存在可复用知识，创建或更新 Note。

## 8. 笔记类型

每篇 Note 必须有一个 `primary_type`，并可有多个 `type`。

支持四类主类型：

- `concept`：技术概念或抽象，例如 DMA、Mutex、Cache、MMU、Interrupt、ABI、Device Tree、SPI。
- `system`：完整系统或子系统，例如 Linux 内存管理、Zephyr Device Model、USB 架构、BLE 协议栈、Boot Flow、编译链接系统。
- `implementation`：具体工程实现，例如 Zephyr SPI Driver、ADC + DMA、USB HID Mouse、Bootloader Upgrade、GPIO Interrupt、Ring Buffer。
- `debug-case`：真实工程问题，例如 HardFault、I2C NACK、CMake BOARD_ROOT 错误、编码器无波形、VPN 导致 LAN 无法访问、BLE 断连、内存越界。

允许混合类型：

```yaml
primary_type: debug-case
type:
  - debug-case
  - implementation
```

`primary_type` 决定正文主结构。

## 9. 查重、更新、合并、拆分

创建新 Note 前必须搜索已有笔记：

- 搜索文件名、title、`note_id`、tags、领域词；
- 同时搜索英文 slug 和可能的中文标题；
- 优先使用 `rg`。

决策规则：

- 新信息只是补充已有概念：更新已有 Note。
- 新信息是某概念的具体实现：创建 Implementation Note，并链接 Concept Note。
- 新信息是真实故障调查：创建 Debug Case，并链接相关概念和实现。
- 两篇 Note 描述同一稳定知识单元：合并到更强的一篇，必要时留下链接。
- 不要创建 `dma-2.md`、`dma-new.md`、`dma-final.md` 之类重复文件。

粒度原则：一篇 Note 对应一个稳定、可独立理解、可被引用的知识单元。

避免过大：

```text
embedded.md
all-debug-notes.md
linux.md
```

避免过碎：

```text
gpio-high-level.md
gpio-low-level.md
gpio-input-one-line.md
```

推荐粒度：

```text
gpio.md
gpio-interrupt.md
gpio-debounce.md
dma-cache-coherency.md
```

## 10. 命名

文件名和 `note_id` 使用稳定英文 slug。标题可以中文。

Knowledge Note 示例：

```text
optical-encoder-power-issue.md
zephyr-device-model.md
dma-cache-coherency.md
virtual-memory.md
```

History 文件名必须包含日期和时间，避免同一天多次讨论相同主题时发生碰撞：

```text
YYYY-MM-DD_HHmm_slug.md
```

示例：

```text
2026-08-12_1409_optical-encoder-power-issue.md
2026-08-12_1832_optical-encoder-power-issue.md
```

对应 `session_id`：

```yaml
session_id: 2026-08-12-1409-optical-encoder-power-issue
```

如果运行环境能够稳定获取秒，也可以使用：

```text
YYYY-MM-DD_HHmmss_slug.md
```

目标是稳定、可排序、避免同日同主题碰撞。不要使用 `-2`、`-new`、`-final` 之类人工后缀。

避免 `issue.md`、`today.md`、`new-note.md`、`summary.md` 等无意义名称。

## 11. Frontmatter

Knowledge Note 推荐包含 `aliases` 和 `status`。

`aliases` 用于兼顾稳定英文 slug、中文标题、英文全称、缩写和常见别名，方便检索和 Obsidian 双链。不要强制每篇 Note 都写大量 aliases；只有存在缩写、中英文名称或常见别名时才添加。

`status` 推荐值：

```text
draft
investigating
confirmed
resolved
deprecated
```

语义：

- `draft`：内容尚未完善；
- `investigating`：Debug 或技术结论仍在调查；
- `confirmed`：知识或结论已有充分依据；
- `resolved`：Debug Case 已解决并完成验证；
- `deprecated`：内容因版本、架构或新知识已经过时。

Knowledge Note 示例：

```yaml
---
note_id: optical-encoder-power-issue
title: 光电编码器供电导致 AB 相无输出问题
aliases:
  - Optical Encoder Power Issue

primary_type: debug-case
type:
  - debug-case
  - implementation

status: resolved

domain:
  - embedded
  - hardware-debug

project: xs_nrf_mouse_firmware

created: 2026-08-12
updated: 2026-08-12

history:
  - ../../history-session/2026-08-12_1409_optical-encoder-power-issue.md

tags:
  - encoder
  - gpio
  - power
  - debugging

history_available: true
---
```

上例假设 Note 位于 `note/embedded/optical-encoder-power-issue.md`。如果 Note 位于 `note/optical-encoder-power-issue.md`，则同一个 History 的相对路径应为：

```yaml
history:
  - ../history-session/2026-08-12_1409_optical-encoder-power-issue.md
```

不要写死 History 相对路径。Note 与 History 建立链接时，必须根据两个文件的真实位置动态计算相对路径。

History 示例：

```yaml
---
session_id: 2026-08-12-1409-optical-encoder-power-issue
project: xs_nrf_mouse_firmware
date: 2026-08-12

note_ids:
  - optical-encoder-power-issue

notes:
  - ../note/embedded/optical-encoder-power-issue.md
---
```

正式支持 N History ↔ N Note：

- 一个 History 可以对应 0、1 或多个 Note；
- 一个 Note 可以对应 0、1 或多个 History；
- Knowledge Note 使用 `history` 数组链接相关 History；
- History 使用 `note_ids` 和 `notes` 数组链接相关 Note；
- 双方建立链接时必须去重；
- 不要额外创建第三份人工映射文件。

一次 Session 沉淀多篇 Note 时，History frontmatter 示例：

```yaml
---
session_id: 2026-08-12-1409-optical-encoder-debug
project: xs_nrf_mouse_firmware
date: 2026-08-12

note_ids:
  - quadrature-encoder
  - gpio-power-control
  - optical-encoder-power-issue

notes:
  - ../note/embedded/quadrature-encoder.md
  - ../note/embedded/gpio-power-control.md
  - ../note/debug/optical-encoder-power-issue.md
---
```

## 12. History 规则

存在有价值的真实对话或调查过程时，创建 History Session。

History 保留：

- 最初问题、环境、原始日志、用户证据；
- 关键代码片段、排查过程、假设、被否定的假设；
- 实验、结果、AI 分析、用户反馈；
- 最终结论和尚未解决的问题。

History 中的 AI Analysis 指可公开、可验证、对工程知识有价值的分析摘要，而不是模型内部隐藏 Chain-of-Thought。

History 允许保存：

- 用户可见的 AI 分析；
- 判断依据和结论摘要；
- 实验设计和 Debug 路径；
- 工具结果；
- Evidence -> Conclusion 的可公开解释；
- 当前会话中已经明确表达出来的 reasoning summary。

History 不要：

- 要求保存模型隐藏 Chain-of-Thought；
- 尝试恢复隐藏内部推理；
- 伪造不存在的详细思维过程。

History 偏向保存问题演化过程和原始上下文，不需要像 Note 一样高度整理。

推荐结构：

```markdown
# Title

## Context

## Initial Problem

## Environment

## Conversation / Investigation

### Step 1

User:
AI:
Evidence:
Result:

## Important Raw Data

## Final State

## Confirmed Conclusions

## Unresolved Questions

## Related Notes
```

如果无法获得真实对话，明确说明或不创建 History。不要伪造对话。

## 13. History Immutability & Note Evolution

History Session 表示某一次真实开发、讨论、调查在当时发生了什么。

History 一旦完成保存，原则上视为历史记录，不应该因为未来认知发生变化而重新篡改过去的调查过程。

后续出现以下情况时：

- 新证据；
- 新实验；
- 新源码；
- 新版本；
- 原结论错误；
- 原假设被推翻；

应当：

1. 新建一个新的 History Session；
2. 更新对应 Knowledge Note；
3. 将新的 History Session 追加到 Note 的 `history`；
4. 必要时在新 History 或 Note 中说明旧结论被修正。

禁止为了让旧 History 与当前最佳结论一致，而重写旧调查过程。

Knowledge Note 表示当前已经获得的最佳、最可靠、最可复用知识状态，因此允许持续演化。

关系应理解为：

```text
History A --\
History B ----> Knowledge Note(Current)
History C --/
```

History 保存时间线。Note 保存当前知识。

## 14. Existing Note Update Safety

更新已有 Note 前：

1. 必须完整读取现有 Note；
2. 理解现有 frontmatter 和正文结构；
3. 默认进行增量合并；
4. 保留与本次修改无关且仍然有效的内容；
5. `created` 日期保持原值；
6. 只更新 `updated`；
7. `history` 只允许追加、去重或明确修复失效链接，不得无故删除；
8. 已有 `references`、`aliases`、`related`、`tags` 等不能因为当前上下文没提到就自动删除；
9. 如果新证据推翻旧知识，应显式修改对应章节；
10. 必要时说明 `Previous Understanding`、`New Evidence`、`Updated Conclusion`；
11. 不允许因为当前 Session 信息比原 Note 少，就生成一个更短版本覆盖原 Note。

原则：

```text
Update 是 knowledge merge，不是 regenerate from current context。
```

## 15. Note 写作风格

Note 应像工程师长期维护的技术知识库：

- 客观、简洁、高信息密度；
- 强调底层机制、抽象关系、工程实践和故障定位；
- 准确优先于流畅；
- 明确表达不确定；
- 不写成普通博客教程；
- 不写成 AI 问答总结。

避免：

```text
当然可以
这是一个非常好的问题
下面我们来看
希望对你有帮助
```

用户使用中文时，正文优先中文；技术标识符、API、命令、文件名保持原文。

## 16. 核心思维模型

程序员笔记优先建立：

```text
WHY -> HOW -> DO
```

尽量回答：

1. 它解决什么问题？
2. 它本质是什么？
3. 内部怎么工作？
4. 核心概念和对象是什么？
5. 什么时候应该使用？
6. 最小如何实现？
7. 边界、限制和代价是什么？
8. 出问题应该怎么调试？

模板是组织知识的工具，不是必须填满的表格。

## 17. Concept Note 模板

```markdown
# Title

## TL;DR

## Problem / Motivation

## Definition

## Essence

## Mental Model

## Core Concepts

## Principle

## Application Scenarios

## Minimal Example

## Boundary

## Trade-offs

## Failure Model / Debug

## Related Knowledge

## References
```

要求：先解释为什么存在，再定义；区分 Essence 和 Principle；必要时用 ASCII 图、分层图、状态机、调用链、数据流或控制流。

## 18. System Note 模板

```markdown
# Title

## TL;DR

## Problem / Motivation

## Scope

## Architecture

## Components and Responsibilities

## Dependency Model

## Data Flow

## Control Flow

## Lifecycle

## Interfaces

## Boundary

## Failure Model

## Debug Methodology

## Related Knowledge

## References
```

重点是组件如何组成系统。

## 19. Implementation Note 模板

```markdown
# Title

## TL;DR

## Goal

## Environment

## Prerequisites

## Minimal Working Example

## API / Interface

## Configuration

## Implementation

## Initialization Order

## Data Flow

## Error Handling

## Concurrency

## Memory / Performance

## Verification

## Pitfalls

## Related Knowledge

## References
```

必须区分：

```text
能运行 != 适合工程使用
```

## 20. Debug Case 模板

```markdown
# Title

## TL;DR

## Environment

## Symptom

## Expected Behavior

## Observed

## Evidence

## Investigation Timeline

## Hypotheses

## Experiments

## Root Cause

## Fix

## Verification

## Why the Fix Works

## Prevention

## Generalized Knowledge

## Related

## References
```

Debug Case 必须保存证明路径：

```text
Symptom -> Evidence -> Hypothesis -> Experiment -> Result -> Root Cause -> Fix -> Verification -> Prevention
```

若根因未完全证明，写 `Current Leading Hypothesis`，不要写 `Root Cause`；frontmatter 的 `status` 不应标为 `resolved`，应使用 `investigating` 或其他更准确状态。

## 21. 事实可信度

不要把推测写成事实。区分：

- `Confirmed`：实际观察或证明；
- `Evidence`：日志、波形、Dump、源码、寄存器、Git diff、Datasheet、实验；
- `Inferred`：由证据推导但未直接观察；
- `Hypothesis`：待验证假设；
- `Experience`：经验判断；
- `Vendor Documentation`：外部文档描述；
- `Source Code`：源码实现行为；
- `Experiment`：实验结果。

当文档和源码行为冲突时，同时记录：

```text
Documentation says ...
Observed implementation does ...
```

涉及 OS、Kernel、SDK、Compiler、Library、Framework、MCU SDK、Hardware revision 时尽量记录版本。不要把某版本特有行为写成普遍规律。

## 22. Debug 方法论

优先自底向上排查，而不是随机改代码。

硬件路径：

```text
Power -> Clock -> Pin -> Electrical Signal -> Peripheral -> Driver -> Protocol -> Service -> Application
```

Crash 路径：

```text
Exception -> PC/LR/SP -> Fault Status -> Call Stack -> Memory Access -> Object Lifetime -> Root Cause
```

网络路径：

```text
Physical -> Link -> IP -> Route -> DNS -> TCP/UDP -> TLS -> Application
```

构建路径：

```text
Toolchain -> Environment -> Configure -> Generate -> Compile -> Link -> Package -> Flash/Deploy
```

## 23. 代码、日志和原始数据

不要把完整仓库、大量日志、完整 dump、完整抓包、大段 AI 对话塞进 Knowledge Note。

大量原始材料优先放入：

```text
history-session/
attachments/
references/
```

如果当前仓库尚无 `attachments/` 或 `references/`，只有确有必要时再创建。Note 正文只保留理解和复现结论所需的关键证据。

代码块必须标注语言。只保留能说明机制、错误原因、最小实现、关键接口或关键修改的代码。

定位源码时优先使用：

```text
file:
symbol/function:
commit:
```

不要依赖固定行号作为唯一定位方式，因为代码会变化。

## 24. 敏感信息

写入 History 或 Note 前，检查敏感信息，但不要过度脱敏导致 Debug Case 无法复现。

Secret 默认必须脱敏：

- Password；
- Token、Access Token、Refresh Token；
- API Key；
- Private Key；
- Cookie；
- Credential、Secret；
- VPN 密码；
- 数据库密码；
- 内网账号密码。

示例：

```text
<REDACTED_TOKEN>
<REDACTED_PASSWORD>
ghp_xxxxxxxxx
```

Environment-sensitive 信息根据实际情况决定是否脱敏：

- 私网 IP；
- 主机名；
- 内部域名；
- 用户名；
- LAN 拓扑；
- 内部服务端口。

对于纯本地技术知识库，私网 IP 默认可以保留，尤其当它参与 Route、Subnet、Proxy、Gateway、VLAN、Proxy bypass 等故障分析时。

示例中这类信息通常可以保留：

```text
192.168.1.100
192.168.133.129
10.0.0.5
172.16.x.x
Subnet
Gateway
Route
Proxy bypass
VLAN
```

如果问题本身涉及 secret-like 值，保留调试所需形状，隐藏真实密钥。除非用户明确要求，不保存完整 Secret。

## 25. 链接和知识连接

优先使用统一 `note_id`、相同 slug 和双向链接。不要额外创建容易失同步的人工映射表。

Note 和 History 建立链接时，必须根据两个文件的真实位置动态计算相对路径：

```text
note/xxx.md -> history-session/xxx.md
  history: ../history-session/xxx.md

note/embedded/xxx.md -> history-session/xxx.md
  history: ../../history-session/xxx.md
```

不允许假设所有 Note 都位于 `note/` 根目录。写入完成后，必须验证双方链接路径在文件系统中真实可解析。

N History ↔ N Note 链接规则：

- Knowledge Note 的 `history` 数组指向 0、1 或多个 History；
- History 的 `note_ids` 与 `notes` 数组指向 0、1 或多个 Note；
- 追加链接时去重；
- 一个 Session 沉淀多个 Note 时，History 不得漏链；
- 一个 Note 被多个 Session 更新时，Note 的 `history` 不得漏链。

使用 Obsidian 双链连接相关知识：

```markdown
[[dma|DMA]]
[[Interrupt]]
[[Cache Coherency]]
```

只链接实际存在或明确计划创建的概念，避免制造大量空链接。

重要 Note 尽量补充：

```text
Prerequisites
Related Concepts
Alternative Technologies
Higher-level Concepts
Lower-level Principles
Implementations
Debug Cases
```

Debug Case 应连接到可泛化的概念和实现；Concept Note 应连接到实现和 Debug Case。

## 26. 执行流程

每次调用本 Skill，按顺序执行：

```text
1. Identify Context
2. Locate Note Repository
3. Inspect Current Project
4. Read Available Evidence
5. Search Existing Notes
6. Determine Note Type and Granularity
7. Decide Create / Update / Merge / Split
8. Fully Read Existing Note Before Updating
9. Save New History Session when valuable real history exists
10. Generate or Incrementally Update Knowledge Note
11. Dynamically Compute Note <-> History Relative Links
12. Build Bidirectional Links and Deduplicate
13. Add Related Knowledge
14. Redact Secrets Without Over-redacting Environment Evidence
15. Validate Files and Actual Relative Paths Through Terminal Reads
16. Report Changed Paths and Uncertainty
```

## 27. 最终验证

结束前检查：

- 目标文件能用 `Get-Content` 读取；
- frontmatter 以 `---` 开始和结束；
- `note_id` 稳定并与 slug 匹配；
- `primary_type` 存在；
- `status` 与正文结论状态一致；
- `Root Cause` 未确认时，`status` 不应直接标为 `resolved`；
- Note -> History 实际路径存在；
- History -> Note 实际路径存在；
- 双向关系的 `note_ids`、`notes`、`history` 一致；
- 一个 Session 对多个 Note 时不得漏链；
- 同时存在 Note 和 History 时必须双向链接；
- 更新 Existing Note 前已经完整读取旧文件；
- `created` 未被错误覆盖；
- `updated` 已正确更新；
- `history` 只被追加、去重或修复失效链接，没有无故删除；
- History Session 没有因为后续结论变化被错误重写；
- 没有不必要的重复 Note；
- 未知、推断、假设已明确标记；
- 大量原始数据没有进入 Note 正文；
- Secret 已脱敏；
- 私网 IP、Subnet、Gateway、Route、Proxy、VLAN 等环境信息没有被无意义过度脱敏；
- 最终回复列出变更路径。

## 28. 输出格式

完成后简短报告：

```text
Created/Updated:
- path/to/note.md
- path/to/history.md

Type:
- primary_type: debug-case
- status: resolved

Links:
- note history: path exists
- history notes: path exists
- note_ids / notes / history are consistent

Uncertainty:
- history_available: false
- root cause remains hypothesis, if applicable
```

不要在最终回复中重复整篇笔记，除非用户要求。
