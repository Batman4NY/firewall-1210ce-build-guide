# Security policies — access + IPS + URL

!!! warning "⚠ WIP"
    Placeholder chapter with outline only. Content to be written as the lab is built. Feedback and PRs welcome.

**Prerequisites:** [Chapter 5: FDM baseline](fdm-baseline.md) — inside/outside zones and routing must be live.

Get baseline security policies in place: access control (allow inside→outside, deny inbound), Snort IPS, URL filtering with Talos categorization.

## Setup steps

### Access control policy

- Default action: **Block** (fail-closed)
- Add rule: `inside_zone → outside_zone`, action: **Allow with IPS + URL filtering**
- Add rule: inbound management traffic — allow specific admin sources to MGMT interface only

**Fill in:** exact rule ordering, service objects for admin protocols, syslog/logging config.

### Snort IPS

- Enable IPS on the allow rules
- IPS rule set: **Balanced Security and Connectivity** (default) or **Security over Connectivity** for lab
- Snort version: Snort 3 (default in FTD 7.6)

**Fill in:** rule tuning for false positives, custom rules, IPS event severity thresholds.

### URL filtering

- Enable URL filtering on the outbound allow rule
- Category-based blocking:
    - Block: Malware, Phishing, Command and Control, Cryptomining, Illegal, Uncategorized
    - Warn: Gambling, Weapons
    - Allow: everything else
- Reputation-based:
    - Block: 1-30 (High Risk, Suspect)
    - Warn: 31-60
    - Allow: 61-100

**Fill in:** custom categories, allow lists for lab domains, HTTPS inspection considerations.

### Application detection

- Application filtering: block P2P, cryptomining, anonymizers
- Log everything (lab-scale is fine)

### Malware / File policy

- Enable file inspection on the outbound allow rule
- File policy: **Block Malware** for common types (PE, PDF, archives)

## Do the thing — Deploy

Same drill as FDM baseline — Deploy takes ~2-3 min. Watch the deploy log for rule compilation errors.

## Verify

- Browse to a test malicious URL (Talos test page): `http://www.testmyids.com/` — should be blocked
- Try downloading a test EICAR file — should be blocked
- Try a P2P protocol (if any lab clients have it) — should be blocked

## Next

Head to [Talos intel + updates](talos-intel.md) to make sure the IPS + URL feeds stay current automatically.
