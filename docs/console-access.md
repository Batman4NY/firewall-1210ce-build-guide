# Console access via ConsolePi

!!! success "✅ Complete"
    This chapter is complete and follows the pattern from the [ConsolePi Build Guide](https://batman4ny.github.io/consolepi-build-guide/connect-cisco-usb-c/) — refer there for the underlying ConsolePi setup if you haven't built one yet.

Before you power on the 1210CE for the first time, get serial console access wired up. You will need the console to see initial boot output, respond to any prompts the FTD image throws at first boot, and set the admin password.

## What you need

- A working ConsolePi on your LAN (see the [ConsolePi Build Guide](https://batman4ny.github.io/consolepi-build-guide/))
- A **USB-A to USB-C data cable** (charge-only cables will not work — verify it can transfer data)
- Physical access to both the ConsolePi and the 1210CE

## Why USB-C (not RJ45)

The **Cisco Secure Firewall 1210CE** ships with a USB-C console port as a first-class option alongside the traditional RJ45. **The USB-C path is strictly easier than RJ45** for one reason: **the target device itself acts as the USB-serial converter**, so you don't need an FTDI or Prolific chip in the cable.

Any USB-A to USB-C data cable works.

## Physical connection

- **USB-A end** into any USB port on the ConsolePi
- **USB-C end** into the 1210CE's USB-C console port (labeled `CONSOLE` on the front panel)

!!! tip "RJ45 vs USB-C"
    The 1210CE has **both** an RJ45 console port and a USB-C console port. If both are plugged in simultaneously, **USB-C wins.** For lab use, USB-C is easier — one cable, no adapter.

## What the ConsolePi sees

The 1210CE enumerates as a USB CDC-ACM (Communications Device Class) device on the ConsolePi. From a shell on the ConsolePi:

```bash
lsusb | grep -i cisco
# Bus 001 Device 005: ID 05a6:0009 Cisco Systems, Inc. Console
```

A `/dev/ttyACM*` character device appears:

```bash
ls /dev/ttyACM*
# /dev/ttyACM0
```

Udev populates useful attributes:

```bash
udevadm info -q property /dev/ttyACM0 | grep -E '^ID_'
# ID_VENDOR=Cisco
# ID_MODEL=Cisco_USB_Console
# ID_MODEL_ID=0009
# ID_SERIAL=Cisco_Cisco_USB_Console
```

## ser2net port mapping

ConsolePi's `ser2net` config maps `/dev/ttyACM*` devices to telnet ports (see the [companion guide](https://batman4ny.github.io/consolepi-build-guide/connect-cisco-usb-c/#ser2net-port-mapping)):

| Device | Telnet port |
|---|---|
| `/dev/ttyACM0` | 9000 |
| `/dev/ttyACM1` | 9001 |
| `/dev/ttyACM2` | 9002 |
| `/dev/ttyACM3` | 9003 |

Assuming the 1210CE is the only USB-C console-attached device on the ConsolePi, it's `/dev/ttyACM0` → telnet port **9000**.

## Connect to the console

From any host on your LAN:

```bash
telnet ConsolePi.local 9000
# or
telnet <consolepi-ip> 9000
```

**Or via SSH + screen** (if you'd rather bypass ser2net):

```bash
ssh pi@ConsolePi.local
sudo screen /dev/ttyACM0 9600
```

Exit `screen`: `Ctrl-A` then `k` to kill, or `Ctrl-A` then `d` to detach.

## Verify the connection

Once connected and the 1210CE is powered on, you should see FTD boot output. If the FW isn't powered yet, hitting Enter should give you a blank line — that means the serial link is up, waiting for output from the FW.

If you see nothing at all — no response, no blank prompt — check:

- Cable is USB-C **data** (not charge-only)
- FW power is on
- The `/dev/ttyACM0` device actually exists (`lsusb`, `ls /dev/ttyACM*`)
- The USB-C cable is fully seated in both ends (looser than you'd expect on the FW)

See the [Troubleshooting chapter](troubleshooting.md) for more.

## Multiple devices

If you're consoled into the 1210CE **and** another device (say, a downstream Catalyst switch) on the same ConsolePi at once, they show up as separate `/dev/ttyACM*` numbered devices. Check `lsusb` to see which is which. `consolepi-menu` gives a friendlier device picker.

## Next

Head to [First boot and initial config](first-boot.md) to power the 1210CE on and run through the initial setup.
