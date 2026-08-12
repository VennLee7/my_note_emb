---
note_id: cortex-m-fault-analysis-methodology
title: Cortex-M Fault 分析方法论：从 PC/LR 到根因
aliases:
  - Cortex-M Fault Analysis Methodology
  - PC LR Fault Debugging
  - HardFault BusFault Debug Method
  - Cortex-M 崩溃寄存器分析

primary_type: concept
type:
  - concept
  - implementation

status: confirmed

domain:
  - embedded
  - cortex-m
  - firmware-debug

project: xs_nrf_mouse_firmware
created: 2026-08-12
updated: 2026-08-12

history:
  - ../../history-session/2026-08-12_152302_zephyr-busfault-battery-pointer-stack-corruption.md

tags:
  - cortex-m
  - hardfault
  - busfault
  - abi
  - stack
  - objdump
  - addr2line
  - debugging

history_available: true
---

# Cortex-M Fault 分析方法论：从 PC/LR 到根因

## TL;DR

Cortex-M fault 不是先猜业务逻辑，而是先还原 CPU 控制流：看 `pc` 要执行哪里，看 `lr` 从哪里来，看寄存器是否对应 `blx rX`、`bx rX`、`pop {..., pc}` 这类控制转移。若 `pc` 是非法地址，先按“跳飞/函数指针/返回地址/栈破坏”路径排查。

典型流程：

```text
Fault registers -> addr2line -> objdump around LR -> identify control-transfer instruction -> trace source register/table -> dump ELF/BIN static data -> if static data correct, inspect runtime corruption -> compare C prototypes/object sizes/callback signatures -> fix -> rebuild/package/verify
```

## Problem / Motivation

嵌入式 firmware 中，BusFault/HardFault 经常不是在根因位置爆炸，而是在函数返回、函数指针调用、调度器恢复、ISR 返回等地方才表现出来。只看最后一条日志或业务分支，容易把问题误判为业务逻辑错误。

Fault 寄存器提供的是低层控制流证据，优先级高于猜测。

## Definition

Cortex-M fault analysis 是基于异常现场寄存器、ELF 符号、反汇编、链接布局、栈和 ABI 规则，重建异常前控制流和数据来源的调试方法。

## Essence

核心不是“看日志”，而是回答三个问题：

1. CPU 当时想执行哪里？即 `pc`。
2. CPU 是怎么到这里的？看 `lr` 和 fault 前的控制转移指令。
3. 这个地址来自哪里？寄存器、栈、函数指针表、返回地址、对象虚表或内存数据。

## Mental Model

```text
Exception frame
  pc  -> faulting instruction or bad target
  lr  -> caller return site / exception return marker
  r0-r3 -> argument/temp registers; often carry bad pointer or call target
  r4-r11 -> callee-saved registers; corruption suggests ABI or stack damage
  sp  -> stack frame; inspect if return address or saved registers look wrong
```

For function-pointer faults:

```text
pc == r3
lr -> instruction after blx r3
r3 loaded from [r4]
=> bad function pointer came from r4-addressed memory
```

Then split the possibilities:

```text
Static source bad?   -> linker section / table / package error
Static source good?  -> runtime memory/register/stack corruption
```

## Core Concepts

### PC

`pc` is where the CPU is fetching or trying to fetch instructions. If `pc` is outside valid flash/text range, think control-flow corruption.

### LR

`lr` usually records where execution should return after a function call. If `lr` maps to the instruction after `blx r3`, then `r3` was the call target.

### BLX / BX

```asm
blx r3
```

means call the address stored in `r3`. A bad `r3` becomes a bad `pc`.

### Callee-saved Registers

ARM ABI requires functions to preserve `r4-r11` if used. If caller state in `r4` changes unexpectedly after a call, suspect stack corruption, callback prototype mismatch, or hand-written assembly ABI violation.

### Thumb Function Pointer Low Bit

On Cortex-M, function pointers often have bit 0 set to indicate Thumb state:

```text
0x0002c131 -> function body at 0x0002c130, Thumb bit set
```

### ELF vs BIN vs Packaged Firmware

ELF contains symbols and logical addresses. BIN is raw bytes. A custom package may add headers before the BIN. When checking whether a table is actually wrong, inspect both ELF and the packaged image if the device runs the package.

## Principle

Use deterministic narrowing rather than random code edits:

1. Resolve addresses.
2. Locate the exact instruction that transferred control.
3. Identify where the target came from.
4. Verify whether the static source contains the bad value.
5. If not static, search for runtime corruption near the last known executed code.

## Minimal Command Set

Resolve `lr` / `pc`:

```powershell
arm-zephyr-eabi-addr2line.exe -e build_rf/zephyr/zephyr.elf -f -C 0x00022fa5 0x00d00228
```

