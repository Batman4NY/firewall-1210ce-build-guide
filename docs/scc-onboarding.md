# Onboarding to Security Cloud Control — cdFMC path

!!! success "🚧 In progress · 2026-07-11 — cdFMC prerequisites captured live · device-onboarding steps queued"
    Prerequisite discovery + Enable-cdFMC flow captured against a fresh SCC tenant. Device-add wizard + FDM enrollment + sftunnel verify steps queued for the current live-capture session.

**Prerequisites:**

- [Ch 4](first-boot.md) verified — FDM reachable at `https://<mgmt-ip>/`
- [Ch 6](fdm-baseline.md) baseline complete — hostname, DNS, static mgmt IP set
- Cisco.com account with SCC entitlement (see Step 0 below)
- Outbound HTTPS from mgmt to `*.cdo.cisco.com` and `*.security.cisco.com` — verify with `curl -sI https://edge.us.cdo.cisco.com/` from a host on the same segment (a `403` or `200` HTTP response = reachable; connection refused / timeout = firewall problem)

Bring the 1210CE under **Cisco Security Cloud Control (SCC)** management via the **Cloud-delivered Firewall Management Center (cdFMC)** path. SCC is the unified cloud pane-of-glass for Cisco's security portfolio; cdFMC is SCC's FTD management surface.

## Tier prerequisite — this chapter's biggest gotcha

**Verified 2026-07-11 on a fresh SCC tenant with Firewall Management _Base_ subscription:**

