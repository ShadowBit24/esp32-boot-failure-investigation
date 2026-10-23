# Erase Verification — Bootloader Region Readback

Evidence ID: EV-06
Related test matrix entries: T12 (see ../TEST_MATRIX.md)
Device under test: ESP32 DevKit V1 (Boards 1, 2, 3 — single purchase batch)
Precondition: forced erase completed (flash_erase_validation.md, EV-05)

# Purpose

To verify that the forced erase of EV-05 physically altered the flash array in the expected address range, by reading back the bootloader region and confirming the erased state (all bytes `0xFF`, the erased state of NOR flash).

A tool reporting "erased successfully" is a claim, not a measurement. This verification converts the claim into a measurement, and simultaneously demonstrates that flash readback returns stable, meaningful data under host control, a fact relied on by EV-08.

# Command / Procedure

1. With the board in download mode, read 256 bytes of flash starting at offset `0x1000` (the bootloader image region) into a file:

```bash
esptool read-flash 0x1000 256 bootcheck.bin
```

2. Dump the file contents:

```bash
certutil -dump bootcheck.bin
```

# Expected Result

If the erase executed across the expected range, every byte in the read region reads `0xFF`. Any non-`0xFF` byte would indicate an incomplete erase, an erase applied to a different range than commanded, or a read path fault.

# Observed Result

Excerpt of the dump of the region read from `0x1000`:

```text
FF FF FF FF FF FF FF FF
FF FF FF FF FF FF FF FF
FF FF FF FF FF FF FF FF
```

The bytes read from the bootloader region are uniformly `0xFF`, the erased state. No residual image data is present in the region.

# Interpretation

The erase operation affected the expected address range: the region that previously held the bootloader image (whose presence is implied by the boot attempts of EV-01) now reads as erased throughout the sampled range. Erase is therefore confirmed at the storage level, not merely at the tool-report level.

This record also fixes a reference point for the read path. Under host control, flash reads return stable, uniform, semantically correct data (an erased region reads as erased). The read path is not generically unreliable; its failure is specific to autonomous boot conditions, as established in EV-01 and EV-09.

# Conclusion

The forced erase is verified effective in the bootloader region. Erase behaviour and host-side read behaviour are both correct. The boot failure cannot be attributed to a failed or partial erase.

# Confidence Level

**High.** Direct readback measurement of the physical array state. (As recorded by the investigator: erase operation affected the expected address range; confidence High.)
