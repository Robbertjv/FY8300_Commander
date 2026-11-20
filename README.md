# FY8300 Commander

A browser-based Web Serial controller for the FeelTech FY8300 three-channel function waveform generator.

## Run

1. Serve this repository from GitHub Pages or another secure web origin (`https://` or `localhost`). Web Serial is not available from an ordinary `file://` page.
2. Use a Chromium-based browser that supports Web Serial.
3. Connect the FY8300 over USB serial.
4. Select the instrument's serial port and connect at **115200 baud**.

The application is the single-page `index.html`; no build step or external JavaScript dependencies are required.

## Documentation

The single authoritative FY8300 technical document is **[FY8300 Protocol and Implementation Guide](docs/FY8300_Protocol_and_Implementation.md)**. It separates:

- **DOCUMENTED** behaviour from the manufacturer's FY8300 Rev. 1.2 protocol;
- **AS BUILT** behaviour found in the current `index.html`;
- **TESTED** observations recorded for the FY8300-30M, firmware 1.2; and
- **UNVERIFIED / CONFLICT** items where the protocol, implementation and/or observations do not agree.

The FY6900 manuals and command reference are historical source material only; see [archive/legacy](archive/legacy/).

## Important compatibility notes

This is an FY8300 adaptation, not a guarantee of compatibility with every model or firmware revision. In particular:

- The tested decimal-Hz frequency write differs from the integer-microhertz write format described in Rev. 1.2.
- Integer readback scaling in the application differs from several examples in Rev. 1.2. Consult the guide before changing conversions.
- Negative offset readback, preset behaviour for CH3, modulation stop behaviour and FY8300 arbitrary-waveform device upload are not established.
- The current source contains a legacy `DDS_WAVE` upload routine; its presence is not evidence that FY8300 upload works.

Test with outputs disconnected from sensitive equipment. Keep raw serial logs and verify each operation on the exact instrument and firmware before relying on it.
