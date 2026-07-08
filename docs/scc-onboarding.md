# Onboarding to Security Cloud Control

!!! warning "⚠ WIP"
    Placeholder chapter with outline only. Content to be written as the lab is built. Feedback and PRs welcome.

**Prerequisites:** [Chapter 4: First boot](first-boot.md) + outbound HTTPS to `*.cdo.cisco.com`.

Bring the 1210CE under **Cisco Security Cloud Control (SCC)** management. SCC is the cloud pane-of-glass for the Cisco security portfolio — the central place customers manage FTD, Umbrella, Duo, XDR, etc.

## What you need

- SCC tenant provisioned (via your Cisco.com credentials)
- 1210CE has FDM configured and a working outbound path (Talos updates working = outbound HTTPS working)
- Smart Licensing registered OR in valid eval state

## Why this approach — onboarding methods

Two main paths:

- **Cloud onboarding** — SCC pulls the FW config over the internet
- **Serial number claim** — provide the FW's serial number to SCC, then confirm on-box

For a lab, cloud onboarding is simpler.

## Do the thing — Cloud onboarding walk-through

**Fill in:** step-by-step SCC UI flow — Add Device → Choose FTD → Cloud method → Registration token → FDM registration screen → device registers.

## Verify registration

- FDM shows a "Managed by SCC" banner
- SCC dashboard shows the 1210CE as **Online**
- Config sync status: **Up to date**

## Edge cases

### Deploy vs push

Once managed by SCC, the source of truth shifts:

- Config changes should be made in **SCC** and pushed down
- FDM becomes read-only for policy (device config still local — networking, health, etc.)

**Fill in:** the specific mode transitions, when to expect FDM changes to sync vs need explicit push.

### Common onboarding issues

- **Registration token expired** — regenerate in SCC
- **Firewall can't reach SCC** — check outbound HTTPS to `*.cdo.cisco.com` from mgmt interface
- **Smart Licensing not registered** — SCC requires valid entitlement

## Next

Head to [Managing via SCC](scc-managed.md) for the day-to-day workflow.
