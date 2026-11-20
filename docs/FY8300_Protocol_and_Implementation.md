# FeelTech FY8300 Commander — Protocol and Implementation Guide

**Guide revision:** 2.0  
**Protocol reference:** FeelTech FY8300 Series Function Waveform Generator — Host Computer Communication Protocol Specification, Rev. 1.2  
**Implementation reviewed:** `index.html` on branch `main`  
**Hardware evidence recorded in this project:** FY8300-30M, firmware 1.2

This is the single FY8300 technical reference for this repository. It deliberately separates three kinds of evidence:

- **DOCUMENTED** — stated or exemplified by the manufacturer's Rev. 1.2 protocol.
- **AS BUILT** — what the current `index.html` application actually implements. This is a source-code observation, not proof that the instrument accepts every command.
- **TESTED** — behaviour explicitly recorded as observed on the FY8300-30M, firmware 1.2. A test observation must not be generalized to other models or firmware without retesting.
- **UNVERIFIED / CONFLICT** — ambiguous manufacturer documentation or behaviour for which this repository does not contain adequate test evidence.

Where the manufacturer's examples conflict with the implementation, both are retained below. Do not treat an implementation conversion as a universal protocol constant.

## 1. Serial connection

| Item | Manufacturer documentation | Current application |
|---|---|---|
| Baud rate | 115200 bit/s | 115200 baud |
| Format | ASCII serial commands | Web Serial API |
| Terminator | LF, byte 0x0A | LF-delimited commands and responses |
| Typical response | Text value followed by LF | Reads one response line |

The application should serialize queries: send a query, consume its response, then send the next. Concurrent requests can cause response-to-command association errors.

## 2. Channel command map

The protocol defines three channel prefixes:

| Channel | Write prefix | Read prefix |
|---|---|---|
| CH1 | `WM` | `RM` |
| CH2 | `WF` | `RF` |
| CH3 | `TF` | `RT` |

| Parameter | CH1 write/read | CH2 write/read | CH3 write/read |
|---|---|---|---|
| Waveform | WMW / RMW | WFW / RFW | TFW / RTW |
| Frequency | WMF / RMF | WFF / RFF | TFF / RTF |
| Amplitude | WMA / RMA | WFA / RFA | TFA / RTA |
| Offset | WMO / RMO | WFO / RFO | TFO / RTO |
| Duty | WMD / RMD | WFD / RFD | TFD / RTD |
| Phase | WMP / RMP | WFP / RFP | TFP / RTP |
| Output | WMN / RMN | WFN / RFN | TFN / RTN |

A write is the write command followed by its parameter text and LF. A query is the read command followed by LF.

**AS BUILT:** The current page contains command mappings for CH1, CH2 and CH3. It has three channel panels and uses the corresponding channel-specific read/write prefixes.

## 3. Frequency

**DOCUMENTED:** Rev. 1.2 describes a 14-digit integer frequency write value expressed in microhertz. Examples include:

| Example | Intended frequency |
|---|---:|
| `WMF100000000` | 100 Hz |
| `WMF000123456` | 0.123456 Hz |
| `WMF000000001` | 0.000001 Hz |

The document describes frequency readback in Hz.

**TESTED / AS BUILT:** The project records a successful decimal-Hz write on an FY8300-30M: `WFF13000000.000000` sets CH2 to 13 MHz. The current application formats frequency writes as Hz with six fractional digits and reads frequency as Hz (no integer divisor).

**CONFLICT:** This decimal-Hz write representation differs from the integer-microhertz representation described in Rev. 1.2. The tested format is the implementation choice for the tested instrument, not proof of compatibility with every FY8300 revision.

## 4. Readback conversion and interpretation

The current application preserves decimal-form responses as engineering values and only applies a divisor to integer-form responses. Leading zeroes are numeric padding.

| Parameter | Current `index.html` integer readback conversion | Decimal response | Manufacturer-document conflict / evidence |
|---|---:|---|---|
| Frequency | Parse directly as Hz | Parse directly as Hz | Rev. 1.2 describes integer-µHz writes; tested device accepted decimal-Hz write |
| Amplitude | raw / 10,000 | Used directly | Rev. 1.2 example `00000010000` → 10.000 V implies a different scale |
| Offset | raw / 1,000 for non-zero values; zero remains zero | Used directly | Rev. 1.2 examples/prose disagree; positive 12.000 V display match is recorded |
| Duty | raw / 1,000 | Used directly | Rev. 1.2 example 689 → 68.9% implies division by 10 |
| Phase | raw / 1,000 | Used directly | Rev. 1.2 example 1289 → 128.9° implies division by 10 |
| Output | Parsed response equal to 255 means ON | — | Rev. 1.2 examples show 0 = OFF and 255 = ON |

