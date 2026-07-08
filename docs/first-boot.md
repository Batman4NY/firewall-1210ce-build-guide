# First boot and initial config

!!! info "📝 Draft"
    Skeleton with the general FTD boot flow. Specific screen captures and exact prompt wording will be filled in during the actual build — anything marked `[verify against unit]` is generic knowledge that should be confirmed against what your specific 1210CE presents.

The 1210CE ships with an **FTD image pre-loaded** on the boot flash. First boot will bring you through the initial setup wizard where you'll:

- Set the `admin` password (from the shipped default)
- Choose management mode (**local** — FDM on-box — or **remote** — FMC / SCC)
- Configure the management interface
- Accept the EULA

## Before you power on

Make sure you have:

- ✅ Serial console access via [ConsolePi](console-access.md) established (telnet ConsolePi.local 9000)
- ✅ Ethernet cable ready for the **management interface** (usually labeled `MGMT`)
- ✅ An IP address plan for the management interface (either DHCP or static)

## Power on

Plug in the power cable. The 1210CE will boot within ~2-3 minutes to the initial FTD prompt. On the console:

```
Firepower login:
```

Default credentials:

- Username: `admin`
- Password: `Admin123` (default from Cisco) `[verify against unit — some ship with the serial number as the initial password]`

## Initial setup wizard

You'll be prompted through several steps:

### Step 1: Accept the EULA

Scroll through and type `YES` to accept.

### Step 2: Change the admin password

You'll be forced to change the admin password before proceeding. Choose something strong; put it in your password manager.

### Step 3: Configure the management interface

The wizard asks:

- **Do you want to configure IPv4?** — Yes
- **Do you want to configure IPv6?** — up to you
- **Configure IPv4 via DHCP or manual?** — for a lab, either works. Manual gives you a predictable IP:
    - IP address: (e.g., 192.168.1.10)
    - Netmask: 255.255.255.0
    - Gateway: 192.168.1.1
    - DNS: 1.1.1.1 or 8.8.8.8

### Step 4: Choose management mode

Two options:

- **Local (FDM)** — manage via the FDM web UI at `https://<mgmt-ip>` directly
- **Remote (FMC / SCC)** — device will register to a management center

For this guide, choose **Local (FDM)**. We'll add SCC management later, [after we have a working baseline](scc-onboarding.md).

### Step 5: Firewall mode

Choose **Routed** (Layer 3). This is what you want for a home lab acting as gateway/router.

### Step 6: Complete initial setup

The FW will do a final config apply and return you to a prompt.

## Verify management access

From your workstation on the same subnet as the mgmt interface:

```bash
ping <mgmt-ip>
# should respond
```

Then browse to:

```
https://<mgmt-ip>/
```

You should see the FDM login screen. Log in with `admin` and the password you set in Step 2.

## First FDM login

FDM will run a further setup dialog:

- **Complete initial setup** — set the outside interface, DNS servers, DHCP servers, etc. (covered in the [FDM baseline chapter](fdm-baseline.md))
- **Cloud services registration** — Cisco offers integration with cloud services from the very first login; you can defer this and come back to it in [the SCC onboarding chapter](scc-onboarding.md)
- **Smart licensing** — either register your smart account now, or start the **90-day evaluation** with all features enabled (recommended for lab)

Skip / defer whatever you're not ready to configure yet. FDM lets you come back and finish these later.

## Common first-boot issues

**Console shows garbled text on first boot.** Some 1210CE units ship with a serial baud mismatch on the initial boot menu — try `Ctrl-A K` to detach if using `screen`, then reconnect. Occasionally power-cycling helps.

**Password not accepted.** If `Admin123` is rejected, try the unit's serial number (found on the pull-out tag on the front). Cisco has shipped both defaults on different production batches.

**Wizard doesn't complete.** If the initial setup wizard errors out mid-way, you can restart it from the FTD CLI:

```
> configure manager local
```

...which resets the device to `local` (FDM) management mode. Then re-run the setup:

```
> reboot
```

...and go back through the wizard on the next boot.

## Next

Head to [FDM baseline — interfaces + routing](fdm-baseline.md) to configure the outside/inside interfaces and get the FW routing traffic.
