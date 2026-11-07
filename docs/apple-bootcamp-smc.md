# Boot Camp (ESD 061-62383, MBP16,1): SMC, fans and GPU power under Windows

Method: static disassembly (objdump; REA used for the KeyManager imports and procedure list, but REA Ghidra
decompilation timed out because another session's target, atikmdag.sys, was loaded). No package binary was run.
Tags: **[C]** = confirmed in code, **[I]** = inferred.

## Bottom line
- **The SMC driver is MacHALDriver.sys, not KeyManager.sys.** [C]
- **No Boot Camp component reads or writes any fan, thermal, power or GPU SMC key.** The only SMC key that
  Apple's Windows software writes is `LKSB` (keyboard backlight). [C, across the scanned binaries]
- So Boot Camp does not cap GPU power or fan speed under Windows. Any cap comes from SMC firmware autonomy
  (it runs fans and thermals itself), ACPI, and the AMD driver's PowerPlay/VBIOS limits. [I]
- **Lever:** MacHALDriver exposes generic SMC read and write IOCTLs with **no key allowlist**. An elevated
  user-mode tool can read any key and write any key (1-32 bytes), for example `F0Md`, `F0Tg`, `F0Mn`, `F1Mn`.
  [C for the driver; I for what those keys do on this T2 machine]

## 1. KeyManager.sys (6.1.7001.0): hotkey event broker, no hardware access
- PDB path `AppleKeyAgentWin-7045.2\...\KeyManager.pdb`. It is a KMDF root-enumerated device (`ROOT\KeyManager`)
  exposing `\Device\KeyManager` and `\DosDevices\KeyManager`. [C]
- Imports are only IoCreateDevice/SymLink, ObReferenceObjectByHandle, KeSetEvent, pool and WDF functions.
  There is no HAL, MmMapIoSpace or IoCallDriver, and **no in/out instructions** (the objdump hits were UTF-16
  strings). [C]
- Device type 0xB403, FILE_ANY_ACCESS. About 56 IOCTLs (0xB4032004..0xB4032133) in three groups. [C]
  - **METHOD_NEITHER (…3/…B/…F) = "register":** InputBuffer is a user event HANDLE. The driver references it
    and adds it to one of about 20 per-slot lists, keyed by FileObject (FUN_140001000).
  - **METHOD_BUFFERED (…0/…4/…8/…C) = "signal":** KeSetEvent on every event in a slot (FUN_140001138).
    Register/signal pairs share a slot, e.g. 0x…53 registers and 0x…54 signals; 0x…5B/0x…5C, 0x…8B/0x…7C,
    0x…8F/0x…80, 0x…1F/0x…18, 0x…6F/0x…18.
  - **Mailbox:** 0xB4032008 stores up to 0x400 bytes and 0xB4032004 reads them back and clears.
    0xB4032078 clears a flag that gates the 0x…18 signal.
- Clients: the keyboard drivers (Keymagic, Keymagic2, AppleKeyboardInternalUSB, AppleWTP, AppleWirelessMouse,
  AppleTPP, AppleDFR) reference KeyManager. So do Bootcamp.exe, AppleControlPanel.exe, AppleOSSMgr.exe and
  BootCampService.exe. It relays Fn/hotkey events between components. **No SMC role.** [C/I]

## 2. BootCampService.exe (6.1.7800.2)
- It opens `\\.\KeyManager`, falling back to `\\.\KeyAgent`. It creates 4 auto-reset events and registers them
  with IOCTLs 0xB4032053, 0xB403205B, 0xB403208B and 0xB403208F. These are its **only** DeviceIoControl calls. [C]
- Its WaitForMultipleObjects loop (5 handles: stop event plus the 4) drives a 16-step **display brightness**
  ramp (level counter capped at 0x10, float scale, timers). The rest of the service handles auto-brightness
  (ALS lux registry values), auto-rotation, EDID via WMI `WmiMonitorDescriptorMethods`, and device-change
  monitoring. [C]
- The service does not import or open MacHALDriver. **It never talks to the SMC.** [C]
- `cAMD` is a false lead. It is the CRT `__isa_available_init` CPUID check (`xor ecx,'cAMD'` / `'enti'` /
  `'Auth'` = "AuthenticAMD") at 0x14000dbb2. [C]
- `EFAn` is also a false lead. It sits at file offset 0x16a94, past `.reloc`, inside the Authenticode
  signature blob (random bytes). [C]
- Nothing in it covers fan modes, GPU power, thermal states, or "Windows is running" signalling. [C]

## 3. MacHALDriver.sys (6.1.7500.0): the real SMC and firmware access driver
- Exposes `\Device\MacHALDriver` and `\DosDevices\MacHALDriver`. Imports Ex{Get,Set}FirmwareEnvironmentVariable,
  IoWMI*, registry functions, KeStallExecutionProcessor and a mutex. [C]
- **Port I/O:** [C]
  - SMC legacy ports 0x300 (data), 0x304 (cmd/status) and 0x31E. The busy-wait polls status bit 0x04, up to
    10000 x 10 µs.
  - CMOS 0x70-0x73.
  - PCI config 0xCF8/0xCFC.
  - SMI port 0xB2/0xB3. The driver picks 0xB2 if PCI 0:0.0 vendor is 0x8086 and 0x52E if it is 0x10DE,
    otherwise it fails the request.
- SMC commands used: 0x10 READ, 0x11 WRITE, 0x12 GET_KEY_BY_INDEX, 0x13 GET_KEY_INFO. These match the standard
  applesmc protocol. [C]
- **No hardcoded SMC FourCCs.** Keys come from the caller. The only immediate tag is the pool tag `MACH`. [C]
- IOCTLs: device type 0x9C40, FILE_ANY_ACCESS, METHOD_BUFFERED. [C]

  | IOCTL | Function | Layout |
  |---|---|---|
  | 0x9C402454 | **SMC read key** | in: key[4] as ASCII in memory order (e.g. "F0Ac"); out: data, length = OutputBufferLength (1-32) |
  | 0x9C402458 | **SMC write key** | in: key[4], 1 ignored byte, data[n]; n = InputLength-5 (1-32); **no allowlist** |
  | 0x9C40245C / 0x9C402460 | key by index / key info (type, size) | in: key[4]+pad (5); out: 0xC |
  | 0x9C402410 / 0x9C402414 | CMOS/RTC NVRAM read/write (0x70-0x73, waits on reg 0xA UIP) | |
  | 0x9C402440/444/448/44C/450 | firmware SMI: 0xB3 = 2, 3 or (x<<4)\|4, then 0xB2 = 0xBF | purpose unknown [I: firmware/boot service, not power] |
  | 0x9C402464, 0x480, 0x484, 0x488, 0x48C | Bluetooth link-key sync (BTHPORT/HIDBTH), EDID override (`DISPLAY\APP9CAC/9CAD`), `ICMDisabledInWindows`, `ServerMode`, firmware variables | [I from strings and callees] |

- **Who uses the SMC IOCTLs:** [C]
  - Bootcamp.exe (tray) and AppleControlPanel.exe call 0x9C402460 (key info on `LKSB`) and 0x9C402458 (write
    `LKSB` = {0x00, u16 level}, with a 0x1000 scale, for keyboard backlight).
  - AppleOSSMgr.exe uses only the CMOS IOCTLs 0x410/0x414.
  - No binary references 0x9C402454 (read) as an immediate.
  - String scans of Bootcamp.exe and AppleControlPanel.exe find no fan, thermal, SMC, GPU or performance
    strings.

## 4. Bootcamp.exe / AppleControlPanel.exe
- Covered above: LKSB keyboard backlight, the SMI call (0x448), CMOS, and Bluetooth/EDID/HAL settings
  (0x464/0x480/0x484). **Nothing about fans or performance.** [C]

## Untrusted-content note
- No instruction-like or prompt-injection text was found in strings or disassembly.

## Lever for Falcon users
- A small elevated user-mode tool can open `\\.\MacHALDriver` (installed by Boot Camp; Apple-signed) and use
  0x9C402454 / 0x9C402458 to read and write SMC keys. This is the same mechanism Macs Fan Control and similar
  tools use on Boot Camp.
  - **Lowest-risk key:** raise the fan minimum (`F0Mn`/`F1Mn`; on T2 Macs these are usually 4-byte float
    `flt `). The SMC still manages fans above that floor. **Check the type with key-info (0x…460) first.** [I]
  - **Riskier:** force mode `F0Md` = 1 plus target `F0Tg`. This disables auto control and needs a watchdog
    that resets it. [I]
  - **Temperatures for monitoring:** `TG0P`/`TG0D` (GPU), `TC0P` (CPU). Power: `PG0R`, `PSTR`. [I]
- **Caveats:** [I]
  - The MBP16,1 is a T2 machine whose SMC lives on the T2. Apple's own driver for this exact model uses legacy
    ports 0x300/0x304, so port I/O works there.
  - Writes are serialized only by MacHALDriver's mutex. Tools that bang the ports directly would race.
  - Device ACL not checked; assume admin is required.
- **Likely gain:** higher fan floors delay CPU/GPU shared-heatsink throttling. This raises sustained clocks,
  not peak. Actual GPU power limits sit in the AMD PPTable/VBIOS (Falcon's domain) and possibly SMC-enforced
  platform power. No Boot Camp-side cap needs removing. [I]

## Confidence
- High that Boot Camp software imposes no fan, thermal or GPU cap.
- High on the IOCTL layouts.
- Medium on the specific fan/temperature key names and types for the MBP16,1 T2 SMC. Verify on hardware by
  enumerating keys with 0x…45C/0x…460 and reading before writing.
