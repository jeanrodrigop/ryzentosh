# Ryzentosh — OpenCore EFI for ASUS TUF Gaming B550M-Plus

Personal, working OpenCore EFI for running macOS on an AMD Ryzen desktop built around the
ASUS TUF Gaming B550M-Plus motherboard. This is the exact configuration used on my own
machine, published as a reference for anyone building a similar AMD hackintosh.

> ⚠️ AMD builds require **kernel patches** (the AMD Vanilla patches) that are specific to
> the CPU core count. This EFI is patched for a **6-core** CPU. If your CPU has a different
> number of cores, the patches **must** be adjusted or macOS will not boot.

## Specification

| **Component**   | **Model**                                        |
| --------------- | ------------------------------------------------ |
| CPU             | AMD Ryzen 5 5600 @ 3.5GHz, 6 Cores / 12 Threads  |
| Motherboard     | ASUS TUF Gaming B550M-Plus (AM4, B550 chipset)   |
| RAM             | 32GB (2 x 16GB) Corsair Vengeance LPX DDR4-3200  |
| GPU             | XFX Radeon RX 5500 XT 8GB (Navi 14)              |
| Audio Chipset   | Realtek ALC1200 (layout-id 7)                    |
| Ethernet        | Realtek RTL8125B 2.5GbE                          |
| OS Disk (NVMe)  | Kingston NV2 1TB                                  |

## Software

| | |
| --- | --- |
| **macOS version**   | 15.x Sequoia |
| **OpenCore version**| 1.0.7 |
| **SMBIOS**          | MacPro7,1 |

`MacPro7,1` is used because it best matches a high-core-count AMD desktop with a discrete
AMD GPU and avoids the power-management / CPU topology assumptions of consumer SMBIOS models.

## Feature status

| Feature | Status | Notes |
| --- | --- | --- |
| GPU acceleration (Metal) | ✅ | Native via `WhateverGreen` + `agdpmod=pikera` for the RX 5500 XT |
| Onboard audio | ✅ | `AppleALC` with `alcid=7` (ALC1200) |
| 2.5GbE Ethernet | ✅ | `LucyRTL8125Ethernet` |
| USB mapping | ✅ | Custom `USBMap.kext` (built for this board) |
| Sleep / Wake | ✅ | SSDT hotpatches + `USBWakeFixup` to prevent instant wake |
| Hibernation | ✅ | `HibernationFixup` |
| CPU power management | ✅ | `SMCAMDProcessor` + `AMDRyzenCPUPowerManagement` |
| Wi-Fi / Bluetooth | ⚠️ | Not configured in this EFI |

## EFI layout

```
EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── ACPI/        # SSDT hotpatches (see below)
    ├── Drivers/     # UEFI drivers
    ├── Kexts/       # Kernel extensions
    ├── Resources/   # OpenCanopy GUI fonts, images, audio
    ├── Tools/
    ├── OpenCore.efi
    └── config.plist
```

### Kexts

Loaded in dependency order (`Lilu` first, then plugins):

| Kext | Version | Purpose |
| --- | --- | --- |
| Lilu | 1.7.2 | Core patching framework (required by most other kexts) |
| VirtualSMC | 1.3.7 | SMC emulation (replaces the Apple SMC chip) |
| WhateverGreen | 1.7.0 | GPU patching for the AMD Radeon RX 5500 XT |
| AppleALC | 1.9.7 | Onboard audio (ALC1200, layout 7) |
| RestrictEvents | 1.1.6 | Hides unsupported hardware warnings / fixes core count reporting |
| LucyRTL8125Ethernet | 1.2.2 | Realtek RTL8125B 2.5GbE driver |
| AMDRyzenCPUPowerManagement | 0.7.2 | Ryzen power-management telemetry |
| SMCAMDProcessor | 1.0.1 | Feeds Ryzen sensor data to VirtualSMC |
| HibernationFixup | 1.5.4 | Fixes hibernation / sleep-image handling |
| AppleMCEReporterDisabler | — | Prevents MCE-related kernel panics on AMD |
| USBMap | 1.0 | Custom USB port mapping for this motherboard |
| USBWakeFixup | 1.0 | Prevents spurious wake-from-sleep |

