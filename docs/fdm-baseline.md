# FDM baseline — hostname, DNS, mgmt IP, NTP

!!! success "✅ Complete · Captured 2026-07-10 live against the reference 1210CE"
    Every `curl` in this chapter was executed against the running box and the response bodies are transcribed verbatim. Empirical timings for each deploy are noted inline.

**Prerequisites:** [Ch 4 First-boot verification](first-boot.md) complete — you're logged in to FDM at `https://<mgmt-ip>/` and clicked through the initial setup dialog.

## Step 0 — Complete FDM's initial-setup wizard (UI-only, one-time)

!!! danger "This gate blocks the REST API — you cannot skip it"
    Every mutating FDM REST endpoint (`/api/fdm/latest/devices/default/...`, `/devicesetup/...`, even `/openapi/spec`) returns `HTTP 403 { "message": "Not allowed - Device initial setup is not completed!" }` until you click through FDM's browser-side initial-setup wizard **at least once**. Read endpoints like `/operational/systeminfo` do work — you can query serial number + version — but nothing that changes state.

    A "lights-out from CLI only" build script cannot bypass this. Complete the UI setup once per box, then everything is API-drivable from that point forward.

**In the browser at `https://<mgmt-ip>/#/login`:**

1. Log in with `admin` and the password you set during the FXOS forced-password-change (Ch 3.5).
2. FDM presents a "Device Setup" wizard. Click **Skip device setup** at the bottom right of the first page:
    - The blue callout at the top confirms this is the right choice: *"Do not use the startup wizard if you want to use Low-Touch Provisioning with... Cisco Defense Orchestrator (CDO) [or] Secure Firewall Management Center."* Your arc ends at SCC (Ch 8), so LTP-friendly means skipping the wizard.
    - A confirmation dialog appears with **Start 90-day evaluation** already checked. Click **CONFIRM**.
    - Wait ~30-60 sec for "Device initialization is in progress" to complete.
3. You land at the FDM **Dashboard**. REST API is now unlocked.

## Auth — get a bearer token

```bash
FDM_HOST=https://192.168.40.122
FDM_PW=$(bao kv get -field=password infra/webui/fw1210ce-admin)
TOKEN=$(curl -sk -X POST "$FDM_HOST/api/fdm/latest/fdm/token" \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\":\"password\",\"username\":\"admin\",\"password\":\"$FDM_PW\"}" \
  | jq -r '.access_token')
```

The token is a 448-char JWT good for ~30 min. Every request below uses `Authorization: Bearer $TOKEN`.

## Step 1 — Custom hostname

The 7.6.x FTD wizard doesn't prompt for a hostname; the box comes up as `firepower`. Change it.

```bash
# GET current
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/devicesettings/default/devicehostnames" | jq
```

Returns a single-item collection:

```json
{
  "items": [{
    "version": "kxaohieaj6r6o",
    "name": null,
    "hostname": "firepower",
    "id": "30e39363-7c98-11f1-990f-eb695078beff",
    "type": "devicehostname"
  }]
}
```

PUT the new value — include the same `id`, `type`, and current `version` (optimistic-concurrency token):

```bash
curl -sk -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/devicesettings/default/devicehostnames/30e39363-7c98-11f1-990f-eb695078beff" \
  -d '{
    "hostname": "fw1210ce",
    "version":  "kxaohieaj6r6o",
    "id":       "30e39363-7c98-11f1-990f-eb695078beff",
    "type":     "devicehostname"
  }'
```

Response `version` bumps to `mwbtp5qjisztu` — the change is staged but not deployed. Every subsequent PUT on this object must reference the current version or FDM returns HTTP 400.

## Step 2 — Custom DNS + search domain

FDM's default DNS points at Cisco Umbrella (`208.67.222.222 / 208.67.220.220 / 2620:119:35::35`). Two levels of object to update:

1. The **DNS Server Group** object (holds the actual IPs + search domain)
2. The **Device DNS Settings** object (references a DNS server group)

