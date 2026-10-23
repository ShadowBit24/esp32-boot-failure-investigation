# Final Root Cause Analysis (RCA) Summary

Document ID: RCA-001
Package ID: EV-11 (see evidence/README.md)
Investigation Type: Embedded Systems Failure Analysis
Platform: ESP32-D0WD-V3 Development Boards (DevKit V1, Boards 1–3, single purchase batch)
Status: Investigation Complete
Root Cause: Not conclusively isolated
Failure Domain: Narrowed and bounded through evidence-based elimination

*Editorial note (2026-10-23): This document supersedes the original EV-11 analysis file. Its evidence items were reconciled to the published evidence package: duplicate items merged, derived observations relabelled as analysis (D-series) rather than raw evidence, and all items mapped to their source files. Item references in this document (E-01 to E-26, D-01 to D-03) are the canonical item-level references for this investigation.*

## Executive Summary

Three ESP32 DevKit V1 development boards from the same manufacturing batch consistently failed to boot uploaded firmware.

Despite repeated application startup failures, all major subsystems involved in firmware deployment and storage were validated independently. The investigation systematically verified:

- CPU operation
- ROM bootloader operation
- UART communication
- USB communication
- Firmware upload capability
- Flash programming capability
- Flash erase capability
- Flash readback capability
- Firmware image integrity

Application execution remained unsuccessful across all three boards.

## Evidence Index

Raw evidence items. Each item is a single recorded observation and cites the package file that preserves it.

| ID | Description | Source |
|---|---|---|
| E-01 | Download mode entered; ROM responds `boot:0x3 (DOWNLOAD_BOOT)`, "waiting for download" | EV-01 serial_logs.md |
| E-02 | CP210x USB-UART interface detected | EV-10 cross_board_comparison.md |
| E-03 | ESP32 identified as ESP32-D0WD-V3 (revision v3.1) | EV-02 flash_id_results.md |
| E-04 | Flash device detected (Manufacturer 68, Device 4016) | EV-02 flash_id_results.md |
| E-05 | Flash size reported as 4 MB | EV-02 flash_id_results.md |
| E-06 | Firmware upload completed successfully | EV-04 firmware_upload_results.md |
| E-07 | Upload hash of data verified | EV-04 firmware_upload_results.md |
| E-08 | Application image detected as valid ESP32 image | EV-07 image_validation.md |
| E-09 | Image checksum valid | EV-07 image_validation.md |
| E-10 | Image validation hash valid | EV-07 image_validation.md |
| E-11 | DIO @ 80 MHz produced `invalid header: 0xd610e111` | EV-09 flash_mode_experiments.md |
| E-12 | DIO @ 40 MHz produced `invalid header: 0x8e465484` | EV-09 flash_mode_experiments.md |
| E-13 | DOUT @ 40 MHz produced `invalid header: 0x5691acba` | EV-09 flash_mode_experiments.md |
| E-14 | FLASH_CRYPT_CNT = 1 | EV-03 espefuse_results.md |
| E-15 | FLASH_CRYPT_CONFIG = 15 | EV-03 espefuse_results.md |
| E-16 | RD_DIS = 1 | EV-03 espefuse_results.md |
| E-17 | DISABLE_DL_DECRYPT = True | EV-03 espefuse_results.md |
| E-18 | JTAG_DISABLE = True | EV-03 espefuse_results.md |
| E-19 | Standard erase blocked by security protections | EV-05 flash_erase_validation.md |
| E-20 | Forced erase completes successfully | EV-05 flash_erase_validation.md |
| E-21 | Bootloader region reads `FF` after erase; erase verified in the expected address range | EV-06 erase_verification.md |
| E-22 | Firmware read back from flash (327,680 bytes) | EV-08 flash_readback_validation.md |
| E-23 | Read-back firmware validates successfully | EV-08 flash_readback_validation.md |
| E-24 | Board 1 application startup failure | EV-10 cross_board_comparison.md |
| E-25 | Board 2 application startup failure | EV-10 cross_board_comparison.md |
| E-26 | Board 3 application startup failure | EV-10 cross_board_comparison.md |

Derived observations. These are analysis products computed from the raw items above. They are listed separately so that no derived statement is counted as raw evidence.

| ID | Description | Derived from |
|---|---|---|
| D-01 | Invalid-header values change with SPI configuration | E-11, E-12, E-13 |
| D-02 | All boards fail application startup (batch pattern) | E-24, E-25, E-26 |
| D-03 | All boards enter download mode and upload firmware successfully (batch pattern) | E-01, E-06, per EV-10 |

## Findings

### FINDING-001 — ROM Bootloader Operational

