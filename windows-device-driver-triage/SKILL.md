---
name: windows-device-driver-triage
description: Use when a device on Windows is not detected or broken.
version: 1.2.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags:
      - windows
      - drivers
      - hardware
      - device-manager
      - usb
      - wifi
      - troubleshooting
    related_skills:
      - systematic-debugging
      - windows-desktop-apps
---

# Windows Device and Driver Triage

## What this does

If you plug something into a Windows PC and it does not work, this skill walks an agent through finding out why and fixing it.

Typical situations:

- You plug in a USB stick, Wi-Fi adapter, printer, webcam, or game controller and Windows does not see it.
- Device Manager shows a device with a yellow warning triangle, or lists it under "Other devices".
- Windows calls a device "unknown", or asks for a driver you do not have.
- The driver disc is missing, or the device came without one.
- The device works for a while and then drops out.
- A freshly installed Windows has several unknown devices listed.

The agent works out what the device actually is, decides whether the fault is hardware or software, finds the correct official driver, checks that it is safe to run, explains the plan to you, and only then installs it. Afterwards it proves the fix worked, and checks again after a restart.

It covers USB devices, PCI and PCIe cards, Wi-Fi and Ethernet adapters, Bluetooth radios, audio, cameras, storage controllers, and chipset devices.

It uses **only built-in Windows tools**: PowerShell, `pnputil`, `sc`, `netsh`, `reg`. No third-party utilities, and no extra installers needed to diagnose.

---

## When to Use

Load this skill when any of the following is true:

- A device shows a **yellow warning triangle** in Device Manager, or sits under "Other devices".
- The user says a device is **not detected**, **not working**, **missing a driver**, or **lost its driver disc**.
- A freshly installed Windows has multiple unknown devices (`SM Bus Controller`, `Base System Device`, `PCI Memory Controller`).
- The user wants a **system health check** for missing or outdated drivers.
- A device works intermittently, drops out, or fails only on certain ports.
- Someone asks you to find an **official driver online** for unbranded or white-label hardware.

**Do not use this skill for:**

- Application faults where the hardware is fine (a program crashes). Use a debugging skill.
- Network configuration with working drivers (DNS, DHCP, firewall). Use a networking skill.
- Performance tuning of a healthy device. Use a profiling skill.

---

## Core principle

> **Classify the failure before you touch a driver.**

Roughly half of "I need a driver" reports are not driver problems at all. A device that cannot complete bus enumeration, that is unseated, underpowered, or physically dead will **never** be fixed by any driver package. Installing one first wastes the user's time and leaves debris to clean up.

This workflow forces classification before action.

---

## Non-negotiable rules

1. **Never install, remove, or modify anything without explicit user confirmation.** Present findings and the exact plan, then wait.
2. **Never guess or accept credentials** (Wi-Fi keys, PINs, BIOS passwords). The user enters them.
3. **Prefer vendor or Microsoft sources over driver-scraper sites.** Scrapers bundle adware and re-signed binaries.
4. **Verify every binary before executing it.** Check the Authenticode signature, and the published hash where one exists.
5. **Never report success without post-change evidence.** Re-query the device and show the result.
6. **Record state before and after.** Write to a log file. Do not rely on scrollback.
7. **Phantom devices are not problems.** Filter `Present -eq $true` when hunting for real faults.
8. **Do not trust a Device Manager screenshot the user sent earlier.** Device Manager caches. Always re-query live.

---

## Phase 0: Scope the request

Establish the symptom in the user's terms, then translate it:

| User says | Likely meaning |
|---|---|
| "Not detected / not showing up" | Bus enumeration failure, or no driver bound |
| "Yellow triangle / exclamation mark" | Device present, driver missing or failing |
| "Works then stops" | Power management, thermal, or loose connection |
| "Slow / dropping" | Driver is fine, link or radio issue |
| "No driver disc / lost the disc" | Driver sourcing task, not a fault |

Capture up front:

- Exact device (brand, model, or the box barcode/EAN)
- Whether it is **currently plugged in**
- What the user has already tried
- Whether the machine has working network access (needed to fetch drivers)

---

## Phase 1: Inventory

Never start from a screenshot. Query the live bus.

