# Console access via ConsolePi

!!! success "✅ Complete · Updated 2026-07-10"
    Chapter revised after a live-fire cable-quality failure — see [Why FTDI over USB-C](#why-ftdi-over-usb-c). The primary console path is now **FTDI + RJ45**; USB-C is the alternative. Follows the pattern from the [ConsolePi Build Guide](https://batman4ny.github.io/consolepi-build-guide/) — refer there for the underlying ConsolePi setup if you haven't built one yet.

Before you power on the 1210CE for the first time, get serial console access wired up. You need the console to see initial boot output, respond to the FTD setup wizard, and set the admin password. This chapter covers both the recommended **FTDI + RJ45** path and the **USB-C** alternative.

## What you need

- A working ConsolePi on your LAN (see the [ConsolePi Build Guide](https://batman4ny.github.io/consolepi-build-guide/))
- **A Cisco-style USB-A to RJ45 console cable with an FTDI FT232 chipset.** Cable Matters is the specific product used in this lab (~$18) — verify the FTDI chipset (not Prolific PL2303, which has driver quirks). Physical access to both the ConsolePi and the 1210CE.

## Why FTDI over USB-C

The 1210CE has **both** an RJ45 console port and a native USB-C console port. Both work. The question is which one you make your primary.

- **FTDI + RJ45 — pick this as primary.** The FTDI FT232 is battle-tested silicon with mature drivers on Linux/macOS/Windows. Data transmission is bit-perfect and deterministic. A Cable Matters FTDI console cable is a known-good, reproducible unit — order the same model and it will work.
- **USB-C native — the alternative.** Convenient (the 1210 is its own USB-serial converter, so any USB-A → USB-C data cable "works") but data-line quality varies wildly between cables. A marginal USB-C cable can enumerate cleanly, look fine, and then silently corrupt bytes — the worst failure mode possible for a console you're relying on during a build.

!!! warning "Cable-quality gotcha we hit on 2026-07-10"
    A short after-market USB-C cable installed during a rack rewire enumerated fine on the ConsolePi as CDC-ACM but corrupted bytes on data transmission. `show version` (short output, fast) succeeded; `show network` (follow-up command with more I/O) returned `show\x07network\x07` — the CLI rejecting the corrupted characters with bells. Swapping to the FTDI + RJ45 path immediately produced clean output. Details in [lessons-2026-07-10.md](lessons-2026-07-10.md). Two takeaways:

    1. **The USB-C cable that ships in the box with the 1210CE is your known-good baseline.** It's Cisco-supplied and vetted for data quality. If you use it, USB-C is safe.
    2. **If you swap it for something tidier or shorter, verify data quality before you rely on it.** Symptom of a bad cable: intermittent bell characters, `show` commands that half-execute, or CLI sessions that "work for a moment then hang." See [Verify the connection](#verify-the-connection) below.

!!! tip "USB-C wins when both plugged"
    The 1210CE routes console output to USB-C whenever both USB-C and RJ45 are plugged in. If FTDI + RJ45 is your primary path and USB-C is also plugged in for redundancy, the RJ45 side stays silent — you'll be talking to the USB-C interface without knowing it. Either unplug USB-C when relying on FTDI, or use consolepi-menu to explicitly pick the FTDI `/dev/ttyUSB*` device.

## Physical connection — FTDI + RJ45 (primary)

- **USB-A end** of the FTDI cable into any USB port on the ConsolePi
- **RJ45 end** into the 1210CE's console port (labeled `CONSOLE` on the front panel)

## Physical connection — USB-C (alternative)

- **USB-A end** of a **USB-A to USB-C data cable** into any USB port on the ConsolePi
- **USB-C end** into the 1210CE's USB-C console port
- Charge-only USB-C cables will not work at all — device won't enumerate as CDC-ACM.
- Marginal-quality data cables enumerate but corrupt data — see the warning above.

## What the ConsolePi sees

### FTDI + RJ45 path (`/dev/ttyUSB0`)

The FTDI cable enumerates as a USB FT232 serial device on the ConsolePi:

```bash
lsusb | grep -i ftdi
# Bus 001 Device 007: ID 0403:6001 Future Technology Devices International, Ltd FT232 Serial (UART) IC
```

A `/dev/ttyUSB*` character device appears:

```bash
ls /dev/ttyUSB*
# /dev/ttyUSB0
```

### USB-C native path (`/dev/ttyACM0`)

The 1210CE's USB-C interface enumerates as a USB CDC-ACM device:

```bash
lsusb | grep -i cisco
# Bus 001 Device 005: ID 05a6:0009 Cisco Systems, Inc. Console
```

A `/dev/ttyACM*` character device appears:

```bash
ls /dev/ttyACM*
# /dev/ttyACM0
```

Udev attributes are useful for identifying which is which if you have multiple consoled devices:

```bash
udevadm info -q property /dev/ttyACM0 | grep -E '^ID_'
# ID_VENDOR=Cisco
# ID_MODEL=Cisco_USB_Console
udevadm info -q property /dev/ttyUSB0 | grep -E '^ID_'
# ID_VENDOR=FTDI
# ID_MODEL=FT232R_USB_UART
```

## ser2net port mapping

ConsolePi's `ser2net` config maps devices to **different port ranges** depending on device family. The FTDI (`ttyUSB*`) and USB-C (`ttyACM*`) paths live in different port ranges:

| Device | Family | Telnet port |
|---|---|---|
| `/dev/ttyUSB0` | FTDI · **primary** | **8000** |
| `/dev/ttyUSB1` | FTDI | 8001 |
| ... | | up to 8020 |
| `/dev/ttyACM0` | CDC-ACM · USB-C | 9000 |
| `/dev/ttyACM1` | CDC-ACM | 9001 |
| ... | | up to 9008 |

Assuming the 1210CE is the only FTDI-consoled device on the ConsolePi, it's `/dev/ttyUSB0` → telnet port **8000**. If you use the USB-C path instead, it's `/dev/ttyACM0` → telnet port **9000**.

!!! danger "Common pitfall"
    Do not assume both paths map to the same port range. The build guide's earlier draft (and this author) spent significant time hitting `:9000-9003` for both paths — the FTDI path did not respond because it's actually on the `:8000-8020` range. Match the port to the device family: `ttyUSB*` → 8000+, `ttyACM*` → 9000+.

## Connect to the console

From any host on your LAN:

```bash
# Primary — FTDI + RJ45
telnet ConsolePi.local 8000
# or
telnet <consolepi-ip> 8000

# Alternative — USB-C
telnet ConsolePi.local 9000
```

**Or via SSH + screen** (bypass ser2net entirely):

```bash
ssh pi@ConsolePi.local
# FTDI path
sudo screen /dev/ttyUSB0 9600
# USB-C path
sudo screen /dev/ttyACM0 9600
```

Exit `screen`: `Ctrl-A` then `k` to kill, or `Ctrl-A` then `d` to detach.

**Or via `consolepi-menu`** (friendlier device picker):

```bash
ssh pi@ConsolePi.local
consolepi-menu
```

Then pick the numbered entry for `ttyUSB0` (FTDI) or `ttyACM0` (USB-C).

## Verify the connection

Once connected and the 1210CE is powered on, you should see FTD boot output or an `fw1210ce login:` prompt. If the FW isn't powered yet, hitting Enter should give you a blank line — that means the serial link is up, waiting for output from the FW.

**Signals that indicate a data-quality problem on the cable** (usually USB-C):

- CLI echoes `\x07` (bell) characters instead of running your commands
- `show` commands complete partially and then fail on the next call
- Session "works for a moment then hangs"
- On the ConsolePi side, `dmesg | grep -i cdc_acm` shows repeated `cdc_acm: urb submit failed` or `urb stopped` messages

If any of the above: **switch to the FTDI path** as a discriminator test. If FTDI is clean, the USB-C cable is confirmed bad — replace it (or fall back to the Cisco-shipped cable that came in the box).

**Common connection issues:**

- Cable is charge-only, not data (USB-C only — no enumeration means the cable can't carry data)
- FW power is off (no output at all)
- `/dev/ttyUSB0` or `/dev/ttyACM0` doesn't exist (device not detected — try re-seating, different USB port on the Pi, or `sudo modprobe ftdi_sio` for FTDI)
- Cable not fully seated (both USB-C and RJ45 need firmer insertion than expected)
- Wrong ser2net port (`:8000` for FTDI, `:9000` for USB-C — see [ser2net port mapping](#ser2net-port-mapping))

See the [Troubleshooting chapter](troubleshooting.md) for more depth.

## Multiple devices

If you have the 1210CE consoled **and** another device (say, a downstream Catalyst switch or another firewall) on the same ConsolePi at once, they show up as separate `/dev/ttyUSB*` or `/dev/ttyACM*` numbered devices. Check `lsusb` to see which vendor is which; `consolepi-menu` shows all attached devices with their bauds so you can pick.

## picocom keybindings — reference

`consolepi-menu` option 1 launches `picocom /dev/ttyUSB0 --baud 9600`. Picocom's escape prefix is `Ctrl-A`. Common combinations you'll actually use:

| Combo | What it does |
|---|---|
| `Ctrl-A  Ctrl-X` | **Exit picocom** (clean — restores tty, releases `/dev/ttyUSB0` so ser2net can serve it) |
| `Ctrl-A  Ctrl-Q` | Exit without resetting tty (leave stale settings behind — rarely what you want) |
| `Ctrl-A  Ctrl-B` | **Change baud rate** — prompts for new value. Easy to hit by accident; if your screen goes to garbage after typing something into picocom, this is the first thing to check |
| `Ctrl-A  Ctrl-A` | Send a literal `Ctrl-A` through to the FTD (rare — but the ASA-CLI diag mode uses `Ctrl-A d` for detach, so you'd need this to send that sequence to picocom's inner target) |
| `Ctrl-A  Ctrl-C` | Toggle local echo |
| `Ctrl-A  Ctrl-H` | Show picocom's own help / status line |
| `Ctrl-A  Ctrl-P` | Pulse DTR (sends a hardware reset signal to the FW's console port — useful only for platforms that support it; harmless on 1210CE) |

!!! warning "Reset picocom's baud back to 9600 if you drift"
    On the 1210CE, the console UART is **hardcoded at 9600 8N1** — you can't change the FW's side. If picocom's baud gets accidentally changed via `Ctrl-A Ctrl-B`, hit `Ctrl-A Ctrl-B` again and enter `9600`. Or `Ctrl-A Ctrl-X` to exit and relaunch via `consolepi-menu` (which always starts at 9600).

## FTD ↔ FXOS prompt navigation

The 1210CE has two CLI planes stacked. Which one you're at depends on how you connected:

| How you reached it | Landing prompt | To reach the other plane |
|---|---|---|
| **Serial console** (via picocom / ser2net telnet `:8000`) | FXOS `firepower#` first, then `connect ftd` → FTD `>` | From FTD `>` — type `exit` to return to FXOS `firepower#`. **`connect fxos` from FTD console is a no-op** — box replies `You came from FXOS Service Manager. Please enter 'exit' to go back.` |
| **SSH to mgmt IP** as `admin` | Straight to FTD `>` (bypasses FXOS on this platform) | From FTD `>` — type `connect fxos`, land at FXOS `firepower#`. `exit` here **closes SSH** rather than returning to FXOS |

The two exit-behavior differences are the classic 1200-series footgun. If you're scripting via SSH and expect `exit` to bounce you between planes, you'll drop the whole session instead.

## Diagnostic CLI (ASA-like mode) — detach without exiting

From the FTD `>` prompt, `system support diagnostic-cli` drops you into an ASA-flavored CLI (`ciscoasa#`). To return to FTD `>` without closing SSH:

```
Ctrl-A  d
```

That's Ctrl-A followed by lowercase `d` (not Ctrl-D). If you're in picocom on top of the console AND in diagnostic-cli inside the FTD, you'd need Ctrl-A Ctrl-A first to send the Ctrl-A through picocom to the FTD, then a plain `d` — a nested-escape gotcha.

## Console recovery cheat sheet

| Symptom | Fastest fix |
|---|---|
| `telnet <ip> 8000` returns `Device open failure: Object was already in use` | Something has `/dev/ttyUSB0` exclusive. Usually picocom from a prior `consolepi-menu` session. `ssh pi@<consolepi> "pgrep -a picocom"` to identify, then kill or `Ctrl-A Ctrl-X` from that session |
| Console shows silence — FW seems alive, no prompt | Old input in FTD's line buffer. Send `Ctrl-U` (line kill) then `\r` to reset the input state and reprint the prompt |
| Console emits garble instead of characters | Baud mismatch. Verify picocom is at 9600 (`Ctrl-A Ctrl-B`, type `9600`) — the 1210CE UART is fixed at 9600 |
| `--More--` pager stuck from a `show` command a while ago | Send `q` to escape the pager, then `\r` to freshen the prompt |
| Picocom stuck, `Ctrl-A Ctrl-X` doesn't respond | Kill the picocom process from ConsolePi over SSH: `sudo pkill picocom`. Then relaunch via `consolepi-menu` |
| Ctrl-U + \r still gets nothing | Only ONE process can hold `/dev/ttyUSB0`. If picocom is running, ser2net is locked out (and vice-versa). Verify with `pgrep -a picocom` + `sudo fuser /dev/ttyUSB0` on ConsolePi |

## Layer 0 stays active across every management model

The ConsolePi + ser2net + FTDI setup you just built isn't just for the Ch 4 first-boot walkthrough. It's a **permanent Layer 0 investment** that keeps working regardless of which management model owns the box.

The physical UART on the FTD's RJ45 console port runs BELOW every management-plane software layer (FDM, cdFMC, on-prem FMC, FMCv). Console access is provided by the FTD kernel + UART hardware; it doesn't ask a manager for permission to answer keystrokes.

### What still works via console across every management transition

| Console operation | FDM only | FDM + SCC hybrid | cdFMC-managed | On-prem FMC | FMCv |
|---|:---:|:---:|:---:|:---:|:---:|
| Login as `admin` with the OpenBao-stored password | ✅ | ✅ | ✅ | ✅ | ✅ |
| FTD `>` prompt — `show version`, `show network`, `show interfaces` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `system support diagnostic-cli` → ASA-like CLI, packet-tracer, capture | ✅ | ✅ | ✅ | ✅ | ✅ |
| `configure network ipv4 manual …` for mgmt IP recovery | ✅ | ✅ | ⚠️ next-deploy may overwrite | ⚠️ next-deploy may overwrite | ⚠️ next-deploy may overwrite |
| FXOS access via `connect fxos` (SSH) or `exit` from FTD `>` (console) | ✅ | ✅ | ✅ | ✅ | ✅ |
| `show tech-support` for TAC bundles | ✅ | ✅ | ✅ | ✅ | ✅ |
| **`configure manager delete`** — break-glass "get me back to FDM" | n/a | ✅ | ✅ | ✅ | ✅ |
| Password reset via console + Admin123 recovery | ✅ | ✅ | ✅ | ✅ | ✅ |
| Watching boot messages during a reboot | ✅ | ✅ | ✅ | ✅ | ✅ |

### Why the investment matters more post-cdFMC (or any cloud manager)

Two new failure modes appear the moment you delegate management to the cloud:

1. **Manager unreachable** — cloud outage, tenant issue, network path broken. The FTD's data plane keeps forwarding traffic, but you can't change policy from the cloud pane. Console is your ONLY inspection + emergency-modify path.
2. **Manager endpoint migration** — SCC / cdFMC endpoints occasionally shift as Cisco rolls out infrastructure changes. sftunnel can break silently — the console is where you see the sftunnel logs and can act on them before the customer notices.

For a customer engagement, the "we set up ConsolePi + ser2net once in Ch 3" work becomes a **permanent break-glass path** — pays off every time the cloud manager has a bad day.

### What's DIFFERENT on the console after moving to a cloud manager

Nothing dramatic. Three subtle behaviors worth knowing:

1. **`show running-config`** reflects the manager-pushed policy, not what FDM would have authored locally. Objects show up with cloud-generated names.
2. **`> configure ...`** commands that would conflict with manager-owned config may print `"Device is managed by external manager"` warnings — cloud-owned config resists local edits. Local overrides may last until next deploy, then get wiped.
3. **FDM UI** (the HTTPS side) shows a **"Managed by cdFMC"** banner and turns most policy edits read-only. That's a UI-side thing; console CLI stays open and works.

### The break-glass sequence (when everything else is broken)

If a cloud manager becomes permanently unreachable and you need to cut the box back to FDM standalone:

```
> configure manager delete
```

That severs the sftunnel and returns the FTD to standalone mode. Policy authored by the cloud manager stays on the box until the next deploy — which now has to come from FDM after you log back into the on-box FDM UI. See [Ch 3.5](remote-factory-reset.md) if you want a full clean slate instead of the delete.

## Next

Head to [First boot and initial config](first-boot.md) to power the 1210CE on and run through the initial setup.
