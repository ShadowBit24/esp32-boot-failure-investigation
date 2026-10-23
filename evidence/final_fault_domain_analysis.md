# Final Fault Domain Analysis

Evidence ID: EV-11
Related documents: all evidence files EV-01 to EV-10; ../POST_MORTEM.md; ../TEST_MATRIX.md
Devices under test: ESP32 DevKit V1 (Boards 1, 2, 3 — single purchase batch)

# Purpose

To consolidate the evidence package into a final fault localisation statement: which parts of the system are confirmed functional, which single behaviour is confirmed failing, what fault domain remains after all eliminations, and which questions remain open. This file is the analytical terminus of the investigation. It claims exactly what the preceding evidence supports, and it records, with equal care, what the evidence does not support.

# Command / Procedure

Analytical procedure (no new measurements are introduced in this file):

1. List every capability exercised in the investigation and its verdict, with the evidence file that establishes it.
2. List every fault hypothesis considered and the evidence that eliminates or retains it.
3. State the minimal domain consistent with all retained observations, including the two that constrain it most tightly: the mode/frequency sensitivity of the failure (EV-09) and the batch-wide security configuration state (EV-03), corroborated behaviourally (EV-05).
4. Record open questions and the experiments that would resolve them.

# Expected Result

A defensible analysis converges: the retained domain must explain (a) flawless host-driven operation of every subsystem, (b) failure of exactly one autonomous behaviour, (c) batch-wide replication, and (d) non-deterministic, configuration-sensitive failure data. Any proposed domain failing to explain all four is rejected.

# Observed Result

## Confirmed functional (by direct measurement)

| Subsystem / capability | Evidence |
|---|---|
| ESP32 CPU and ROM bootloader execution | EV-01 (download mode operates; boot stages reached) |
| CP210x USB interface | EV-10 (enumeration, all boards) |
| UART communication | EV-01, EV-04 |
| Download mode (BOOT/EN strapping) | EV-01, Procedure A |
| Flash detection and identification | EV-02 |
| Flash write | EV-04 |
| Flash readback (host-driven) | EV-08 |
| Flash erase (forced) and erase state | EV-05, EV-06 |
| Firmware image validity | EV-07 |

## Confirmed failing (by direct measurement)

| Behaviour | Evidence |
|---|---|
| Application startup (autonomous boot execution) | EV-01 Procedure B; EV-09; EV-10 |

## Eliminated fault domains

Dead MCU; USB failure; UART failure; download-mode failure; flash write failure; flash storage corruption; corrupt or mis-built firmware image; failed erase; single-unit defect.

## Remaining fault domain

```text
ROM Bootloader
      ↓
SPI Flash Interface
      ↓
Flash Security / Configuration
      ↓
Application Startup
```

# Interpretation

The system's behaviour partitions cleanly. Every operation the host can drive succeeds; the one operation the device performs autonomously, reading its boot image under boot parameters, fails. The fault therefore resides not in any component's basic function, all of which are demonstrated, but in the boot-phase interaction among the three middle elements of the domain chain: the ROM's read of the flash over the SPI interface, under the device's security and configuration state.

Two retained facts constrain mechanism hypotheses:

1. **Configuration sensitivity without remediation (EV-09).** The invalid-header value varies deterministically with SPI mode and frequency, while no tested configuration boots. The transfer at boot is therefore implicated directly, and simple explanations in terms of a single marginal setting are excluded across the tested range.
2. **Non-virgin security state, batch-wide (EV-03, EV-05).** Flash encryption counter active, configuration populated, read-disable asserted, download decryption disabled: a provisioned posture, identical across the batch, whose interaction with boot-time image handling (including any expectation of encrypted versus plaintext image handling at boot) is a retained candidate mechanism that host-side tests cannot exonerate, precisely because host-side operations are performed by the downloader under its own rules.

The open branch concerning test-setup artefact is assessed as low: the identical setup passes these boards on six host-driven capabilities, and the failure pattern is too specific (one capability, configuration-sensitive data) to attribute to the rig. It is retained formally only because no known-good control unit from a different batch was available to close it by direct comparison.

## Open questions

- What is the functional consequence of FLASH_CRYPT_CNT = 1 (with FLASH_CRYPT_CONFIG = 15) for the ROM's boot-time image handling on these specific devices, given that host-side read/write operations are unaffected?
- What are the transfer characteristics of the manufacturer `0x68` (BoyaMicro BY25Q32BS) flash under ESP32 boot-mode read patterns, as distinct from downloader read patterns? Bus capture during a failed boot is the discriminating measurement.
- Is the SPI timing/compatibility behaviour at boot within the flash device's correct operating envelope, and does it degrade across the mode/frequency space in the manner the header signatures suggest?
- How does the observed security configuration interact with boot: does any part of the boot path apply decryption or access rules that the downloader path does not, and would that account for valid stored data being rejected at validation?

Each open question is paired, in the repository README (Future Work), with the experiment that resolves it: a known-good control board through the identical matrix; a logic-analyser capture of the SPI bus during failed boot; flash device substitution on one unit; and a field-by-field eFuse comparison against a virgin device.

# Conclusion

The fault is isolated to the domain bounded by the ROM bootloader, the SPI flash interface, the flash security/configuration state, and application startup. Seven alternative domains are eliminated by direct measurement, and the elimination is replicated across three boards. Within the retained domain, no single component or mechanism is identified as root cause by the current evidence, and none is claimed. The batch is non-viable for firmware execution; the investigation's product is this localisation, the evidence chain supporting it, and the defined experiments that would extend it.

# Confidence Level

- Fault domain isolation: **High.** Convergent evidence from ten preceding records, three-unit replication, and a complete elimination account.
- Batch-level applicability: **High.** Per EV-10.
- Single-component root cause attribution: **Not established.** Deliberately not claimed. Attribution requires bus-level measurement and a control unit, which are recorded as future work.