```powershell
# Every PRESENT device with a problem
Get-PnpDevice -PresentOnly |
  Where-Object { $_.Status -ne 'OK' } |
  Select-Object Status, Class, FriendlyName, InstanceId, Problem, ProblemDescription |
  Sort-Object Class | Format-Table -AutoSize | Out-String -Width 250
```

```powershell
# Full USB tree (adapt the filter for PCI, HDAUDIO, BTHENUM, etc.)
Get-PnpDevice -PresentOnly |
  Where-Object { $_.InstanceId -like 'USB*' } |
  Select-Object Status, Class, FriendlyName, InstanceId |
  Format-Table -AutoSize | Out-String -Width 250
```

```powershell
# Deep properties for one suspect device
Get-PnpDeviceProperty -InstanceId '<INSTANCE_ID>' -KeyName `
  DEVPKEY_Device_HardwareIds, `
  DEVPKEY_Device_CompatibleIds, `
  DEVPKEY_Device_BusReportedDeviceDesc, `
  DEVPKEY_Device_DriverProblemDesc, `
  DEVPKEY_Device_LocationInfo, `
  DEVPKEY_Device_Parent |
  Select-Object KeyName, Data | Format-List
```

### `DEVPKEY_Device_BusReportedDeviceDesc` is the underrated one

This returns the **raw string the hardware announces about itself**, before any driver renames it. It is how you prove that a strange name in Device Manager (for example `802.11n NIC`, `USB2.0 WLAN`, `Generic USB Hub`) is actually the device the user is holding. Use it whenever the friendly name does not match what the user expects.

### Environment facts worth collecting

```powershell
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version, BuildNumber, OSArchitecture
Get-CimInstance Win32_BaseBoard | Select-Object Manufacturer, Product
$p = New-Object Security.Principal.WindowsPrincipal([Security.Principal.WindowsIdentity]::GetCurrent())
$p.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)   # elevation state
```

---

## Phase 2: Classify

The `Problem` code tells you which bucket you are in. **Do not skip this.**

| Problem | Meaning | Action |
|---|---|---|
| `CM_PROB_NONE` | Healthy | Nothing to fix |
| `CM_PROB_FAILED_INSTALL` (28) | No matching driver | Driver task. Go to Phase 3 |
| `CM_PROB_DEVICE_NOT_THERE` (24) | Device absent or stale node | Ignore if `Present=False`. On PS/2 keyboard and mouse this is normal on modern boards |
| `CM_PROB_PHANTOM` | Historical ghost entry | Ignore. Not a live fault |
| `CM_PROB_DISABLED` (22) | Administratively disabled | Ask why before enabling |
| `CM_PROB_FAILED_START` (10) | Driver loaded but device will not start | Power, firmware, or resource conflict |
| `CM_PROB_DRIVER_FAILED_LOAD` (39) | Driver corrupt or blocked | Reinstall or re-sign |
| **`CM_PROB_DEVICE_DESCRIPTOR_FAILURE` (43)** | **Bus could not read the descriptor** | **NOT a driver problem. Go to Phase 2b** |

### Phase 2b: Descriptor failure decision tree

Symptom: Device Manager shows `Unknown USB Device (Device Descriptor Request Failed)`, hardware ID `USB\DEVICE_DESCRIPTOR_FAILURE`, instance ID `USB\VID_0000&PID_0002\...`.

`VID_0000` means Windows received **no vendor ID at all**. The device never identified itself. No INF file can match it, so **no driver can fix this.** Say so plainly instead of downloading drivers.

Work the ladder in order, cheapest first:

1. **Reseat the plug.** Most common cause. Push firmly, and check for bent or recessed pins.
2. **Different port.** Move from USB 3.0 (blue) to USB 2.0 (black). Cheap 2.0 devices with marginal signal integrity often fail SuperSpeed negotiation on xHCI ports.
3. **Bypass hubs and front-panel headers.** Use rear I/O directly on the board. Front-panel headers on budget boards deliver poor power.
4. **Power.** A bus-powered device on an already-loaded hub will fail. Try a powered hub.
5. **Different machine.** This is the conclusive test. Fails everywhere means the device is dead.
6. **Check for multiple failed instances** at different ports. A device that failed on ports 1, 4 and 9 across attempts is a hardware fault, not a port fault:

   ```powershell
   Get-PnpDevice | Where-Object { $_.InstanceId -like 'USB\VID_0000&PID_0002*' } |
     Select-Object Status, Present, InstanceId | Format-Table -AutoSize
   ```

