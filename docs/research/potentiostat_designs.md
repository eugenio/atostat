# Open-Source Potentiostat Design Research

## Atopile Packages Available

### DACs
| Package | Resolution | Interface | Notes |
| --- | --- | --- | --- |
| `adi-ad5693r` | 16-bit | I2C | Good for voltage control |
| `microchip-mcp4725` | 12-bit | I2C | Single channel |
| `microchip-mcp4728` | 12-bit | I2C | 4-channel |
| `ti-dac6578` / `ti-dac7578` | 12-bit | I2C | 8-channel |

### ADCs
| Package | Resolution | Interface | Notes |
| --- | --- | --- | --- |
| `nuvoton-nau7802` | **24-bit** | I2C | 2-ch differential, PGA, 23 ENOB -- **BEST FIT** |
| `microchip-mcp3421` | 18-bit | I2C | Single-ch, differential, 2.048V ref |
| `ti-ads1115` | 16-bit | I2C | 4-ch, PGA, delta-sigma |
| `ti-ads7830` | 8-bit | I2C | 8-ch, housekeeping only |

### Current/Power Sensing
| Package | Notes |
| --- | --- |
| `ti-ina228` | 85V, 20-bit I2C power monitor |
| `ti-ina232` | 48V, 16-bit |
| `ti-ina237` / `ti-ina238` | 85V, 16-bit |
| `ti-ina3221` | Triple-channel |
| `ti-ina185a1` | 26V bi-directional current sense amp |

### Analog Switches / Muxes
| Package | Notes |
| --- | --- |
| `ruben-iteng/analog-devices-max14778` | **Dual 4:1 mux, +-25V above/below rails** -- BEST for TIA gain switching |
| `ti-ts5a22362` | DPDT analog switch, low R_on |
| `ti-cd74hc4067sm` | 16-channel analog mux/demux |
| `ruben-iteng/logic (SN74CB3Q3251)` | 8:1 FET mux, fast, low-voltage |

### I2C Infrastructure
| Package | Notes |
| --- | --- |
| `ti-tca9548a` | 8-ch I2C mux (multichannel addressing) |
| `ti-iso1640x` | I2C isolator (channel isolation) |
| `adi-ltc4311` | I2C accelerator |
| `adi-ltc4316` | I2C address translator |

### MCUs
| Package | Notes |
| --- | --- |
| `espressif-esp32-s3` | WiFi/BLE |
| `raspberry-rp2040` | Dual-core Cortex-M0+ |
| `st-stm32h723` | High-perf Cortex-M7 |

### Power
| Package | Notes |
| --- | --- |
| `ti-tps63020` | Buck-boost, 1.8-5.5V in, 3A -- battery operation |
| `ruben-iteng/dcdc-converters (b0505s1wr3)` | **Isolated 5V-to-5V, 1W** -- per-channel isolation |
| `st-ldk220` | Adjustable LDO, 200 mA, low noise |
| `ti-tps82130` | 17V MicroSiP module, 2A, integrated inductor |
| `ti-tps563201` | 3A sync buck, SOT-23-6 |

### Additional Useful Packages
| Package | Notes |
| --- | --- |
| `ti-drv135` | Balanced line driver, bipolar supply (+-10V), force/sense pins |
| `ruben-iteng/connectors` | **Banana plugs** for electrode connections, QWIIC |
| `ruben-iteng/logic (ISO1540)` | I2C isolator, bidirectional |
| `ruben-iteng/relays` | Mechanical relays for high-current switching |
| `atopile/generics` | Op-amp topologies (VoltageFollower, InvertingAmplifier, DiffAmp, InstrumentationAmplifier, Integrator), R/C/L, VDiv, LPF, interfaces (Power, FloatPower, I2C, SPI, Analog, DiffPair) |
| `atopile/cell-sim` | 16-ch isolated cell simulator, closest architecture reference |

### Key Gaps -- Packages We Must Create
1. **Precision op-amps** -- No OPA188, OPA2277, AD8628, LMP7721, OPA129 etc. Must subclass generic `Opamp`
2. **Voltage references** -- No REF5025, LT1009, ADR4525 etc.
3. **High-res SPI ADCs** -- No ADS1256 (24-bit SPI), ADS1263 (32-bit), AD7177
4. **Potentiostat AFE ICs** -- No AD5940/AD5941, ADuCM355, LMP91000
5. **Bipolar/negative supply generators** -- No charge pump or inverting converter for -2V to -5V rail
6. **Digital potentiometers** -- `maxim-ds1841` and `maxim-ds3502` exist but unverified for this use

---

## Open-Source Potentiostat Projects

