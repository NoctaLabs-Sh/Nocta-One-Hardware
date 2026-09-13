# OpenVault — Design Review Actions

Review date: 2026-08-02 · KiCAD 10.0.3 · Board rev "V3" (45.00 × 69.00 mm, 4-layer)

Status after this review: **0 DRC errors, 9 warnings.** All findings below were verified against
`kicad-cli` output, the board geometry, and the nPM1300 / STM32WB5MMG / TPS62840 datasheets.

> Verification note: DRC must be run with `--refill-zones`. The on-disk zone fills were stale and
> produced 30 phantom violations and 71 phantom unconnected items. **Fill All Zones (Ctrl+B) and save.**
>
> ```
> /Applications/KiCad/KiCad.app/Contents/MacOS/kicad-cli pcb drc Hardware/OpenVault.kicad_pcb \
>   -o /tmp/drc.json --format json --severity-all --refill-zones --schematic-parity --units mm
> ```

---

## 1. CRITICAL — nPM1300 BUCK input decoupling

### The problem

IC1 `PVDD` (pin 4, the BUCK power input) is at **(146.35, 87.925)**. Every capacitor on the `VSYS`
net is clustered near U2, ~17 mm away:

| Cap | Value | Distance to IC1 PVDD |
|-----|-------|----------------------|
| C3  | 10 µF | **17.41 mm** |
| C4  | 10 µF | **17.83 mm** |
| C26 | 4.7 µF | **17.86 mm** |

**IC1 has no input capacitor near it at all.** This is BUCK2's input loop — the converter that makes
your main **+3V3 logic rail**. At roughly 1 nH/mm, a 17 mm out-and-back loop is 30+ nH; a 0.2 A
switching edge in 5 ns puts on the order of a volt of spike on PVDD. Expect ringing, EMI, and
degraded regulation.

Nordic, §9.3.4 PCB guidelines:

> "The BUCK supply voltage should be decoupled with high performance capacitors **as close as
> possible to the supply pins**. Long power supply lines on the PCB should be avoided. All device
> grounds, VDD connections, and VDD bypass capacitors **must be connected as close as possible to
> the device**."

Their reference BOM (Table 42) pairs a **10 µF** bulk with a **100 nF 0201** at the pin.

### Status: C3 moved ✅ (2026-08-02)

C3 relocated (142.25, 71.775) → **(146.35, 83.55)**. Distance to PVDD: **17.41 mm → 3.60 mm**.
Roughly a 5× cut in the feed-side inductance. It cannot easily go closer — L1 occupies the space
directly left of the QFN (correctly, the SW node must stay short) and VOUT2 (pin 32) sits above.
**3.6 mm is acceptable for the bulk cap.** Leave it.

Leave **C4** (10 µF bulk) and **C26** (4.7 µF) where they are — C26 at 2.40 mm is correctly serving
as the TPS62840's `CI`, which its datasheet specifies as 4.7 µF.

### Why a 10 µF alone cannot work here — f_BUCK = 3.6 MHz

Every MLCC is C in series with its ESL, so it only decouples **below** its self-resonant frequency
`SRF = 1/(2π√(L·C))`. Above SRF it is an inductor.

**C3 as installed:**

| Term | Value |
|---|---|
| 10 µF 0603 X5R at ~3.7 V DC bias (derating) | C_eff ≈ 5 µF |
| MLCC ESL | ~0.9 nH |
| 3.6 mm feed to PVDD (~1 nH/mm) | ~3.6 nH |
| Return path — nearest PVSS GND via is 2.4–2.8 mm | ~2.5 nH |
| **Total loop** | **≈ 7 nH** |

`SRF = 1/(2π√(7 nH × 5 µF))` ≈ **850 kHz**.

**The BUCK switches at 3.6 MHz** (datasheet §6.3.6, f_BUCK, PWM mode). C3 is operating ~4× above
its own resonance — at the switching frequency it is inductive. It does bulk energy storage and
low-frequency ripple, and contributes **nothing** at 3.6 MHz.

**A 100 nF 0402 at ~1.5 mm with its own ground via:** C_eff ≈ 85 nF (little bias derating),
ESL ~0.5 nH + mounting ~0.5 nH + 1.5 mm feed ~1.5 nH + local via return ~0.3 nH ≈ 2.8 nH
→ **SRF ≈ 10 MHz**. That covers 3.6 MHz and its second harmonic — exactly the band C3 cannot serve.

