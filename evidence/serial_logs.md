# Serial Log Evidence — ROM Boot Output and Download Mode

Evidence ID: EV-01
Related test matrix entries: T2, T3, T4, T5, T14, T15, T16 (see ../TEST_MATRIX.md)
Device under test: ESP32 DevKit V1 (Boards 1, 2, 3 — single purchase batch)
Physical record: [photos/board-front-powered-usb.jpg](photos/board-front-powered-usb.jpg) — board under USB power, power LED illuminated, during serial evaluation.

# Purpose

To establish, from primary serial output, two independent facts:

1. The ESP32 ROM bootloader is operational and the board can be placed into UART download mode by the documented BOOT/EN strapping sequence.
2. Under autonomous boot, the ROM bootloader rejects the application image with an `invalid header` error, and the error value is not constant.

Serial output is the primary evidence source for this investigation. All subsequent analysis depends on the integrity of these records; they are therefore preserved verbatim.

# Command / Procedure

## Procedure A — Download mode verification

1. Connect the board via USB (CP210x interface, COM6).
2. Open the serial monitor at the ROM boot baud rate.
3. Hold BOOT.
4. Press EN.
5. Release EN.
6. Release BOOT.
7. Record ROM output.

## Procedure B — Autonomous boot observation

1. With valid firmware written to flash (see firmware_upload_results.md, EV-04), remove power or press EN to reset the device without holding BOOT.
2. Record the complete ROM output from reset.
3. Repeat under alternate SPI flash configurations (DIO @ 80 MHz, DIO @ 40 MHz, DOUT @ 40 MHz). Configuration-specific results are recorded in flash_mode_experiments.md (EV-09); the primary failure record is preserved here.

# Expected Result

Procedure A: ROM reports boot mode `0x3` (DOWNLOAD_BOOT) and waits for a host download. This is the documented behaviour of a functional ESP32 ROM with correct BOOT (GPIO0) and EN strapping.

Procedure B: ROM reports boot mode `0x13` (SPI_FAST_FLASH_BOOT), reads the image header from SPI flash, validates the magic byte (`0xE9`), and transfers control to the loaded image. The application starts.

# Observed Result

## Procedure A — Download mode (verbatim)

```text
rst:0x10 (RTCWDT_RTC_RESET),boot:0x3 (DOWNLOAD_BOOT(UART0/UART1/SDIO_REI_REO_V2))
waiting for download
```

## Procedure B — Boot failure (verbatim)

```text
invalid header: 0x3ffea272
ets Jul 29 2019 12:21:46
rst:0x10 (RTCWDT_RTC_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
```

## Procedure B — Header values observed under alternate configurations (verbatim)

```text
invalid header: 0xd610e111
invalid header: 0x8e465484
invalid header: 0x5691acba
```

The complete set of invalid-header values recorded across the investigation is: `0x3ffea272`, `0xd610e111`, `0x8e465484`, `0x5691acba`. No value repeats as a constant across configurations, and none corresponds to a valid ESP32 image header.

# Interpretation

Procedure A establishes four facts simultaneously: the ROM bootloader executes its own code correctly; the BOOT button and its GPIO0 strapping path are functional; the EN reset circuit is functional; and the UART download path is operational end to end. Any hypothesis attributing the boot failure to ROM corruption, strapping failure, or UART failure is excluded by this record.

Procedure B establishes the failure precisely: the ROM reaches the flash-read stage of SPI_FAST_FLASH_BOOT and the image validation fails. The ROM is therefore accessing the flash interface at boot; what it retrieves does not validate. The non-constancy of the header value is a load-bearing observation. A fixed data corruption in flash would return the same invalid value on every boot under the same configuration, and EV-08 (flash_readback_validation.md) independently proves the stored image is structurally valid. The variability of the returned header, including its sensitivity to SPI configuration (EV-09), localises the defect to the boot-time transfer, not to the stored data.

# Conclusion

The ROM bootloader, BOOT/EN strapping, and UART download path are functional. Autonomous application boot fails at image header validation, with non-deterministic header values. The failure is confined to the boot-time flash read path.

# Confidence Level

**High.** Both records are direct, verbatim serial output, reproduced across three boards (see cross_board_comparison.md, EV-10). No inference is required to establish the facts in the Conclusion; interpretation of mechanism is deferred to final_fault_domain_analysis.md (EV-11).