- **Evidence:** E-01, D-03
- **Supporting observation:** `rst:0x10 (RTCWDT_RTC_RESET),boot:0x3 (DOWNLOAD_BOOT(UART0/UART1/SDIO_REI_REO_V2))` — "waiting for download"
- **Interpretation:** The ESP32 ROM bootloader consistently enters download mode and remains accessible through UART.
- **Conclusion:** ROM bootloader functionality verified.
- **Status:** ELIMINATED AS FAILURE SOURCE
- **Confidence:** High

### FINDING-002 — USB/UART Communication Operational

- **Evidence:** E-02, D-03
- **Supporting observation:** CP210x USB-UART bridge detected successfully.
- **Interpretation:** Host communication path remains operational throughout testing.
- **Conclusion:** USB communication not contributing to observed failure.
- **Status:** ELIMINATED AS FAILURE SOURCE
- **Confidence:** High

### FINDING-003 — MCU Functional

- **Evidence:** E-01, E-03
- **Supporting observation:** ESP32-D0WD-V3 detected consistently.
- **Interpretation:** CPU executes ROM bootloader code correctly.
- **Conclusion:** Processor not identified as root cause.
- **Status:** ELIMINATED AS FAILURE SOURCE
- **Confidence:** High

### FINDING-004 — SPI Flash Device Detected

- **Evidence:** E-04, E-05
- **Supporting observation:** Manufacturer: 68; Device: 4016; Detected flash size: 4 MB.
- **Interpretation:** The flash device responds correctly to identification commands.
- **Conclusion:** Complete flash-device failure not supported by evidence.
- **Status:** ELIMINATED AS COMPLETE FLASH FAILURE
- **Confidence:** High

### FINDING-005 — Firmware Upload Successful

- **Evidence:** E-06, E-07, D-03
- **Supporting observation:** "Hash of data verified."
- **Interpretation:** Firmware images are successfully transmitted and programmed into flash memory.
- **Conclusion:** Firmware upload process functioning correctly.
- **Status:** ELIMINATED AS FAILURE SOURCE
- **Confidence:** High

### FINDING-006 — Firmware Image Valid

- **Evidence:** E-08, E-09, E-10
- **Supporting observation:** Detected image type: ESP32; Checksum: valid; Validation hash: valid.
- **Interpretation:** The generated application image passes structural integrity verification.
- **Conclusion:** Firmware image corruption not supported.
- **Status:** ELIMINATED AS FAILURE SOURCE
- **Confidence:** High

### FINDING-007 — Firmware Successfully Stored In Flash

- **Evidence:** E-22, E-23
- **Supporting observation:** Read-back image validates successfully.
- **Interpretation:** Application image remains recoverable and valid after storage.
- **Conclusion:** Flash storage integrity verified.
- **Status:** ELIMINATED AS FAILURE SOURCE
- **Confidence:** High

### FINDING-008 — Security Configuration Present

- **Evidence:** E-14, E-15, E-16, E-17, E-18, E-19
- **Supporting observation:** FLASH_CRYPT_CNT = 1; FLASH_CRYPT_CONFIG = 15; RD_DIS = 1; DISABLE_DL_DECRYPT = True; JTAG_DISABLE = True; standard erase refused on security grounds (E-19), behaviourally corroborating the eFuse record.
- **Interpretation:** Security-related eFuse configuration exists consistently across all devices, in a non-factory-virgin state.
- **Conclusion:** Security configuration remains a viable contributing factor.
- **Status:** CONTRIBUTING FACTOR POSSIBLE
- **Confidence:** Medium

### FINDING-009 — Boot Failure Sensitive To SPI Flash Configuration

- **Evidence:** E-11, E-12, E-13, D-01
- **Supporting observation:** DIO @ 80 MHz: `0xd610e111`; DIO @ 40 MHz: `0x8e465484`; DOUT @ 40 MHz: `0x5691acba`.
- **Interpretation:** Changing flash communication configuration changes failure output, while the stored data is proven constant (FINDING-007). The corruption is introduced in the boot-time transfer.
- **Conclusion:** SPI flash communication path strongly implicated.
- **Status:** HIGHLY RELEVANT
- **Confidence:** High

### FINDING-010 — Failure Reproduced Across Three Independent Boards

- **Evidence:** E-24, E-25, E-26, D-02
- **Supporting observation:** Three boards tested. Three boards fail application startup.
- **Interpretation:** Failure not isolated to a single hardware unit.
- **Conclusion:** Underlying cause likely common to batch, configuration, or component selection.
- **Status:** HIGHLY RELEVANT
- **Confidence:** High