This is why moving C3 alone does not fix the problem. The 100 nF is not extra insurance; **it is the
part that does the actual work at the switching frequency.**

### Add — capacitors ✅ added to schematic 2026-08-02

| Ref | Value | Footprint | LCSC | Net |
|-----|-------|-----------|------|-----|
| **C29** | 100 nF | C_0402_1005Metric | C1525 | `VSYS` ↔ `GND` (decouples `PVDD`, pin 4) |
| **C30** | 100 nF | C_0402_1005Metric | C1525 | `+3V3` ↔ `GND` (decouples `VDDIO`, pin 12) |

Note: `PVDD` **is** the `VSYS` net and `VDDIO` **is** the `+3V3` net — there is no separate net to
attach to. The decoupling is created entirely by **PCB placement**, not by the schematic connection.

There was previously **no small-value HF capacitor anywhere on `VSYS`** — only 0603/0805 bulk.

### PCB placement (free-space scanned against real pad geometry, 0.15 mm clearance)

| Ref | Place at | Rot | Distance to pin |
|-----|----------|-----|-----------------|
| **C29** | **(145.00, 89.45)** | 0 | **2.04 mm** to PVDD (~1.6 mm copper-to-copper) |
| **C30** | **(148.15, 93.00)** | 0 | 2.41 mm to VDDIO |

C29: put the pad **nearest IC1** on `VSYS`; the far pad on `GND`. It also sits 1.45 mm from PVSS2
(pin 6), which is what you want for the return. Estimated loop ≈ 3.3 nH → **SRF ≈ 9.5 MHz**,
comfortably above the 3.6 MHz switching frequency.

**0201 alternative:** the 0.89 mm gap left of the QFN misses fitting a 0402 by 0.03 mm (needs
0.92 mm). An 0201 fits at **(145.55, 87.60) rot 90 → 0.86 mm** from PVDD (SRF ≈ 12.4 MHz), matching
Nordic's reference exactly (their C13 is a 100 nF **0201**). Better, but not decisive, and 0201 is
unpleasant to hand-rework. Take it only if matching the reference matters — change the footprint to
`C_0201_0603Metric`.

C30 at 2.41 mm is fine: VDDIO is a low-current logic supply for TWI/GPIO, not a switching node.

Board is **single-sided assembly** (no B.Cu components), so a bottom-side cap under pin 4 was
rejected — it would force a second assembly setup for one part.

### Add — vias (equally important, currently missing)

**1. Exposed-pad thermal vias — there are currently ZERO.**
The 3.6 × 3.6 mm EP is `AVSS`: both the device's analog ground reference *and* its only thermal path.
Today it reaches In1.Cu only by spreading laterally through the F.Cu pour to a via ~1.5 mm outside
the pad. That is poor for ground impedance and poor for heat during charging (thermal shutdown is
120 °C).

→ Add a **3 × 3 array of vias at ~1.0 mm pitch inside the EP**, using the existing 0.45 mm / 0.2 mm
stack so no new drill size and no upcharge. The small drill also limits solder wicking; if you want
belt-and-braces, JLCPCB offers epoxy-filled/capped via-in-pad as a paid option.

**2. Local ground vias at PVSS1 / PVSS2.**
Nearest GND via is **2.80 mm** from pin 2 and **2.37 mm** from pin 6, so the return current runs
that far sideways on F.Cu before reaching the plane — roughly +2.5 nH, which nearly doubles the loop.

→ Place one GND via **immediately at** each of pin 2 and pin 6. In1.Cu is only 0.2 mm below F.Cu, so
a via there is ~0.3 nH. **This single change benefits both C3 and the new 100 nF**, and is the
cheapest inductance you will ever buy back.

### Layout rules for the rework

- Keep the PVDD → cap → PVSS loop area minimal; cap on the same layer, no vias in the *feed* path.
- Ground the caps with their **own** via straight down to In1.Cu — never route ground laterally on F.Cu.
- L1 (2.2 µH, datasheet nominal) stays tight to `SW2` (pin 5) — it already is. Do not disturb it.

---

