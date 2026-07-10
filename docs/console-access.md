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

## Next

Head to [First boot and initial config](first-boot.md) to power the 1210CE on and run through the initial setup.
