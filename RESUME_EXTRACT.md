# Resume Extract — ESP32 Boot Failure Investigation

Copy-ready material drawn only from this repository's documented evidence.

---

## Resume bullet points

- Led a structured fault-isolation investigation into a batch-level ESP32 boot failure affecting 3 DevKit V1 boards from one production batch, all failing to execute verified firmware with non-deterministic `invalid header` errors at the ROM bootloader stage.
- Designed and executed a 17-test validation matrix (USB enumeration, UART/ROM access, download mode, flash identification, eFuse audit, erase and erase-verification, write/readback with structural image validation, SPI mode/frequency sweep, cross-board replication), isolating the failure to application execution only.
- Eliminated seven fault domains with evidence (MCU, USB, UART, download mode, flash write, flash read, corrupt image) and narrowed the remaining fault domain to the ROM bootloader → SPI flash interface → flash security/configuration path.
- Identified abnormal eFuse state on all units (FLASH_CRYPT_CNT = 1, FLASH_CRYPT_CONFIG = 15, RD_DIS = 1) via espefuse, and showed the invalid-header value varies with SPI mode/frequency, characterising a marginal boot-time flash-read fault rather than data corruption.
- Documented the study as a complete engineering record: post-mortem, investigation log, test matrix, failure tree, troubleshooting flowchart.

## Technical skills demonstrated

- esptool: flash-id, erase-flash (incl. forced erase under active security features), flash readback, image-info structural validation
- espefuse: security-state audit and interpretation
- ESP32 ROM bootloader behaviour and boot-image header structure
- SPI flash interface modes (DIO / DOUT) and clock-frequency effects
- UART/serial diagnostics, CP210x USB-UART bring-up

## Engineering concepts demonstrated

- Fault isolation by systematic elimination
- Controlled single-variable experiments
- Cross-sample (multi-unit) replication before generalisation
- Distinction between root cause and fault domain; evidence-scoped conclusions
- Security configuration (eFuse / flash encryption) analysis
- Pass/fail test design and coverage accounting

## Interview talking points

1. **Why the changing header value matters.** Constant wrong data = corrupt content. Changing wrong data = unreliable transfer. That single observation redirects the whole investigation, and the mode-dependence table proves it.
2. **The eFuse finding.** What FLASH_CRYPT_CNT = 1 and RD_DIS = 1 on supposedly new boards suggest, why the standard erase refused, and what you would compare against a virgin board.
3. **Why you did not claim a root cause.** The evidence isolates a domain, not a component. Claiming more would be guessing; the report defines the three experiments that would discriminate further.
4. **The control-board gap.** The study's main limitation, stated in its own Limitations section: no known-good board from another batch. Naming your own study's weakness before the interviewer does is the point.
5. **Revision of earlier work.** An earlier single-board investigation reached a tidier conclusion this study could not reproduce. How you handled that (preserved, documented, superseded) is a professional-judgement story.

## Why a hiring manager cares

Hardware validation and test engineering consist of exactly this: a failing unit, a symptom, a matrix, disciplined elimination, and a conclusion scoped to the evidence. This repository demonstrates the full cycle on a real fault, self-directed, with documentation at the standard expected of a working validation engineer.