## 2. CRITICAL — Timekeeping architecture

### The constraint

Three verified facts combine badly:

1. STM32 `VBAT` (pin 15) is tied to **+3V3** — no independent backup domain supply.
2. nPM1300 Ship/Hibernate modes **"disconnect the battery from the system"** → VSYS drops → BUCK2 off → +3V3 off.
3. Therefore the RTC backup domain loses power and **wall-clock time is lost on every power-down.**

The nPM1300's own TIMER does not help: it is a 24-bit countdown *wake* timer, not a calendar, and at
the default 16 ms tick it maxes out around 3 days.

This matters because **TOTP (RFC 6238)** is dead without a trustworthy clock.

### Layer A — Stop using Ship mode as "off" *(this revision, one rework)*

Use **STM32 STOP2 with LSE + RTC running** as the idle/off state. Reserve nPM1300 Ship mode for
factory shipping and long-term storage only.

| State | Current |
|---|---|
| STM32 STOP2 + RTC | ~1–2 µA |
| nPM1300 quiescent (I_QBAT) | 800 nA |
| TPS62840 disabled | ~60 nA |
| **Total** | **~3 µA** |

**Blocker: `+3V3_EINK` is always on.** U2's `EN` (pin 4) is strapped to `VSYS`, so the e-ink panel's
`VDDIO`/`VCI` stay powered in STOP2. Panel standby is typically µA–mA and would swamp the 3 µA budget.

**Rework:** cut `EN` from `VSYS`; drive it from a free STM32 GPIO. Free, unrouted pads on the board
include PA0(3), PA4(55), PC0(12), PC1(9), PC2(8), PC3(7), PD0(33), PD1(34), PE0(79), PE1(64).

Safe by construction: on MCU reset the GPIO goes high-Z and the TPS62840's internal 450 kΩ pulldown
disables the rail. E-ink retains its image with no power, so dropping the rail costs nothing visually.

### Layer B — Back up the VBAT domain *(next revision)*

Survives battery depletion and cell swaps:

- Cut pin 15 from +3V3
- Feed via a Schottky from +3V3, with a supercap to GND

Sizing: `t = C·ΔV/I`. VBAT domain draws ~0.5–1 µA; usable swing 3.3 V → 1.55 V.
**0.22 F ≈ 4–9 days**; **1 F ≈ weeks**. Measure actual VBAT current at bring-up and size from that.

### Layer C — Calibration and resync *(firmware, required regardless)*

A ±20 ppm LSE drifts **1.7 s/day, 52 s/month**. TOTP's ±30 s window breaks in **2–4 weeks**.

- **`RTC_CALR` smooth calibration** (±488 ppm range, 0.954 ppm steps). Measure drift against host time
  over a known interval, store the trim in flash, apply at boot. Gets to a few ppm — months between resyncs.
- **Resync on every USB attach.** Use `RTC_ISR.INITS` to detect a cold backup domain; mark TOTP
  unavailable until resynced.
- **Enforce monotonicity.** A malicious host setting the clock forward harvests future TOTP codes.
  Never accept time going backwards; surface large jumps to the user.
- **Verify the "32.774 kHz" figure** in the module datasheet (§3.3) empirically — almost certainly a
  typo for 32.768, but taken literally it is +183 ppm ≈ 16 s/day. A 24 h run against host time both
  confirms it and yields the calibration constant.

### Do you need a TCXO RTC?

Only if TOTP must work standalone for months with no host. A DS3231M (±5 ppm ≈ 13 s/month, own VBAT
pin) would do it. **Recommendation: skip it** — a password manager sees a host regularly, and A+B+C
covers the requirement without another part or attack surface.

---

## 3. Other decoupling placement fixes

All are *moves* of existing parts unless noted.

| Ref | Value | Currently | Target | Why |
|-----|-------|-----------|--------|-----|
| **C6** | 2.2 µF | 8.0 mm from IC1 `VBAT`(19) | **< 2 mm** | Correct value per Nordic ref BOM, wrong place |
| **C2** | 1 µF | 15.1 mm from IC1 `VBUS`(21) | **< 2 mm** | USB input decoupling |
| **C25** | 100 nF | 9.16 mm from U3 `VCC`(8) | **< 2 mm** | SPI flash at 50–100 MHz needs local HF decoupling |
| C5 / C1 | 10 µF | 5.1 / 6.3 mm from `VOUT2`(32) | ≤ 3 mm | BUCK2 output; less critical than input |

