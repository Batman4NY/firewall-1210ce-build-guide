# Remote factory reset — FXOS cross-version reimage

!!! success "✅ Complete · Corrected 2026-07-10 after live validation"
    Canonical Cisco remote factory reset for the 1210CE. Fully lights-out. Works air-gapped when a suitable target package is already staged. **On the 1200 series, "reset" is a cross-version reimage** — same-version `force` is a silent no-op on this build, so reset always targets a version different from the currently-running one.

Once the 1210CE is racked and remote, "factory reset" can no longer mean pressing a button or booting a USB. This chapter is the exact procedure for wiping the box back to first-boot state entirely via console + management network — the deterministic starting point for every walk-through in this guide.

## The reset mechanism — cross-version, not same-version

On the 1200-series with FTD 7.6.x, the FXOS command `install security-pack version <ver> force` **short-circuits when `<ver>` equals the currently-running version**, reporting `Firmware Upgrade Message: up-to-date` and doing nothing. The `force` flag overrides *cross-version compatibility warnings*, not the *same-version guard*.

The only Cisco-supported lights-out reset on this hardware is therefore:

> **`install security-pack version <different-version>`** where `<different-version>` is present on the FW's local firmware store and does NOT match the currently-running version.

FXOS treats this as a full reimage:

- Wipes FTD user config, policies, license state, VDB, LSP
- Wipes FXOS-side admin config and forces a new password on next login
- Reboots through kickstart, then reimages the FTD from the target package
- Generates a **new UUID** on the reimaged FTD (evidence this is a wipe, not upgrade-in-place)
- Total wall-clock: ~17-18 min for the reimage itself, ~5 min for the FXOS + FTD first-boot flows

### Verified against every other candidate reset path

| Candidate | Meets the lights-out bar? | Why / why not |
|---|---|---|
| **FXOS `install security-pack` — cross-version** (this chapter) | ✅ | The only CLI-driven reset that actually reimages on 1200-series 7.6.x. Cisco-supported. Deterministic. Atomic FXOS + FTD resync. |
| FXOS `install security-pack` — same-version + `force` | ❌ | Silent no-op on this build. `Firmware Upgrade Message: up-to-date`. Force flag semantics differ from what you'd expect. |
| FXOS `erase configuration` | ❌ | Doesn't exist at any observable scope on 1200-series FXOS 2.16. Enumerated live. |
| FTD `configure factory-default` | ❌ | Not in FTD 7.6.0-113 or 7.6.4-69's CLI. Stale references in older Cisco Community posts + earlier guides. |
| FTD `configure manager delete` + `configure network reset` | ❌ | Partial. Doesn't re-arm the setup wizard, doesn't clear policies or license. |
| FDM UI Reset to Factory Default | ❌ | Requires a browser. Opposite of lights-out. |
| ROMMON reset | ❌ | Physical console interrupt required at boot. Not remote. |

## Pre-flight — inventory of what's on the FW

Console into the box (see [Ch 3 — Console access](console-access.md)):

```
telnet <consolepi-ip> 8000
```

On the 1200 series, the console lands you in **FXOS**, not FTD. If the terminal shows the FTD `>` prompt (from a prior session), type `exit` to reach the FXOS `firepower#` (or `<hostname>#`) prompt.

Enumerate the local firmware store:

```
firepower# scope firmware
firepower /firmware# show package
Name                                          Package-Vers
--------------------------------------------- ------------
Cisco_Secure_FW_TD_1200-7.6.0-113.sh.REL.tar  7.6.0-113
Cisco_Secure_FW_TD_1200-7.6.4-69.sh.REL.tar   7.6.4-69
cisco-asa-csf1200.9.22.1.1.SPA                9.22.1.1
```

Also check the currently-running version:

```
firepower# scope system
firepower /system# show version
    Version:      7.6.4-113
    Package-Vers: 7.6.4-69
firepower /system# top
```

### Selecting the reset target

The reset target must be **on the local firmware store AND different from the currently-running version**. Two useful scenarios:

- **Rolling back to factory** — running > factory pre-stage. Target the pre-staged Cisco-signed package (e.g., `7.6.0-113` in the example above). Zero download required — the payload is already on the FW.
- **Reimaging onto a fresh patched baseline** — running < available patched version, or you want a specific target. Target a downloaded package (e.g., `7.6.4-69`). Requires the staging step below if the target isn't already on the store.

If neither condition applies (e.g., factory-fresh box where the only package on flash matches the running version), you MUST stage a different version before the reset will actually run.

## Optional — staging a target package on the FW

Skip this section if `show package` already lists a version you want to reset onto that isn't the running version.

### Step S1 — Get the package from Cisco.com

CCO login required. Portal path:

`software.cisco.com` → **Downloads Home** → **Security** → **Firewalls** → **Next-Generation Firewalls (NGFW)** → **Secure Firewall 1200 Series** → **Secure Firewall 1210CE** → **Firepower Threat Defense (FTD) Software** → target release

Per release, two files are listed. Pick the install package, NOT the CC hotfix:

