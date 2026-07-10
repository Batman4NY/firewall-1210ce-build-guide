# Security policies — access rules + IPS + URL + file/malware

!!! success "✅ Complete · Captured 2026-07-10 live against the reference 1210CE"
    The FDM security-policy API surface + one big honest finding about smart-license eval mode that no other guide seems to call out clearly.

**Prerequisites:** [Ch 6 FDM baseline](fdm-baseline.md) complete AND you have a **registered Smart Account** or are ready to work with the eval limitations documented below.

## The load-bearing precondition — Smart Account state

**Verified live on the reference 1210CE on FTD 7.6.4-69 with eval-only smart-license state**:

Adding **any** access rule to the default access policy — even a bare `inside_zone → outside_zone PERMIT LOG_NONE` — causes `POST /operational/deploy` to fail with:

```
state:      DEPLOY_FAILED
cliLines:   !!!FdmCliStart
            strong-encryption-disable
            no dp-tcp-proxy
            !!!FdmCliEnd
```

Confirmed by:

1. Enabling `THREAT` / `MALWARE` / `URLFILTERING` licenses (`POST /license/smartlicenses`) doesn't help — same failure with them enabled and with them removed.
2. Discarding all pending changes + deploying returns state `DEPLOYED` in ~7 seconds — the failure appears the instant any access rule change is staged.
3. Post-upgrade retest (7.6.0-113 → 7.6.4-69 → CC-7.6.4.1-1) reproduced the exact same failure with the exact same `cliLines`.
4. `GET /license/smartagentstatuses` on the eval-only box reports **`exportControl: null`**.

The interpretation from the evidence: the FTD data-plane's `strong-encryption` state depends on the export-controlled features flag on the Smart Account, and the box refuses to deploy any policy that would touch the crypto/decryption path (which access rules do, even without IPS or URL filtering) while `exportControl` is null. `POST /license/smartlicenses` for THREAT/MALWARE/URL doesn't set that flag — those are separate feature licenses.

### Practical guidance

- **Lab / POC / RMA-return boxes on eval only** — you can read the security-policy state via API (documented below) but you cannot deploy any access-rule change. Everything in the "action" sections of this chapter requires a real Smart Account.
- **Production and customer boxes** — register a Smart Account (`POST /license/smartagentconnections`) BEFORE entering this chapter. That's documented in [Ch 14 Licensing](licensing.md). This chapter assumes you've done it.
- **Air-gapped / classified** — use a Permanent License Reservation (PLR); [Ch 14](licensing.md) covers that flow.

The rest of this chapter shows the API patterns. All GETs are captured live from the reference box; the POST/PUT payloads are validated against the schema but their deploy is gated on Smart Account state.

## Auth

```bash
FDM_HOST=https://192.168.40.10
FDM_PW=$(bao kv get -field=password infra/webui/fw1210ce-admin)
TOKEN=$(curl -sk -X POST "$FDM_HOST/api/fdm/latest/fdm/token" \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\":\"password\",\"username\":\"admin\",\"password\":\"$FDM_PW\"}" \
  | jq -r '.access_token')
```

## Factory-default policy state

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/policy/accesspolicies"
```

```json
{"items":[{
  "name":          "NGFW-Access-Policy",
  "id":            "c78e66bc-cb57-43fe-bcbf-96b79b3475b3",
  "defaultAction": {"action":"DENY"}
}]}
```

**Default action is `DENY` (fail-closed).** No rules are pre-seeded. Every allow is opt-in via explicit rule.

## Zones + Cisco-shipped policy objects

```bash
# Zones (interface groupings)
curl -sk -H "Authorization: Bearer $TOKEN" "$FDM_HOST/api/fdm/latest/object/securityzones"
# → inside_zone (90c377e0-...), outside_zone (b1af33e1-...)

# Intrusion policies (built-in)
curl -sk -H "Authorization: Bearer $TOKEN" "$FDM_HOST/api/fdm/latest/policy/intrusionpolicies"
# → 8 policies, four pairs of {Base / Cisco Talos suffix}

