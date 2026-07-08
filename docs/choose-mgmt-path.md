# Choosing your management path

!!! info "📝 Draft"
    Freshly written. Substance is stable — only the screen-grabs from the actual SCC onboarding and cdFMC migration flows are still to come. Anything marked `[verify against unit]` is generic knowledge that should be confirmed against your specific 1210CE.

!!! info "📍 You are here"
    ```json
    {
      "hostname": "fw1210ce",
      "ftd": "FTD 7.6.0 (Build 113)",
      "mgmt_ip": "192.168.1.161",
      "domain": "uppernyack.com",
      "layer": "Layer 1 (FDM standalone)",
      "show_managers": "Managed locally.",
      "console_paths": "USB-C /dev/ttyACM0 → ser2net :9000 (primary) + FTDI RJ45 (standby, silent while USB-C plugged)"
    }
    ```

    The box in front of you is a **working FDM standalone**. You can browse to `https://192.168.1.161/` right now and log in. Everything from here on is a choice about **who owns the config**: this box, this box + a cloud pane of glass, a cloud-only pane of glass, or a full FMC VM you host yourself.

This chapter is the interlude. Before we start pushing policy in [Ch 6 — FDM baseline](fdm-baseline.md), you should know which management model you're building toward — because the *thing you build in FDM this week* is either the final artifact (Layer 1), a bootstrap for cloud sync (Layer 2), or scaffolding you'll migrate off entirely (Layer 3 / Layer 4). Same policy, very different lifecycle.

## Acronym decoder

| Acronym | Expands to | What it actually is |
|---|---|---|
| **FTD** | Firepower Threat Defense | The image running on the box — the unified ASA + Firepower software. Not a manager. |
| **FDM** | Firepower Device Manager | On-box web UI (`https://<mgmt-ip>/`). Manages **one** FTD. Ships with the image. |
| **FMC** | Firepower / Firewall Management Center | The heavy multi-device manager. Physical appliance or VM. |
| **FMCv** | FMC virtual | The VM edition of FMC. Runs on your ESXi/KVM/Debian-KVM host. Same feature set as the appliance. |
| **cdFMC** | cloud-delivered FMC | Cisco-hosted FMC-as-a-service. Same UX as FMCv, no VM to run. Reached through SCC. |
| **SCC** | Security Cloud Control | Cisco's cloud management pane (formerly Cisco Defense Orchestrator / CDO). Front door to cdFMC and to FDM/ASA orchestration. |
| **FMT** | Firewall Migration Tool | Cisco's tool for moving config from FDM → cdFMC (or ASA → FTD, etc.). |

!!! warning "There is no vFDM"
    FDM is **on-box only**. There is no "virtual FDM" you can run somewhere. If you want a manager that lives off the box, your choices are **SCC** (cloud), **cdFMC** (cloud), or **FMCv** (your VM). Anyone telling you they run "vFDM" is either confused or means FMCv.

## The four layers

Think of management as four layers stacked on top of the same FTD image. You can stop at any layer. Moving up a layer is additive on the way up (you keep the layer below as a fallback for a while); moving back down is a factory reset.

!!! success "✅ Layer 0 — Console (always active)"
    Serial console via ConsolePi (USB-C `/dev/ttyACM0` → telnet :9000). Always available regardless of what manager owns the box. This is your break-glass path — if every other layer goes dark, you get here from an RJ45 or USB-C cable and a laptop.

!!! success "✅ Layer 1 — FDM standalone (CURRENT)"
    The box manages itself. `https://192.168.1.161/`, `admin` account, everything local. This is what you have right now, coming out of [Ch 4 — First boot](first-boot.md). Full FTD feature set is available, no cloud dependency, no license server round-trips beyond the 90-day eval or your Smart Account. The ceiling: one device, one pane, no orchestration.

!!! tip "⏭️ Layer 2 — FDM + SCC (NEXT)"
    Same FDM you're using now, plus the box **also** registers to **SCC**. Config still authored in FDM; SCC gives you the cloud inventory, out-of-band change tracking, template-based deployment across multiple devices, and a single audit trail. FDM remains the source of truth for this device's running config. This is the path we're taking next, in [Ch 9 — SCC onboarding](scc-onboarding.md).

