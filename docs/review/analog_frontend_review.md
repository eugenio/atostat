# Analog Front-End Design Review

## P0 Blockers (Must Fix)

### 1. No Op-Amp ICs
**All op-amp modules** have placeholder interfaces only -- no actual IC.
- **Fix**: Create AD8608 (quad CMOS, 1 pA bias, R-R I/O) atopile package. Instantiate in control_amplifier, re_buffer, switched_gain_tia (3 of 4 sections). For TIA 10nA range, add LMP7721 (3 fA bias) option.

### 2. No Cell Disconnect Switch
No mechanism to disconnect the cell during power-up/idle/power-down. Safety hazard.
- **Fix**: Add DPDT relay on CE and WE lines in `channel_module.ato`. Route control through isolator.

### 3. CD74HC4067 Unsuitable for TIA Feedback
- R_on: 70-100 ohm (350% error on 20 ohm range)
- Leakage: 100 nA at 25C (10x full-scale at 10 nA range)
- Cannot pass negative voltages (CMOS logic-level mux)
- **Fix**: Replace with MAX14778 (+-25V, low R_on, low leakage) for ranges 0-5. Discrete low-R_on switches for 200/20 ohm ranges. Latching relays for 100M/20M ranges.

### 4. DAC Output is Unipolar
AD5693R outputs 0 to 5V only. Cannot apply negative potentials.
- **Fix**: Add subtractor circuit (precision diff amp) after DAC to produce +-Vref output, or use virtual-ground-at-midscale approach.

### 5. No Bipolar Supply
Isolated side is single-ended 5V. Op-amps need +-5V for bipolar operation.
- **Fix**: Add inverting charge pump (LM27761 or TPS60403) on isolated side to generate -5V from +5V.

## P1 Major Issues

| # | Issue | File | Fix |
|---|-------|------|-----|
| 6 | TIA missing non-inverting input interface | switched_gain_tia.ato | Add opamp_noninverting_in, connect to ground reference |
| 7 | TIA gain_select lines dangling (not connected to controller) | channel_module.ato | Route through GPIO expander (PCA9535) on isolated I2C |
| 8 | NAU7802 too slow (320 SPS max) for CV | adc_interface.ato | Replace with ADS1256 (30 kSPS, SPI) or add fast ADC path |
| 9 | No galvanostat mode | channel_module.ato | Add SPDT switch selecting RE buffer vs TIA output as control amp feedback |
| 10 | CE output tapped before snubber | control_amplifier.ato | Move ce_output connection to after snubber resistor |
| 11 | RE buffer feedback loop incomplete | re_buffer.ato | Wire opamp_output to opamp_inverting_in inside module |

## P2 Moderate Issues

| # | Issue | Fix |
|---|-------|-----|
| 12 | ADC voltage channel reads V_RE (redundant) | Measure V_WE vs V_RE instead |
| 13 | No TIA input protection | Add BAT54S Schottky clamp diodes |
| 14 | No guard ring driver | Add unity-gain buffer for guard net |
| 15 | DAC 1.6 kHz filter limits fast techniques | Make switchable |
| 16 | No EIS capability | Add AC excitation path (DDS + summing network) |

## P3 Nice-to-Have

| # | Issue | Fix |
|---|-------|-----|
| 17 | No CE voltage clamp | Add TVS diode (SMBJ5.0A) |
| 18 | Fixed compensation (200 Hz) | Make switchable for different cell types |

## Noise Analysis Summary
- 10 nA range (100 MOhm): 0.24 pA RMS noise floor at 340 Hz BW -- acceptable
- NAU7802: ~20 ENOB at 320 SPS, adequate resolution but too slow
- 1 kohm ADC protection resistors: acceptable (51 nV RMS additional noise)
- DAC 1.6 kHz filter: OK for standard CV, limits fast techniques
