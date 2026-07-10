# Pre-compaction state snapshot — 2026-07-10 late-morning

Written just before Claude conversation compaction. Everything below is CONFIRMED by direct in-session evidence — must survive the compaction boundary.

## The 1210 — actual state right now

- **Mgmt IP:** `192.168.10.59` (via DHCP) — **CHANGED SUBNET**: now on **AI subnet 192.168.10.0/24**, not the 192.168.1.0/24 home LAN where it was yesterday. Rack rewire moved it. Yesterday's `.161` no longer answers.
- Default gateway: `192.168.10.1`
- DNS: `192.168.1.3` (cross-subnet — home LAN's Pi-hole)
- Domain: `uppernyack.com`
- MAC (management0): `78:11:9d:6a:80:80` (unchanged)
- UUID: `dc0bef0a-66b5-11f1-9951-911e3bad6766` (unchanged — same physical box)
- Version: FTD `7.6.0 (Build 113)` on FXOS `2.16(0.128)`
- Manager mode: confirmed `Managed locally` (Layer 1, FDM standalone) — journey did NOT regress
- **OpenBao entry `infra/webui/fw1210ce-admin`** updated with new `host` + `fdm_url`

## Console access — post-rewire

**PRIMARY (working, clean):**
- FTDI RJ45 → ConsolePi USB-A → `/dev/ttyUSB0` → ser2net TCP `:8000`
- Command: `telnet 192.168.1.121 8000`
- Wake sequence: `Ctrl-C`, `Ctrl-U`, `Enter` → FTD `>` prompt
- Data is bit-perfect — `show network` runs clean

**BROKEN (do not reconnect):**
- USB-C native CDC-ACM → `/dev/ttyACM0` → ser2net TCP `:9000`
- Replacement USB-C cable installed during rewire enumerates as CDC-ACM but drops/corrupts bytes on data line
- Symptom: short commands (`show version`) succeed; follow-up commands (`show network`) return `show\x07network\x07` bells
- Currently unplugged; do not reconnect until known-good USB-C data cable is on hand

## The port-mapping misassumption that ate ~40 min

`/etc/ser2net.yaml` config on the Pi maps:
- `/dev/ttyUSB*` (FTDI) → ports **8000-8020**
- `/dev/ttyACM*` (CDC-ACM/USB-C) → ports **9000-9008**

I spent significant session time probing `9000-9003` for BOTH console paths. Should have probed `8000` for FTDI first. Once I did, everything worked immediately.

## ConsolePi

- IP: `192.168.1.121` (LAN DHCP for now — static planned later today)
- SSH working; **credentials in OpenBao at `infra/ssh/consolepi`** (user `pi`, password stored, my pub key added)
- Hardware: new permanent Pi (`b8:27:eb:01:cc:ba` / CPU serial `fa01ccba`) — different from yesterday's Pi
- ser2net config: `/etc/ser2net.yaml`, service active
- `/dev/ttyUSB0` present (FTDI FT232R, serial `BG03CZLH`, VID `0403`)
- `consolepi-menu` shows `1. ttyUSB0 [9600 8N1]` — enumeration confirmed working

## Root-cause verdicts (evidence-backed)

- **CONFIRMED**: replacement USB-C cable installed during rewire is data-line marginal. Enumerates as CDC-ACM but corrupts bytes on data path.
- **FALSIFIED**: earlier hypothesis that `connect ftd` from FXOS drops into a restricted CLI mode. Not the cause — FTDI path from FXOS via `connect ftd` runs `show network` clean.

## OpenBao entries (all current as of this snapshot)

- `infra/webui/fw1210ce-admin` — admin login for FTD/FDM UI. `host=192.168.10.59`, `fdm_url=https://192.168.10.59`, password set 2026-07-09 via console reset
- `infra/ssh/consolepi` — SSH login for the ConsolePi. `host=192.168.1.121`, user `pi`, password stored

## Immediate next-move queue (post-compaction)

1. **Order a known-good USB-C data cable** — restores primary console path (currently only FTDI works)
2. **Set static IPs** — 1210 mgmt + ConsolePi (10-min task, resolves DHCP-re-lease-on-reboot gotcha permanently). New Q: keep 1210 on 192.168.10.x AI subnet, or move back to 192.168.1.x home LAN?
3. **Factory reset + Ch 4 first-boot capture** (~15 min) — today's original plan, now unblocked by working console
4. **SCC onboarding** — Layer 1 → Layer 2, capture Ch 9 material

## Cross-references

- Full lessons doc (assumption audit + gotchas + runbooks): [`lessons-2026-07-10.md`](lessons-2026-07-10.md)
- Site now on metallic-paper aesthetic (from yesterday's work) — no state change needed
- 1210CE build guide ch 5 "Choosing your management path" published + sanity-reviewed clean
- Peer intro draft still queued (ready to send when Fabian wants)
