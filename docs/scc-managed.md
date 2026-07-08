# Managing via SCC

!!! warning "⚠ WIP"
    Placeholder chapter with outline only. Content to be written as the lab is built. Feedback and PRs welcome.

**Prerequisites:** [Chapter 8: SCC onboarding](scc-onboarding.md) — device Online in SCC.

Day-to-day workflow once the FW is SCC-managed.

## Do the thing

### Policy management shift

- All access control, IPS, URL, file policies now edited in SCC
- Object management (network objects, service objects) unified across all managed devices
- Push changes to selected devices from SCC

**Fill in:** SCC UI walk-through, screenshots, workflow examples.

### Event correlation

SCC's event stream shows:

- IPS events (from Talos rules)
- Connection events
- Malware detections
- URL blocks

Cross-device correlation is where SCC earns its keep at scale.

**Fill in:** event filtering, drill-down, saved views.

### Backup + config export

- Config backup: manual from SCC UI + scheduled backups if the tenant supports it
- Export: full device config as XML/JSON

**Fill in:** backup schedule setup, restore workflow, disaster-recovery pattern.

## Edge cases and multi-device

### Multi-device workflow

- Add a second FTD to SCC (Cisco simulator, ASAv, or another physical unit)
- Share policies across both — one place to change, pushed to both
- This is the "one lab, two devices" story

**Fill in:** shared object usage, policy inheritance, deployment orchestration.

### XDR integration

Once XDR is enabled (SCC → XDR tab), correlation across FW + endpoint + email + cloud starts appearing.

**Fill in:** XDR onboarding steps, incident-based views.

## Next

Head to [Duo MFA on FDM/SCC](duo-integration.md) to lock down admin access.
