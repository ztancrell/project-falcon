# Kernel comparison: Apple Boot Camp vs. Falcon kernels, MBP16,1 Radeon Pro 5500M (1002:7340 SUBSYS_020F106B REV_40)

Kernels:
- **Apple**: atikmdag.sys 26.20.13003.5002 (B348746), image base 0x140000000.
- **Live** (what the MBP actually runs): amdkmdag 32.0.21045.11001. Used as "the Falcon kernel" for this comparison.
- 32.0.12033.5029 (Falcon repo) was checked with `strings` only. Its flag strings match the live kernel's.

Method: Ghidra (REA) timed out importing the 60 MB Apple kernel at 330 s, so I stopped using it. I worked with objdump plus a RIP-relative xref scanner and .pdata function bounds (throwaway scripts, not kept). Addresses are VAs.
Tags: **[C]** = confirmed in disassembly, **[I]** = inferred.
None of the strings or code I read contained instructions or prompt-injection text.

## 1. "Falcon" SMC firmware: Southern Islands only, so it does nothing for Navi14
- [C] Apple `0x1412efac8` reads `PP_EnableLoadFalconSmcFirmware` (string 0x1404596a8). It picks firmware by internal chip revision `[hwmgr+0x10]`:
  - rev 0x14–0x27 (Pitcairn) gets a Falcon blob at 0x1424bb440.
  - rev 0x28–0x3B (Cape Verde) gets 0x14251aed0.
  - Any other revision prints "No proper Falcon smc firmware is found…for this ASIC variant!" (line 0xb24).
  - When the flag is 0, it picks the normal or kicker blobs (0x14249b610 / 0x1424fb0a0 / 0x1424db270).
  - The function lives beside `PhwSIslands_Load*SMCFirmware` (SI hwmgr). The revision ranges are AMD's SI ids (Pitcairn = 20, Cape Verde = 40).
- [I] The blobs sit in slots spaced 0x1FE30 bytes apart (about 127 KB). I did not find a version field.
- [C] The live kernel and 32.0.12033 contain neither the `PP_EnableLoadFalconSmcFirmware` string nor any SI SMC loader. **The key is dead on the kernel Falcon uses, and was always irrelevant to Navi14.** No power, TDC/EDC or thermal values are involved.
- The "SMU 11_0_falcon" strings in the 32.x kernels are only sensor-log text for an SMU11 variant name. I found nothing that loads alternate firmware for them.

## 2. The four flags
- **PP_Apple_Bootcamp_Enable**
  - [C] Present in both kernels, in the same SW-SMU platform-event handler: Apple `0x1412be850`, live `0x1413433ac`.
  - The handler takes three platform events: thermal-state change, power-source (AC/DC) change, and a "power limit" event (`[evt+0x10]`, value `[evt+0x14]`).
  - When the flag is 1 and `[hwmgr+0x919]` is 1, the live kernel calls SMU vtable slot +0x10e0 (`0x141340af4`) directly with the platform's value. The error string is "Failed to set power limit by forced mode!".
  - Otherwise the value is wrapped as a generic event (0xc/0xb) via `0x141343754`.
  - [I] The flag lets the platform (Apple ACPI) force the GPU power limit. I could not see the values Apple's firmware sends.
  - [C] In the Apple kernel only, a second reader `0x14134fe74` applies when family == 0x6E (SI): it sets bytes +0xbe0 = 0x10 and +0xbe4 = 1. It does not apply to Navi.
- **PP_Falcon_QuickTransition_Enable**
  - [C] Read in four per-SMU11-variant config readers in the live kernel (e.g. `0x14135e320`) into `cfg[+1]`, next to `PP_ACDCGpioDisabled` (`cfg[+0]` = value != 1).
  - Its consumer, `PhwNavi10_SetFeaturesPlatformCaps` at `0x14135f4cc`, checks one thing: if QuickTransition = 1 but AC/DC GPIO switching is off, it asserts "ACDCSwitchGpioSupport is not enabled!" and clears caps bits 4 and 18. With GPIO on (the default), it changes nothing.
  - [I] It is an AC/DC hardware-switch feature and has no effect on clocks while on AC.
- **PP_EnableLoadFalconSmcFirmware**: see section 1. It is SI-only in Apple's kernel and missing from the 32.x kernels.
- **KMD_BootCampPlatform**
  - [C] Live kernel: an entry in a 56-entry KMD registry table at 0x1407b7c20 (id 0x1a, default 0, DWORD), built by `0x1400da0e0`.
  - I did not find which code reads id 0x1a, so I can't say what it does. There is no UTF-16 string xref in code.
  - [I] I found no link between it, or vendor 0x106B, and OverDrive/Tuning/WattMan in the paths I examined. That is not proven.

