# BLE Actuator Board

A small (40 × 40 mm, 4-layer) battery-powered Bluetooth LE board that drives a micro gearmotor latch into mechanical end stops and senses whether a magnetic closure is shut. It has no buttons: it is configured over BLE, charges from USB-C, and spends almost all its life asleep, so it needs to run for a month or more per charge.

**Status:** prototype rev 1. Schematic and layout are complete and pass ERC/DRC (KiCad 10). The board has not been ordered yet. **I'd value a review before I send it to fab**; see [what I'd like feedback on](#what-id-like-feedback-on).

## Contents

| Path | What it is |
|---|---|
| `kicad/ble-actuator-board.kicad_pro` | KiCad 10 project (schematic, PCB, custom design rules) |
| `kicad/easyeda_lib.pretty`, `kicad/easyeda_lib.3dshapes` | Project-local footprints and 3D models (the project's `fp-lib-table` points here) |
| `bom.csv` | Bill of materials with MPNs and LCSC part numbers (JLCPCB assembly) |

Schematic symbols are embedded in the schematic, so the project opens without any external libraries.

---

## 1. Functional requirements

### 1.1 What the hardware must do

| ID | Requirement |
|---|---|
| R-1 | Drive a 6 mm DC gearmotor in both directions to move a latch between two compliant hard stops. |
| R-2 | Limit motor current so that stall torque is **≤150 g·cm** (60% of the gearbox's 250 g·cm limit) and stall current stays below the motor rail's 200 mA rating. Stall must be cut off in hardware, without relying on firmware. |
| R-3 | Supply the motor from a regulated 3.0 V rail, never straight from the battery (VBAT reaches 4.2 V; the motor's maximum is 3.6 V). |
| R-4 | Detect the closure's open/closed state with an off-board Hall switch, with ≤250 ms latency, able to wake the MCU from sleep, and drawing no static current through pull-ups. |
| R-5 | Other magnets in the mechanism must not be able to hold the Hall sensor in the "closed" state. This is a mechanical rule, listed here because it drives sensor placement. |
| R-6 | Bluetooth LE through a pre-certified module (to reuse its FCC/ISED modular grant). |
| R-7 | Charge a 1S LiPo from USB-C (power-only) with a C-to-C charger, including from a fully depleted cell. |
| R-8 | Qualify charging on cell temperature. The pack is 2-wire, so the NTC is on the PCB next to the cell. |
| R-9 | Ship in a near-zero-current ship mode and wake when USB is plugged in. No user buttons. |
| R-10 | Audible locator: a piezo loud enough to hear at ~3 m, driven differentially from the MCU. |
| R-11 | RGB status LED that draws nothing when off. |
| R-12 | ESD protection on USB-C VBUS and CC. |
| R-13 | SWD programming and debug through a Tag-Connect footprint (no header). |
| R-14 | The PMIC reports events (USB insert/remove, charger) to the MCU on an interrupt line, and its watchdog can reset the MCU. |

### 1.2 Performance targets

| Metric | Target | How it will be checked |
|---|---|---|
| Closure detection latency | ≤250 ms | Scope: magnet edge to GPIO |
| Latch travel time | ≤0.5 s | Scope on motor current |
| Actuation reliability | ≥99.9% over 5,000 cycles | Automated cycle rig |
| Sleep current (whole system) | **≤10 µA** | PPK2 across the R2 link |
| Battery life | ≥1 month typical | Power budget + PPK2 profile |
| Locator audibility | Audible at ~3 m | SPL meter |
| Stall torque | ≤150 g·cm | 1 cm arm on a scale |

---

## 2. Hardware design overview

### 2.1 Block diagram

```
                        ┌──────────────────────── PCB ────────────────────────┐
 USB-C (6-pin) ─ESD─────┤ VBUS ─► nPM1300                                     │
  CC1/CC2 ─ESD──────────┤          ├─ VBAT ◄── R2 ◄──────── J2 (JST-PH) ◄─────┼── 1S LiPo (2-wire)
                        │          ├─ NTC ◄── TH1 (on board, at cell)         │
                        │          ├─ VSYS ─── RGB LED anode                  │
                        │          │           cathodes ◄─ LED0/1/2 sinks     │
                        │          ├─ BUCK2 3V0 ─► MDBT50Q VDD/VDDH           │
                        │          │              DRV5032 VDD ────────────────┼── U4 pads → off-board Hall switch
                        │          ├─ BUCK1 VMOT ─► DRV8213 VM + VCC ─► motor ┼── M1 pads → gearmotor
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

### 2.2 Key components

| Function | Ref | Part |
|---|---|---|
| MCU / BLE | U1 | Raytac MDBT50Q-P1MV2 (nRF52840 module, PCB antenna) |
| PMIC: charger, 2 bucks, LED sinks, ship mode | U2 | Nordic nPM1300 (QFN-32) |
| Motor driver | U3 | TI DRV8213RTER (WQFN-16) |
| Hall switch (off-board) | U4 | TI DRV5032FBLPGM, TO-92, omnipolar, push-pull, 5 Hz sampling |
| Gearmotor (off-board) | M1 | INEED IND-GM-609-699, 6 mm planetary, 699:1 (136:1 also being evaluated) |
| Locator | BZ1 | KELIKING KLJ-1230 passive piezo, 12 × 12 mm |
| Status LED | D4 | Common-anode RGB, P4-1204RGBWS2-1.5T-A |
| USB-C | J1 | Same Sky UJC-H-G-SMT-2-P6-TR, 6-pin power-only |
| VBUS TVS / CC ESD | D1 / D2, D3 | LRC LTVS16H12T5G (12 V unidirectional) / onsemi ESD9B5.0ST5G |
| Battery connector | J2 | JST-PH 2-pin SMD. Prototype cell: Adafruit 1578, 500 mAh, with PCM |
| Charge NTC | TH1 | Murata NCP18XH103F03RB, 10 kΩ, β 3380 K |

### 2.3 Power rails

| Net | Source | Voltage | Loads | Notes |
|---|---|---|---|---|
| `VBUS` | USB-C | 5 V | nPM1300 only | Input limit raised to 500 mA by firmware (boots at 100 mA) |
| `VBAT` | Cell through R2 (0 Ω) | 3.0–4.2 V | nPM1300 only | Lift R2 to measure whole-system current |
| `VSYS` | nPM1300 | ≈VBAT | RGB LED anode, PMIC PVDD | Never feeds the motor |
| `3V0` | BUCK2 | 3.0 V | nRF module, Hall switch (via 100 Ω), pull-ups, Tag-Connect | Always on outside ship mode. Set by VSET2 resistor at power-up |
| `VMOT` | BUCK1 | 3.0 V | DRV8213 VM **and** VCC, its straps and pull-up, VREF divider | **On only during actuation.** Held off at power-up (VSET1 grounded) |

Sleep current budget (target ≤10 µA): nRF52840 System ON with RTC ~2–3 µA, nPM1300 quiescent with one buck on ~1–2 µA, DRV5032FB ~0.7 µA, DRV8213 ~0 (unpowered), resistor leakage ~0. To be measured on the first build.

### 2.4 Subsystems

**USB-C input and ESD.** CC1/CC2 go directly to the nPM1300, whose internal Rd pull-downs present the sink, as in Nordic's reference design. There are no discrete 5.1 kΩ resistors, because they would put the combined Rd out of spec. D1 is a unidirectional TVS with a 12 V working voltage. That way a faulty 9–12 V charger only trips the PMIC's 5.5 V overvoltage cutoff instead of dumping current into the TVS, and the clamp (~19 V at 30 A) stays below the nPM1300's 22 V VBUS absolute maximum. D2/D3 on CC have V_BR ≥ 5.8 V, above the CC pins' 5.5 V absolute maximum.

**Battery and charge temperature.** The cell is a 2-wire pack with its own PCM, so TH1 sits on the board edge against the battery pocket. It is kept away from U2 and the inductors so charger heat doesn't make it read warm. Firmware sets the cold threshold at ~5 °C (instead of 0 °C) to allow for the board warming faster than the cell. An open or shorted NTC reads outside the charge window, so charging stops.

**PMIC (nPM1300).** Values follow Nordic's Configuration 1. BUCK2 is the always-on 3.0 V logic rail, because only VSET2 can select 3.0 V at power-up (150 kΩ). BUCK1 is the motor rail, held off at boot by grounding VSET1. GPIO0 is NRESETOUT to the nRF reset, so the PMIC watchdog resets the MCU without dropping `3V0`. GPIO1 is the interrupt line. SHPHLD goes to a test pad only.

**MCU module (MDBT50Q-P1MV2).** VDD and VDDH are tied together on `3V0` (normal-voltage mode); USB is unused. The module sits on the board edge with the antenna facing out and a full-width copper keep-out on all four layers. PWM, I²C and motor lines avoid the module's low-frequency-only pins.

**Motor driver (DRV8213).**
- **Power:** the whole driver, including VCC, its tri-level straps and its nFAULT pull-up, runs from `VMOT`, so it is completely unpowered between actuations. This sidesteps a datasheet gap: it doesn't say whether the internal strap bias (75 kΩ/130 kΩ) is switched off in sleep, and if it isn't, the straps would cost ~60 µA.
- **Straps:**
  - IMODE = High: current regulation always on.
  - SMODE = Low: latch the outputs off and assert nSTALL on stall. VREF comes from a divider on `VMOT` (1.00 V).
  - GAINSEL = High through a 0 Ω (R10): 10–60 mA range.
- **Current limit:** ITRIP = V_VREF / (R7 × A_IPROPI) starts at ~40 mA (R7 = 5.1 kΩ, 0603 so it's easy to swap during calibration).
- **Inrush and sleep:** C14 (2.2 nF C0G) sets ~14 ms of inrush blanking. There is no nSLEEP pin on this package: the driver auto-sleeps with IN1 = IN2 = 0.
- **Motor and current sense:** no capacitor across the motor. IPROPI also goes to an nRF ADC input for diagnostics.

**Hall switch (DRV5032FB, off-board).** It hangs on short wires from three solder pads. A 100 Ω series resistor on its supply (with 100 nF at the pads) and a 1 kΩ series resistor on its output protect `3V0` and the GPIO from faults and ESD on the wire. The output is push-pull, so there is no pull resistor and no static current.

**Piezo.** Driven differentially from two high-drive nRF GPIOs (one PWM channel inverted) for 6 Vp-p from a 3.0 V rail, with no series parts. Both legs sit low at idle. Firmware sweeps around the resonance peak measured at bring-up.

**RGB LED.** The anode is on `VSYS` and the cathodes go to the nPM1300's 5 mA current sinks, so there are no series resistors and no LED on an nRF GPIO. Because green and blue drop out as VSYS falls toward 3.3 V, low-battery indication uses red only.

**Debug and test.** Tag-Connect TC2030 (legged footprint) carries SWD, nRESET and 3V0 sense. Test pads are on every rail plus I²C, PMIC_INT, HALL_SENSE, motor nFAULT and IPROPI, SHPHLD, CC1/CC2 and GND.

### 2.5 nRF52840 pin map

| Net | nRF pin | Module pin | Notes |
|---|---|---|---|
| `SDA` / `SCL` | P0.04 / P0.05 | 20 / 21 | TWIM to nPM1300 (addr 0x6B), 10 kΩ pull-ups |
| `BUZ_A` / `BUZ_B` | P0.06 / P0.08 | 22 / 24 | PWM, high drive, one inverted |
| `MOT_IN1` / `MOT_IN2` | P1.08 / P1.09 | 25 / 26 | DRV8213 internal pull-downs |
| `MOT_nFAULT` | P0.13 | 37 | nFAULT + nSTALL, pulled to `VMOT`; reads low while `VMOT` is off |
| `HALL_SENSE` | P0.14 | 36 | SENSE wake, no pull |
| `PMIC_INT` | P0.15 | 39 | SENSE wake |
| `MOT_IPROPI` | P0.02 / AIN0 | 11 | SAADC |
| `nRESET` | P0.18 | 40 | Tag-Connect, 10 kΩ/100 nF, nPM1300 GPIO0 |

P0.00/P0.01 are left free for an optional 32 kHz crystal. Every other GPIO is no-connect.

### 2.6 PCB

- **Size and stackup:** 40 × 40 mm, four 4 mm mounting holes. 4-layer 1.6 mm, JLCPCB JLC7628 stackup: L1 signal / L2 solid GND / L3 power pours (`VSYS`, `3V0`, `VMOT`) / L4 signal + GND fill. ENIG finish planned (0.5 mm QFNs, DFN1610 TVS, LGA module).
- **Rules (custom `.kicad_dru`):** L2 stays GND-only, and the buck switch nodes stay on L1 with no vias.
- **Net classes:**

| Class | Nets | Width default / min | Clearance |
|---|---|---|---|
| Power_High | `VBUS`, `VBAT`, `VSYS` | 0.5 / 0.25 mm | 0.2 mm |
| Switch | SW1, SW2 | 0.4 / 0.25 mm, ≤5.5 mm long, L1 only | 0.2 mm |
| Power | `3V0`, `VMOT`, motor outputs | 0.3 / 0.2 mm | 0.15 mm |
| Analog | IPROPI, NTC, VSET2, VREF, TINRUSH | 0.2 / 0.15 mm | 0.2 mm, plus 0.5 mm from SW nodes |
| Default | everything else | 0.2 / 0.15 mm | 0.15 mm |

- **Layout priorities:**
  - The nPM1300 PVDD capacitors return to PVSS1/PVSS2, and each buck output capacitor returns to its own PVSS pin before reaching the plane.
  - Thermal-via arrays under U2 and U3.
  - The motor loop (BUCK1 → C12 → VM → OUT → motor → PGND) is kept tight.
  - The TVS parts sit at J1, and the J1 shell is stitched to GND.
  - Analog nets and I²C are kept away from the switch nodes.

### 2.7 Assembly

JLCPCB assembles everything except:
- **U4 (Hall switch)** and **M1 (gearmotor):** off-board parts hand-soldered to wire pads after assembly. Their symbols carry the pad footprints and are marked DNP.
- **TP1–TP16:** bare test pads (DNP).
- **Battery:** plugs into J2. JST-PH polarity isn't standardized between vendors, so check the plug against the J2 silkscreen before connecting.

J1's shell legs are through-hole in plated slots.

---

## 3. What I'd like feedback on

1. **nPM1300 layout:** buck loops, PVSS returns, thermal pad, and the switch-node routing.
2. **Antenna region:** whether the keep-out and module placement on a 40 mm board are adequate.
3. **DRV8213 configuration:** the strap choices, running VCC from the switched motor rail, and the ITRIP/VREF setup for a ~40–65 mA stall limit. Also: does the nFAULT line pulled to `VMOT` while read by a `3V0` GPIO (both 3.0 V ±5%) look acceptable?
4. **USB-C ESD and TVS choice** and placement.
5. **Off-board wiring protection** (series R on the Hall switch supply and output).
6. **4-layer stackup and power-pour strategy:** is L3-as-power with signals on L4 sensible here?
7. **DFM** for JLC assembly: custom footprints for the piezo, LED, and DFN1610 TVS; SOD-923 parts; and the legged Tag-Connect footprint.
8. Anything that would stop the board reaching **≤10 µA** in sleep.

## Known open items

- Motor stall current and time-to-stall at 3.0 V are not measured yet. ITRIP (R7) and C14 will be tuned on the bench.
- Final gear ratio (699:1 vs 136:1) and final battery cell.
- RF performance in the final enclosure.