Already good — **do not move**: C1 → `VDDIO` 1.78 mm · IC2 +3V3 1.75 mm · IC3 +3V3 1.18 mm.

---

## 4. Schematic fixes

| Item | Action |
|------|--------|
| **VOUT1 (pin 1) tied to VSYS** | **Leave unconnected.** BUCK1 is correctly disabled (VSET1 grounded = "0 V OFF" per Table 18), but active discharge is *forced* on power-cycle reset — 2 kΩ ≈ 2 mA parasitic load on VSYS |
| ~~C15 / C18 pin 2~~ | **RETRACTED — not a real defect.** KiCAD's ERC is **nondeterministic on this schematic**: three runs on an identical file gave 35 / 33 / 35 violations, with `pin_not_connected` varying between 1 and 3 and naming different caps each time (C15/C18/C20, then C16/C19/C20, then only C15). The netlist consistently shows all of them connected. **Never gate a decision on a single ERC run — run it several times, or trust the netlist.** Worth a brief visual check of that cap cluster, but low priority |
| **U2 symbol pin types** | All pins are typed *Unspecified* — this alone generates 12 spurious ERC warnings. Set correct electrical types |
| **VBAT `power_pin_not_driven`** | Add a PWR_FLAG (battery connector pins are passive) |
| **`PVSS2` label on GND** | Drop the redundant label |
| **TP1–TP8** | No footprints assigned, not on board. Assign, or tick *Exclude from board* |

---

## 5. PCB / DRC cleanup (9 warnings)

| Item | Count | Action |
|------|-------|--------|
| Clearance violations, 0.09–0.14 mm vs 0.15 mm netclass | 9 | Reroute. Clusters at (129.6–132.8, 98.0–99.6) and (139.1–142.1, 93.3–93.9) |
| `missing_courtyard` | 6 | Add F.CrtYd to L1, L3, U2, J03, J4, SW2 |
| `lib_footprint_mismatch` | 2 | IC2 and SW2 — both expected, see §6 |
| `lib_footprint_issues` | 1 | `SamacSys_Parts` not in `fp-lib-table`; move IC1's footprint into `OpenVaultFootprints.pretty` |
| `fp-lib-table` dead entry | 1 | Delete `.pretty` → `C:/Users/byte/Downloads/.pretty.pretty` |

---

## 6. Deliberate deviations — do NOT "fix"

**IC2: 17 LGA lands removed on purpose.** Board copy has 69 pads, library has 86. Missing:
`1, 2, 42, 43, 44, 56, 61, 62, 63, 72–78, 83` (plus IC1 pad 8 / GPIO1). All are unused GPIO carrying
no net — verified against netlist and datasheet. Ten sites have copper routed beneath: `SECURE_CSN`
(42, 43), `DISPLAY_CSN` (72–74), `MCU_SPI_MOSI` (75), `MCU_SPI_SCK` via (76), `GND` (61, 62, 77).

Safe at fab and assembly — no pad means no stencil aperture, so no solder is deposited there.

- **Never run Tools → Update Footprints from Library on IC2.** It restores all 86 pads while the
  traces stay, shorting unused GPIO to the SPI chip-selects.
- The permanent `lib_footprint_mismatch` on IC2 records this state.
- DRC is blind to this region forever. An F.Cu keepout over those land sites would make it enforceable.

**SW2 mounting holes** were converted `thru_hole` → `np_thru_hole` during this review (they had
drill 0.8 mm = pad 0.8 mm, i.e. zero annular ring). That cleared 4 errors. The resulting
`lib_footprint_mismatch` on SW2 is expected until the fix is pushed back to the library.

**IC3 footprint clearance override of 0.19 mm** was removed — it exceeded the TROPIC01's own
0.15 mm pad-to-EP gap, so the part could never satisfy its own rule.

---

## 7. Verified correct — leave alone

