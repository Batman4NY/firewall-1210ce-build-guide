# Talos intel + updates — VDB, Snort rules, geolocation, SI feeds

!!! success "✅ Complete · Captured 2026-07-10 live against the reference 1210CE"
    Manual and scheduled update flows for the four Talos-fed content databases: VDB, SRU (Snort Rule Updates), Geolocation, and Security Intelligence Feeds. Every `curl` transcribed verbatim from the reference build.

Cisco Talos is the threat-intel back-end for FTD's IPS, URL filtering, malware detection, and geolocation. This chapter is about pulling that intel down and keeping it fresh via the FDM REST API.

## What Talos feeds into the FW

- **VDB** — Vulnerability Database. The AppID + navlgo signatures used by app-identification (name-your-web-app filtering) + IPS reference lookup.
- **SRU** — Snort Rule Updates. Actual IPS rules Talos ships as CVEs are triaged and new attack patterns identified.
- **Geolocation database** — IP-to-country mappings for geo-based rules.
- **Security Intelligence feeds** — bad-IP, bad-URL, DGA-domain lists that FTD blocks in-line before any policy match.
- **URL categorization + reputation** — the Talos category taxonomy powers Ch 7's URL filtering.

Talos publishes updates on rolling cadences (VDB monthly-ish, SRU daily, Geo weekly, SI feeds continuous). FTD pulls on schedule OR on-demand.

## Prerequisites

- Ch 6 baseline complete — mgmt IP + DNS configured
- Outbound HTTPS from mgmt reachable to `updates.cisco.com` — verify with `curl -sI https://updates.cisco.com/` from the box
- Smart Licensing in a valid state (any eval or registered mode) — updates require entitlement

## Auth

```bash
FDM_HOST=https://192.168.40.10
TOKEN=$(curl -sk -X POST "$FDM_HOST/api/fdm/latest/fdm/token" \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\":\"password\",\"username\":\"admin\",\"password\":\"$(bao kv get -field=password infra/webui/fw1210ce-admin)\"}" \
  | jq -r '.access_token')
```

## Baseline — check what's currently installed

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/operational/systeminfo/default" | jq '.vdbVersion'
```

Reference build, factory-fresh state right after Ch 3.5 reset:

```json
{
  "vdbCurrentVersion":  "392",
  "vdbCurrentBuild":    "0",
  "vdbReleaseDate":     "2024-08-02 19:06:46",     ← from factory image
  "vdbPackageName":     "Cisco_Base_VDB-4.5.0-392.sh.REL.tar",
  "appIDRevision":      "133",
  "navlRevision":       "153",
  "lastSuccessVDBDate": "2026-07-10 19:46:45Z"
}
```

The `vdbCurrentVersion` is the pre-update baseline. A fresh reset always lands you on the factory-shipped VDB, which lags Talos-current by however long the box sat in inventory.

## Manual update — trigger all four content types

All four update actions follow the same pattern: `POST /action/update<type>` with a minimal payload, returns a job ID immediately, poll `/jobs/<type>updates` for completion.

### VDB

```bash
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/action/updatevdb" \
  -d '{"type":"vdbupdateimmediate"}'
```

Response:

```json
{
  "id":           "21bffa7f-7caa-11f1-901f-7fddd6a3dad9",
  "scheduleType": "IMMEDIATE",
  "user":         "admin",
  "type":         "updatevdbimmediate"
}
```

Poll:

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/jobs/vdbupdates?limit=1" | jq '.items[0]'
```

Status progression:

```
QUEUED       (immediate, <1s)
IN_PROGRESS  Task execution started
IN_PROGRESS  Adding VDB updates to database
SUCCESS      (durations vary; reference: 4m 43s)
```

### SRU (Snort Rule Update)

```bash
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/action/updatesru" \
  -d '{"type":"sruupdateimmediate"}'
```

Poll `/jobs/sruupdates`. Reference: **~52 seconds**.

### Geolocation database

```bash
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/action/updategeolocation" \
  -d '{"type":"geolocationupdateimmediate"}'
```

Poll `/jobs/geolocationupdates`. Reference: **~22 seconds**.

### Serial execution behavior

**Important**: FTD runs these one at a time even if you POST them concurrently. In the reference session, VDB fired first + started, SRU and Geo queued behind it. Timing observed:

```
17:55:31  POST VDB, SRU, Geo (near-simultaneous)
17:55:31  VDB    IN_PROGRESS
17:55:31  SRU    QUEUED
17:55:31  Geo    QUEUED
18:00:14  VDB    SUCCESS   (4m 43s)
18:00:26  SRU    IN_PROGRESS
18:01:06  SRU    SUCCESS   (~40s)
18:01:14  Geo    IN_PROGRESS
18:01:28  Geo    SUCCESS   (~14s)
```

Plan for this if you're scripting a "hit all updates" cycle — total wall-clock is the sum, not the max.

## Verify — did the updates land?

Same `GET /operational/systeminfo/default` from baseline, but the values change:

```json
// After the manual VDB update above
{
  "vdbCurrentVersion":  "433",                    ← 392 → 433
  "vdbReleaseDate":     "2026-07-08 14:51:14",   ← current, not factory
  "lastSuccessVDBDate": "2026-07-10 21:55:58Z",  ← this session's timestamp
  "appIDRevision":      "433"
}
```

