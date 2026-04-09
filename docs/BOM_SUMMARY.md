# Atostat Bill of Materials Summary

Build configuration: 2 channels (from `main.ato:App`).
Total component count: ~189 (from ato build output).

---

## 1. Key ICs

### System-Level ICs (1 per system)

| Part Number | Function | Package | Qty | Notes |
|---|---|---|---|---|
| ESP32-S3 | WiFi/USB gateway MCU | QFN-56 | 1 | atopile registry `atopile/espressif-esp32-s3 v0.3.0` |
| TCA9548A | I2C 8-channel multiplexer | TSSOP-24 | 1 | atopile registry `atopile/ti-tca9548a v0.2.0` |
| SN65HVD230 | CAN bus transceiver (1 Mbps) | SOIC-8 | 1 | Local package `parts/Texas_Instruments_SN65HVD230DR` |
| SN74HC138 | 3-to-8 SPI CS demux | TSSOP-16 | 1 | Local package `parts/Texas_Instruments_SN74HC138PWR` |
| TPS63020 | Buck-boost 5V regulator | QFN-14 | 1 | atopile registry `atopile/ti-tps63020 v0.4.0` |
| LDK220 | 3.3V LDO (system rail) | SOT-23-5 | 1 | atopile registry `atopile/st-ldk220 v0.3.0` |
| TPS60403 | Charge pump inverter (-5V) | SOT-23-5 | 1 | Local package `parts/Texas_Instruments_TPS60403DBVR` |

### Per-Channel ICs (multiply by number of channels)

| Part Number | Function | Package | Qty/Ch | Notes |
|---|---|---|---|---|
| RP2040 | Per-channel real-time controller | QFN-56 | 1 | Placeholder (`has_part_removed` trait); atopile registry `atopile/raspberry-rp2040 v0.2.5` |
| ADS1256 | Fast 24-bit SPI ADC (30 kSPS) | SSOP-28 | 1 | Local package `parts/Texas_Instruments_ADS1256IDBR` |
| NAU7802 | Slow 24-bit I2C ADC (320 SPS) | SOP-16 | 1 | atopile registry `atopile/nuvoton-nau7802 v0.4.0` |
| AD5693R | 16-bit I2C DAC | MSOP-10 | 1 | atopile registry `atopile/adi-ad5693r v0.2.3` |
| AD8608 | Quad precision CMOS op-amp | TSSOP-14 | 4 | `parts/Analog_Devices_AD8608ARUZ`; used in: TIA (x1), control amp (x1), RE buffer (x1), DAC subtractor (x1) |
| MAX14778 | Dual 4:1 analog mux (TIA gain) | TSSOP-16 | 2 | atopile registry `ruben-iteng/analog-devices-max14778 v0.1.2`; mux A (ranges 0-3) + mux B (ranges 4-7) |
| ISO7741 | Quad digital isolator (5 kV) | SSOP-16 | 2 | `parts/Texas_Instruments_ISO7741DBQR`; 1x GPIO isolator + 1x SPI isolator |
| ISO1640 | I2C digital isolator | SOIC-8 | 1 | atopile registry `atopile/ti-iso1640x v0.3.0` |
| B0505S-1WR3 | 5V-5V isolated DC-DC (1W) | SIP-4 | 1 | atopile registry `ruben-iteng/dcdc-converters v0.2.2`; Y-cap removed variant |
| LDK220 | 3.3V LDO (isolated analog) | SOT-23-5 | 1 | atopile registry `atopile/st-ldk220 v0.3.0`; post-isolation clean supply |
| TS5A22362 | Dual SPDT analog switch | VSSOP-8 | 5 | atopile registry `atopile/ti-ts5a22362 v0.2.1`; cell disconnect (x1), mode switch (x1), TIA bank select (x1), control amp comp (x2) |
| CD74HC4067 | 16:1 analog mux | TSSOP-24 | 0 | atopile registry `atopile/ti-cd74hc4067sm v0.1.1`; listed in deps but **not instantiated** in current design (TIA uses MAX14778 instead) |

### IC Count Summary (2-channel build)

| Category | Count |
|---|---|
| System-level ICs | 7 |
| Per-channel ICs (x2) | 38 (19 per channel) |
| **Total ICs** | **45** |

Note: RP2040 has `has_part_removed` trait (placeholder). The physical RP2040 module (crystal, flash, decoupling) is intended as a daughter-board or will be resolved when atopile trait issues are fixed.

---

## 2. Passive Components Estimate

Counted from .ato source files. Excludes passives internal to registry packages (ESP32-S3, TCA9548A, etc. may bring their own).

