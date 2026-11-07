# Macs Fan Control settings for gaming (MacBookPro16,1)

The fan preset used on the MacBook Pro 16" (2019, Radeon Pro 5500M) for the BF6 test recorded in
[DESIGN.md](../DESIGN.md) (2026-11-07). Faster fans let the firmware give the CPU and GPU more of their shared power
budget. In that test the CPU went from 1.7 to 2.3 GHz and the GPU power limit from 35 to 50 W.

## Settings

Preset name: `sensor cpu/gpu`. Set each fan with **Custom… → Sensor-based value**:

| Fan | Sensor (SMC key) | Speed starts increasing at | Max temperature |
|---|---|---|---|
| Left | CPU PECI (`TCXC`) | 50 °C | 78 °C |
| Right | AMD Radeon Pro 5500M (`TGDD`) | 50 °C | 70 °C |

Both fans are at full speed during games and quiet on the desktop. Turn on starting with Windows in the app's
preferences so the preset comes back after a reboot. Never set the fans slower than Apple's minimum.

## Import instead of clicking

Macs Fan Control keeps its presets in `HKCU\Software\CrystalIDEA Software\Macs Fan Control`. `CustomPresets` is
base64 of `sensor cpu/gpu|2,TCXC,50,78|2,TGDD,50,70` (one `mode,sensor,start,max` group per fan, left fan first;
mode 2 is sensor-based). Close Macs Fan Control, save this as a `.reg` file and double-click it.
It replaces any custom presets you already have.

```reg
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Software\CrystalIDEA Software\Macs Fan Control]
"ActivePreset"="Custom:0"
"CustomPresets"=hex(7):63,00,32,00,56,00,75,00,63,00,32,00,39,00,79,00,49,00,\
  47,00,4e,00,77,00,64,00,53,00,39,00,6e,00,63,00,48,00,56,00,38,00,4d,00,69,\
  00,78,00,55,00,51,00,31,00,68,00,44,00,4c,00,44,00,55,00,77,00,4c,00,44,00,\
  63,00,34,00,66,00,44,00,49,00,73,00,56,00,45,00,64,00,45,00,52,00,43,00,77,\
  00,31,00,4d,00,43,00,77,00,33,00,4d,00,41,00,3d,00,3d,00,00,00,00,00
```
