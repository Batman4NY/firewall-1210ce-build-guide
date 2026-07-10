# Pre-compaction state snapshot — 2026-07-10 afternoon (revision 2)

Written just before Claude conversation compaction. Everything CONFIRMED by direct in-session evidence.

## Current activity

**Validation reset RUNNING under Bash task `b5mgt202h`** — launched at ~15:24 EDT.

- **Target**: `install security-pack version 7.6.0-113` (cross-version DOWNGRADE from current 7.6.4-69 back to factory pre-staged)
- **Script**: `/tmp/claude-1000/-home-fabian/6dba75c3-8307-42ec-9551-496d49432b88/scratchpad/reset-validate.py`
- **Log**: `/home/fabian/epoch-dev/firewall-1210ce-build-guide/captures/reset-validate-2026-07-10.log`
- **Expected wall-clock**: ~22-25 min end-to-end (empirical from prior run: install fire → first-boot ~17m, → wizard done ~22m)
- **Task-notification**: direct on script exit (no wrapper watchdog)

## Post-compaction: WHAT TO DO

1. **Check task `b5mgt202h`** — if completed with exit 0, read `/tmp/claude-1000/-home-fabian/6dba75c3-8307-42ec-9551-496d49432b88/tasks/b5mgt202h.output` for the final summary (elapsed timings, final version, new UUID, mgmt IP).
2. **Verify final state**: `> show version` should return `7.6.0 (Build 113)` with a NEW UUID (different from `3c2bdc6c-7c8f-11f1-9ef7-9d09498b33b8`).
3. **Update OpenBao**: `bao kv patch infra/webui/fw1210ce-admin host=192.168.40.<x>` where `<x>` is the new lease (Fabian moved port profile → 192.168.40.0/24 mgmt VLAN mid-run).
4. **Add UDM DHCP reservation**: MAC `78:11:9d:6a:80:80` → suggested `192.168.40.10` (static band, matches ConsolePi pattern).
5. **Real timing block for Ch 3.5**: replace the current empirical timings with the fresh validation-run numbers.
6. **Push doc commits**: locally committed as `50e432c` + subsequent edits; push blocked on `gh` token — Fabian's active `bfgarcia` account only has `pull:true, push:false` on `Batman4NY/firewall-1210ce-build-guide`. Ask Fabian to run `gh auth refresh --scopes repo` or push manually.

## Topology at compaction (Fabian just moved mgmt to VLAN 40)

| Component | Interface | IP | Network |
|---|---|---|---|
| ConsolePi | eth0 | 192.168.40.5/24 (STATIC) | mgmt VLAN 40 |
| FTD `management0` (post-reset expected) | | 192.168.40.x (DHCP) | mgmt VLAN 40 — **just moved via port profile change** |
| FTD `Ethernet1/1` outside | | 192.168.1.198 (DHCP factory default) | home LAN |
| FTD console (RJ45 → FTDI → ConsolePi) | ttyUSB0 → ser2net :8000 | `telnet 192.168.40.5 8000` | mgmt VLAN 40 |
| Workstation | eno1 | 192.168.10.3 | 192.168.10.0/24 |

**Note**: I over-extrapolated "AI Subnet" as the name for `192.168.10.0/24` earlier — Fabian corrected. Use IP range in doc-facing prose, not "AI Subnet."

## Architecture findings — CORRECTED from earlier drafts

### Reset mechanism (CRITICAL — I confabulated earlier)

- **`install security-pack version <same-version> force` is a SILENT NO-OP on 1200-series 7.6.x.** FXOS reports "Install started" and "Force option: true" but does nothing. `show detail` reveals `Firmware Upgrade Message: up-to-date`.
- **The only lights-out CLI reset that actually reimages is `install security-pack version <different-version>`** — cross-version. Force flag overrides *compatibility warnings*, NOT the *same-version guard*.
- **Cross-version signature**: FXOS emits `If you proceed with the upgrade X-Y, it will do the following: - upgrade to platform version ... - reimage the system from CSP ftd version X to Y - During the upgrade, the system will be reboot`. If you don't see "reimage the system" and "system will be reboot", the reset didn't fire.
- **New UUID** = definitive reimage evidence. Pre-reset UUID `dc0bef0a-66b5-...` → post-reset `3c2bdc6c-7c8f-...`.

### FTD 7.6.4 wizard flow (much shorter than pre-7.6)

Post-reimage, the console lands in **FXOS** (not FTD). Two-stage first-boot:

1. **FXOS first login** — `firepower login: admin` / `Password: Admin123` → forced password change → drops to `firepower#`
2. **`connect ftd`** — triggers FTD wizard. Prompts (verified live on 7.6.4):
   - EULA (page + `YES`)
   - `Configure IPv4 via DHCP or manually? (dhcp/manual) [manual]:`
   - `Configure IPv6 via DHCP, router, or manually? (dhcp/router/manual) [dhcp]:`
   - `Manage the device locally? (yes/no) [yes]:`
   - `Successfully performed firstboot initial configuration steps...`

**No prompts for hostname, DNS, search domains, proxy, or firewall mode on 7.6.4.** Those default to `firepower` / Cisco+OpenDNS / no proxy / routed. Custom values go via FDM UI in Ch 6 or FTD CLI `configure network ...`.

## Doc changes committed locally (not yet pushed)

- `docs/remote-factory-reset.md` — Ch 3.5 rewritten around cross-version install truth
- `docs/production-upgrade.md` — **DELETED** (folded into Ch 3.5; two-chapter split was based on false assumption that same-version reset works)
- `docs/first-boot.md` — Ch 4 rewritten as post-reset verify + choose-your-next-chapter landing
- `docs/troubleshooting.md` — updated stale-command warning
- `mkdocs.yml` — Ch 3.6 nav entry removed
- `docs/lessons-2026-07-10.md` — comprehensive findings appendix (804 lines total)
- `captures/*.log` — session's raw pexpect transcripts (passwords redacted post-run in each)

## Key captures on disk

- `captures/staging-2026-07-10.txt` — package staging session
- `captures/fxos-syntax-verify-2026-07-10.log` — `install security-pack ?` help enumeration
- `captures/reset-execution-2026-07-10.log.v4-reimage-only` — first successful reimage (7.6.0 → 7.6.4)
- `captures/wizard-finish-2026-07-10.log` — FTD wizard drive
- `captures/post-wizard-verify-2026-07-10.log` — show version / network / managers
- `captures/reset-validate-2026-07-10.log` — validation reset in progress

## Sequence of resets on this box today

1. 12:10 EDT — misfired `configure factory-default` (invalid command, silent bell-reject) — no state change
2. 13:43 EDT — `install security-pack 7.6.0-113 force` — silent no-op (up-to-date)
3. 14:33 EDT — `install security-pack 7.6.4-69` — REAL reimage 7.6.0-113 → 7.6.4-69, UUID changed
4. 15:24 EDT — `install security-pack 7.6.0-113` — VALIDATION reset (in flight at compaction)