!!! warning "Cisco Umbrella group is system-protected"
    You cannot rename `CiscoUmbrellaDNSServerGroup` or change its IPs. Attempting PUT returns:
    ```
    "invalidUmbrellaDnsServerGroup": You cannot change the name or DNS server IP addresses in the system-defined CiscoUmbrellaDNSServerGroup object.
    ```
    The correct pattern is to **POST a new group** and then **PUT the mgmt DNS settings to reference it.**

### Create a new DNS server group

```bash
curl -sk -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/object/dnsservergroups" \
  -d '{
    "name":         "LocalDNSServerGroup",
    "dnsServers":   [{"ipAddress": "192.168.1.3", "type": "dnsserver"}],
    "timeout":      2,
    "retries":      2,
    "searchDomain": "uppernyack.com",
    "type":         "dnsservergroup"
  }'
```

Response includes the new object's `id` and `version` — save both.

### Repoint the mgmt DNS settings

```bash
curl -sk -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/devices/default/mgmtdnssettings/00000014-0000-0000-0000-000000000001" \
  -d '{
    "version":        "<current-mgmtdns-version>",
    "name":           "DeviceDNSSettings",
    "dnsServerGroup": {
      "version": "<new-group-version>",
      "name":    "LocalDNSServerGroup",
      "id":      "<new-group-id>",
      "type":    "dnsservergroup"
    },
    "id":   "00000014-0000-0000-0000-000000000001",
    "type": "devicednssettings"
  }'
```

## Deploy — hostname + DNS (safe, non-disruptive)

```bash
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/operational/deploy" -d '{}'
```

Returns a deploy object with `id` and `state: "QUEUED"`. Poll `GET /operational/deploy/<id>` until `state: "DEPLOYED"` or `"FAILED"`. On this reference build, hostname + DNS deploy in **2m 17s** (queued 16:09:17 → deployed 16:11:34).

## Step 3 — Static management IP

The 7.6.x wizard leaves mgmt on DHCP. To pin it (matches your UDM DHCP reservation if you have one), change to static.

!!! warning "This is the disruptive change"
    Changing the mgmt IP tears down your current HTTPS/API session when the deploy activates. Do this change and deploy alone — reconnect on the new IP afterwards.

```bash
# GET current — copy id + version for the PUT
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/devicesettings/default/managementips" | jq
```

Response:

```json
{
  "items": [{
    "version":      "fq7jq5dhwmlti",
    "ipv4Mode":     "DHCP",
    "ipv4Address":  "192.168.40.122",
    "ipv4NetMask":  "255.255.255.0",
    "ipv4Gateway":  "192.168.40.1",
    "ipv6Mode":     "STATIC",
    "mtu":          1500,
    "linkState":    "UP",
    "id":           "311355f4-7c98-11f1-990f-67c03bb038c5",
    "type":         "managementip"
  }]
}
```

PUT the static config:

```bash
curl -sk -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/devicesettings/default/managementips/311355f4-7c98-11f1-990f-67c03bb038c5" \
  -d '{
    "version":     "fq7jq5dhwmlti",
    "ipv4Mode":    "STATIC",
    "ipv4Address": "192.168.40.10",
    "ipv4NetMask": "255.255.255.0",
    "ipv4Gateway": "192.168.40.1",
    "ipv6Mode":    "STATIC",
    "mtu":         1500,
    "linkState":   "UP",
    "id":          "311355f4-7c98-11f1-990f-67c03bb038c5",
    "type":        "managementip"
  }'
```

Deploy. Your session drops at the moment the interface flip activates — reconnect on the new IP:

```bash
FDM_HOST=https://192.168.40.10   # new value
TOKEN=$(curl -sk -X POST "$FDM_HOST/api/fdm/latest/fdm/token" \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\":\"password\",\"username\":\"admin\",\"password\":\"$FDM_PW\"}" \
  | jq -r '.access_token')
```

Verify:

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/devicesettings/default/managementips" \
  | jq '.items[0] | {ipv4Mode, ipv4Address, ipv4Gateway, linkState}'
