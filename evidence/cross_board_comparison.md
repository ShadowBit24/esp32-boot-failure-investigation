# Cross-Board Comparison

Evidence ID: EV-10
Related test matrix entries: T17, and T1–T16 as replicated (see ../TEST_MATRIX.md)
Devices under test: ESP32 DevKit V1, Board 1, Board 2, Board 3 — single purchase batch (Amazon)
Physical record: [photos/board-front-powered-usb.jpg](photos/board-front-powered-usb.jpg) (powered unit under test), [photos/board-front-module-detail.jpg](photos/board-front-module-detail.jpg) (ESP-32 module, USB-UART device U2, BOOT and EN buttons), [photos/board-back-devkitv1.jpg](photos/board-back-devkitv1.jpg) (reverse side, "ESP32 DEVKITV1" silkscreen).

# Purpose

To determine whether the failure characterised in EV-01 through EV-09 is a property of a single defective unit or a property shared by the batch, by applying the same test set to three boards from the same purchase batch under identical conditions, and comparing results capability by capability.

This distinction controls every downstream claim. Single-unit results support statements about one board. Replicated results support statements about the batch, exclude most handling and assembly explanations, and determine the correct disposition (unit repair versus batch rejection).

# Command / Procedure

Each board was subjected to the same procedures referenced by the preceding evidence files:

- Host interface and power checks (EV-01, EV-02 preconditions)
- Download mode entry (EV-01, Procedure A)
- Flash identification (EV-02)
- Firmware upload with verification (EV-04)
- Flash readback with re-validation (EV-08)
- eFuse summary read (EV-03)
- Autonomous boot attempt (EV-01, Procedure B)

Boards were tested under the same host, cabling, tooling, and firmware. Board identity is the only intended variable.

# Expected Result

Two outcomes are possible, and the procedure is designed to distinguish them:

- **Unit-defect pattern:** results diverge between boards; at least one board executes the application. Conclusions would then apply to the failing unit(s) individually.
- **Batch pattern:** results converge; all boards pass every host-side capability and fail application execution identically. Conclusions then apply at batch level.

# Observed Result

| Capability | Board 1 | Board 2 | Board 3 |
|---|---|---|---|
| CP210x USB detection | PASS | PASS | PASS |
| Download mode | PASS | PASS | PASS |
| Flash detection | PASS | PASS | PASS |
| Firmware upload | PASS | PASS | PASS |
| Flash readback | PASS | PASS | PASS |
| eFuse access | PASS | PASS | PASS |
| Application execution | FAIL | FAIL | FAIL |

Summary, as recorded:

```text
PASS: CP210x Detection, Download Mode, Flash Detection,
      Firmware Upload, Flash Readback, eFuse Access
FAIL: Application Execution
```

Three boards from the same manufacturing batch show nearly identical behaviour across the full capability set. The eFuse security configuration is likewise shared across all three devices (EV-03).

# Interpretation

The batch pattern is observed, unambiguously. The probability that three independent units exhibit the same single-capability failure, the same pass profile on six other capabilities, and the same non-default eFuse configuration, through independent random defects, is negligible. The parsimonious explanation is a common property of the batch: shared component sourcing (the same flash device, EV-02), shared provisioning state (EV-03), or shared manufacturing conditions.

This replication also performs a methodological function for the whole package. It exonerates the test setup in practice (the same setup that fails these boards at execution passes them at every host-driven operation and passes known-good hardware generally), and it excludes explanations tied to one board's handling history, soldering, or connector wear.

The limit of this evidence is equally clear: with all samples drawn from one batch, the comparison cannot separate "this batch" from "this board design as sold by this vendor". That separation requires a control unit from a different batch or vendor, which is recorded as the primary future experiment in the repository documentation.

# Conclusion

The failure is a batch-level property, not an isolated unit defect. All conclusions of this investigation are stated at batch level on the strength of this replication. Disposition of the batch as non-viable for firmware execution (see the companion repository, esp32_hardware_test) is supported.

# Confidence Level

**High.** Three-unit replication with a fully convergent result pattern and no divergent observation in any capability.
