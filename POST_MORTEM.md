# Post-Mortem — ESP32 Batch Boot Failure (Application Execution)

**Incident class:** Hardware validation / board bring-up failure
**Devices:** 3 × ESP32 DevKit V1, single purchase batch (Amazon)
**Severity:** Total loss of function on all units (no application execution)
**Status:** Fault domain isolated. Root cause not isolated to a single component; discriminating experiments defined.

---

## 1. Summary

Three ESP32 DevKit V1 boards from one batch failed to execute any user firmware. At reset, the ROM bootloader rejected the boot image with `invalid header` errors whose values were non-deterministic (`0x3ffea272`, `0xd610e111`, `0x8e465484`, `0x5691acba`). All host-side functions — USB enumeration, UART, ROM download mode, flash identification, flash write, flash readback with structural image validation, eFuse access — passed on all three boards. The failure is confined to the boot-time read/execution path.

## 2. Impact

- 3 of 3 boards unusable for firmware execution. No partial functionality: a board that cannot boot its application has no degraded mode.
- Development on the batch halted at bring-up; no application-level debugging was possible on these units.
- Because all units in the batch are affected identically, any project planned against this batch inherits the failure. The batch is quarantined for execution purposes pending the experiments in §6.

## 3. Detection

The failure was detected at first firmware upload: esptool completed write and verification without error, but the serial console showed ROM `invalid header` output and a reset loop instead of application output. Detection was immediate and unambiguous; the investigation that followed was about localisation, not detection.

## 4. What was verified (evidence summary)

| Domain | Method | Outcome |
|---|---|---|
| USB enumeration | CP210x detection on host | PASS, all boards |
| Power | Power LED operation | PASS, all boards |
| UART / ROM access | Serial console, ROM responses | PASS, all boards |
| Download mode | BOOT/EN strap sequence → `boot:0x3 (DOWNLOAD_BOOT)`, "waiting for download" | PASS, all boards |
| Flash identity | `esptool flash-id` → Mfr `0x68`, Device `0x4016`, 4 MB | PASS, all boards |
| Flash write | Firmware upload + esptool verification | PASS, all boards |
| Flash read | Readback dump + `image-info`: checksum valid, validation hash valid, ESP32 image | PASS, all boards |
| Flash erase | `erase-flash` refused (security features active); `erase-flash --force` succeeded; read at `0x1000` = all `0xFF` | PASS with force |
| eFuse access | `espefuse summary` readable | PASS (access); values abnormal, §5 |
| Application execution | Reset after verified flash, all SPI modes tested | **FAIL, all boards** |

Full procedures and observations: INVESTIGATION_LOG.md. Full matrix: TEST_MATRIX.md.

## 5. Analysis

### 5.1 The symptom is non-deterministic, and that is the key datum
A corrupted image returns the same wrong header every boot. These boards returned *different* wrong headers, and the value changed systematically with SPI configuration: DIO @ 80 MHz → `0xd610e111`; DIO @ 40 MHz → `0x8e465484`; DOUT @ 40 MHz → `0x5691acba`. The stored data is provably correct (readback + image validation). Therefore the corruption is introduced at boot-time read, and its dependence on mode/clock identifies the interface behaviour — not the data — as a variable in the fault.

### 5.2 The downloader path and the boot path diverge
Under ROM download mode, the host drives flash operations and everything passes. At autonomous boot, the ROM reads the same flash under boot parameters and fails. Any hypothesis must explain both halves. "Bad flash contents" fails (contents validate). "Dead board" fails (ROM, UART, and download functions all work). Hypotheses consistent with both halves live in: boot-time SPI interface behaviour (timing/signal integrity at boot settings), or security/configuration state that alters boot behaviour but not downloader behaviour.

### 5.3 Security configuration is abnormal
`FLASH_CRYPT_CNT = 1` (flash-encryption counter in an enabled state), `FLASH_CRYPT_CONFIG = 15`, `RD_DIS = 1` (eFuse read-disable set), corroborated behaviourally by the erase refusal until `--force`. Boards sold as new are not expected to present with encryption counter and read-disable state set. This does not by itself prove the boot mechanism — interactions between flash encryption state, image encryption expectations, and boot header parsing are configuration-specific — but it places the flash security/configuration domain inside the fault boundary and explains behaviour (erase refusal) that the interface hypothesis alone does not.

### 5.4 Batch replication changes the claim
Identical results across three units move the conclusion from "a defective board" to "a property of this batch" (common flash device, common prior state, or common manufacturing/provisioning history). Single-unit reasoning would have been misleading here.

## 6. Conclusion and next experiments

**Conclusion (as supported):** the fault domain is

```
ROM Bootloader → SPI Flash Interface → Flash Security / Configuration → Application Startup
```

with seven alternative domains eliminated by test. No single-component root cause is claimed.

**Discriminating experiments, in order of information per hour:**
1. Known-good control board (different batch/vendor) through the identical matrix. Separates batch property from test-setup artefact. If the control passes test 10 with the same host/tools, the setup is exonerated completely.
2. Logic-analyzer capture of the SPI bus during failed boot vs download-mode read. Directly tests the marginal-interface hypothesis and shows what the ROM actually receives.
3. Flash device substitution on one unit. Separates flash-device behaviour from SoC/eFuse state.
4. Field-by-field eFuse comparison against a virgin board.

## 7. Relation to prior work

An earlier single-board investigation (see the companion repository, esp32_hardware_test) examined the same fault class on a unit with the same flash device family (manufacturer `0x68`), recorded header `0x6aebae05`, and observed a successful boot after forcing DOUT @ 40 MHz.

This study could **not** reproduce a successful application boot at DOUT @ 40 MHz, or in any other tested mode, on any of the three boards. The earlier observation is therefore treated as unit-specific or intermittent, not as a general remediation, and the earlier post-mortem's root-cause framing ("signal timing degradation", stated as determined) is narrowed by this study to one hypothesis inside the fault domain in §6. The earlier post-mortem has since been corrected by its author (2026-10-23) to record the outcome as unresolved and the remediation attempts as unsuccessful, consistent with this study; where its original framing differed, this study's replicated data governs.

Recording this plainly matters: the difference between the two documents is the difference between a single-unit anecdote and a replicated result, and a reader is entitled to see both.

## 8. Corrective actions (process)

- Quarantine the batch for execution-dependent work; retain one unit as the investigation reference.
- Add a boot-execution check to incoming-board acceptance: flash a known image, verify application output, not merely flash verification. Flash verification alone would have accepted these boards.
- Add an eFuse summary capture to first-power-on records for new boards. Security state is part of a board's identity and is cheapest to record before anything writes to the device.
- Any future board purchase for a project: buy one unit first, run acceptance, then buy the batch.
