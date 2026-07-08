# Licensing — eval to production

!!! info "📝 Draft"
    Structure and prose are in place and fact-checked against the Cisco Secure Firewall 7.6 licensing docs. **Screenshots will be added during the actual build session** (FDM Smart Licensing pages, SSM token-generation flow, entitlement views). Specific FDM 7.6 UI paths should also be confirmed against the FDM 7.6 config guide before publishing — everything below is consistent with 7.0–7.3 FDM guides.

The 1210CE ships with a **90-day evaluation** licensing mode that unlocks the full FTD feature set. This chapter covers how to work with it and how to transition to Smart Licensing before the 90 days run out.

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

FTD licensing on the 1210CE is layered: an **Essentials** (base) entitlement for the platform, plus three optional term-based subscription entitlements for feature groups:

- **IPS** (called **Threat** in older docs) — intrusion policies, file policies, and Security Intelligence feed downloads (URL / DNS / network SI)
- **Malware Defense** — malware cloud lookup (SHA lookups) and dynamic file analysis
- **URL Filtering** — access control rules matching on URL category / reputation, and the URL category/reputation feed updates

Each is a separate SKU. In eval mode, all three are automatically active on top of Essentials.

!!! note "Terminology"
    Cisco renamed a few of these in recent releases. If you see **Threat** in one doc and **IPS** in another, they're the same entitlement. Same for **Malware** → **Malware Defense**. This guide uses the current names.

## What happens when a license expires

The behavior when an entitlement expires (or when eval mode ends without Smart Licensing) is **per-entitlement**, and it isn't a graceful "fall back to cached data" story on all three:

- **IPS / Threat expiry** — intrusion policies and file policies stop being applied, and the system stops downloading Security Intelligence feed updates. Existing config stays in place, but the enforcement path drops out.
- **URL Filtering expiry** — access control rules with URL category conditions **immediately stop filtering URLs**, and the system stops downloading URL category/reputation updates. This is a hard stop on category-based filtering, not a degraded-cache mode.
- **Malware Defense expiry** — malware cloud lookups and dynamic file analysis stop; file policies themselves are gated by the IPS entitlement above.
- The device still functions as a **stateful L3/L4 firewall** — access control by IP / port / zone continues to work.

So the practical impact is: L4 firewalling survives; L7 inspection (IPS, malware lookup, URL categorization) largely does not. Plan to have Smart Licensing registered before the 90-day eval clock hits zero.

## Renewals

Smart Licensing is subscription-based. Term durations:

- 1 year, 3 year, 5 year typical
- Auto-renewal available per smart account settings
- Cisco Renewals team notifies ~90 days before expiry

## Deep-dive

The [official Cisco Smart Licensing guide](https://www.cisco.com/c/en/us/products/software/smart-accounts/software-licensing.html) is the authoritative source. This chapter is a lab-focused summary; production licensing has more nuance.

## Next

Head to [Troubleshooting](troubleshooting.md) if something isn't working.
