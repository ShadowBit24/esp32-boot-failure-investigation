# Firmware Image Validation

Evidence ID: EV-07
Related test matrix entries: T13, file-side component (see ../TEST_MATRIX.md)
Device under test: firmware image set for ESP32 DevKit V1 (Boards 1, 2, 3 — single purchase batch)

# Purpose

To validate the application image file itself, independently of any board, using Espressif's own structural checks. This isolates one variable completely: if the image file were malformed, mis-built, or corrupt, the boot failure would require no hardware explanation at all. This evidence closes that possibility at the file level, before any board-side readback is considered (EV-08 closes it at the storage level).

# Command / Procedure

The application image produced by the build/upload flow is inspected with esptool's image analysis:

```bash
esptool image-info app.bin
```

# Expected Result

For a correctly generated ESP32 application image: the tool recognises the image type, parses the header and segment structure, and reports the embedded checksum and the validation hash as valid. A build or file corruption fault would present as an unrecognised image, a checksum failure, or a hash failure.

# Observed Result

```text
Detected image type: ESP32
Checksum:            valid
Validation hash:     valid
Flash mode:          DIO
Flash size:          4MB
Project name:        arduino-lib-builder
```

| Check | Result |
|---|---|
| Image type recognition | ESP32 image |
| Checksum | Valid |
| Validation hash (SHA-256) | Valid |
| Declared flash mode | DIO |
| Declared flash size | 4 MB (consistent with EV-02) |

# Interpretation

The image is structurally sound by the toolchain's own criteria. Its header, the same header whose corruption is reported at boot (EV-01), is present and well-formed in the file; its checksum and cryptographic validation hash both verify. The declared flash size matches the physically detected device (EV-02), excluding a size-configuration mismatch.

The significance of this record is eliminative. Combined with EV-04 (verified upload) it establishes that correct data was generated and correct data was sent. Combined with EV-08 (readback validation) it establishes that correct data was stored. The boot failure therefore cannot be located in the image's content at any point in its lifecycle: generation, transfer, or storage. What remains is the retrieval of that content by the ROM at boot, under boot conditions.

# Conclusion

The application image is valid. Firmware image corruption is not identified, at the file level, as a contributor to the boot failure.

# Confidence Level

**High.** Deterministic structural validation by the vendor toolchain. (As recorded by the investigator: firmware image corruption not identified; confidence High.)
