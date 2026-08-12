---
note_id: zephyr-busfault-battery-pointer-stack-corruption
title: Zephyr BusFault：电池采样指针类型不匹配导致栈破坏
aliases:
  - Zephyr BusFault Battery Pointer Stack Corruption
  - Instruction bus error after svc_vbus_detect_init
  - drv_battery_vbat_get_level int32_t pointer bug

primary_type: debug-case
type:
  - debug-case
  - implementation

status: resolved

domain:
  - embedded
  - zephyr
  - cortex-m
  - firmware-debug

project: xs_nrf_mouse_firmware
created: 2026-08-12
updated: 2026-08-12

history:
  - ../../history-session/2026-08-12_152302_zephyr-busfault-battery-pointer-stack-corruption.md

tags:
  - busfault
  - cortex-m
  - zephyr
  - stack-corruption
  - function-pointer
  - abi
  - adc
  - battery

history_available: true
---

# Zephyr BusFault：电池采样指针类型不匹配导致栈破坏

## TL;DR

一次 Zephyr RF app 启动后 `Instruction bus error`，fault 现场为 `pc=r3=0x00d00228`、`lr=0x00022fa5`。`lr` 反查到 `app_svc_init()` 的 `entry->init()`，反汇编显示 CPU 执行了 `blx r3`。ELF 和 `app.bin` 中的 `app_svc_init_section` 均无 `0x00d00228`，因此不是链接表错误，而是运行时栈/寄存器被破坏。

根因是 `drv_battery_vbat_get_level(int32_t *value)` 被传入 `uint16_t *`，驱动按 4 字节写入 2 字节对象，覆盖 `svc_vbus_detect_init()` 的栈帧，返回到 `app_svc_init()` 后表遍历状态损坏，最终把数据当函数指针跳转。

## Environment

- Project: `xs_nrf_mouse_firmware`
- Target observed from artifacts: RF app
- Build output: `build_rf/zephyr/zephyr.elf`
- `CONFIG_FLASH_LOAD_OFFSET=0x21000`
- `CONFIG_MAIN_STACK_SIZE=3584`
- `CONFIG_XIP=y`
- `CONFIG_ARM_MPU=y`
- `CONFIG_MPU_STACK_GUARD` disabled
- Packaging: `app.bin = 0x1000 app header + zephyr.bin`

## Symptom

Boot log reached battery service init, then faulted:

```text
[INF] [svc][battery] detect init
ADC init ok done
[INF] [svc][battery] current_battery_level :2
***** BUS FAULT *****
  Instruction bus error
r3/a4:  0x00d00228 r14/lr:  0x00022fa5
Faulting instruction address (r15/pc): 0x00d00228
```

## Expected Behavior

`app_svc_init()` should iterate through all registered service init entries in `app_svc_init_section`, call each valid init function once, then return to `main()`.

## Observed

`app_svc_init()` attempted to call `0x00d00228` as a function pointer. This address was not a valid text symbol and did not exist in the static init table.

## Evidence

### `lr` points to function pointer call site

`addr2line` resolved `0x00022fa5` to `app_svc_init()` in `src/app/app_main.c` around `entry->init()`.

Relevant disassembly:

```asm
22f9a: ldr.w r3, [r4], #4
22f9e: cmp   r3, #0
22fa2: blx   r3
22fa4: b.n   22f88
```

Fault registers:

```text
r3 = 0x00d00228
pc = 0x00d00228
lr = 0x00022fa5
```

This proves control reached `blx r3` with a bad function pointer.

### ELF init table is valid

`app_svc_init_section` in ELF contained:

```text
0x00025ad9
0x000281a5
0x000293bd
0x0002a9c1
0x0002ab05
0x0002b52d
0x0002b8e1
0x0002c131 -> svc_vbus_detect_init + 1
```

`0x00d00228` was absent.

### Packaged app init table is also valid

Using RF load offset and app header:

```text
ELF address:       0x0004b7bc
RF app offset:     0x00021000
zephyr.bin offset: 0x0002a7bc
app.bin offset:    0x0002b7bc
```

`app.bin` bytes at `0x2b7bc` matched the ELF init table. This ruled out packaging corruption of `app_svc_init_section`.

### Last log localizes the runtime corruption site

The last log came from `svc_vbus_detect_init()`:

```text
[svc][battery] current_battery_level :2
```

This function is also the last entry in `app_svc_init_section`.

### Root cause code pattern

Driver API:

```c
int drv_battery_vbat_get_level(int32_t *value);
```

Driver writes through the 32-bit pointer:

```c
*value *= VD_RATIO_3_1;
```

Unsafe original call pattern:

```c
static uint16_t adc_battery_voltage = 0;
drv_battery_vbat_get_level(&adc_battery_voltage);

uint16_t boot_adc_mv = 0;
drv_battery_vbat_get_level(&boot_adc_mv);
```