### Tier 1: Best Architecture References

#### tdstat (Thomas Dobbelaere)
- **Repo**: https://github.com/thomasdob/tdstat
- **ICs**: MAX5217 (16-bit DAC), MCP3422 (18-bit ADC), TL431 ref, PIC16F1459
- **Architecture**: Classic 3-electrode with relay-switched TIA feedback resistors (4 ranges)
- **Range**: +-6.144V measurable, ~$50 BOM
- **Key insight**: Relay-switched TIA gain resistor approach, detailed circuit wiki
- **Relevance**: Directly adaptable topology, extend to 8 decades

#### MYSTAT (Matthew Yates)
- **Repo**: https://github.com/matthew-yates/MYSTAT
- **Range**: +-200 mA, +-12V
- **Published**: HardwareX (peer-reviewed)
- **Key insight**: Potentiostat AND galvanostat modes
- **Relevance**: Best ref for higher current ranges, KiCad-based

#### DStat (University of Toronto)
- **Paper**: https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0140349
- **Key feature**: Picoampere current measurement, USB-powered
- **Relevance**: Best ref for low-noise nA measurement path

#### Rodeostat (IO Rodeo)
- **Repo**: https://github.com/iorodeo/potentiostat
- **ICs**: Teensy 3.2, 12-bit DAC, 16-bit ADC
- **Range**: +-1/10/100/1000 uA (4 ranges), +-1/2/5/10V
- **Key insight**: JSON API, expansion headers, modular
- **Relevance**: Software API model, 4-range architecture

#### PassStat
- **Paper**: https://www.hardware-x.com/article/S2468-0672(22)00035-9/fulltext
- **Performance**: <1 pA noise, 8000 V/s scan rate, ohmic drop compensation
- **Cost**: ~10 EUR analog board + Teensy
- **Relevance**: Best noise performance of any open-source design

### Tier 2: Integrated AFE Approach (AD5940/AD5941)

#### FreiStat-IoT (University of Freiburg)
- **Repo**: https://github.com/IMTEK-FreiStat/FreiStat-IoT
- **ICs**: AD5941 + Adafruit Feather M0 WiFi
- **Features**: CV, LSV, amperometry, EIS
- **Published**: Analytical Chemistry (2023)
- **Relevance**: Demonstrates AD5941 as per-channel AFE

#### HunStat2
- **Repo**: https://github.com/hunstat2/HunStat2
- **ICs**: AD5941 + XIAO RP2040
- **Cost**: ~$30, <1 hour assembly
- **Relevance**: Simplest AD5941 design reference

#### ACEstat
- **Paper**: ACS Analytical Chemistry 2022
- **ICs**: ADuCM355 (AD5940 AFE + ARM Cortex-M3 on one chip)
- **Relevance**: Per-channel autonomous measurement node concept

### Tier 3: Specialized

| Project | Key Feature | Relevance |
| --- | --- | --- |
| CheapStat | Pioneer open-source potentiostat, ~$80 | Educational baseline |
| KickStat | LMP91000 coin-sized, 4.5 nA LOD | Integrated IC approach (too limited) |
| HELPStat | Handheld EIS (200kHz-0.15Hz), BLE | Most recent (2024) with onboard EIS fitting |
| NanoStat | ESP32, 2 ICs total, web UI, <$25 | Minimal viable potentiostat |
| PSoC-Stat | Cypress PSoC 5LP programmable analog | Interesting but platform-locked |
| PocketEC | Platform-agnostic app, multi-AFE support | Software architecture reference |

### Tier 4: High-Performance DAQ

#### Open COVG DAQ
- **Repo**: https://github.com/lucask07/open_covg_daq_pcb
- **ICs**: 4x AD7961 (16-bit 5 MSPS), ADS8686, 6x AD5453, OpalKelly FPGA
- **Key insight**: Standardized HDMI daughter-card interface, <2 us feedback
- **Relevance**: Modular daughter-card concept applicable to multichannel

### Multichannel Potentiostat Array (PLOS ONE 2021)
- **Paper**: https://pmc.ncbi.nlm.nih.gov/articles/PMC8445456/
- **Architecture**: Stackable boards, 8-64 channels at ~$8/channel
- **Range**: +-1.5 uA per channel (low-current only)
- **Relevance**: Key reference for stackable multichannel architecture

---

## Key ICs for Our Design