| File | Size | Purpose | Pick? |
|---|---:|---|:---:|
| `Cisco_Secure_FW_TD_1200-<ver>-<build>.sh.REL.tar` | ~970 MB | Install & upgrade package | ✅ |
| `Cisco_Secure_FW_TD_1200_Hotfix_CC-<ver>.<hf>-<build>.sh.REL.tar` | ~220 MB | Common Criteria hotfix — regulated deployments only | ❌ |

!!! warning "Filename convention on the 1200 series"
    Cisco's 1000/2100 series uses `cisco-ftd-fp1k.<version>.SPA`. The **1200 series uses a different structure**: `Cisco_Secure_FW_TD_1200-<version>-<build>.sh.REL.tar`. Cisco's own download page says **"Do not untar"** — the `.tar` file is the install package; the extractor lives inside the FXOS installer.

### Step S2 — Stage on the ConsolePi

```bash
# On ConsolePi (one-time)
sudo install -d -m 755 -o pi -g pi /srv/firmware-cache

# From your workstation
scp ~/Downloads/Cisco_Secure_FW_TD_1200-7.6.4-69.sh.REL.tar pi@<consolepi-ip>:/srv/firmware-cache/
```

Verify checksums match between Cisco's download page and your local file. For the reference file used in this guide (7.6.4-69):

```
469eb3e389f2fa14c57799d87e8290d389e0a53e81103c70b4a25d6fc12636b3  Cisco_Secure_FW_TD_1200-7.6.4-69.sh.REL.tar
```

Why ConsolePi as the staging host:

- Already on the management VLAN — no cross-network firewall rules to open
- Already the console gateway — one trust boundary for all OOB access
- SSH server already running — SCP works with zero additional daemons
- Persists across resets — every future reset onto the same version reuses the staged file

### Step S3 — Download the package into the FW's firmware store

```
firepower /firmware# download image scp://pi@<consolepi-ip>/srv/firmware-cache/Cisco_Secure_FW_TD_1200-7.6.4-69.sh.REL.tar
```

FXOS's SCP client prompts for the `pi` user's password on the first connection to a given host. Downloads run as **background tasks** — the prompt returns immediately, which does not mean the download is done. Poll:

```
firepower /firmware# show download-task detail
    State: Downloading  →  Downloaded
```

Wait for `State: Downloaded`. Confirm the package is registered:

```
firepower /firmware# show package
```

### Air-gap variant — `usbA:` transfer

For SCIF / classified deployments where SCP can't cross the boundary, use the front-panel USB port. FAT32-format a USB drive, copy the package on, insert into the 1210CE:

```
firepower /firmware# download image usbA:/Cisco_Secure_FW_TD_1200-7.6.4-69.sh.REL.tar
```

Same background-task polling flow.

## Reset procedure

### 1 — Verify pre-flight

- `show package` lists your intended target version.
- `scope system` / `show version` confirms the target does NOT equal the running version.

### 2 — Move to the auto-install scope

```
firepower# scope firmware
firepower /firmware# scope auto-install
firepower /firmware/auto-install#
```

### 3 — Fire the install

```
firepower /firmware/auto-install# install security-pack version 7.6.4-69
```

Substitute your target version. **Do NOT append `force`** — cross-version installs benefit from FXOS's built-in compatibility checks; `force` overrides them, which is not what you want during a reset.

FXOS emits a summary of what's about to happen:

```
The system is currently installed with security software package 7.6.0-113, which has:
   - The platform version: 2.16.0.128
   - The CSP (ftd) version: 7.6.0.113
If you proceed with the upgrade 7.6.4-69, it will do the following:
   - upgrade to the new platform version 2.16.1.147
   - reimage the system from CSP ftd version 7.6.0.113 to the CSP ftd version 7.6.4.69
During the upgrade, the system will be reboot

Do you want to proceed ? (yes/no): yes
This operation upgrades firmware and software on Security Platform Components
Here is the checklist of things that are recommended before starting Auto-Install
(1) Review current critical/major faults
(2) Initiate a configuration backup

Do you want to proceed? (yes/no): yes

Triggered the install of software package version 7.6.4-69
Force option: false
Install started. This will take several minutes.
```

Type `yes` at each prompt. Full word required — `y` is rejected.

## What to expect during reimage

Empirical measurements from this guide's reference build (7.6.0-113 → 7.6.4-69 on a 1210CE, 2026-07-10):

| Milestone | Wall-clock | Δ from install fire |
|---|---:|---:|
| `install security-pack` fires + `yes` × 2 accepted | 0 | — |
| FTD shutdown sequence starts (`rc6.d/K00ftd.sh stop`) | +57s | +57s |
| Mgmt IP drops (interface offline) | +62s | +62s |
| FXOS emits `Bundle version in firmware package is empty, need to re-install` | +4m 2s | +4m 2s |
| First `firepower login:` prompt reappears on console | +17m 21s | **+17m 21s** |
| Wizard `Successfully performed firstboot initial configuration steps` | +39m 40s | — |
| FTD `>` prompt reachable with `show version` returning target | ~+40m 30s | — |