### ACPI (SSDT hotpatches)

| SSDT | Purpose |
| --- | --- |
| SSDT-CPUR.aml      | Fixes CPU object naming so processors enumerate correctly |
| SSDT-EC.aml        | Embedded controller stub required by macOS |
| SSDT-USBX.aml      | USB power properties |
| SSDT-USBW.aml      | USB wake support |
| SSDT-GPRW.aml      | Renames `GPRW` → `XPRW` to fix instant wake-from-sleep |
| SSDT-SBUS-MCHC.aml | SMBus / memory controller hub devices for App Store / iCloud |

### Drivers

| Driver | Purpose |
| --- | --- |
| OpenRuntime.efi      | Required OpenCore runtime services |
| OpenCanopy.efi       | Graphical boot picker |
| HfsPlus.efi          | HFS+ filesystem support (needed to read macOS recovery) |
| AudioDxe.efi         | Boot-chime / boot audio support |
| ResetNvramEntry.efi  | Adds a "Reset NVRAM" entry to the boot picker |

## Key configuration notes

- **boot-args:** `keepsyms=1 debug=0x100 agdpmod=pikera alcid=7`
  - `agdpmod=pikera` — required for AMD Navi GPUs to fix the black-screen-on-boot issue.
  - `alcid=7` — selects the AppleALC audio layout for the ALC1200.
  - `keepsyms=1 debug=0x100` — keeps symbols and verbose panic info (useful while tuning;
    can be removed for a "release" setup).
- **SIP (System Integrity Protection):** disabled (`csr-active-config = 00000000`).
- **SecureBootModel:** `Disabled` (AMD kernel patches are incompatible with secure boot).
- **PickerMode:** `External` (OpenCanopy graphical picker).
- **AMD Vanilla kernel patches:** the `Kernel → Patch` section contains the standard
  community AMD patches (`AuthenticAMD`, CPUID, core-count, PAT/MTRR fixes). They are
  authored for a **6-core** CPU — change the `cpuid_cores_per_package` value if your CPU
  differs.

## Installation

1. Generate your **own** SMBIOS data (serial, board serial, UUID) with
   [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) and put it under
   `PlatformInfo → Generic` in `config.plist`. **Do not reuse the values in this repo.**
2. Set the BIOS to the recommended hackintosh defaults: disable Secure Boot, CSM, Fast Boot,
   and Resizable BAR; enable Above 4G Decoding; set SVM/IOMMU as needed.
3. Copy the `BOOT` and `OC` folders to the `EFI` folder of a FAT32 ESP partition.
4. Verify your config against the OpenCore reference with
   [ProperTree](https://github.com/corpnewt/ProperTree) /
   [OCValidate](https://github.com/acidanthera/OpenCorePkg).

> If your USB ports behave differently, regenerate `USBMap.kext` for your board using
> [USBMap](https://github.com/corpnewt/USBMap) or `USBToolBox` — port maps are
> hardware-specific.

## Post-install

- **`windows-time-correction.zip`** — registry files (`WinUTCOff.reg` / `WinUTCOn.reg`) to
  fix the clock drift on a Windows dual-boot. macOS stores the RTC in UTC while Windows
  expects local time; importing `WinUTCOn.reg` makes Windows treat the RTC as UTC so both
  OSes agree.

## References & credits

- [OpenCore Install Guide (Dortania)](https://dortania.github.io/OpenCore-Install-Guide/)
- [AMD OSX / AMD Vanilla kernel patches](https://github.com/AMD-OSX/AMD_Vanilla)
- [Acidanthera](https://github.com/acidanthera) — OpenCorePkg, Lilu, VirtualSMC, WhateverGreen, AppleALC and more
- [CorpNewt](https://github.com/corpnewt) — GenSMBIOS, ProperTree, USBMap

## Disclaimer

This documentation is published for the sole purpose of learning and tech enthusiasm, with
no guarantees of any kind. I'm not responsible for any harm or loss of data of any kind that
may occur. Hackintoshing is unsupported by Apple — proceed at your own risk and always keep
backups.