Only after the device **enumerates** does driver work become meaningful.

> **Related trap:** `Unknown USB Device (Invalid Configuration Descriptor)` means the descriptor was read but is malformed. Usually a genuinely faulty device or a bad hub. Same ladder.

---

### Phase 2c: Audio plays but the microphone does not record

The classic report: "I hear sound in my headset but the mic does not work, and it worked before."

This is almost never a driver fault. It is nearly always **the wrong default input device**. Work the ladder in this order:

1. **What is Windows actually listening to?** Read the default capture endpoint and its mute and gain. A webcam mic left as the default at low gain is the single most common cause.
2. **Do a real capture test.** A device that reports healthy can still deliver silence. Record from it and measure the level (see Phase 7).
3. **Check the jack.** A headset with two 3.5mm plugs has separate output (green) and input (pink). Sound working proves nothing about the mic. The pink plug must be in the pink jack, never blue (line-in) and never green.
4. **Check the inline mute.** Many headsets have a small switch on the cable or a button on the earcup.
5. **Check jack detection.** If the vendor's audio driver is absent (the codec is on Microsoft's generic `hdaudio.inf`), there is no jack retasking and no plug-in popup, so a combo jack may never register a headset mic.

Enumerate every endpoint and its true state from the registry:

```powershell
$base = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\MMDevices\Audio\Capture'
Get-ChildItem $base | ForEach-Object {
  $p     = Get-ItemProperty $_.PSPath
  $props = Get-ItemProperty (Join-Path $_.PSPath 'Properties')
  $nm    = ($props.PSObject.Properties | Where-Object { $_.Name -like '*},2' }).Value
  $st    = switch ($p.DeviceState) { 1 {'ACTIVE'} 2 {'DISABLED'} 4 {'NOTPRESENT'} 8 {'UNPLUGGED'} }
  '{0,-10} {1}' -f $st, $nm
}
```

`UNPLUGGED` on every `Headset` entry, while `Speakers` is the only active output, means Windows sees no headset mic jack in use. That is a physical routing problem and **no driver will fix it**.

Set the default input for **all three roles**, not just Console. Dictation and call apps often follow the Communications role independently, so fixing only Console leaves the bug in place for the app the user actually cares about. The `IPolicyConfig` COM interface (`{870AF99C-171D-4F9E-AF0D-E63DF40C2BC9}`) exposes `SetDefaultEndpoint(deviceId, role)` for roles 0, 1 and 2.

Then set the gain and unmute, and prove it with a real recording:

```powershell
ffmpeg -f dshow -i audio="Microphone (Device Name)" -t 5 -af volumedetect -f null -
```

`max_volume` around -20 dB is a live signal. `-91 dB` is digital silence. Also run `ffmpeg -list_devices true -f dshow -i dummy` to see exactly which capture devices actually exist.

**Do not install an audio driver to fix a wrong default device.** Reinstalling drivers for this symptom never fixes it and wastes the user's time.

---

## Phase 3: Identify the hardware

Once you have a real hardware ID, or need to identify unbranded hardware:

### From a hardware ID

```powershell
# Pull the VID/PID out and search the web for the pair plus "driver"
# e.g. USB\VID_350B&PID_9101  becomes  "350B 9101 driver"
```

Vendor IDs of note: `8086` Intel, `10EC` Realtek, `148F` and `0E8D` Ralink/MediaTek, `0BDA` Realtek USB, `0A12` generic Bluetooth, `1D6B` Linux Foundation.

### From unbranded hardware (no model number, no FCC ID)

Cheap white-label devices are identified by the **barcode/EAN on the box** and by **disc label codes**, not by brand:

1. Read the barcode digits from a photo of the packaging.
2. Search the exact EAN in quotes. Retail listings almost always name the chipset.
3. Cross-check the **box specification table** (frequency band, channel count, modulation) against the chipset's real capability.
4. Treat advertised speeds as fiction. A "600 Mbps" 2.4 GHz single-band dongle is physically a 150 Mbps 1x1 part. State the real ceiling so the user is not surprised later.

