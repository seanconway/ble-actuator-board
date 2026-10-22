# Review request: nRF52840 + nPM1300 smart-lock board, 4-layer, before first JLC order

I'm about to order the first prototype of a small IoT smart-lock board and would really appreciate expert eyes on it before it goes to fab and assembly. KiCad 10 project, BOM, and the full design spec are in this repo ([README.md](README.md)).

## What it is

The board goes in a pocket-sized container with a motorized lock. It senses the lid with a Hall switch and drives a 6 mm gearmotor that swings a locking bar into compliant hard stops, using stall detection instead of an encoder. Everything is controlled over BLE from a phone app. There are no buttons: it ships in PMIC ship mode and wakes on USB-C. The main targets are **≤10 µA average sleep** and a month or more per charge on a ~500 mAh 1S LiPo.

- **Board:** 40 × 40 mm, 4-layer 1.6 mm (JLC7628): signal / GND / 3V0 plane / signal. All parts on top. JLC assembly; the motor and Hall switch are hand-soldered to wire pads afterwards.
- **MCU:** Raytac MDBT50Q-P1MV2 (nRF52840 module, PCB antenna) on the top edge.
- **Power:** Nordic nPM1300: charger, ship mode, and two bucks. BUCK2 is the always-on 3.0 V logic rail. BUCK1 is a 3.0 V motor rail that is only on during actuation.
- **Motor:** TI DRV8213 (RTE), powered entirely from the switched motor rail so it draws nothing in sleep. Current regulation is always on, and the outputs latch off on stall. ITRIP starts at ~40 mA, which caps stall torque at ≤150 g·cm.
- **Lid sensor:** TI DRV5032FB Hall switch, off-board on short wires, with series R and decoupling at the pads.
- **Other parts:** USB-C power-only with a 12 V unidirectional TVS on VBUS and ESD diodes on CC (CC wired straight to the nPM1300's internal Rd). Differential piezo locator. Common-anode side-view RGB LED on the PMIC's LED sinks. Board NTC for charge temperature (2-wire cell). Tag-Connect for SWD.

ERC and DRC are clean, with 0 unconnected items and schematic/PCB parity OK. Design rationale for every value is in the README (§2 lists the datasheet constraints that drove the choices).

## Where I'd most value review

In rough priority order, the items most likely to cost a respin:

1. **nPM1300 layout (README §4.3, §7.1).** Buck input/output loops (C15/C16 to PVSS1/PVSS2, C10/C6 returns), switch-node routing on L1, and the exposed pad. I'm using KiCad's stock QFN-32 ThermalVias footprint, not Nordic's. Its 3 × 3 paste windows sit over the via array, so I'm worried about solder wicking and voiding at JLC. Should I tent or plug the vias, or shrink the paste?
2. **Antenna region (§4.4).** The module is on the top edge, with a 4.4 mm full-width keep-out on all four layers and a GND/3V0 plane under the rest of the module. Is that adequate on a 40 mm board with a motor, magnets, and a LiPo nearby?
3. **USB-C footprint and ESD (§4.1).** The Same Sky UJC-H-G-SMT-2-P6-TR uses KiCad's GCT USB4125 6-pin footprint. I checked the pad pitches, widths, slot sizes, and offsets against Same Sky's recommended layout and they match. Is the TVS and ESD choice and placement sensible? Is anything wrong with relying on the nPM1300's internal Rd only?
4. **DRV8213 configuration (§4.6).** Questions:
   - Is running VCC and all the tri-level straps from the switched VM rail acceptable?
   - Is the VREF divider (1.0 V) with GAINSEL High and R7 = 5.1 kΩ a sensible starting point for a ~40–65 mA stall limit?
   - nFAULT is pulled up to the motor rail but read by a GPIO on the logic rail. Both are 3.0 V ±5%. Is that acceptable?
5. **Power integrity on 4 layers (§7.1).** L2 is solid GND and L3 is one 3V0 plane. VSYS (≤1 A) and VMOT run as 0.3–0.5 mm tracks on the outer layers. Is that fine, or should VSYS get copper on L3?
6. **Sleep current (§3.3).** Is there anything in the schematic that would stop the whole board reaching ≤10 µA, such as leakage paths, PMIC GPIO defaults, or ESD parts?
7. **DFM and assembly at JLC.** Questions:
   - Custom footprints: the piezo (no land pattern in its datasheet), the side-view RGB LED, the DFN1610 TVS, and the Inpaq 2016 inductors. SOD-923 ESD parts.
   - The USB-C shell slots need a through-hole step.
   - The test pads carry signal-name references (two are both `GND`) and the three fiducials are all `REF**`. Will duplicate designators cause trouble in the CPL file?

## Things I already know about

- TH1 (the charge NTC) currently sits mid-board about 3.5 mm from the PMIC. My own rule says it should be at the board edge by the cell and away from heat sources. Is this worth moving now, or is it acceptable with a cold-threshold margin?
- The side-view LED's lens appears to face into the board rather than out of the nearby edge. I still need to settle the enclosure light path.
- Not yet measured: motor stall current, final gear ratio, final cell, and RF performance in the enclosure. R7, C14, and R10 are on 0603 pads so they can be swapped during bench calibration.

Thanks in advance. Even a quick "this looks wrong" on any of the above helps.
