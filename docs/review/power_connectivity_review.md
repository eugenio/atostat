# Power, Isolation & Connectivity Architecture Review

## P0 Blockers

### 1. No Bipolar Supply (shared with analog review)
Single 5V rail cannot produce negative potentials. Cannot do reduction chemistry.
- **Fix**: Add TPS60403 charge pump (-5V, 60 mA) or replace B0505S with dual-output isolated converter (Murata MEE3S0505SC: +5V/-5V). Virtual ground at 2.5V is a cheaper workaround but halves dynamic range.

### 2. No Post-Isolation Analog Filtering
B0505S switching ripple (50-100 mV_pp at 100-300 kHz) goes directly to ADC AVDD. Limits effective resolution to ~14 bits, wasting the NAU7802's 24-bit capability.
- **Fix**: Add low-noise LDO (TPS7A4700 or LDK220) between B0505S and analog components. PSRR >70 dB at 100 kHz reduces ripple to <30 uV.

### 3. TIA Gain Select Lines Floating (shared with analog review)
No connection from gain_select[4] to any controller. Cannot change current range.
- **Fix**: Add ISO7741 quad digital isolator + GPIO, or PCA9535 I2C GPIO expander on isolated bus.

## P1 Major Issues

| # | Issue | Fix |
|---|-------|-----|
| 4 | Per-channel TCA9548A unnecessary (DAC 0x0C, ADC 0x2A don't conflict) | Remove, save $1.50 + 6 mA + 50 us latency per channel |
| 5 | NAU7802 DRDY not routed through isolation | Add digital isolator channel for interrupt, saves ~30% I2C bandwidth |
| 6 | No external trigger/sync I/O | Add BNC connectors + optocoupler for lab instrument sync |
| 7 | No AVDD/DVDD separation on isolated side | Add ferrite bead + decoupling between analog and digital power |
| 8 | WiFi TX bursts cause 3.3V rail dips | Add ferrite + 100 uF or dedicated LDO for ESP32-S3 |

## P2 Scalability & Performance

| # | Issue | Fix |
|---|-------|-----|
| 9 | I2C bus bottleneck: max ~56 SPS/channel at 8 channels | Replace NAU7802 with SPI ADC (ADS1256, 30 kSPS) + SPI isolator |
| 10 | No per-channel real-time controller | Add RP2040 per channel (isolated side) for autonomous control loop |
| 11 | B0505S (1W) insufficient for high-current ranges | Upgrade to 2-3W converter, or bypass TIA for >10 mA |
| 12 | No multi-channel sync mechanism | Add CAN bus or hardware trigger line for simultaneous start |

## P3 Nice-to-Have

| # | Issue | Fix |
|---|-------|-----|
| 13 | I2C limited to 400 kHz | Add LTC4311 accelerator for 1 MHz Fast-mode Plus |
| 14 | No power-good monitoring | Add TPS3839 voltage supervisor per rail |
| 15 | Per-channel TCA9548A overkill | Replace with LTC4316 address translator if needed |

## Power Budget Analysis (per channel)

| Component | Typical | Peak |
|-----------|---------|------|
| ADC (NAU7802) | 2.4 mA | 5 mA |
| DAC (AD5693R) | 0.5 mA | 1.5 mA |
| Control amp op-amp | 1.5 mA | 15-30 mA (driving CE) |
| RE buffer op-amp | 1 mA | 2 mA |
| TIA op-amp | 1.5 mA | 5 mA |
| Mux + isolator | 3.5 mA | 5.5 mA |
| **Total** | **~10 mA (50 mW)** | **~60 mA (300 mW)** |

At high-current range (100 mA cell current): CE driver alone = 500 mW, exceeding B0505S 1W budget.

## Bus Throughput Analysis

8 channels at 400 kHz I2C:
- Per-channel cycle: ~275 us (switch + ADC read + DAC write)
- 8 channels sequential: 2.2 ms = **454 Hz max scan rate total**
- Per channel at 8 active: **~56 SPS** (inadequate for CV)

With SPI ADC at 10 MHz: per-channel read = 2.4 us, enabling 30 kSPS per channel.