### Confirm the chipset family, not the exact part

Prefer a driver package that covers a **family** (for example the MediaTek/Ralink RT2870, RT3070, RT3370, RT5370, RT5372 and RT5572 package) over one pinned to a single part. White-label vendors reuse reference designs, and family packages bind to more real-world units.

---

## Phase 4: Source the driver

Source priority, best first:

1. **Microsoft Update Catalog or Windows Update.** WHQL-signed, no third party involved.
2. **Silicon vendor** (Intel, MediaTek, Realtek, AMD). Reference drivers.
3. **Board or device OEM.** Validated for that product.
4. **Chipset INF utility.** For the yellow-bang system devices, see below.
5. Generic mirrors, only when 1 to 3 are gone, and verify hard.

### Chipset INF clears the mystery yellow bangs

Devices named `SM Bus Controller`, `PCI Memory Controller`, `PCI Data Acquisition and Signal Processing Controller`, and `Base System Device` are almost always **chipset INF gaps**, not missing functional drivers. Install the Intel or AMD Chipset Device Software and they disappear.

**Be honest about scope:** Intel states plainly that the Chipset INF Utility *does not install USB or AGP drivers*. Do not let a user believe it will fix a USB fault.

### Verification gate, mandatory before execution

```powershell
Get-ChildItem 'C:\path\*.exe' | ForEach-Object {
  $s = Get-AuthenticodeSignature $_.FullName
  [PSCustomObject]@{ File=$_.Name; Status=$s.Status; Signer=$s.SignerCertificate.Subject }
} | Format-List

# Compare against the vendor's published hash where one exists
Get-FileHash 'C:\path\file.exe' -Algorithm SHA256
```

**Accept only `Status = Valid`** with a signer that matches the expected vendor. Reject unsigned files, `NotSigned`, `HashMismatch`, or a signer you cannot explain. Record the hash in your report.

### Extracting a driver from an InstallShield package

When the vendor only ships a setup EXE and you want the bare INF:

```powershell
Start-Process -FilePath 'setup.exe' `
  -ArgumentList '/extract_all:"C:\out"' -Wait -PassThru
```

This yields `Disk1\data1.cab`, `data2.cab` and `setup.ini`. Then stage the INF directly:

```powershell
pnputil /add-driver 'C:\out\Disk1\<name>.inf' /install
```

---

## Phase 5: Confirm with the user

**Stop here.** Present, in this order:

1. **What is actually wrong.** One sentence, in plain language.
2. **What it is not.** Explicitly rule out the wrong theory ("this is not a driver problem, no driver can fix it").
3. **The plan.** Numbered steps, each marked safe, reversible, or needs-elevation.
4. **What changes on their system.** New driver, new services, new startup entries, disk space.
5. **Risks and the rollback.** What breaks if it goes wrong, and how to undo it.

Wait for explicit approval. "Yes", "go ahead", "do it". Not silence.

---

## Phase 6: Install

### Elevation from a non-elevated agent

Most agents run at medium integrity. Driver installation needs high integrity. Write a script, then launch it elevated:

```powershell
$p = Start-Process -FilePath 'powershell.exe' `
  -ArgumentList '-NoProfile','-ExecutionPolicy','Bypass','-File','C:\path\install.ps1' `
  -Verb RunAs -PassThru
$null = $p | Wait-Process -Timeout 420 -ErrorAction SilentlyContinue
```

This raises a **UAC prompt the user must click**. Tell them before you run it.

> **UIPI gotcha:** you cannot drive an elevated window with synthetic input from a medium-integrity process. PostMessage is silently dropped. If you need to interact with an elevated GUI, run the automation daemon elevated. Do not assume a click landed.

### Script skeleton

```powershell
$ErrorActionPreference = 'Continue'
$log = 'C:\path\install_log.txt'
function L($m) { "$((Get-Date).ToString('HH:mm:ss'))  $m" | Tee-Object -FilePath $log -Append | Out-Null; Write-Output $m }

L '--- BEFORE ---'
(Get-PnpDevice -PresentOnly | Where-Object { $_.Status -ne 'OK' } |
  Select-Object Status,Class,FriendlyName | Format-Table -AutoSize | Out-String) | % { L $_ }

