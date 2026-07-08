# Licensing — eval to production

!!! info "📝 Draft"
    This chapter has skeleton content — most of the structure is in place, specific screen captures and gotchas will be filled in during the actual build.

The 1210CE ships with **90-day evaluation** licensing that unlocks all features. This chapter covers how to work with it and transition to Smart Licensing before the 90 days run out.

## What eval mode unlocks

Full FTD 7.6 feature set:

- Access control (with all subscriptions)
- IPS + Snort updates
- URL filtering + reputation
- Malware inspection
- SSL decryption
- HA (if paired with a second unit)

Everything works. The 90-day clock starts on first boot.

## Track eval time

FDM → Device → Smart Licensing → shows days remaining on eval.

## When to convert to Smart Licensing

Options:

- **Cisco Employee Program** — most Cisco employees have a Cisco.com account with a smart account entitled to lab licensing. Register via Smart Software Manager.
- **Partner allocation** — Cisco Partners get NFR licenses for demo devices.
- **Purchased device** — commercial customers use their real smart account.

## Smart Licensing registration

- Log into [Cisco Smart Software Manager](https://software.cisco.com/software/csws/ws/platform/home)
- Generate a **new token** from your smart account/virtual account
- FDM → Smart Licensing → Register → paste the token
- FDM registers with cloud, pulls entitlements, activates features

## What entitles what

The 1210CE needs three entitlements for a full feature deployment:

- **Threat** — IPS + Snort rule feeds
- **Malware** — file inspection / SHA feeds
- **URL** — URL categorization + reputation

Each is a separate SKU. In eval mode, all three are automatically active.

## Unlicensed operation

If eval expires or Smart Licensing isn't set up:

- **IPS**: stops enforcing (deploys succeed but no new blocks)
- **URL filtering**: works with cached categories, no fresh updates
- **Malware**: works with cached hashes, no fresh feeds
- The FW still functions as a stateful firewall

Not a hard cliff — degraded but not dead.

## Renewals

Smart Licensing is subscription-based. Term durations:

- 1 year, 3 year, 5 year typical
- Auto-renewal available per smart account settings
- Cisco Renewals team notifies ~90 days before expiry

## Deep-dive

The [official Cisco Smart Licensing guide](https://www.cisco.com/c/en/us/products/software/smart-accounts/software-licensing.html) is the authoritative source. This chapter is a lab-focused summary; production licensing has more nuance.

## Next

Head to [Troubleshooting](troubleshooting.md) if something isn't working.
