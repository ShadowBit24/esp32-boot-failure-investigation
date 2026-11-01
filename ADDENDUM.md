# Addendum to RCA-001: Follow-Up Recovery Experiment, 2026-11-01

Document ID: ADD-001 (addendum to RCA-001, evidence package EV-11)
Date: 2026-11-01
Status: Published 2026-11-01 as a follow-up record to RCA-001
Related record: RECOVERY_ANALYSIS.md in the esp32-automated-test repository

## Purpose

RCA-001 closed this investigation with a bounded fault domain and three remaining root cause categories. It did not claim a root cause. This addendum records a follow-up experiment, run on 2026-11-01, that tested the second of those categories directly:

- **RCA-CAT-002, Flash Security Configuration Interaction** (RCA-001 assessment: potential contributor, confidence Medium). Supporting evidence in RCA-001: E-14, E-15, E-16, E-17, E-19 (FLASH_CRYPT_CNT = 1, FLASH_CRYPT_CONFIG = 15, RD_DIS = 1, download decrypt disabled, standard erase refused on security grounds).

## The unit under test

One ESP32-D0WD-V3 development board (DevKit V1 class) with the same flash device as the case study batch (manufacturer 0x68, device 0x4016, 4 MB), attached to the host as COM6. At the start of the session the board presented the case study symptom set: repeated `invalid header: 0x3ffea272` at reset (a value recorded in this investigation, see README and EV-09), RTC watchdog resets, and no application execution, while chip-ID, flash-ID, and flash-status diagnostics all passed. A prior eFuse inspection of the board recorded flash encryption enabled (FLASH_CRYPT_CNT = 1).

Scope note: the recovery record does not label this unit as Board A, B, or C from the original three-board matrix. This addendum therefore reports a single-unit result and does not, by itself, change any batch-level statement in RCA-001.

## Experiment

Hypothesis under test: the board's flash encryption was already enabled, while the images being written to it were plaintext. Under that state, the ROM reads encrypted content where it expects a plaintext boot image, which is consistent with invalid-header output at boot while every host-driven operation passes.

Method, as recorded in the recovery analysis (times UTC, 2026-11-01):

1. 04:57, baseline run of the automated validation framework against the board as found: boot classified INVALID_IMAGE, no firmware tests ran, flash diagnostics passed, board health score 15/100 (DEFECTIVE).
2. Built an encryption-aware second-stage bootloader (ESP-IDF 6.1) configured to require flash encryption already enabled. This configuration does not enable encryption and writes no eFuses.
3. Wrote the boot chain with esptool `write-flash --encrypt` (DOUT, 40 MHz, 4 MB): bootloader at 0x1000 (28,288 bytes), the board's existing partition table at 0x8000 (4,096 bytes, layout unchanged, rewritten in encrypted form), the OTA selection record at 0xe000 (8,192 bytes), and the application at 0x10000 (742,848 bytes).
4. Re-ran the validation framework after each stage.

## Observed result

- 05:22 UTC: the new bootloader loaded the partition table and app0, logged that flash encryption is enabled, and the firmware reached its READY announcement and passed all seven on-device tests. (The host run still reported an error at this point because of a missing closing brace in the firmware's summary line, a reporting defect in the test tool, not a boot failure.)
- 05:30 UTC, final recorded run: overall PASS, firmware tests 7/7, flash diagnostics 3/3, Wi-Fi scan detected 10 access points, health score 100/100 (HEALTHY).

No `erase_flash` command appears in the recorded recovery operations, and no eFuse write was performed. The encrypted writes replaced the sectors they targeted, as any flash write does. Whether a full-chip erase occurred at any point in this board's earlier history could not be determined from the retained records and is not claimed either way.

## Interpretation

For this unit, the encryption-mismatch hypothesis accounts for the full symptom pattern that RCA-001 documented: host-driven flash operations succeed because the downloader path works regardless; boot fails because the ROM expects content the plaintext writes did not provide; and the failure clears when the boot chain is written in the form the device's security state requires. The recovery analysis rates this root cause High confidence (0.95) for this board. The mode-dependence recorded in EV-09 (header values changing with SPI settings) is not re-tested by this experiment and is neither confirmed nor contradicted by it.

## Effect on RCA-001

- The RCA-001 fault domain (ROM Bootloader, SPI Flash Interface, Flash Security / Configuration, Application Startup) stands as published. This result sits inside it.
- RCA-CAT-002 is upgraded for this single unit from "potential contributor" to "most likely root cause, high confidence," on the strength of a successful discriminating experiment: change only the form of the written images to match the recorded security state, and the board boots.
- The batch-level disposition in RCA-001 (batch rejected as non-viable for firmware execution, root cause not claimed for the batch) is unchanged. The experiment that would change it is the one RCA-001 already defines in its Future Work, now with a concrete procedure: apply the identical encryption-aware workflow to the remaining boards from the batch and record whether each recovers. If they recover, the batch property is explained by a shared security state, not by three independent hardware defects. If they do not, the domain narrows again, differently.
- The RCA-001 closure statement is unaffected: "closed" there means the localisation and disposition, and this addendum is a follow-up record, not a rewrite of the original analysis.

## Limitations of this addendum

- Single unit, and the unit is not cross-identified against Boards A, B, and C in the source record.
- esptool cannot verify encrypted flash by readback, so the written content is confirmed by boot behaviour and test results, not by byte comparison.
- The board's earlier history is incompletely recorded (shell history without timestamps or outcomes), so prior erase or configuration events cannot be ruled in or out.
- The automated framework's health score is a diagnostic heuristic, reported here because it is how the runs were recorded, not as a certification measure.

## Source

All facts, times, sizes, and confidence values in this addendum are taken from RECOVERY_ANALYSIS.md and the example reports in the esp32-automated-test repository (github.com/ShadowBit24/esp32-automated-test), which preserve the underlying run reports and serial captures.
