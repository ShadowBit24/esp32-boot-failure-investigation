# Architecture — Boot Chain and Diagnostic Setup

## 1. ESP32 boot chain (as relevant to this fault)

```mermaid
flowchart LR
    A[Power / EN reset] --> B[ESP32 ROM bootloader<br/>mask ROM, always executes]
    B --> C{SPI flash read:<br/>boot image header}
    C -->|Header valid<br/>magic 0xE9| D[Load boot image]
    D --> E[Second-stage bootloader]
    E --> F{SPI flash read:<br/>application image header}
    F -->|Header valid| G[Application entry]
    C -->|Header invalid<br/>THIS STUDY| H[invalid header: 0x........<br/>reset loop]
    F -->|Header invalid| H
```

Facts the investigation relies on:

- The ROM bootloader is mask ROM: it cannot be corrupted by flashing, and Entry 03 of the investigation log proves it executes (download mode works).
- A valid image header starts with magic byte `0xE9`. Observed boot values (`0x3ffea272` etc.) are not valid headers and are not stable.
- Flash regions relevant here: bootloader image at `0x1000` (erase verified at this address), application image at `0x10000`.

## 2. The two flash-read paths

```mermaid
flowchart TD
    subgraph HOSTPATH [Path 1 — host-driven, ROM download mode: ALL TESTS PASS]
        H1[Host PC] -->|USB| H2[CP210x USB-UART]
        H2 -->|UART| H3[ROM downloader]
        H3 -->|SPI, downloader parameters| H4[(SPI flash<br/>Mfr 0x68, 4 MB)]
        H4 -->|readback| H3
    end
    subgraph BOOTPATH [Path 2 — autonomous boot: FAILS]
        B1[ROM bootloader] -->|SPI, boot parameters| H4
        H4 -->|header bytes| B1
        B1 -->|invalid header| B2[Reset loop]
    end
```

The investigation's central structural fact: Path 1 and Path 2 traverse the same physical flash and the same SPI pins, under different drivers and parameters. Path 1 passes completely (write, verify, erase-verify, readback + image validation). Path 2 fails with parameter-dependent garbage. The fault therefore lives in what differs between the paths: boot-time interface behaviour and/or the security/configuration state that conditions boot.

## 3. Diagnostic setup

```mermaid
flowchart LR
    PC[Host PC<br/>esptool / espefuse / serial monitor] -->|USB| BR[CP210x bridge<br/>on DevKit board]
    BR -->|UART TX/RX| SOC[ESP32 SoC<br/>ROM + eFuse block]
    SOC -->|Quad/Dual SPI<br/>GPIO6–GPIO11| FLASH[(SPI flash<br/>0x68 / 0x4016 / 4 MB)]
    SOC -.->|strapping: BOOT, EN| OP[Operator<br/>download-mode sequence]
```

Components and their verified state in this study:

| Component | Role | State |
|---|---|---|
| Host PC + esptool/espefuse | Drives Path 1, captures evidence | Functional |
| CP210x bridge | USB↔UART | Enumerates, communicates (T1, T3) |
| ESP32 ROM | Executes download mode and boot attempt | Executes (T4, T5) |
| eFuse block | Security/configuration state | Accessible; abnormal values (T9) |
| SPI flash | Stores boot + application images | Host-side operations all pass (T6–T8, T11–T13) |
| SPI interface at boot | Carries the header read in Path 2 | **Suspect — inside the remaining fault domain** |
