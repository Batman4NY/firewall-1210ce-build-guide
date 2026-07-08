# Talos intel + updates

!!! warning "⚠ WIP"
    Placeholder chapter with outline only. Content to be written as the lab is built. Feedback and PRs welcome.

**Prerequisites:** [Chapter 7: Security policies](security-policies.md) deployed.

Cisco Talos is the intelligence engine behind IPS and URL filtering. This chapter covers making sure updates flow automatically and Talos data is being pulled in.

## What Talos feeds into the FW

- **Snort rule updates** — new IPS signatures for emerging threats
- **URL categorization + reputation** — Talos scores every URL 1-100 and tags with categories
- **Malware / SHA feeds** — file hashes known bad
- **Security intelligence feeds** — bad IPs, malicious domains, DGA algorithms

Talos data updates every few minutes; FW pulls updates on a scheduled interval.

## Do the thing — Configure automatic updates

FDM → Device → Updates. Enable:

- **Rule updates**: automatic, daily
- **Geolocation database**: automatic, weekly
- **VDB (Vulnerability DataBase)**: automatic, weekly
- **URL filtering database**: automatic, daily

**Fill in:** exact click paths, screenshots.

## Verify

### Verify update flow

Check the recent updates in FDM to confirm rule/URL DB pulls are succeeding. If they fail:

- Verify outbound HTTPS to `updates.cisco.com` and `api.threatgrid.com` is allowed
- Verify Smart Licensing is in a good state (updates require valid entitlement)

### Verify Talos is actively pulling

FTD CLI:

```
> show version
```

...should show current Snort rule version + last update timestamp.

```
> system support diagnostic-cli
> show database processes
```

...will show if the DB update processes are healthy.

## Edge cases

### Talos intel in SCC (once onboarded)

Once the FW is onboarded to SCC (see [SCC onboarding](scc-onboarding.md)), Talos threat feed activity shows in the SCC event stream. This is where the "one pane" story earns its keep.

## Next

Head to [Onboarding to Security Cloud Control](scc-onboarding.md) to bring the FW under SCC management.
