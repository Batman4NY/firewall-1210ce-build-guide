# Troubleshooting

!!! warning "⚠ WIP"
    Placeholder chapter with outline only. Content to be written as the lab is built. Feedback and PRs welcome.

**Symptoms are grouped by build phase.** Jump to the phase that matches when the issue appeared: [Console](#console-shows-nothing-after-power-on) · [First boot / FDM reachability](#cannot-reach-fdm-after-first-boot) · [Deploy](#deploy-fails) · [Talos updates](#talos-updates-not-pulling) · [URL filtering](#url-filtering-not-blocking-test-urls) · [SCC onboarding](#scc-onboarding-fails) · [Duo](#duo-login-loop) · [Umbrella](#umbrella-tunnel-down) · [ThousandEyes](#thousandeyes-agent-not-registering) · [Nuclear reset (Ch 3.5)](remote-factory-reset.md).

Organized by symptom. Find your symptom, walk the checks, skip to next if not applicable.

## Console shows nothing after power on

Fastest triage is the FTDI + RJ45 primary path — it isolates cable-quality issues from platform issues in one step. Full details in [Ch 3 — Console access](console-access.md).

- FW power is on (front LED)
- **FTDI cable enumerated on ConsolePi**: `lsusb | grep FTDI` returns `0403:6001 FT232`
- **`/dev/ttyUSB0` exists** on ConsolePi (`ls /dev/ttyUSB*`) — primary FTDI + RJ45 path
- **ser2net serving port `:8000`** for the FTDI: `telnet <consolepi-ip> 8000`
- Nothing else holds `/dev/ttyUSB0` exclusive (no `picocom` process from a stale `consolepi-menu` session)
- If you're on the alternative USB-C path: cable is **data**, not charge-only; `/dev/ttyACM0` exists (`ls /dev/ttyACM*`); ser2net port is `:9000` — see the [ser2net port mapping table](console-access.md#ser2net-port-mapping)
- Note: the 1210CE routes console output to USB-C when both interfaces are plugged in. If FTDI is your primary but USB-C is also connected, unplug USB-C or explicitly pick `/dev/ttyUSB0` via `consolepi-menu`.

## Cannot reach FDM after first boot

**Fill in:** MGMT IP verification, subnet checks, browser HTTPS cert warnings, common firewall rules that block MGMT from LAN.

## Deploy fails

**Fill in:** common deploy errors, rule compilation issues, object dependency conflicts.

## Talos updates not pulling

- Outbound HTTPS to `updates.cisco.com` and `api.threatgrid.com` allowed?
- Smart Licensing state OK?
- FDM → Updates page — check last-attempted timestamp + error

## URL filtering not blocking test URLs

- Access control rule has URL filtering enabled?
- Deploy completed successfully?
- Test URL is in a blocked category (verify against Talos categorization)
- HTTPS traffic requires SSL decryption for full URL visibility

## SCC onboarding fails

**Fill in:** common SCC onboarding errors — expired tokens, network reachability, smart license state.

## Duo login loop

**Fill in:** SAML config issues, IdP metadata mismatch, callback URL problems.

## Umbrella tunnel down

- IKE Phase 1 params match Umbrella side?
- PSK correct?
- Outside interface reachable from Umbrella POP?
- FTD's IKE traffic allowed outbound (should be by default)

## ThousandEyes agent not registering

- Pi has outbound internet?
- Cisco email address on the agent registration matches your tenant?
- Rate limit not hit (240 req/min TE API)

## FW in a weird state

Nuclear option: **[Ch 3.5 — Remote factory reset](remote-factory-reset.md)**. That runs the Cisco-supported FXOS cross-version reimage and returns the FW to first-boot state entirely over the console + management network.

Save your smart license token first — you'll need to re-register.

!!! warning "Stale commands in earlier drafts of this guide"
    Older drafts and Cisco Community posts reference `> configure factory-default` at the FTD `>` prompt, or `erase configuration` in FXOS, or `install security-pack version <same-version> force` — **none of these actually reset a 1200-series box on FTD 7.6.x**. Enumerated live on the hardware. Use [Ch 3.5](remote-factory-reset.md), which uses `install security-pack version <different-version>` — the only Cisco-supported CLI reset that actually reimages.

!!! warning "Stale command in earlier drafts of this guide"
    Older versions of this chapter (and older Cisco community posts) point at `> configure factory-default` at the FTD `>` prompt. That command **does not exist on FTD 7.6.0-113 for the 1200 series** — enumerated live on the box, no match. The build's factory-reset surface is FXOS, not FTD. Use [Ch 3.5](remote-factory-reset.md).

## Still stuck?

- [Cisco Secure Firewall 1200 Series Getting Started Guide](https://www.cisco.com/c/en/us/support/security/secure-firewall-1200-series/products-installation-and-configuration-guides-list.html) is the authoritative reference
- [Cisco TAC](https://www.cisco.com/c/en/us/support/index.html) if you have a support contract
- Open an [issue](https://github.com/Batman4NY/firewall-1210ce-build-guide/issues) here if you found something wrong or unclear in this guide