| IC | Type | Key Specs | Role |
| --- | --- | --- | --- |
| AD5940/AD5941 | Electrochemical AFE | Dual excitation, potentiostat+EIS, sequencer | Per-channel low-current AFE |
| ADuCM355 | AFE + MCU | AD5940-class + ARM Cortex-M3 | Per-channel autonomous node |
| AD8608 | Quad CMOS op-amp | R-R I/O, 1 pA bias, low noise | RE buffer, TIA, control amp |
| MAX9913 | Precision op-amp | Femtoamp input bias | Ultra-low-current TIA (nA) |
| OPA2277 | Dual precision op-amp | 10 uV offset, 0.1 uV/C drift | Differential voltage measurement |
| ADS1256 | 24-bit delta-sigma ADC | 30 kSPS, PGA 1-64x | Per-channel digitization |
| LTC2500-32 | 32-bit SAR ADC | 148 dB dynamic range | Calibration reference |
| MAX5217 | 16-bit DAC | SPI, low-glitch | Waveform generation |
| LMP91000 | Integrated potentiostat | I2C, programmable TIA | Too limited for our specs |

---

## Analog Design Patterns

### Control Amplifier (CE Driver)
- Inverting config: RE feedback on inv input, V_set on non-inv
- Cell impedance (Cdl, Rs, Rct) enters feedback loop -- stability critical
- Include RC compensation (tdstat: R21/C7 at 200 Hz rolloff + snubber R9/C8)
- Consider programmable compensation for different cell types

### Electrometer / RE Buffer
- Unity-gain, zero current draw from RE
- Must have GOhm-TOhm input impedance, pA-or-less bias current
- JFET/CMOS input mandatory: AD8608 (1 pA typ), OPA2277

### Switched-Gain TIA (WE Current Measurement)
- V_out = -I_cell x R_f
- 8 decades (10 nA to 1 A) requires ~8 feedback resistors:
  - 200 MOhm, 20 MOhm, 2 MOhm, 200 kOhm, 20 kOhm, 2 kOhm, 200 Ohm, 20 Ohm
- Use analog switches in feedback path (TS5A22362 already in atopile)
- Noise-critical at highest gain (200 MOhm): thermal noise + op-amp input noise
- Power-critical at lowest gain (20 Ohm): dissipation + op-amp output current

### Guard Rings (nA-Level Measurement)
- Surround high-Z traces with driven guard at same potential
- Both PCB sides, continuous rectangles, no gaps
- For TIA input: guard driven by virtual ground
- Layout concern in KiCad post-atopile netlist generation

### Bipotentiostat Architecture
- Two WE channels sharing one CE driver + RE buffer
- Each WE has independent TIA, DAC (potential offset), ADC
- Cross-talk prevention: careful routing, differential measurement
- Maps cleanly to modular per-channel design

### Multichannel Architecture Options
1. **Independent channels**: Each channel = complete potentiostat (CE, RE, WE). No crosstalk.
2. **Shared CE/RE, multiple WEs**: One control amp, one RE buffer, multiple TIA channels. Requires galvanic isolation.
3. **Multiplexed**: One potentiostat, analog mux switches cells. Not simultaneous.

**Recommended**: Hybrid -- independent isolated channel modules that CAN share CE/RE for bipotentiostat mode.

---

## Recommended Architecture

### Hybrid Approach
- **AD5941 as per-channel low-current AFE** (10 nA to ~100 uA) with free EIS
- **Discrete op-amp circuitry** for high-current range (100 uA to 1 A)
- **Galvanic isolation** per channel (isolated DC-DC + digital isolators)
- **Central controller** (STM32H7 or RP2040) coordinates via isolated SPI

### Atopile Modules to Create
1. `control-amplifier` -- CE driver with programmable compensation
2. `re-buffer` -- High-impedance RE voltage follower
3. `switched-gain-tia` -- 8-range TIA with analog switch selection
4. `current-range-selector` -- Switch bank + logic for auto-ranging
5. `adc-interface` -- 24-bit ADC with PGA
6. `dac-waveform-gen` -- 16-bit DAC for cell voltage control
7. `channel-module` -- Complete single-WE potentiostat channel
8. `bipotentiostat-controller` -- Shared CE/RE + dual WE coordination
9. `galvanic-isolator` -- Power + signal isolation per channel
10. `power-supply` -- Bipolar rails (+-2V minimum, likely +-5V internal)
11. `channel-controller` -- Per-channel MCU with Ethernet/USB/WiFi
12. `backplane-comms` -- Inter-channel bus + host interface

---

## Connectivity & Software Architecture

### Requirement
Each channel or bipotentiostat pair must be individually readable and controllable via
Ethernet, USB, or WiFi. The software must be scriptable and have a polished GUI.

### Hardware Connectivity Options

