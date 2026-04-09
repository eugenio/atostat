# atostat

A modular multichannel potentiostat with bipotentiostat capabilities, designed and built with [atopile](https://atopile.io).

## Key Specifications

- **Current range:** 10 nA to 1 A per channel (8 decades, switched-gain TIA)
- **Voltage range:** +/-2 V (bipolar DAC output via AD5693R + subtractor)
- **Channels:** 2 wired (architecture supports up to 8)
- **Bipotentiostat:** Yes (independent per-channel control)
- **Galvanic isolation:** 5 kV rated per channel (B0505S DC-DC + ISO1640 I2C + ISO7741 SPI/GPIO)
- **Modes:** Potentiostat and galvanostat (analog feedback path switching)
- **Connectivity:** WiFi, USB (ESP32-S3), CAN bus (SN65HVD230)
- **ADC:** Dual per channel -- ADS1256 (fast, 24-bit SPI) + NAU7802 (slow, 24-bit I2C)
- **DAC:** AD5693R (16-bit, I2C) with bipolar output stage
- **Sync:** Hardware trigger input, sync output, shared sync bus for multi-channel triggering

## Architecture Overview

The design is organized into three top-level modules wired together in `main.ato`:

**PowerSupply** -- Generates all voltage rails from a 5-12 V DC input.

- TPS63020 buck-boost converter (5 V analog rail)
- LDK220 LDO (3.3 V digital rail from 5 V)
- TPS60403 charge pump inverter (-5 V rail)

**ConnectivityGateway** -- Central coordinator built around an ESP32-S3.

- TCA9548A I2C multiplexer (8 channel fan-out)
- SPI bus with 74HC138 3-to-8 chip-select demux
- CAN bus transceiver (SN65HVD230) for multi-instrument sync
- External trigger input and sync pulse output

**PotentiostatChannel** (x2 wired) -- Fully isolated analog front end.

- Galvanic isolation: B0505S isolated DC-DC (custom no-Y-cap driver) + ISO1640 I2C isolator + ISO7741 SPI/GPIO isolator
- Control amplifier: AD8608 (quad rail-to-rail op-amp)
- Reference electrode buffer (high-impedance follower)
- Switched-gain TIA: MAX14778 analog mux selecting 8 gain resistor decades
- Mode switch: potentiostat/galvanostat feedback path selection
- Cell disconnect: TS5A22362 analog switch
- ADC interface: ADS1256 (SPI, fast sampling) + NAU7802 (I2C, precision)
- DAC waveform: AD5693R with bipolar subtractor output stage
- Per-channel RP2040 controller (placeholder -- see known issues)

## Module Listing

All custom modules live in `lib/`:

| File | Module | Description |
| ---- | ------ | ----------- |
| `power_supply.ato` | PowerSupply | Multi-rail PSU (5 V, 3.3 V, -5 V) |
| `connectivity.ato` | ConnectivityGateway | ESP32-S3 + I2C mux + SPI CS demux + CAN |
| `channel_module.ato` | PotentiostatChannel | Complete isolated channel assembly |
| `control_amplifier.ato` | ControlAmplifier | AD8608-based control amp with CE drive |
| `re_buffer.ato` | REBuffer | High-impedance reference electrode buffer |
| `switched_gain_tia.ato` | SwitchedGainTIA | 8-decade gain TIA (MAX14778 mux) |
| `adc_interface.ato` | ADCInterface | Dual ADC (ADS1256 + NAU7802) |
| `dac_waveform.ato` | DACWaveform | AD5693R DAC with bipolar output |
| `mode_switch.ato` | ModeSwitch | Potentiostat/galvanostat feedback selector |
| `galvanic_isolator.ato` | GalvanicIsolator | B0505S + ISO1640 I2C isolation |
| `spi_isolator.ato` | SPIIsolator | ISO7741 SPI/GPIO isolation |
| `channel_controller.ato` | ChannelController | Per-channel RP2040 (placeholder) |
| `spi_cs_demux.ato` | SPIChipSelectDemux | 74HC138 3-to-8 CS demux |
| `can_bus.ato` | CANBus | SN65HVD230 CAN transceiver |
| `eis_excitation.ato` | EISExcitation | EIS sine excitation (stub) |
| `ads1256.ato` | ADS1256 | ADS1256 24-bit ADC driver |
| `ad8608.ato` | AD8608 | AD8608 quad op-amp driver |
| `iso7741.ato` | ISO7741 | ISO7741 digital isolator driver |
| `b0505s_no_ycap.ato` | B0505S_NoYcap | B0505S DC-DC without Y-cap (DRC fix) |
| `tps60403.ato` | TPS60403 | TPS60403 charge pump inverter driver |
| `ferrite_bead.ato` | FerriteBead | Murata BLM15AG601SN1D ferrite bead |

## Dependencies

All hardware dependencies are from the atopile registry (see `ato.yaml`):

- `atopile/ti-tps63020` -- Buck-boost converter
- `atopile/st-ldk220` -- LDO regulator
- `atopile/espressif-esp32-s3` -- WiFi/BLE MCU
- `atopile/ti-tca9548a` -- I2C multiplexer
- `atopile/ti-iso1640x` -- Isolated I2C
- `atopile/adi-ad5693r` -- 16-bit DAC
- `atopile/nuvoton-nau7802` -- 24-bit ADC (I2C)
- `atopile/ti-ts5a22362` -- Analog switch
- `atopile/raspberry-rp2040` -- Channel controller MCU
- `ruben-iteng/analog-devices-max14778` -- Analog mux (TIA gain)
- `ruben-iteng/connectors` -- Board connectors
- `ruben-iteng/dcdc-converters` -- Isolated DC-DC

## Building

Requires [atopile](https://atopile.io) >= 0.15.3.

```bash
ato build
```

The build compiles `main.ato:App`, runs all 18 build stages, and picks 189 components. Output goes to `build/`.

## Current Status

**V0.4** -- Two channels fully wired with galvanic isolation, SPI isolation, and CS demux. The build passes cleanly.

### Known Issues

- **RP2040 placeholder:** The per-channel RP2040 controller is a stub. Full integration is blocked by an upstream atopile issue with `has_single_electric_reference_shared` traits in isolated power domains ([atopile/atopile#1812](https://github.com/atopile/atopile/issues/1812)).
- **Tracking issue:** [eugenio/atostat#1](https://github.com/eugenio/atostat/issues/1)

See [docs/V0.4_ROADMAP.md](docs/V0.4_ROADMAP.md) for the full roadmap including resolved and open items.

## License

MIT -- see [LICENSE.txt](LICENSE.txt).
