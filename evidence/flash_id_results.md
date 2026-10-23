# Flash Identification Results

Evidence ID: EV-02
Related test matrix entries: T6 (see ../TEST_MATRIX.md)
Device under test: ESP32 DevKit V1 (Boards 1, 2, 3 — single purchase batch)

# Purpose

To identify the SPI flash device fitted to the boards under test, confirm that the device responds to host interrogation over the SPI interface under ROM download-mode control, and verify that the detected capacity matches the board configuration used by the firmware build (4 MB).

Flash identity is a prerequisite for interpreting every other flash-related result in this package: the manufacturer and device codes establish which component's behaviour the investigation is characterising.

# Command / Procedure

The board is placed in download mode (Procedure A, serial_logs.md, EV-01), then interrogated with esptool:

```bash
esptool.exe --chip esp32 -p COM6 flash-id
```

# Expected Result

esptool establishes communication with the ROM downloader, reads the SPI flash JEDEC identification, and reports a manufacturer code, a device code, and a detected flash size consistent with the physical device. A non-responsive or misidentified device would indicate a dead flash, a wiring/CS fault, or an interface failure at the downloader level.

# Observed Result

```text
Chip type:          ESP32-D0WD-V3 (revision v3.1)
Manufacturer:       68
Device:             4016
Detected flash size: 4MB
Flash voltage:      3.3V (set by a strapping pin)
```

| Field | Value | Note |
|---|---|---|
| SoC | ESP32-D0WD-V3, revision v3.1 | As reported by the ROM via esptool |
| Flash manufacturer | `0x68` | BoyaMicro |
| Flash device | `0x4016` | BY25Q32BS, 32 Mbit |
| Detected size | 4 MB | Matches the device code and the build configuration |
| Flash voltage | 3.3 V | Set by strapping pin, as reported |

The identification was consistent across the boards tested.

# Interpretation

The flash device responds correctly and deterministically to host interrogation. The SPI interface, chip-select path, and flash device are all capable of correct digital communication under downloader conditions, at the downloader's operating parameters. The detected capacity (4 MB) agrees with the device code (`0x4016`) and with the partition configuration of the uploaded firmware (EV-04), excluding a size/configuration mismatch as a cause of the boot failure.

This result is one half of the central contrast of the investigation: the same interface and device that return correct, repeatable identification here return non-deterministic header data at autonomous boot (EV-01, EV-09). Any root-cause hypothesis must account for both observations.

# Conclusion

The SPI flash device is present, correctly identified as a BoyaMicro BY25Q32BS (manufacturer `0x68`, device `0x4016`, 4 MB), and operational under host control. Flash absence, death, or misidentification is excluded as a failure cause.

# Confidence Level

**High.** Direct tool output against the physical device, consistent across boards. (As recorded by the investigator: flash chip operational; confidence High.)
