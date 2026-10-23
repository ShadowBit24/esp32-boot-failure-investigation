# Test Matrix — ESP32 Multi-Board Boot Failure Investigation

Devices: ESP32 DevKit V1, Boards A / B / C, single purchase batch.
Standard: a test passes only on observed behaviour, recorded at run time. "PASS (access)" means the function operated; it does not certify the values it returned as normal.

---

## Matrix

| ID | Test | Procedure | Tool | Expected | Observed | Result |
|---|---|---|---|---|---|---|
| T1 | USB enumeration | Connect board; confirm CP210x device on host | Host OS | Device enumerates | CP210x detected, all boards | PASS |
| T2 | Power indication | Observe power LED on connection | Visual | LED on | LED on, all boards | PASS |
| T3 | UART communication | Open serial console; exchange with ROM | Serial monitor | ROM responds | ROM output received | PASS |
| T4 | ROM bootloader access | Reset; read ROM boot output | Serial monitor | Boot log printed | Boot log printed, incl. `invalid header` errors | PASS (access) |
| T5 | Download mode entry | Hold BOOT, press EN, release EN | Strapping + serial | `boot:0x3 (DOWNLOAD_BOOT)`, "waiting for download" | Exactly as expected | PASS |
| T6 | Flash identification | Query flash ID | `esptool flash-id` | Valid manufacturer/device, size detected | Mfr `0x68`, Device `0x4016`, 4 MB | PASS |
| T7 | Firmware upload | Flash user firmware | esptool | Write completes | Write completes, all boards | PASS |
| T8 | Write verification | esptool verify after write | esptool | Verify OK | Verify OK, all boards | PASS |
| T9 | eFuse audit | Read eFuse summary | `espefuse summary` | Readable; virgin state expected on new boards | Readable. `FLASH_CRYPT_CNT=1`, `FLASH_CRYPT_CONFIG=15`, `RD_DIS=1` — not virgin state | PASS (access) / ABNORMAL values |
| T10 | Flash erase (standard) | Erase entire flash | `esptool erase-flash` | Erase completes | Refused: active security features detected | FAIL (blocked by security state) |
| T11 | Flash erase (forced) | Erase with force flag | `esptool erase-flash --force` | Erase completes | Erase completed successfully | PASS |
| T12 | Erase verification | Read bootloader region `0x1000` | esptool read | All bytes `0xFF` | `FF FF FF FF FF FF FF FF` | PASS |
| T13 | Readback integrity | Dump flash; validate image structure | `esptool image-info` | Checksum valid, hash valid, ESP32 image | Checksum valid; validation hash valid; image type ESP32 | PASS |
| T14 | Boot, DIO @ 80 MHz | Reset with firmware flashed in DIO/80 config | Serial monitor | Application executes | `invalid header: 0xd610e111`; no execution | FAIL |
| T15 | Boot, DIO @ 40 MHz | As T14, DIO @ 40 MHz | Serial monitor | Application executes | `invalid header: 0x8e465484`; no execution | FAIL |
| T16 | Boot, DOUT @ 40 MHz | As T14, DOUT @ 40 MHz | Serial monitor | Application executes | `invalid header: 0x5691acba`; no execution | FAIL |
| T17 | Cross-board replication | T1–T16 applied to Boards A, B, C | All above | Consistent per board | Identical pattern on all three boards | CONFIRMED (batch-level) |

## Coverage summary

| Category | Tests | Pass | Fail | Notes |
|---|---|---|---|---|
| Host interface (USB/UART/power) | T1–T3 | 3 | 0 | |
| ROM functions | T4–T5 | 2 | 0 | |
| Flash operations | T6–T8, T11–T13 | 6 | 0 | T10 counted separately below |
| Security configuration | T9–T10 | 0 clean | 1 blocked | Access works; state is abnormal and blocks standard erase |
| Application execution | T14–T16 | 0 | 3 | The defect under investigation |
| Replication | T17 | confirmed | | Batch-level property established |

## Reading this matrix

- The pass/fail pattern is the finding: everything a host can drive passes; the one autonomous behaviour (boot execution) fails. Any proposed root cause must reproduce this exact pattern, including the mode-dependence in T14–T16 and the security-state facts in T9–T10.
- T9 is deliberately not scored as a simple pass. eFuse *access* passed; the *state* is evidence, and it is abnormal for new boards.
- Tests not yet run (recorded as gaps, not skipped silently): known-good control board through T1–T17; SPI bus capture during T14–T16; flash substitution. See POST_MORTEM.md §6.
