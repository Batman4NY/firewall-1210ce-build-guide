# Umbrella SASE tunnel

!!! warning "⚠ WIP"
    Placeholder chapter with outline only. Content to be written as the lab is built. Feedback and PRs welcome.

**Prerequisites:** [Chapter 6: FDM baseline](fdm-baseline.md) — FTD with reachable outside IP.

Wire **Cisco Umbrella** (DNS-layer filtering + SIG for full SWG) into the FTD via a SASE tunnel. Umbrella eval / free tier is available for lab use.

!!! note "Licensing note"
    Umbrella integration on FTD requires the DNS Security (Umbrella) module — see [Chapter 14: Licensing](licensing.md).

## Umbrella tenant setup

- Sign up at [umbrella.com](https://umbrella.com/) — Cisco Employee Program covers Umbrella eval
- Get API keys for automation
- Set up initial policy (block malware, phishing, uncategorized)

**Fill in:** Umbrella tenant setup steps, dashboard tour.

## Network deployment

Two approaches:

- **DNS forwarding** — point the FTD's DHCP server to Umbrella DNS (208.67.222.222 / .220.220). Simple, DNS-only filtering.
- **SIG (Secure Internet Gateway) tunnel** — IPsec tunnel from FTD to Umbrella. Full SWG (web filtering, cloud DLP, etc.)

For lab: DNS forwarding first, add SIG tunnel later.

## DNS forwarding config

- FDM → Objects → Network → create DNS server object for Umbrella
- FDM → Device → System Settings → DNS → point clients at Umbrella
- Deploy

**Fill in:** exact FDM steps.

## SIG tunnel config

- Umbrella dashboard → Deployments → Network Tunnels → Add tunnel
- Provide the FTD's outside IP as the tunnel endpoint
- Get pre-shared key + tunnel config
- FDM → Site-to-Site VPN → Add IKEv2 tunnel to Umbrella
- Configure route: outbound web traffic → Umbrella tunnel

**Fill in:** IKE proposals, IPsec params, routing config.

## Verify

- `dig google.com @<internal-client>` — should return Umbrella-annotated response
- Browse to a test malicious URL — should be blocked by Umbrella (different block page than FTD's URL filtering)
- Umbrella dashboard should show the FTD as an active tunnel client

## Next

Head to [ThousandEyes on the FW](thousandeyes-integration.md) for network path visibility.
