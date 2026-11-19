# FY8300 Rev. 1.2 protocol notes

These notes summarize the uploaded **FeelTech FY8300 Series Function Waveform Generator – Host Computer Communication Protocol Specification, Rev. 1.2**. The document specifies 115200 bps and LF (0x0A) command termination.

## Channel command families

| Parameter | CH1 write/read | CH2 write/read | CH3 write/read |
|---|---|---|---|
| Waveform | WMW / RMW | WFW / RFW | TFW / RTW |
| Frequency | WMF / RMF | WFF / RFF | TFF / RTF |
| Amplitude | WMA / RMA | WFA / RFA | TFA / RTA |
| Offset | WMO / RMO | WFO / RFO | TFO / RTO |
| Duty | WMD / RMD | WFD / RFD | TFD / RTD |
| Phase | WMP / RMP | WFP / RFP | TFP / RTP |
| Output | WMN / RMN | WFN / RFN | TFN / RTN |

## Conversions applied

- Frequency write: Hz × 1,000,000, rounded to an integer and left-padded to 14 digits. The document’s examples are inconsistently padded, so verify this representation on the actual instrument.
- Amplitude readback: raw value / 1000 (example: 10000 means 10.000 V).
- Duty readback: raw value / 10 (example: 689 means 68.9%).
- Phase readback: raw value / 10 (example: 1289 means 128.9°).
- Output readback: 0 means OFF; 255 means ON.

## Known ambiguities and omissions

- Offset readback: examples and prose do not consistently agree across the channel sections. The UI therefore logs the raw returned value and does not apply a guessed conversion.
- Frequency readback: the document says the returned value is in Hz and shows a decimal-formatted example. Confirm parsing and precision with the device.
- CH1 modulation commands are documented; CH2/CH3 modulation is not assumed.
- Sweep direction command SMD appears in the original FY6900 project but is not present in the FY8300 Rev. 1.2 command summary.
- The FY8300 document does not specify the binary arbitrary-waveform upload handshake or payload format. The original project’s DDS_WAVE upload is therefore disabled.
- The document describes preset storage/loading as applying to the current two channels; CH3 inclusion is not established.
- The document does not specify a dedicated modulation STOP command.

## Key command groups retained from the document

- Measurement: WCC, WCZ, WCP, WCG, RCG, RCF, RCC, RCT, RC+, RC-, RCD.
- Sweep: SOB, SST, SEN, STI, SMO, SBE, SXY.
- Presets: USN, ULN.
- Synchronization: USA, USD, RSA. The prose around RSA return values contains a contradictory sentence; treat 0/255 interpretation with care.
- Device identification: UID, UMO.

## Hardware validation checklist

1. Connect at 115200 baud and confirm UMO/UID replies.
2. Read and write a harmless waveform setting on each channel.
3. Confirm the frequency write string and readback for 0.123456 Hz and 100 Hz.
4. Verify amplitude, duty, phase and output-state readbacks.
5. Characterize offset readback at negative, zero and positive offsets before enabling conversion.
6. Verify that modulation and sweep commands behave as expected.
7. Do not enable arbitrary upload until a separate FY8300 upload specification or verified capture is available.
