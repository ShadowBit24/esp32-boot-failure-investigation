# Investigation Log — ESP32 Multi-Board Boot Failure

Devices under test: ESP32 DevKit V1, Boards A, B, C (single purchase batch, Amazon).
Entries are ordered by execution sequence. Unless an entry names one board, the procedure was run on all three boards with identical results.

---

## Entry 01 — Symptom capture

**Procedure:** Flash known user firmware via the standard upload flow; reset; observe serial console.
**Observed (all boards):** ROM bootloader rejects the boot image. Recorded header values across runs:
```
invalid header: 0x3ffea272
invalid header: 0xd610e111
invalid header: 0x8e465484
invalid header: 0x5691acba
```
A valid ESP32 image header begins with magic byte `0xE9`. The observed values are neither valid nor repeatable.
**Result:** Failure reproduced consistently. Application never executes.

## Entry 02 — Hardware validation

**Procedure:** Connect each board; check host-side fundamentals before any firmware reasoning.
**Checks and results:**
- CP210x USB detection on host: PASS
- Power LED operation: PASS
- UART communication: PASS
- ROM bootloader access: PASS
- Download mode access: PASS

**Result:** PASS. The boards are alive and reachable; the fault is not power, USB, or UART.

## Entry 03 — Download mode test

**Procedure:** Hold BOOT → press EN → release EN; read ROM output.
**Observed:**
```
boot:0x3 (DOWNLOAD_BOOT)
waiting for download
```
**Result:** PASS. Boot-mode strapping and the ROM downloader respond correctly. This also proves the ROM itself executes its own code — the failure is specific to loading the *application* image.

## Entry 04 — Flash detection

**Procedure:** `esptool flash-id`
**Observed:**
```
Manufacturer: 68
Device: 4016
Detected flash size: 4MB
```
**Result:** PASS. The flash device identifies and communicates under host control.

## Entry 05 — Firmware validation

**Procedure:** Upload firmware; allow esptool write and verification to complete; reset and observe.
**Observed:** Upload completes; flash writes complete; esptool verification passes. On reset, the board returns to the Entry 01 symptom.
**Result:** PASS for write/verify path. FAIL for execution (see Entry 10). The divergence between these two results is the central puzzle of the investigation.

## Entry 06 — eFuse analysis

**Procedure:** `espefuse summary`
**Observed:**
```
FLASH_CRYPT_CNT    = 1
FLASH_CRYPT_CONFIG = 15
RD_DIS             = 1
```
**Interpretation recorded at the time:** flash-encryption counter in an enabled state; encryption configuration at 15; eFuse read-disable set. Not the expected state of a factory-virgin device. eFuse access itself works.
**Result:** Access PASS. Values flagged as abnormal and carried into the analysis.

## Entry 07 — Flash erase test

**Procedure:** `esptool erase-flash`
**Observed:** Refused — active security features detected.
**Procedure (continued):** `esptool erase-flash --force`
**Observed:** Flash memory erased successfully.
**Result:** PASS with force. The refusal corroborates Entry 06 behaviourally: security features are active on these devices, by configuration, not by conjecture.

## Entry 08 — Erase verification

**Procedure:** Read flash at `0x1000` (bootloader region) after forced erase.
**Observed:**
```
FF FF FF FF FF FF FF FF
```
**Result:** PASS. The bootloader region reads as fully erased. The flash honours erase and reports the erased state correctly on readback.

## Entry 09 — Firmware readback validation

**Procedure:** Re-flash firmware; dump the flash image; validate the dump structurally: `esptool image-info app.bin`
**Observed:**
```
Checksum: valid
Validation hash: valid
Detected image type: ESP32
```
**Result:** PASS. Three facts established independently of the boot behaviour: the firmware was generated correctly, uploaded correctly, and stored correctly. Whatever the ROM receives at boot, the image *in flash* is right.

## Entry 10 — Flash mode tests

**Procedure:** Rebuild/re-flash under changed SPI configurations, one variable at a time; record the header value at boot.

| Configuration | Observed header | Execution |
|---|---|---|
| DIO @ 80 MHz | `0xd610e111` | FAIL |
| DIO @ 40 MHz | `0x8e465484` | FAIL |
| DOUT @ 40 MHz | `0x5691acba` | FAIL |

**Result:** FAIL for execution in all modes. Critical observation: the invalid-header *value changes with the interface configuration*. Constant corruption of stored data cannot do that; Entry 09 already proved the stored data is valid. The fault is introduced in the boot-time read and is sensitive to interface parameters.

## Entry 11 — Cross-board validation

**Procedure:** Full matrix (Entries 02–10) applied to Boards A, B, and C.
**Observed:**

| Capability | Board A | Board B | Board C |
|---|---|---|---|
| Download mode | PASS | PASS | PASS |
| Flash detection | PASS | PASS | PASS |
| Upload capability | PASS | PASS | PASS |
| Readback capability | PASS | PASS | PASS |
| eFuse accessibility | PASS | PASS | PASS |
| Application execution | FAIL | FAIL | FAIL |

**Result:** The fault replicates identically across the batch. Conclusions may be stated at batch level; single-unit explanations (one damaged board, one bad solder joint) are excluded by replication.

## Entry 12 — Elimination accounting and fault domain

**Eliminated by the entries above:** dead MCU (02, 03); USB failure (02); UART failure (02); download-mode failure (03); flash write failure (05); flash read failure under host control (08, 09); corrupt uploaded image (09).

**Remaining fault domain:**
```
ROM Bootloader → SPI Flash Interface → Flash Security / Configuration → Application Startup
```

**Stopping rule applied:** the investigation stops at the boundary the available instruments can defend. Bus-level capture and a known-good control board are required to go further and are recorded as future work rather than improvised around.
