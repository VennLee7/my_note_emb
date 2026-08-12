---
session_id: 2026-08-12-152302-zephyr-busfault-battery-pointer-stack-corruption
project: xs_nrf_mouse_firmware
date: 2026-08-12

note_ids:
  - zephyr-busfault-battery-pointer-stack-corruption
  - cortex-m-fault-analysis-methodology

notes:
  - ../note/debug/zephyr-busfault-battery-pointer-stack-corruption.md
  - ../note/embedded/cortex-m-fault-analysis-methodology.md
---

# Zephyr BusFault From Battery Pointer Stack Corruption

## Context

Project: `xs_nrf_mouse_firmware`.

Observed environment from workspace and build artifacts:

- Zephyr / NCS firmware project with `build_rf`, `build_ble`, `build_boot` outputs.
- RF app build uses `CONFIG_FLASH_LOAD_OFFSET=0x21000`.
- `build_rf/zephyr/.config` includes `CONFIG_MAIN_STACK_SIZE=3584`, `CONFIG_XIP=y`, `CONFIG_HW_STACK_PROTECTION=y`, `CONFIG_ARM_MPU=y`, `CONFIG_FAULT_DUMP=2`, and `CONFIG_MPU_STACK_GUARD` disabled.
- App packaging script `utilities/firmware_v3/2_pack_app_bin.py` creates `app.bin` as `4 KiB app header + zephyr.bin`.

## Initial Problem

Device booted into RF app, printed normal startup logs, then faulted after battery init:

```text
[INF] [svc][battery] detect init
ADC init ok done
[INF] [svc][battery] current_battery_level :2
***** BUS FAULT *****
  Instruction bus error
r0/a1:  0x00000000  r1/a2:  0x00000001  r2/a3:  0x00001900
r3/a4:  0x00d00228 r12/ip:  0xdfc80000 r14/lr:  0x00022fa5
 xpsr:  0x28000000
Faulting instruction address (r15/pc): 0x00d00228
```

## Investigation

### 1. Identify build artifact and app address

Commands used:

```powershell
Get-ChildItem -Force
rg --files
git status --short
Get-ChildItem -Recurse -File -Include *.elf,*.map,*.lst,*.hex,*.bin | Select-Object FullName,Length,LastWriteTime | Format-Table -AutoSize
Get-ChildItem -Recurse -File -Path build,build_boot,build_rf,build_ble -ErrorAction SilentlyContinue | Where-Object { $_.Name -match 'zephyr\.elf|zephyr\.map|app\.elf|merged|\.config|runners\.yaml' } | Select-Object FullName,Length,LastWriteTime | Format-Table -AutoSize
Select-String -Path build_rf\zephyr\.config -Pattern "CONFIG_FLASH_BASE_ADDRESS|CONFIG_FLASH_LOAD_OFFSET|CONFIG_FLASH_LOAD_SIZE|CONFIG_BOOTLOADER|CONFIG_XIP|CONFIG_MAIN_STACK_SIZE|CONFIG_ISR_STACK_SIZE"
Select-String -Path build_ble\zephyr\.config -Pattern "CONFIG_FLASH_BASE_ADDRESS|CONFIG_FLASH_LOAD_OFFSET|CONFIG_FLASH_LOAD_SIZE|CONFIG_BOOTLOADER|CONFIG_XIP|CONFIG_MAIN_STACK_SIZE|CONFIG_ISR_STACK_SIZE"
```

Relevant result:

```text
build_rf:  CONFIG_FLASH_LOAD_OFFSET=0x21000
build_ble: CONFIG_FLASH_LOAD_OFFSET=0x60000
```

The boot log's `app_addr = 0x00021000` matched the RF app, so analysis used `build_rf/zephyr/zephyr.elf`.

### 2. Resolve `lr` and faulting `pc`

Commands used:

```powershell
Select-String -Path build_rf\CMakeCache.txt -Pattern "CMAKE_ADDR2LINE|CMAKE_OBJDUMP|CMAKE_NM"
& 'C:/ncs/toolchains/66cdf9b75e/opt/zephyr-sdk/arm-zephyr-eabi/bin/arm-zephyr-eabi-addr2line.exe' -e build_rf/zephyr/zephyr.elf -f -C 0x00022fa0 0x00022fa4 0x00022fa5 0x00d00228
```

Result:

```text
0x00022fa5 -> app_svc_init, src/app/app_main.c:42
0x00d00228 -> ??
```

Inference: `pc=0x00d00228` is not a known code symbol. `lr=0x00022fa5` points at the `entry->init()` call site in `app_svc_init()`.

### 3. Disassemble `app_svc_init`

Command used:

```powershell
& 'C:/ncs/toolchains/66cdf9b75e/opt/zephyr-sdk/arm-zephyr-eabi/bin/arm-zephyr-eabi-objdump.exe' -d -S --start-address=0x00022f70 --stop-address=0x00022fc0 build_rf/zephyr/zephyr.elf
```

Key assembly:

```asm
00022f7c <app_svc_init>:
22f84: ldr r4, =__start_app_svc_init_section
22f86: ldr r5, =__stop_app_svc_init_section
...
22f9a: ldr.w r3, [r4], #4
22f9e: cmp   r3, #0
22fa2: blx   r3
22fa4: b.n   22f88
```

Fault state had `r3=0x00d00228` and `pc=0x00d00228`, so the CPU executed `blx r3` with a bad function pointer.

### 4. Inspect static service init table

Commands used:

```powershell
& 'C:/ncs/toolchains/66cdf9b75e/opt/zephyr-sdk/arm-zephyr-eabi/bin/arm-zephyr-eabi-nm.exe' -n build_rf/zephyr/zephyr.elf | Select-String -Pattern "__start_app_svc_init_section|__stop_app_svc_init_section|svc_.*init|app_svc_init"
& 'C:/ncs/toolchains/66cdf9b75e/opt/zephyr-sdk/arm-zephyr-eabi/bin/arm-zephyr-eabi-objdump.exe' -s --start-address=0x0004b7bc --stop-address=0x0004b7dc build_rf/zephyr/zephyr.elf
```

Result:

```text
0004b7bc R __start_app_svc_init_section
0004b7dc R __stop_app_svc_init_section

Contents of section app_svc_init_section:
 4b7bc d95a0200 a5810200 bd930200 c1a90200
 4b7cc 05ab0200 2db50200 e1b80200 31c10200
```

Decoded little-endian pointers:

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

The bad address `0x00d00228` was not present in the ELF init table.

### 5. Inspect packaged `app.bin`

Because device runs packaged firmware, `app.bin` was also checked.

Address conversion:

```text
ELF init table address: 0x0004b7bc
RF app load offset:     0x00021000
zephyr.bin offset:      0x0004b7bc - 0x00021000 = 0x0002a7bc
app.bin offset:         0x0002a7bc + 0x1000 = 0x0002b7bc
```

Command used:

```powershell
$p='app.bin'; $off=0x2b7bc
$fs=[IO.File]::OpenRead((Resolve-Path $p))
$fs.Seek($off,[IO.SeekOrigin]::Begin) | Out-Null
$buf=New-Object byte[] 32
$fs.Read($buf,0,32) | Out-Null
$fs.Close()
for($i=0;$i -lt $buf.Length;$i+=16){
  $chunk=$buf[$i..([Math]::Min($i+15,$buf.Length-1))]
  '{0:X8}: {1}' -f ($off+$i), (($chunk | ForEach-Object { '{0:X2}' -f $_ }) -join ' ')
}
```

Result matched the ELF table:

```text
0002B7BC: D9 5A 02 00 A5 81 02 00 BD 93 02 00 C1 A9 02 00
0002B7CC: 05 AB 02 00 2D B5 02 00 E1 B8 02 00 31 C1 02 00
```

