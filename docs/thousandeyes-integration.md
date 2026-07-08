# ThousandEyes on the FW

!!! warning "⚠ WIP"
    Placeholder chapter with outline only. Content to be written as the lab is built. Feedback and PRs welcome.

**Prerequisites:** [Chapter 6: FDM baseline](fdm-baseline.md); FTD 7.6+ for native integration path.

Add **Cisco ThousandEyes** for end-to-end path visibility from the FW outward. Cisco tenant access via employee credentials.

!!! note "Licensing note"
    ThousandEyes on Nexus/FTD requires a ThousandEyes tenant + endpoint agent. See [Chapter 14: Licensing](licensing.md).

## ThousandEyes tenant

- Access via [thousandeyes.com](https://thousandeyes.com/) with Cisco credentials
- Existing Cisco tenant already has 200+ cloud agents worldwide
- Add a local agent for home-lab-side visibility

## Local agent — deploy on a Pi

The most common home-lab pattern:

- Raspberry Pi 4 (or spare Pi 3B+)
- ThousandEyes Endpoint Agent for Linux
- Runs on the inside network — sees the path from home to any test target

**Fill in:** exact Pi setup steps, agent install, test config.

## FTD as a target

- Add a **test** in the ThousandEyes dashboard: Network → HTTP Server → target the FTD's outside IP or a hosted URL on the FW-served demo
- Schedule: every 5 minutes
- Add path visualization

## Native FTD integration (post-onboarding)

FTD 7.6+ has native ThousandEyes integration — the FTD itself can act as a test target. Configure via FDM:

**Fill in:** FDM's ThousandEyes integration steps, endpoint agent registration.

## Verify

- ThousandEyes dashboard → Test → see path from Pi to FTD to internet
- Path visualization shows FTD as a hop
- Latency + loss stats populated within 5-10 min

## Use case

The main demo value: **when a customer says "the firewall is slow"**, point at the ThousandEyes test showing the actual latency spike is upstream. Firewall vendors get blamed for problems they don't own — this proves the fault location in one screen.

## Next

Head to [Licensing — eval to production](licensing.md) for how to graduate from 90-day eval to real smart licensing.
