# FDM baseline — interfaces + routing

!!! warning "⚠ WIP"
    Placeholder chapter with outline only. Content to be written as the lab is built. Feedback and PRs welcome.

Get the FW routing traffic — outside interface up, inside interface serving a LAN, default route working, DHCP handing out addresses.

## Outside interface

- Physical: WAN-side port (typically labeled `Ethernet1/1` on the 1210CE) — cabled to your ISP router or upstream L3 device
- Config: IP address (static or DHCP client), zone: `outside_zone`

**Fill in:** exact FDM click-path, screenshots of the interface config panel, gotchas around DHCP client mode.

## Inside interface

- Physical: LAN-side port (typically `Ethernet1/2` — or a bridge group across `Ethernet1/2` through `Ethernet1/8`)
- Config: IP address (typically 192.168.x.1/24), zone: `inside_zone`
- DHCP server: enable, pool, DNS

**Fill in:** exact FDM steps, VLAN considerations if using tagged sub-interfaces, downstream switch config recommendations.

## Management interface

- Already configured during [First boot](first-boot.md)
- Ensure it's on a **separate network** from the inside traffic path — MGMT should not share a subnet with inside

## Routing

- Default route: `0.0.0.0/0` → outside gateway
- Any static routes to reach lab subnets on the inside if you have multiple L3 segments

**Fill in:** static route dialog screenshots, dynamic routing (OSPF/BGP) if applicable for lab.

## DNS + NTP

- DNS servers: Cisco Umbrella (208.67.222.222 / 208.67.220.220) once you add Umbrella, or 1.1.1.1 / 8.8.8.8 for now
- NTP: pool.ntp.org or a local NTP server

## Verify

- SSH from inside network to outside test host — should egress correctly
- `ping` from outside to any WAN destination
- `traceroute` from a client on inside network — should hop through the FW

## Deploy the config

FDM stages changes until you click **Deploy**. Deploy takes ~2-3 minutes on the 1210. If it fails, check the deploy history for the specific error.

## Next

Head to [Security policies — access + IPS + URL](security-policies.md) to add basic ACL, Snort IPS, and URL filtering.