- The modern SCC "Add Device → Firewall Threat Defense" flow surfaces **cdFMC-only** onboarding
- FDM-hybrid mode (Layer 2 in [Ch 5's deployment matrix](choose-mgmt-path.md)) is **NOT offered at Base tier**
- FDM-hybrid requires **Firewall Management _Premier_ tier** OR the legacy pre-Security-Cloud-Control CDO console

Decision tree for what to do based on your customer's tenant entitlement:

| Customer tenant tier | Available onboarding paths | Chapter that applies |
|---|---|---|
| **Firewall Management Base** | cdFMC only | This chapter (Ch 9) |
| **Firewall Management Premier** | cdFMC + FDM-hybrid | This chapter for cdFMC; a Premier-tier-hybrid variant is a future edit |
| **Legacy CDO console** (pre-SCC unification, still exists on some accounts) | FDM-hybrid (traditional CDO onboard) | A legacy-CDO variant is a future edit |
| **No tenant / no entitlement** | Register account at `sign-on.security.cisco.com`, then Base tier is free with 25-device cap for lab use | Start here — Step 0 below |

## Onboarding-mode consequences you have to accept before proceeding

The cdFMC onboarding path is destructive to FDM-authored policy. Cisco's own SCC UI warns:

> *"the firewall device manager will no longer manage the device, and all existing policy configurations on the device will be lost except for the basic interface configurations. Therefore, you must configure the policies from the corresponding manager."*

Concrete impact:

| Config category | Survives cdFMC onboard? |
|---|---|
| Interface / management IP / hostname (Ch 6) | ✅ ("basic interface configurations") |
| DNS / NTP / static routes (Ch 6) | ✅ (basic) |
| Access rules / IPS / URL filter (Ch 7) | ❌ — re-authored in cdFMC |
| Content DBs (VDB / SRU / Geo from Ch 8) | ✅ (on-device, not in FDM policy) |
| Smart License registration state | ✅ (kept) |

**If your FDM already has a policy set you can't afford to lose**, back it up first — [Ch 6 Backup and restore](fdm-baseline.md#backup) or via `POST /action/configexport` — before firing this chapter.

## Step 0 — Verify SCC tenant + Firewall Management entitlement

### 0a. Sign in to Security Cloud Sign-On

Browser to `https://sign-on.security.cisco.com` and log in with the Cisco.com account that owns the tenant.

**Verify:** the landing page shows a **"Security Cloud Control"** tile under **Your applications**. If not, your account has no SCC entitlement — provision it (or a free trial) before continuing.

### 0b. Enter or create the target organization

Click **Launch** on the Security Cloud Control tile. SSO redirect prompts to pick an organization (if you have several) or offers **Create new organization**.

**For lab or first-time customer onboards**, create a dedicated organization named for the box or engagement (e.g. `fw1210ce-lab`, `customer-name-poc`). Reasons:

- Devices land in a clean inventory instead of mixing into demo-org context
- Post-engagement cleanup is a single tenant delete instead of hunting for specific devices
- Regional edge endpoint is decided at org creation — pin to the customer's actual region

### 0c. Claim / enable Firewall Management

The new organization defaults to no active subscriptions. Path: **Administration → Subscriptions**.

Look for an in-banner "**Your organization is entitled to use Firewall Management. Enable Firewall Management now.**" notification. Click **Enable**. Cisco auto-detects the account-level entitlement and provisions the tenant-level Firewall Management binding — takes ~5 seconds. Result screen shows:

```
Product:   Cisco Security Cloud Control Firewall Management Base
Region:    North America     (or your tenant's region)
Quantity:  1 instance
Status:    Active
```

Record the **subscription ID** (long UUID). It's useful for audit / support tickets against the tenant.

**Do NOT enable Multicloud Defense** even though its notification appears — that's a separate product for cloud-native workloads (formerly Valtix), unrelated to the 1210CE hardware FTD.

## Step 1 — Enable cdFMC in the tenant

Navigate to **Firewall** section of the SCC left nav → **Security Devices**. First-time state:

```
Displaying 0 of 0 results
No devices or services found. You must onboard a device or service to get started.
```

Click the **`+`** button (top-right corner of the devices table). SCC opens the **Onboard FTD Device** wizard.

The wizard's first panel is exclusively titled **"Enable cdFMC"** for Base-tier tenants — click it. cdFMC provisioning begins:

- **Duration:** typically 15-30 min for Base tier (Cisco doesn't commit a hard SLA)
- **Behavior during provisioning:** you can navigate away; cdFMC access unlocks when ready
- **Signal it's done:** notification bell + email; **Firewall Management** entry appears in the app switcher

!!! info "Field TBD — captured on next iteration"
    The exact wall-clock provisioning time and any region-selection prompts are being captured in this reference session. Empirical timing will replace this callout in the next commit.

## Step 2 — Add Device wizard (in cdFMC)

!!! warning "🚧 Placeholder — capture in progress"
    After cdFMC provisions, the Add Device wizard opens. Steps captured empirically on the live tenant, filled in the next commit:

    1. Device category = **Firewall Threat Defense**
    2. Deployment = **Standalone**
    3. Registration method = **Registration Key** (recommended)
    4. Device name, description, software version fields
    5. **Token + endpoint + NAT ID** generated by SCC — the load-bearing values

## Step 3 — FDM-side enrollment

!!! warning "🚧 Placeholder"
    FDM API sequence to enroll to the cdFMC endpoint using the token from Step 2:

    ```
    POST /api/fdm/latest/action/cloudservices/enroll
    body: registration key + endpoint + NAT ID
    ```

    Or via FDM UI: **Device → System Settings → Cloud Services → Enroll**.

## Step 4 — Verify sftunnel established

!!! warning "🚧 Placeholder"
    - SCC inventory shows the device as **Online**
    - FDM shows a **"Managed by cdFMC"** banner
    - `GET /api/fdm/latest/operational/cloudservicesinfo/...` returns `connected: true`
    - cdFMC device list shows the FTD's serial + IP

## Step 5 — Round-trip test

!!! warning "🚧 Placeholder"
    A no-op deploy from cdFMC (change a device label, deploy) to prove the pipe works both ways.

## Common onboarding gotchas — reference

- **Token expires** — SCC-side tokens have a ~1 hour countdown. If the FDM enroll doesn't fire in that window, regenerate. Ch 9's flow queues the FDM POST first so we're pasting-and-firing back-to-back.
- **FW can't reach `edge.<region>.cdo.cisco.com`** — check that mgmt has outbound HTTPS. Talos content updates from Ch 8 working = outbound path healthy.
- **Smart License in bad state** — SCC accepts EVAL entitlement for onboard. If Smart Licensing shows `Not Compliant`, sort that first before onboard — cdFMC won't let you deploy policy against a non-compliant license.
- **NAT / IP mismatch** — if the FTD is behind NAT (like our lab where mgmt has ISP behind UDM Pro), the NAT ID field on the SCC token matters. Cisco docs at [DevNet](https://developer.cisco.com/docs/cdo/) have the specifics.

## Next

Once the sftunnel is up and Step 5 round-trip passes, head to [Ch 10 — Managing via cdFMC](scc-managed.md) for the day-to-day workflow.