The second case is especially dangerous because `boot_adc_mv` is a stack variable inside `svc_vbus_detect_init()`.

## Investigation Timeline

1. Identify RF app from `CONFIG_FLASH_LOAD_OFFSET=0x21000` and boot log `app_addr=0x00021000`.
2. Use `addr2line` to map `lr=0x00022fa5` to `app_svc_init()`.
3. Disassemble `app_svc_init()` and identify `blx r3` function-pointer call.
4. Dump `app_svc_init_section` from ELF; verify no `0x00d00228`.
5. Dump corresponding bytes from packaged `app.bin`; verify no `0x00d00228`.
6. Use last log to focus on `svc_vbus_detect_init()`.
7. Search battery sampling calls and compare function prototypes.
8. Find `uint16_t *` passed to `int32_t *` API.
9. Fix callers to use `int32_t` receive variables and explicit safe conversion to `uint16_t`.
10. Build RF app and regenerate `app.bin`.

## Root Cause

A 32-bit output API was called with 16-bit storage:

```c
int drv_battery_vbat_get_level(int32_t *value);
uint16_t boot_adc_mv;
drv_battery_vbat_get_level(&boot_adc_mv); // wrong object size
```

The callee writes 4 bytes through `value`, but the caller provided only a 2-byte object. In the stack-variable case, this overwrote adjacent stack data in `svc_vbus_detect_init()`. After returning, `app_svc_init()` recovered corrupted state and read a bogus init function pointer, then executed `blx r3` to `0x00d00228`.

## Fix

Use 32-bit variables for all direct calls to `drv_battery_*_get_level(int32_t *)`, then explicitly narrow for the existing filter code:

```c
static int32_t adc_battery_voltage = 0;
int32_t boot_adc_mv = 0;

static uint16_t battery_mv_to_u16(int32_t mv)
{
    if (mv <= 0) {
        return 0;
    }

    if (mv > UINT16_MAX) {
        return UINT16_MAX;
    }

    return (uint16_t)mv;
}
```

Runtime sampling:

```c
if (drv_battery_vbat_get_level(&adc_battery_voltage) < 0) {
    return;
}

battery_adc_filter_process(battery_mv_to_u16(adc_battery_voltage));
```

Boot sampling:

```c
int32_t boot_adc_mv = 0;
for (int i = 0; i < 5; i++) {
    drv_battery_vbat_get_level(&boot_adc_mv);
    if (boot_adc_mv > 2000) break;
    osal_delay_ms(2);
}

battery_adc_filter_init(battery_mv_to_u16(boot_adc_mv));
```

Also align VBUS receive storage with the API:

```c
int32_t g_app_vbus_value[10] = {0};
int32_t g_app_vbus_average = 0;
int32_t g_app_vbus_average_last = 0;
```

## Verification

Commands:

```powershell
cmake --build build_rf
python -m py_compile utilities\firmware_v3\2_pack_app_bin.py
python utilities\firmware_v3\2_pack_app_bin.py
```

Results:

- RF build passed.
- `svc_battery_manager.c` no longer emitted the relevant pointer-size or implicit-declaration warnings after header fixes.
- Packaged `app.bin` regenerated successfully:

```text
build_rf\zephyr\zephyr.bin --> app.bin : size=0x2BA24, crc=0xD15A6B7A
```

## Why the Fix Works

The driver API contract is now respected: every direct output pointer passed to `drv_battery_vbat_get_level()` and `drv_battery_vbus_get_level()` points to at least 4 bytes of storage. The old 16-bit battery filter still receives a bounded `uint16_t`, but only after the 32-bit driver write has completed safely.

## Prevention

- Treat `incompatible pointer type` as a hard error in firmware builds.
- Do not ignore implicit function declaration warnings; they hide ABI and prototype mismatch bugs.
- For output parameters, align object type exactly with API declaration.
- For ADC millivolt values, use a single project-level type or wrapper API rather than mixing `uint16_t`, `uint32_t`, and `int32_t` across layers.
- Enable stack diagnostics where feasible: `CONFIG_STACK_SENTINEL`, `CONFIG_MPU_STACK_GUARD`, or thread analyzer options during debug builds.

## Generalized Knowledge

If a Cortex-M fault shows:

```text
pc == rX
lr points to a call site
instruction near lr is blx rX
```

then the immediate issue is usually a bad function pointer or corrupted return/control state. Check the source of `rX`; if it comes from a table, dump the static table from ELF and packaged binary. If the table is correct, investigate runtime memory corruption, especially stack writes in the last function called before the fault.

## Related

- [Cortex-M Fault 分析方法论](../embedded/cortex-m-fault-analysis-methodology.md)
- [History Session](../../history-session/2026-08-12_152302_zephyr-busfault-battery-pointer-stack-corruption.md)
