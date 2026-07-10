# Remote factory reset — FXOS reimage

!!! success "✅ Complete · Written 2026-07-10"
    Canonical Cisco-recommended remote factory reset for the 1210CE. Run this before Ch 4 when you need a clean first-boot state, and every time thereafter when you need to start over. Every step is executable over console + management SSH — no physical access to the box required.

Once the 1210CE is racked and remote, "factory reset" can no longer mean pressing a button or booting a USB. This chapter is the exact procedure for wiping the box back to first-boot state entirely via console + management network — the precondition for the fresh wizard walk-through in [Ch 4 (First boot)](first-boot.md).

## Why FXOS `install security-pack` is the answer

Every other candidate fails at least one of the lights-out constraints:

| Candidate | Meets the bar? | Why / why not |
|---|---|---|
| **FXOS `install security-pack`** | ✅ | Cisco-explicit "recommended" for 1000/2100/1200 series. Verified present on this build at `/firmware/auto-install# ?`. Wipes FXOS + FTD in sync so the two layers can't drift. Fully remote-executable. Deterministic first-boot state. |
| FXOS `erase configuration` | ❌ | Doesn't exist on 7.6.0-113 / 1200-series FXOS. Enumerated every visible scope (`system`, `chassis`, `security`, `ssa`, `firmware`, `auto-install`, `org`) — no `erase`. Even where it does exist on older platforms, Cisco's own guide flags it as "may leave FXOS mismatched." |
| FTD `configure factory-default` | ❌ | Not in FTD 7.6.0-113's CLI. Older guides that cite it are stale. |
| FTD `configure manager delete` + `configure network reset` | ❌ | Partial. Leaves policies, license state, VDB, LSP intact. Doesn't re-arm the setup wizard. Not a factory reset. |
| FDM UI Reset | ❌ | Requires a browser + human clicks. Opposite of lights-out. |
| ROMMON reset | ❌ | Requires physical boot-interrupt at the console. Doesn't work when no one's at the box. |

Result: for a lights-out guide, there is exactly one Cisco-recommended remote reset path.

## Prerequisites — one-time install-package staging

`install security-pack` reads a Cisco install file (`cisco-ftd-fp1k.<version>.SPA`, ~1.5–2 GB) reachable over SCP on the management network. Stage it **once on ConsolePi**; every subsequent reset reuses the same file with zero further setup.

Why ConsolePi and not somewhere else:

- Already on the management VLAN — no cross-network firewall rules to open
- Already the console gateway — one trust boundary for all OOB access
- SSH server already running — SCP works with zero additional daemons
- Persists across resets — the FW gets wiped, ConsolePi stays as the recovery seed

### Step 1 — Get the package from Cisco.com

Download `cisco-ftd-fp1k.<version>.SPA` from Cisco.com → Software → Downloads → Cisco Secure Firewall 1200 Series → your model (1210CE) → your target version. This requires a CCO login.

For this lab we're staying on **7.6.0-113** (same as installed, gives us a bit-perfect reset to a known state). If you're taking the opportunity to move up, pick the newer 7.6.x patch instead — the procedure is identical.

### Step 2 — Stage on ConsolePi

```bash
# On ConsolePi (192.168.40.5 in this lab)
sudo install -d -m 755 -o pi -g pi /srv/firmware-cache

# From your workstation
scp cisco-ftd-fp1k.7.6.0-113.SPA pi@192.168.40.5:/srv/firmware-cache/
```

That's the last time you touch the package until Cisco cuts a new version.

## Reset procedure

Console into the FW first (see [Ch 3 — Console access](console-access.md)):

```bash
telnet 192.168.40.5 8000
```

Log in as `admin`. You'll land at the FTD `>` prompt.

### 1 — Enter FXOS

The FTD `>` prompt is downstream of FXOS. To reach FXOS Service Manager, type `exit`:

```
> exit
fw1210ce#
```

!!! tip "This is not a logout"
    On the 1210CE (and the whole 1200/1000/2100 line), the serial console lands you in FXOS by default; the FTD `>` prompt is entered from FXOS with `connect ftd`. So `exit` from FTD returns you to FXOS. `connect fxos` from FTD does nothing useful — the box will tell you "You came from FXOS Service Manager. Please enter 'exit' to go back."

### 2 — Download the install package to the FW

```
fw1210ce# scope firmware
fw1210ce /firmware# download image scp://pi@192.168.40.5:/srv/firmware-cache/cisco-ftd-fp1k.7.6.0-113.SPA
```

FXOS will prompt for the ConsolePi's `pi` user password ([OpenBao `infra/ssh/consolepi`](../../.claude/CLAUDE.md#openbao)). It then transfers the file into the FW's local firmware store — typically 3–7 minutes over the mgmt LAN.

Track progress:

```
fw1210ce /firmware# show download-task
fw1210ce /firmware# show package
```

Wait for state `Downloaded`. Then verify:

```
fw1210ce /firmware# verify security-pack version 7.6.0-113
```

### 3 — Trigger the reimage

Move to the install scope and fire:

```
fw1210ce /firmware# scope auto-install
fw1210ce /firmware/auto-install# install security-pack version 7.6.0-113 force
```

FXOS will emit **two confirmation prompts**. Type `yes` at each. It will not accept `y` — the full word is required.

At the second `yes`, the reimage begins. The FW reboots into the reimage process and console output goes quiet on and off across several reboot cycles.

## What to expect during reimage

- **Total wall-clock**: 30–60 minutes typical. First 10–15 min is preparation + first reboot; the middle 20–30 min is image install (very quiet on the console); last 5–10 min is the second reboot and FTD process startup.
- **Console output**: comes and goes. Long silent gaps are normal and do NOT mean the box is hung. If you can, do not touch the console — spurious keystrokes at some phases can corrupt state.
- **Management IP**: gone. Post-reimage the mgmt interface returns to Cisco's factory default until the wizard configures it.
- **Passwords**: `admin` returns to `Admin123` (default) until the wizard forces a change.
- **Policies, license, VDB, LSP, manager**: all wiped.

## After the reimage

The console lands you at the first-boot setup wizard:

```
Firepower login: admin
Password: Admin123
```

Head to [Ch 4 (First boot and initial config)](first-boot.md) for the wizard walk-through.

## Related

- [Ch 3 — Console access via ConsolePi](console-access.md) — required for this chapter
- [Ch 4 — First boot and initial config](first-boot.md) — the wizard flow after reimage completes
- [Ch 15 — Troubleshooting](troubleshooting.md) — pointer back here for the "FW in a weird state" nuclear option

## Next

If the reset ran, head straight to [Ch 4 (First boot)](first-boot.md) to capture the wizard.
