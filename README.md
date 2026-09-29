# Vault-Keeper
Non-destructive Windows host reconnaissance payload for the USB Rubber Ducky. Collects identity, OS, firmware, hardware, network configuration, security posture, local admins, installed software, and top processes, then writes one plain-text report to the target's Desktop.
# VaultKeeper — Comprehensive Windows Host Profiler

Non-destructive Windows host reconnaissance payload for the USB Rubber
Ducky. Collects identity, OS, firmware, hardware, network configuration,
security posture, local admins, installed software, and top processes,
then writes one plain-text report to the target's Desktop.

- **Author:** D-L3aN
- **Version:** 1.0
- **Category:** recon
- **Target:** Windows 10 / 11
- **Requires:** PowerShell 5.1+

---

## Authorised Use Only

This payload is intended for **authorised security testing, red team
engagements, and internal IT audits only**. Deploying it against systems
you do not own or do not have explicit written permission to test is
illegal in most jurisdictions.

VaultKeeper is deliberately **non-destructive** and **read-only**. It
does **not**:

- Modify the registry, filesystem, or system configuration
- Create users, scheduled tasks, services, or any persistence
- Read the contents of any user document
- Collect WiFi passwords, cached credentials, browser data, or tokens
- Make outbound network connections unless `#COLLECT_PUBLIC_IP TRUE`
- Delete PowerShell history unless `#CLEAR_HISTORY TRUE`

Both of the above switches default to `FALSE`. Leave them `FALSE` unless
your engagement scope explicitly authorizes them.

---

## What it collects

| Section | Contents |
|---|---|
| Identity | Timestamp, hostname, user, workgroup, PS version, chassis, manufacturer, model |
| Operating System | Caption, version, build, architecture, install date, last boot, time zone |
| Firmware | BIOS vendor, version, **release date**, serial, mainboard |
| Hardware | CPU model/cores/threads, total RAM, disk models/sizes, GPUs |
| Network | IPv4 addresses, gateway, DNS, MACs of up interfaces |
| WiFi Profiles | Profile names and count only — **no keys** |
| Security Posture | AV product, firewall state, UAC, BitLocker, Secure Boot, TPM |
| Local Administrators | Group membership via `net localgroup` |
| Installed Software | Up to 40 programs (configurable), deduplicated across HKLM/WOW6432Node/HKCU |
| Top Processes | Top 15 by working set |

---

## Configuration

All settings are `DEFINE` statements in `payload.txt`. Changes to the
`#COLLECT_PUBLIC_IP`, `#CLEAR_HISTORY`, `#SOFTWARE_TOP`, and
`#PROCESSES_TOP` values require re-running `build.ps1` and pasting the
new Base64 string back into `payload.txt`.

| Define | Default | Purpose |
|---|---|---|
| `#BOOT_DELAY` | `2500` | Wait after insertion before typing |
| `#SETTLE` | `2500` | Wait after launching PowerShell |
| `#SOFTWARE_TOP` | `40` | Max programs listed (`0` = unlimited) |
| `#PROCESSES_TOP` | `15` | Max processes listed by RAM |
| `#COLLECT_PUBLIC_IP` | `FALSE` | Query `api.ipify.org` for public IP |
| `#CLEAR_HISTORY` | `FALSE` | Delete PSReadLine history on exit |

---

## Building the Base64

DuckyScript's `STRING` command cannot reliably encode the special
characters a full PowerShell recon pipeline requires. VaultKeeper ships
the collector as a Base64-encoded `-EncodedCommand` block instead,
which contains only `A-Z a-z 0-9 + / =` — all of which map cleanly on
any US keyboard layout.

To regenerate the Base64 after editing `build.ps1`:

```powershell
powershell -ExecutionPolicy Bypass -File .\build.ps1
```

Copy the single-line output, then paste it into `payload.txt` in place
of `<REPLACE_WITH_BASE64_FROM_build.ps1>`.

---

## Sample output