### Amplitude

**AS BUILT:** `getEngineeringValue(..., 10000)` is used for amplitude integer readbacks. Example: raw 50000 becomes 5.000 V.

**UNVERIFIED / CONFLICT:** The manufacturer's example implies another scale. Do not silently replace the code divisor without capturing raw responses from the target instrument and comparing them with its display.

### Offset

**DOCUMENTED:** Rev. 1.2 gives examples such as raw 611 → −0.389 V and raw 16782 → +6.782 V, while its prose describes a 10,000-code centre. The examples and prose are not mutually consistent.

**AS BUILT:** Decimal responses are displayed directly. Integer zero is treated as 0 V; other integer values are divided by 1,000.

**TESTED evidence recorded by the project:** raw 12000 displayed as 12.000 V, matching the FY8300 front panel. This is a positive-offset observation. Negative offset conversion is not established by that observation.

### Duty and phase

**AS BUILT:** Integer duty and phase readbacks are divided by 1,000; decimal responses are used directly. The source comments describe generic firmware variation, but the code's actual divisors are 1,000.

**CONFLICT:** Rev. 1.2 examples imply division by 10. The project does not provide enough raw-response test evidence to declare either scale universally correct.

### Output state

Rev. 1.2 examples show a zero response for OFF and 255 for ON. The current application treats a parsed value of 255 as ON. Confirm this on each target firmware.

## 5. Waveform codes

Waveform numbering is channel-dependent. Do not decode CH2 or CH3 using the CH1 table.

The CH1 list in Rev. 1.2 includes:

| Code | Waveform |
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

**AS BUILT:** The application has channel-specific handling for waveform values; CH1 and CH2 do not use identical display-code adjustments. Consult the corresponding channel table in the manufacturer's document before extending or changing mappings.

## 6. Modulation

**DOCUMENTED:** Rev. 1.2 describes modulation on CH1. The source value selects the modulation source (including CH2); it does not mean the modulation function is configured on CH2.

| Function | Write | Read | Values |
|---|---|---|---|
| Mode | WPFx | RPF | 0 ASK; 1 FSK; 2 PSK; 3 Trigger; 4 AM; 5 FM; 6 PM |
| Source | WPMx | RPM | 0 CH2; 1 external AC; 2 manual; 3 external DC |
| FSK second frequency | WFKvalue | RFK | Hz |
| Trigger pulse count | WPNvalue | RPN | Up to 1,048,575 |
| Manual trigger | WPO | — | Manual trigger |
| AM modulation rate | WPRvalue | RPR | Percent |
| FM frequency offset | WFMvalue | RFM | Hz |
| PM phase offset | WPPvalue | RPP | Degrees |

**AS BUILT:** The Commander presents a modulation control panel and sends the documented modulation parameter commands. The implementation is intended for CH1 modulation.

**UNVERIFIED:** A dedicated modulation STOP command is not established in Rev. 1.2. Do not assume the older FY6900 STOP behaviour is valid for FY8300.

## 7. Measurement

Rev. 1.2 command summary:

| Command | Purpose |
|---|---|
| WCCx | Coupling: 0 DC, 1 AC |
| WCZx | Clear counter (example uses 0) |
| WCPx | Pause measurement |
| WCGx | Gate: 0 = 1 s, 1 = 10 s, 2 = 100 s |
| RCG | Gate-time readback |
| RCF | Measured frequency |
| RCC | Count |
| RCT | Period, ns |
| RC+ | Positive pulse width, ns |
| RC- | Negative pulse width, ns |
| RCD | Duty cycle |

Interpret RCF using the selected gate time. Rev. 1.2's RCG readback description appears inconsistent for code 2 (100 s in the setting section, but a repeated 10 s in readback prose). Verify on hardware.

## 8. Sweep

Rev. 1.2 command summary:

| Command | Purpose |
|---|---|
| SOBx | Object: 0 frequency, 1 amplitude, 2 offset, 3 duty |
| SSTvalue | Start |
| SENvalue | End |
| STIvalue | Time, seconds |
| SMOx | Mode: 0 linear, 1 logarithmic |
| SBEx | Enable: 0 off, 1 on |
| SXYx | Source: 0 time, 1 external analogue VCO input |

