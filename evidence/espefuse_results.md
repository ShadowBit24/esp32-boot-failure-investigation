# eFuse Security Configuration Results

Evidence ID: EV-03
Related test matrix entries: T9 (see ../TEST_MATRIX.md)
Device under test: ESP32 DevKit V1 (Boards 1, 2, 3 — single purchase batch)

# Purpose

To audit the ESP32 eFuse configuration of the boards under test, with specific attention to security-related fields (flash encryption state, read-disable state, download-mode restrictions, JTAG state), and to establish whether that configuration is (a) readable, (b) consistent across the batch, and (c) consistent with the state expected of factory-new, unprovisioned devices.

eFuse state is one-time-programmable and cannot be altered by any firmware operation performed in this investigation. It is therefore stable evidence: what is recorded here describes the devices as received.

# Command / Procedure

Each board is placed in download mode (Procedure A, serial_logs.md, EV-01) and its eFuse summary is read with espefuse:

```bash
espefuse.exe --port COM6 summary
```

The security-relevant fields are extracted from the summary output. The procedure is read-only; no eFuse was burned at any point in this investigation.

# Expected Result

For factory-new, unprovisioned ESP32 devices, the expected state is: flash encryption counter at its initial (disabled) value, no read-disable asserted on the relevant blocks, and no download-mode or debug restrictions beyond defaults. Deviation from this state indicates the devices were provisioned, configured, or otherwise written to before receipt.

# Observed Result

```text
FLASH_CRYPT_CNT    = 1
FLASH_CRYPT_CONFIG = 15
RD_DIS             = 1
DISABLE_DL_DECRYPT = True
JTAG_DISABLE       = True
```

| Field | Observed value | Reference state (unprovisioned device) |
|---|---|---|
| FLASH_CRYPT_CNT | 1 | 0 (flash encryption disabled) |
| FLASH_CRYPT_CONFIG | 15 | 0 |
| RD_DIS | 1 | 0 (no read-disable asserted) |
| DISABLE_DL_DECRYPT | True | False |
| JTAG_DISABLE | True | False |

Observed on: Board 1, Board 2, Board 3. All boards share a similar security configuration.

# Interpretation

Three independent observations converge:

1. **Flash encryption state is non-initial.** On ESP32, flash encryption is enabled when FLASH_CRYPT_CNT has an odd number of bits set; a value of 1 is an enabled state, and FLASH_CRYPT_CONFIG = 15 (0xF) is the fully populated configuration value. These are not the values of a device that has never been configured.
2. **Read-disable is asserted.** RD_DIS = 1 restricts subsequent reading of eFuse block contents, a step normally taken deliberately during provisioning, not a manufacturing default.
3. **The state is corroborated behaviourally.** DISABLE_DL_DECRYPT = True is consistent with the erase behaviour recorded in EV-05, where a standard erase was refused on security grounds and a forced erase was required. The eFuse record and the tool behaviour agree, which excludes a misread of either.

JTAG_DISABLE = True is recorded for completeness. It has no bearing on the boot failure (JTAG is not in the boot path), but it is part of the same pattern: these devices present a provisioned security posture.

The batch consistency of this state is itself evidence. Three independently packaged boards do not arrive at identical non-default eFuse configurations by random defect; the configuration was applied to the batch, or to the devices before board assembly, by a deliberate process whose nature is outside the reach of the available instruments.

# Conclusion

A security-related eFuse configuration is present, and is present consistently, across all devices in the batch. The devices are not in a factory-virgin configuration state. This configuration domain remains inside the fault boundary for the boot failure (see EV-11); the precise mechanism by which it interacts with boot-time image handling is an open question recorded in EV-11, not a conclusion of this file.

# Confidence Level

**High**, for the facts recorded (field values, batch consistency, non-virgin state). The causal weight of this configuration in the boot failure is explicitly **not** rated here; it is assessed in final_fault_domain_analysis.md (EV-11).