Inference: init table was not wrong in link output or packaged binary. The bad pointer was produced by runtime corruption, most likely corrupted `r4`, stack, or table memory.

### 6. Use last log to find runtime corruption site

Last log before fault:

```text
[INF] [svc][battery] current_battery_level :2
```

Search command:

```powershell
rg -n "battery|current_battery_level|svc_battery|svc_power|SVC_REGISTER" src include device boards
```

This located `svc_vbus_detect_init()` in `src/service/power/svc_battery_manager.c`. This function is also the last entry in `app_svc_init_section`.

### 7. Find pointer-size mismatch

Search command:

```powershell
rg -n "drv_battery_vbat_get_level\(|drv_battery_vbus_get_level\(" src include device
```

Driver API:

```c
int drv_battery_vbat_get_level(int32_t *value);
int drv_battery_vbus_get_level(int32_t *value);
```

Driver behavior in `src/driver/battery/drv_battery.c`:

```c
int drv_battery_vbat_get_level(int32_t *value)
{
    int ret = bsp_adc_sample(0, BATTERY->pvAdc, value);
    if (ret < 0) {
        return -1;
    }
    *value *= VD_RATIO_3_1;
    return 0;
}
```

Original unsafe call pattern in `svc_battery_manager.c`:

```c
static uint16_t adc_battery_voltage = 0;
drv_battery_vbat_get_level(&adc_battery_voltage);

uint16_t boot_adc_mv = 0;
drv_battery_vbat_get_level(&boot_adc_mv);
```

Confirmed root cause: `drv_battery_vbat_get_level()` writes through `int32_t *`, but callers passed addresses of 16-bit objects. The function writes 4 bytes into 2-byte storage. For `boot_adc_mv`, this corrupts the stack frame of `svc_vbus_detect_init()`. After returning to `app_svc_init()`, the saved iterator register or stack state can be corrupted, causing `app_svc_init()` to fetch `0x00d00228` as a fake function pointer and execute `blx r3`.

## Fix Applied

Changed the receive variables passed to `drv_battery_*_get_level(int32_t *)` to 32-bit types, then explicitly converted to `uint16_t` only when entering the old battery filter logic.

Key patch shape:

```c
static int32_t adc_battery_voltage = 0;
int32_t g_app_vbus_value[10] = {0};
int32_t g_app_vbus_average = 0;
int32_t g_app_vbus_average_last = 0;

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

Runtime VBAT sampling:

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

Also added missing prototypes in `include/driver/drv_battery.h`:

```c
int drv_battery_init(void);
int drv_battery_charge_init(void);
```

And included `service/svc_xshid.h` to remove an implicit declaration warning for `xshid_send_cmd()`.

## Verification

Commands used:

```powershell
cmake --build build_rf
python -m py_compile utilities\firmware_v3\2_pack_app_bin.py
python utilities\firmware_v3\2_pack_app_bin.py
```

Results:

- `cmake --build build_rf` passed.
- `svc_battery_manager.c` no longer emitted the relevant C compile warnings after header fixes.
- Linker still emitted existing orphan-section warnings for custom `app_*_init_section`; these were not the root cause of this BusFault.
- Packaging output:

```text
build_rf\zephyr\zephyr.bin --> app.bin : size=0x2BA24, crc=0xD15A6B7A
```

## Confirmed Conclusions

- `0x00d00228` was not present in `app_svc_init_section` in ELF or packaged `app.bin`.
- The CPU jumped to `0x00d00228` through `blx r3` in `app_svc_init()`.
- The bad pointer was produced by runtime corruption, not by static link table generation.
- The confirmed corruption source was passing `uint16_t *` to `drv_battery_vbat_get_level(int32_t *)`.
- The stack corruption happened in or before `svc_vbus_detect_init()` return and surfaced in the caller's init-table loop.

## Related Notes

- ../note/debug/zephyr-busfault-battery-pointer-stack-corruption.md
- ../note/embedded/cortex-m-fault-analysis-methodology.md