### Per-Channel Passives

| Module | Resistors | Capacitors | Inductors/Ferrites |
|---|---|---|---|
| SwitchedGainTIA | 13 | 8 | 0 |
| ControlAmplifier | 5 | 6 | 0 |
| REBuffer | 1 | 3 | 0 |
| DACWaveform | 6 | 2 | 0 |
| ADCInterface | 4 | 0 | 0 |
| ADS1256 driver | 7 | 3 | 0 |
| AD8608 (x4) | 0 | 8 | 0 |
| ISO7741 (x2) | 4 | 4 | 0 |
| GalvanicIsolator | 0 | 8 | 2 |
| B0505S driver | 0 | 3 | 2 |
| ModeSwitch | 1 | 0 | 0 |
| ChannelController | 1 | 1 | 0 |
| ChannelModule (top) | 0 | 2 | 0 |
| **Per-channel total** | **~42** | **~48** | **~4** |

### System-Level Passives

| Module | Resistors | Capacitors | Inductors/Ferrites |
|---|---|---|---|
| ConnectivityGateway | 3 | 3 | 1 |
| CANBus | 2 | 1 | 0 |
| SPIChipSelectDemux | 0 | 1 | 0 |
| PowerSupply | 0 | 6 | 0 |
| TPS60403 driver | 0 | 3 | 0 |
| **System total** | **~5** | **~14** | **~1** |

### Grand Total Passives (2-channel build)

| Type | Count |
|---|---|
| Resistors | ~89 |
| Capacitors | ~110 |
| Inductors/Ferrites | ~9 |
| **Total passives** | **~208** |

Note: The 189 total from the ato build includes ICs + passives and may differ because registry packages resolve to specific picked parts. Some passives may be merged by the solver if they share the same net and value.

---

## 3. JLCPCB Assembly Notes

### JLCPCB Basic Library (low/no extra fee)

These parts are commonly stocked and incur no extended-part surcharge:

- 0402/0603/0805 resistors (all standard E96 values used)
- 0402/0603/0805 ceramic capacitors (100nF, 1uF, 10uF, 100uF, 4.7pF, 47pF, 470pF, 1nF, 4.7nF, 2.2uF, 22uF)
- Ferrite beads: Murata BLM15AG601SN1D (LCSC C76884) -- basic
- SN65HVD230DR (CAN transceiver) -- basic
- SN74HC138PWR (3-to-8 decoder) -- basic
- LDK220M-R (SOT-23-5 LDO) -- basic

### JLCPCB Extended Library (per-part surcharge applies)

These parts require the extended library fee (~$3 per unique extended part):

| Part | Reason |
|---|---|
| ESP32-S3-WROOM-1 | Specialty MCU module |
| RP2040 | Specialty MCU (placeholder, not placed yet) |
| ADS1256IDBR | Precision ADC, specialty |
| NAU7802SGI | 24-bit ADC, specialty |
| AD5693RACPZ | 16-bit DAC, specialty |
| AD8608ARUZ | Quad precision op-amp, specialty |
| MAX14778ETE+ | Dual 4:1 analog mux, specialty |
| ISO7741DBQR | Quad digital isolator, specialty |
| ISO1640QDWRQ1 | I2C isolator, specialty |
| TPS63020DSJR | Buck-boost regulator, specialty |
| TPS60403DBVR | Charge pump, may be basic depending on stock |
| B0505S-1WR3 | Isolated DC-DC, specialty (may need manual placement) |
| TS5A22362DGSR | Dual SPDT switch, extended |
| CD74HC4067SM96 | 16:1 mux, extended (not used in current design) |

### Assembly Considerations

- **B0505S-1WR3**: SIP-4 through-hole package. JLCPCB SMT assembly does not support THT by default; may require hand-soldering or their THT service add-on.
- **RP2040**: Currently a placeholder (`has_part_removed`). Not assembled; intended as a future daughter-board or direct placement once atopile trait resolution is fixed.
- **100 MOhm resistor (TIA range 0)**: 0805 package, 5% tolerance. Verify JLCPCB stock for this value; high-value resistors may have limited availability.
- **20 MOhm / 10 MOhm resistors**: 0603 package. Check extended library availability.
- **Precision 0.1% resistors (10k)**: Used in DAC subtractor. Typically extended parts on JLCPCB.
- **Dual-supply operation**: The design uses +5V / -5V (via TPS60403) for analog op-amps. Ensure correct polarity marking in assembly files.

### Estimated Extended Part Surcharge

With ~12 unique extended parts at ~$3 each = ~$36 per assembly run (one-time, not per board).
