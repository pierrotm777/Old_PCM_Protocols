# Graupner/JR SPCM 1024 Port to ATmega2560 and Protothreads Integration

## 1. Porting Objective

After validating the SPCM generator on ESP32, the objective was to port
the same protocol to the **ATmega2560**, the target corresponding to the
OpenAVRc environment.

The key requirement was no longer only to generate the correct bits, but
also to reproduce timing accurately enough for a genuine Graupner RF
module to acquire the signal quickly, maintain the link, and recover
immediately after a power interruption.

The second objective was to progressively adapt the generator to the
**Protothreads (PT)** model used in OpenAVRc's `PROTO_PCM.cpp`.

## 2. Why the ATmega2560

Initial experiments were performed around the ATmega328P/Uno. This
target quickly became too limited in SRAM to comfortably hold the double
buffers and intermediate data required by the generator.

The ATmega2560 provides sufficient RAM and, more importantly,
corresponds to the final target of the project.

The selected output is:

`D12 = PB6 = OC1B`

The signal is therefore directly associated with Timer1's hardware
**Output Compare B** output.

## 3. Final Validated Timing Architecture

The decisive improvement was to stop generating edges in software and
let Timer1 hardware handle them.

Timer1 runs with a `/8` prescaler:

-   CPU clock: 16 MHz;
-   Timer1 clock: 2 MHz;
-   resolution: 0.5 µs per timer count.

The validated constants include:

``` cpp
static constexpr uint16_t SYNC_COUNTS  = 830;       // 415 us
static constexpr uint32_t FRAME_COUNTS = 88050UL;   // 44.025 ms
```

OC1B operates in **toggle on compare match** mode. The edge is therefore
produced directly by the hardware peripheral and does not depend on ISR
execution time.

The ISR essentially only schedules the next event:

``` cpp
OCR1B += DurationBuf[a][idx];
```

This architecture proved much more robust than the earlier approaches.

## 4. Double Buffering

Two duration buffers are used:

``` cpp
DurationBuf[2][MAX_DURATIONS]
```

While Timer1 transmits one buffer, the other can be prepared.

`ActiveBuf`, `PendingBuf`, and `PendingReady` synchronize the producer
of the next frame with the ISR.

This separation is fundamental: **Timer1 transmits, the builder
prepares**.

The builder must never interfere directly with the timing of an already
scheduled edge.

## 5. Validation of V11 Non-PT

The V11 FAILSAFE non-PT version became the first fully stable ATmega2560
reference.

It validated:

-   RF module acquisition;
-   frame period;
-   timing stability;
-   C1 to C8;
-   `SWEEP` mode;
-   C9;
-   Fail-Safe;
-   immediate recovery after power is removed and restored.

This version should be preserved as a safety reference for all future
development.

## 6. Moving to Protothreads

The move to PT was deliberately progressive.

The intended principle is the same as in OpenAVRc: split construction of
the next frame into small cooperative operations so that the processor
is not monopolized.

An essential Protothreads rule is that an automatic local variable must
not be assumed to remain valid across a `PT_YIELD()`.

All information required after a Yield must therefore be stored in
persistent context.

For this generator, that state is grouped in `GraPtCtx`.

## 7. PT1 to PT3: Identifying the Problems

The first PT adaptations were useful for identifying the architectural
limitations.

PT1 produced a very clean timing signal and the receiver acquired the
link, but the servos remained fixed.

PT2 attempted to advance the PT builder from the Timer1 ISR. This proved
to be the wrong direction: frame timing became disturbed and the servos
no longer even returned correctly to neutral.

PT3 returned to the safe principle of keeping PT outside the ISR and
Timer1 independent. The servos returned to 1500 µs, but still did not
respond.

The `SHOW` diagnostic then provided the decisive information:

``` text
frame=1974 swaps=0 aborts=0 active=0 pending=1 ready=0
```

