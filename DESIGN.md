# Project Falcon – Design

## Idea

Up-to-date AMD Windows drivers for the AMD GPUs in Macs (Boot Camp): Polaris/Vega, RDNA1/2 and the Radeon Pro 5600M.
Falcon doesn't ship AMD's binaries. The user downloads the official AMD package. Falcon adds a modified INF with the Mac
PCI IDs and a catalog signed with the Project Falcon certificate, plus an older kernel (`amdkmdag.sys`) where the new
one dropped support. `install-driver.cmd` puts all of this in place with one menu choice.

**Current focus:** the best gaming frame rates on the MacBook Pro 16" 2019 (MacBookPro16,1) Radeon Pro 5500M 8GB.

## Open questions

- **Root CA trust.** The cert's private key is public in this repo, and the cert is added to the Root and TrustedPublisher
  stores. That means anyone can sign code that every Falcon user's PC trusts. Should the installer remove it from Root
  after `pnputil` finishes? Would that break driver updates or reinstalls? This needs testing and the upstream dev's input.
- Upstream (icywire/project-falcon) has issues turned off. Changes go to upstream as PRs from the ztancrell fork.

## Decided

- 2026-11-04: Users' PCs get only the public `.cer`, never the `.pfx` private key. The private key is only needed for
  signing, which happens on the maintainer's machine.
- 2026-11-04: The installer stays a readable batch script; it won't be rewritten in Rust (or as any other compiled .exe).
  Run time is all `pnputil` and `xz`. An unsigned admin .exe would get blocked by SmartScreen and antivirus. Users
  should be able to read what they run as admin, and a script is easy for upstream to accept. If batch ever gets too
  limiting, the next step is PowerShell.
- 2026-11-04: Focus is user-friendliness: the installer detects the GPU and the extracted AMD package and recommends the
  menu option to pick.
- 2026-11-07: Falcon stays focused on AMD GPU drivers. Fixing Apple's other Boot Camp drivers (keyboard, trackpad, audio
  and so on) is out of scope: it wouldn't raise frame rates, and patched Apple kernel drivers can't be re-signed so that
  Windows loads them normally (that needs test-signing mode or Microsoft signing). Apple's drivers are studied only for
  limits they put on the GPU.
- 2026-11-07: Keep `PP_Apple_Bootcamp_Enable=1` in the INF (Apple's default; Falcon already sets it). Superseded the open
  question "Apple-side GPU limits". Findings from reverse-engineering Apple's Boot Camp drivers with REA, plus BF6 tests on
  a MacBookPro16,1 (5500M, driver 26.9.2, kernel 32.0.21045.11001):
  - Apple's Boot Camp INF, MacHALDriver, KeyManager and BootCampService set no GPU power, clock or fan limits. The only SMC
    key Apple's Windows software writes is `LKSB` (keyboard backlight). Limits come from the SMC/ACPI firmware and the VBIOS.
  - `PP_EnableLoadFalconSmcFirmware` only affects Pitcairn/Cape Verde GPUs and is a no-op on 32.x kernels.
    `PP_Falcon_QuickTransition_Enable` only goes with AC/DC switching. Apple's 5500M kernel (26.20.13003.5002) has no
    Navi14 power code that current stock kernels lack. The 32.x kernel honours `PP_PhmSoftPowerPlayTable` (MorePowerTool).
  - With `PP_Apple_Bootcamp_Enable=1` the firmware moves the GPU power limit between 30 and 50 W (mostly 35 W) to share one
    power/heat budget with the CPU. Setting it to 0 pins the limit at 50 W, but in CPU-bound BF6 that ran worse: average
    52 vs 58 FPS, slowest 1% of frames 37.5 vs 40, CPU at 1.5 vs 1.7 GHz, GPU hot spot peak 96 vs 93 °C.
  - On this machine the levers are a MorePowerTool undervolt, a higher minimum fan speed (Macs Fan Control writes SMC keys
    through MacHALDriver), and lowering CPU-heavy game settings.
  - Full reports: [docs/apple-bootcamp-smc.md](docs/apple-bootcamp-smc.md) (SMC, fans, GPU power) and
    [docs/apple-bootcamp-kernel.md](docs/apple-bootcamp-kernel.md) (Apple vs Falcon kernel).
