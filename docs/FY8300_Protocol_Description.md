# FeelTech FY8300 – Host Communication Protocol and Practical Implementation Guide

**Document version:** 1.0  
**Date:** 20 November 2026  
**Protocol reference:** FeelTech FY8300 Series Function Waveform Generator – Host Computer Communication Protocol Specification, Rev. 1.2  
**Implementation reference:** [Robbertjv/FY8300_Commander](https://github.com/Robbertjv/FY8300_Commander), branch `main`  
**Hardware observations:** FeelTech FY8300-30M, firmware 1.2

> This guide distinguishes the manufacturer's published protocol from behaviour observed on the tested FY8300-30M and from the conversion rules used by FY8300 Commander. Where they differ or remain uncertain, the discrepancy is stated explicitly.

## 1. Scope and evidence labels

This document describes serial framing, channel command families, readback interpretation, engineering-unit conversions, modulation, measurement, sweep, system commands and practical implementation notes.

- **[SPEC]** Stated or exemplified in protocol Rev. 1.2.
- **[TEST]** Observed on the FY8300-30M, firmware 1.2, during Commander development.
- **[COMMANDER]** Behaviour intended by the current Commander implementation.
- **[GAP / CONFLICT]** Ambiguous, inconsistent or not fully verified.

The official PDF remains the authority for published command definitions. The Commander source is the authority for what the web application currently implements.

## 2. Serial transport and framing

| Parameter | Value |
|---|---|
| Baud rate | 115200 bit/s [SPEC] |
| Encoding | ASCII text [SPEC] |
| Command terminator | Line Feed (LF), byte 0x0A [SPEC] |
| Response terminator | LF, byte 0x0A [SPEC] |
| Response format | Usually a text value followed by LF |

Example query:

```text
RMF\n
```

The instrument returns a line terminated by LF. A read response generally contains the value, not the original command name.

**Implementation note:** The browser Commander uses the Web Serial API and LF-delimited response lines. Serialize queries: send one query, consume its response, then send the next. Overlapping queries can make response-to-command matching unreliable. Frequency write/readback operations are queued to avoid collisions with polling.

## 3. Channel prefixes and command map

| Channel | Write prefix | Read prefix |
|---|---|---|
| CH1 | `WM` | `RM` |
| CH2 | `WF` | `RF` |
| CH3 | `TF` | `RT` |

| Suffix | Parameter |
|---|---|
| `W` | Waveform |
| `F` | Frequency |
| `A` | Amplitude |
| `O` | Offset |
| `D` | Duty cycle |
| `P` | Phase |
| `N` | Output enable |

| Parameter | CH1 write / read | CH2 write / read | CH3 write / read |
|---|---|---|---|
| Waveform | `WMW` / `RMW` | `WFW` / `RFW` | `TFW` / `RTW` |
| Frequency | `WMF` / `RMF` | `WFF` / `RFF` | `TFF` / `RTF` |
| Amplitude | `WMA` / `RMA` | `WFA` / `RFA` | `TFA` / `RTA` |
| Offset | `WMO` / `RMO` | `WFO` / `RFO` | `TFO` / `RTO` |
| Duty | `WMD` / `RMD` | `WFD` / `RFD` | `TFD` / `RTD` |
| Phase | `WMP` / `RMP` | `WFP` / `RFP` | `TFP` / `RTP` |
| Output | `WMN` / `RMN` | `WFN` / `RFN` | `TFN` / `RTN` |

A write consists of the write command, its parameter text, then LF. A query consists of the read command followed by LF.

## 4. General readback parsing

1. Read one complete LF-terminated response line.
2. Remove the LF and surrounding whitespace.
3. Preserve the original raw string for diagnostics.
4. Parse the numeric text.
5. Apply the parameter-specific scale or conversion.
6. Format the engineering value for display.

Leading zeroes are padding. For example, `0000000255` parses numerically as 255. A decimal point is significant: do not apply an integer scale factor again to a response that is already expressed as a decimal engineering value.

## 5. Raw response bytes/strings to displayed values

This section is especially important: the raw serial response is not always already in the same units shown by the instrument or Commander.

### 5.1 Frequency

**[SPEC]** Rev. 1.2 describes a 14-digit integer frequency write value in microhertz (µHz). Its examples include:

| Write command | Meaning in Rev. 1.2 |
|---|---:|
| `WMF100000000` | 100 Hz |
| `WMF000123456` | 0.123456 Hz |
| `WMF000000001` | 0.000001 Hz (1 µHz) |

The specification's readback section describes the returned frequency in Hz.

**[TEST / COMMANDER]** On the tested FY8300-30M, a decimal-Hz write was verified: `WFF13000000.000000` sets CH2 to 13 MHz. Commander therefore sends frequency in Hz with six fractional digits and parses frequency readback directly as Hz (no divisor).

| Raw readback | Conversion | Human-readable value |
|---|---|---:|
| `11990000.000000` | Parse as decimal Hz | 11.99 MHz |
| `00000010.000000` | Parse as decimal Hz | 10 Hz |
| `30000000.000000` | Parse as decimal Hz | 30 MHz |

**[GAP / CONFLICT]** The decimal-Hz write format verified on the tested device differs from the integer-µHz write description in Rev. 1.2. Commander follows the tested format. Verify before using with a different model or firmware.

### 5.2 Amplitude

**[COMMANDER]** Integer-form readback is converted using a divisor of 10,000; decimal-form readback is used directly.

`Amplitude (V) = raw numeric value / 10,000` (integer response only).

| Raw response | Calculation | Display |
|---|---:|---:|
| `50000` | 50000 / 10000 | 5.000 V |
| `12345` | 12345 / 10000 | 1.235 V |
| `5.000` | Already decimal | 5.000 V |

**[GAP / CONFLICT]** Rev. 1.2 includes a readback example `00000010000` representing 10.000 V, which implies a different integer scale. The table above documents the Commander conversion, not a universal manufacturer rule. Verify raw values on the target firmware. The specification's amplitude write example uses decimal volts, e.g. `WMA12.351\n`.

### 5.3 Offset

Offset is a known discrepancy between the Rev. 1.2 explanation and tested front-panel matching.

**[SPEC]** Rev. 1.2 describes a 10,000-code centre and gives examples including raw `611` → −0.389 V and `16782` → +6.782 V. Its prose about subtracting the raw value from 10,000 is not consistent with all its examples.

**[TEST / COMMANDER]** The tested FY8300-30M returned values that were compared with the front-panel display. The current Commander conversion selected to match observed display values is:

`Offset (V) = raw numeric value / 1,000` (integer response only).

| Raw response | Calculation | Display |
|---|---:|---:|
| `100` | 100 / 1000 | 0.100 V |
| `9000` | 9000 / 1000 | 9.000 V |
| `10000` | 10000 / 1000 | 10.000 V |
| `11015` | 11015 / 1000 | 11.015 V |
| `12000` | 12000 / 1000 | 12.000 V |
| `10.100` | Already decimal | 10.100 V |

The conversion was selected so that, for example, `RMO` returning `12000` displays `12.000`, matching the FY8300 front panel. **This is not the signed-offset interpretation described in Rev. 1.2. Negative offset readback has not been fully verified.** Do not infer a negative conversion from the positive-value table.

Offset writes use signed decimal volts in the specification, e.g. `WMO-2.352\n`.

### 5.4 Duty cycle

**[COMMANDER]** Integer-form readback is converted using a divisor of 1,000; decimal-form response is used directly.

`Duty (%) = raw numeric value / 1,000` (integer response only).

| Raw response | Calculation | Display |
|---|---:|---:|
| `68900` | 68900 / 1000 | 68.9 % |
| `50000` | 50000 / 1000 | 50.0 % |
| `68.9` | Already decimal | 68.9 % |

**[GAP / CONFLICT]** Rev. 1.2 examples show raw `689` as 68.9%, implying division by 10. This differs from the Commander divisor above. Confirm the exact firmware's raw response before changing or reusing the conversion. Duty write examples use decimal percent, e.g. `WMD50.1\n`.

### 5.5 Phase

**[COMMANDER]** Integer-form readback is converted using a divisor of 1,000; decimal-form response is used directly.

`Phase (degrees) = raw numeric value / 1,000` (integer response only).

| Raw response | Calculation | Display |
|---|---:|---:|
| `128900` | 128900 / 1000 | 128.9° |
| `218900` | 218900 / 1000 | 218.9° |
| `128.9` | Already decimal | 128.9° |

**[GAP / CONFLICT]** Rev. 1.2 examples `1289` → 128.9° and `2189` → 218.9° imply division by 10, unlike the Commander integer divisor. Verify the raw format on the target instrument. Phase write examples use decimal degrees, e.g. `WMP123.4\n`.

### 5.6 Output state

| Write value | Meaning |
|---|---|
| `0` | OFF |
| `1` | ON |

Readback examples in Rev. 1.2:

| Raw response | Meaning |
|---|---|
| `0000000000` | OFF |
| `0000000255` | ON |

Commander considers the output ON when the parsed response value is 255.

### 5.7 Conversion quick reference

| Parameter | Commander integer readback conversion | Display units | Important qualification |
|---|---|---|---|
| Frequency | Parse directly as Hz (tested decimal response) | Hz / kHz / MHz | Differs from Rev. 1.2 integer-µHz write description |
| Amplitude | raw / 10,000 | V | Rev. 1.2 example implies another scale |
| Offset | raw / 1,000 | V | Empirical display matching; negative values not verified |
| Duty | raw / 1,000 | % | Rev. 1.2 examples imply /10 |
| Phase | raw / 1,000 | degrees | Rev. 1.2 examples imply /10 |
| Output | raw == 255 means ON | ON/OFF | 0/255 readback examples |

These are implementation conversions, not interchangeable universal protocol constants. When a response already contains a decimal point, it represents an engineering value and should not be scaled again.

## 6. Waveform codes

Waveform numbering is channel-dependent. Do not assume CH1 and CH2/CH3 use identical codes.

The CH1 readback list in Rev. 1.2 includes:

| Code | CH1 waveform |
|---:|---|
| 0 | SINE |
| 1 | Square |
| 2 | Rectangle |
| 3 | Trapezoid |
| 4 | CMOS |
| 5 | Adj-Pulse |
| 6 | DC |
| 7 | TRGL |
| 8 | Ramp |
| 9 | NegRamp |
| 10 | Stair TRGL |
| 11 | Stairstep |
| 12 | NegStair |
| 13 | PosExponen |
| 14 | NegExponen |
| 15 | P-Fall-Exp |
| 16 | N-Fall-Exp |
| 17 | PosLogarit |
| 18 | NegLogarit |
| 19 | P-Fall-Log |
| 20 | N-Fall-Log |
| 21 | P-Full-Wav |
| 22 | N-Full-Wav |
| 23 | P-Half-Wav |
| 24 | N-Half-Wav |
| 25 | Lorentz-Pu |
| 26 | Multitone |
| 27 | Random-Noi |
| 28 | ECG |
| 29 | Trapezoid |
| 30 | Sinc-Pulse |
| 31 | Impulse |
| 32 | AWGN |
| 33 | AM |
| 34 | FM |
| 35 | Chirp |
| 36–99 | Arbitrary waveforms 1–64 |

The CH2/CH3 tables have a different ordering (for example, DC is code 5 and TRGL code 6). Use the relevant channel's table in the official PDF when decoding those channels.

## 7. Modulation (CH1)

Rev. 1.2 documents modulation as a CH1 function.

| Function | Write command | Read command | Values / interpretation |
|---|---|---|---|
| Modulation mode | `WPFx` | `RPF` | 0 ASK; 1 FSK; 2 PSK; 3 Trigger; 4 AM; 5 FM; 6 PM |
| Modulation source | `WPMx` | `RPM` | 0 CH2; 1 external AC; 2 manual; 3 external DC |
| FSK second frequency | `WFKvalue` | `RFK` | Hz |
| Trigger pulse count | `WPNvalue` | `RPN` | Maximum 1,048,575 |
| Manual trigger | `WPO` | — | Issues a manual trigger |
| AM modulation rate | `WPRvalue` | `RPR` | Percent; 23.4 means 23.4% |
| FM frequency offset | `WFMvalue` | `RFM` | Hz |
| PM phase offset | `WPPvalue` | `RPP` | Degrees |

Source value 0 selects CH2 as the internal modulation source; it does not mean that modulation is configured on CH2. Commander treats the modulation panel as CH1-only. A modulation-stop command is not included because a corresponding stop command is not established by Rev. 1.2.

## 8. Measurement commands

| Command | Direction | Purpose |
|---|---|---|
| `WCCx` | Write | Coupling: 0 DC, 1 AC |
| `WCZx` | Write | Clear counter (Rev. 1.2 example uses 0) |
| `WCPx` | Write | Pause measurement |
| `WCGx` | Write | Gate: 0 = 1 s, 1 = 10 s, 2 = 100 s |
| `RCG` | Read | Gate-time readback |
| `RCF` | Read | Measured frequency |
| `RCC` | Read | Count |
| `RCT` | Read | Period, ns |
| `RC+` | Read | Positive pulse width, ns |
| `RC-` | Read | Negative pulse width, ns |
| `RCD` | Read | Duty cycle |

Measurement conversions and interpretation:

| Raw value / context | Conversion | Meaning |
|---|---|---|
| RCF = 668, gate = 1 s | 668 Hz | 1-second gate |
| RCF = 668, gate = 10 s | 668 / 10 | 66.8 Hz |
| RCF = 668, gate = 100 s | 668 / 100 | 6.68 Hz |
| RCC = 668 | Integer count | 668 counts |
| RCT = 60668 | Raw numeric value in ns | 60,668 ns |
| RC+ = 60668 | Raw numeric value in ns | 60,668 ns |
| RC- = 60668 | Raw numeric value in ns | 60,668 ns |
| RCD = 668 | Divide by 10 | 66.8% |

Read the gate setting before interpreting RCF. Rev. 1.2's RCG readback text appears to repeat 10 s for code 2, while the WCG setting section specifies 100 s. Treat this as a documentation inconsistency and verify on hardware.

## 9. Sweep commands

| Command | Purpose |
|---|---|
| `SOBx` | Sweep object: 0 frequency, 1 amplitude, 2 offset, 3 duty |
| `SSTvalue` | Sweep start |
| `SENvalue` | Sweep end |
| `STIvalue` | Sweep time, seconds |
| `SMOx` | Mode: 0 linear, 1 logarithmic |
| `SBEx` | Sweep enable: 0 off, 1 on |
| `SXYx` | Source: 0 time, 1 external analogue VCO input |

SST/SEN units depend on the selected object: frequency in Hz, amplitude and offset in V, duty in %. Rev. 1.2 contains apparent typographical errors in some SEN examples (SSN/SSNT); the command summary identifies SEN as the end-value command. Commander leaves sweep direction controls disabled because a reliable FY8300 direction command has not been established.

## 10. System and synchronization

| Command | Purpose |
|---|---|
| `USNxx` | Save current channel parameters to memory slot xx |
| `ULNxx` | Load parameters from memory slot xx |
| `USAx` | Enable synchronization object x |
| `USDx` | Cancel synchronization object x |
| `RSAx` | Read synchronization status |
| `UBZx` / `RBZ` | Buzzer state |
| `UMSx` / `RMS` | Uplink mode |
| `UULx` / `RUL` | Uplink enable |
| `UID` | Instrument ID |
| `UMO` | Model |

Synchronization object values:

| Code | Object |
|---:|---|
| 0 | Waveform |
| 1 | Frequency |
| 2 | Amplitude |
| 3 | Offset |
| 4 | Duty |
| 5 | Square-wave rise time |
| 6 | Square-wave fall time |
| 7 | Output switch |

For RSA, 0 means disabled and 255 enabled. Rev. 1.2 uses 0/255 conventions for buzzer and uplink state. The PDF contains an uplink-enable naming inconsistency (UUL/UML); verify the detailed command and hardware before relying on it.

## 11. Front-panel key simulation

Rev. 1.2 lists the following key identifiers:

| Code | Key |
|---:|---|
| 01 | WAVE |
| 02 | MEAS |
| 03 | SWEEP |
| 04 | MOD |
| 05 | SYNC |
| 06 | SYS |
| 07 | MORE |
| 08 | CH1 |
| 09 | CH2 |
| 10–14 | F1–F5 |
| 15 | LEFT |
| 16 | RIGHT |
| 17 | OK |
| 18 | UP |
| 19 | DOWN |

Consult the KEY command section of the official PDF for exact syntax before implementing new key operations.

## 12. Arbitrary waveform files and upload limitation

Commander includes a local arbitrary-waveform editor for 8,192 samples. Local `.fy` import/export represents each sample using two bytes (low byte first, then high byte), for 16,384 bytes total.

**Local file import/export is not a device upload.** Rev. 1.2 does not establish a confirmed FY8300 serial arbitrary-waveform upload frame. A legacy FY6900 `DDS_WAVE` handshake has not been confirmed as valid for FY8300. Commander therefore leaves direct serial upload disabled.

## 13. Practical implementation notes

The Commander project includes three channel panels, Web Serial transport, LF line handling, channel-specific prefixes, frequency write/readback sequencing, readback polling, CH1 modulation controls and a local arbitrary-waveform editor. Some controls are deliberately disabled where the FY8300 command or upload protocol is not sufficiently established.

When changing firmware support, keep raw serial response strings in logs and update the conversion tables only after testing against the instrument display. Do not silently treat a discrepancy in the PDF as resolved.

## 14. Validation procedure for another instrument/firmware

1. Record exact model and firmware version.
2. Connect at 115200 baud and confirm LF (0x0A) framing.
3. Query one parameter at a time and save the exact raw response string.
4. Set a known value on the front panel and query again.
5. Compare raw response to the front-panel engineering value.
6. Test zero, a small value, a larger value and negative values where supported.
7. Verify write and readback paths independently.
8. Keep the raw log and mark the conversion as verified for that model/firmware.

## 15. References

1. FeelTech, *FY8300 Series Function Waveform Generator – Host Computer Communication Protocol Specification*, Rev. 1.2 (PDF supplied with the project).
2. [FY8300 Commander GitHub repository](https://github.com/Robbertjv/FY8300_Commander), branch `main`, particularly `index.html`.

---
**End of document**
