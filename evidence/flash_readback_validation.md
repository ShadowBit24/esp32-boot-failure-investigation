# Flash Readback Validation

Evidence ID: EV-08
Related test matrix entries: T13 (see ../TEST_MATRIX.md)
Device under test: ESP32 DevKit V1 (Boards 1, 2, 3 — single purchase batch)

# Purpose

To prove that the application image, as physically stored in the boards' flash, is complete and intact, by reading the application region back out of the device and subjecting the recovered file to the same structural validation as the source image (EV-07).

This is the strongest content-level test available with host-side tooling. Upload verification (EV-04) is the tool attesting to its own work; readback validation is an independent measurement of the stored state, made after the fact, through a separate operation.

# Command / Procedure

1. With the board in download mode, after a verified upload (EV-04), read the application region of flash (offset `0x10000`, length `0x50000`) into a file:

```bash
esptool read-flash 0x10000 0x50000 app.bin
```

2. Validate the recovered file:

```bash
esptool image-info app.bin
```

# Expected Result

The read completes for the full region (0x50000 = 327,680 bytes). The recovered file parses as an ESP32 image with valid checksum and valid validation hash, identical in standing to the source image validated in EV-07. Any storage corruption, incomplete write, or read-path data fault under host control would appear as a read error, a size discrepancy, or a validation failure.

# Observed Result

Read:

```text
Read 327680 bytes successfully.
```

Validation of the recovered image:

```text
Checksum:              valid
Validation hash:       valid
Detected image type:   ESP32
```

# Interpretation

The full chain of custody of the application image is now closed by measurement at every step:

| Stage | Evidence | Result |
|---|---|---|
| Image generated | EV-07 (image-info on build output) | Valid |
| Image transferred and written | EV-04 (upload with hash verification) | Verified |
| Image stored and recovered | EV-08 (this record) | Valid |

The data the ROM requires at boot is present in flash, byte-accessible, and structurally valid. Under host control, the flash read path returns this data correctly and completely.

The boundary of this evidence must be stated exactly: readback is performed by the ROM downloader at the downloader's parameters, on host demand. It does not observe the autonomous boot read, which is performed by the ROM boot path at boot parameters. The investigation's failure lives in the difference between those two reads, and this record is what makes that statement precise rather than rhetorical: the same device, the same array, the same region, read correctly by one path and rejected by the other (EV-01).

# Conclusion

Flash read functionality is verified under host control, and the stored application image is intact. Data corruption in storage is excluded as the boot failure cause.

# Confidence Level

**High.** Independent readback measurement with structural re-validation. (As recorded by the investigator: flash read functionality verified; confidence High.)
