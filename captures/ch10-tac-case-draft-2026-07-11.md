# Cisco TAC case draft — 2026-07-11

## Case summary (one line)

Fresh SCC Base-tier tenant `1210CE-Lab` (region us) cannot mint an API token with `Firewall Management` scope — no UI path found. Empirically-verified across three token-mint surfaces. Blocks Firewall Manager REST API automation.

## Tenant details

- **Tenant name:** `1210CE-Lab`
- **Tenant hostname (cdFMC):** `1210ce-lab--ti0cqk.app.us.cdo.cisco.com`
- **Enterprise ID:** `309b5034-2f70-4b77-b5ca-1c34a5ed2e22`
- **Region:** us
- **Subscription tier:** Firewall Management Base
- **cdFMC provisioning:** Complete (`fw1210ce` device onboarded and synced 2026-07-11)
- **User initiating request:** `bgarcia2@cisco.com` (Federated / Active)
- **User assigned roles:**
  - Direct: `Security Cloud Control — Organization Administrator` (Static)
  - Direct: `Firewall Management — Super Administrator` (Static, assigned 2026-07-11 during troubleshooting)
  - Group: "All Products Administrator" system group — carries `Firewall Management: Super Administrator` and `Multicloud Defense: Administrator`

## Symptom

Every request to `https://api.us.security.cisco.com/firewall/v1/*` returns HTTP 400 through the SCC gateway, regardless of URL variant tried. The response body's `path` field shows the gateway rewriting to `/api/platform/scc-gateway/request/api/rest/v1/*`.

Request auth header uses `Authorization: Bearer <TOKEN>` where `<TOKEN>` is a JWT minted from the tenant's `Platform Management → API Keys → Generate API key` UI. The `Accept: application/json` header is present. The token's JWT payload includes:
- `iss: https://idbroker-b-us.webex.com/idb`
- `token_type: Bearer`
- `user_type: machine`
- `machine_type: bot`
- `org_id: d1dbb9db-29f4-4547-9de5-3b436015f0f0`
- `cis_uuid: 806dc3e8-4c11-4a87-a7d3-db7abf0cfc40` (matches the API key ID in SCC UI)

## Sample request/response

```
Request:
  GET https://api.us.security.cisco.com/firewall/v1/inventory/managers?q=deviceType:CDFMC
  Authorization: Bearer <TOKEN>
  Accept: application/json

Response:
  HTTP/1.1 400 Bad Request
  Content-Type: application/json

  {
    "timestamp": "2026-07-11T14:45:33.076Z",
    "path": "/api/platform/scc-gateway/request/api/rest/v1/inventory/managers",
    "status": 400,
    "error": "Bad Request",
    "requestId": "ea776cf8-1271752"
  }
```

Same behavior on: `/firewall/v1/token`, `/firewall/v1/inventory/devices`, `/firewall/v1/cdfmc/api/fmc_platform/v1/info/domain`.

Direct-to-cdFMC hostname (`https://1210ce-lab--ti0cqk.app.us.cdo.cisco.com/api/fmc_platform/v1/info/domain`) returns HTTP 401 with:

```json
{"error":{"category":"FRAMEWORK","messages":[{"description":"SCC token is invalid"}],"severity":"ERROR"}}
```

## Root cause hypothesis

The token was minted from `Platform Management → API Keys` which — per empirical UI observation — only offers `Security Cloud Control` as an assignable product/service. The token therefore has SCC Platform scope only, not Firewall Manager scope. The SCC gateway sees the token (returns 400 not 401) but refuses to route it to the Firewall Manager product surface.

## UI paths exhaustively checked for Firewall-Manager-scoped API token minting

| Attempted path | Result |
|---|---|
| `Platform Management → API Keys → Generate API key` | Product/Service dropdown offers only "Security Cloud Control". No "Firewall Management" option. |
| `Platform Management → Administrator Access → + Invite` | 3-step wizard requires First Name / Last Name / Email. Creates human federated users only. No `API Only User` checkbox in any step. |
| `Platform Management → Administrator Access → Admin groups → All Products Administrator → Add users` | Same human-user invite flow. No API-only option. |
| cdFMC (Cloud-Delivered Firewall Management Center) → username dropdown → `User Preferences` | Only offers Time Zone. No `My Tokens`, no `Generate API Token`. |

Cisco DevNet docs at [docs.defenseorchestrator.com/cdfmc/t-generatean-api-token.html](https://docs.defenseorchestrator.com/cdfmc/t-generatean-api-token.html) describe a `Preferences → General Preferences → My Tokens` path that does not exist in the 2026 UI on this tenant. Cisco docs at [edge.ci.cdo.cisco.com/content/docs/t-create-api-only-users.html](https://edge.ci.cdo.cisco.com/content/docs/t-create-api-only-users.html) describe a `Administration → API User Management` path that also does not exist as a menu item in this tenant.

## The specific ask

Please confirm which of the following applies to this tenant, and enable Firewall Manager API access accordingly:

1. **Is Firewall Manager REST API access included in the Base tier subscription?** If yes, our tenant is missing the UI path to mint the token — please enable the `Administration → API User Management` (or equivalent) page.
2. **If Firewall Manager REST API access requires a Premier-tier or add-on subscription**, please document that requirement clearly and provide the SKU / add-on ordering path.
3. **If our tenant has a configuration bug** preventing the API-Only-User creation UI from appearing, please correct it on the backend.

## Attachments

Screenshots documenting all four UI paths I checked have been captured empirically today and can be attached to the case.

## Reference

This tenant is being used to develop the open-source [Cisco Secure Firewall 1210CE Build Guide](https://github.com/Batman4NY/firewall-1210ce-build-guide). Ch 10 of that guide is currently blocked on this API token question. Your resolution will be documented in the guide with proper attribution.