#### Option A: Per-Channel MCU with Network Stack (Recommended)
- Each channel board carries an ESP32-S3 (WiFi+USB) or W5500/W5100S (Ethernet)
- Each channel gets its own IP address or USB endpoint
- Channels are independently discoverable via mDNS/Zeroconf
- Central host aggregates channels but is not required for single-channel use
- **Pros**: True independence, any channel works standalone, natural scaling
- **Cons**: Higher per-channel BOM cost (~$3-5 for ESP32-S3)

#### Option B: Central Controller with Per-Channel Addressing
- Single powerful MCU/SBC (STM32H7, RP2040+W5500, or RPi Pico W) per chassis
- Internal SPI/I2C bus to channel analog boards
- Host exposes per-channel API endpoints via multiplexed addressing
- **Pros**: Lower per-channel cost, simpler channel boards
- **Cons**: Single point of failure, limited by bus bandwidth for many channels

#### Option C: Hybrid (Best of Both)
- Each channel has a lightweight MCU (RP2040) for real-time analog control
- A central gateway MCU/SBC (ESP32-S3 or RPi) provides Ethernet/WiFi/USB
- Internal bus (SPI or CAN) between gateway and channel MCUs
- Gateway exposes per-channel REST/WebSocket/SCPI endpoints
- **Pros**: Real-time performance on channel + network flexibility on gateway
- **Cons**: Two MCU types to firmware

### MCU Candidates (atopile packages available)

| MCU | Package | Ethernet | WiFi | USB | Notes |
| --- | --- | --- | --- | --- | --- |
| ESP32-S3 | `espressif-esp32-s3` | Via W5500 | Built-in | USB-OTG | Best WiFi option |
| RP2040 | `raspberry-rp2040` | Via W5500 | No | USB 1.1 | Best for real-time analog control |
| STM32H723 | `st-stm32h723` | Built-in MAC | No | USB-OTG | Best for Ethernet-native, high perf |
| ESP32-C3 | `espressif-esp32-c3` | No | Built-in | USB-JTAG | Low-cost WiFi option |

### Software Stack

```
+-----------------------------------------------+
|              Polished GUI (Desktop/Web)        |
|  PyQt6 / Electron / SvelteKit                 |
|  - Per-channel dashboard with live plots       |
|  - Experiment builder (drag-and-drop)          |
|  - Data export (CSV, HDF5, MDAT)              |
+-----------------------------------------------+
|              Scripting Layer                    |
|  Python SDK (atostat-py)                       |
|  - channel = Atostat.connect("192.168.1.10")  |
|  - channel.set_potential(0.5)                  |
|  - data = channel.run_cv(-0.5, 0.5, 0.1)     |
|  - Jupyter notebook integration               |
+-----------------------------------------------+
|              Protocol Layer                     |
|  REST API / WebSocket / SCPI over TCP          |
|  - GET /channels/{id}/status                   |
|  - POST /channels/{id}/experiment              |
|  - WS /channels/{id}/stream (real-time data)   |
|  - SCPI: :CHANnel1:POTential 0.5              |
+-----------------------------------------------+
|              Firmware                           |
|  Per-channel MCU (ESP-IDF / RP2040 SDK)        |
|  - Real-time control loop (1-100 kHz)          |
|  - Auto-ranging state machine                  |
|  - Data buffering + streaming                  |
|  - mDNS/Zeroconf discovery                     |
+-----------------------------------------------+
```

### Software Reference Projects
| Project | Approach | Relevance |
| --- | --- | --- |
| Rodeostat | JSON API over USB, Python lib, web GUI | API design model |
| FreiStat-IoT | WiFi + cloud, Arduino firmware | IoT pattern |
| PocketEC/Voyager | Multi-AFE platform-agnostic app | Software abstraction |
| NanoStat | Self-hosted web UI on ESP32 | Embedded web server |
| HELPStat | BLE + standalone, onboard fitting | Edge computing |
| EmStat Pico | MethodSCRIPT scripting language | Domain-specific scripting |

### Key Software Design Decisions
1. **Protocol**: REST + WebSocket for modern tooling; optional SCPI for lab integration
2. **Discovery**: mDNS (e.g., `channel-01.atostat.local`) for zero-config networking
3. **Scripting**: Python SDK as primary, with Jupyter integration for interactive experiments
4. **GUI**: Web-based (SvelteKit or similar) served from gateway, or desktop PyQt6
5. **Data format**: Streaming via WebSocket for real-time plots; HDF5/CSV for archival
6. **Firmware**: FreeRTOS on ESP32-S3 or bare-metal on RP2040 for deterministic timing
7. **Calibration**: Per-channel calibration data stored in flash, queryable via API