| Block | Evidence |
|-------|----------|
| **Antenna** | Planes cut back to Y≈55.18: **3.5 mm** from board edge, **2.7 mm** into module. Matches both numbers in ST Figure 4 (1.3 mm edge offset / 2.5 mm clearance). `ANT_IN`+`RF_OUT` grounded and `ANT_NC` soldered to an unconnected pad — all three explicitly required by §6.1 |
| **USB** | `USBD±` skew **0.000 mm**; `P_USBD±` 3.0 mm (USB 2.0 budget ≈ 15 mm); **zero vias** — no impedance discontinuities. USBLC6 orientation correct (D− 1→6, D+ 3→4) |
| **BUCK2** | VSET2 = 330 kΩ → 3.3 V (Table 19); L1 = 2.2 µH = datasheet nominal |
| **TPS62840** | R4 = 267 kΩ → 3.3 V (Table 1, E96 nominal, band 256.32–277.68 kΩ). MODE/STOP terminated, EN valid. VSET parasitic C ≈ **0.15 pF** vs 100 pF limit (~228× margin), 0.75 mm from SW node |
| **E-ink booster** | Complete: L2 10 µH, Q1 NMOS, R3 0.47 Ω sense, R2 10 kΩ gate pulldown, D1/D2/D3 charge pump → PREVGH/PREVGL, all six panel rails decoupled |
| **MCU support** | BOOT0 = 10 kΩ to **GND**; NRST 100 nF; SWD on TC2030; all supply pins correct (pin 17 is `VDD`, the symbol's "VDDSMPS" label is a naming quirk) |
| **SHPHLD** | Button to GND; 50 kΩ **internal** pull-up (Table 33). Datasheet §8.7 literally describes "SHPHLD is connected to SW2" |
| **I²C / flash** | 4.7 kΩ pull-ups both lines; flash CS 10 kΩ pull-up; WP#/HOLD# tied high |

---

## 8. Firmware requirements

- **`RCC_BDCR_LSEDRV[1:0] = 10`** (medium-high drive) — required by module datasheet §3.3 for the
  integrated LSE. Miss it and the crystal may not start reliably.
- **`HSEGMC[2:0] = 0b011`**; do **not** modify `RCC_HSECR` — HSE is factory-tuned, loaded by hardware.
- No clock bypass mode — both crystals are internal to the SiP.
- Check `RTC_ISR.INITS` at boot to detect a cold backup domain.

---

## 9. Security checklist (hardware password manager)

- **RDP Level 2 permanently disables SWD**, and J5 (TC2030) is the only debug path. Once burned the
  device is unflashable — plan provisioning first.
- **Rollback protection:** an attacker can snapshot W25Q128, burn PIN attempts, and restore it. The
  monotonic try counter must live in TROPIC01 or STM32 internal flash — never solely on external flash.
- **Everything on W25Q128 needs AEAD** (AES-GCM / ChaCha20-Poly1305), not bare encryption.
- **PIN verification must happen inside TROPIC01**, with the secure element owning rate limiting.
- **Host-supplied time is attacker-controlled** — see §2 Layer C monotonicity rule.
- **BLE is remote attack surface** on a vault device — decide deliberately whether it ships enabled.
- Configure BOR so brownout during unlock resets cleanly rather than executing at marginal supply.

---

## 10. Suggested order

1. **This revision (rework):** ~~C3 move~~ ✅ · 100 nF at PVDD · 100 nF at VDDIO · 9 EP thermal vias · GND vias at PVSS1/PVSS2 · U2 `EN` → GPIO
2. **This revision (schematic):** VOUT1 disconnect · C15/C18 check · U2 pin types
3. **Firmware:** LSE drive setting · RTC calibration + resync + monotonicity
4. **Cleanup:** 9 clearance spots · courtyards · lib tables · test points
5. **Next spin:** VBAT backup supercap · remaining decoupling moves (C6, C2, C25) · F.Cu keepout over IC2's removed lands

---

## 11. Open items

- **E-ink panel datasheet not in repo** — rail voltages inferred from booster topology only.
  Driver family inferred as SSD16xx/UC8xxx-style from the GDR/RESE/PREVGH/PREVGL/BS pinout.
- **Battery spec / power budget undocumented** — cannot sanity-check charge settings or rail sizing.
- **`Hardware/.step` and `Hardware/.stl`** (4.4 MB / 3.5 MB) are committed files with empty basenames
  from a bad 3D export, referenced by nothing.