Between the "Install started" line and the reappearance of `firepower login:`, the console goes silent for stretches of 5-10 min. This is normal and does not mean the box is hung.

- **Console output**: comes and goes across several reboot cycles. Long silent gaps up to 10 min are normal. Do not touch the console — spurious keystrokes at some phases can corrupt state.
- **Management IP**: gone. Post-reimage the mgmt interface returns to DHCP defaults until the wizard reconfigures it.
- **Passwords**: `admin` returns to `Admin123` (default) until the FXOS first-login flow forces a change.
- **UUID**: regenerates. `show version` will report a new UUID — confirmation this was a real reimage, not an upgrade-in-place.
- **Policies, license, VDB, LSP, manager**: all wiped.

## First-boot flow — FXOS side then FTD side

The 1200-series post-reimage flow is different from older FTD platforms. There are TWO first-boots to walk through in sequence.

### FXOS first login (mandatory password change)

The console prompts:

```
firepower login: admin
Password: Admin123

Cisco Firepower Extensible Operating System (FX-OS) v2.16.1 (build 147)
Cisco Secure Firewall 1210CE Threat Defense v7.6.4 (build 69)

Hello admin. You must change your password.
Enter new password: <choose a strong password>
Confirm new password: <same>
```

At the second confirm, FXOS drops you to `firepower#`. The FTD wizard does NOT trigger automatically — you must invoke it.

### FTD wizard on first `connect ftd`

```
firepower# connect ftd
```

FTD displays the EULA. Page through with `Space` until the acceptance prompt, then type `YES`:

```
Please enter 'YES' or press <ENTER> to AGREE to the EULA: YES
System initialization in progress. Please stand by.
```

The 7.6.4 setup wizard asks a short set of questions (verified live on the reference build):

```
Configure IPv4 via DHCP or manually? (dhcp/manual) [manual]: dhcp
Configure IPv6 via DHCP, router, or manually? (dhcp/router/manual) [dhcp]: none
No DNS servers specified to configure.
No domain name specified to configure.
No hostname name specified to configure.
Setting DHCP for IPv4: management0
Updating routing tables, please wait...
All configurations applied to the system. Took 3 Seconds.

Manage the device locally? (yes/no) [yes]: yes
Configuring firewall mode to routed

Update policy deployment information
    - add device configuration
Successfully performed firstboot initial configuration steps for
Secure Firewall Device Manager for Secure Firewall Threat Defense.

>
```

That's it. The 7.6.4 wizard is **substantially shorter than pre-7.6 FTD flows** — it doesn't prompt for hostname, DNS servers, search domains, proxy, or firewall mode. Those default to sensible values (hostname `firepower`, DNS to Cisco/OpenDNS resolvers, no proxy, routed mode). Custom values go in via FDM UI in [Ch 6 — FDM baseline](fdm-baseline.md), or via FTD CLI `configure network ...` commands.

## Verify

At the FTD `>` prompt:

```
> show version
Model:    Cisco Secure Firewall 1210CE Threat Defense (86) Version 7.6.4 (Build 69)
UUID:     3c2bdc6c-7c8f-11f1-9ef7-9d09498b33b8       ← new UUID confirms real reimage
VDB:      392

> show network
Hostname: firepower                                   ← factory default
DNS:      208.67.222.222, 208.67.220.220, 2620:119:35::35   ← Cisco/OpenDNS defaults
IPv4:     Configuration: DHCP  →  192.168.10.59 / 255.255.255.0
Gateway:  192.168.10.1
MAC:      78:11:9d:6a:80:80

> show managers
No managers configured.
```

The reset is complete. Head to [Ch 6 — FDM baseline](fdm-baseline.md) to set your custom hostname, DNS, and static mgmt IP through the FDM UI. Then [Ch 5 — Choose your management path](choose-mgmt-path.md) or the specific onboarding chapter for your chosen manager.

## Change-record hygiene for regulated fleets

For fleets under formal change control (banks, healthcare, FedRAMP), the reset event is a change of software version and a security-baseline event that generates audit artifacts:

- Attach this chapter to the change record.
- Include the SHA-256 of the target install package (staged file) in the change ticket.
- Include the "before" `show version` and "after" `show version` outputs (with the changed UUID) as evidence the reimage actually happened.
- If the target version has any active PSIRT advisories against it, note the risk-accept in the change record.

## Related

- [Ch 3 — Console access via ConsolePi](console-access.md) — required for this chapter
- [Ch 6 — FDM baseline](fdm-baseline.md) — set custom hostname, DNS, static mgmt IP, etc. via FDM UI after the wizard
- [Ch 5 — Choose your management path](choose-mgmt-path.md) — pick FDM-standalone / SCC-hybrid / cdFMC / FMCv after Ch 6
- [Ch 15 — Troubleshooting](troubleshooting.md) — "FW in a weird state" points back here

## Next

If the reset ran, head to [Ch 6 — FDM baseline](fdm-baseline.md) to configure the interfaces and policies through the FDM UI. Or, if the box is destined for SCC / cdFMC management, jump directly to [Ch 8 — SCC onboarding](scc-onboarding.md).