```

Update your OpenBao entry:

```bash
bao kv patch infra/webui/fw1210ce-admin host=192.168.40.10 fdm_url=https://192.168.40.10
```

## Step 4 — Verify interfaces + zones (read-only)

At this point the box has factory-default data-plane interfaces:

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/devices/default/interfaces?limit=20" | jq
```

On the reference build:

| hardwareName | name | mode | linkState | ipv4Mode | ipv4Address |
|---|---|---|---|---|---|
| `Management1/1` | `management` | `ROUTED` | `UP` | `STATIC` | `192.168.40.10/24` |
| `Ethernet1/1` | `outside` | `ROUTED` | `UP` | `DHCP` | `192.168.1.196/24` |
| `Ethernet1/2` — `Ethernet1/8` | (unassigned) | `SWITCHPORT` | `DOWN` | `STATIC` | (no IP) |

The 1200-series factory default:

- **Ethernet1/1** = `outside` zone, ROUTED, DHCP client. Cable to your ISP/upstream.
- **Ethernet1/2 through Ethernet1/8** = switchports on VLAN1 (default `inside` bridge at `192.168.95.1/24`). Any host on ports 2-8 gets a DHCP lease from the FTD.

For a lab that's a firewall behind another router (the typical home scenario), the outside interface picks up a DHCP lease and outbound NAT is auto-configured. Nothing to change unless your inside network needs a different subnet or you're inserting the FW as the LAN's primary router.

## Step 5 — NTP

The 7.6.x default NTP config is **disabled** (`enabled: false`) and points at Cisco's `sourcefire.pool.ntp.org`. Enable it and switch to a source you trust.

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$FDM_HOST/api/fdm/latest/devicesettings/default/ntp" | jq
```

Returns two NTP objects — the operational one is the all-zeros GUID at `00000004-0000-0000-0000-000000000002`. The other is a legacy Sourcefire object that FDM keeps for backward compatibility; don't touch it.

```bash
curl -sk -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$FDM_HOST/api/fdm/latest/devicesettings/default/ntp/00000004-0000-0000-0000-000000000002" \
  -d '{
    "version":    "<current-version>",
    "enabled":    true,
    "ntpServers": ["0.pool.ntp.org","1.pool.ntp.org","2.pool.ntp.org","3.pool.ntp.org"],
    "id":         "00000004-0000-0000-0000-000000000002",
    "type":       "ntp"
  }'
```

Deploy takes **1m 22s** on the reference build.

## FDM API patterns learned in this chapter

For any subsequent FDM chapter (Ch 7 security policies, Ch 8 Talos), the following patterns repeat:

1. **Auth**: `POST /api/fdm/latest/fdm/token` with `grant_type=password` → 30-min JWT
2. **Discovery**: `GET /apispec/ngfw.json` (not `/api/fdm/latest/openapi/spec` — that returns 404 despite Cisco's docs) → 1.2 MB OpenAPI spec, 528 paths
3. **Optimistic concurrency**: every mutation requires the current `version` — if you send stale, FDM returns HTTP 400 `{"key": "Validation", "code": "invalidVersion"}`
4. **System-protected objects**: some Cisco-provided defaults (e.g., `CiscoUmbrellaDNSServerGroup`) refuse rename/data mutation; POST a new object and PUT the parent to reference it
5. **Staging vs deployment**: PUT/POST stages changes on the box; nothing takes effect until `POST /operational/deploy` completes. Multiple staged changes deploy atomically.
6. **Deploy polling**: `GET /operational/deploy/<id>` returns `state` in `{QUEUED, DEPLOYING, DEPLOYED, FAILED}`. Typical wall-clock: 1-2 min for simple changes, 3-5 min for interface/policy changes.

## Verify

- **Ping** from your workstation to the new mgmt IP → 100% loss-free
- **Access** `https://<new-mgmt-ip>/` in the browser → FDM UI loads
- `show network` on the FTD console → matches the API's reported values

## Next

Head to [Ch 7 — Security policies](security-policies.md) to build access rules, IPS profile, and URL filtering via FDM API.
