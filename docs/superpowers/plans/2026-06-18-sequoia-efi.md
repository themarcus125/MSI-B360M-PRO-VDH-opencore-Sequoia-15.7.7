# macOS Sequoia OpenCore EFI — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a fresh, ocvalidate-clean OpenCore EFI for macOS Sequoia on the MSI B360M Pro-VDH (i3-8100 / UHD 630, iGPU-only), create the Sequoia install USB, install, and reach a stable desktop that boots from the internal NVMe.

**Architecture:** Assemble a minimal OpenCore EFI from the latest RELEASE on the host Mac, configure `config.plist` per the Dortania Coffee Lake desktop baseline, and graft in four board-specific decisions proven on this exact board (SMBIOS `Macmini8,1`, CFG-lock quirks `false`, UHD 630 connector data, IONVMe DMA-reset patch). Then build the install USB with `createinstallmedia`, install, and migrate the EFI to the NVMe.

**Tech Stack:** OpenCorePkg (latest RELEASE), Acidanthera kexts (Lilu/VirtualSMC/WhateverGreen/AppleALC/RestrictEvents/NVMeFix/Bluetooth), AirportItlwm, RealtekRTL8111, USBToolBox, GenSMBIOS/macserial, `ocvalidate`, `/usr/libexec/PlistBuddy`, `createinstallmedia`, macOS Sequoia 15.x.

## Global Constraints

- **macOS target:** Sequoia 15.x (exact latest 15.x chosen from `softwareupdate --list-full-installers` at build time).
- **OpenCore:** latest RELEASE build (1.0.x) — same OpenCore version's `ocvalidate` MUST be used to validate the `config.plist`.
- **SMBIOS:** `Macmini8,1` only (never iMac19,1 — caused XCPM/launchd reboot on this board).
- **CFG-lock quirks:** `AppleXcpmCfgLock = false` and `AppleCpuPmCfgLock = false` (CFG Lock is disabled in BIOS; `true` caused a launchd reset).
- **Display:** HDMI or DVI-D only — macOS has no VGA support on this board.
- **Graphics:** iGPU-only (Nvidia dGPU removed).
- **Validation gate:** every `config.plist` change is followed by `ocvalidate`; never boot an EFI that fails `ocvalidate`.
- Build artifacts live under `/Users/khoango/Desktop/hackintosh/build/`. The old `MSI-B360M-PRO-VDH-OPENCORE/` folder is reference-only, never the base.

---

## Phase 1 — Repo & component gathering (host Mac)

### Task 1: Initialize git repo and build scaffold

**Files:**
- Create: `/Users/khoango/Desktop/hackintosh/.gitignore`
- Create: `/Users/khoango/Desktop/hackintosh/build/` (directory tree)

**Interfaces:**
- Produces: build tree at `build/downloads/` (raw archives) and `build/EFI/` (assembled EFI).

- [ ] **Step 1: Initialize git** (the design spec is currently uncommitted)

```bash
cd /Users/khoango/Desktop/hackintosh
git init
```

- [ ] **Step 2: Create .gitignore** — exclude large/secret artifacts

```bash
cat > /Users/khoango/Desktop/hackintosh/.gitignore <<'EOF'
build/downloads/
*.dmg
*.zip
.DS_Store
MSI-B360M-PRO-VDH-OPENCORE/.git/
EOF
```

- [ ] **Step 3: Create the build tree**

```bash
mkdir -p /Users/khoango/Desktop/hackintosh/build/downloads
mkdir -p /Users/khoango/Desktop/hackintosh/build/EFI
```

- [ ] **Step 4: Verify**

Run: `ls -la /Users/khoango/Desktop/hackintosh/build && git -C /Users/khoango/Desktop/hackintosh status`
Expected: `downloads/` and `EFI/` exist; git shows untracked `docs/` and `.gitignore`.

- [ ] **Step 5: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add .gitignore docs/
git commit -m "chore: init repo, build scaffold, and Sequoia EFI design + plan"
```

---

### Task 2: Download OpenCore RELEASE and extract the EFI skeleton

**Files:**
- Create: `build/downloads/OpenCore-RELEASE.zip`
- Create: `build/EFI/BOOT/`, `build/EFI/OC/` (from the OpenCore sample)

**Interfaces:**
- Produces: `build/oc-release/` (unzipped OpenCore), `build/oc-release/Utilities/ocvalidate/ocvalidate`, and a base `build/EFI/` skeleton with `OpenCore.efi`, `BOOTx64.efi`, `Docs/Sample.plist`.

- [ ] **Step 1: Fetch the latest OpenCore RELEASE asset URL**

```bash
cd /Users/khoango/Desktop/hackintosh/build/downloads
curl -s https://api.github.com/repos/acidanthera/OpenCorePkg/releases/latest \
  | grep browser_download_url | grep -i 'RELEASE.zip' | cut -d'"' -f4