Timer1 was therefore continuing to transmit frames, but no newly
constructed PT frame was ever published. The initial 1500 µs buffer was
simply being repeated.

## 8. PT4: First Fully Functional PT Version

PT4 deliberately simplified the Protothread.

The PT performs:

1.  channel snapshot;
2.  logical frame construction;
3.  one Yield;
4.  physical conversion;
5.  one Yield;
6.  a call to the already validated V11 `buildDurations()`;
7.  buffer publication;
8.  waiting for the ISR to consume it.

This version worked immediately: neutral positions, commands, and Sweep
operation were restored.

PT4 is therefore the **stable hybrid PT reference**.

It uses Protothreads to sequence frame preparation, but
`buildDurations()` is still executed in one monolithic operation.

## 9. PT5 FULL-PT

PT5 was developed from PT4 without modifying Timer1, OC1B, or the ISR.

The important difference is that duration construction itself became
cooperative.

The required state is stored in `GraPtCtx`, including:

``` cpp
uint16_t bitIndex;
uint16_t ones;
uint32_t previous;
uint32_t transition;
uint32_t delta;
```

The builder processes one physical transition, saves its state, executes
`PT_YIELD()`, and then resumes with the next transition.

There is therefore no longer a monolithic `buildDurations()` call
preparing the entire frame.

After correcting a simple duplicate declaration of variables in the
context, **PT5 FULL-PT FIX1 was validated on the real hardware**.

The receiver acquires the signal and the servos respond correctly.

## 10. Final PT5 Architecture

The final organization can be summarized as follows:

``` text
loop()
  |
  +-- consoleTask()
  +-- updateSweep()
  |
  +-- GraBuildNextFramePt()
        |
        +-- snapshot C1..C10 + Fail-Safe
        +-- logical frame construction
        +-- PT_YIELD
        +-- physical conversion
        +-- PT_YIELD
        |
        +-- physical transition
        +-- PT_YIELD
        +-- physical transition
        +-- PT_YIELD
        +-- ...
        |
        +-- end of frame
        +-- publish complete buffer

Timer1 / OC1B
  |
  +-- hardware edge generation
  +-- short ISR
  +-- read active buffer
  +-- switch to ready buffer at the frame boundary
```

Critical timing and data construction are therefore clearly separated.

## 11. Fail-Safe

V11 Fail-Safe support was retained throughout the PT conversion.

It remains compile-time configurable:

``` cpp
#define SPCM_FAILSAFE_ENABLED 1
```

This is intentional because OpenAVRc does not yet necessarily manage
Fail-Safe in the way expected by this generator.

With a value of `0`, the relevant fields remain in HOLD/OFF and
Fail-Safe commands can be blocked.

With a value of `1`, C1 to C8 Fail-Safe positions are inserted into the
SPCM service cycle.

## 12. References to Preserve

Three stages are especially important and should not be overwritten
during final integration:

**V11 FAILSAFE non-PT**\
Reference for the protocol and Timer1/OC1B timing.

**V11 FAILSAFE PT4**\
Simple and functional Protothreads reference.

**V11 FAILSAFE PT5 FULL-PT FIX1**\
Current FULL-PT reference validated on real hardware.

The rule for future work is not to modify the validated Timer1/OC1B
engine without a measured reason. OpenAVRc-oriented development should
focus primarily on integrating the PT builder into its existing
infrastructure.

## 13. Porting Result

The port produced an ATmega2560 Graupner/JR SPCM generator that is:

-   compatible with the genuine RF module tested;
-   capable of controlling C1 to C8 and C9;
-   compatible with the studied service cycle and Fail-Safe;
-   stable at approximately 44.025 ms per frame;
-   based on Timer1/OC1B for timing-critical edges;
-   double-buffered;
-   and, in PT5 FULL-PT, fully split into Protothread steps for frame
    construction.

This version provides a directly usable foundation for the next stage:
clean integration of the SPCM generator into OpenAVRc's `PROTO_PCM.cpp`
architecture.