Disassemble around a suspicious address:

```powershell
arm-zephyr-eabi-objdump.exe -d -S --start-address=0x00022f70 --stop-address=0x00022fc0 build_rf/zephyr/zephyr.elf
```

List symbols by address:

```powershell
arm-zephyr-eabi-nm.exe -n build_rf/zephyr/zephyr.elf
```

Find init-table boundaries:

```powershell
arm-zephyr-eabi-nm.exe -n build_rf/zephyr/zephyr.elf | Select-String "__start_app_svc_init_section|__stop_app_svc_init_section"
```

Dump section bytes:

```powershell
arm-zephyr-eabi-objdump.exe -s --start-address=0x0004b7bc --stop-address=0x0004b7dc build_rf/zephyr/zephyr.elf
```

Search for suspect APIs:

```powershell
rg -n "drv_battery_vbat_get_level\(|drv_battery_vbus_get_level\(" src include device
```

Check Zephyr final config:

```powershell
Select-String -Path build_rf\zephyr\.config -Pattern "CONFIG_FLASH_LOAD_OFFSET|CONFIG_MAIN_STACK_SIZE|CONFIG_MPU_STACK_GUARD|CONFIG_FAULT_DUMP"
```

## Debug Methodology

### 1. Classify the fault shape

```text
pc invalid                  -> jump/return/function pointer corruption
pc == rX                    -> likely bx/blx through register
lr maps to call site         -> identify exact call instruction
fault on data address        -> inspect load/store address and access width
fault after last log returns -> suspect stack or saved-register corruption
```

### 2. Use `addr2line` but do not stop there

`addr2line` maps addresses to source lines. It does not explain how control reached the bad address. Always pair it with `objdump`.

### 3. Prefer assembly for control-flow questions

C source hides whether a call is direct or indirect. Assembly reveals:

```text
bl func     direct call
blx r3      indirect function-pointer call
pop {...pc} return through stack
```

### 4. Dump static tables before blaming runtime

If a function pointer came from a section/table, inspect the table in ELF. If firmware is packaged, also inspect the packaged bytes at the corresponding offset.

### 5. If static data is correct, search runtime corruption

Common causes:

- Writing through pointer of wrong type/size.
- `memcpy` length exceeds destination size.
- Array index overflow.
- Callback function signature mismatch.
- Stack overflow.
- Hand-written assembly not preserving callee-saved registers.
- Use-after-free or stale pointer in systems with heap/dynamic objects.

## Failure Model / Debug

### Pointer output type mismatch

Pattern:

```c
int api(int32_t *out);
uint16_t value;
api(&value); // writes 4 bytes into 2-byte object
```

Failure may surface later, especially if the object is on stack.

### Function pointer table corruption

Pattern:

```asm
ldr.w r3, [r4], #4
blx   r3
```

If `r3` is bad:

1. Dump the table range pointed to by `r4` at link time.
2. Check whether the bad value exists in static data.
3. If absent, suspect corrupted iterator register or memory.

### Stack corruption after a function returns

Symptoms:

```text
last log inside function is normal
fault occurs in caller immediately after return
r4-r11 or lr look implausible
pc is data-looking value
```

## Practical Checklist

```text
[ ] Identify build artifact matching boot address.
[ ] Read fault registers: pc, lr, sp, r0-r3, r4 if available.
[ ] addr2line pc/lr.
[ ] objdump around lr and pc if pc is valid.
[ ] Identify control transfer instruction.
[ ] Trace source register or stack slot.
[ ] Dump static table or object if involved.
[ ] Check package offset if device uses custom app.bin.
[ ] Search last log's function for pointer writes, buffers, casts, callbacks.
[ ] Compare all output parameter types against prototypes.
[ ] Rebuild and confirm warnings are removed.
[ ] Package and flash/test.
```

## Boundary

This method localizes control-flow and memory-corruption classes of faults. It does not replace hardware-level checks for power, clock, pinmux, bus electrical issues, or peripheral-specific status registers. Use the fault path together with hardware and driver evidence.

## Related Knowledge

- [Zephyr BusFault：电池采样指针类型不匹配导致栈破坏](../debug/zephyr-busfault-battery-pointer-stack-corruption.md)
- [History Session](../../history-session/2026-08-12_152302_zephyr-busfault-battery-pointer-stack-corruption.md)

## References

- Local evidence: `build_rf/zephyr/zephyr.elf`, `build_rf/zephyr/zephyr.map`, `build_rf/zephyr/.config`, `app.bin`.
- Local source: `src/app/app_main.c`, `include/osal/osal.h`, `src/service/power/svc_battery_manager.c`, `src/driver/battery/drv_battery.c`, `include/driver/drv_battery.h`.