# ... do the work, logging every step and every exit code ...

L '--- AFTER ---'
(Get-PnpDevice -PresentOnly | Where-Object { $_.Status -ne 'OK' } |
  Select-Object Status,Class,FriendlyName | Format-Table -AutoSize | Out-String) | % { L $_ }
```

### Useful commands

```powershell
pnputil /scan-devices                      # rescan bus
pnputil /restart-device "<INSTANCE_ID>"    # power-cycle one device
pnputil /add-driver "x.inf" /install       # stage and bind
pnputil /enum-drivers                      # what is in the store
pnputil /delete-driver oemNN.inf /uninstall
```

> `pnputil` ignores `/force` when combined with `/uninstall`. It will tell you so. Not an error.

---

## Phase 7: Verify

Verification is **not** "the installer said success". It is four separate observations:

```powershell
# 1. Device node healthy
Get-PnpDevice -InstanceId '<INSTANCE_ID>' | Select-Object Status, Problem, FriendlyName

# 2. The intended driver is the one bound
Get-CimInstance Win32_PnPSignedDriver |
  Where-Object { $_.DeviceID -like '*VID_XXXX&PID_XXXX*' } |
  Select-Object DeviceName, DriverVersion, DriverProviderName, InfName, IsSigned

# 3. The device actually functions (adapt per class)
netsh wlan show interfaces            # Wi-Fi: state, SSID, rates, signal
Get-NetAdapter -Name 'X'              # link state and speed
Test-Connection 1.1.1.1 -Count 3      # real traffic

# 4. Nothing else broke
Get-PnpDevice -PresentOnly | Where-Object { $_.Status -ne 'OK' } |
  Select-Object Status,Class,FriendlyName | Format-Table -AutoSize
```

### Then prove throughput, not just link state

A connected adapter can still be broken. Measure:

```powershell
curl.exe -s -o "$env:TEMP\t.bin" -w "%{size_download} bytes in %{time_total}s = %{speed_download} B/s`n" `
  --max-time 120 "https://speed.cloudflare.com/__down?bytes=10000000"
```

Compare measured throughput against the negotiated PHY rate. **Throughput close to the PHY rate is the success signal.** It means zero retransmission loss. A 36 Mbps link delivering 39 Mbps of payload is perfect. A 300 Mbps link delivering 8 Mbps is a real fault.

> On Windows, native `curl.exe` cannot resolve `/dev/null` when MSYS path translation is off. Always write to a real file path.

### Then reboot and verify again

Driver binding that survives a reboot is the only durable proof. Re-run the same checks after restart. If the device reverts to a descriptor failure, you have a **physical** fault. Say so and stop chasing software.

---

## Phase 8: Clean up the leftovers

Vendor Wi-Fi and Bluetooth utilities leave a wide footprint. After confirming the working driver, offer to remove the superseded package. Check all six locations:

```powershell
# 1. Driver store duplicates
pnputil /enum-drivers        # look for two INFs covering the same device

# 2. Services
Get-CimInstance Win32_Service | Where-Object { $_.PathName -match 'VendorName' }

# 3. Startup folder
Get-ChildItem 'C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp',
              "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup"

# 4. Run keys
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run',
                 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run'

# 5. Program Files
Test-Path 'C:\Program Files (x86)\VendorName'

# 6. Orphaned Apps and Features entry
Get-ItemProperty 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*' |
  Where-Object { $_.DisplayName -match 'VendorName' }
```

**Before deleting a Program Files folder, prove the working driver does not live there:**

```powershell
Get-ChildItem 'C:\Windows\System32\DriverStore\FileRepository' -Directory -Filter '<driver>*'
```

If the driver is in `DriverStore\FileRepository`, the vendor utility folder is dead weight and safe to remove.

**Order matters:** uninstall via the vendor's own uninstaller first, then sweep orphans manually. Verify the device after each step. Do not batch destructive operations.

---

## Reference: common failure signatures

