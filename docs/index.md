# Cisco Secure Firewall 1210CE Build Guide

!!! warning "Work in Progress"
    This guide is being written as I build out the lab. Chapters tagged **✅ complete** are ready to follow. Chapters tagged **📝 draft** have skeleton content and are being fleshed out. Chapters tagged **⚠ WIP** have placeholders only. Check the [chapter status matrix](#chapter-status) below.

!!! danger "Personal project — not Cisco-official"
    Home lab build guide by a Cisco employee on their own time. Not affiliated with, endorsed by, sponsored by, or representing Cisco Systems, Inc. Product references are to publicly available Cisco offerings.

A build guide for setting up the **Cisco Secure Firewall 1210CE** as a home-lab node — from unboxing through **FTD 7.6** baseline, **Security Cloud Control (SCC)** onboarding, and integration with the surrounding Cisco security portfolio.

## What you'll end up with

A working **Cisco Secure Firewall 1210CE** running **FTD 7.6**, managed both locally and centrally:

- **Locally** via FDM (Firepower Device Manager, on-box web UI) — good for lab-scale
- **Centrally** via Security Cloud Control (SCC) — good for the "here's how customers do it" story

Plus the surrounding portfolio wired in as time and licensing allow:

| Product | Integration | Chapter |
|---|---|---|
| **Talos intel** | Automatic IPS + URL feed | [Talos intel](talos-intel.md) |
| **Duo** | MFA on FDM + SCC admin access | [Duo integration](duo-integration.md) |
| **Umbrella** | SASE tunnel for DNS-layer filtering | [Umbrella integration](umbrella-integration.md) |
| **ThousandEyes** | Local agent + FW-side visibility | [ThousandEyes integration](thousandeyes-integration.md) |
| **XDR** | Correlation across FW + endpoint + email + cloud | Post-onboarding — see the SCC chapter |

## Why the 1210CE

The **1210CE** is the smallest current-gen (2025+) Cisco Secure Firewall with **full FTD 7.6 feature parity** — same software image as the 3100 and 4200 series, just at a lab-scale form factor and price point. That matters for two reasons:

1. **Every workflow you build here scales up unchanged.** The FDM config export imports into a 3100. The SCC-managed config path is identical. Nothing you learn on the 1210 becomes wrong at the 4200 tier.
2. **You get a real enterprise gateway on your desk.** Not a virtual FTDv, not a simulated Sourcefire — an actual metal box with real interfaces you can plug real gear into.

If you have access to a 1210CE via the Cisco Employee Program, partner allocation, or purchase — this guide takes you from unopened box to a working demo node.

## Prerequisites

- A **Cisco Secure Firewall 1210CE** (see [Bill of materials](bill-of-materials.md))
- A working **ConsolePi** or equivalent serial console access — see the [companion guide](https://batman4ny.github.io/consolepi-build-guide/)
- **Cisco.com credentials** with entitlement to download FTD images (Cisco employees / partners / customers with a valid support contract)
- **Cisco Smart Software Manager** account — for smart licensing (or 90-day eval)
- **Cisco Security Cloud Control** tenant access (via the same Cisco.com credentials)
- **Home network** with an internet uplink you can put the FW between (or a routed subnet where the FW can act as gateway)
- **A LAN switch** on the inside — the 1210 doesn't have enough ports to be your whole home LAN, so it needs a downstream switch (any managed switch works)

## Time budget

| Phase | Time |
|---|---|
| Unbox + rack + cable | 30 min |
| Console access + first boot | 30 min |
| FDM initial setup wizard | 15 min |
| Interfaces + routing baseline | 30 min |
| Security policies (IPS + URL) | 30 min |
| SCC onboarding | 30-60 min |
| Duo + Umbrella + TE integrations | ~1 hr each |
| **Baseline (through Talos intel)** | **~2.5 hrs** |
| **Full portfolio (all integrations)** | **~7-8 hrs** |

Reasonable to do the baseline in an evening and layer integrations on over subsequent weekends.

## What this guide is NOT

- **Not a substitute for the [official Cisco Secure Firewall 1200 Series Getting Started Guide](https://www.cisco.com/c/en/us/support/security/secure-firewall-1200-series/products-installation-and-configuration-guides-list.html).** Read that too. This guide is opinionated on top of official docs — it captures the specific pattern for a home-lab / demo-bench build, and skips or condenses steps that are less relevant at that scale.
- **Not production hardening guidance.** The recommendations here are lab-appropriate. Production deployments need proper change control, HA, DR, secrets management, monitoring, etc.
- **Not affiliated with, endorsed by, or representing Cisco Systems, Inc.** Personal project, personal opinions.

## Chapter status

Status extracted from each chapter's opening admonition, cross-checked against actual file content and live-capture commits (last update: 2026-07-11).

| # | Chapter | Status |
|---|---|---|
| 1 | [Overview](index.md) | 📝 Draft |
| 2 | [Bill of materials](bill-of-materials.md) | 📝 Draft — baseline complete; SKU + rack accessories still being iterated |
| 3 | [Console access via ConsolePi](console-access.md) | ✅ Complete |
| 3.5 | [Remote factory reset](remote-factory-reset.md) | ✅ Complete — includes the FXOS reimage flow used as the standard reset path |
| 3.7 | [FTD upgrade — in-place](upgrade.md) | ✅ Complete — live-captured 2026-07-10 |
| 4 | [First boot verification](first-boot.md) | ✅ Complete — rewritten 2026-07-10 after live capture |
| 5 | [Choosing your management path](choose-mgmt-path.md) | 📝 Draft — SCC subscription-tier finding integrated |
| 6 | [FDM baseline — interfaces + routing](fdm-baseline.md) | ✅ Complete — live-captured 2026-07-10 |
| 7 | [Security policies — access + IPS + URL](security-policies.md) | ✅ Complete — live-captured 2026-07-10 including strong-crypto trap |
| 8 | [Talos intel + updates](talos-intel.md) | ✅ Complete — live-captured 2026-07-10; cdFMC-supersession callout added 2026-07-11 |
| 9 | [Onboarding to Security Cloud Control](scc-onboarding.md) | ✅ Live-captured 2026-07-11 · 12 screenshots · Task Manager CSV · full state machine + retry gotcha |
| 10 | [Managing via cdFMC](scc-managed.md) | 🚧 Partial — UI walkthrough + REST research complete; API round-trip Step 4 blocked on TAC case for Firewall Manager API access on Base tier (see [captures/ch10-tac-case-draft-2026-07-11.md](https://github.com/Batman4NY/firewall-1210ce-build-guide/blob/main/captures/ch10-tac-case-draft-2026-07-11.md)) |
| 11 | [Duo MFA on FDM/SCC](duo-integration.md) | ⚠ WIP — outline only |
| 12 | [Umbrella SASE tunnel](umbrella-integration.md) | ⚠ WIP — outline only |
| 13 | [ThousandEyes on the FW](thousandeyes-integration.md) | ⚠ WIP — outline only |
| 14 | [Licensing — eval to production](licensing.md) | 📝 Draft — prose stable; live captures + FDM 7.6 UI paths pending |
| 15 | [Troubleshooting](troubleshooting.md) | ⚠ WIP — outline plus specific 2026-07-11 gotchas from Ch 9/10 walkthrough |

Legend: **✅ complete** — ready to follow · **📝 draft** — real content with known-open items · **🚧 partial** — substantial content but explicit blocker · **⚠ WIP** — outline / placeholder only

## Feedback

Something wrong, unclear, or out of date? [Open an issue](https://github.com/Batman4NY/firewall-1210ce-build-guide/issues) or reach out via [github.com/Batman4NY](https://github.com/Batman4NY).
