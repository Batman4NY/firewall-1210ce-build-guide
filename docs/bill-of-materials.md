# Bill of materials

!!! info "📝 Draft"
    Baseline BOM below is complete for the initial build. Rack accessories, cabling, and downstream switch recommendations are still being iterated.

## The firewall

- **1 × Cisco Secure Firewall 1210CE** — the **ASA-image** SKU is `CSF1210CE-ASA-K9`; the **FTD-image** SKU should be pulled from the current Cisco price book for the software train you want `[confirm exact SKU]`
    - The 1200 Series uses the **CSF** prefix; the legacy **FPR** prefix belongs to the Firepower 1000 / 2100 / 3100 / 4100 / 9300 families
    - 1U desktop or 1U rack form factor
    - Ships with an image (ASA or FTD) pre-loaded depending on the SKU
    - Includes rack ears

!!! tip "Definitive SKU: ask the box"
    The authoritative SKU for a unit already in hand is what the firewall itself reports. Once you've got console access wired up (see [Chapter 4 — First boot and initial config](first-boot.md)), `show inventory` at the FTD converged CLI prints the PID Cisco assigned to your specific unit — that beats any price-book lookup.

    Expected output on a 1210CE:

    ```
    > show inventory
    Name: "module 0", DESCR: "Secure Firewall 1210 Compact Appliance"
    PID: CSF-1210CE        , VID: V01     , SN: FJC3xxxxxxx
    ```

    - **PID** `CSF-1210CE` — the hardware base identifier (`CSF` = **C**isco **S**ecure **F**irewall). This is what shows on the box; the *orderable* SKU (`CSF1210CE-ASA-K9`, `CSF1210CE-*-K9` for the FTD image) is the bundled hardware + image + license combination and lives in the price book, not on the unit.
    - **VID** — hardware revision (V01 on the current shipping build).
    - **SN** — your unit's serial number. You'll need it for Smart Licensing registration and any RMA.

    A companion `show version` on the same converged CLI prints the running FTD train — this guide is written and tested against **FTD 7.6.0 (Build 113)**:

    ```
    > show version
    --------------------[ <hostname> ]--------------------
    Model                     : Cisco Secure Firewall 1210CE Threat Defense (86) Version 7.6.0 (Build 113)
    LSP version               : lsp-rel-YYYYMMDD-nnnn
    VDB version               : 392
    ```

**Sourcing:** Cisco Employee Program (if you're a Cisco employee), Cisco Partner allocation, or via a Cisco Reseller. Full evaluation licensing (all features, 90 days) is available on any 1210CE via [Cisco Smart Software Manager](https://software.cisco.com/software/csws/ws/platform/home).

## Serial console access

- **1 × ConsolePi** (Raspberry Pi 3B+ or Pi 4)
    - See the [ConsolePi Build Guide](https://batman4ny.github.io/consolepi-build-guide/) for the build recipe
- **1 × USB-A to USB-C data cable**
    - The 1210CE's console port is **USB-C native CDC-ACM** — no USB-serial adapter needed
    - Any working USB-C data cable (not charge-only) works
    - See [Console access chapter](console-access.md) for details

## Networking

You'll need at minimum:

- **Cat 6 patch cables** (short, in various lengths — 1 ft, 3 ft, 6 ft as needed)
    - For interconnecting FW / switch / uplink devices
- **Downstream managed switch** for the inside network
    - The 1210CE has limited port count; downstream switch handles the LAN
    - Any managed Cisco Catalyst / Meraki / third-party switch works
    - VLAN support is helpful for segmentation
- **Uplink to the internet**
    - Typically the WAN side of your home router (cable modem, fiber, etc.)
    - Either bridge the ISP box to the 1210 outside interface, or plug into a LAN port and let the 1210 be a downstream L3 device

## Rack / power

- **Rack space:** 1U (rack-mount kit included with the 1210CE)
- **Power:** 100-240V AC, standard C13 IEC cable (ships with the unit)
- **Cooling:** the 1210CE is passively-audible under load — a fan you'll hear. Plan rack placement accordingly.

## Optional but useful

- **Cisco UCS C220 M8** (or equivalent) in the same rack for compute workloads
    - See the [SE Lab roadmap](https://batman4ny.github.io/roadmap/) for how this fits the Secure AI Factory story
- **Smart PDU** for remote power cycling (once you've locked yourself out of the FW at least once, you'll appreciate it)
- **Rack shelf** for the ConsolePi if you're rack-mounted
- **USB-C angled adapter** if the cable geometry from ConsolePi to FW is awkward

## What you do NOT need

- **A separate FMC (Firepower Management Center).** The 1210 is fine to run on-box via FDM for lab use, and centrally via Security Cloud Control once onboarded. FMC is optional and mostly relevant at enterprise scale.
- **Additional interface modules.** The 1210 has enough onboard ports for a lab.
- **A separate console server.** ConsolePi covers that (that's what the [companion guide](https://batman4ny.github.io/consolepi-build-guide/) is for).

## Next

Head to [Console access via ConsolePi](console-access.md) to establish serial console before you power on the FW for the first time.
