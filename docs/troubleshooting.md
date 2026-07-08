# Troubleshooting

!!! warning "⚠ WIP"
    Placeholder chapter with outline only. Content to be written as the lab is built. Feedback and PRs welcome.

Organized by symptom. Find your symptom, walk the checks, skip to next if not applicable.

## Console shows nothing after power on

- Cable is USB-C **data** (not charge-only)
- FW power is on (front LED)
- `/dev/ttyACM0` exists on ConsolePi (`lsusb`, `ls /dev/ttyACM*`)
- Cable fully seated (looser than expected on FW side)
- USB-C wins over RJ45 — if RJ45 is also plugged in, unplug it

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

Nuclear option:

```
> configure factory-default
```

...will wipe the FW back to factory. Then re-run the [First boot](first-boot.md) flow. Save your smart license token — you'll need to re-register.

## Still stuck?

- [Cisco Secure Firewall 1200 Series Getting Started Guide](https://www.cisco.com/c/en/us/support/security/secure-firewall-1200-series/products-installation-and-configuration-guides-list.html) is the authoritative reference
- [Cisco TAC](https://www.cisco.com/c/en/us/support/index.html) if you have a support contract
- Open an [issue](https://github.com/Batman4NY/firewall-1210ce-build-guide/issues) here if you found something wrong or unclear in this guide