## 3. Mac-ID checks
- [C] Neither kernel has an immediate 0x020F or 0x0210 subsystem ID, or a combined 0x020F106B dword.
- [C] Live kernel, vendor 0x106B compares:
  - `0x141487e88` (PAGECALC ASIC-caps init). For caps-table entries flagged 0x4 on Apple boards, it clears bit 0 of `[caps+0x204]` and uses mask 0x1c3001b7 instead of the table's mask. [I] Meaning unknown.
  - `0x14149af63` (PAGED3PC). If vendor = Apple and (dev 7310 with internal rev 0xd4, or dev 7340 with internal rev 0x21d), it sets a display-object byte (+0x523). [I] Probably a display/panel quirk, not power.
- [C] Apple kernel: six 0x106B compares in PowerPlay. All are Polaris (rev 0x50–0x63: `0x14136e7d8`, `0x14138116c`, `0x1413d3134`) or SI (`0x1413b00e0`, gated on the Bootcamp byte +0xbe4). **None are Navi14.** So the Apple kernel has no Navi14-specific Apple power code that the live kernel lacks.

## 4. BD-key values (59,24,06,D2)
- Not resolved. I found no `0xbd` immediate in `.text`, so the code that builds the key name is in a PAGE* section or uses a table.
- [I] The name encoding is known (see the deobfuscate-pci-id skill), but what the 4-byte value means is still unknown.

## 5. Soft PowerPlay table (MorePowerTool)
- [C] The live kernel honours it. `0x141434510` (PAGESIPC, the SW-SMU PP-table loader) uses a registry descriptor `{name "PP_PhmSoftPowerPlayTable", buf, size}` at 0x144822694.
  - It reads the value into a buffer of up to 0x2000 bytes via `0x141432d50`.
  - If the size is nonzero, it sets override = 1 (`[r14+0x18]`) and copies the registry table in place of the VBIOS table (`0x140796940`). Otherwise it reads the table size from the VBIOS (`0x141433420`).
  - [I] This loader is shared by all SW-SMU ASICs, so it applies to Navi14.
- The Apple kernel also has the classic `PP_PhmSoftPowerPlayTable` handling (refs `0x1413500de`…).

## 6. MiscControl
- Not resolved. The string is referenced from data tables at file offsets 0xac7350 and 0x5949830 (PAGECALD). I did not trace the bits.

## Extra: PP_PhmSoftWTTable (384 bytes on the MBP)
- [C] `0x14135d3e4` builds the Navi10-family display **watermark table**: 2 × 4 rows at 0x2c stride, each with min/max DCEF or SOC clock, min/max UCLK and a WM set. Total size 0x180 = 384 bytes. It passes the result to `0x14133c370` with key `PP_PhmSoftWTTable`.
- [I] The driver writes this as a snapshot or soft copy of the DC watermark ranges. "0000 ffff" is the default 0–65535 MHz range. It is a display-memory watermark table, not a performance tuning knob. Leave it alone.

## Suggestions for Falcon
1. **Kernel**: keep 32.0.21045.11001 or 32.0.12033.5029. The Apple 26.20 kernel has no Navi14 Apple power code those kernels lack (high confidence). The 32.x kernels drop only SI and Polaris Apple paths, which don't matter here.
2. **INF**:
   - `PP_EnableLoadFalconSmcFirmware` can be dropped. It is a no-op on 32.x (high confidence).
   - Keep `PP_Apple_Bootcamp_Enable=1`. It lets Apple's platform power-limit events apply directly, as on Apple's driver (medium confidence that it's harmless or helpful).
   - `PP_Falcon_QuickTransition_Enable` is harmless (high confidence).
   - `KMD_BootCampPlatform`: keep it as Apple does (unknown effect).
3. **MorePowerTool**: a soft PP table will be honoured by this kernel (high confidence). Read the current VBIOS power limit and TDC limits with MPT itself; I extracted no 5500M limit values. Raise PPT in small steps and watch temperatures, because MBP16 cooling is the practical ceiling. Undervolting through the OD table is the likelier fps win.
   - [I] The forced-mode Bootcamp path may let Apple's ACPI override PPT at runtime. If an MPT PPT raise doesn't hold, test `PP_Apple_Bootcamp_Enable=0`.
