# FTD upgrade via FDM API — major + hotfix

!!! success "✅ Complete · Captured 2026-07-10 live against the reference 1210CE"
    Two upgrade flows validated on the same box: **major** (7.6.0-113 → 7.6.4-69, 21m 33s) and **CC hotfix** (7.6.4-69 + CC-7.6.4.1-1, 11m 34s). Every `curl` transcribed verbatim from the reference build.

## Upgrade vs. reset — different operations

- **[Ch 3.5 Remote factory reset](remote-factory-reset.md)** wipes user config, regenerates UUID, requires cross-version FXOS `install security-pack`. Destructive on purpose — used to "return to zero."
- **This chapter** applies a new FTD software version **in place**, config preserved, via FDM's REST API. UUID stays the same. Same-major-version patches and CC compliance hotfixes are always upgrades, never resets.

Both paths coexist in the guide because Cisco's own docs split them (Reimage Guide vs Upgrade Guide).

## Prerequisites

- Ch 4 verification complete — you're logged in to FDM.
- FDM initial-setup wizard skipped (Ch 6 Step 0) — the REST API is unlocked.
- The upgrade package downloaded from Cisco.com to your workstation. Portal path (from Ch 3.5 Step 1):

    `software.cisco.com` → **Downloads Home** → **Security** → **Firewalls** → **Next-Generation Firewalls (NGFW)** → **Secure Firewall 1200 Series** → **Secure Firewall 1210CE** → **Firepower Threat Defense (FTD) Software** → target release

    - `Cisco_Secure_FW_TD_1200-<ver>-<build>.sh.REL.tar` — the major/point-release package (~970 MB)
    - `Cisco_Secure_FW_TD_1200_Hotfix_CC-<ver>.<hotfix>-<build>.sh.REL.tar` — CC certification hotfix (~220 MB)

## Auth

```bash
FDM_HOST=https://192.168.40.10
FDM_PW=$(bao kv get -field=password infra/webui/fw1210ce-admin)
TOKEN=$(curl -sk -X POST "$FDM_HOST/api/fdm/latest/fdm/token" \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\":\"password\",\"username\":\"admin\",\"password\":\"$FDM_PW\"}" \
  | jq -r '.access_token')
```

## Path A — Major upgrade (`7.6.0-113 → 7.6.4-69`)

### 1. Upload the package

```bash
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -F "fileToUpload=@Cisco_Secure_FW_TD_1200-7.6.4-69.sh.REL.tar" \
  "$FDM_HOST/api/fdm/latest/action/uploadupgrade"
```

Response confirms the file landed at `/var/sf/updates/` on the FTD:

```json
{
  "name":     "/var/sf/updates/Cisco_Secure_FW_TD_1200-7.6.4-69.sh.REL.tar",
  "fileName": "/var/sf/updates/Cisco_Secure_FW_TD_1200-7.6.4-69.sh.REL.tar",
  "id":       "Cisco_Secure_FW_TD_1200-7.6.4-69.sh.REL.tar",
  "type":     "fileuploadstatus"
}
```

**Empirical: 969 MB uploaded in 69 seconds** over the mgmt LAN.

### 2. Verify the package registered

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/managedentity/upgradefiles"
```

Returns metadata parsed from the package's manifest:

```json
{
  "items": [{
    "upgradeFileName": "Cisco_Secure_FW_TD_1200-7.6.4-69.sh.REL.tar",
    "updateVersion":   "7.6.4-69",
    "upgradeFrom":     "7.6.0",
    "rebootRequired":  true,
    "id":              "ebc20059-7c9f-11f1-990f-79fc8521a356",
    "type":            "upgradefile"
  }]
}
```

Save the `id` — it's the target for the readiness check + upgrade.

### 3. Run the readiness check

```bash
UPGRADE_FILE_ID="ebc20059-7c9f-11f1-990f-79fc8521a356"
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/action/upgradereadinesscheck" \
  -d "{
    \"upgradeFile\": {\"id\":\"$UPGRADE_FILE_ID\",\"type\":\"upgradefile\"},
    \"type\":\"upgradereadinesscheck\"
  }"
```

Poll status:

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/operational/upgradereadinesscheckstatus"
```

Expect `state: "SUCCESS"` when done. Empirical: **32 scripts run, 0 failed, 83 seconds**.

### 4. Fire the upgrade

Any pending config changes MUST be resolved first. Deploy them or discard them, then:

```bash
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/action/upgrade" \
  -d "{
    \"upgradeFile\": {\"id\":\"$UPGRADE_FILE_ID\",\"type\":\"upgradefile\"},
    \"type\":\"upgradeimmediate\"
  }"
```

If pending changes exist, you'll get:

```
"You must deploy all uncommited changes before starting a system upgrade."
```

Discard pending: `DELETE /operational/pendingchanges`. Or deploy: `POST /operational/deploy -d '{}'`. Then re-fire the upgrade.

