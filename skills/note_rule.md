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

示例：

```text
optical-encoder-power-issue.md
zephyr-device-model.md
dma-cache-coherency.md
virtual-memory.md
```

History 文件名包含日期：

```text
2026-08-12_optical-encoder-power-issue.md
```

避免 `issue.md`、`today.md`、`new-note.md`、`summary.md` 等无意义名称。

## 11. Frontmatter

Knowledge Note 示例：

```yaml
---
note_id: optical-encoder-power-issue
title: 光电编码器供电导致 AB 相无输出问题
primary_type: debug-case
type:
  - debug-case
  - implementation
domain:
  - embedded
  - hardware-debug
project: xs_nrf_mouse_firmware
created: 2026-08-12
updated: 2026-08-12
history:
  - ../history-session/2026-08-12_optical-encoder-power-issue.md
tags:
  - encoder
  - gpio
  - power
  - debugging
history_available: true
---
```

History 示例：

```yaml
---
session_id: 2026-08-12-optical-encoder-power-issue
note_id: optical-encoder-power-issue
project: xs_nrf_mouse_firmware
date: 2026-08-12
note:
  - ../note/embedded/optical-encoder-power-issue.md
---
```

一篇 Note 可以对应多个 History Session：

```yaml
history:
  - ../history-session/session-a.md
  - ../history-session/session-b.md
  - ../history-session/session-c.md
```

不要额外建立容易失同步的人工映射表。使用统一 `note_id`、相同 slug 和双向链接。

## 12. History 规则

存在有价值的真实对话或调查过程时，创建 History Session。

History 保留：

- 最初问题、环境、原始日志、用户证据；
- 关键代码片段、排查过程、假设、被否定的假设；
- 实验、结果、AI 分析、用户反馈；
- 最终结论和尚未解决的问题。

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

## Related Note
```

如果无法获得真实对话，明确说明或不创建 History。不要伪造对话。

## 13. Note 写作风格

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

## 14. 核心思维模型

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

## 15. Concept Note 模板

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

## 16. System Note 模板

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

## 17. Implementation Note 模板

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

## 18. Debug Case 模板

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

若根因未完全证明，写 `Current Leading Hypothesis`，不要写 `Root Cause`。

## 19. 事实可信度

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

## 20. Debug 方法论

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

## 21. 代码、日志和原始数据

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

## 22. 敏感信息

写入 History 或 Note 前，检查并默认脱敏：

- API Key、Token、Password、Private Key、Cookie；
- Credential、Secret、内网账号密码；
- 其他明显敏感值。

示例：

```text
<REDACTED_TOKEN>
<REDACTED_PASSWORD>
ghp_xxxxxxxxx
192.168.x.x
```

如果问题本身涉及敏感值，保留调试所需形状，隐藏真实值。除非用户明确要求，不保存完整敏感信息。

## 23. 链接和知识连接

优先使用统一 `note_id` + 双向链接 + 相同 slug。

使用 Obsidian 双链连接相关知识：

```markdown
[[DMA]]
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

## 24. 执行流程

每次调用本 Skill，按顺序执行：

```text
1. Identify Context
2. Locate Note Repository
3. Inspect Current Project
4. Read Available Evidence
5. Search Existing Notes
6. Determine Note Type and Granularity
7. Decide Create / Update / Merge / Split
8. Save History Session when valuable real history exists
9. Generate or Update Knowledge Note
10. Build Bidirectional Links
11. Add Related Knowledge
12. Redact Sensitive Data
13. Validate Files Through Terminal Reads
14. Report Changed Paths and Uncertainty
```

## 25. 最终验证

结束前检查：

- 目标文件能用 `Get-Content` 读取；
- frontmatter 以 `---` 开始和结束；
- `note_id` 稳定并与 slug 匹配；
- `primary_type` 存在；
- Note 与 History 的相对链接可解释，尽量有效；
- 同时存在 Note 和 History 时必须双向链接；
- 没有不必要的重复 Note；
- 未知、推断、假设已明确标记；
- 大量原始数据没有进入 Note 正文；
- 明显敏感信息已脱敏；
- 最终回复列出变更路径。

## 26. 输出格式

完成后简短报告：

```text
Created/Updated:
- path/to/note.md
- path/to/history.md

Type:
- primary_type: debug-case

Links:
- note -> history
- history -> note

Uncertainty:
- history_available: false
- root cause remains hypothesis, if applicable
```

不要在最终回复中重复整篇笔记，除非用户要求。