# File policies (built-in)
curl -sk -H "Authorization: Bearer $TOKEN" "$FDM_HOST/api/fdm/latest/policy/filepolicies"
# → Block Malware All, Malware Cloud Lookup - No Block

# URL categories (Cisco Talos taxonomy)
curl -sk -H "Authorization: Bearer $TOKEN" "$FDM_HOST/api/fdm/latest/object/urlcategories?limit=200"
# → 144 categories total; high-risk ones useful for the block-list: Malware Sites, Phishing,
#   Botnets, Cryptomining, Exploits, Newly Seen Domains, Ebanking Fraud, etc.
```

!!! danger "Gotcha: Cisco Talos intrusion policies can't be referenced directly from an access rule"
    The four `- Cisco Talos` variants are system-defined and refuse to be attached to an access rule:
    ```
    "acRuleCannotUseSystemDefinedPolicy": The selected intrusion policy is defined by the system. You must select a custom policy.
    ```
    Use the four non-Talos variants (`Balanced Security and Connectivity`, `Connectivity Over Security`, `Maximum Detection`, `Security Over Connectivity`) which are user-editable. If you want the Talos rule set, clone a Talos variant into a custom policy first.

## Adding an access rule — the API payload

For an `inside → outside` allow with IPS + File policy:

```bash
POL_ID="c78e66bc-cb57-43fe-bcbf-96b79b3475b3"

curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/policy/accesspolicies/$POL_ID/accessrules" \
  -d '{
    "name":            "inside-to-outside-allow",
    "ruleAction":      "PERMIT",
    "eventLogAction":  "LOG_BOTH",
    "sourceZones":      [{"id":"90c377e0-b3e5-11e5-8db8-651556da7898","type":"securityzone","name":"inside_zone"}],
    "destinationZones": [{"id":"b1af33e1-b3e5-11e5-8db8-afdc0be5453e","type":"securityzone","name":"outside_zone"}],
    "intrusionPolicy":  {"id":"2541f9b6-7c98-11f1-990f-6b76e5c45725","type":"intrusionpolicy","name":"Balanced Security and Connectivity"},
    "filePolicy":       {"id":"bf8b32a2-7f47-11e5-b36f-4ad0763818b0","type":"filepolicy","name":"Block Malware All"},
    "type":             "accessrule"
  }'
```

Rule POST returns 200 with a new `id` and `ruleId` — the change is **staged**. Deploy applies it (or fails per the precondition above).

## URL category filtering — the payload

Block a high-risk category set on a separate DENY rule placed BEFORE the allow rule:

```bash
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/policy/accesspolicies/$POL_ID/accessrules" \
  -d '{
    "name":            "block-high-risk-url",
    "ruleAction":      "DENY",
    "eventLogAction":  "LOG_BOTH",
    "sourceZones":      [{"id":"90c377e0-b3e5-11e5-8db8-651556da7898","type":"securityzone","name":"inside_zone"}],
    "destinationZones": [{"id":"b1af33e1-b3e5-11e5-8db8-afdc0be5453e","type":"securityzone","name":"outside_zone"}],
    "urlFilter": {
      "urlObjects": [],
      "urlCategories": [
        {"urlCategory":{"id":"abba9b63-bb10-4729-b901-2e2aa0f01001","type":"urlcategory","name":"Malware Sites"}, "type":"urlcategorymatcher"},
        {"urlCategory":{"id":"abba9b63-bb10-4729-b901-2e2aa0f01003","type":"urlcategory","name":"Phishing"}, "type":"urlcategorymatcher"},
        {"urlCategory":{"id":"abba9b63-bb10-4729-b901-2e2aa0f01004","type":"urlcategory","name":"Botnets"}, "type":"urlcategorymatcher"},
        {"urlCategory":{"id":"abba9b63-bb10-4729-b901-2e2aa0f01006","type":"urlcategory","name":"Exploits"}, "type":"urlcategorymatcher"},
        {"urlCategory":{"id":"abba9b63-bb10-4729-b901-2e2aa0f02112","type":"urlcategory","name":"Cryptomining"}, "type":"urlcategorymatcher"},
        {"urlCategory":{"id":"abba9b63-bb10-4729-b901-2e2aa0f01020","type":"urlcategory","name":"Newly Seen Domains"}, "type":"urlcategorymatcher"}
      ],
      "type": "embeddedurlfilter"
    },
    "type": "accessrule"
  }'