On the reference box, VDB jumped 392 → 433 in a single update. Any box that's been reset from factory (or shipped and stored for a while) benefits from a first-thing-post-Ch-6 refresh cycle.

For SRU, check the intrusion policy state via `GET /policy/intrusionpolicies/{id}` — the underlying rule count changes are visible in the response payload but there's no single "SRU version" field the way VDB has.

For Geolocation, check `GET /object/geolocations` — count of country objects should be at Talos-current.

## Scheduled updates

FDM supports recurring updates via `/managedentity/vdbupdateschedules`, `/managedentity/geolocationupdateschedules`, and similar endpoints for the other content types.

### API surface

```
POST /managedentity/vdbupdateschedules            create schedule
GET  /managedentity/vdbupdateschedules            list schedules
PUT  /managedentity/vdbupdateschedules/{id}       modify
DELETE /managedentity/vdbupdateschedules/{id}     remove
```

Similar for geolocation. SRU schedule creation uses `POST /action/updatesru` with `scheduleType: RECURRING`.

### Schedule creation via API — a real gotcha

Attempts on the reference build with the obvious payloads (using cron-style `runTimes`, using `days`/`startTime`/`timeZone`) both returned:

```
"Unable to schedule job VDBUpdateSchedule. Invalid start trigger."
```

The `runTimes` field is documented in the schema as a plain `string` with no format hint. The FDM UI creates schedules successfully, so the payload format exists — but its exact shape isn't obvious from `/apispec/ngfw.json`.

**Practical guidance**:

1. Create the schedule via the FDM UI (**Device → Updates → Configure**) — that's the reliable path.
2. Then `GET /managedentity/vdbupdateschedules` and inspect the resulting object's `runTimes` value — that's the format your future scripts should replicate.
3. Once you have a known-good payload, `POST` to create new schedules or `PUT` to modify.

Cisco's DevNet has open issues around FDM schedule-creation API friendliness that are worth tracking.

## Security Intelligence feeds

SI feeds behave differently — they update **continuously** from Cisco's cloud rather than on a pull-schedule. The API endpoint is `/action/securityintelligenceupdatefeeds` but requires SI to be enabled on an active policy first (`GET /policy/securityintelligencepolicies`).

For a factory-fresh box with no SI policy configured, POSTing to the update-feeds endpoint returns a syntax error — the endpoint expects an SI policy context that doesn't yet exist. Set up SI in Ch 7 (URL/DNS/Network SI policies) before invoking the feed-update endpoint.

## Full-cycle "make everything fresh" script

For a new box or one that's been sitting on the shelf:

```bash
#!/bin/bash
FDM_HOST=https://192.168.40.10
TOKEN=$(curl -sk -X POST "$FDM_HOST/api/fdm/latest/fdm/token" \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\":\"password\",\"username\":\"admin\",\"password\":\"$FDM_PW\"}" \
  | jq -r '.access_token')

for UPDATE in vdb sru geolocation; do
  echo ">> firing $UPDATE update"
  curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    "$FDM_HOST/api/fdm/latest/action/update$UPDATE" \
    -d "{\"type\":\"${UPDATE}updateimmediate\"}" > /dev/null
done

# Poll until all three complete
while true; do
  STATUSES=$(for J in vdbupdates sruupdates geolocationupdates; do
    curl -sk -H "Authorization: Bearer $TOKEN" \
      "$FDM_HOST/api/fdm/latest/jobs/$J?limit=1" | jq -r '.items[0].status'
  done)
  echo "$STATUSES" | grep -qE 'IN_PROGRESS|QUEUED' && sleep 30 && continue
  echo "$STATUSES" | grep -q FAILED && echo "one job FAILED" && break
  echo "all three complete" && break
done

echo ">> post-update VDB version:"
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/operational/systeminfo/default" \
  | jq '.vdbVersion.vdbCurrentVersion'
```

Empirical total wall-clock on the reference box: **~6 minutes** for all three when starting from a factory-fresh VDB.

## Troubleshooting

**Update job status is FAILED with "cloud unreachable" or similar**

- Check DNS from the box: `> configure network dns show`
- Check outbound HTTPS from mgmt: `> ping outside 8.8.8.8` (won't hit updates.cisco.com but confirms egress)
- Check licensing state: `GET /license/smartagentstatuses` — `registrationStatus` should be `EVAL` or `REGISTERED`

**Update succeeds but rules don't seem to activate**

- SRU updates don't auto-deploy. Trigger a `POST /operational/deploy` after SRU update to push the new rules into the active policy.

**VDB doesn't want to update past a certain version**

- Some VDB versions require a matching FTD version. Cisco's compatibility matrix has the mapping. If you're on an older FTD, do the FTD upgrade first (Ch 3.7) then VDB.

## Related

- [Ch 3.7 FTD upgrade](upgrade.md) — FTD version has to match VDB compatibility floor for content updates to install
- [Ch 6 FDM baseline](fdm-baseline.md) — the deploy pattern used after SRU
- [Ch 7 Security policies](security-policies.md) — where the fresh SRU rules actually get applied via IPS policies on access rules
- [Ch 14 Licensing](licensing.md) — Smart Licensing state gates all update entitlement

## Next

If your box has SCC / cdFMC as the destination management path, head to [Ch 9 SCC onboarding](scc-onboarding.md). Otherwise you're at the operational plateau for FDM standalone — [Ch 15 Troubleshooting](troubleshooting.md) is the reference for what breaks in the field.