| Signature | Real cause | Fix |
|---|---|---|
| `VID_0000&PID_0002`, descriptor failure | Enumeration failure | Reseat, change port, check power, or replace a dead device |
| Device under "Other devices", odd raw name | No driver bound | Identify via `BusReportedDeviceDesc`, install driver |
| `SM Bus Controller` or `Base System Device` bang | Chipset INF missing | Chipset Device Software |
| Two INFs for one device | Superseded package left behind | Delete the older `oemNN.inf` |
| Wi-Fi connects, no internet | DHCP or DNS, not driver | `ipconfig /renew`, check gateway |
| Device works, then vanishes | USB selective suspend | Disable power management on the root hub |
| Works on USB 2.0, fails on 3.0 | Signal integrity | Keep it on the 2.0 port |
| `CM_PROB_DEVICE_NOT_THERE` on PS/2 | No PS/2 hardware present | Normal. Ignore |
| Sound plays but the mic records nothing | Wrong default input device | Set the correct default for all three roles |
| Headset mic silent, `Headset` endpoint UNPLUGGED | Pink plug not in the mic jack | Move it to the pink jack, not blue or green |
| Mic healthy but captures silence | Inline mute switch on the cable | Slide it off |
| Every `Headset` endpoint UNPLUGGED, only `Speakers` active | Combo jack not detected, or mic plug in an output jack | Use a jack that carries input, or a TRRS splitter |

---

## Anti-patterns

- **Avoid:** downloading a driver the moment a user says "needs a driver". Classify first.
- **Avoid:** treating `VID_0000&PID_0002` as a driver problem. It cannot be.
- **Avoid:** trusting a Device Manager screenshot. It caches, so re-query.
- **Avoid:** reporting phantom devices as faults.
- **Avoid:** installing from a driver-scraper site without signature verification.
- **Avoid:** believing the box. "600 Mbps" on a 2.4 GHz single-band adapter means 150 Mbps 1x1.
- **Avoid:** claiming success because an installer exited 0. Verify the device.
- **Avoid:** deleting a vendor folder before proving the driver lives in DriverStore.
- **Avoid:** asking the user to paste a password into chat. They enter it themselves.

---

## Worked example: white-label USB Wi-Fi dongle

**Report:** "Bought a USB Wi-Fi dongle, drivers only on DVD, no DVD drive. It is plugged in."

1. **Inventory.** `Unknown USB Device (Device Descriptor Request Failed)`, `USB\VID_0000&PID_0002`, `CM_PROB_DEVICE_DESCRIPTOR_FAILURE`.
2. **Classify.** Bus-level enumeration failure. **Not a driver problem.** Told the user directly.
3. **Identify.** No FCC ID, no brand. Box EAN `8699258561108` searched in quotes returned RT5370 across multiple retail listings. Box claimed "600 Mbps"; the spec table said 2.4 GHz single-band, so really 150 Mbps 1x1.
4. **Also found** six unrelated yellow bangs: `SM Bus Controller`, `PCI Memory Controller`, `Base System Device`, `PCI Data Acquisition`. This was a chipset INF gap on a fresh Windows install.
5. **Source and verify.** Intel Chipset INF (SHA256 matched Intel's published hash exactly, signed `Intel Corporation`). MediaTek RT2870-family driver 1.5.39.165 (signed `MEDIATEK INC.`). Both accepted.
6. **Confirm.** Presented the plan, ruled out the driver theory, user approved.
7. **Resolve enumeration.** Reseating cleared the descriptor failure. Device enumerated as `USB\VID_350B&PID_9101`.
8. **Install.** Chipset INF cleared all six bangs. The device bound to a newer WHQL `zt9101d_drv.inf` v15.56.1.960 that Windows supplied.
9. **Verify.** Device `OK / CM_PROB_NONE`, radio on, three networks visible, connected, DHCP lease obtained, 39 Mbps measured against a 36 Mbps PHY rate (saturated, zero loss).
10. **Reboot and re-verify.** Binding persisted, still connected, throughput unchanged.
11. **Clean up.** Superseded `netr28ux.inf` deleted from the driver store, two `MediatekRegistryWriter` services removed, startup shortcut removed, 108 MB utility folder removed after proving the driver lived in `DriverStore\FileRepository`. Device unaffected.

**The decisive insight:** the user's screenshot showed `802.11n NIC` under "Other devices". `DEVPKEY_Device_BusReportedDeviceDesc` returned exactly `802.11n NIC` for the *working* device, proving it was the same hardware mid-repair, not a second fault. The screenshot was a stale cached view.