```
--- IDENTITY ---
Timestamp    : 2025-05-02 09:14:07 +00:00
Hostname     : WKSTN-042
User         : CORP\jdoe
Workgroup    : CORP
PSVersion    : 5.1.22621.2506
Manufacturer : Dell Inc.
Model        : Latitude 5540
Chassis      : 10

--- OPERATING SYSTEM ---
Caption      : Microsoft Windows 11 Pro
Version      : 10.0.22631  Build 22631
Architecture : 64-bit
...

--- FIRMWARE ---
Vendor       : Dell Inc.
Version      : 1.14.2
Release Date : 2024-11-08 00:00:00
Serial       : 7XK9L43
...

--- SECURITY POSTURE ---
AV           : Windows Defender
Firewall     : Domain=True Private=True Public=True
UAC (EnableLUA) : 1
BitLocker C: : On
Secure Boot  : True
TPM Present  : True
...
```

---

## Differences from NullSec System Profiler

VaultKeeper was inspired by NullSec System Profiler (bad-antics) and
reuses its core idea, but makes several deliberate changes:

1. **No credential collection.** NullSec extracts saved WiFi passwords
   in cleartext via `netsh wlan show profile key=clear`. VaultKeeper
   collects profile names only. This makes the payload usable in
   engagements where credential collection is out of scope.
2. **No unconditional cloud calls.** NullSec queries
   `api.ipify.org` on every run. VaultKeeper gates this behind
   `#COLLECT_PUBLIC_IP`, default `FALSE`.
3. **No unconditional history deletion.** NullSec deletes the
   PowerShell history file at the end of every run. VaultKeeper gates
   this behind `#CLEAR_HISTORY`, default `FALSE`. Forensic cleanliness
   is an operator decision, not a payload default.
4. **Base64 `-EncodedCommand`.** NullSec's long pipelines are pasted
   as individual `STRINGLN` lines, which frequently fail to compile
   when the client's keyboard layout differs from US. VaultKeeper
   encodes the entire collector and injects it as one line.
5. **Explicit OS check.** NullSec depends on the passive-detection
   extension to set `$_OS`; if detection is partial, the payload can
   still fire into a non-Windows host. VaultKeeper makes the
   `$_OS == WINDOWS` check explicit and bails with `STOP_PAYLOAD`.
6. **Sectioned, greppable output.** Section markers
   (`--- SECTION ---`) let you slice reports with `grep` or `awk`
   without parsing whitespace.
7. **Consistent timestamped filename** on the Desktop so multiple
   runs on the same host don't overwrite each other.

---

## Tested on

- Windows 11 23H2 — PowerShell 5.1 — Defender enabled — **Pass**
- Windows 10 22H2 — PowerShell 5.1 — Defender enabled — **Pass**

---

## Known limitations

- **Windows only.** macOS and Linux are not handled.
- **English header labels only.** Values are language-independent.
- **EDR visibility.** `-Exec Bypass` and hidden-window PowerShell are
  noisy on modern EDR. Expect flags — that is a finding, not a bug.
- **BitLocker + redirected Desktop.** On enterprise configurations the
  Desktop may be redirected; verify the report lands where you expect.
- **WiFi section requires the WLAN AutoConfig service.** On hosts where
  it is stopped, the section is empty.

---

## Deployment notes

1. **Pre-engagement** — confirm in writing which machines the payload
   will run against and what data the client authorizes you to collect.
   Keep `#COLLECT_PUBLIC_IP` and `#CLEAR_HISTORY` `FALSE` unless both
   are explicitly in scope.
2. **During** — do not modify this payload to exfiltrate in the field
   without updating the client agreement.
3. **Post-engagement** — remove the report from each target, or hand
   it to the client as a deliverable, per your SOW.
4. **Report to the client** — include the fact that unauthenticated
   physical USB access alone was sufficient to enumerate installed
   software, network configuration, and security posture. That is the
   actionable finding.

---