```

**Order matters — access rules process top-down.** Put block rules ABOVE allow rules or the allow will match first. FDM's rule-reordering API is a separate endpoint (`PUT` on the access-rule with `ruleId` field); this chapter's default POST appends at the bottom.

## Feature-license activation

Before the rule-deploy path fails on strong-encryption, you'll separately need to activate the per-feature smart licenses:

```bash
for LIC in THREAT MALWARE URLFILTERING; do
  curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    "$FDM_HOST/api/fdm/latest/license/smartlicenses" \
    -d "{\"licenseType\":\"$LIC\",\"count\":1,\"type\":\"license\"}"
done
```

Each returns `compliant: true` on eval mode. But without export-controlled features enabled on a **registered** Smart Account, the rule deploy still fails per the precondition section.

## Deploy pattern

Same as [Ch 6 Step 8](fdm-baseline.md#step-8-deploy-the-queued-changes):

```bash
POST /api/fdm/latest/operational/deploy   # returns deploy id, state QUEUED
GET  /api/fdm/latest/operational/deploy/<id>   # poll until state DEPLOYED or DEPLOY_FAILED
```

If DEPLOY_FAILED, get the CLI-level cause:

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/operational/deploymenterrors/<deploy-id>"
```

If you see `strong-encryption-disable` in `cliLines`, that's the export-control gate. Register a Smart Account and try again.

## Discarding pending changes

If a deploy fails and you want to reset the box's staged-config state before doing anything else:

```bash
curl -sk -X DELETE -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/operational/pendingchanges"
```

Returns HTTP 204. All uncommitted PUTs and POSTs are dropped. A subsequent empty `POST /operational/deploy -d '{}'` deploys clean in ~7 sec — good for confirming the box's baseline is deployable independent of your changes.

## Verify — access-policy state after deploy

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/policy/accesspolicies/$POL_ID/accessrules?limit=25"
```

Fields to check per rule:

- `ruleAction` matches intent (`PERMIT` / `DENY` / `TRUST`)
- `intrusionPolicy.name` / `filePolicy.name` present if you set them
- `urlFilter.urlCategories[]` populated if URL filtering
- `eventLogAction` matches your logging strategy (`LOG_NONE` / `LOG_AT_END_OF_CONNECTION` / `LOG_BOTH`)

## Runtime verification

Traffic side:

- From a client on the inside network → outbound to `www.google.com` — should pass (permit rule)
- From same client → outbound to a Talos test URL (e.g., `www.testmyids.com`) — should be blocked with a Snort event
- `GET /operational/hitcounts` on the policy — counters should tick up on the matching rules

Log side:

- FDM → **Monitoring → Events → Access Rules** — see hits with the rule name column
- Syslog to your SIEM if `syslogServer` is set on the rule or on the policy default action

## Related

- [Ch 6 FDM baseline](fdm-baseline.md) — the deploy pattern used here
- [Ch 8 Talos intel + updates](talos-intel.md) — VDB, Snort rule updates, URL category refresh
- [Ch 14 Licensing](licensing.md) — Smart Account registration → export-controlled features unlock → rule deploy works

## Next

Head to [Ch 8 Talos intel + updates](talos-intel.md) to configure automatic VDB + Snort rule refresh, or [Ch 9 SCC onboarding](scc-onboarding.md) if you're moving off standalone-FDM management.