### 5. Poll upgrade status

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/operational/upgradestatus"
```

Sample response mid-upgrade:

```json
{
  "state":              "IN_PROGRESS",
  "updateType":         "major",
  "baseVersion":        "7.6.0-113",
  "targetVersion":      "7.6.4-69",
  "progressPercent":    15,
  "message":            "Preparing to upgrade... (200_pre/500_stop_system.sh)",
  "timeRemainingInSeconds": 2100
}
```

## What to expect during the upgrade

**Empirical timings from the reference build (7.6.0-113 → 7.6.4-69):**

| Milestone | Wall-clock | Notes |
|---|---|---|
| Upload package (969 MB) | 69s | Multipart POST from workstation over mgmt LAN |
| Readiness check | 83s | 32 scripts; verifies platform support, DB schema, HA state |
| `POST /action/upgrade` accepted | 0s | Instant — moves upgrade to IN_PROGRESS |
| First 15% progress | ~3m 20s | Pre-upgrade scripts run, FTD apps stop |
| Deep reboot (FDM 503 → connection refused) | ~5-8m | Kernel boot, filesystem checks, image swap |
| FDM webserver returns (503 → app initializing) | ~15m from start | HTTP responds again |
| FDM API 200 (upgrade completed) | **21m 33s** | `state: COMPLETED` |

Cisco estimated 41 min in the readiness output; actual was half of that. The estimate is a worst-case ceiling.

## Verification — did the upgrade actually apply?

Three checks, in order of strictness:

```bash
# 1. Software version (must equal target)
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/operational/systeminfo/default" | jq '.softwareVersion'
# → "7.6.4-69"

# 2. UUID (must be UNCHANGED from before upgrade — proves it was upgrade, not reset)
ssh admin@192.168.40.10 'show version' | grep UUID
# → UUID: 9179b5e8-7c96-11f1-be1f-f5c66e7fad63   (same as pre-upgrade)

# 3. Config preserved
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/devicesettings/default/devicehostnames" | jq '.items[0].hostname'
# → "fw1210ce"   (whatever you set in Ch 6)
```

**The UUID unchanged is the definitive test.** A reset regenerates it; an upgrade preserves it. Chapter 4 lists this as one of its three "reset actually ran" checks — same test in reverse.

## Path B — CC hotfix (`7.6.4-69 → 7.6.4.1-1`)

The Common Criteria hotfix bumps FXOS to a CC-certified build. It's uploaded, checked, and fired via the **same API endpoints as the major upgrade** — the platform decides "major" vs "hotfix" based on the package manifest.

Same three commands as Path A, just pointing at the hotfix file:

```bash
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -F "fileToUpload=@Cisco_Secure_FW_TD_1200_Hotfix_CC-7.6.4.1-1.sh.REL.tar" \
  "$FDM_HOST/api/fdm/latest/action/uploadupgrade"

curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/action/upgradereadinesscheck" \
  -d '{"upgradeFile":{"id":"<hotfix-id>","type":"upgradefile"},"type":"upgradereadinesscheck"}'

curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/action/upgrade" \
  -d '{"upgradeFile":{"id":"<hotfix-id>","type":"upgradefile"},"type":"upgradeimmediate"}'
```

## Hotfix vs major upgrade — what differs

Both are validated end-to-end on the same box. The distinction matters for SEs describing the change to customers:

| Property | Major upgrade | CC hotfix |
|---|---|---|
| Package size | ~970 MB | ~220 MB |
| Upload time on reference LAN | 69s | 18s |
| `updateType` in `/operational/upgradestatus` | `major` | `hotfix` |
| Full-package `softwareVersion` bump? | ✅ `7.6.0-113 → 7.6.4-69` | ❌ **stays** `7.6.4-69` |
| `sspOsVersion` (FXOS platform) bump? | ✅ `2.16(0.128) → 2.16(1.147)` | ✅ `2.16(1.147) → 2.16(1.1400)` |
| UUID preserved? | ✅ | ✅ |
| Config preserved? | ✅ | ✅ |
| Wall-clock end-to-end | 21m 33s | 11m 34s |
| Dashboard "Software" field shows the change? | ✅ | ❌ (dashboard reads softwareVersion) |
| Proof it applied | `systeminfo.softwareVersion` matches target | `upgradestatus.targetVersion` shows `7.6.4.1-1` |

!!! warning "Hotfix silent-success trap"
    A CC hotfix leaves `softwareVersion` at the base value — an SE glancing at the FDM Dashboard after a hotfix cannot tell it applied. The definitive proof lives in `GET /operational/upgradestatus` (retained after completion) — `targetVersion: 7.6.4.1-1` + `updateType: hotfix` + `state: COMPLETED`. Also check `sspOsVersion` in systeminfo — the FXOS platform build changes even when the FTD version doesn't.

## Roll back within 10 days

If the upgrade broke something, `POST /action/revertupgrade` rolls back to the previous version within the 10-day rollback window:

```bash
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/action/revertupgrade" -d '{}'
```

Rollback duration is similar to the upgrade itself (~20 min for a major).

## Change-record hygiene

For regulated fleets, treat every upgrade as an audit event:

- Attach this chapter to the change record
- Include the `sha256sum` of the upload package (from Cisco's download page)
- Include the pre-upgrade `systeminfo.softwareVersion` + `sspOsVersion` and the post-upgrade values as before/after evidence
- Include the `POST /operational/deploy` post-upgrade result confirming the config-baseline redeploy succeeded (see Ch 6 Step 8's poll pattern)

## Related

- [Ch 3.5 Remote factory reset](remote-factory-reset.md) — the destructive alternative when you need "return to zero"
- [Ch 6 FDM baseline](fdm-baseline.md) — deploy pattern reused here
- [Ch 14 Licensing](licensing.md) — upgrade behavior + Smart License registration relationship

## Next

If the upgrade landed, proceed to [Ch 5 — Choose your management path](choose-mgmt-path.md) or resume whatever chapter you were on.
