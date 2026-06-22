# MSI B360M PRO-VDH — OpenCore Hackintosh (macOS Sequoia 15.7.7)

A clean, `ocvalidate`-passing [OpenCore](https://github.com/acidanthera/OpenCorePkg) EFI for an
iGPU-only **Intel Coffee Lake** desktop on the MSI B360M Pro-VDH, targeting **macOS Sequoia 15.7.7**.

The complete, ready-to-flash EFI lives in [`build/EFI/`](build/EFI/).

## Hardware

| Component | Detail |
|-----------|--------|
| **CPU** | Intel Core i3-8100 (Coffee Lake, 4C/4T, locked 3.6 GHz, no Turbo) |
| **iGPU** | Intel UHD 630 (the only graphics — Nvidia dGPU removed) |
| **Motherboard** | MSI B360M Pro-VDH |
| **RAM** | 16 GB DDR4 |
| **Storage** | Kingston NV1 512 GB NVMe |
| **Wi-Fi/BT** | Intel AX210 |
| **Ethernet** | Realtek GbE |
| **Display out** | HDMI / DVI-D (⚠️ no VGA — macOS has no VGA support) |

**SMBIOS:** `Macmini8,1` — the correct identity for an iGPU-only i3-8100B; `iMac19,1` caused
launchd/XCPM reboots on this board.

## What this build does

Everything below is working / configured on this EFI:

- ✅ **Full graphics acceleration** on UHD 630 via a board-proven framebuffer
  (`ig-platform-id 0x3E9B0000` + per-connector data + `enable-hdmi20` + `force-online`) — no black screen at GUI handoff.
- ✅ **CPU power management** without CFG-Lock (disabled in BIOS; `AppleXcpmCfgLock` / `AppleCpuPmCfgLock` both `false`) via `SSDT-PLUG`.
- ✅ **Audio** via AppleALC (`alcid=1`).
- ✅ **Ethernet** via RealtekRTL8111.
- ✅ **Wi-Fi** via `itlwm` + the [HeliPort](https://github.com/OpenIntelWireless/HeliPort) app (AirportItlwm has no Sequoia build).
- ✅ **Bluetooth** via IntelBluetoothFirmware + IntelBTPatcher + BlueToolFixup.
- ✅ **NVMe** support for the Kingston NV1 via NVMeFix.
- ✅ **USB port map** — codeless `USBMap.kext` (15-port UIAC map).
- ✅ **OpenCanopy** graphical boot picker.

## EFI layout

```
build/EFI/
├── BOOT/BOOTx64.efi
└── OC/
    ├── ACPI/        SSDT-PLUG, SSDT-EC-USBX, SSDT-AWAC, SSDT-PMC
    ├── Drivers/     OpenRuntime, OpenCanopy, HfsPlus, ResetNvramEntry
    ├── Kexts/       Lilu, VirtualSMC (+SMCProcessor/SMCSuperIO), WhateverGreen,
    │                AppleALC, RestrictEvents, NVMeFix, RealtekRTL8111, USBMap,
    │                itlwm, IntelBluetoothFirmware, IntelBTPatcher, BlueToolFixup
    ├── config.plist
    └── OpenCore.efi
```

**Boot-args:** `-v keepsyms=1 debug=0x100 alcid=1` · **SIP:** enabled · **SecureBootModel:** Default

## Repo contents

- [`build/EFI/`](build/EFI/) — the assembled, flashable EFI (validated with `ocvalidate`, 0 issues).
- [`build/oc-release/`](build/oc-release/) — the matching OpenCore RELEASE package (Docs + Utilities, incl. `ocvalidate`).
- [`BiosSettings/`](BiosSettings/) — BIOS-settings reference (screenshots) for this board.
- [`docs/`](docs/) — design notes and build plan.

## Install (summary)

1. Create a Sequoia install USB (`softwareupdate --fetch-full-installer` → `createinstallmedia`).
2. Mount the USB's EFI partition and copy in `build/EFI/`.
3. Apply BIOS settings (see [`BiosSettings/`](BiosSettings/)): disable CFG-Lock, enable above-4G decoding, set iGPU as primary, etc.
4. Boot the USB and install macOS.
5. Copy `build/EFI/` to the NVMe's EFI partition to boot without the USB.
6. Install the HeliPort app for Wi-Fi.

> ⚠️ The committed `config.plist` ROM is set to this machine's real NIC MAC. If you fork this for your
> own board, generate a fresh SMBIOS serial and set your own ROM.
