# FY8300 Commander

A browser-based serial remote for the **FeelTech FY8300** function generator, adapted from the original [FY6900 Commander](https://github.com/katzenjens/fy6900commander).

## Run

1. Open the GitHub Pages URL for this repository (after enabling Pages), or serve this repository from a local web server.
2. Use a Chromium-based browser with Web Serial support.
3. Connect the FY8300 over USB serial.
4. Select the serial device and connect at **115200 baud**.

The interface is a single-page HTML application. No build step or external JavaScript dependencies are required.

## FY8300 support status

- CH1, CH2 and CH3 basic waveform, frequency, amplitude, offset write, duty, phase and output controls use the FY8300 Rev. 1.2 command families.
- Frequency writes are sent as a 14-digit integer count of microhertz.
- Amplitude readback is scaled by 1000; duty and phase readback by 10.
- Offset readback is deliberately left as a raw value in the serial log because the Rev. 1.2 document contains inconsistent conversion examples. The entered offset is not overwritten by an uncertain conversion.
- Modulation is CH1-only in the protocol document.
- Sweep uses documented commands only. Sweep direction is disabled because the source project’s SMD command is not in FY8300 Rev. 1.2.
- Arbitrary waveform serial upload is disabled: the FY8300 Rev. 1.2 document does not define the binary upload framing used by the original FY6900 project.
- Preset save/load remains available, but the protocol describes storing the current two channels; CH3 preset behavior is not confirmed.
- Modulation STOP is disabled because no corresponding stop command is specified in the protocol document.

## Important

This is a protocol-based adaptation, **not yet hardware-validated**. Test with outputs disconnected from sensitive equipment and verify each command/readback on your own FY8300 before relying on it. In particular, verify offset readback behavior and the 14-digit frequency representation against the instrument.

See [FY8300_PROTOCOL_NOTES.md](FY8300_PROTOCOL_NOTES.md) for the documented command map and known ambiguities.