!!! note "⏳ Layer 3 — cdFMC (queued after Layer 2)"
    Full manager migration. Use **FMT** to translate the FDM policy into a cdFMC-native policy, cut over, and retire FDM as the manager. From this point on, FDM is read-only on the box — every knob goes through cdFMC. Gets you full FMC feature depth (correlation rules, deep IPS tuning, multi-device policy inheritance) without running your own VM.

Layer 4 — **FMCv on your own Debian/KVM host** — is the "I want to own the whole stack" option. Same feature set as cdFMC, no Cisco cloud tenancy, but you now own patching, backups, and HA for the manager itself. Out of scope for this guide's plan of record, but the comparison matrix below covers it so you can see the tradeoff.

## Comparison matrix

<div class="matrix-wrap">
<table class="mgmt-matrix">
  <thead>
    <tr>
      <th class="col-q">Question</th>
      <th class="col-fdm">FDM only</th>
      <th class="col-fdm-scc">FDM + SCC</th>
      <th class="col-cdfmc">cdFMC (via SCC)</th>
      <th class="col-fmcv">FMCv on Debian</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td class="col-q"><strong>Who's the boss?</strong></td>
      <td class="col-fdm">The box.</td>
      <td class="col-fdm-scc">The box; SCC observes + templates.</td>
      <td class="col-cdfmc">cdFMC in Cisco's cloud.</td>
      <td class="col-fmcv">Your VM.</td>
    </tr>
    <tr>
      <td class="col-q"><strong>Where does config live?</strong></td>
      <td class="col-fdm">On-box only.</td>
      <td class="col-fdm-scc">On-box, mirrored + versioned in SCC.</td>
      <td class="col-cdfmc">In cdFMC; box pulls it.</td>
      <td class="col-fmcv">In FMCv; box pulls it.</td>
    </tr>
    <tr>
      <td class="col-q"><strong>Cloud goes down — what breaks?</strong></td>
      <td class="col-fdm">Nothing. No cloud.</td>
      <td class="col-fdm-scc">Nothing traffic-wise; you lose SCC orchestration until it's back.</td>
      <td class="col-cdfmc">You lose the manager. Traffic keeps flowing on last-deployed policy; no new changes.</td>
      <td class="col-fmcv">Nothing. No cloud.</td>
    </tr>
    <tr>
      <td class="col-q"><strong>Local control retained?</strong></td>
      <td class="col-fdm">Full.</td>
      <td class="col-fdm-scc">Full — FDM is still authoritative.</td>
      <td class="col-cdfmc">Read-only on-box; FDM is disabled as a writer.</td>
      <td class="col-fmcv">Read-only on-box; FMCv is authoritative.</td>
    </tr>
    <tr>
      <td class="col-q"><strong>Cost to walk away?</strong></td>
      <td class="col-fdm">Zero.</td>
      <td class="col-fdm-scc">Un-register from SCC; box keeps running.</td>
      <td class="col-cdfmc">Factory reset + re-baseline in FDM.</td>
      <td class="col-fmcv">Un-register + factory reset.</td>
    </tr>
    <tr>
      <td class="col-q"><strong>Feature depth</strong></td>
      <td class="col-fdm">FDM subset of FTD (most, not all).</td>
      <td class="col-fdm-scc">Same as FDM.</td>
      <td class="col-cdfmc">Full FMC feature set.</td>
      <td class="col-fmcv">Full FMC feature set.</td>
    </tr>
    <tr>
      <td class="col-q"><strong>Matches what customers run?</strong></td>
      <td class="col-fdm">Small sites / branch.</td>
      <td class="col-fdm-scc">Multi-site with light central ops.</td>
      <td class="col-cdfmc">Cisco's push for new deployments.</td>
      <td class="col-fmcv">Regulated / air-gapped / self-hosted shops.</td>
    </tr>
  </tbody>
</table>
</div>
<p class="matrix-caption"><b>Highlighted column</b> = the path this lab is currently on (Layer 1 · FDM only). It moves as the journey progresses.</p>

## Plan of record — the journey

