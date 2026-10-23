# Firmware Upload Results

Evidence ID: EV-04
Related test matrix entries: T7, T8 (see ../TEST_MATRIX.md)
Device under test: ESP32 DevKit V1 (Boards 1, 2, 3 — single purchase batch)

# Purpose

To verify that firmware can be written to the boards' flash over the UART download path, that the write is verified by the programming tool at the time of writing, and that the upload process itself can be excluded as the source of the boot failure.

This evidence separates two claims that are routinely conflated in bring-up debugging: "the firmware uploaded successfully" and "the board boots". This investigation is precisely the case where the first is true and the second is false, so the upload record must stand on its own.

# Command / Procedure

1. Place the board in download mode (Procedure A, serial_logs.md, EV-01).
2. Upload the firmware image set (bootloader, partition table, application) to flash on COM6 using the standard esptool-based upload flow.
3. Allow the tool's write verification to complete.
4. Record the upload sequence output verbatim.

# Expected Result

The tool connects to the ROM downloader, writes each image segment to its flash offset, verifies each write by hash comparison against the source data, and resets the device via the RTS pin. Any failure of the UART path, the downloader, or the flash write mechanism would surface here as a connection, write, or verification error.

# Observed Result

```text
Connected to ESP32 on COM6
Writing bootloader
Writing partitions
Writing application
Hash of data verified
Hard resetting via RTS pin
```

Each stage completed without error. The hash verification step, in which the tool confirms that the data written matches the data supplied, passed for the uploaded image set. The device was then hard-reset by the tool, after which it exhibited the boot failure recorded in EV-01, Procedure B.

# Interpretation

The write path is fully functional under host control: UART communication, downloader operation, flash programming, and in-tool verification all succeed. The uploaded content is subsequently proven correct by two independent checks that do not rely on the upload tool's own verification: structural validation of the image file (EV-07) and readback of the stored image from flash with re-validation (EV-08).

The sequence in this record, successful verified upload immediately followed by boot failure on the tool-initiated reset, is the defining symptom of the investigation. It eliminates the entire class of "bad flash upload" explanations and directs the investigation at the boot-time read path and device configuration.

# Conclusion

The firmware upload process is successful and verified, and is not identified as a failure source. The boot failure occurs after a provably correct upload, on the subsequent reset.

# Confidence Level

**High.** Direct tool output, corroborated independently by EV-07 and EV-08. (As recorded by the investigator: upload process not identified as failure source; confidence High.)
