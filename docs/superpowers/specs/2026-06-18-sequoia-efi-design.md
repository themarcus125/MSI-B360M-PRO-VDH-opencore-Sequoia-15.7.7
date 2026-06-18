# macOS Sequoia OpenCore EFI — Design Spec

**Date:** 2026-06-18
**Target:** Fresh OpenCore EFI + Sequoia install media + install/post-install for the MSI B360M Pro-VDH Hackintosh
**Approach:** A — fresh Dortania-style build on the latest OpenCore RELEASE, grafting in the board-specific decisions already proven on this exact motherboard.

## Goal

Build a clean, current, minimal OpenCore EFI to install and run **macOS Sequoia (15.x)** on the i3-8100 / UHD 630 system, create the Sequoia install USB, complete the install, and reach a stable iGPU-only desktop that boots from the internal NVMe.

The existing `MSI-B360M-PRO-VDH-OPENCORE/` folder in this repo is an old example and is **not** the basis for this build. We start fresh from the latest OpenCore RELEASE.

## Hardware

| Component | Detail |
|---|---|
| CPU | Intel i3-8100 (Coffee Lake, locked 3.6 GHz, no Turbo) |
| iGPU | Intel UHD 630 (iGPU-only; Nvidia dGPU removed) |
| Motherboard | MSI B360M Pro-VDH (outputs: VGA / DVI-D / HDMI — **use HDMI or DVI only, macOS has no VGA**) |
| RAM | 16 GB DDR4 |
| Storage | Kingston NV1 512 GB NVMe |
| Wi-Fi | Intel AX210 |
| Ethernet | Realtek GbE |

## Key decisions

- **macOS version:** Sequoia 15.x — best stability-to-recency balance for Coffee Lake/UHD 630; mature kext + OpenCore support. (Sonoma 14 = safest fallback; Tahoe 26 = too fragile for iGPU-only.)
- **OpenCore:** latest RELEASE build (1.0.x — pull current at build time).
- **SMBIOS:** `Macmini8,1` (matches i3-8100B, iGPU-only; iMac19,1 previously caused XCPM/launchd reboot).
- **Wi-Fi:** `AirportItlwm` (native menu-bar UI). Version-locked to the macOS build — must swap the matching kext on every major update. No AirDrop/Continuity (Intel card).

## Section 1 — EFI structure

Standard layout: `EFI/BOOT/BOOTx64.efi` + `EFI/OC/` with `OpenCore.efi`, `config.plist`, and `ACPI/`, `Drivers/`, `Kexts/`, `Tools/`, `Resources/`.

### Drivers (`EFI/OC/Drivers/`)
- `OpenRuntime.efi` — required (memory map / NVRAM)
- `HfsPlus.efi` — picker can see HFS+/recovery volumes
- `OpenCanopy.efi` — graphical boot picker (uses `Resources/`)
- `ResetNvramEntry.efi` — reset-NVRAM picker entry

### Kexts (`EFI/OC/Kexts/`) — Lilu loads first
- `Lilu.kext` — patching framework (base)
- `VirtualSMC.kext` + `SMCProcessor.kext` + `SMCSuperIO.kext` — SMC emulation + sensors
- `WhateverGreen.kext` — UHD 630 graphics
- `AppleALC.kext` — onboard audio (ALC892; layout-id via `alcid`)
- `RealtekRTL8111.kext` — Realtek GbE
- `AirportItlwm.kext` — **Sequoia-matched build** for AX210 Wi-Fi
- `IntelBluetoothFirmware.kext` + `IntelBTPatcher.kext` + `BlueToolFixup.kext` — AX210 Bluetooth
- `USBToolBox.kext` + `UTBMap.kext` — USB (UTBMap generated on the machine post-install)
- `NVMeFix.kext` — power management for the Kingston NV1
- `RestrictEvents.kext` — quiets unsupported-hardware noise for Macmini8,1

### ACPI / SSDTs (`EFI/OC/ACPI/`) — Coffee Lake desktop set
- `SSDT-PLUG.aml` — CPU power management (XCPM plugin-type) for the i3-8100
- `SSDT-EC-USBX.aml` — embedded controller + USB power (desktop variant)
- `SSDT-AWAC.aml` — RTC/clock fix required on B360 (AWAC) boards
- `SSDT-PMC.aml` — native NVRAM on 300-series

## Section 2 — config.plist key decisions

Board-specific values are the proven ones; the rest is the standard Coffee Lake desktop baseline.

### PlatformInfo / SMBIOS
- `Macmini8,1`
- Generate fresh `SystemSerialNumber`, `MLB`, `SmUUID` (GenSMBIOS/macserial); set `ROM` = real Ethernet MAC

