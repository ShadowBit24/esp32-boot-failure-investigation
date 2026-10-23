# SPI Flash Mode and Frequency Experiments

Evidence ID: EV-09
Related test matrix entries: T14, T15, T16 (see ../TEST_MATRIX.md)
Device under test: ESP32 DevKit V1 (Boards 1, 2, 3 — single purchase batch)

# Purpose

To determine whether the boot failure is sensitive to the SPI flash interface configuration, by varying one interface parameter at a time (flash mode, then clock frequency), holding all other conditions constant, and recording the exact boot behaviour for each configuration.

This experiment discriminates between two fault classes:

- **Content-class faults** (corrupt data, wrong image, configuration-independent defects) predict identical failure behaviour in every configuration, because the data being read does not change.
- **Interface-class faults** (marginal transfer behaviour at boot parameters) predict failure behaviour that varies with the interface configuration, because the transfer conditions are the variable.

The experiment is designed so that either class can, in principle, be excluded by the result.

# Command / Procedure

For each configuration in the test set:

1. Set the SPI flash mode and frequency in the build configuration.
2. Rebuild and re-upload the firmware under that configuration (upload and verification per EV-04).
3. Reset the device without BOOT held.
4. Record the ROM output exactly, including the invalid-header value.

Test set:

| Run | Flash mode | Frequency |
|---|---|---|
| 1 | DIO | 80 MHz |
| 2 | DIO | 40 MHz |
| 3 | DOUT | 40 MHz |

Mode and frequency are the only variables changed between runs. Firmware content, board, cabling, and host are unchanged. The stored image is independently proven constant and valid by EV-07 and EV-08.

# Expected Result

- If the fault is content-class: the same invalid-header value (or an identical validation failure) in all three runs, since the ROM reads identical stored data.
- If the fault is interface-class: failure in all runs (the interface remains marginal), but with header values that vary between configurations, since the incorrectly transferred data depends on the transfer conditions.

Successful execution in any run would identify that configuration as a remediation and redirect the investigation accordingly.

# Observed Result

| Flash Mode | Frequency | Result |
|---|---|---|
| DIO | 80 MHz | `invalid header: 0xd610e111` |
| DIO | 40 MHz | `invalid header: 0x8e465484` |
| DOUT | 40 MHz | `invalid header: 0x5691acba` |

No configuration produced a successful boot. Every configuration produced a *different* invalid-header value. Changing the SPI flash configuration changes the invalid-header signature.

# Interpretation

The observed pattern matches the interface-class prediction and contradicts the content-class prediction on its decisive point. The stored data did not change between runs (EV-08); what the ROM received did change, and it changed in step with the interface parameters. The corruption is therefore introduced during the boot-time transfer, under the control of mode and clock settings, not resident in the stored image.

Two further conclusions are supported:

1. **Frequency reduction does not remediate.** Halving the clock from 80 MHz to 40 MHz changed the failure signature but did not resolve the failure. (In the companion single-board investigation recorded in esp32_hardware_test, still lower clocks, 26 MHz and 20 MHz, were also attempted without resolution.) A marginal-timing explanation that predicts recovery at reduced clock is weakened by this result; the fault is configuration-sensitive without being configuration-curable across the tested range.
2. **Mode reduction does not remediate.** DOUT, the least demanding transfer mode tested, fails in the same manner. The failure is not specific to dual- or quad-line operation.

The experiment does not, by itself, identify a mechanism. It establishes the class (interface/transfer at boot, in interaction with the device's configuration state per EV-03) and excludes the alternative class. Mechanism-level discrimination requires bus instrumentation, recorded as future work in the repository README.

# Conclusion

The failure behaviour is sensitive to SPI configuration: the invalid-header value is a function of flash mode and frequency. The fault is localised to the boot-time flash transfer, not to the stored firmware content. No tested configuration remediates the failure.

# Confidence Level

**High**, for the recorded behaviour and the class discrimination (direct observation, single-variable method, replicated across boards). Mechanism attribution is deferred to EV-11 and carries the lower confidence stated there.
