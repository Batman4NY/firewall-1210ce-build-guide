# Troubleshooting

!!! warning "⚠ WIP — being backfilled with empirical gotchas from live capture sessions"
    Chapter outline plus specific gotchas encountered during 2026-07-10 (Ch 3-8 baseline) and 2026-07-11 (Ch 9-10 SCC/cdFMC) live-build sessions. Every entry cross-references the chapter and screenshot where the issue was first captured. Feedback and PRs welcome.

**Symptoms are grouped by build phase.** Jump to the phase that matches when the issue appeared: [Console](#console-shows-nothing-after-power-on) · [First boot / FDM reachability](#cannot-reach-fdm-after-first-boot) · [Deploy](#deploy-fails) · [Talos updates](#talos-updates-not-pulling) · [URL filtering](#url-filtering-not-blocking-test-urls) · [SCC/cdFMC onboarding](#sccdfmc-onboarding-fails) · [SCC/cdFMC after onboarding](#sccdfmc-after-onboarding) · [Duo](#duo-login-loop) · [Umbrella](#umbrella-tunnel-down) · [ThousandEyes](#thousandeyes-agent-not-registering) · [Nuclear reset (Ch 3.5)](remote-factory-reset.md).

Organized by symptom. Find your symptom, walk the checks, skip to next if not applicable.

## Console shows nothing after power on

Fastest triage is the FTDI + RJ45 primary path — it isolates cable-quality issues from platform issues in one step. Full details in [Ch 3 — Console access](console-access.md).

- FW power is on (front LED)
- **FTDI cable enumerated on ConsolePi**: `lsusb | grep FTDI` returns `0403:6001 FT232`
- **`/dev/ttyUSB0` exists** on ConsolePi (`ls /dev/ttyUSB*`) — primary FTDI + RJ45 path
- **ser2net serving port `:8000`** for the FTDI: `telnet <consolepi-ip> 8000`
- Nothing else holds `/dev/ttyUSB0` exclusive (no `picocom` process from a stale `consolepi-menu` session)
- If you're on the alternative USB-C path: cable is **data**, not charge-only; `/dev/ttyACM0` exists (`ls /dev/ttyACM*`); ser2net port is `:9000` — see the [ser2net port mapping table](console-access.md#ser2net-port-mapping)
- Note: the 1210CE routes console output to USB-C when both interfaces are plugged in. If FTDI is your primary but USB-C is also connected, unplug USB-C or explicitly pick `/dev/ttyUSB0` via `consolepi-menu`.

## Cannot reach FDM after first boot

**Fill in:** MGMT IP verification, subnet checks, browser HTTPS cert warnings, common firewall rules that block MGMT from LAN.

## Deploy fails

### Strong-crypto trap on eval Smart License

Captured on 2026-07-10 during Ch 7 policy authoring on the reference lab. FDM deploy fails when the FTD is running in Smart License **evaluation** mode with `exportControl:null` — even if `Threat`, `Malware`, and `URL License` entitlements are all enabled.

**Signature on the FDM side:** deploy job returns a policy-validation failure referencing strong encryption or export-controlled features.

**Root cause on the reference lab:** the FTD was registered to CSSM but the registration token did NOT have "Allow export-controlled functionality" checked, leaving `exportControl:null`. Effectively equivalent to eval-mode: strong-crypto features are unauthorized.

**Fixes** (any one resolves it):

- Re-register the FTD with a Smart Software Manager token that has "Allow export-controlled functionality" checked
- Author policy that avoids strong-crypto features (no S2S VPN with AES, no RA VPN with SSL, no strong TLS decryption ciphers)
- Move to cdFMC management ([Ch 9](scc-onboarding.md)) — the same trap surfaces at the cdFMC level per [Ch 10 Step 6](scc-managed.md#step-6-strong-crypto-where-the-gate-lives-once-cdfmc-owns-the-box), but the license state moves to cdFMC's virtual account, which may already be registered with export-controlled entitlement in your customer engagement

See [Ch 7 → Strong-crypto trap section](security-policies.md) for the full empirical narrative + Cisco doc source citations.

## Talos updates not pulling

- Outbound HTTPS to `updates.cisco.com` and `api.threatgrid.com` allowed?
- Smart Licensing state OK?
- FDM → Updates page — check last-attempted timestamp + error

## URL filtering not blocking test URLs

- Access control rule has URL filtering enabled?
- Deploy completed successfully?
- Test URL is in a blocked category (verify against Talos categorization)
- HTTPS traffic requires SSL decryption for full URL visibility

## SCC/cdFMC onboarding fails

The specific gotchas below were captured empirically during the reference-lab onboarding on 2026-07-11. See [Ch 9 — SCC onboarding](scc-onboarding.md) for the full walkthrough.

### `Registration timed out` in cdFMC Task Manager

Captured verbatim from cdFMC's Task Manager CSV export ([captures/ch9-ch10-task-manager-report-2026-07-11.csv](https://github.com/Batman4NY/firewall-1210ce-build-guide/blob/main/captures/ch9-ch10-task-manager-report-2026-07-11.csv), 2026-07-11):

```
Register  fw1210ce: Registration timed out. Please check connectivity and registration id
          Category: Register  Status: FAILURE  Duration: 2m 6s
```

**Root cause on the reference lab:** the first `configure manager add` attempt on the FTD was interrupted before the `yes/no` confirmation was answered. cdFMC's own 2-minute registration timer expired before the FTD ever initiated the outbound sftunnel.

**Fix:** the SCC-side registration key is NOT invalidated by this timeout. Re-fire the same `configure manager add` command on the FTD (with the `yes` confirmation this time), and cdFMC accepts the retry cleanly. See [Ch 9 Step 3a](scc-onboarding.md#3a-ftd-side-poll-show-managers) for the callout.

### FTD's `show managers` stuck at `In progress` past 15 minutes

The reference lab went from `configure manager add` to `Registration : Completed` in ~4 minutes. Cisco docs cite up to 15 min.

If `In progress` persists past ~15 minutes:

- Verify outbound HTTPS from the FTD mgmt IP to the cdFMC hostname:
  ```
  > ping system 1210ce-lab--ti0cqk.app.us.cdo.cisco.com count 3
  ```
- Confirm the registration key from Ch 9 Step 2d has not been regenerated on the SCC side (each regeneration invalidates the previous one)
- Check the cdFMC Task Manager (bell icon → Tasks tab) for a specific error string — Cisco's error strings there are more actionable than the FTD-side `In progress` state

### SCC "All" tab shows `Not Synced` for a Synced device

The `Configuration Status` column on the SCC Security Devices page's **All** tab can lag the truthful state by up to 10 minutes per [Cisco's documented propagation window](https://developer.cisco.com/docs/cisco-security-cloud-control-firewall-manager/cdfmc-managed-ftds-only-deploy-ftd-device-changes/). Reference lab observed 23 minutes of lag on 2026-07-11.

**Fix:** click the **FTD** tab (sibling to All) — the FTD-tab reads directly from cdFMC and shows the authoritative state. See [Ch 9 Step 3b-1](scc-onboarding.md#3b-1-the-all-tab-and-the-ftd-tab-disagree-and-why) for the full analysis with screenshots.

### "Health Warning" on cdFMC — `hmdaemon exited N time(s)`

Captured from Task Manager on 2026-07-11:

```
Health  1210ce-lab--ti0cqk  Process Status  hmdaemon exited 5 time(s).  WARNING
```

The device in this warning is **cdFMC's own management appliance** (`1210ce-lab--ti0cqk`), NOT `fw1210ce`. `hmdaemon` is the Health Monitor daemon running on cdFMC. Typically benign transient (auto-recovers). Only escalate to TAC if the count keeps climbing over days.

## SCC/cdFMC after onboarding

### `Deployment Notes: Deployment after re...` — what is Deploy_Job_1?

cdFMC auto-triggers an initial policy deploy as part of onboarding, without any user action. The reference lab captured `Deploy_Job_1` — `Deployed by: System` — completing in 3m 23s at 09:03-09:06 EDT after Registration Completed at 09:01. See [Ch 10's "cdFMC auto-deploys the initial policy assignment" section](scc-managed.md#cdfmc-auto-deploys-the-initial-policy-assignment-retroactively-surfaced-by-the-notifications-pane) for the empirical timeline.

**No traffic passes through the FTD after Deploy_Job_1** — the Default Access Control Policy pushed has `Default Action: Access Control: Block All Traffic` (see [Ch 10 → "CRITICAL: the deployed Default ACP blocks all through-traffic by default"](scc-managed.md#critical-the-deployed-default-acp-blocks-all-through-traffic-by-default)). Author at least one Allow rule and Deploy again before expecting traffic to flow.

### "Failed to load the out-of-band configuration differential" on 1210CE

Empirically observed on 2026-07-11 — click `Check Latest Status` next to `Out-of-band configuration status:` on cdFMC's Device Summary page:

```
Failed to load the out-of-band configuration differential
Out of band change detection for break-fix changes are not supported for the device.
```

**This is expected behavior on the 1210CE.** Cisco has not enabled out-of-band change detection for this hardware family in FTD 7.6.4-69. Consequence: cdFMC will NOT flag changes made via direct SSH to the FTD as drift. See [Ch 10 → "1210CE-specific gotcha: out-of-band change detection is NOT supported"](scc-managed.md#1210ce-specific-gotcha-out-of-band-change-detection-is-not-supported) for the mitigation pattern.

### `HTTP 400` on `/firewall/v1/*` endpoints from a Base-tier tenant API token

An API token minted from SCC's `Platform Management → API Keys` page has product scope `Security Cloud Control` only (empirically the only option in that Product/Service dropdown on Base tier as of 2026-07-11). Calls to `https://api.<region>.security.cisco.com/firewall/v1/*` return HTTP 400 (not 401) with a gateway rewrite path in the response body:

```
{"path":"/api/platform/scc-gateway/request/api/rest/v1/token","status":400,"error":"Bad Request"}
```

**This is a scope mismatch, not a URL error or a token expiration.** The SCC gateway recognizes the token but refuses to route to the Firewall Manager backend. See [Ch 10 → Step 1](scc-managed.md#step-1-mint-the-api-token) for the exhaustive UI-path check that yielded no working alternative on Base tier. Reference-lab TAC case draft at [captures/ch10-tac-case-draft-2026-07-11.md](https://github.com/Batman4NY/firewall-1210ce-build-guide/blob/main/captures/ch10-tac-case-draft-2026-07-11.md) — outcome will resolve whether Base tier ever unlocks Firewall Manager API or whether that requires a paid tier upgrade.

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

Nuclear option: **[Ch 3.5 — Remote factory reset](remote-factory-reset.md)**. That runs the Cisco-supported FXOS cross-version reimage and returns the FW to first-boot state entirely over the console + management network.

Save your smart license token first — you'll need to re-register.

!!! warning "Stale commands in earlier drafts of this guide"
    Older drafts and Cisco Community posts reference `> configure factory-default` at the FTD `>` prompt, or `erase configuration` in FXOS, or `install security-pack version <same-version> force` — **none of these actually reset a 1200-series box on FTD 7.6.x**. Enumerated live on the hardware. Use [Ch 3.5](remote-factory-reset.md), which uses `install security-pack version <different-version>` — the only Cisco-supported CLI reset that actually reimages.

!!! warning "Stale command in earlier drafts of this guide"
    Older versions of this chapter (and older Cisco community posts) point at `> configure factory-default` at the FTD `>` prompt. That command **does not exist on FTD 7.6.0-113 for the 1200 series** — enumerated live on the box, no match. The build's factory-reset surface is FXOS, not FTD. Use [Ch 3.5](remote-factory-reset.md).

## Still stuck?

- [Cisco Secure Firewall 1200 Series Getting Started Guide](https://www.cisco.com/c/en/us/support/security/secure-firewall-1200-series/products-installation-and-configuration-guides-list.html) is the authoritative reference
- [Cisco TAC](https://www.cisco.com/c/en/us/support/index.html) if you have a support contract
- Open an [issue](https://github.com/Batman4NY/firewall-1210ce-build-guide/issues) here if you found something wrong or unclear in this guide