## Eliminated Failure Hypotheses

| Hypothesis | Status | Supporting Evidence |
|---|---|---|
| Dead ESP32 MCU | Eliminated | E-01, E-03 |
| Broken USB Interface | Eliminated | E-02 |
| Broken UART Interface | Eliminated | E-01 |
| Download Mode Failure | Eliminated | E-01, D-03 |
| Upload Process Failure | Eliminated | E-06, E-07 |
| Corrupt Firmware Image | Eliminated | E-08, E-09, E-10 |
| Flash Programming Failure | Eliminated | E-06, E-07 |
| Flash Read Failure | Eliminated | E-22, E-23 |
| Erase Failure | Eliminated | E-20, E-21 |
| Single Board Defect | Eliminated | E-24, E-25, E-26 |

## Remaining Fault Domain

Validated functional: ROM Bootloader, USB Interface, UART Interface, Firmware Image, Flash Programming, Flash Readback. The failure sits in the one transition none of those cover:

```text
ROM Bootloader
      ↓
SPI Flash Interface
      ↓
Flash Security / Configuration
      ↓
Application Startup
```

Supporting evidence: D-01, E-14, E-15, E-16, D-02.

Confidence: High (for the domain as bounded; see the certainty distinction in the Closure Statement).

## Probable Root Cause Categories

### RCA-CAT-001 — SPI Flash Communication Compatibility

- **Evidence:** E-11, E-12, E-13, D-01
- **Rationale:** Failure signatures change when SPI communication parameters change.
- **Assessment:** Most strongly supported remaining hypothesis.
- **Confidence:** High that the SPI communication path is implicated in the failure; Medium that it is the sole root cause. The configuration sensitivity is established by direct observation; the mechanism inside the path (timing, signal integrity, device behaviour at boot parameters) is not isolated by the current evidence, and is not claimed.

### RCA-CAT-002 — Flash Security Configuration Interaction

- **Evidence:** E-14, E-15, E-16, E-17, E-19
- **Rationale:** Security-related eFuse settings consistently present on all boards, corroborated behaviourally by the erase interlock.
- **Assessment:** Potential contributor requiring further investigation.
- **Confidence:** Medium

### RCA-CAT-003 — Batch-Specific Hardware or Component Behaviour

- **Evidence:** E-24, E-25, E-26, D-02
- **Rationale:** Three independent boards exhibit nearly identical behavior.
- **Assessment:** Manufacturing-batch-specific issue remains possible.
- **Confidence:** Medium

## Open Questions

Carried from the investigation record; each is paired with its resolving experiment in the repository README (Future Work).

- What is the functional consequence of FLASH_CRYPT_CNT = 1 (with FLASH_CRYPT_CONFIG = 15) for ROM boot-time image handling on these devices?
- What are the transfer characteristics of the manufacturer 68 (BoyaMicro BY25Q32BS) flash under ESP32 boot-mode read patterns, as distinct from downloader read patterns? (Resolving measurement: SPI bus capture during a failed boot.)
- Is the SPI timing/compatibility behaviour at boot within the flash device's correct operating envelope across the tested mode/frequency space?
- How does the observed security configuration interact with the boot path, given that host-driven operations are unaffected?

## Final Engineering Assessment

Validated functional: ESP32 MCU; ROM Bootloader; CP210x Interface; UART Communication; Download Mode; Flash Detection; Flash Programming; Flash Erase; Flash Readback; Firmware Validation.

Validated failing: Application Startup.

Final conclusion: Based on all collected evidence, the most defensible engineering conclusion is that user firmware fails during the transition between ROM bootloader execution and application startup. CPU failure, USB failure, UART failure, flash programming failure, flash read failure, firmware corruption, and download-mode failure were eliminated through direct testing. The remaining fault domain is constrained to the interaction between the ROM bootloader, the SPI flash subsystem, the flash security configuration, and the application startup sequence.

## RCA Closure Statement

- **Investigation Status:** Closed, with disposition. The batch is rejected as non-viable for firmware execution, and the failure is localised to the domain stated above. Closure applies to the localisation and the batch disposition; the mechanism-level open questions listed in this document remain open and are tracked as future work in the repository README. "Closed" in this statement must not be read as "root cause identified".
- **Evidence Confidence:** High
- **Root Cause Certainty:** Medium
- **Failure Domain Certainty:** High
- **Total Evidence Collected:** 26 raw evidence items (E-01 to E-26), plus 3 derived observations (D-01 to D-03) recorded separately as analysis
- **Major Findings:** 10
- **Eliminated Failure Hypotheses:** 10
- **Remaining Root Cause Categories:** 3
