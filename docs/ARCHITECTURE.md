# atostat Architecture Overview

Modular multichannel potentiostat/bipotentiostat. Two independently isolated
channels, each capable of potentiostat and galvanostat modes. Current range
10 nA -- 100 mA full-scale, cell voltage +/-2.5 V.

Written in [atopile](https://atopile.io) hardware description language.

---

## 1. High-Level Block Diagram

```
                          +-------------------+
   5-12 V DC in --------->|   PowerSupply     |
                          |  TPS63020 buck-   |
                          |  boost -> 5 V     |
                          |  LDK220 LDO -> 3.3V
                          |  TPS60403 -> -5 V |
                          +---+---+---+-------+
                              |   |   |
               +--------------+   |   +--------------+
               |  5 V             | 3.3 V             |  5 V
               v                  v                   v
   +-----------+------+   +------+--------+   +------+-----------+
   | PotentiostatCh 0 |   | Connectivity  |   | PotentiostatCh 1 |
   | (isolated)       |   | Gateway       |   | (isolated)        |
   +------------------+   | (ESP32-S3)    |   +-------------------+
                          +---------------+

   Each channel receives host 5 V for its B0505S isolated DC-DC.
   The gateway runs on 3.3 V.
   Host 3.3 V also powers the SPI isolator host side on each channel.
```

---

## 2. Per-Channel Signal Flow

Each `PotentiostatChannel` contains the complete analog front-end on the
isolated side of a galvanic barrier.

```
                         ISOLATED SIDE
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  AD5693R DAC (I2C, 16-bit)                                  │
  │      |                                                       │
  │      | 0-5 V unipolar                                        │
  │      v                                                       │
  │  Switchable RC filter (1 kOhm + 100 nF, fc ~1.6 kHz)       │
  │      |                                                       │
  │      v                                                       │
  │  Bipolar subtractor (V_dac - V_ref = +/-2.5 V)              │
  │      |                                                       │
  │      v                                                       │
  │  Control Amplifier (AD8608 sect. A)                          │
  │      |  ^                                                    │
  │      |  | feedback (re_input)                                │
  │      |  |                                                    │
  │      |  ModeSwitch (TS5A22362 SPDT)                         │
  │      |    |potentiostat        |galvanostat                  │
  │      |    |                    |                              │
  │      |  RE Buffer            TIA output                      │
  │      |  (AD8608 sect. A)     (current_output)                │
  │      |    ^                                                  │
  │      |    |                                                  │
  │      v    |                                                  │
  │  CE ←[cell switch]← ctrl_amp    RE ──────→ RE Buffer        │
  │                                                              │
  │  WE ←[cell switch]→ TIA input                               │
  │                      (AD8608 sect. A)                        │
  │                      8-range switched gain                   │
  │                      (2x MAX14778 + TS5A22362 bank select)   │
  │                         |                                    │
  │                         v                                    │
  │               ┌─────────────────────┐                        │
  │               │ ADC Interface       │                        │
  │               │  CH1: I_cell (TIA)  │                        │
  │               │  CH2: V_cell (WE-RE)│                        │
  │               │                     │                        │
  │               │ Fast: ADS1256       │                        │
  │               │  24-bit, 30 kSPS    │                        │
  │               │  SPI to RP2040      │                        │
  │               │                     │                        │
  │               │ Slow: NAU7802       │                        │
  │               │  24-bit, 320 SPS    │                        │
  │               │  I2C to RP2040      │                        │
  │               └─────────────────────┘                        │
  │                                                              │
  │  RP2040 controller ← SPI to ADS1256, I2C to DAC + NAU7802   │
  │      | gain select GPIOs -> TIA mux address lines            │
  │      | cell_enable, mode_select GPIOs                        │
  └──────┼───────────────────────────────────────────────────────┘
         | isolated SPI (through ISO7741 SPI isolator)
         |
   ══════╪════ ISOLATION BARRIER ════════════════════════
         |
      HOST SIDE
```

### ADC signal routing

| ADC Channel | Positive Input        | Negative Input     | Measures          |
|-------------|-----------------------|--------------------|-------------------|
| CH1         | TIA current_output    | Isolated GND       | Cell current      |
| CH2         | WE potential (TIA Vg) | Buffered RE        | Applied cell voltage (V_WE - V_RE) |

---

## 3. Isolation Boundary

Every channel has a galvanic isolation barrier implemented by `GalvanicIsolator`.

### Host side (non-isolated)

- ESP32-S3 gateway (WiFi, USB, CAN, trigger/sync)
- TCA9548A I2C mux (fallback path)
- SN74HC138 SPI CS demux
- PowerSupply rails: 5 V, 3.3 V, -5 V
- Host side of each ISO7741 SPI isolator (powered from host 3.3 V)

### Isolated side (per channel)

- B0505S 1 W isolated DC-DC (5 V host -> 5 V isolated)
- LDK220 LDO (5 V iso -> 3.3 V clean analog, AVDD)
- Pi-filter ferrite (3.3 V analog -> 3.3 V digital, DVDD)
- RP2040 real-time controller
- All analog front-end: DAC, control amp, RE buffer, mode switch, TIA, ADCs
- Cell disconnect switches (TS5A22362), electrode connections

### Signals crossing the barrier

| Signal     | Direction        | Isolator           | Notes                              |
|------------|------------------|--------------------|------------------------------------|
| SPI (SCLK, MOSI, CS) | Host -> Isolated | ISO7741 (SPI iso) | 3 of 4 channels A->B |
| SPI (MISO)            | Isolated -> Host | ISO7741 (SPI iso) | 1 channel B->A       |
| I2C (SDA, SCL)        | Bidirectional    | ISO1640            | Fallback path        |
| GPIO out x3           | Host -> Isolated | ISO7741 (GPIO iso) | TIA gain select (legacy) |
| DRDY                  | Isolated -> Host | ISO7741 (GPIO iso) | ADS1256 data ready   |
| Power                 | Host -> Isolated | B0505S DC-DC       | 5 V, 1 W, no Y-cap  |

---

## 4. Communication Topology

### Primary: SPI (high-speed data path)

```
ESP32-S3  ──SPI bus──┬──[74HC138 CS demux]──CS[0]──[ISO7741]──RP2040 ch0
  GPIO IO10 = master CS    │                                     |
  GPIO IO21/38/39 = addr   ├──CS[1]──[ISO7741]──RP2040 ch1      |
                           ├──CS[2..7] (expansion)               |
                           :                                     |
                                                    RP2040 ──SPI0──> ADS1256
                                                    RP2040 ──I2C0──> DAC + NAU7802
```

The 74HC138 decoder is gated by the ESP32 master CS: when CS goes low the
addressed output goes low, selecting exactly one channel. Address is latched
before CS assertion. Supports up to 8 channels.

Each channel RP2040 uses two SPI peripherals:
- SPI1 (slave): host communication through ISO7741
- SPI0 (master): local ADS1256 fast ADC

### Fallback: I2C (low-speed, legacy)

```
ESP32-S3  ──I2C──> TCA9548A mux ──ch[0]──[ISO1640]──> isolated I2C bus ch0
                                  ──ch[1]──[ISO1640]──> isolated I2C bus ch1
                                  ──ch[2..7] (expansion)
```

The TCA9548A fans out the master I2C to 8 channels. Each channel's isolated
I2C connects to AD5693R DAC (addr 0x0C) and NAU7802 ADC (addr 0x2A) --
no per-channel address mux needed since the addresses differ.

### Synchronization

| Bus        | IC          | Purpose                               |
|------------|-------------|---------------------------------------|
| CAN        | SN65HVD230  | Multi-board synchronization, 1 Mbps   |
| Sync bus   | GPIO + 100R | Hardware trigger line, all channels    |
| Trigger in | GPIO IO16   | External instrument trigger input      |
| Sync out   | GPIO IO17 + 100R | Sync pulse to external instruments |

---

## 5. Power Domains

### System-level rails (PowerSupply)

| Rail     | Source                 | Voltage       | Purpose                        |
|----------|------------------------|---------------|--------------------------------|
| Host 5 V | TPS63020 buck-boost   | 5.0 V +/- 3%  | Channel DC-DC input, op-amp supply |
| Host 3.3 V | LDK220 LDO from 5 V | 3.3 V +/- 3% | ESP32-S3, I2C mux, SPI CS demux, SPI iso host side |
| Host -5 V | TPS60403 charge pump  | -5.0 V (magnitude 4.5-5.5 V) | Negative op-amp rail (system-level) |

### Per-channel isolated rails (GalvanicIsolator)

| Rail              | Source                          | Voltage       | Purpose                              |
|-------------------|---------------------------------|---------------|--------------------------------------|
| Iso 5 V raw       | B0505S DC-DC from host 5 V     | 5.0 V nominal | Op-amp supply, charge pump input     |
| Iso 3.3 V analog (AVDD) | LDK220 LDO from iso 5 V | 3.3 V +/- 3% | ADS1256 AVDD, NAU7802                |
| Iso 3.3 V digital (DVDD) | Pi-filter from AVDD     | 3.3 V         | RP2040, ISO7741 iso side, ISO1640 iso side |

The AVDD/DVDD separation uses a pi-filter (10 uF -- ferrite 600 Ohm@100 MHz -- 10 uF)
to prevent digital switching noise from reaching the ADC analog supply.

Note: Isolated -5 V rail for bipolar op-amp operation is planned but not yet
instantiated due to an atopile solver limitation with the TPS60403 inverted
power model.

---

## 6. Key Component Choices

| Component | Part | Why |
|-----------|------|-----|
| **Gateway MCU** | ESP32-S3 | WiFi + USB + BLE in a single module; sufficient GPIO for SPI master + CS demux address + CAN + sync. |
| **Channel MCU** | RP2040 | Dual-core for deterministic real-time measurement loop; PIO blocks enable precise SPI timing; low cost. |
| **Isolated DC-DC** | B0505S-1WR3 (no Y-cap) | 1 W isolated DC-DC in SIP-4; Y-cap removed to prevent common-mode leakage that degrades isolation for electrochemical cells. |
| **I2C isolator** | ISO1640 (TI) | Bidirectional I2C isolation with integrated pull-ups; automotive-grade (Q1 variant). |
| **SPI/GPIO isolator** | ISO7741 (TI) | 4-channel digital isolator, 3+1 directional split matches SPI topology (SCLK/MOSI/CS out, MISO in). |
| **SPI CS demux** | SN74HC138 | Classic 3-to-8 decoder; active-low outputs match SPI CS convention; gated by master CS for clean enable/disable. |
| **I2C mux** | TCA9548A | 8-channel I2C mux for fallback addressing; each channel gets its own isolated I2C segment. |
| **Control amp op-amp** | AD8608 (quad) | Rail-to-rail I/O, 10 MHz GBW, 0.065 uV/C drift, low noise (8 nV/rtHz); quad package shares across control amp, RE buffer, subtractor, TIA sections. |
| **Fast ADC** | ADS1256 | 24-bit delta-sigma, 30 kSPS, SPI; suitable for CV, SWV, DPV at scan rates up to ~1 V/s. |
| **Slow ADC** | NAU7802 | 24-bit, 320 SPS, I2C; low-power precision for OCP and long chronoamperometry experiments. |
| **DAC** | AD5693R | 16-bit, I2C, internal 2.5 V reference with 2x gain (0-5 V output); bipolar output achieved via external subtractor. |
| **TIA gain mux** | MAX14778 (dual 4:1) | Low on-resistance (< 10 Ohm), low charge injection analog mux; two units give 8 gain ranges spanning 7 decades. |
| **Cell disconnect / mode / filter / comp switches** | TS5A22362 | Dual SPDT analog switch, low Ron (~1 Ohm), rail-to-rail; used throughout for cell disconnect, mode select, filter bypass, and compensation network switching. |
| **Buck-boost (5 V)** | TPS63020 | Wide input 2-12 V to 5 V, 96% efficiency, 4 A switch current; handles both USB 5 V and barrel-jack inputs. |
| **LDO (3.3 V)** | LDK220 | Low noise (6.5 uVrms), high PSRR (>70 dB @ 100 kHz), 200 mA; used both system-level and per-channel for clean analog supply. |
| **Charge pump (-5 V)** | TPS60403 | Unregulated inverter, VIN -> -VIN; minimal external components (2 caps). |
| **RE buffer** | AD8608 section + 10 MOhm input R | Ultra-high input impedance prevents current draw from reference electrode; 10 MOhm limits fault current to sub-uA. Guard ring driver on section B. |

---

## 7. Module Inventory

| File | Module | Role |
|------|--------|------|
| `main.ato` | `App` | Top-level: PSU, gateway, 2 channels, electrode connectors, trigger/sync |
| `lib/power_supply.ato` | `PowerSupply` | 5 V buck-boost, 3.3 V LDO, -5 V charge pump |
| `lib/connectivity.ato` | `ConnectivityGateway` | ESP32-S3, TCA9548A I2C mux, SPI CS demux, CAN, sync |
| `lib/channel_module.ato` | `PotentiostatChannel` | Full isolated channel: isolator + controller + AFE |
| `lib/galvanic_isolator.ato` | `GalvanicIsolator` | B0505S DC-DC, ISO1640 I2C iso, ISO7741 GPIO iso, AVDD/DVDD split |
| `lib/spi_isolator.ato` | `SPIIsolator` | ISO7741-based isolated SPI (SCLK/MOSI/CS out, MISO in) |
| `lib/channel_controller.ato` | `ChannelController` | RP2040 placeholder, GPIO mapping, SPI/I2C peripherals |
| `lib/control_amplifier.ato` | `ControlAmplifier` | Error amplifier driving CE, 3 switchable RC comp networks |
| `lib/re_buffer.ato` | `REBuffer` | Unity-gain follower for RE, guard ring driver |
| `lib/mode_switch.ato` | `ModeSwitch` | SPDT selecting RE buffer or TIA feedback (pot/galv mode) |
| `lib/switched_gain_tia.ato` | `SwitchedGainTIA` | 8-range TIA (10 nA -- 100 mA FS), 2x MAX14778 + bank switch |
| `lib/adc_interface.ato` | `ADCInterface` | Dual ADC: ADS1256 (SPI, fast) + NAU7802 (I2C, slow) |
| `lib/dac_waveform.ato` | `DACWaveform` | AD5693R + RC filter + bipolar subtractor (+/-2.5 V out) |
| `lib/spi_cs_demux.ato` | `SPIChipSelectDemux` | 74HC138 3-to-8 decoder for per-channel SPI CS |
