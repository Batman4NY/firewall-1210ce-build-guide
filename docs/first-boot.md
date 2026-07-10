# First boot verification + choose your next step

!!! success "✅ Complete · Rewritten 2026-07-10 after live capture"
    Post-reset verification and routing to the next chapter. The actual first-boot flow (FXOS login + FTD wizard) is captured in [Ch 3.5 — Remote factory reset](remote-factory-reset.md) inline with the reset procedure — this chapter is the acceptance checkpoint after the wizard completes.

You just ran [Ch 3.5](remote-factory-reset.md), the FXOS reimage completed, and the FTD wizard landed you at the `>` prompt. This chapter is the post-reset acceptance checkpoint before you dive into management configuration.

## Verify the reset actually happened

At the FTD `>` prompt:

```
> show version
Model:    Cisco Secure Firewall 1210CE Threat Defense (86) Version <target>
UUID:     <NEW UUID>                                          ← must differ from pre-reset UUID
VDB:      <build>

> show network
Hostname: firepower                    ← factory default (wizard doesn't prompt)
DNS:      208.67.222.222, ...          ← Cisco/OpenDNS defaults
IPv4:     Configuration: DHCP  →  <mgmt-ip> / 255.255.255.0
Gateway:  <gw>
MAC:      <mac>                        ← unchanged, hardware MAC

> show managers
No managers configured.                ← wizard set local-mgmt but no external manager attached
```

**Three checks that confirm a real reimage** (not an upgrade-in-place or a same-version no-op):

1. **UUID changed** — the pre-reset UUID and the post-reset UUID must differ. If they match, `install security-pack` short-circuited with `Firmware Upgrade Message: up-to-date` and you got no reimage. Go back to [Ch 3.5 pre-flight](remote-factory-reset.md#pre-flight-inventory-of-what-s-on-the-fw) and confirm your target version differs from the running version.
2. **`show network` reports factory-default hostname** `firepower` and Cisco/OpenDNS resolvers — the wizard doesn't prompt for these on 7.6.4, so if you see the custom hostname/DNS you set previously, the reset didn't wipe FTD config.
3. **`show managers`** returns `No managers configured` — any prior FMC/SCC registration was cleared.

If all three check out, the reset is confirmed.

## Sanity-check management reachability

From your workstation on the same subnet as the mgmt interface, ping the mgmt IP that `show network` reported:

```
$ ping -c 3 <mgmt-ip>
```

Should respond. Then browse to `https://<mgmt-ip>/` — you should get the FDM login screen. Log in with `admin` and the password you set during the FXOS forced-password-change.

If FDM's cert warning bothers you, that's expected on a fresh reset — the cert is self-signed from FXOS and won't match any hostname. Ch 6 covers replacing it if you want that fixed early.

## Custom configuration deferred to FDM baseline

The 7.6.4 wizard is intentionally minimal — it doesn't prompt for hostname, DNS servers, search domains, proxy, or firewall mode. Everything except IPv4 DHCP/manual and manager-local/remote defaults to sensible values.

You'll set the following via FDM UI in [Ch 6 — FDM baseline](fdm-baseline.md):

- **Custom hostname** (e.g., `fw1210ce.uppernyack.com`)
- **Custom DNS servers** (e.g., `192.168.1.3` for a local Pi-hole)
- **Search domains** (e.g., `uppernyack.com`)
- **Static mgmt IP** (if you want to lock the DHCP-leased address)
- **NTP servers** (either Cisco defaults or local NTP)
- **Outside interface** (Ethernet1/1 by default)
- **Inside interface bridge** (Ethernet1/2 with VLAN1 as default)
- **Smart licensing** (either register your smart account now, or start the 90-day evaluation)

## Choose your management path

The 1210CE supports four management modes, each with its own onboarding flow. Pick based on the customer / lab scenario:

| Path | Chapter | When to use |
|---|---|---|
| **FDM standalone** (Layer 1) | [Ch 6 — FDM baseline](fdm-baseline.md) | Small deployments, POC, home labs. Local management via the on-box FDM UI. |
| **SCC hybrid** (Layer 2) | [Ch 8 — SCC onboarding](scc-onboarding.md) | Devices managed centrally by Cisco Security Cloud Control with local FDM as fallback. |
| **cdFMC** (Layer 3) | [Ch 5 — Choose management path](choose-mgmt-path.md#l3-cloud-delivered-fmc-cdfmc) | Cloud-delivered FMC for large multi-tenant deployments. |
| **FMCv** (Layer 3) | [Ch 5 — Choose management path](choose-mgmt-path.md#l3-fmcv) | On-premises Firewall Management Center virtual appliance for full-featured management. |

Not sure which fits? Read [Ch 5 — Choosing your management path](choose-mgmt-path.md) — it has the decision matrix and criteria.

## Common first-boot issues

**FDM UI shows a `Setup did not complete` banner after login.** The wizard ran but the FTD hasn't finished expanding its policy engine (usually within 2-3 min after the `>` prompt appears). Refresh the browser after a minute; the banner should clear.

**Mgmt IP is unexpected / on the wrong subnet.** The wizard used DHCP — you got whatever the upstream DHCP server offered. If that's not the subnet you wanted, either:

1. Change the upstream DHCP scope or move the mgmt cable to the right port, then reboot the FW to re-lease.
2. Set a static IP via FTD CLI: `> configure network ipv4 manual <ip> <netmask> <gw>`.
3. Set a static IP via FDM UI in [Ch 6 — FDM baseline](fdm-baseline.md).

**Can't reach FDM from workstation.** Confirm:

- Workstation is on a subnet with a route to the mgmt IP.
- Any intermediate firewall rules allow TCP 443 to the mgmt IP.
- Mgmt interface has `Link: up` in `show network`.

**Wizard prompted for password but didn't take it.** The FXOS-side login prompts for password ONCE (forced change). The FTD side does NOT prompt for password on `connect ftd` — the FXOS admin credential is inherited. If you see a "password" prompt inside FTD's flow, it's likely an FTD sub-command prompt (e.g., `configure network dns` might ask for confirmation), not a wizard prompt.

## Next

Head to [Ch 6 — FDM baseline](fdm-baseline.md) for FDM-standalone deployments, or [Ch 5 — Choose your management path](choose-mgmt-path.md) if you're still deciding.