The lab is going to walk every layer above Layer 0, in order, so you (and any customer you're mirroring) see the full lifecycle end-to-end:

- [x] **Step 1 — FDM standalone.** ✅ **Done.** This is where you are right now. FDM is authoritative, no cloud attachments. Continues through [Ch 6 — FDM baseline](fdm-baseline.md), [Ch 7 — Security policies](security-policies.md), [Ch 8 — Talos intel](talos-intel.md).
- [ ] **Step 2 — Onboard to existing SCC tenant.** ⏭️ **Next.** Register `fw1210ce` to the Salient SCC tenant. Config stays on-box; SCC gets an inventory entry and a change log. See [Ch 9 — SCC onboarding](scc-onboarding.md).
- [ ] **Step 3 — FMT migration to cdFMC.** ⏳ **Queued after Step 2.** Run the Firewall Migration Tool, translate the FDM policy into cdFMC, cut over. FDM becomes read-only. See [Ch 10 — Managing via SCC](scc-managed.md).
- [ ] **Step 4 — Factory reset back to Layer 1.** ⏳ **Available anytime.** Documented so the reader knows the "walk away" path exists and what it costs. See [Ch 15 — Troubleshooting](troubleshooting.md).

You can skip Step 3 and stay on Step 2 forever — that's a legitimate end state, and probably the right one for a small-office deployment. Step 3 is only worth it if you need full FMC feature depth or you're standardizing on cdFMC across a fleet.

## Facts worth knowing before you commit

- **SCC and cdFMC are the same tenant.** cdFMC is provisioned *inside* an SCC tenant, not alongside it. If you already have SCC, you already have the front door to cdFMC — you just need to enable the cdFMC service on the tenant.
- **FMT is the only supported FDM → cdFMC path.** There is no "just click a checkbox" migration. FMT reads the FDM config, produces an FMC-shaped policy, and lets you review before pushing. Plan a maintenance window.
- **FDM → cdFMC is one-way in practice.** Cisco documents a rollback, but it's a factory reset + re-baseline in FDM. Treat Step 3 as a commit, not a toggle.
- **You cannot run FDM and cdFMC as co-authoritative.** Whichever registered last owns the box. If FDM writes, cdFMC's picture goes stale; if cdFMC writes, FDM is locked read-only. Pick one writer.
- **SCC (Step 2) does not lock FDM.** This is the friendly middle ground — you can un-register the box from SCC without touching the running config. It's cheap to try and cheap to back out of.
- **90-day evaluation licensing works for all four layers.** You do not need to have your Smart Account wired up to complete the journey; you just need it wired up before day 90 or the box's advanced features (URL filtering, malware, IPS updates) start declining renewal.
- **Console (Layer 0) never goes away.** Even at Step 3, USB-C `/dev/ttyACM0` still gets you a shell. Every layer above it can fail without stranding you — this is why we invested in ConsolePi first.
- **Factory reset is the escape hatch, and it's clean.** Two distinct motions worth knowing: `configure manager local` reverts management to on-box FDM but *keeps* the running config; `configure factory-default` (or the FDM factory-reset workflow) actually empties the config back to day-zero; a full ROMMON re-image is available if the config layer itself is corrupt. There is no "stuck in cdFMC" state — every state has a documented way back to Layer 1. Ch 15 covers each path with exact syntax.

## Prerequisites

- ✅ [Ch 4 — First boot and initial config](first-boot.md) complete. You have `fw1210ce` at `192.168.1.161` in FDM local-managed mode.
- ✅ Console access via ConsolePi established (see [Ch 3 — Console access](console-access.md)) — you'll want this reachable through every layer transition.

## Next

Continue building the Layer 1 foundation before adding cloud management on top:

- **Layer 1 build-out** → [Ch 6 — FDM baseline](fdm-baseline.md), then [Ch 7 — Security policies](security-policies.md), then [Ch 8 — Talos intel](talos-intel.md).
- **Step 2 — add SCC** → [Ch 9 — SCC onboarding](scc-onboarding.md) (after the Layer 1 baseline is stable).
- **Step 3 — migrate to cdFMC** → [Ch 10 — Managing via SCC](scc-managed.md) (after Step 2).
- **Step 4 — walk it back** → [Ch 15 — Troubleshooting](troubleshooting.md) (available anytime).
