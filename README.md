# Cisco Secure Firewall 1210CE Build Guide

> [!IMPORTANT]
> **Personal project — not Cisco-official.**
> This is a home lab build guide by a Cisco employee on their own time. Not affiliated with, endorsed by, sponsored by, or representing Cisco Systems, Inc. Product references are to publicly available Cisco offerings.

> [!WARNING]
> **Work in progress.** Chapters marked ⚠ WIP or 📝 DRAFT below are actively being written as I build out the lab. Feedback and pull requests welcome.

A build guide for setting up the **Cisco Secure Firewall 1210CE** as a home-lab node — from unboxing through **FTD 7.6** baseline, **Security Cloud Control** onboarding, and integration with **Talos**, **ThousandEyes**, **Duo**, and **Umbrella**.

**Live guide:** https://batman4ny.github.io/firewall-1210ce-build-guide/

## What you end up with

A production-grade Cisco Secure Firewall running FTD 7.6, managed both locally via **FDM** (Firepower Device Manager) and centrally via **Security Cloud Control (SCC)**, with the surrounding Cisco security portfolio wired in:

- **Talos intel** driving IPS + URL filtering
- **Duo MFA** on admin access to FDM and SCC
- **Umbrella** SASE tunnel from the firewall for DNS-layer filtering
- **ThousandEyes** local agent + FW-side visibility
- **XDR** correlation across the platform (post-onboarding)

## Who this is for

- **Cisco SEs and network engineers** who want a working Secure Firewall home lab as a demo bench
- **Anyone** who's obtained a 1210CE (via Cisco Employee Program, partner allocation, or purchase) and wants to build out a full-stack demo
- **Practitioners** who prefer a step-by-step walk-through over "figure it out from the FMC guide"

## What this is NOT

- **Not a substitute for the [Cisco Secure Firewall 1200 Series Getting Started Guide](https://www.cisco.com/c/en/us/support/security/secure-firewall-1200-series/products-installation-and-configuration-guides-list.html).** This is opinionated on top of official docs.
- **Not a production hardening guide** — this is lab-grade
- **Not affiliated with, endorsed by, or representing Cisco Systems, Inc.**

## Chapter status

| Chapter | Status |
|---|---|
| Overview | 📝 Draft |
| Bill of materials | 📝 Draft |
| Console access via ConsolePi | ✅ Complete |
| First boot and initial config | 📝 Draft |
| Choosing your management path | 📝 Draft |
| FDM baseline — interfaces + routing | ⚠ WIP |
| Security policies — access + IPS + URL | ⚠ WIP |
| Talos intel + updates | ⚠ WIP |
| Onboarding to Security Cloud Control | ⚠ WIP |
| Managing via SCC | ⚠ WIP |
| Duo MFA on FDM/SCC | ⚠ WIP |
| Umbrella SASE tunnel | ⚠ WIP |
| ThousandEyes on the FW | ⚠ WIP |
| Licensing — eval to production | 📝 Draft |
| Troubleshooting | ⚠ WIP |

## Start here

Head to [Overview](https://batman4ny.github.io/firewall-1210ce-build-guide/) and walk the guide top-to-bottom.

## Feedback

Open an issue or reach out via [github.com/Batman4NY](https://github.com/Batman4NY).
