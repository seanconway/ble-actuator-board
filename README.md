# BLE Smart-Lock Board: Hardware Design Spec

**Product:** a pocket-sized container with an IoT smart lock. A Hall switch senses the lid, and a small gearmotor drives a locking bar into place. The board talks to a phone app over Bluetooth LE, charges from USB-C, and has no buttons.
**Scope:** prototype PCB rev 1: schematic, off-board parts, layout, and how the board fits into the enclosure
**Status:** layout complete, ERC/DRC clean (KiCad 10), not yet ordered

| Path | Contents |
|---|---|
| `kicad/ble-actuator-board.kicad_pro` | KiCad 10 project: schematic, PCB, custom design rules (`.kicad_dru`) |
| `kicad/easyeda_lib.pretty`, `kicad/easyeda_lib.3dshapes` | Project-local footprints and 3D models (referenced through the project `fp-lib-table`) |
| `bom.csv` | Bill of materials: MPN, LCSC part number, footprint, package, ratings |
| `REVIEW_REQUEST.md` | Short overview and the questions for reviewers |

This spec is the reference the schematic and layout are checked against. It turns the product requirements into a buildable design: every part, value, footprint, and pin connection, plus layout rules and bring-up. Requirement IDs (`HW-*`, `EN-*`, `FW-*`, `OI-*`, `CM-*`) are defined in [Appendix A](#appendix-a-requirements-referenced) and [Appendix B](#appendix-b-open-items). Items marked **Proposed** are design choices made here that the requirements do not settle.

Datasheet short names: **PS** = nPM1300 Product Specification v1.3, **MDBT** = Raytac MDBT50Q spec Ver. L, **DRV8213** = TI SLVSGV9, **DRV5032** = TI DRV5032. Reference designators are the ones in `kicad/ble-actuator-board.kicad_sch`. Test pads carry signal-name references on the PCB (see §4.10).

**Design rule for this board:** no DNP parts and no 0 Ω resistors except where bench work is expected to need them. DNP is used only to keep JLC from placing a part at a footprint: U4 and M1 (off-board parts whose symbols carry their own wire-pad footprints) and the test pads TP1–TP16 (bare pads, nothing to place). No on-board component footprint is left unpopulated. The board has two fitted 0 Ω resistors: R2 (the VBAT current-measurement link) and R10 (the GAINSEL strap, lifted to change the gain range). The rework options this rule removed, and what each would have bought, are listed in §13.

---

## 1. Block diagram

```
                        ┌──────────────────────── PCB ────────────────────────┐
 USB-C (6-pin) ─ESD─────┤ VBUS ─► nPM1300                                     │
  CC1/CC2 ─ESD──────────┤          ├─ VBAT ◄── R2 ◄──────── J2 (JST-PH) ◄─────┼── 1S LiPo (2-wire)
                        │          ├─ NTC ◄── TH1 (on board)                  │
                        │          ├─ VSYS ─┬─ RGB LED anode                  │
                        │          │        │   cathodes ◄─ LED0/1/2 sinks    │
                        │          ├─ BUCK2 3V0 ─► MDBT50Q VDD/VDDH          │
                        │          │              DRV5032 VDD ────────────────┼── U4 pads → off-board lid sensor
                        │          ├─ BUCK1 VMOT ─► DRV8213 VM + VCC ─► motor ┼── M1 pads → gearmotor (lock)
                        │          ├─ TWI ◄────────► nRF (I²C master)         │
                        │          ├─ GPIO1 (PMIC_INT) ─► nRF                 │
                        │          └─ GPIO0 (NRESETOUT) ─► nRF nRESET         │
                        │                                                     │
                        │ MDBT50Q-P1MV2 ─ PWM ±──► piezo (differential)       │
                        │               ─ IN1/IN2 ─► DRV8213 (auto-sleep)    │
                        │               ◄ nFAULT/nSTALL, IPROPI (ADC)         │
                        │               ◄ HALL_SENSE (GPIO SENSE)             │
                        │               ─ SWD ─► Tag-Connect (J3)             │
                        └─────────────────────────────────────────────────────┘
```

BUCK2 supplies `3V0` because only VSET2 can select 3.0 V at power-up; BUCK1 supplies `VMOT` (§2, D2).

## 2. Datasheet constraints that shape the design

These findings from the datasheets override or refine the original requirements' assumptions. The schematic reflects all of them.

| # | Constraint | Source | Design consequence |
|---|---|---|---|
| D1 | The DRV8213 RTE package has **no nSLEEP pin**. It sleeps when IN1 = IN2 = 0 for t_AUTOSLEEP (0.75–1.05 ms), drawing ≤20 nA on VM and ≤12 nA on VCC. IN1/IN2 have internal 100 kΩ pull-downs. | DRV8213 §6, §8.5.2 | No external pull-downs on IN1/IN2. HW-3 is met by IN1 = IN2 = 0 plus BUCK1 (`VMOT`) off. |
| D2 | **Only VSET2 can select 3.0 V at power-up.** The VSET1 table tops out at 2.7 V (250–500 kΩ); VSET2 selects 3.0 V with 150 kΩ. The resistor must be ≤5% tolerance. | PS Tables 17, 18 | BUCK2 is the always-on `3V0` rail: R3 = 150 kΩ 1% on VSET2 boots it directly at 3.0 V, so no firmware voltage change is needed after a PMIC power cycle. BUCK1 is `VMOT`. |
| D3 | Grounding a VSET pin holds that buck **off at power-up** under software control. | PS §6.3.1 | VSET1 tied directly to GND, so BUCK1 (`VMOT`) stays off until firmware enables it. Matches FW-A1. |
| D4 | DRV8213 **VREF ≤ min(3.3 V, VM − 1.25 V) = 1.75 V** at VM = 3.0 V. With SMODE = Hi-Z the driver ignores VREF and uses 510 mV internally. | DRV8213 §7.3, §8.4.2.2 | VREF comes from a divider on VMOT, never a rail. SMODE is strapped Low so the divider is used. |
| D5 | **ITRIP target is ~40–65 mA, not ~200 mA.** INEED rates the 699:1 at 220 g·cm and the 136:1 at 140 g·cm, both at 60 mA, so ≤150 g·cm sits at or below ~60 mA. 60 mA is the boundary between DRV8213 gain ranges. | INEED motor spec, DRV8213 Table 8-3 | GAINSEL strapped High by fitted 0 Ω R10: 4900 µA/A, accurate over 10–60 mA, OCP ≥160 mA (below the buck's 200 mA rating). This range brackets the ≤150 g·cm point for both gearboxes. The Hi-Z range (60–350 mA) would start the 699:1 at ~220 g·cm, above HW-2. Lift R10 for Hi-Z only if calibration needs more than 60 mA. |
| D6 | nPM1300 watchdog → **NRESETOUT on a GPIO** resets the host without power-cycling BUCK2 (`3V0`). | PS §7.3.2 | GPIO0 wired to nRF nRESET. The RTC survives a watchdog reset. |
| D7 | The VBUS current limit starts at **100 mA** until firmware writes VBUSINILIM0 and TASK.UPDATE.ILIMSW, and reverts on every replug and reset. | PS §6.1.3 | Covered by FW-B6. Firmware reapplies it on each VBUS-insert event. |
| D8 | Module pins 3–14, 50, 52, 54, and 56–61 are **standard drive, low-frequency (≤10 kHz) only**. PWM and I²C do not count as low frequency. | MDBT §2.5–2.6 | The pin map (§5) keeps PWM, I²C, and motor lines off these pins. IPROPI (analog) is allowed there. |
| D9 | VBUSOUT needs a decoupling capacitor even when unused. Unused LDO/load-switch pins go to GND. | PS §6.1.5, Fig 63 | C8 on VBUSOUT; LSIN/LSOUT pins grounded. |
| D10 | The nPM1300 LED pins are 5 mA current sinks. | PS §6.5 | No LED series resistors. |
| D11 | nPM1300 GPIO0/GPIO1 boot as inputs with a 500 kΩ pull-down. On nRESET this draws ~6 µA against the nRF's reset pull-up until firmware makes it an output. | PS §6.6.1 | Firmware configures GPIO0 early in boot (§12). |
| D12 | The DRV8213 tri-level inputs (GAINSEL, IMODE, SMODE) are biased by internal 75 kΩ to VCC and 130 kΩ to GND, and the datasheet does not say whether that bias is switched off in sleep. A hard strap against it would draw ~23 µA (IMODE high) plus ~40 µA (SMODE low). The logic pins (IN1, IN2, GAINSEL, IMODE, SMODE, nFAULT, nSTALL) are rated −0.3 to 5.75 V **to GND, not to VCC**; only TINRUSH (≤ VCC) and IPROPI (≤ VM + 0.3 V) track a supply. The RTE package allows VCC = VM. | DRV8213 §7.1, Fig 8-15 | The whole driver runs from `VMOT`: VCC, IMODE, the GAINSEL strap (R10), and the nFAULT/nSTALL pull-up (R6). With BUCK1 off the driver is unpowered, so strap bias, R6, and R10 draw nothing in sleep. Driving IN1/IN2 while VMOT is off stays within the absolute maximum ratings. |

## 3. Power design

### 3.1 Rails (net names)

| Net | Source | Voltage | Loads | Notes |
|---|---|---|---|---|
| `VBUS` | USB-C (J1) | 5 V | nPM1300 VBUS only | Input limit 500 mA set by firmware (FW-B6, D7) |
| `VBAT` | Battery (J2) through R2 | 3.0–4.2 V | nPM1300 VBAT only | Nothing else may load VBAT (PS §8.2). Charge temperature comes from the board NTC TH1 (§4.2). |
| `VSYS` | nPM1300 pin 20 | ≈VBAT (1 A max) | PVDD, LED anode | Green and blue are fully lit only while VSYS exceeds their Vf plus 0.5 V sink headroom (§4.8) |
| `3V0` | BUCK2 (VOUT2) | 3.0 V (set by VSET2 at power-up) | nRF VDD/VDDH, VDDIO, lid sensor (through R12), I²C and reset pull-ups, Tag-Connect VCC | Always on outside ship mode (D2). Full-board plane on L3 (§7.1). |
| `VMOT` | BUCK1 (VOUT1) | 3.0 V (set by firmware) | DRV8213 VM and VCC, IMODE, GAINSEL strap (R10), nFAULT pull-up (R6), VREF divider | **Enabled only during actuation** (FW-A1, D3). Never feed the motor from VSYS (HW-1). |
| `GND` | — | 0 V | — | Solid plane on L2 (In1.Cu), §7.1 |

### 3.2 Requirements checklist

Checked against the schematic and layout; unchecked items need bench data.

- [ ] BUCK1 (`VMOT`) output capacitance sized for motor inrush. Confirm 200 mA headroom against measured stall current (OI-H1).
- [x] Every buck output capacitor (C10, C6) keeps ≥4 µF effective capacitance at its 3.0 V DC bias, with ESR ≤50 mΩ (PS Table 20). All 10 µF parts are Samsung CL10A106MA8NRNC (X5R 25 V 0603), Nordic's Configuration 1 part. C1 and C12 use the same part.
- [x] DRV8213 VM bulk (C12, 10 µF) and 0.1 µF bypass (C11) placed at the pin.
- [ ] ITRIP set by R7 (R_IPROPI) and the VREF divider so stall current stays below 200 mA **and** stall torque is ≤150 g·cm (HW-2). R7 is 0603 so it can be swapped after bench calibration (OI-H2).
- [x] VREF from a divider on VMOT (R9/R8), ≤1.75 V (D4). It draws nothing while BUCK1 is off.
- [x] DRV8213 sleep relies on IN1 = IN2 = 0 and its internal pull-downs (D1). No external parts.
- [x] DRV8213 VCC, IMODE, R10, and R6 on `VMOT`, so the driver is unpowered whenever BUCK1 is off (D12). SMODE tied to GND.
- [x] nPM1300 GPIO0 wired directly to nRF nRESET (D6).
- [x] Receptacle CC1/CC2 wired directly to nPM1300 CC1/CC2. No discrete pull-downs; the internal Rd presents the sink (HW-11). A 5.1 kΩ in parallel would put the combined Rd (~2.55 kΩ) out of spec.
- [x] SHPHLD routed to a test pad only. It has an internal 50 kΩ pull-up (HW-6).
- [x] nPM1300 GPIO1 routed to an nRF GPIO as the PMIC interrupt (HW-12).
- [ ] Board NTC TH1 (10 kΩ) from the nPM1300 NTC pin to GND with no capacitor >100 pF on it (PS §6.2.4) ✓. **Placement does not yet follow §4.2:** TH1 sits mid-board about 3.5 mm from U2's body, not at the board edge by the battery. Firmware sets 10 kΩ and TH1's β in ADCNTCRSEL.

### 3.3 Sleep current budget (target ≤10 µA average, FW-B4)

| Contributor | Estimate | Source |
|---|---|---|
| nRF52840 System ON, RTC + LFRC, RAM retention | ~2–3 µA | nRF52840 PS |
| nPM1300 quiescent (BUCK2 on, hysteretic mode) | ~1–2 µA (check PS) | nPM1300 PS |
| DRV5032FB | ~0.69 µA typ at 3 V (1.8 µA max; the 0.54 µA figure is at 1.8 V) | DRV5032 PS |
| DRV8213 in sleep | ~0 µA (unpowered: VM and VCC both on `VMOT`) | HW-3, D12 |
| Resistor leakage (pull-ups, VREF divider) | ~0 µA (see §11, sleep-leak audit) | Design |
| LFRC calibration (periodic) | TBD | OI-F10 |
| Leakage (ESD, LED) | TBD | Measure |
| **Total** | **TBD (OI-F7)** | PPK2 on first build, across R2 |

Advertising is budgeted separately (OI-F5).

## 4. Schematic, by subsystem

Passive defaults: `Capacitor_SMD:C_0402_1005Metric` / `Resistor_SMD:R_0402_1005Metric`. **0603** (`C_0603_1608Metric` / `R_0603_1608Metric`) is used for the 10 µF, 2.2 µF, and 1 µF capacitors, for R2, R3, R7, and R10, and for TH1. Every footprint is listed in `bom.csv`.

### 4.1 USB-C input and ESD (J1, D1–D3, C3)

| Ref | Value / part | Footprint | Connections |
|---|---|---|---|
| J1 | **Same Sky UJC-H-G-SMT-2-P6-TR** (LCSC C45343229): USB-C 6-pin power-only, horizontal top-mount. 5 A / 30 V, 10,000 mating cycles. Four through-hole shell legs in plated slots carry the plug load, so the board needs a through-hole solder step for them. | KiCad stock `Connector_USB:USB_C_Receptacle_GCT_USB4125-xx-x-0190_6P_TopMnt_Horizontal`. Checked against the Same Sky recommended layout (datasheet p.3): same pad row (pitches 1.00 / 3.04 / 5.50 mm, widths 0.70 / 0.76 / 0.80 mm, 1.20 mm tall), same four 0.60 × 1.20 mm slots in 1.10 × 1.70 mm pads at 8.64 × 3.80 mm, same 0.68 mm pad offset from the front slots. The 3D model is the GCT part. The receptacle mouth overhangs the board edge by ~0.6 mm. | A9/B9 VBUS → `VBUS`. A12/B12 GND and SH → GND. A5 CC1 → U2.23. B5 CC2 → U2.24. |
| D1 | **LRC LTVS16H12T5G** (LCSC C172439): unidirectional TVS. V_RWM 12 V, V_BR 12.5 V min, V_C 16 V max at 10 A and 23 V max at 60 A (8/20 µs), ±30 kV IEC 61000-4-2. Requirement: unidirectional; V_BR well above the 5.5 V VBUS maximum; clamp < 22 V (nPM1300 VBUS absolute maximum) at the ~30 A peak of an IEC 61000-4-2 8 kV contact strike. This part gives ~19 V at 30 A (datasheet Fig. 3). | Custom `imports:DFN1610-2L_L1.6-W1.0-P1.22` (below) | Pad 1 (cathode) → `VBUS`, pad 2 (anode) → GND. |
| D2, D3 | **onsemi ESD9B5.0ST5G** (LCSC C111566): bidirectional ESD, V_RWM 5.0 V, V_BR 5.8 V min, 15 pF, I_R ≤100 nA, IEC 61000-4-2 ±18 kV. Requirement: V_RWM ≥ 5.0 V and V_BR(min) > 5.5 V (the nPM1300 CC absolute maximum), bidirectional OK. | `Diode_SMD:D_SOD-923` (1.0 × 0.6 mm): machine placement only | D2 on CC1, D3 on CC2, each to GND |
| C3 | 1.0 µF X7R 50 V (Samsung CL10B105KB8NQNC). Requirement: X5R or better, ≥10 V. | 0603 | `VBUS` to GND at U2.21 (PS Table 39 "C1") |

**Why D1 is unidirectional:** VBUS only ever carries positive voltage, but ESD testing (IEC 61000-4-2) and hot-plug ringing hit it with both polarities. The nPM1300's VBUS pin is rated −0.3 V minimum (PS Table 4). A unidirectional TVS conducts forward on a negative strike and holds VBUS to a forward drop: 1.2 V max at 10 mA for D1, rising to a few volts at ESD current. A bidirectional TVS lets VBUS swing to its full negative breakdown (about −8 V) before clamping, which pushes the strike current through the PMIC's own pin-protection instead. Unidirectional parts also clamp lower on positive strikes than bidirectional parts of the same size. CC1/CC2 carry low-current signals and may use either type.

**Why a 12 V working voltage:** the nPM1300 disconnects VBUS above 5.5 V (VBUSOVP, PS §6.1.2) and survives up to 22 V. With V_BR ≥ 12.5 V, a faulty 9 V or 12 V charger only trips the PMIC's overvoltage cutoff, and the TVS does not conduct. A 6–7 V TVS would carry the charger's full current on the same fault until it failed. The trade-off is a higher clamp, which still stays under 22 V at ESD current.

**Why a 5 V working voltage is enough on CC:** with the nPM1300's internal Rd present, CC sits at about 2.1 V or less, even when the charger advertises 3 A. The ESD part must not conduct below the nPM1300's 5.5 V CC absolute maximum, which D2/D3 guarantee through V_BR ≥ 5.8 V. Their working voltage (5.0 V) only sets the leakage rating, which is ≤100 nA and flows only while a cable is attached.

**D1 footprint (DFN1610-2, LRC land pattern, datasheet p.4):** two pads 0.62 × 1.00 mm at 1.22 mm pitch (0.60 mm gap, 1.84 mm overall). Pad 1 = cathode. The part is leadless and needs reflow or hot air.

**Layout (as built)**
- D1, D2, and D3 sit 2–4 mm from the J1 pads, before the traces reach anything else.
- VBUS is routed at 0.5 mm (0.25 mm neck-down at pads), short and direct to U2.21.
- J1's shell slots are on the GND pour.

### 4.2 Battery and charge temperature sensing (J2, BT1, R2, C4, TH1)

| Ref | Value / part | Footprint | Connections |
|---|---|---|---|
| J2 | **JST S2B-PH-SM4-TB(LF)(SN)** (LCSC C295747): 2-pin JST-PH header, 2.0 mm pitch, SMD right-angle. Placed by JLC. Symbol `Device:Battery`, so the cell is drawn by J2 itself. | `Connector_JST:JST_PH_S2B-PH-SM4-TB_1x02-1MP_P2.00mm_Horizontal` | Pin 1 (+) → R2 → `VBAT`. Pin 2 → GND. |
| BT1 | Prototype cell: **Adafruit 1578**, 1S LiPo 500 mAh, 29 × 36 × 4.75 mm, PCM fitted (cuts out at 3.0 V), UN38.3 report available, JST-PH plug fitted. Final cell is open (OI-H8). | none | Not in the schematic; J2's symbol stands for it. Bought separately and plugged into J2. |
| R2 | 0 Ω | 0603 | Between J2.1 and `VBAT`. Lift it to insert the PPK2 for whole-system current. |
| C4 | 2.2 µF X5R 16 V (Samsung CL10A225KO8NNNC). X5R is rated to 85 °C, above the cell's charge window. | 0603 | `VBAT` to GND at U2.19 (PS Table 39 "C6") |
| TH1 | **Murata NCP18XH103F03RB**: 10 kΩ NTC, 1%, β(25/50) = 3380 K ±1% | 0603 | U2.18 (NTC) to GND. **On the board**, see placement below. |

**Why the NTC is on the board:** 3-wire cells with an integrated NTC are hard to source in prototype quantities. TH1 measures the temperature next to the cell, and the nPM1300 uses it to qualify charging (cold/cool/warm/hot thresholds, with reduced current in the cool and warm bands). Neither the safety standards nor the PMIC require the NTC to be inside the pack. What is required is that the cell stays within its specified charging temperature range under normal and single-fault conditions (IEC 62368-1 Annex M). This approach meets that as long as the rules below hold:

- **Thermal coupling:** place TH1 on the board edge under or beside the battery pocket, as close to the cell as the layout allows. Use a thermal pad or foam tape in the enclosure to press the cell against that area. No copper pour connects TH1 to the PMIC or bucks.
- **Distance from heat sources:** keep TH1 away from U2, L1, and L2. Charger dissipation that reaches TH1 makes it read warm, which stops charging early. That is safe, but it costs charge time.
- **Cold margin:** a board warms faster than a cell after the device is brought in from the cold, so TH1 can read warmer than the cell for the first few minutes. Set the firmware cold threshold to about 5 °C instead of 0 °C, and use the reduced-current cool band below about 10 °C (§12).
- **Charge rate:** keep the charge current at or below 0.5C so cell self-heating stays small compared with the thresholds.
- **Fault behavior:** an open TH1 reads as very cold and a shorted TH1 reads as very hot. Both fall outside the charge window, so charging stops. Confirm this on the bench (§9).
- **Secondary protection:** the cell's PCM (overcharge, over-discharge, overcurrent, short) is still required (OI-H8). The production cell must carry IEC 62133-2 and UN38.3 (§14). Adafruit lists a UN38.3 report for the 1578 but no IEC 62133-2 certificate; that is acceptable for bench prototypes only.

> **Current layout deviates from the first two rules.** TH1 is at (54.3, 56.0), mid-board and ~3.5 mm from the U2 body, ~12 mm from L1/L2, and ~13 mm from J2. The cell position relative to the board is set by the enclosure and is not final (OI-H8). See the review questions.

**Battery connector and hand-soldered leads:** the cell plugs into J2. The motor (M1) and lid sensor (U4) solder to wire pads after JLC assembly; their symbols carry the pad footprints and are marked DNP so JLC places nothing there. JST-PH cells are not polarity-standardized: vendors wire the plug both ways, so a cell from another source can fit J2 reversed and put −4.2 V on `VBAT`. Before the first connection of any cell, measure its plug with a meter against the J2 silkscreen (§9), and swap the crimps in the plug housing if they are reversed. The 2-pin plug cannot reach the lid-sensor pads, but a cell lead soldered there by mistake would put 4.2 V on `3V0`, above the nRF's 3.6 V maximum.

**Layout:** VBAT is routed at 0.5 mm (0.25 mm neck-down). The NTC trace stays away from SW1/SW2. No capacitor larger than 100 pF on NTC.

### 4.3 PMIC: nPM1300 (U2)

Values follow PS Configuration 1 (Table 39). **Footprint:** KiCad stock `Package_DFN_QFN:QFN-32-1EP_5x5mm_P0.5mm_EP3.6x3.6mm_ThermalVias`, not the Nordic `nordic-lib-kicad` footprint. Its exposed pad is 3.6 × 3.6 mm with a 3 × 3 array of 0.3 mm vias (0.5 mm annular pad) at 1.0 mm pitch to a 2.5 × 2.5 mm GND pad on B.Cu, and a 3 × 3 array of 0.97 mm paste windows at 1.2 mm pitch. The paste windows overlap the via locations, and the vias are tented on the bottom only by solder mask. Check the exposed pad and paste apertures against PS §9.2.1.

**Parts**

| Ref | Value | Footprint | Function |
|---|---|---|---|
| C15 | 10 µF X5R 25 V | 0603 | PVDD (4) to PVSS1 (2) |
| C16 | 10 µF X5R 25 V | 0603 | PVDD (4) to PVSS2 (6) |
| C7 | 10 µF X5R 25 V | 0603 | VSYS (20) to GND |
| C8 | 1.0 µF X7R 50 V (same part as C3) | 0603 | VBUSOUT (22) to GND (D9) |
| L1 | **Inpaq WIP201610S-2R2ML** (LCSC C964111): 2.2 µH ±20%, DCR 117 mΩ typ / 140 mΩ max, Isat 2.45 A min, Irms 2.0 A min, shielded. Requirement (PS Table 19): DCR ≤ 400 mΩ, Isat > 350 mA, Imax > 200 mA. | Custom `imports:L_Inpaq_WIP201610S_2.0x1.6mm` (Inpaq land pattern A = 1.6, B = 0.9, C = 2.0 mm) | SW1 (3) to `VMOT` |
| C10 | 10 µF X5R 25 V | 0603 | `VMOT` output cap, returns to PVSS1 |
| L2 | Same as L1 | Same as L1 | SW2 (5) to `3V0` |
| C6 | 10 µF X5R 25 V | 0603 | `3V0` output cap, returns to PVSS2 |
| R3 | 150 kΩ 1% (selects 3.0 V; ≤5% tolerance required) | 0603 | VSET2 (16) to GND |
| C9 | 100 nF X7R 16 V | 0402 | VDDIO (12) to GND, at the pin |
| R4, R5 | 10 kΩ | 0402 | SDA, SCL pull-ups to `3V0` |
| TP1 | Test pad | `TestPoint:TestPoint_Pad_D1.0mm` | SHPHLD (15) |

**Pin-by-pin**

| Pin | Name | Connects to |
|---|---|---|
| 1 | VOUT1 | `VMOT` (feedback; join at C10) |
| 2 | PVSS1 | GND (C15 and C10 return here) |
| 3 | SW1 | L1 → `VMOT` |
| 4 | PVDD | `VSYS`, C15, C16 |
| 5 | SW2 | L2 → `3V0` |
| 6 | PVSS2 | GND (C16 and C6 return here) |
| 7 | GPIO0 | `nRESET`, direct (D6) |
| 8 | GPIO1 | `PMIC_INT` → U1.39 |
| 9–11 | GPIO2–4 | No-connect flags |
| 12 | VDDIO | `3V0`, C9 |
| 13 / 14 | SDA / SCL | `SDA` / `SCL` → U1.20 / U1.21, R4 / R5 to `3V0`. Address 0x6B. |
| 15 | SHPHLD | TP1 only. No pull-up or capacitor. |
| 16 | VSET2 | R3 (150 kΩ) → GND. Never floating (PS §6.3.2). |
| 17 | VSET1 | GND, direct (< 100 Ω) |
| 18 | NTC | TH1 to GND |
| 19 | VBAT | `VBAT`, C4 |
| 20 | VSYS | `VSYS`, C7 |
| 21 | VBUS | `VBUS`, C3 |
| 22 | VBUSOUT | C8 only |
| 23 / 24 | CC1 / CC2 | J1.A5 / J1.B5 |
| 25 / 26 / 27 | LED0 / LED1 / LED2 | D4 blue / green / red cathodes (§4.8) |
| 28–31 | LSIN1, LSOUT1, LSIN2, LSOUT2 | GND (D9) |
| 32 | VOUT2 | `3V0` (feedback; join at C6) |
| 33 | AVSS / exposed pad | GND |

**Layout (critical for buck noise and efficiency)**
- Place C15 and C16 right at pin 4, returning to pins 2 and 6. These two loops matter most.
- Return C10 to pin 2 and C6 to pin 6 before joining the GND plane, with a GND via next to each PVSS pin.
- Keep SW1/SW2 nodes short and compact. No vias in SW nodes, and no other signals under the inductors.
- Exposed pad: 3 × 3 thermal-via array to the L2 GND plane (from the footprint).
- Keep SDA/SCL, NTC, and VSET away from the SW nodes (0.5 mm minimum, enforced by DRC).
- Follow Nordic's QFN reference layout (PS §9.3.5).

### 4.4 MCU module: MDBT50Q-P1MV2 (U1)

| Ref | Value | Footprint | Connections |
|---|---|---|---|
| U1 | Raytac MDBT50Q-P1MV2 (LCSC C5119772), nRF52840, PCB antenna | `RF_Module:Raytac_MDBT50Q` | — |
| C1 | 10 µF X5R 25 V | 0603 | Module VDD to GND (MDBT §8.3 "C11") |
| C5 | 100 nF X7R 16 V | 0402 | Module VDD to GND, closest to pins 28/30 |

**Power and fixed pins**
- Pins 28 (VDD) and 30 (VDDH) tied together → `3V0` (normal-voltage mode, MDBT §8.3).
- Pin 31 (DCCH) → no-connect. REG0 is unused in normal-voltage mode; REG1's DC/DC inductor is inside the module, so firmware can enable DCDCEN.
- Pin 32 (VBUS) → GND (USB disabled, MDBT §8.5). Pins 34/35 (D−/D+) → no-connect.
- All module GND pins (1, 2, 15, 33, 55) → GND.
- Pins 17/18 (P0.00/P0.01): unused, no-connect. Keeps the 32.768 kHz crystal option open (OI-F10).
- Pins 52/54 (P0.09/P0.10): unused, no-connect. They default to NFC and are low-frequency only.
- Signal pins per §5. Every other GPIO is no-connect.

**Layout (MDBT §2.3), as built**
- The module sits on the top board edge with the antenna end outward, away from the battery connector, motor pads, and sensor pads.
- A rule area spans the full board width from the top edge to 4.4 mm in (y = 35.0–39.4 mm) on all four copper layers and forbids copper pours. The rule itself allows tracks and vias, but none are routed inside it.
- Solid GND under the rest of the module.
- To do before ordering: send Raytac the layout PDF for a free antenna review.

### 4.5 Debug: Tag-Connect TC2030 (J3)

| Pin | Signal | Connects to |
|---|---|---|
| 1 | VCC | `3V0` (target voltage sense) |
| 2 | SWDIO | U1.51 |
| 3 | nRESET | `nRESET` (U1.40, P0.18), shared with R1, C2, and U2.7 |
| 4 | SWCLK | U1.53 |
| 5 | GND | GND |
| 6 | SWO | No-connect |

- R1 10 kΩ, `nRESET` to `3V0`. P0.18 has an internal pull-up, so R1 is redundant but harmless.
- C2 100 nF, `nRESET` to GND.
- Footprint: `Connector:Tag-Connect_TC2030-IDC-FP_2x03_P1.27mm_Vertical` (legged, 3 NPTH holes). Footprint only; no part is placed.

### 4.6 Motor driver: DRV8213RTER (U3) and motor (M1)

**Parts**

| Ref | Value | Footprint | Function |
|---|---|---|---|
| U3 | TI DRV8213RTER (LCSC C31829946) | Custom `imports:RTE0016C-MFG` (TI land pattern) | — |
| C11 | 100 nF X7R | 0402 | VM bypass, at pin 5 (DRV8213 Table 8-1) |
| C12 | 10 µF X5R 25 V | 0603 | VM bulk |
| C13 | 100 nF | 0402 | VCC (2) bypass to GND, at the pin. VCC is on `VMOT` (D12). |
| R6 | 10 kΩ | 0402 | nFAULT/nSTALL pull-up to `VMOT` (D12) |
| R7 | **5.1 kΩ 1%** to start (ITRIP ≈ 40 mA with R10 fitted) | **0603** | IPROPI (1) to GND. Swapped during calibration. |
| R9 / R8 | 200 kΩ / 100 kΩ 1% | 0402 | `VMOT` → R9 → VREF (16) → R8 → GND. VREF = 1.00 V. |
| C14 | 2.2 nF C0G 50 V 5%, **Murata GRM1555C1H222JA01D**, to start. C0G is required: the capacitor's tolerance enters the t_INRUSH budget directly (DRV8213 §9.2.1.3.1.1.1), and X7R would add ±10% tolerance plus ±15% temperature drift. | 0402 | TINRUSH (10) to GND. t_INRUSH = 6.5×10⁶ × C ≈ 14 ms. Range 0.8 nF–0.8 µF. Tune after OI-H1. |
| R10 | 0 Ω, **fitted** | 0603 (easier to lift than 0402) | GAINSEL (14) to `VMOT`, selecting High gain (D5, D12). Lift it to select Hi-Z. |
| M1 | INEED IND-GM-609-699, 6 mm planetary gearmotor, 699:1, 3.0 V rated (alt 136:1, OI-H14). Symbol `Motor:Motor_DC`, **DNP**: the symbol carries the motor's wire pads, and the motor is hand-soldered to them after assembly. | `Connector_Wire:SolderWire-0.1sqmm_1x02_P3.6mm_D0.4mm_OD1mm` (1.6 mm pads, 0.6 mm drill) | Pad 1 → OUT1, pad 2 → OUT2. The pads sit on the right board edge. INEED gives the leads as **20 ± 2 mm**; extend them with fine wire if the motor sits farther away. |

**Pin-by-pin**

| Pin | Name | Connects to |
|---|---|---|
| 1 | IPROPI | R7 to GND; `MOT_IPROPI` → U1.11 (P0.02/AIN0) |
| 2 | VCC | `VMOT`, C13 (D12) |
| 3 | nSTALL | Tied to pin 4 |
| 4 | nFAULT | `MOT_nFAULT` → R6 to `VMOT` → U1.37 (P0.13). nSTALL must see the pull-up at power-up to enable stall detection (DRV8213 §8.4.3); with R6 on `VMOT` it rises with VCC, so it always does. The line reads low whenever `VMOT` is off. |
| 5 | VM | `VMOT`, C11, C12 |
| 6 | OUT1 | M1 pad 1 (motor lead) |
| 7 | PGND | GND |
| 8 | OUT2 | M1 pad 2. **No capacitor across the motor.** It fights current blanking. |
| 9 | GND | GND |
| 10 | TINRUSH | C14 to GND |
| 11 | SMODE | **GND, direct** (Low) |
| 12 | IN2 | `MOT_IN2` → U1.26 (P1.09). No external pull-down. |
| 13 | IN1 | `MOT_IN1` → U1.25 (P1.08). No external pull-down. |
| 14 | GAINSEL | R10 (fitted 0 Ω) to `VMOT` = High. Lifted = Hi-Z. |
| 15 | IMODE | **`VMOT`, direct** (High) |
| 16 | VREF | R9/R8 divider midpoint |
| 17 | Thermal pad | GND |

**Strap settings (tri-level pins)**

| Pin | Setting | How | Why |
|---|---|---|---|
| GAINSEL | **High**: 4900 µA/A, 10–60 mA range, OCP ≥ 160 mA | R10 fitted | Keeps the first stall test below 150 g·cm with either gearbox, and OCP sits below the buck's 200 mA rating. Lift R10 to select **Hi-Z** (1050 µA/A, 60–350 mA, OCP ≥ 0.8 A) only if calibration needs more than 60 mA (D5). |
| IMODE | **High**: current regulation always on | Tied to `VMOT` (= VCC) | ITRIP caps torque at all times (HW-2) |
| SMODE | **Low**: on stall, latch outputs off and pull nSTALL low | Tied to GND | Hardware stall cutoff independent of firmware; uses the external VREF (D4). Firmware reads nSTALL as end of travel or jam. |

**ITRIP = V_VREF / (R7 × A_IPROPI)**, with V_VREF = 1.00 V:

| GAINSEL | R7 | ITRIP |
|---|---|---|
| High (4900 µA/A, R10 fitted) | 5.1 kΩ | 40 mA ← **start here** |
| High | 4.02 kΩ | 51 mA |
| High | 3.40 kΩ | 60 mA |
| Hi-Z (1050 µA/A, R10 lifted) | 15.8 kΩ | 60 mA |
| Hi-Z | 12.1 kΩ | 79 mA |
| Hi-Z | 9.53 kΩ | 100 mA |
| Hi-Z | 6.34 kΩ | 150 mA |

Raise ITRIP step by step during OI-H2 using a 1 cm arm and a scale. Stop at 150 g·cm and stay below 200 mA (the BUCK1 rating). For reference, INEED rates the 699:1 at 220 g·cm and the 136:1 at 140 g·cm, both at 60 mA. Because SMODE = Low makes ITRIP the stall threshold too, ITRIP must also sit clearly above the peak running current, or the driver will latch off mid-travel. V_IPROPI at ITRIP equals V_VREF (1.0 V), which is ≤ VM − 1.25 V, so the current mirror stays accurate and the nRF SAADC can read it (gain 1/6, 0.6 V reference).

**Layout (as built)**
- The BUCK1 → C12 → VM → OUT → motor → PGND → GND loop is kept tight, with C11 directly at pin 5 and C12 next to it.
- VM and OUT1/OUT2 are routed at 0.3 mm (`VMOT` up to 0.5 mm).
- Footprint `imports:RTE0016C-MFG` (TI's recommended land pattern). The 1.68 mm thermal pad has four 0.3 mm vias (0.56 mm pads) to GND, tented by solder mask on the bottom, with four 0.42 mm corner paste windows kept clear of the vias so solder does not wick down them. `RTE0016C-IPC_A/B/C` are in the project library as alternatives for hand assembly.
- R7 is next to pin 1 with its GND straight into the plane. `MOT_IPROPI` runs from U3 across the board to its test pad (`IPRO`) near the left edge, on L1 and L4, away from OUTx, SW1, and SW2.
- R10 can be lifted with an iron without disturbing other parts.

> **Why the driver runs from `VMOT` (D12):** the datasheet does not say whether the internal 75 kΩ/130 kΩ tri-level bias is switched off in sleep. If it stays on, straps against it would draw ~60 µA, six times the sleep budget. With VCC and every strap on `VMOT`, the driver is unpowered whenever BUCK1 is off, so the question no longer affects sleep current. VCC and VM ramp together over BUCK1's ~1.2 ms soft start, well inside the 2 V/µs slew limit.
>
> **nRF pin margin:** `MOT_nFAULT` is pulled to `VMOT` but read by an nRF pin powered from `3V0`. Both bucks are 3.0 V ±5%, so in the worst case the pin sits ~0.3 V above the nRF's VDD, at the edge of its VDD + 0.3 V absolute maximum. R6 limits any injection to microamps, and the clamp diode does not conduct below ~0.5 V, so this is safe. Do not set `VMOT` above `3V0`.

### 4.7 Lid sensor: DRV5032FB (U4, off-board)

| Ref | Value / part | Footprint | Connections |
|---|---|---|---|
| U4 | **DRV5032FBLPGM** (LCSC C2862715; the FA variant is the alternate, OI-H3), TO-92, omnipolar, push-pull, 5 Hz sampling. **DNP**: the symbol carries the sensor's 3 wire pads, and the sensor is hand-soldered to them on short leads after assembly. Kept in the BOM, not placed by JLC. | `Connector_Wire:SolderWire-0.1sqmm_1x03_P3.6mm_D0.4mm_OD1mm` | Pad 1 = VCC → R12 → `3V0`. Pad 2 = GND. Pad 3 = OUT → R11. |
| R12 | 100 Ω | 0402 | `3V0` → U4 pad 1, next to pad 1. Keeps a short or ESD strike on the off-board wire from collapsing or hitting `3V0`. Worst-case drop is 0.27 V at the sensor's 2.7 mA peak, leaving ≥2.7 V at U4, and C17 at the pads supplies the 40 µs sampling pulses. |
| R11 | 1 kΩ | 0402 | U4 pad 3 → `HALL_SENSE` → U1.36 (P0.14), next to pad 3. Limits ESD and fault current from the off-board wire. Draws no DC current. |
| C17 | 100 nF X7R 16 V | 0402 | **On the board**, between the pad 1 supply (after R12) and pad 2 (GND) (HW-4). Keep the off-board wire short so this cap still decouples the sensor. |

No pull-up or pull-down on `HALL_SENSE`: the sensor output is push-pull (HW-4).

**Off-board assembly (HW-4, HW-10)**
- Mount U4 in the enclosure near the lid magnet, flat branded face toward the magnet.
- Short twisted 3-conductor wire or a small flex to the U4 pads (right board edge, below the motor pads).
- The magnet plan (OI-H4) must make it impossible for the lid-retention magnets alone to hold the sensor "closed" while the lid is open. Verify with the TI magnetic sense simulator, then on the bench.

### 4.8 RGB LED: TUOZHAN P4-1204RGBWS2-1.5T-A (D4)

- **Common-anode RGB, side-emitting**, 3.2 × 1.0 × 1.5 mm, white diffused lens, 120° viewing angle (LCSC C2827295). It replaces the top-view Nationstar NH-B1515RGBA-GF from earlier revisions, whose footprint is still in the project library but unused.
- **Pinout (datasheet p.2): 1 = G−, 2 = B−, 3 = R−, 4 = anode (+).** The schematic's `LED_ABGR` symbol is mapped to match: A = 4, GK = 1, BK = 2, RK = 3. The symbol's Value field still reads `NH-B1515RGBA-GF` and should be updated.
- **Footprint:** custom `imports:LED-SMD_4P-L3.2-W1.5_P4-1204RGBWS2-1.5T-A`, four pads in one row (order 3-1-2-4): inner pads 0.5 × 0.4 mm at ±0.42 mm, outer pads 0.6 × 0.4 mm at ±1.30 mm. A silkscreen dot marks pin 4 (anode). The silkscreen dome shows the lens facing the footprint's −Y side.
- **Placement:** D4 is 1.2 mm from the bottom board edge, rotated 0°, so the lens faces **into** the board (toward U2), not out of the edge. The enclosure light path must suit this (open item).
- **Channels:** anode → `VSYS`. Blue cathode → LED0 (25), green → LED1 (26), red → LED2 (27). Firmware maps the channels.
- Moisture-sensitive packaging (sealed bag with desiccant). Reflow at most twice, within 12 h. JLC handles baking for assembly.

| Color | Vf min / max at 20 mA (datasheet) | Intensity at 20 mA | Lowest VSYS for full drive (Vf max + 0.5 V sink headroom, PS Table 24) |
|---|---|---|---|
| Red | 1.8 / 2.4 V | 100–200 mcd | ~2.9 V worst case: visible across the whole battery range |
| Green | 2.8 / 3.4 V | 500–700 mcd | ~3.9 V worst case at 20 mA; lower at the 5 mA sink current |
| Blue | 2.8 / 3.4 V | 120–250 mcd | ~3.9 V worst case at 20 mA; lower at the 5 mA sink current |

The datasheet gives Vf only at 20 mA. At the nPM1300's 5 mA sink current, Vf is lower (see its Vf–If curve), so the dropout points are lower than the table. Measure them at bring-up (§9). Green and blue dim or drop out as the cell runs down, so a low-battery indication must use red alone (OI-H9). Green and blue are used for pairing and status, which mostly happen on USB power or at a healthy charge.
- **No series resistors** (D10). No cathode on an nRF GPIO (HW-5).

### 4.9 Piezo locator: KELIKING KLJ-1230 (BZ1)

Replaces the Murata PKLCS1212E4001-R1, which DigiKey lists as Last Time Buy. Same form: passive (externally driven) piezo, 12 × 12 × 3.0 mm SMD, LCP housing (LCSC C201042).

| Parameter | KLJ-1230 | Use in this design |
|---|---|---|
| SPL | ≥80 dB at 10 cm, 3 Vp-p square, 4.1 kHz | Driven at 6 Vp-p differential (HW-9) |
| Maximum drive | 20 Vp-p | 6 Vp-p |
| Capacitance | 16 nF ±30% at 1 kHz | ~0.8–1.0 mA average at 4 kHz, 6 Vp-p |
| Current | ≤1 mA at 3 Vp-p, 4 kHz | — |
| Operating temperature | −20 to +70 °C | — |

- Symbol `Device:Buzzer`. **Footprint: custom** `imports:BUZ-SMD_L12.0-W12.0`. The datasheet gives no land pattern; its drawing shows two terminals 1.2 mm wide × 4.2 mm long, centred on two opposite 12 mm edges. The footprint uses two 2.0 × 5.0 mm pads centred on the body edges (12 mm apart), so each extends 1 mm past the body for a visible fillet. Check against the part before ordering.
- The **sound hole is near one corner** of the top face, not centred. The footprint does not yet mark it on the silkscreen. Align the enclosure port with it (OI-M7).
- The datasheet's typical response curve appears to peak near 3.8 kHz and 4.6 kHz, with a dip near 4.0 kHz (read from a low-resolution plot). Measure the peak at bring-up (§9) and centre the locate sweep on it (FW-Z1).
- BZ1.1 → `BUZ_A` → U1.22 (P0.06). BZ1.2 → `BUZ_B` → U1.24 (P0.08). Direct, no series parts. Routed at 0.3 mm.
- Firmware drives both pins high-drive (H0H1), one PWM channel inverted, for 6 Vp-p (HW-9). Both legs sit low at idle.
- BZ1 sits on the left half of the board, clear of the antenna keep-out.

### 4.10 Test points

All test pads are `TestPoint:TestPoint_Pad_D1.0mm`, bare pads marked DNP so JLC places nothing on them. On the PCB, their reference designators were renamed to signal names so the silkscreen labels the pads:

| Schematic | PCB reference (silkscreen) | Net |
|---|---|---|
| TP1 | `SHP` | SHPHLD |
| TP2 | `VBUS` | `VBUS` |
| TP3 | `VBAT` | Cell side of R2 (`Net-(J2-+)`), so it reads the cell even with R2 lifted |
| TP4 | `VSYS` | `VSYS` |
| TP5 | `3V0` | `3V0` |
| TP6 | `VMOT` | `VMOT` |
| TP7 | `IPRO` | `MOT_IPROPI` |
| TP8 | `nFault` | `MOT_nFAULT` |
| TP9 | `TP9` | CC1 |
| TP10 | `CC` | CC2 |
| TP11 | `HALL` | `HALL_SENSE` |
| TP12 | `INT` | `PMIC_INT` |
| TP13 / TP14 | `SDA` / `SCL` | `SDA` / `SCL` |
| TP15 / TP16 | `GND` / `GND` | GND |

The PCB references no longer match the schematic, TP15/TP16 share the reference `GND`, and the three fiducials are all `REF**`. Running "Update PCB from Schematic" will rename the pads back to TP1–TP16. See the review questions about the assembly (CPL) file.

Current measurement: lift R2 and put the PPK2 across its pads to measure the whole system.

## 5. nRF52840 pin allocation

All signal pins avoid the low-frequency-only set (D8) except IPROPI, which is analog. Pins may be swapped within the unrestricted set; if swapped, update this table and the firmware board file.

| Signal (net) | nRF pin | Module pin | Dir | Notes |
|---|---|---|---|---|
| `SDA` | P0.04 | 20 | I/O | TWIM, external pull-up R4 |
| `SCL` | P0.05 | 21 | Out | TWIM, external pull-up R5 |
| `BUZ_A` | P0.06 | 22 | Out | PWM, high drive |
| `BUZ_B` | P0.08 | 24 | Out | PWM (inverted), high drive |
| `MOT_IN1` | P1.08 | 25 | Out | DRV8213 internal pull-down |
| `MOT_IN2` | P1.09 | 26 | Out | DRV8213 internal pull-down |
| `MOT_nFAULT` (nFAULT + nSTALL) | P0.13 | 37 | In | Open-drain, R6 pull-up to `VMOT`. Reads low while `VMOT` is off: arm the IRQ only during actuation. |
| `HALL_SENSE` | P0.14 | 36 | In | Lid sensor. SENSE wake, no pull |
| `PMIC_INT` | P0.15 | 39 | In | SENSE wake, from nPM1300 GPIO1 (HW-12) |
| `MOT_IPROPI` | P0.02 / AIN0 | 11 | Analog | SAADC, diagnostics and OI-H1 capture |
| `nRESET` | P0.18 | 40 | — | Tag-Connect, R1/C2, nPM1300 GPIO0 |
| SWDIO | — | 51 | — | Tag-Connect |
| SWDCLK | — | 53 | — | Tag-Connect |
| *Reserved* | P0.00/P0.01 | 17/18 | — | 32 kHz crystal option |
| *Avoid* | P0.09/P0.10 | 52/54 | — | NFC by default |

Every other GPIO is no-connect. Unused nRF pins default to disconnected inputs, which draw no current.

## 6. Bill of materials (prototype)

`bom.csv` is the assembly BOM (MPN, LCSC part number, footprint, package, ratings, DNP). This table is the functional summary.

| Ref | Function | Part / value | Package / footprint | Fitted | Spec ref |
|---|---|---|---|---|---|
| U1 | MCU / BLE | Raytac MDBT50Q-P1MV2 | Module | Yes | §4.4 |
| U2 | PMIC | Nordic nPM1300-QEAA | QFN32 5×5, stock ThermalVias footprint | Yes | §4.3 |
| U3 | Motor driver | TI DRV8213RTER | WQFN-16 (RTE), TI land pattern | Yes | §4.6 |
| U4 | Lid sensor | TI DRV5032FBLPGM | TO-92, off-board; symbol carries `SolderWire-0.1sqmm_1x03` pads | DNP: off-board, hand-soldered to its pads | §4.7, OI-H3 |
| M1 | Lock actuator | INEED IND-GM-609-699 (alt 136:1) | 6 mm gearmotor, off-board; symbol carries `SolderWire-0.1sqmm_1x02` pads | DNP: off-board, hand-soldered to its pads | OI-H14 |
| BT1 | Battery | Prototype: Adafruit 1578, 1S LiPo 500 mAh, PCM fitted, JST-PH plug. Production: IEC 62133-2 / UN38.3 | Off-board, not in schematic | Plugs into J2, bought separately | OI-H8 |
| TH1 | Charge temperature | Murata NCP18XH103F03RB, 10 kΩ 1%, β 3380 K | 0603 | Yes | §4.2 |
| BZ1 | Locator | KELIKING KLJ-1230 (LCSC C201042) | 12×12×3 mm SMD, custom footprint | Yes | §4.9 |
| D4 | Indicator | TUOZHAN P4-1204RGBWS2-1.5T-A, common-anode RGB, side-emitting (LCSC C2827295) | 3.2 × 1.0 mm, custom footprint | Yes | §4.8 |
| D1 | VBUS ESD | LRC LTVS16H12T5G, unidirectional TVS, 12 V (LCSC C172439) | DFN1610-2, custom footprint | Yes | §4.1 |
| D2, D3 | CC ESD (CC1, CC2) | onsemi ESD9B5.0ST5G, bidirectional, V_BR ≥ 5.8 V (LCSC C111566) | SOD-923 | Yes | §4.1 |
| J1 | Power input | Same Sky UJC-H-G-SMT-2-P6-TR, USB-C 6-pin power-only (LCSC C45343229) | SMD + 4 through-hole shell legs; stock GCT USB4125 footprint (matches Same Sky layout) | Yes | §4.1 |
| J2 | Battery connector | JST S2B-PH-SM4-TB(LF)(SN), 2-pin JST-PH (LCSC C295747) | SMD right-angle, 2.0 mm pitch | Yes | §4.2 |
| J3 | Debug | Tag-Connect TC2030 footprint | — (no part) | — | §4.5 |
| TP1–TP16 | Test pads | — | 1.0 mm pad | DNP (bare pads) | §4.10 |
| L1, L2 | Buck inductors | Inpaq WIP201610S-2R2ML, 2.2 µH, DCR ≤ 140 mΩ, Isat ≥ 2.45 A (LCSC C964111) | 2016, custom footprint | Yes | §4.3 |
| C1, C6, C7, C10, C12, C15, C16 | Bulk / buck | Samsung CL10A106MA8NRNC, 10 µF X5R 25 V ±20% (Nordic's Configuration 1 part). C6/C10 must keep ≥4 µF effective at 3.0 V (§3.2). | 0603 | Yes | §4.3, §4.4, §4.6 |
| C4 | VBAT | Samsung CL10A225KO8NNNC, 2.2 µF X5R 16 V | 0603 | Yes | §4.2 |
| C3, C8 | VBUS, VBUSOUT | Samsung CL10B105KB8NQNC, 1.0 µF X7R 50 V | 0603 | Yes | §4.1, §4.3 |
| C2, C5, C9, C11, C13, C17 | Decoupling / reset / sensor | Samsung CL05B104KO5NNNC, 100 nF X7R 16 V | 0402 | Yes | §4.3–§4.7 |
| C14 | TINRUSH | Murata GRM1555C1H222JA01D, 2.2 nF C0G 50 V 5% (tune) | 0402 | Yes | §4.6 |
| R1, R4, R5, R6 | Pull-ups | 10 kΩ 1% | 0402 | Yes | §4.3, §4.5, §4.6 |
| R2 | VBAT measurement link | 0 Ω | 0603 | Yes | §4.2 |
| R3 | VSET2 (BUCK2 = 3.0 V) | 150 kΩ 1% | 0603 | Yes | §4.3 |
| R7 | R_IPROPI | 5.1 kΩ 1% (tune) | 0603 | Yes | §4.6 |
| R8, R9 | VREF divider | 100 kΩ / 200 kΩ 1% | 0402 | Yes | §4.6 |
| R10 | GAINSEL strap (High) | 0 Ω | 0603 | Yes | §4.6 |
| R11 | Lid signal series | 1 kΩ | 0402 | Yes | §4.7 |
| R12 | Lid sensor supply series | 100 Ω | 0402 | Yes | §4.7 |
| (3 × `REF**`) | Fiducials | `Fiducial:Fiducial_1mm_Mask2mm` (PCB only, excluded from BOM) | — | — | §7 |
| — | Magnets | Lid sensor and retention magnets | TBD | — | HW-10, OI-H4 |

## 7. Layout guidance (whole board)

Subsystem-specific rules are in §4. Board-level facts and rules:

- **Board:** 40 × 40 mm square, four Ø4.0 mm unplated mounting holes centred 3.5 mm in from each corner edge. All components on the top side. Three 1 mm fiducials (top side).
- **Antenna:** module antenna at the top board edge, away from battery, motor, and magnets, with the copper keep-out of §4.4. RF is verified in the enclosure (OI-H11); the enclosure and lid materials near the antenna affect it.
- **Motor loop:** BUCK1 → DRV8213 → motor loop tight, bulk capacitor next to VM.
- **nPM1300:** follow the reference layout for the buck inductors and capacitors; thermal-pad via array.
- **ESD:** TVS parts at J1; ground returns at the connector.
- **Test points:** all on the top side, reachable with the board in a fixture.

### 7.1 Stackup

4-layer, 1.6 mm, JLCPCB standard stackup **JLC7628** (JLC04161H-7628). The board file's stackup is set to these values.

| Layer | KiCad layer | Role (as built) | Copper | Dielectric below |
|---|---|---|---|---|
| L1 | F.Cu | Signal: components, buck power stage, power tracks; GND fill | 1 oz (0.035 mm) | 7628 prepreg, 0.21 mm |
| L2 | In1.Cu | **Solid GND plane**, the reference for L1 | 0.5 oz (0.0152 mm) | Core, 1.065 mm |
| L3 | In2.Cu | **`3V0` plane**: a single pour across the whole board, no tracks | 0.5 oz (0.0152 mm) | 7628 prepreg, 0.21 mm |
| L4 | B.Cu | Signal: routing overflow (including some `VSYS`), GND fill | 1 oz (0.035 mm) | — |

Rules for this stackup:
- **L2 stays solid and GND-only.** No tracks or other-net pours on it (enforced by DRC). GND pads get vias to L2 at the pad, and the U2/U3 thermal vias land in it. GND fill on L1/L4 is stitched to L2.
- **L3 is a `3V0` plane.** `VSYS` and `VMOT` are routed as tracks on L1 (and L4 for part of `VSYS`), not as L3 pours. If power tracks are ever put on L3, inner copper is 0.5 oz, so a track needs about twice its outer-layer width for the same current (IPC-2221 internal, 10 °C rise: ~1.8 mm for 1 A, ~0.2 mm for 0.2 A); the DRC rules enforce 1.8 mm for Power_High and 0.4 mm for Power/Switch on In2.Cu.
- **Buck switch nodes (SW1/SW2) stay on L1** with the inductors, with no vias (§4.3, enforced by DRC). L2 directly beneath gives the input and output loops their tightest return path.
- **Signals on L4** reference the continuous L3 `3V0` plane.
- **Antenna keep-out applies to all four layers** (§4.4): the L2 GND plane and L3 `3V0` plane are cut back too, not just the outer layers.
- **Finish (Proposed): ENIG.** The 0.5 mm-pitch QFNs, the DFN1610 TVS, the side-view LED, and the module's LGA pads all want a flat finish. HASL is cheaper and usually works, but it is less reliable on these parts. The board file's copper-finish field is still "None"; select ENIG when ordering.

### 7.2 Net classes and trace widths

Net classes are defined in `ble-actuator-board.kicad_pro`. Hard minimums and layer rules are in `ble-actuator-board.kicad_dru`. Widths assume 1 oz outer copper (IPC-2221 external, 10 °C rise). On the low-current rails they are set for voltage drop and loop inductance, not heating.

| Class | Nets | Worst-case current | Outer width, default / min | L3 width min | Clearance | Via (dia / drill) |
|---|---|---|---|---|---|---|
| Power_High | `VBUS`, `VBAT` (both sides of R2), `VSYS` | VBUS ≤1.5 A at the maximum input limit (500 mA planned), VBAT/VSYS ≤1 A | 0.5 / 0.25 mm | 1.8 mm | 0.2 mm | 0.8 / 0.4 mm, ≥2 per layer change |
| Switch | SW1, SW2 | ~0.4 A peak inductor current | 0.4 / 0.25 mm, ≤5.5 mm long | L1 only | 0.2 mm | none |
| Power | `3V0`, `VMOT`, motor OUT1/OUT2 | ≤200 mA (buck rating / ITRIP) | 0.3 / 0.2 mm | 0.4 mm | 0.15 mm | 0.6 / 0.3 mm |
| GND | GND | return currents | 0.4 / 0.15 mm (planes and fills preferred) | — (L2 plane) | 0.15 mm | 0.6 / 0.3 mm |
| Analog | `MOT_IPROPI`, NTC, VSET2, VREF, TINRUSH | µA | 0.2 / 0.15 mm | — | 0.2 mm, plus 0.5 mm from SW nodes | 0.6 / 0.3 mm |
| Default | everything else: I²C, SWD, GPIOs, LED cathodes, CC, lid sensor | ≤10 mA | 0.2 / 0.15 mm | — | 0.15 mm | 0.6 / 0.3 mm |

- **Minimums:** each class minimum still carries that class's worst-case current (IPC-2221 external, 1 oz, 10 °C rise). Route at the default width; the minimum is for short runs into pads and caps.
- **Neck-down:** inside the U1, U2, and U3 courtyards (and L1/L2 for the switch nodes), power tracks may narrow to 0.15 mm to enter pads 0.24–0.4 mm wide. GND stays at 0.2 mm or wider there, because U2's PVSS1/PVSS2 pins carry buck switching current.
- **Thermal spokes:** pads on U1, U2, U3, C10, and TH1 may connect to the GND pour with a single spoke.
- **I²C:** `SDA`/`SCL` also keep 0.5 mm from the SW nodes (§4.3).
- **GND to SW nodes:** 0.15 mm (the GND clearance), not the 0.2 mm Switch clearance. This lets U2's PVSS1/PVSS2 escape at full width between the SW tracks.
- **Board minimums:** 0.127 mm track and clearance, 0.3 mm drill, 0.5 mm via diameter, 0.1 mm annular ring, 0.25 mm hole-to-hole, 0.5 mm copper to edge. These are JLC's 1–2 layer limits, kept as margin; the 4-layer process allows finer, but this board doesn't need it.
- **As built:** 516 track segments and 139 vias (135 at 0.6/0.3 mm, 4 at 0.8/0.4 mm). DRC: 0 violations, 0 unconnected, 0 schematic-parity issues.

## 8. Enclosure integration

| Req | Requirement |
|---|---|
| EN-2 | The locking bar rotates perpendicular to the lid's travel. Lid loads go into the enclosure, not the gearbox (OI-M2). Hard stops are compliant (OI-M8). |
| EN-3 | Magnets retain and align the closed lid so the bar engages without the user holding it. |
| HW-10 | The retention magnets cannot fake a "closed" sensor reading. |
| — | Sound port with gasket for the piezo (OI-M7). Light path for the side-emitting LED (§4.8). Battery pocket sized per OI-H8. |

## 9. Bring-up plan

1. **Power-off checks:** shorts on VBUS, VBAT, VSYS, `3V0`, and `VMOT`. Then hand-solder the lid sensor (U4) and motor (M1) leads to their pads, and repeat the short check on `3V0` and `VMOT`. Leave the cell unplugged until step 3.
2. **USB only, no battery:** with a C-to-C cable and a USB-C-only charger (not A-to-C), confirm the charger turns on VBUS through the nPM1300's internal Rd alone, and the board powers up (HW-11, OI-H6).
3. **Battery fitted:** measure the cell's JST-PH plug polarity with a meter against the J2 silkscreen, then plug it into J2. Confirm ship-mode entry (no VBUS) and exit on VBUS insertion with a C-to-C charger, which only supplies VBUS if Rd is present in ship mode (OI-H7, HW-6, HW-11). Check SHPHLD exit from TP1.
4. **BUCK2 (`3V0`):** confirm it comes up at 3.0 V from VSET2, then program over Tag-Connect and run a blinky test. Confirm `VMOT` (BUCK1) stays at 0 V.
5. **TWI to nPM1300:** read IDs. Configure the charger, NTC, LEDs, and VBUS limit (FW-B6).
6. **PMIC_INT:** check that VBUS insertion and removal events reach the nRF (HW-12).
7. **Lid sensor:** check open/close edges, latency (≤250 ms), and System ON wake.
8. **Motor:** with BUCK1 on, run one direction into a hard stop and capture current and time-to-stall on the scope or PPK2. This produces the **OI-H1** data and sets ITRIP (**OI-H2**), `T_jam_min`, and `T_act_max`. Start at ITRIP ≈40 mA (High gain, R7 = 5.1 kΩ) and raise it step by step while measuring torque. If the target is above 60 mA, lift R10 and use the Hi-Z R7 values (§4.6). At high ITRIP the 699:1 gearbox can exceed its 250 g·cm limit at stall.
9. **Piezo:** sweep 3–5.5 kHz and find the SPL peak (the KLJ-1230 curve shows two, near 3.8 and 4.6 kHz). Set the locate sweep centre to it (FW-Z1), then measure SPL at 10 cm and 3 m.
10. **LED:** with a bench supply on VBAT, step down from 4.2 V and record where green and blue visibly dim at the 5 mA sink current. This sets the §4.8 thresholds and confirms the red-only low-battery choice.
11. **Sleep current:** lift R2, measure the whole system with the PPK2 across its pads (OI-F7), then refit R2. Target ≤10 µA.
12. **Charge temperature:** with a thermocouple taped to the cell, charge at room temperature, from a cold soak (device at about 0 °C, plugged in at room temperature), and in a warm chamber. Confirm charging stops or derates before the cell leaves its rated range, and record the TH1-to-cell offset. Open and short TH1 and confirm charging stops.

## 10. Validation targets

| Metric | Target | Method |
|---|---|---|
| Lid detection latency | ≤250 ms | Scope: magnet edge to GPIO / log timestamp |
| Lock/unlock travel | ≤0.5 s | Scope on motor current |
| Actuation reliability | ≥99.9% over 5,000 cycles | Automated cycle rig (firmware test command) |
| Sleep current | ≤10 µA | PPK2 across R2 |
| Battery life | ≥1 month typical | Power budget + PPK2 profile |
| Locate audibility | Audible at ~3 m | Phone SPL meter in a quiet room, plus pocket and couch tests |
| Stall torque | ≤150 g·cm | 1 cm arm on a scale (OI-H2) |

## 11. Schematic review checklist

Status against the current schematic: items 1–9 pass, with the exceptions noted.

1. **ERC clean:** no floating pins; every unused pin has a no-connect flag; PWR_FLAGs on `VBUS`, `VBAT`, GND, and the lid-sensor supply after R12 (and on `3V0`, `VMOT`, or `VSYS` only if ERC reports them undriven). The U2 LSIN/LSOUT pins tied to GND (D9) raise output-to-output conflicts; these are excluded in the project with a comment. **Pass (0 errors).**
2. **Net names** match §3.1 and §5: `VBUS`, `VBAT`, `VSYS`, `3V0`, `VMOT`, `SDA`, `SCL`, `nRESET`, `PMIC_INT`, `MOT_IN1`, `MOT_IN2`, `MOT_nFAULT`, `MOT_IPROPI`, `HALL_SENSE`, `BUZ_A`, `BUZ_B`. **Pass.**
3. **Pin-by-pin:** U2 matches §4.3, U3 matches §4.6, U1 matches §4.4 and §5, J3 matches §4.5. **Pass.**
4. **Rails:** nothing on `VBAT` except U2.19, C4, and R2. Only U2.1, L1, C10, U3 VM/VCC/IMODE (pins 5, 2, 15), C11, C12, C13, R6, R10, and the VREF divider (R9) on `VMOT`. Nothing on `3V0` connects to U3. LED anode on `VSYS`. No motor load on `VSYS` (HW-1). **Pass.**
5. **Parts and values** match §6. U4 is DRV5032**FB**. D1 is LTVS16H12T5G, with its cathode on `VBUS`. R3 is 150 kΩ on VSET2, and VSET1 is grounded. C14 is C0G (GRM1555C1H222JA01D), not X7R. D4's symbol pins are A = 4, GK = 1, BK = 2, RK = 3 (§4.8). **Pass, except that D4's Value field still reads NH-B1515RGBA-GF.**
6. **Fitted/DNP:** the only DNP symbols are U4, M1, and TP1–TP16. The only 0 Ω resistors are R2 and R10, both fitted. **Pass.**
7. **Footprints:** every symbol has one. J2 is the JST-PH header, placed by JLC. U4 and M1 carry their own wire-pad footprints and are DNP. BT1 is not in the schematic. C17 is on the board at the U4 pads. TH1 is on the board. BZ1, D1, D4, L1/L2, and U3 use custom footprints (§4). **Pass.**
8. **Sleep-leak audit:** no resistor to a rail conducts in sleep. I²C pull-ups (bus idles high), R1 (nRESET high), R6, R10, and the VREF divider (all on `VMOT`, which is off), and R11 and R12 (series only). Expected resistor leakage is ~0 µA. **Pass.**
9. **No forbidden parts:** no CC pull-downs, no IN1/IN2 pull-downs, no LED resistors, no capacitor across the motor, no capacitor >100 pF on NTC, no pull resistor on `HALL_SENSE`. **Pass.**

## 12. Firmware dependencies created by the hardware

| Item | Firmware action | Source |
|---|---|---|
| BUCK2 (`3V0`) boots at 3.0 V from VSET2 | No action. Do not reprogram BUCK2's voltage. | D2 |
| BUCK1 (`VMOT`) off at boot | Set BUCK1 to 3.0 V and enable it only during actuation | D3, FW-A1 |
| VBUS limit 100 mA at boot | Write 500 mA on each VBUS insert | D7, FW-B6 |
| GPIO0 on nRESET | Configure as open-drain output (NRESETOUT) early in boot; this removes the ~6 µA pull-down leak | D6, D11 |
| GPIO1 as PMIC_INT | Configure as interrupt output | HW-12 |
| Pin reset on P0.18 | Enable `CONFIG_GPIO_AS_PINRESET`; without it, nRESET from the debugger or PMIC watchdog does nothing | §4.5 |
| NTC | Select 10 kΩ and TH1's β in ADCNTCRSEL. Cold threshold ≈5 °C, reduced-current cool band below ≈10 °C, hot and warm per the cell datasheet. Charge current ≤0.5C. | §4.2 |
| Piezo | High drive (H0H1), inverted second channel, both legs low at idle. Centre the locate sweep on the peak measured at bring-up. | §4.9, §9 |
| LED | LED0 = blue, LED1 = green, LED2 = red. Low-battery indication is red only: green and blue drop out near the bottom of the cell's range. | §4.8 |
| DRV8213 | IN1 = IN2 = 0 outside actuation; read nSTALL as end of travel or jam | D1, §4.6 |
| DRV8213 on `VMOT` | `MOT_nFAULT` is low whenever BUCK1 is off. Arm its interrupt only after `VMOT` is up plus t_WAKE (≤250 µs), and disarm it before disabling BUCK1. Keep BUCK1 at or below the `3V0` voltage. | D12 |

## 13. Removed rework options

The design removes every DNP part and 0 Ω resistor that is not expected to be needed (see the design rule at the top). This section records what each would have provided, how to get the same result without it, and when to add it back.

| Removed option | Benefit it would have given | How to get the same result without it | Add it back if |
|---|---|---|---|
| **0 Ω strap on IMODE** (to `VMOT`) | Lift it to change IMODE to Hi-Z without cutting a trace | Cut the short trace to pin 15. Changing IMODE is a design change, not a calibration step: HW-2 requires regulation always on. | A later design wants IMODE configurable per build |
| **0 Ω strap on SMODE** (to GND) | Lift it to change SMODE to Hi-Z (indication-only stall, internal 510 mV VREF) | Cut the trace to pin 11. Hi-Z would also make the VREF divider pointless (D4). | Bench testing shows the latched stall cutoff is unwanted, for example it trips at end-of-travel where firmware should decide |
| **DNP opposite-rail 0 Ω on IMODE and SMODE** | Change either pin to the opposite logic level by fitting a part | Cut and bodge-wire the pin to the other rail | Same as above |
| **Second GAINSEL footprint (to GND)** | Select GAINSEL Low | Not needed. Low is a high-current range that is useless at this board's 40–65 mA ITRIP. | Never for this motor |
| **C_IPROPI, 10 nF DNP** (IPROPI to GND) | Filter IPROPI if motor-start transients trip current regulation early (DRV8213 §8.4.2.2) | Stack a 0603 capacitor on top of R7, which is already IPROPI-to-GND. C14 (TINRUSH) already blanks stall detection for ~14 ms at start. | Bench shows false regulation trips and stacking is not acceptable for a second build |
| **Piezo series 0 Ω × 2** | Room to add up to ~100 Ω per leg to soften GPIO edges for EMI | Not needed. The drive is ~4 kHz over short traces, and nRF high-drive outputs are internally current-limited. Series resistance would also reduce SPL. | Part 15B pre-scan (CM-3) shows emissions from the piezo drive |
| **Piezo bleed resistor, 10–47 kΩ DNP** (across BZ1) | Drain charge generated by mechanical shock on the piezo | While powered, firmware holds both legs low, so the piezo is shorted through the GPIOs. The GPIO ESD diodes clamp what's left, and the KLJ-1230 is small. | Drop tests show GPIO damage or resets on impact |
| **0 Ω on GPIO0 → nRESET** | Disconnect the PMIC watchdog reset line to isolate it during debug (for example, to rule out the ~6 µA boot leak, D11) | Cut the single trace. The line cannot brick the board: the nPM1300 configuration is lost on a power cycle, so pulling battery and USB recovers it. | First build shows reset or leakage problems traced to this line |
| **JP, `3V0` → `VDD_NRF`** (solder jumper) | Measure nRF current alone, separate from PMIC and peripherals | Measure the whole system across R2 and subtract the datasheet figures (§3.3). The sleep target (≤10 µA) is a whole-system number. | The whole-system sleep current misses target and the excess can't be attributed |

Other optional items from earlier drafts, not carried over:
- **Net ties on PVSS1/PVSS2.** Replaced by the layout rule in §4.3 that returns C10/C15 to pin 2 and C6/C16 to pin 6.

## 14. Compliance (hardware-related)

- FCC/ISED modular approval through the Raytac grant. The host still needs Part 15B testing (CM-3).
- UN38.3 documentation for the cell. CE/UKCA, RoHS, and REACH for EU/UK sale (CM-4).

## 15. Open items owned by hardware

🔴 OI-H1, OI-H4, OI-H8, OI-M1, OI-M2, OI-M6 · 🟡 OI-H2, OI-H11, OI-H14, OI-M7, OI-M8 · 🟢 OI-H3, OI-H7, OI-H12, OI-M3. Descriptions in [Appendix B](#appendix-b-open-items).

---

## Appendix A: Requirements referenced

**Hardware**
- **HW-1** The motor shall be supplied from a regulated 3.0 V rail (BUCK1), never directly from VSYS. VSYS can reach 4.2 V, above the motor's 3.6 V maximum.
- **HW-2** The DRV8213's current regulation (ITRIP) shall limit motor current below BUCK1's 200 mA rating and limit output torque at stall to ≤150 g·cm (60% of the gearbox's 250 g·cm instantaneous limit). The ITRIP setting shall be calibrated on the bench (OI-H2).
- **HW-3** The DRV8213 shall be unpowered except during actuation: its VM, its VCC, and its strap and pull-up supplies all come from BUCK1, which is off outside actuation. IN1 = IN2 = low (auto-sleep) is held whenever it is powered and idle.
- **HW-4** The lid sensor shall be a push-pull output variant, so no pull-up resistor conducts while the lid is closed. It is mounted off-board with a 100 nF decoupling capacitor on the PCB at its solder pads, short twisted three-conductor wiring (or a small flex), and its flat (branded) sensing face toward the lid magnet.
- **HW-5** The LED shall be common-anode, with anodes on VSYS and cathodes on the nPM1300 sinks. No LED cathode shall connect to an nRF GPIO.
- **HW-6** The device shall ship in nPM1300 ship mode (~370 nA) and wake on USB-C insertion (VBUS). No user button is fitted. SHPHLD shall be brought out to a test pad for bring-up.
- **HW-7** Programming and debug shall be through a Tag-Connect footprint (SWD).
- **HW-8** ESD protection shall be fitted on USB-C VBUS and CC.
- **HW-9** The piezo shall be driven differentially from two nRF GPIOs (PWM, one channel inverted), giving 6 Vp-p from the 3.0 V rail. It draws no current when idle and ~1 mA average while sounding. Its sound port shall align with an enclosure port and be sealed to it with a gasket.
- **HW-10** The lid sensor's magnet and the lid-retention magnets shall be placed so the retention magnets cannot hold the sensor in the "closed" state while the lid is open. One magnet may serve both roles.
- **HW-11** USB-C CC1 and CC2 shall connect directly to the nPM1300 CC1/CC2 pins, as in Nordic's reference design, with no discrete pull-downs. A USB-C-to-C charger shall supply power with a fully depleted cell and in ship mode (verified at bring-up, OI-H6).
- **HW-12** One nPM1300 GPIO shall be an interrupt line to the nRF, reporting VBUS insertion and removal and other PMIC events.

**Enclosure**
- **EN-2** The locking bar shall rotate in a plane perpendicular to the lid's direction of travel, so lid loads are carried by the enclosure rather than the gearbox.
- **EN-3** When closed, the lid shall be magnetically retained and aligned so the locking bar can engage without the user holding the lid in place.

**Firmware (hardware-facing)**
- **FW-A1** Actuation sequence: check battery voltage; enable BUCK1 (forced PWM); wake the DRV8213; drive in the required direction; stop on stall detection or timeout (`T_act_max`, about 1 s); sleep the DRV8213 and disable BUCK1. The motor is de-energized within 100 ms of stall detection. Travel ends at mechanical hard stops; there is no encoder.
- **FW-B4** Average sleep current ≤10 µA, excluding advertising.
- **FW-B6** At boot (and on each VBUS insert), set the nPM1300 VBUS input current limit to 500 mA.
- **FW-Z1** On a locate command, play repeating chirps that sweep a few hundred Hz around the piezo's measured resonance peak, for up to `T_locate` (e.g. 30 s).

**Compliance**
- **CM-3** FCC/ISED modular approval via the Raytac module grant; host-level unintentional-radiator testing (FCC Part 15B) still required.
- **CM-4** UN38.3 documentation for the lithium cell; CE/UKCA, RoHS, and REACH if selling in the EU or UK.

## Appendix B: Open items

| ID | Item | Priority | Plan |
|---|---|---|---|
| OI-H1 | Measure the motor's stall current and time-to-stall into the printed hard stop at 3.0 V | 🔴 | Bench supply + scope/PPK2. Sets ITRIP, `T_jam_min`, and `T_act_max`. |
| OI-H2 | Calibrate ITRIP to ≤150 g·cm at stall | 🟡 | 1 cm arm on the output shaft pressing a scale (1 g = 1 g·cm); raise ITRIP until ~150 g; confirm the bar still actuates reliably. ITRIP is set by R7, so it can change after the PCB is built. |
| OI-H3 | Lid sensor sampling rate: FB (5 Hz, 0.54 µA) vs FA (20 Hz, 1.3 µA) | 🟢 | Bench both; default to FB unless 200 ms latency feels slow |
| OI-H4 | Magnet plan: sensor magnet vs. lid retention/alignment magnets; sizes, grades, positions; no false "closed" reading | 🔴 | TI magnetic sense simulator, then a bench test with the enclosure |
| OI-H6 | *Resolved.* CC wired directly to the nPM1300 internal Rd per Nordic's reference (a discrete 5.1 kΩ in parallel would be out of spec). Bring-up confirms C-to-C charging with no battery and from ship mode. | — | §9 step 2–3 |
| OI-H7 | Verify ship-mode entry/exit, VBUS wake, and VBUS events reaching the nRF interrupt line (confirmed in the PS) | 🟢 | Bench test |
| OI-H8 | Battery cell capacity, dimensions, and protection-circuit trip current | 🔴 | Set by the enclosure volume and power budget (OI-F7) |
| OI-H9 | *Resolved.* Low-battery indication uses red only, because green and blue need VSYS well above the bottom of the cell range | — | Bench check at bring-up |
| OI-H11 | RF performance in the enclosure with battery, motor, magnets, and lid nearby | 🟡 | RSSI/range test on first build |
| OI-H12 | USB-C: power-only, or USB data for DFU/debug? | 🟢 | Power-only; Tag-Connect for programming |
| OI-H14 | Gear ratio: 699:1 vs 136:1. Both share the 250 g·cm gearbox limit. 136:1 multiplies motor torque ~5× less (finer ITRIP control, more stall margin), runs faster (242 vs 47 rpm), and is ~1.8 mm shorter. | 🟡 | Prototype both; choose based on the bar's measured required torque |
| OI-M1 | Locking bar geometry, hard stop positions, throw angle | 🔴 | CAD + printed iteration |
| OI-M2 | Motor mounting and coupling to the bar; check that lid loads are radial, not into the gears | 🔴 | CAD review |
| OI-M3 | Enclosure process for early units (in-house FDM vs. MJF/SLS service) | 🟢 | Cost/finish comparison |
| OI-M6 | Lid snap/alignment tolerance: will the retention magnets align the bar channel reliably across print variation? | 🔴 | Printed iteration + cycle testing |
| OI-M7 | Piezo sound port and gasket; check audibility (~45–55 dB expected at 3 m) in realistic spots such as a pocket or couch cushion | 🟡 | Printed enclosure + SPL meter |
| OI-M8 | Compliant hard stops (flexure or elastomer pad) so the bar never hits rigid plastic at stall | 🟡 | CAD + cycle testing with OI-H2 |
| OI-F5 | BLE advertising strategy and its share of the power budget | 🟡 | Power profiling |
| OI-F7 | Full power budget with measured numbers | 🟡 | PPK2 on first prototype |
| OI-F10 | LFRC calibration interval and its current cost (no 32 kHz crystal fitted) | 🟢 | Measure on prototype |
