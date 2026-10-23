# Flash Erase Validation

Evidence ID: EV-05
Related test matrix entries: T10, T11 (see ../TEST_MATRIX.md)
Device under test: ESP32 DevKit V1 (Boards 1, 2, 3 — single purchase batch)

# Purpose

To test the boards' flash erase behaviour under both standard and forced conditions, for two reasons:

1. Erase is a required step in the recovery procedure for a corrupted boot image; its availability determines which recovery options exist.
2. Erase behaviour is a behavioural probe of the device's security state. A refusal on security grounds, if observed, independently corroborates the eFuse record (EV-03) without relying on eFuse reads.

# Command / Procedure

With the board in download mode (Procedure A, serial_logs.md, EV-01), a full-chip erase is attempted first in standard form, then, following the refusal, in forced form:

```bash
esptool erase-flash
esptool erase-flash --force
```

# Expected Result

On a device without active flash-security features, the standard command erases the full flash array and reports success. On a device with active security features, esptool refuses the standard erase as a safety measure; this refusal is itself a documented, information-bearing behaviour of the tool. The forced form proceeds, with the tool accepting the operator's explicit override.

# Observed Result

Standard erase:

```text
Active security features detected,
erasing flash is disabled as a safety measure.
```

Result: erase refused. No data was erased by this command.

Forced erase:

```text
Flash memory erased successfully.
```

Result: full-chip erase completed.

# Interpretation

The standard erase refusal is a positive security-state detection by an independent tool through a behavioural path, not an eFuse read. It confirms, from a second direction, the conclusion of EV-03: flash security features are active on these devices. Two methods that share no mechanism (eFuse field inspection, erase interlock behaviour) report the same underlying state.

The forced erase success confirms that the flash array honours erase commands and that the interlock is a policy gate, not a device malfunction: when the operator overrides, the operation executes normally. This distinction matters for fault localisation, because it separates "flash cannot erase" (a device defect, not observed) from "flash erase is security-gated" (a configuration property, observed).

# Conclusion

Flash erase capability is confirmed. Standard erase is blocked by active security features, corroborating the eFuse security configuration recorded in EV-03; forced erase completes successfully. Security protections were detected and are a property of the devices as received.

# Confidence Level

**High.** Direct tool output for both outcomes, mutually corroborating with EV-03. (As recorded by the investigator: flash erase capability confirmed; confidence High.)
