# ESP32 Boot Failure Investigation — Multi-Board Case Study

A controlled fault-isolation investigation into a batch-level ESP32 boot failure: three ESP32 DevKit V1 boards from a single purchase batch that accept, store, and verify firmware correctly, yet never execute it.

This repository is an engineering case study. The deliverable is the investigation itself: the test matrix, the evidence, the eliminated fault domains, and the fault domain that remains.

---

# Overview

Three ESP32 DevKit V1 boards from the same Amazon batch consistently failed to boot user firmware. At reset, the ROM bootloader reported `invalid header` errors with values that changed between runs and between SPI configurations:

```
invalid header: 0x3ffea272
invalid header: 0xd610e111
invalid header: 0x8e465484
invalid header: 0x5691acba
```

A valid ESP32 image header begins with magic byte `0xE9`. None of the observed values match, and none repeat reliably.

Every other subsystem on the boards tested functional: USB enumeration, UART communication, ROM download mode, flash identification, flash write, flash readback with image validation, and eFuse access all passed on all three boards. The failure is confined to application execution.

**Outcome:** seven fault domains eliminated with evidence. Remaining fault domain isolated to the ROM bootloader → SPI flash interface → flash security/configuration path. Final root cause not claimed; the next discriminating experiments are defined in [Future Work](#future-work).

---

# Problem Statement

Boards that pass every upload and verification step but never run the uploaded firmware break the assumption most debugging starts from ("if it flashed and verified, the board works"). The investigation had to determine, with evidence, which stage of the boot chain fails:

```
Power/Reset → ROM Bootloader → SPI Flash Read → Image Header Parse → Application Startup
```

...and to do so in a way that distinguishes a single defective unit from a batch-level condition.

---

# Objectives

1. Reproduce the boot failure consistently and record its exact symptoms.
2. Verify each subsystem independently (USB, UART, power, download mode, flash identity, flash write/read, image integrity, eFuse state).
3. Test whether SPI flash mode or clock frequency changes the failure.
4. Replicate the result across all three boards from the batch before generalising any conclusion.
5. Eliminate fault domains explicitly; claim only what the evidence supports.

---

# Hardware Setup

| Item | Detail |
|---|---|
| DUT | 3 × ESP32 DevKit V1, single purchase batch (Amazon) |
| USB-UART bridge | CP210x (enumerates correctly on all boards) |
| Flash device | Manufacturer `0x68`, Device `0x4016`, 4 MB (per `esptool flash-id`) |
| Host connection | USB, serial console at ROM bootloader verbosity |

Board identifiers in this study: Board A, Board B, Board C (one batch, three units).

---

# Software Setup

| Tool | Use |
|---|---|
| esptool | `flash-id`, `erase-flash`, `erase-flash --force`, image readback, `image-info` validation |
| espefuse | eFuse summary and security-state inspection |
| Serial monitor | ROM bootloader output capture |

---

# Architecture

The boot chain and diagnostic setup are documented in [assets/architecture.md](assets/architecture.md). In brief: on reset, the ESP32 ROM bootloader reads the boot image header from SPI flash; a valid header (magic `0xE9`) leads to image load and application entry. In this study the header returned over the SPI interface is non-deterministic garbage, so execution never starts, while host-initiated flash operations over the same physical interface succeed.

---

# Methodology

1. **Symptom capture first.** Record exact ROM output before changing anything.
2. **Subsystem verification.** Test each link in the chain independently, cheapest and most discriminating tests first.
3. **One variable at a time.** SPI mode and frequency were changed individually; the header value was recorded after each change.
4. **Readback validation.** Never trust a write: dump the flash and validate the image structure (`image-info`: checksum and validation hash) before concluding the stored firmware is correct.
5. **Security state audit.** Inspect eFuse configuration, because flash encryption and read-disable state affect both erase behaviour and boot behaviour.
6. **Cross-board replication.** Repeat the full matrix on all three boards. A result that does not replicate is an anecdote, not a finding.
7. **Elimination accounting.** Maintain the list of eliminated domains and the evidence that eliminated each.

Full procedure and raw observations: [INVESTIGATION_LOG.md](INVESTIGATION_LOG.md). Full matrix: [TEST_MATRIX.md](TEST_MATRIX.md).

---

# Test Matrix

Summary (full matrix with procedures and observed values in [TEST_MATRIX.md](TEST_MATRIX.md)):

| # | Test | Result |
|---|---|---|
| 1 | Hardware validation (USB, power LED, UART, ROM access, download mode) | PASS |
| 2 | Download mode entry (`boot:0x3 (DOWNLOAD_BOOT)`) | PASS |
| 3 | Flash detection (`flash-id`: 0x68 / 0x4016 / 4 MB) | PASS |
| 4 | Firmware upload, flash write, write verification | PASS |
| 5 | eFuse audit (state recorded, see Evidence) | PASS (access) — abnormal values, see RCA |
| 6 | Flash erase (required `--force`; security features active) | PASS with force |
| 7 | Erase verification at `0x1000` (all `0xFF`) | PASS |
| 8 | Firmware readback + `image-info` (checksum valid, hash valid, ESP32 image) | PASS |
| 9 | SPI mode/frequency sweep (DIO 80 MHz, DIO 40 MHz, DOUT 40 MHz) | FAIL — application still does not execute; header value changes per mode |
| 10 | Cross-board replication (Boards A, B, C) | All boards identical: PASS on tests 1–8, FAIL on application execution |

**Score: 9 of 10 test groups pass. The only failing behaviour is application execution.**

---

# Investigation Process

The investigation proceeded from the outside of the system inward: host interface (USB/UART) → ROM access (download mode) → flash identity → flash write → flash readback and image validation → security configuration (eFuse) → interface parameters (SPI mode/clock) → replication across boards.

The troubleshooting flow, generalised for reuse on any "flashes fine, won't boot" board, is in [assets/troubleshooting_flowchart.md](assets/troubleshooting_flowchart.md). The fault tree, with eliminated and open branches marked, is in [assets/failure_tree.md](assets/failure_tree.md).

---

# Evidence

The full evidence package, eleven structured evidence files (EV-01 to EV-11) plus the photographic record, is published in [evidence/](evidence/README.md). Each file follows a fixed format (Purpose, Command / Procedure, Expected Result, Observed Result, Interpretation, Conclusion, Confidence Level) and preserves the recorded data verbatim. Key observations, as recorded (full log in [INVESTIGATION_LOG.md](INVESTIGATION_LOG.md)):

**Boot failure (all boards):** ROM reports `invalid header` with non-repeating values: `0x3ffea272`, `0xd610e111`, `0x8e465484`, `0x5691acba`.

**Download mode:**
```
boot:0x3 (DOWNLOAD_BOOT)
waiting for download
```

**Flash identification:**
```
Manufacturer: 68
Device: 4016
Detected flash size: 4MB
```

**eFuse state (espefuse summary):**
```
FLASH_CRYPT_CNT    = 1
FLASH_CRYPT_CONFIG = 15
RD_DIS             = 1
```

**Erase behaviour:** `esptool erase-flash` reported active security features; `erase-flash --force` completed. Post-erase read at `0x1000` returned `FF FF FF FF FF FF FF FF`, confirming the bootloader region was erased.

**Readback validation:** dumped image passed `esptool image-info`: checksum valid, validation hash valid, image type ESP32.

**Mode dependence of the fault:**
| Configuration | Observed header |
|---|---|
| DIO @ 80 MHz | `0xd610e111` |
| DIO @ 40 MHz | `0x8e465484` |
| DOUT @ 40 MHz | `0x5691acba` |

Changing SPI mode/frequency changes the (still invalid) header value. The data returned at boot is wrong *differently* under different interface settings.

---

# Results

- The boards are not dead: ROM executes, download mode works, the host interface is fully functional.
- The flash devices identify, erase, write, and read back correctly under host control; the stored image is byte-valid by esptool's own structural checks.
- Application execution fails on all three boards, in all tested SPI modes and frequencies.
- eFuse state is not factory-virgin on these boards (see Root Cause Analysis).
- The fault therefore sits between "valid image stored in flash" and "application entry": the ROM → flash read path at boot, and/or the security/configuration state that governs it.

---

# Root Cause Analysis

**Eliminated, with evidence:** dead MCU; USB failure; UART failure; download-mode failure; flash write failure; flash read failure (under host control); corrupt uploaded image.

**Not eliminated — the remaining fault domain:**

```
ROM Bootloader → SPI Flash Interface → Flash Security / Configuration → Application Startup
```

Two findings point into this domain:

1. **Marginal boot-time flash reads.** Host-driven reads (via the ROM downloader) return correct data, and image validation passes; boot-time reads return non-deterministic garbage whose value tracks SPI mode and clock settings. The read path behaves differently under boot conditions than under downloader conditions. This is characteristic of a marginal interface (timing, signal integrity, or flash device behaviour at boot parameters), not of fixed data corruption: fixed corruption would return the *same* wrong value every time.
2. **Abnormal security configuration.** `FLASH_CRYPT_CNT = 1` indicates the flash-encryption counter is in an enabled state, `FLASH_CRYPT_CONFIG = 15`, and `RD_DIS = 1` (eFuse read-disable set). A standard erase also refused to run until forced, consistent with active security features. Virgin boards are not expected to present this way; the state is consistent with flash/eFuse configuration that interacts with the boot process and was not fully characterised by host-side tests.

**What is NOT claimed:** a single defective component. The evidence narrows the fault to a domain spanning the SPI flash interface and the flash security/configuration state. Discriminating further requires bus-level measurement and a known-good control board (see Future Work). This report claims a fault domain, not a root cause, because that is what the data supports.

Full analysis: [POST_MORTEM.md](POST_MORTEM.md).

---

# Limitations

- **No control board.** All three boards came from one batch; there is no known-good unit from a different batch in the study. Batch-level and design-level causes cannot be separated without one.
- **No bus instrumentation.** No oscilloscope or logic-analyzer captures of the SPI lines during a failed boot were taken. Interface marginality is inferred from mode/frequency dependence, not measured directly.
- **eFuse interpretation is bounded.** eFuse values are reported as read. Their full interaction with this flash device and boot configuration is characterised as a hypothesis to test, not a proven mechanism.
- **Host-side readback passes.** esptool readback validates the image as stored; it cannot observe what the ROM actually receives at boot. The gap between those two reads is precisely the open fault domain.

---

# Lessons Learned

Full version: [LESSONS_LEARNED.md](LESSONS_LEARNED.md). In brief:

- "Flashes and verifies" proves the *downloader* path, not the *boot* path. They share hardware and differ in conditions; a fault can live in exactly one of them.
- A symptom that changes when you change interface parameters is information: non-deterministic wrong data is a different fault class from constant wrong data, and it redirects the investigation from content to interface.
- Check security state early on any board that behaves inconsistently. eFuse configuration explained this study's erase behaviour and reframed the boot problem.
- Replicate across units before concluding. One board is a defect; three boards from one batch is a batch property.
- Record eliminated domains with the same care as findings. The elimination list is what makes the remaining domain defensible.

---

# Future Work

1. **Control experiment:** run the identical matrix on a known-good ESP32 board from a different batch/vendor. This is the highest-value next test.
2. **Bus capture:** logic-analyzer capture of SPI CLK/CS/DATA during failed boot vs download-mode read, to test the marginal-interface hypothesis directly.
3. **Flash substitution:** on one board, replace the flash device (or boot from a known-good external image source) to separate flash-device behaviour from SoC/eFuse state.
4. **eFuse deep-dive:** full espefuse dump comparison against a virgin board, field by field.
5. **Extend the evidence package:** the structured evidence records and board photographs are published under [evidence/](evidence/README.md). Original timestamped terminal capture files will be added alongside them as bench work continues.

---

## Repository contents

| File | Contents |
|---|---|
| README.md | This document |
| POST_MORTEM.md | Formal post-mortem: impact, analysis, fault domain, corrective next steps |
| INVESTIGATION_LOG.md | Step-by-step investigation record with commands and observations |
| TEST_MATRIX.md | Complete test matrix with procedures, expected vs observed, per-board results |
| LESSONS_LEARNED.md | Transferable lessons for board bring-up and validation work |
| RESUME_EXTRACT.md | Resume bullets, skills, and interview talking points drawn from this study |
| evidence/ | The evidence package: 11 structured evidence files (EV-01 to EV-11) with an index, plus board photographs |
| assets/architecture.md | Boot chain and diagnostic setup, with diagrams |
| assets/failure_tree.md | Fault tree with eliminated and open branches |
| assets/troubleshooting_flowchart.md | Reusable diagnostic flow for "flashes fine, won't boot" failures |

## Related work

This case study extends an earlier single-board investigation and diagnostic harness: [esp32_hardware_test](https://github.com/ShadowBit24/esp32_hardware_test). The relationship between the two, including where this study's data narrows the earlier conclusion, is documented in POST_MORTEM.md §7.