```
Expected: one URL ending in `OpenCore-<version>-RELEASE.zip`. Note the `<version>` — it is the OpenCore version for the whole build.

- [ ] **Step 2: Download and unzip**

```bash
cd /Users/khoango/Desktop/hackintosh/build/downloads
URL=$(curl -s https://api.github.com/repos/acidanthera/OpenCorePkg/releases/latest | grep browser_download_url | grep -i 'RELEASE.zip' | cut -d'"' -f4)
curl -L "$URL" -o OpenCore-RELEASE.zip
rm -rf ../oc-release && mkdir -p ../oc-release
unzip -q OpenCore-RELEASE.zip -d ../oc-release
```

- [ ] **Step 3: Seed the EFI skeleton from the X64 sample**

```bash
cd /Users/khoango/Desktop/hackintosh/build
rm -rf EFI && cp -R oc-release/X64/EFI ./EFI
cp oc-release/Docs/Sample.plist EFI/OC/config.plist
```

- [ ] **Step 4: Verify skeleton + ocvalidate binary present**

Run:
```bash
ls /Users/khoango/Desktop/hackintosh/build/EFI/OC/OpenCore.efi \
   /Users/khoango/Desktop/hackintosh/build/EFI/BOOT/BOOTx64.efi \
   /Users/khoango/Desktop/hackintosh/build/oc-release/Utilities/ocvalidate/ocvalidate
```
Expected: all three paths exist.

- [ ] **Step 5: Strip unused sample kexts/ACPI/drivers from the skeleton**

```bash
cd /Users/khoango/Desktop/hackintosh/build/EFI/OC
rm -f ACPI/*.aml
rm -rf Kexts/* Tools/*
# keep only the drivers we use; remove the rest
cd Drivers && ls | grep -vE '^(OpenRuntime.efi|OpenCanopy.efi|ResetNvramEntry.efi|HfsPlus.efi)$' | xargs -I{} rm -f {} ; cd ..
```
Note: `HfsPlus.efi` is NOT in the OpenCore package. It is added in Task 5.

- [ ] **Step 6: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/EFI
git commit -m "feat: seed minimal EFI skeleton from OpenCore RELEASE"
```

---

### Task 3: Download the kexts

**Files:**
- Create: `build/EFI/OC/Kexts/<each>.kext`

**Interfaces:**
- Consumes: `build/EFI/OC/Kexts/` (from Task 2).
- Produces: all release kexts in place: `Lilu`, `VirtualSMC` (+ `SMCProcessor`, `SMCSuperIO`), `WhateverGreen`, `AppleALC`, `RestrictEvents`, `NVMeFix`, `RealtekRTL8111`, `IntelBluetoothFirmware`, `IntelBTPatcher`, `BlueToolFixup`. (AirportItlwm + USBToolBox handled in Tasks 4 and 18.)

- [ ] **Step 1: Define a helper to fetch the latest RELEASE asset from a repo**

```bash
cd /Users/khoango/Desktop/hackintosh/build/downloads
getrelease() { # $1=owner/repo  $2=grep-pattern for asset
  curl -s "https://api.github.com/repos/$1/releases/latest" \
    | grep browser_download_url | grep -iE "$2" | grep -i 'RELEASE' | head -1 | cut -d'"' -f4
}
```

- [ ] **Step 2: Download Acidanthera kext bundles** (each zip contains the .kext)

```bash
cd /Users/khoango/Desktop/hackintosh/build/downloads
for repo in acidanthera/Lilu acidanthera/VirtualSMC acidanthera/WhateverGreen \
            acidanthera/AppleALC acidanthera/RestrictEvents acidanthera/NVMeFix \
            acidanthera/IntelBluetoothFirmware; do
  url=$(getrelease "$repo" '\.zip'); echo "$repo -> $url"
  curl -L "$url" -o "$(basename $repo).zip"
done
# Realtek ethernet
url=$(getrelease "Mieze/RTL8111_driver_for_OS_X" '\.zip'); curl -L "$url" -o RealtekRTL8111.zip
```
Expected: each `echo` prints a non-empty URL and each `.zip` downloads.

- [ ] **Step 3: Extract the needed kexts into the EFI**

```bash
cd /Users/khoango/Desktop/hackintosh/build/downloads
K=/Users/khoango/Desktop/hackintosh/build/EFI/OC/Kexts
tmp=$(mktemp -d)
for z in Lilu VirtualSMC WhateverGreen AppleALC RestrictEvents NVMeFix IntelBluetoothFirmware RealtekRTL8111; do
  rm -rf "$tmp/$z" && mkdir -p "$tmp/$z" && unzip -q "$z.zip" -d "$tmp/$z"
done
cp -R "$tmp/Lilu/Lilu.kext" "$K/"
cp -R "$tmp/VirtualSMC/Kexts/VirtualSMC.kext" "$K/"
cp -R "$tmp/VirtualSMC/Kexts/SMCProcessor.kext" "$K/"
cp -R "$tmp/VirtualSMC/Kexts/SMCSuperIO.kext" "$K/"
cp -R "$tmp/WhateverGreen/WhateverGreen.kext" "$K/"
cp -R "$tmp/AppleALC/AppleALC.kext" "$K/"
cp -R "$tmp/RestrictEvents/RestrictEvents.kext" "$K/"
cp -R "$tmp/NVMeFix/NVMeFix.kext" "$K/"
cp -R "$tmp/IntelBluetoothFirmware/IntelBluetoothFirmware.kext" "$K/"
cp -R "$tmp/IntelBluetoothFirmware/IntelBTPatcher.kext" "$K/"
cp -R "$tmp/IntelBluetoothFirmware/BlueToolFixup.kext" "$K/"
cp -R "$tmp/RealtekRTL8111/RealtekRTL8111.kext" "$K/" 2>/dev/null || \
  find "$tmp/RealtekRTL8111" -name 'RealtekRTL8111.kext' -maxdepth 4 -exec cp -R {} "$K/" \;
```

- [ ] **Step 4: Verify all kexts present and have a binary**

Run:
```bash
cd /Users/khoango/Desktop/hackintosh/build/EFI/OC/Kexts
for k in Lilu VirtualSMC SMCProcessor SMCSuperIO WhateverGreen AppleALC RestrictEvents NVMeFix IntelBluetoothFirmware IntelBTPatcher BlueToolFixup RealtekRTL8111; do
  test -f "$k.kext/Contents/MacOS/$k" && echo "OK $k" || echo "MISSING $k"
done
```
Expected: every line prints `OK`. (Lilu must load first; bundle paths matter — order is set in config.plist Task 7-equivalent / ACPI+Kernel task.)

- [ ] **Step 5: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/EFI/OC/Kexts
git commit -m "feat: add Acidanthera + Realtek kexts to EFI"
```

---

### Task 4: Download AirportItlwm (Sequoia build) and add HfsPlus driver

**Files:**
- Create: `build/EFI/OC/Kexts/AirportItlwm.kext`
- Create: `build/EFI/OC/Drivers/HfsPlus.efi`

**Interfaces:**
- Consumes: Kexts/ and Drivers/ dirs.
- Produces: AirportItlwm.kext (Sequoia-matched) and HfsPlus.efi present.

- [ ] **Step 1: Download the AirportItlwm release zip**

```bash
cd /Users/khoango/Desktop/hackintosh/build/downloads
curl -s https://api.github.com/repos/OpenIntlWifi/OpenIntlWifi/releases/latest \
  | grep browser_download_url | grep -i 'itlwm' | cut -d'"' -f4
```
If no asset, use the canonical source: `https://github.com/zxystd/itlwm/releases` (download the latest `AirportItlwm` zip). Save as `AirportItlwm.zip`.

- [ ] **Step 2: Extract the Sequoia (15.x) variant**

```bash
cd /Users/khoango/Desktop/hackintosh/build/downloads
rm -rf itlwm-tmp && mkdir itlwm-tmp && unzip -q AirportItlwm.zip -d itlwm-tmp
# the zip contains per-OS folders; pick the Sequoia one
find itlwm-tmp -ipath '*equoia*' -name 'AirportItlwm.kext' -maxdepth 6
```
Expected: a path under a `Sequoia` / `15` folder. Copy it:
```bash
SRC=$(find /Users/khoango/Desktop/hackintosh/build/downloads/itlwm-tmp -ipath '*equoia*' -name 'AirportItlwm.kext' -maxdepth 6 | head -1)
cp -R "$SRC" /Users/khoango/Desktop/hackintosh/build/EFI/OC/Kexts/AirportItlwm.kext
```

- [ ] **Step 3: Add HfsPlus.efi** (from the well-known OcBinaryData repo)

```bash
cd /Users/khoango/Desktop/hackintosh/build/downloads
curl -L https://github.com/acidanthera/OcBinaryData/raw/master/Drivers/HfsPlus.efi \
  -o /Users/khoango/Desktop/hackintosh/build/EFI/OC/Drivers/HfsPlus.efi
```

- [ ] **Step 4: Add OpenCanopy Resources** (icons/audio for the GUI picker)

```bash
cd /Users/khoango/Desktop/hackintosh/build/downloads
rm -rf OcBinaryData && git clone --depth 1 https://github.com/acidanthera/OcBinaryData.git
cp -R OcBinaryData/Resources/* /Users/khoango/Desktop/hackintosh/build/EFI/OC/Resources/
```

- [ ] **Step 5: Verify**

Run:
```bash
ls /Users/khoango/Desktop/hackintosh/build/EFI/OC/Kexts/AirportItlwm.kext/Contents/MacOS/AirportItlwm \
   /Users/khoango/Desktop/hackintosh/build/EFI/OC/Drivers/HfsPlus.efi
ls /Users/khoango/Desktop/hackintosh/build/EFI/OC/Resources/Image | head -1
```
Expected: AirportItlwm binary + HfsPlus.efi exist; Resources/Image has content.

- [ ] **Step 6: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/EFI/OC/Kexts/AirportItlwm.kext build/EFI/OC/Drivers/HfsPlus.efi build/EFI/OC/Resources
git commit -m "feat: add AirportItlwm (Sequoia), HfsPlus, OpenCanopy resources"
```

---

### Task 5: Add the Coffee Lake desktop SSDTs

**Files:**
- Create: `build/EFI/OC/ACPI/SSDT-PLUG.aml`, `SSDT-EC-USBX.aml`, `SSDT-AWAC.aml`, `SSDT-PMC.aml`

**Interfaces:**
- Consumes: `build/EFI/OC/ACPI/`.
- Produces: four prebuilt `.aml` files.

- [ ] **Step 1: Download prebuilt SSDTs from Dortania's Getting-Started-With-ACPI repo**

```bash
cd /Users/khoango/Desktop/hackintosh/build/EFI/OC/ACPI
BASE=https://github.com/dortania/Getting-Started-With-ACPI/raw/master/extra-files/compiled
curl -L "$BASE/SSDT-PLUG.aml"     -o SSDT-PLUG.aml
curl -L "$BASE/SSDT-EC-USBX-desktop.aml" -o SSDT-EC-USBX.aml
curl -L "$BASE/SSDT-AWAC.aml"     -o SSDT-AWAC.aml
curl -L "$BASE/SSDT-PMC.aml"      -o SSDT-PMC.aml
```

- [ ] **Step 2: Verify they are real AML (start with the `SSDT` signature)**

Run:
```bash
cd /Users/khoango/Desktop/hackintosh/build/EFI/OC/ACPI
for f in SSDT-PLUG SSDT-EC-USBX SSDT-AWAC SSDT-PMC; do
  head -c 4 "$f.aml" | grep -q SSDT && echo "OK $f" || echo "BAD $f"
done
```
Expected: four `OK` lines. (A `BAD` line means an HTML error page was downloaded — re-fetch.)

- [ ] **Step 3: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/EFI/OC/ACPI
git commit -m "feat: add Coffee Lake desktop SSDTs (PLUG, EC-USBX, AWAC, PMC)"
```

---

### Task 6: Retrieve the proven board-specific values (iGPU framebuffer + NVMe patch)

**Files:**
- Create: `build/reference/proven-deviceproperties.txt`, `build/reference/proven-nvme-patch.txt`

**Interfaces:**
- Produces: the exact `framebuffer-con0/1/2-*` byte values, `enable-hdmi20`, `force-online`, and the `IONVMeController::resetDMA` Find/Replace/Mask bytes, captured to text files for Tasks 9 and 8.

> **Why this task exists:** memory records these were copied from your known-good "EFI 0.6.0" reference, which is NOT in this repo. We must source the exact bytes. Order of preference: (a) your real prior working EFI, (b) the Linaro1985 community EFI already in this repo's example git history (same board), (c) standard WhateverGreen defaults.

- [ ] **Step 1: Try to locate your prior working EFI** (USB stick, Time Machine, or the hackintosh's ESP)

```bash
mkdir -p /Users/khoango/Desktop/hackintosh/build/reference
find /Volumes -iname config.plist 2>/dev/null
```
If a prior `config.plist` is found, set `PRIOR=/path/to/that/config.plist` and skip to Step 3.

- [ ] **Step 2: Fallback — extract the same-board community config from the example repo's git history**

```bash
cd /Users/khoango/Desktop/hackintosh/MSI-B360M-PRO-VDH-OPENCORE
git -c safe.directory='*' show HEAD:EFI/OC/config.plist > /tmp/ref-config.plist 2>/dev/null \
  && echo "extracted community config" || echo "no config in history — use WhateverGreen defaults"
PRIOR=/tmp/ref-config.plist
```

- [ ] **Step 3: Dump the DeviceProperties block for the iGPU**

```bash
/usr/libexec/PlistBuddy -c "Print :DeviceProperties:Add:PciRoot(0x0)/Pci(0x2,0x0)" "$PRIOR" \
  | tee /Users/khoango/Desktop/hackintosh/build/reference/proven-deviceproperties.txt
```
Expected: a block listing `AAPL,ig-platform-id`, `framebuffer-con0-*`, `framebuffer-con1-*`, `framebuffer-con2-*`, `enable-hdmi20`, `force-online`. Record these exact values. If the fallback config lacks framebuffer-con entries, record only `AAPL,ig-platform-id=0x3E9B0000` and note that custom connectors will be added only if a black screen recurs (Task 16 contingency).

- [ ] **Step 4: Dump the NVMe kernel patch, if present**

```bash
/usr/libexec/PlistBuddy -c "Print :Kernel:Patch" "$PRIOR" 2>/dev/null \
  | tee /Users/khoango/Desktop/hackintosh/build/reference/proven-nvme-patch.txt
grep -i nvme -A8 /Users/khoango/Desktop/hackintosh/build/reference/proven-nvme-patch.txt || \
  echo "No NVMe patch in reference — confirm whether Sequoia still needs IONVMeController::resetDMA"
```

- [ ] **Step 5: Commit the captured reference values**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/reference
git commit -m "docs: capture proven iGPU framebuffer + NVMe patch reference values"
```

---

## Phase 2 — config.plist construction (host Mac)

All edits use `/usr/libexec/PlistBuddy` against `build/EFI/OC/config.plist`. After Phase 2, Task 13 runs `ocvalidate`.

### Task 7: PlatformInfo / SMBIOS (`Macmini8,1` + generated identity)

**Files:**
- Modify: `build/EFI/OC/config.plist` (`PlatformInfo`)

**Interfaces:**
- Consumes: the sample `config.plist`.
- Produces: `PlatformInfo:Generic` populated with `Macmini8,1`, a generated serial/MLB/UUID, and `ROM` = real Ethernet MAC.

- [ ] **Step 1: Download GenSMBIOS / macserial and generate an identity**

```bash
cd /Users/khoango/Desktop/hackintosh/build/downloads
rm -rf GenSMBIOS && git clone --depth 1 https://github.com/corpnewt/GenSMBIOS.git
MAC=$(find GenSMBIOS -name macserial.command -o -name macserial | head -1)
chmod +x GenSMBIOS/Scripts/macserial 2>/dev/null || true
GenSMBIOS/Scripts/macserial -m Macmini8,1 -n 1
```
Expected: one line `SERIAL | MLB`. Record both. Generate a UUID: `uuidgen`.

- [ ] **Step 2: Set PlatformInfo fields** (substitute the generated values)

```bash
P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
SERIAL=PASTE_SERIAL; MLB=PASTE_MLB; UUID=$(uuidgen)
/usr/libexec/PlistBuddy -c "Set :PlatformInfo:Generic:SystemProductName Macmini8,1" "$P"
/usr/libexec/PlistBuddy -c "Set :PlatformInfo:Generic:SystemSerialNumber $SERIAL" "$P"
/usr/libexec/PlistBuddy -c "Set :PlatformInfo:Generic:MLB $MLB" "$P"
/usr/libexec/PlistBuddy -c "Set :PlatformInfo:Generic:SystemUUID $UUID" "$P"
/usr/libexec/PlistBuddy -c "Set :PlatformInfo:Generic:ROM 001122334455" "$P"   # placeholder; replaced in Step 3
/usr/libexec/PlistBuddy -c "Set :PlatformInfo:Automatic true" "$P"
/usr/libexec/PlistBuddy -c "Set :PlatformInfo:UpdateSMBIOS true" "$P"
```

- [ ] **Step 3: Set ROM to the real Ethernet MAC** (read it after first boot if unknown now; for now use the board's NIC MAC if available)

Note: `ROM` should be the lowercase, no-colon hex of the primary NIC MAC, e.g. `1c1b0d112233`. If unknown until macOS boots, leave the placeholder and update in Task 19. Record this as a known follow-up.

- [ ] **Step 4: Verify**

Run: `/usr/libexec/PlistBuddy -c "Print :PlatformInfo:Generic" /Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist`
Expected: `SystemProductName = Macmini8,1`, non-empty serial/MLB/UUID.

- [ ] **Step 5: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/EFI/OC/config.plist
git commit -m "feat(config): set Macmini8,1 SMBIOS + generated identity"
```

---

### Task 8: ACPI add-list, Kernel quirks, and the NVMe patch

**Files:**
- Modify: `build/EFI/OC/config.plist` (`ACPI:Add`, `Kernel:Add`, `Kernel:Quirks`, `Kernel:Patch`)

**Interfaces:**
- Consumes: SSDTs (Task 5), kexts (Tasks 3–4), NVMe patch reference (Task 6).
- Produces: each SSDT registered, each kext registered in load order (Lilu first), CFG-lock quirks `false`, and the IONVMe patch (if confirmed needed).

- [ ] **Step 1: Register the four SSDTs** (each as an `ACPI:Add` dict with `Path` + `Enabled=true`)

```bash
P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
for s in SSDT-PLUG.aml SSDT-EC-USBX.aml SSDT-AWAC.aml SSDT-PMC.aml; do
  i=$(/usr/libexec/PlistBuddy -c "Print :ACPI:Add" "$P" | grep -c "Dict {")
  /usr/libexec/PlistBuddy -c "Add :ACPI:Add:$i dict" "$P"
  /usr/libexec/PlistBuddy -c "Add :ACPI:Add:$i:Path string $s" "$P"
  /usr/libexec/PlistBuddy -c "Add :ACPI:Add:$i:Comment string $s" "$P"
  /usr/libexec/PlistBuddy -c "Add :ACPI:Add:$i:Enabled bool true" "$P"
done
```

- [ ] **Step 2: Register kexts in load order** (Lilu → VirtualSMC + plugins → WhateverGreen → AppleALC → RestrictEvents → NVMeFix → RealtekRTL8111 → AirportItlwm → IntelBluetoothFirmware → IntelBTPatcher → BlueToolFixup)

For each kext add an entry with `BundlePath`, `ExecutablePath`, `PlistPath`, `Enabled=true`:
```bash
addkext() { P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
  i=$(/usr/libexec/PlistBuddy -c "Print :Kernel:Add" "$P" | grep -c "Dict {")
  /usr/libexec/PlistBuddy -c "Add :Kernel:Add:$i dict" "$P"
  /usr/libexec/PlistBuddy -c "Add :Kernel:Add:$i:BundlePath string $1.kext" "$P"
  /usr/libexec/PlistBuddy -c "Add :Kernel:Add:$i:ExecutablePath string Contents/MacOS/$1" "$P"
  /usr/libexec/PlistBuddy -c "Add :Kernel:Add:$i:PlistPath string Contents/Info.plist" "$P"
  /usr/libexec/PlistBuddy -c "Add :Kernel:Add:$i:Enabled bool true" "$P"
  /usr/libexec/PlistBuddy -c "Add :Kernel:Add:$i:Comment string $1" "$P"
  /usr/libexec/PlistBuddy -c "Add :Kernel:Add:$i:MinKernel string '' " "$P"
  /usr/libexec/PlistBuddy -c "Add :Kernel:Add:$i:MaxKernel string '' " "$P"
}
for k in Lilu VirtualSMC SMCProcessor SMCSuperIO WhateverGreen AppleALC RestrictEvents NVMeFix RealtekRTL8111 AirportItlwm IntelBluetoothFirmware IntelBTPatcher BlueToolFixup; do addkext "$k"; done
```
Note: `BlueToolFixup` lives inside the IntelBluetoothFirmware bundle download; ensure its `BundlePath` is just `BlueToolFixup.kext` (it was copied flat in Task 3).

- [ ] **Step 3: Set Kernel quirks** (the proven CFG-lock values + desktop defaults)

```bash
P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
/usr/libexec/PlistBuddy -c "Set :Kernel:Quirks:AppleXcpmCfgLock false" "$P"
/usr/libexec/PlistBuddy -c "Set :Kernel:Quirks:AppleCpuPmCfgLock false" "$P"
/usr/libexec/PlistBuddy -c "Set :Kernel:Quirks:DisableIoMapper true" "$P"
/usr/libexec/PlistBuddy -c "Set :Kernel:Quirks:PanicNoKextDump true" "$P"
/usr/libexec/PlistBuddy -c "Set :Kernel:Quirks:PowerTimeoutKernelPanic true" "$P"
/usr/libexec/PlistBuddy -c "Set :Kernel:Quirks:XhciPortLimit false" "$P"
```

- [ ] **Step 4: Add the IONVMe DMA-reset patch ONLY if Task 6 confirmed it** (carry the exact Find/Replace/Mask from `build/reference/proven-nvme-patch.txt`)

If confirmed needed, add a `Kernel:Patch` entry with the exact base64 `Find`/`Replace`/`Mask`, `Identifier=com.apple.iokit.IONVMeFamily`, `Enabled=true`. If Task 6 found no patch and the reference didn't use one, SKIP this step and note: "NVMe patch deferred — only add if the Kingston NV1 fails to mount during install (Task 16 contingency)."

- [ ] **Step 5: Verify**

Run:
```bash
P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
/usr/libexec/PlistBuddy -c "Print :ACPI:Add" "$P" | grep -c Path
/usr/libexec/PlistBuddy -c "Print :Kernel:Add" "$P" | grep -c BundlePath
/usr/libexec/PlistBuddy -c "Print :Kernel:Quirks:AppleXcpmCfgLock" "$P"
```
Expected: `4` SSDT paths, `13` kext bundles, `AppleXcpmCfgLock = false`.

- [ ] **Step 6: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/EFI/OC/config.plist
git commit -m "feat(config): register SSDTs+kexts, set CFG-lock quirks, NVMe patch"
```

---

### Task 9: DeviceProperties — UHD 630 framebuffer

**Files:**
- Modify: `build/EFI/OC/config.plist` (`DeviceProperties:Add`)

**Interfaces:**
- Consumes: proven values from `build/reference/proven-deviceproperties.txt` (Task 6).
- Produces: `PciRoot(0x0)/Pci(0x2,0x0)` populated with `AAPL,ig-platform-id` and (if used) the framebuffer-con data.

- [ ] **Step 1: Create the iGPU device path and set ig-platform-id**

```bash
P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
DEV="PciRoot(0x0)/Pci(0x2,0x0)"
/usr/libexec/PlistBuddy -c "Add :DeviceProperties:Add:$DEV dict" "$P"
/usr/libexec/PlistBuddy -c "Add :DeviceProperties:Add:$DEV:AAPL,ig-platform-id data 00009B3E" "$P"
```
Note: `0x3E9B0000` stored little-endian as the data bytes `00 00 9B 3E`.

- [ ] **Step 2: Add the proven connector data IF Task 6 captured it** (exact bytes from the reference file)

For each of `framebuffer-con0-*`, `framebuffer-con1-*`, `framebuffer-con2-*`, `enable-hdmi20`, `force-online`, `framebuffer-patch-enable`, add a `data` entry with the exact value from `proven-deviceproperties.txt`, e.g.:
```bash
# Example shape — replace HEXBYTES with the exact captured value:
/usr/libexec/PlistBuddy -c "Add :DeviceProperties:Add:$DEV:framebuffer-patch-enable data 01000000" "$P"
# /usr/libexec/PlistBuddy -c "Add :DeviceProperties:Add:$DEV:framebuffer-con0-enable data 01000000" "$P"
# ...repeat for each captured con0/1/2 key with its exact bytes...
```
If Task 6 found NO connector data (fallback path), SKIP this step — boot with `ig-platform-id` only, and treat custom connectors as the Task 16 black-screen contingency.

- [ ] **Step 3: Verify**

Run: `/usr/libexec/PlistBuddy -c "Print :DeviceProperties:Add:PciRoot(0x0)/Pci(0x2,0x0)" /Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist`
Expected: `AAPL,ig-platform-id` present; any captured framebuffer keys present.

- [ ] **Step 4: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/EFI/OC/config.plist
git commit -m "feat(config): UHD 630 ig-platform-id + framebuffer connectors"
```

---

### Task 10: Booter quirks + NVRAM / boot-args

**Files:**
- Modify: `build/EFI/OC/config.plist` (`Booter:Quirks`, `NVRAM`)

**Interfaces:**
- Produces: Coffee Lake desktop booter quirks, install-time boot-args, SIP enabled, `alcid` set for audio.

- [ ] **Step 1: Set Booter quirks**

```bash
P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
for q in AvoidRuntimeDefrag EnableSafeModeSlide ProvideCustomSlide RebuildAppleMemoryMap SetupVirtualMap SyncRuntimePermissions; do
  /usr/libexec/PlistBuddy -c "Set :Booter:Quirks:$q true" "$P"
done
/usr/libexec/PlistBuddy -c "Set :Booter:Quirks:ResizeAppleGpuBars integer -1" "$P"
```

- [ ] **Step 2: Set boot-args + csr-active-config** (install/debug args; `alcid=1` as a starting audio layout)

```bash
P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
BA="NVRAM:Add:7C436110-AB2A-4BBB-A880-FE41995C9F82"
/usr/libexec/PlistBuddy -c "Set :$BA:boot-args -v keepsyms=1 debug=0x100 alcid=1" "$P"
/usr/libexec/PlistBuddy -c "Set :$BA:csr-active-config data 00000000" "$P"
/usr/libexec/PlistBuddy -c "Set :NVRAM:WriteFlash true" "$P"
```

- [ ] **Step 3: Verify**

Run:
```bash
P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
/usr/libexec/PlistBuddy -c "Print :Booter:Quirks:RebuildAppleMemoryMap" "$P"
/usr/libexec/PlistBuddy -c "Print :NVRAM:Add:7C436110-AB2A-4BBB-A880-FE41995C9F82:boot-args" "$P"
```
Expected: `true`; boot-args string includes `-v ... alcid=1`.

- [ ] **Step 4: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/EFI/OC/config.plist
git commit -m "feat(config): booter quirks + install boot-args + SIP on"
```

---

### Task 11: UEFI drivers, Misc/Security, and picker

**Files:**
- Modify: `build/EFI/OC/config.plist` (`UEFI:Drivers`, `UEFI:Quirks`, `Misc:Security`, `Misc:Boot`)

**Interfaces:**
- Produces: drivers registered, USB ownership/boot-var quirks set, `SecureBootModel=Disabled`, `ScanPolicy=0`, OpenCanopy picker enabled, `Vault=Optional`.

- [ ] **Step 1: Register UEFI drivers in order**

```bash
P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
# clear then add
while /usr/libexec/PlistBuddy -c "Print :UEFI:Drivers:0" "$P" >/dev/null 2>&1; do /usr/libexec/PlistBuddy -c "Delete :UEFI:Drivers:0" "$P"; done
i=0; for d in OpenRuntime.efi OpenCanopy.efi ResetNvramEntry.efi HfsPlus.efi; do
  /usr/libexec/PlistBuddy -c "Add :UEFI:Drivers:$i dict" "$P"
  /usr/libexec/PlistBuddy -c "Add :UEFI:Drivers:$i:Path string $d" "$P"
  /usr/libexec/PlistBuddy -c "Add :UEFI:Drivers:$i:Enabled bool true" "$P"
  i=$((i+1)); done
```
(If the sample uses a plain string array for `UEFI:Drivers` instead of dicts, add each as `string $d` — check `Print :UEFI:Drivers` first and match the existing element type.)

- [ ] **Step 2: UEFI quirks + Misc security + picker**

```bash
P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
/usr/libexec/PlistBuddy -c "Set :UEFI:Quirks:ReleaseUsbOwnership true" "$P"
/usr/libexec/PlistBuddy -c "Set :UEFI:Quirks:RequestBootVarRouting true" "$P"
/usr/libexec/PlistBuddy -c "Set :Misc:Security:SecureBootModel Disabled" "$P"
/usr/libexec/PlistBuddy -c "Set :Misc:Security:Vault Optional" "$P"
/usr/libexec/PlistBuddy -c "Set :Misc:Security:ScanPolicy integer 0" "$P"
/usr/libexec/PlistBuddy -c "Set :Misc:Boot:ShowPicker true" "$P"
/usr/libexec/PlistBuddy -c "Set :Misc:Boot:Timeout integer 5" "$P"
/usr/libexec/PlistBuddy -c "Set :Misc:Boot:PickerMode String External" "$P"
/usr/libexec/PlistBuddy -c "Set :Misc:Boot:PickerVariant String Auto" "$P"
```

- [ ] **Step 3: Verify**

Run:
```bash
P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
/usr/libexec/PlistBuddy -c "Print :UEFI:Drivers" "$P"
/usr/libexec/PlistBuddy -c "Print :Misc:Security:SecureBootModel" "$P"
```
Expected: 4 drivers listed; `SecureBootModel = Disabled`.

- [ ] **Step 4: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/EFI/OC/config.plist
git commit -m "feat(config): UEFI drivers, security, OpenCanopy picker"
```

---

### Task 12: Lint and validate the config

**Files:**
- Modify: `build/EFI/OC/config.plist` (fixes from validation)

**Interfaces:**
- Consumes: full assembled EFI + matching `ocvalidate`.
- Produces: a `config.plist` that passes `plutil` and `ocvalidate` with zero errors.

- [ ] **Step 1: Lint the plist syntax**

Run: `plutil -lint /Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist`
Expected: `OK`.

- [ ] **Step 2: Run ocvalidate from the SAME OpenCore version**

Run:
```bash
/Users/khoango/Desktop/hackintosh/build/oc-release/Utilities/ocvalidate/ocvalidate \
  /Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
```
Expected: ends with `0 problems found`. Fix any reported keys (re-edit with PlistBuddy) and re-run until clean.

- [ ] **Step 3: Confirm every referenced file physically exists**

Run:
```bash
cd /Users/khoango/Desktop/hackintosh/build/EFI/OC
for a in $(/usr/libexec/PlistBuddy -c "Print :ACPI:Add" config.plist | awk '/Path =/{print $3}'); do test -f "ACPI/$a" && echo "OK $a" || echo "MISSING $a"; done
for k in $(/usr/libexec/PlistBuddy -c "Print :Kernel:Add" config.plist | awk '/BundlePath =/{print $3}'); do test -d "Kexts/$k" && echo "OK $k" || echo "MISSING $k"; done
```
Expected: all `OK`, no `MISSING`.

- [ ] **Step 4: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/EFI/OC/config.plist
git commit -m "fix(config): pass ocvalidate clean"
```

---

## Phase 3 — Install media (host Mac)

### Task 13: Build the Sequoia install USB

**Files:**
- External: a USB drive (16 GB+), erased.

**Interfaces:**
- Consumes: host Mac, internet.
- Produces: a bootable *Install macOS Sequoia* USB volume.

- [ ] **Step 1: List and fetch the full installer**

```bash
softwareupdate --list-full-installers
sudo softwareupdate --fetch-full-installer --full-installer-version 15.x   # use the latest 15.x listed
```
Expected: `/Applications/Install macOS Sequoia.app` exists afterward.

- [ ] **Step 2: Erase the USB** (Disk Utility → Show All Devices → whole device → Mac OS Extended (Journaled), GUID, name `USB`). Verify:

Run: `diskutil list`
Expected: a volume `/Volumes/USB` on a GUID disk.

- [ ] **Step 3: Write the installer**

```bash
sudo /Applications/Install\ macOS\ Sequoia.app/Contents/Resources/createinstallmedia --volume /Volumes/USB --nointeraction
```
Expected: finishes with `Install media now available at "/Volumes/Install macOS Sequoia"`.

- [ ] **Step 4 (contingency): If createinstallmedia fails**, use OpenCore's `macrecovery.py` to download a recovery image instead, and place `com.apple.recovery.boot` on the USB. Note which path was used.

---

### Task 14: Copy the EFI onto the USB's EFI partition

**Files:**
- External: USB EFI (ESP) partition.

**Interfaces:**
- Consumes: validated `build/EFI` (Task 12), install USB (Task 13).
- Produces: a USB that boots OpenCore and chains to the installer.

- [ ] **Step 1: Identify and mount the USB ESP**

```bash
diskutil list   # find the USB's disk number, e.g. disk4; its EFI is disk4s1
sudo diskutil mount /dev/disk4s1   # adjust to the real identifier
```
Expected: `/Volumes/EFI` mounts.

- [ ] **Step 2: Copy the EFI tree to the ESP root**

```bash
sudo rm -rf /Volumes/EFI/EFI
sudo cp -R /Users/khoango/Desktop/hackintosh/build/EFI /Volumes/EFI/EFI
```

- [ ] **Step 3: Verify**

Run: `ls /Volumes/EFI/EFI/OC/OpenCore.efi /Volumes/EFI/EFI/BOOT/BOOTx64.efi /Volumes/EFI/EFI/OC/config.plist`
Expected: all exist.

- [ ] **Step 4: Unmount**

Run: `sudo diskutil unmount /Volumes/EFI`
Expected: unmounts cleanly.

---

## Phase 4 — Install & post-install (at the machine)

### Task 15: Configure BIOS

**Files:** none (firmware settings). Reference: `MSI-B360M-PRO-VDH-OPENCORE/BiosSettings/*.png`.

**Interfaces:**
- Produces: BIOS configured for OpenCore boot.

- [ ] **Step 1:** Enter BIOS (Del at POST). Load Optimized Defaults.
- [ ] **Step 2:** Set/confirm: Boot mode UEFI; **Fast Boot OFF**; **Secure Boot OFF**; **XHCI Hand-off ON**; **Above 4G Decoding ON** (if present); iGPU as primary / Multi-Monitor ON; DVMT pre-alloc 64 MB; Serial/COM OFF if present. Confirm **CFG Lock disabled** (matches the `false` quirks).
- [ ] **Step 3: Verify** by saving, rebooting, and confirming the USB appears as a UEFI boot entry.
Expected: USB boots to the OpenCore picker.

---

### Task 16: Install macOS Sequoia

**Files:** none (installs to the NVMe).

**Interfaces:**
- Consumes: bootable USB (Tasks 13–14), BIOS (Task 15).
- Produces: a working Sequoia install on the Kingston NVMe.

- [ ] **Step 1:** At the OpenCore picker, choose *Install macOS Sequoia*. (Verbose boot is on — watch for stalls.)
- [ ] **Step 2:** In Disk Utility, erase the Kingston NVMe as **APFS**, scheme **GUID**.
- [ ] **Step 3:** Run the installer. On each reboot, re-select the installer (then later the new macOS volume) in the OpenCore picker.
- [ ] **Step 4:** Complete macOS Setup Assistant to the desktop.
- [ ] **Step 5: Verify** boot to desktop with working display over HDMI/DVI.
Expected: desktop appears, mouse/keyboard work on the one known-good USB port.

**Contingencies (from Task 6/8/9 deferrals):**
- *Black screen at GUI handoff:* add the proven `framebuffer-con0/1/2` data to DeviceProperties (Task 9 Step 2), re-`ocvalidate`, re-copy EFI, retry.
- *NVMe not mounting / install errors:* enable the IONVMe `resetDMA` patch (Task 8 Step 4).
- *Reboot loop at launchd:* re-confirm SMBIOS is `Macmini8,1` and both CFG-lock quirks are `false`.

---

### Task 17: Migrate the EFI to the NVMe

**Files:** NVMe ESP partition.

**Interfaces:**
- Produces: the system boots OpenCore from the internal NVMe without the USB.

- [ ] **Step 1: Mount the NVMe ESP** (from the booted macOS)

```bash
diskutil list   # find the NVMe (Apple_APFS) disk; its EFI is e.g. disk0s1
sudo diskutil mount /dev/disk0s1
```

- [ ] **Step 2: Copy the validated EFI to the NVMe ESP**

```bash
sudo rm -rf /Volumes/EFI/EFI
sudo cp -R /Users/khoango/Desktop/hackintosh/build/EFI /Volumes/EFI/EFI
```

- [ ] **Step 3:** In BIOS, set the NVMe (UEFI OpenCore) first in boot order. Remove the USB.
- [ ] **Step 4: Verify** the machine boots to desktop from the NVMe alone.
Expected: OpenCore picker → Sequoia, no USB attached.

---

### Task 18: Build the USB port map (USBToolBox)

**Files:**
- Create: `build/EFI/OC/Kexts/USBToolBox.kext`, `build/EFI/OC/Kexts/UTBMap.kext`
- Modify: `build/EFI/OC/config.plist` (register both kexts)

**Interfaces:**
- Consumes: a booted Sequoia desktop.
- Produces: a real per-port USB map replacing reliance on `XhciPortLimit`.

- [ ] **Step 1:** Download USBToolBox tool + kext from `https://github.com/USBToolBox/tool` (and `USBToolBox.kext` from its releases). Run the tool in macOS, discover ports, plug a USB2 and USB3 device into each physical port to map them, then **Generate** → produces `UTBMap.kext`.
- [ ] **Step 2:** Copy `USBToolBox.kext` and `UTBMap.kext` into `build/EFI/OC/Kexts/`. Register both in `Kernel:Add` (use the `addkext` helper from Task 8; `UTBMap.kext` has no executable — set `ExecutablePath` empty and keep `PlistPath=Contents/Info.plist`).
- [ ] **Step 3:** `ocvalidate` (Task 12 Step 2), then copy the EFI to the NVMe ESP (Task 17 Step 2).
- [ ] **Step 4: Verify** in macOS that all physical ports enumerate (System Information → USB) and the previously dead ports now work.
- [ ] **Step 5: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/EFI/OC/Kexts/USBToolBox.kext build/EFI/OC/Kexts/UTBMap.kext build/EFI/OC/config.plist
git commit -m "feat: add USBToolBox + generated UTBMap port map"
```

---

### Task 19: Harden the config

**Files:**
- Modify: `build/EFI/OC/config.plist` (`Misc:Security:SecureBootModel`, NVRAM boot-args, `PlatformInfo:Generic:ROM`)

**Interfaces:**
- Produces: production config — Secure Boot on, quiet boot, real ROM.

- [ ] **Step 1: Set the real ROM** (read the primary NIC MAC in macOS)

```bash
ROM=$(ifconfig en0 | awk '/ether/{gsub(":","",$2);print $2}')
/usr/libexec/PlistBuddy -c "Set :PlatformInfo:Generic:ROM $ROM" /Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
```

- [ ] **Step 2: Trim boot-args and enable Secure Boot**

```bash
P=/Users/khoango/Desktop/hackintosh/build/EFI/OC/config.plist
/usr/libexec/PlistBuddy -c "Set :NVRAM:Add:7C436110-AB2A-4BBB-A880-FE41995C9F82:boot-args keepsyms=1 alcid=1" "$P"
/usr/libexec/PlistBuddy -c "Set :Misc:Security:SecureBootModel Default" "$P"
```

- [ ] **Step 3:** `ocvalidate` clean (Task 12 Step 2), copy EFI to NVMe ESP (Task 17 Step 2), reboot.
- [ ] **Step 4: Verify** the machine boots cleanly (no `-v`) and is stable.
- [ ] **Step 5: Commit**

```bash
cd /Users/khoango/Desktop/hackintosh
git add build/EFI/OC/config.plist
git commit -m "chore(config): harden — real ROM, Secure Boot Default, quiet boot"
```

---

### Task 20: Final verification

**Files:** none.

**Interfaces:**
- Consumes: the installed system.
- Produces: a signed-off "definition of done" checklist.

- [ ] **Step 1:** Confirm each item:
  - `ocvalidate` passes on the final `config.plist`
  - Boots to desktop from NVMe alone
  - About This Mac → **Macmini8,1**, i3-8100, 16 GB RAM
  - **Intel UHD Graphics 630** with acceleration: System Information → Graphics/Displays shows "Metal Supported"; UI is smooth at native resolution over HDMI/DVI
  - Audio out works (adjust `alcid` layout if not — try 1, 5, 7, 11)
  - Ethernet up (`ifconfig`/Network pref)
  - Wi-Fi in the menu bar (AirportItlwm)
  - Bluetooth on
  - Kingston NV1 shows as internal storage
- [ ] **Step 2:** Record any deferred items (sleep/wake, iMessage/FaceTime) for a follow-up session. Update the project memory `hackintosh-build.md` with the final working state.
- [ ] **Step 3: Commit** any final notes/docs.

---

## Notes / Out of Scope
- Sleep/wake tuning and iServices (iMessage/FaceTime) validation are deferred until the base system is stable.
- No dGPU configuration (iGPU-only).
- AirportItlwm is version-locked: on every macOS major update, swap in the matching `AirportItlwm.kext` before rebooting.
