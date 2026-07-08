# Duo MFA on FDM/SCC

!!! warning "⚠ WIP"
    Placeholder chapter with outline only. Content to be written as the lab is built. Feedback and PRs welcome.

Add Duo MFA to admin access on FDM and SCC. Free Duo tenant (Duo Free) supports the basic pattern.

## Duo tenant setup

- Sign up at [duo.com](https://duo.com/) — Duo Free is the 10-user free tier
- Create an **application** for FDM SAML integration
- Create an **application** for SCC SAML integration (or wire via Duo SSO)

**Fill in:** exact Duo admin panel walk-through, application types, callback URLs.

## FDM SAML config

- FDM → Objects → Certificates → Import Duo's SAML IdP metadata
- FDM → Users → Configure SAML for admin authentication
- Assign Duo group to FDM admin role

**Fill in:** exact click path, screenshots, XML/metadata exchange steps.

## SCC SAML config

- SCC has native SAML — integrate with your Duo tenant
- Optionally: Duo SSO in front for a full SSO story

**Fill in:** SCC-side SAML config, user mapping.

## Enroll users

- Register your admin user in Duo (mobile push, TOTP, etc.)
- Test end-to-end: SCC/FDM login → SAML redirect to Duo → Duo prompt → success

## Verify

- Log out of FDM/SCC
- Log back in — should redirect to Duo, prompt for MFA
- Bypass: use a break-glass local admin (documented, in a safe)

**Fill in:** break-glass procedure documentation.

## Next

Head to [Umbrella SASE tunnel](umbrella-integration.md) for DNS-layer filtering.