### Kernel → Quirks
- `AppleXcpmCfgLock = false`, `AppleCpuPmCfgLock = false` — CFG Lock is disabled in BIOS (true previously caused a launchd reset)
- `DisableIoMapper = true` (handle VT-d)
- `PanicNoKextDump = true`, `PowerTimeoutKernelPanic = true`
- `XhciPortLimit = false` (real USB map used)

### Kernel → Patch
- IONVMe DMA-reset patch (`IONVMeController::resetDMA`) for the Kingston NVMe — carried over; **confirm still required under Sequoia** before relying on it.

### DeviceProperties — UHD 630 at `PciRoot(0x0)/Pci(0x2,0x0)`
- `AAPL,ig-platform-id = 0x3E9B0000`
- `framebuffer-con0/1/2-*` connector data + `enable-hdmi20` + `force-online` (the layout that fixed the GUI-handoff black screen)
- HDMI/DVI outputs only

### Booter → Quirks (Coffee Lake desktop)
- `AvoidRuntimeDefrag`, `EnableSafeModeSlide`, `ProvideCustomSlide`, `RebuildAppleMemoryMap`, `SetupVirtualMap`, `SyncRuntimePermissions` = true
- `ResizeAppleGpuBars = -1`

### NVRAM / boot-args
- Install/debug: `-v keepsyms=1 debug=0x100` + `alcid=<layout>`; trim `-v` after install
- `csr-active-config = 00000000` (SIP enabled) for OTA-clean system; relax temporarily only if troubleshooting

### UEFI / Misc
- Drivers per Section 1; `ReleaseUsbOwnership = true`, `RequestBootVarRouting = true`
- `SecureBootModel = Disabled` for install, switch to `Default` once stable
- Picker: `ShowPicker = true`, `Timeout = 5`, OpenCanopy GUI

Every config change is validated with `ocvalidate` before booting.

## Section 3 — Sequoia install media creation

Built on the host Mac via `createinstallmedia`.

1. **Get the full installer** (host may only surface latest in App Store, so use softwareupdate):
   ```
   softwareupdate --list-full-installers
   sudo softwareupdate --fetch-full-installer --full-installer-version 15.x
   ```
   Produces `/Applications/Install macOS Sequoia.app`.
2. **Prepare USB** (16 GB+, erases it): Disk Utility → Show All Devices → select whole device → erase as **Mac OS Extended (Journaled)**, scheme **GUID Partition Map** (e.g. name `USB`).
3. **Write installer:**
   ```
   sudo /Applications/Install\ macOS\ Sequoia.app/Contents/Resources/createinstallmedia --volume /Volumes/USB
   ```
4. **Add EFI:** mount the USB's EFI partition (`diskutil list` to find it, or MountEFI) and copy our `EFI/` folder to its root.

Result: one USB that boots OpenCore and runs the Sequoia installer.

**Plan B:** if the fetched installer refuses to build media, use OpenCore's `macrecovery.py` to grab a recovery image instead.

## Section 4 — BIOS, install, post-install, verification

### BIOS (MSI B360M Pro-VDH — `BiosSettings/` screenshots are the reference)
- Load Optimized Defaults; boot mode UEFI; disable Fast Boot and Secure Boot
- CFG Lock disabled (already set — basis for the Cfg-lock quirks being `false`)
- Enable XHCI Hand-off; enable Above 4G Decoding if present
- iGPU as primary / Multi-Monitor enabled; DVMT pre-alloc 64 MB
- Leave VT-d as-is (handled by `DisableIoMapper = true`); disable Serial/COM if present

### Install flow
1. Boot USB → OpenCore picker → *Install macOS Sequoia*
2. Disk Utility → erase the Kingston NVMe as **APFS / GUID**
3. Run installer (several reboots; pick installer then the new macOS volume each time)
4. Finish macOS setup

### Post-install
- **EFI → NVMe:** mount the NVMe EFI partition, copy our `EFI/` there, set NVMe first in BIOS boot order — boots without the USB
- **USB map:** run USBToolBox in macOS, map real ports, generate `UTBMap.kext`, add to EFI (replaces deprecated SSDT-UIAC/USBInjectAll)
- **Harden:** `SecureBootModel = Default`, remove `-v` from boot-args

### Verification — definition of done
- `ocvalidate` passes on the final config
- Boots to desktop from NVMe alone
- About This Mac: Macmini8,1, correct CPU / 16 GB RAM
- Intel UHD Graphics 630 with full acceleration (smooth UI, correct resolution) over HDMI/DVI
- Audio out works (tuned `alcid`), Ethernet up, Wi-Fi in menu bar, Bluetooth on
- NVMe shows as internal storage
- Optional / deferrable: sleep/wake, iMessage/FaceTime

## Out of scope (for now)
- Sleep/wake tuning, iServices (iMessage/FaceTime) validation — deferred post-stability
- Any dGPU configuration (Nvidia removed; iGPU-only)