Start/end units depend on the selected object (Hz, V or %). Some examples in Rev. 1.2 contain apparent command-name typos; the command summary identifies SEN as the end-value command.

**AS BUILT:** The current application implements sweep setup and control using the documented command family. Sweep direction is not implemented in the current page. The legacy FY6900 `SMD` direction command is not part of the FY8300 Rev. 1.2 command summary and must not be sent as if it were an FY8300 command.

## 9. System, presets and synchronization

| Command | Purpose |
|---|---|
| USNxx | Save current channel parameters to slot xx |
| ULNxx | Load parameters from slot xx |
| USAx | Enable synchronization object x |
| USDx | Cancel synchronization object x |
| RSAx | Read synchronization status |
| UBZx / RBZ | Buzzer state |
| UMSx / RMS | Uplink mode |
| UULx / RUL | Uplink enable |
| UID | Instrument ID |
| UMO | Model |

Synchronization objects: 0 waveform, 1 frequency, 2 amplitude, 3 offset, 4 duty, 5 square rise time, 6 square fall time, 7 output switch.

Rev. 1.2 uses 0/255 state conventions in several places. Its RSA and uplink descriptions contain inconsistencies; verify return values and exact command spelling before relying on them.

**PRESETS:** Rev. 1.2 describes preset storage/loading for the current two channels. CH3 inclusion is not established by that wording. The current application exposes preset controls, but CH3 preset behaviour should be treated as unverified unless separately tested.

## 10. Arbitrary waveform editor and upload

**AS BUILT:** The current application includes a local arbitrary-waveform editor and file import/export. The editor represents 8,192 samples, with two bytes per sample (low byte first, then high byte), for a 16,384-byte local file payload.

**IMPORTANT DISTINCTION:** Local `.fy` file handling is not the same as uploading a waveform to the instrument.

**UNVERIFIED / SAFETY NOTE:** Rev. 1.2 does not establish a complete FY8300 serial arbitrary-waveform upload handshake and payload format. The current source still contains a legacy `DDS_WAVE` handshake and transfer routine. Its presence in the code does not prove that it is valid for FY8300. Treat device upload as experimental and do not rely on it until verified against a manufacturer specification or a captured, successful FY8300 transfer.

## 11. Front-panel key simulation

Rev. 1.2 lists key identifiers: 01 WAVE, 02 MEAS, 03 SWEEP, 04 MOD, 05 SYNC, 06 SYS, 07 MORE, 08 CH1, 09 CH2, 10–14 F1–F5, 15 LEFT, 16 RIGHT, 17 OK, 18 UP, 19 DOWN. Consult the exact KEY syntax in the manufacturer's document before implementing it.

## 12. Current application: as-built summary

The current `index.html` is the active three-channel FY8300 application. Source inspection confirms:

- Three channel configurations with WM/RM, WF/RF and TF/RT command families.
- Frequency formatting as decimal Hz with six fractional digits.
- Integer readback scaling: amplitude /10,000; duty /1,000; phase /1,000; non-zero offset /1,000. Decimal responses bypass scaling.
- Output readback is interpreted as ON when the value is 255.
- CH1-oriented modulation controls.
- Sweep controls without the legacy FY6900 SMD direction command.
- A local arbitrary waveform editor and a legacy device-upload routine whose FY8300 compatibility is not established.

Source inspection establishes what the browser code sends and displays. It does not establish that every command has been accepted by every hardware revision.

## 13. Validation procedure

For a different FY8300 model or firmware:

1. Record the exact model and firmware.
2. Connect at 115200 baud and confirm LF framing.
3. Query one parameter at a time and save the exact raw response.
4. Set a known value on the front panel and query again.
5. Compare the raw response with the front-panel engineering value.
6. Test zero, small and large values, plus negative values where supported.
7. Verify writes and readbacks independently on all three channels.
8. Test modulation, sweep, presets and upload separately.
9. Keep raw logs and label each conversion as tested only for the model/firmware actually used.

## 14. References and provenance

1. FeelTech, *FY8300 Series Function Waveform Generator — Host Computer Communication Protocol Specification*, Rev. 1.2 (manufacturer document supplied with the project).
2. `index.html` in this repository, branch `main` (current application implementation).
3. Project observations recorded for the FY8300-30M, firmware 1.2. These observations are explicitly identified as TESTED above; undocumented behaviour remains unverified.

---
**End of FY8300 guide**
