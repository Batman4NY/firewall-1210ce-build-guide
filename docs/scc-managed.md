# Managing via cdFMC — day-to-day workflow

!!! warning "🚧 Placeholder — API surface enumerated, walkthrough queued"
    Ch 10 shifts from "Managing via SCC" (the legacy hybrid framing) to **Managing via cdFMC** (the current SCC-owned FTD story on Base tier). The FMC REST API is the load-bearing interface — different endpoint surface, different auth model, different deploy pattern than FDM's API.

**Prerequisites:** [Ch 9 — SCC onboarding](scc-onboarding.md) — device Online in cdFMC.

Day-to-day workflow once the FTD is cdFMC-managed. This chapter is captured against the same cdFMC instance you provisioned in Ch 9, so the endpoints + IDs match your box.

## Auth model shift — FMC REST API, not FDM

The management API surface changes fundamentally after cdFMC onboard:

| Property | FDM (pre-onboard) | cdFMC (post-onboard) |
|---|---|---|
| Base URL | `https://<mgmt-ip>/api/fdm/latest/` | `https://<region>.security.cisco.com/fmc/api/fmc_config/v1/` (or the tenant's cdFMC-specific hostname) |
| Auth | Basic `POST /fdm/token` with admin credentials | Token exchange via SecureX / Security Cloud OAuth |
| Auth-token TTL | 30 min | 30 min (matches FDM), but refresh flow is different |
| Deploy model | `POST /operational/deploy` on the FTD | `POST /domain/{uuid}/deployment/deploymentrequests` on cdFMC — deploys to N devices from one call |
| Object scope | Per-device | Domain-scoped (default domain), shared across all devices in the cdFMC instance |

The cdFMC REST API is a superset of the on-prem FMC REST API — same endpoints, same schemas, just running in the cloud.

## Step 1 — Auth to cdFMC

!!! warning "🚧 Placeholder"
    ```bash
    CDFMC_HOST=https://<region>.security.cisco.com    # from your tenant post-provisioning
    # Auth via SecureX OAuth — captured in next iteration
    ```

## Step 2 — Confirm device shows in cdFMC domain

!!! warning "🚧 Placeholder"
    `GET /fmc/api/fmc_config/v1/domain/{domainId}/devices/devicerecords` returns the enrolled 1210CE with its serial and management IP.

## Step 3 — Round-trip: change a device label, deploy, verify

!!! warning "🚧 Placeholder"
    A no-op-ish change (edit device description) then `POST /deployment/deploymentrequests` to prove the pipe works both ways.

## Step 4 — Author a real access rule via cdFMC

!!! warning "🚧 Placeholder — this is where the strong-crypto trap becomes relevant"
    On our reference lab box, the smart-license eval state blocks strong-crypto policy deploy. cdFMC-side rule authoring will succeed (rules are stored in the cloud) but the deploy TO the device will hit the same `strong-encryption-disable` we saw in Ch 7. Register a Smart Account first (Ch 14) if you need to actually run rules through this managed FTD.

## Step 5 — Object management shift

!!! warning "🚧 Placeholder"
    Network objects, port objects, application objects, URL objects — all live in cdFMC and are re-used across every device managed by the same cdFMC instance. Contrast with FDM where each object is device-local.

## Step 6 — Multi-device workflow

!!! warning "🚧 Placeholder"
    Add a second FTD (simulator or physical) to the same cdFMC. Share a policy. Deploy to both from one call. This is where cdFMC (and any FMC) starts earning its licensing cost.

## Step 7 — Backup + config export

!!! warning "🚧 Placeholder"
    - cdFMC → System → Tools → Backup / Restore (per Cisco docs)
    - Scheduled snapshots
    - Export path: full config as compressed archive, restorable to a fresh cdFMC instance

## Step 8 — Event correlation + Talos content updates from cdFMC

!!! warning "🚧 Placeholder"
    - cdFMC pulls Talos content on its own schedule for all managed devices (VDB, SRU, Geolocation)
    - Per-device SRU / VDB update triggering from cdFMC (contrast with Ch 8's per-device FDM API)
    - Event stream cross-device correlation

## What breaks in FDM after cdFMC onboard

Reference table for what happens to the FDM API surface on the FTD after enrollment:

!!! warning "🚧 Placeholder — captured empirically during Ch 9 execution"
    - Read endpoints: mostly still work (systeminfo, interfaces, some object reads)
    - Write endpoints: return HTTP 4xx with a "managed by cdFMC" error
    - Deploy: rejected with "managed externally"
    - The **Managed by cdFMC** banner appears in FDM UI

## Rollback — un-enrolling from cdFMC

!!! warning "🚧 Placeholder"
    `POST /api/fdm/latest/action/cloudservices/unenroll` on the FDM side, plus removing the device from cdFMC's inventory. Result: FTD returns to FDM-standalone mode, but policies authored in cdFMC do NOT come back — FDM restarts at its pre-onboard config baseline.

## Next

Once cdFMC operations are steady-state, integration chapters unlock:

- [Ch 11 — Duo MFA on cdFMC](duo-integration.md) — lock down cdFMC admin access
- [Ch 12 — Umbrella SASE tunnel from cdFMC](umbrella-integration.md) — cloud-to-cloud SASE
- [Ch 13 — ThousandEyes agent on the FTD](thousandeyes-integration.md) — deploy TE from cdFMC
