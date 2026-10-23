# Troubleshooting Flowchart — "Flashes Fine, Won't Boot" (ESP32 class)

A reusable diagnostic flow, generalised from this investigation. Each step names the observation that lets you proceed. The discipline: never advance on an assumption; every arrow is an observed result.

```mermaid
flowchart TD
    S[Board flashes and verifies,<br/>but application does not run] --> Q1{ROM output<br/>on serial?}
    Q1 -->|No| P1[Power / USB / UART path:<br/>LED, enumeration, cables, drivers]
    Q1 -->|Yes| Q2{Download mode<br/>reachable?<br/>BOOT + EN sequence}
    Q2 -->|No| P2[Strapping fault:<br/>BOOT/EN circuit, GPIO0 state]
    Q2 -->|Yes| Q3{flash-id<br/>sane?}
    Q3 -->|No| P3[Flash device / SPI wiring:<br/>dead flash, solder, wrong CS]
    Q3 -->|Yes| Q4{Readback image<br/>valid?<br/>image-info on a dump}
    Q4 -->|No| P4[Write-path fault:<br/>re-flash, verify, check image build]
    Q4 -->|Yes| Q5{eFuse / security<br/>state normal?<br/>espefuse summary}
    Q5 -->|Abnormal| P5[Configuration domain:<br/>encryption counter, RD_DIS,<br/>erase behaviour. Record state;<br/>test erase standard vs forced]
    Q5 -->|Normal| Q6{SPI mode / clock<br/>sweep changes<br/>the symptom?}
    P5 --> Q6
    Q6 -->|Symptom changes<br/>with settings| P6[Interface domain:<br/>boot-time read marginal.<br/>Next: bus capture, control board]
    Q6 -->|Symptom constant| P7[Constant wrong data:<br/>re-examine Q4 closure,<br/>address/partition config]
    P6 --> R{Replicates on<br/>other units?}
    P7 --> R
    R -->|Yes| C1[Batch-level property:<br/>fault domain report,<br/>quarantine batch]
    R -->|No| C2[Unit-level defect:<br/>component isolation on<br/>the single unit]
```

## Notes on using this flow

- **Order is deliberate.** The first four questions cost minutes and close the majority of real-world cases. Security-state and interface work (Q5, Q6) is expensive in time and interpretation; it is wasted effort if Q1–Q4 are not closed first.
- **Record exact error values at every step.** In this investigation, the *values* of the invalid headers, and how they changed at Q6, were the evidence that separated the interface domain from the content domain. "It still fails" is not an observation; "fails with 0x8e465484 at DIO 40 MHz" is.
- **Erase behaviour is a free security probe.** A standard erase that refuses, where a forced erase succeeds, tells you security features are active before you ever read an eFuse (Q5 corroboration).
- **Replication is the last gate, not the first.** Establish the fault on one unit, then replicate. Conclusions drawn before replication are unit anecdotes regardless of how thorough the single-unit work was.
- **Stopping rule.** When the next discriminating step needs equipment or a control unit you do not have, the correct output is a fault-domain statement plus the defined next experiment — not a best guess dressed as a root cause.
