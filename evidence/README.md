# Evidence Package — Index

Raw and structured evidence for the ESP32 multi-board boot failure investigation. Each file follows the same format (Purpose, Command / Procedure, Expected Result, Observed Result, Interpretation, Conclusion, Confidence Level) and preserves the recorded data verbatim. Conclusions across the package are traceable: every claim in ../POST_MORTEM.md and ../TEST_MATRIX.md resolves to one or more files below.

## Evidence files

| ID | File | Establishes | Test matrix |
|---|---|---|---|
| EV-01 | [serial_logs.md](serial_logs.md) | ROM and download mode functional; boot fails at header validation with non-deterministic values | T2–T5, T14–T16 |
| EV-02 | [flash_id_results.md](flash_id_results.md) | Flash identified: BoyaMicro `0x68` / `0x4016`, 4 MB; device responsive under host control | T6 |
| EV-03 | [espefuse_results.md](espefuse_results.md) | Non-virgin security eFuse state, identical on all 3 boards | T9 |
| EV-04 | [firmware_upload_results.md](firmware_upload_results.md) | Upload completes with hash verification; failure occurs on the subsequent reset | T7, T8 |
| EV-05 | [flash_erase_validation.md](flash_erase_validation.md) | Standard erase refused on security grounds; forced erase succeeds | T10, T11 |
| EV-06 | [erase_verification.md](erase_verification.md) | Bootloader region reads `0xFF` after erase; erase effective in the expected range | T12 |
| EV-07 | [image_validation.md](image_validation.md) | Application image structurally valid (checksum, validation hash, ESP32 image) | T13 |
| EV-08 | [flash_readback_validation.md](flash_readback_validation.md) | Stored image read back from flash (327,680 bytes) and re-validated | T13 |
| EV-09 | [flash_mode_experiments.md](flash_mode_experiments.md) | Failure signature varies with SPI mode/frequency; no configuration remediates | T14–T16 |
| EV-10 | [cross_board_comparison.md](cross_board_comparison.md) | Identical pass/fail pattern on all 3 boards; batch-level property | T17 |
| EV-11 | [final_fault_domain_analysis.md](final_fault_domain_analysis.md) | Eliminations, retained fault domain, open questions, confidence statement | All |

## Photographic record

| File | Contents |
|---|---|
| [photos/board-front-powered-usb.jpg](photos/board-front-powered-usb.jpg) | Board under USB power during evaluation; power LED illuminated (supports T2) |
| [photos/board-front-module-detail.jpg](photos/board-front-module-detail.jpg) | Component side detail: ESP-32 module, USB-UART device (U2), BOOT and EN buttons, pin silkscreen |
| [photos/board-back-devkitv1.jpg](photos/board-back-devkitv1.jpg) | Reverse side: "ESP32 DEVKITV1" silkscreen, confirming board identity |

## Provenance

Tool outputs and serial records were captured during the investigation on the investigator's bench (host port COM6, Windows tooling: esptool.exe, espefuse.exe, certutil). Board photographs were supplied by the investigator on 2026-10-23 and are published unedited except for orientation correction and format conversion. Data values in this package are as recorded; where a record is an excerpt (EV-06), it is labelled as such in its file.
