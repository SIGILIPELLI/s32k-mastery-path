# 09 · Safety & Robustness Basics

An infotainment bug is annoying; a brake-controller bug can kill. That
asymmetry is why automotive firmware culture is different, and why the S32K
ships hardware you won't find on hobby MCUs: watchdogs designed for
supervision, ECC on memory, CRC engines, clock and voltage monitors. This
module gives you the working vocabulary of **ISO 26262** in one page, then
makes it concrete: watchdog usage done right, CRC for data integrity, ECC,
and the defensive-firmware patterns that appear in the capstone and in
every production code base you'll ever read.

## ISO 26262 in one page

**ISO 26262** is the functional-safety standard for road-vehicle E/E
systems. "Functional safety" means: *the system avoids unreasonable risk
caused by malfunctions* — not "never fails," but "fails in a controlled,
analyzed way."

The core workflow: for each hazard (e.g. "unintended full brake
application at highway speed"), assess three factors — **Severity**,
**Exposure** (how often the driving situation occurs), and
**Controllability** (can a driver catch it?) — and derive an **ASIL**:

| Level | Meaning | Example systems |
|-------|---------|-----------------|
| QM | Quality-managed, no special safety measures | Infotainment, seat memory |
| ASIL A | Lowest safety integrity | Rear lights |
| ASIL B | Moderate | Instrument cluster, many body functions |
| ASIL C | High | Adaptive cruise elements |
| ASIL D | Highest — most rigorous processes and mechanisms | Braking, steering, airbags |

Higher ASIL ⇒ stricter everything: requirements tracing, MC/DC test
coverage, hardware diagnostic coverage, independence of reviews. Two ideas
you'll meet constantly:

- **Safe state.** Every safety concept defines where to go on failure —
  often "outputs off." A wiper ECU's safe state is *wipers parked*, not
  *wipers at maximum*. Firmware's job is to *detect* faults and *reach* the
  safe state, every time. (For steering there is no "off" — that's why
  higher redundancy exists — but for Level 1 designs, outputs-off thinking
  is the right habit.)
- **Decomposition & mechanisms.** ASIL requirements are met by layering
  mechanisms: lockstep cores (S32K3), ECC, watchdogs, plausibility checks,
  monitoring. The S32K1 targets up to ASIL B systems; S32K3 up to ASIL D.

## Watchdogs: supervision, not ritual

A **watchdog** resets the MCU unless firmware "feeds" it periodically. From
module 3 you know the S32K1 **WDOG** is *already running at reset* (~1 ms
via the 128 kHz LPO). Production firmware re-enables it deliberately:

```c
/* Register-level: 100 ms timeout from the 128 kHz LPO clock */
void wdog_init(void)
{
    WDOG->CNT   = 0xD928C520u;                 /* unlock                  */
    WDOG->TOVAL = 12800u;                      /* 100 ms × 128 kHz        */
    WDOG->CS    = WDOG_CS_EN_MASK              /* enable                  */
                | WDOG_CS_CLK(1)               /* LPO clock               */
                | WDOG_CS_UPDATE_MASK;         /* allow later reconfig    */
}

void wdog_feed(void)
{
    WDOG->CNT = 0xB480A602u;                   /* refresh key             */
}
```

The pattern that separates real supervision from ritual: **never feed the
dog from a timer interrupt.** A timer ISR happily keeps firing while your
main loop is wedged in a corrupted linked list — the watchdog then
guarantees nothing. Feed it from the main loop, *conditionally*:

```c
for (;;) {
    if (g_tick_10ms != last) {
        last = g_tick_10ms;
        bool ok = true;
        ok &= sensor_task_10ms();       /* each task returns health */
        ok &= can_task_10ms();
        ok &= outputs_task_10ms();
        if (ok) { wdog_feed(); }        /* all tasks alive & sane → feed */
    }
}
```

Also standard: check the **reset cause register** (S32K1: the RCM module's
status registers) at boot — firmware that knows "I'm starting because the
watchdog bit me" can log it, count it, and refuse to re-enter the state
that caused it. The WDOG also supports **window mode** (feeding *too early*
also resets — catches runaway loops that spin through the feed), and
Level 3 adds *external* watchdogs with challenge-response for when the MCU
itself is suspect.

## CRC: data integrity

RAM and buses flip bits; flash wears; CAN frames arrive from other —
possibly buggy — ECUs. A **CRC** (cyclic redundancy check) is the standard
detection tool, and the S32K has a hardware CRC unit so it's nearly free.
Standard uses:

- **Flash image check:** the build pipeline appends a CRC32 over the
  application image; the bootloader (Level 2) verifies before jumping.
  A half-flashed ECU must never run.
- **NVM/EEPROM records:** calibration data stored with a CRC; corrupt ⇒
  fall back to safe defaults, set a fault code.
- **End-to-end (E2E) protection:** safety-relevant CAN signals carry an
  in-payload CRC plus the alive counter from module 7 — protecting against
  *software* bugs (stale buffers, frozen tasks) that CAN's own wire-CRC
  can't see. That's the AUTOSAR E2E concept, and you'll implement a small
  version in the capstone.

```c
/* SDK-style shape: hardware CRC unit */
crc_user_config_t crcCfg = { /* CRC-32, standard polynomial, reflection */ };
CRC_DRV_Init(0u, &crcCfg);
uint32_t crc = CRC_DRV_GetCrcResult(0u /* after feeding data words */);
```

## ECC and other silicon safety nets

The S32K's flash and RAM are protected by **ECC** (error-correcting codes):
extra bits stored alongside data let hardware **correct single-bit errors**
and **detect double-bit errors** on every read. Single-bit events are
silently fixed (optionally reported); double-bit errors raise a bus fault —
by design: *crashing loudly beats computing quietly with garbage*, because
the watchdog + reset path then restores a known-good state. The same
philosophy powers the chip's **clock monitors** (loss-of-clock detection),
**low-voltage detect** (defined reset instead of brownout chaos), and on
S32K3, **lockstep cores** — two cores execute every instruction and
hardware compares results continuously.

The pattern behind all of it: **detect → contain → reach safe state →
report.** Memorize that chain; every mechanism in this module is one link
of it.

## Defensive firmware patterns

Habits that mark automotive-grade C, all reappearing in the capstone:

- **Verify your init.** Read back critical configuration; if the PLL never
  locks or the CAN controller won't synchronize, act (retry, degrade,
  safe state) — don't sail on.
- **Plausibility-check inputs** (module 6): range, rate-of-change, and
  cross-channel agreement; debounce faults before reacting; substitute safe
  values; record a **DTC** (diagnostic trouble code — Level 2's UDS reads
  them out).
- **Timeout every external dependency** (module 7): a consumed CAN signal
  that stops arriving must flip to "invalid" after a defined time, with
  defined behavior.
- **Default cases that mean something.** Every `switch` on a state variable
  has a `default:` that treats the impossible as a fault — because ECC just
  taught you memory *can* lie.
- **Initialize outputs to safe, and on any fault, return them to safe.**
  First lines of `main()` after clock init: outputs off, *then* bring up
  the world.
- **MISRA C** (Level 4): the industry's C subset — no dynamic allocation
  after init, checked conversions, no undefined behavior. Start the habit
  now: compile with `-Wall -Wextra` and treat warnings as errors.

## Cheat sheet

| Item | Notes |
|------|-------|
| ISO 26262 | Functional-safety standard; hazard → S/E/C assessment → ASIL |
| ASIL | A (lowest) … D (highest); QM = no safety requirement |
| Safe state | Defined degraded/off condition every fault path must reach |
| S32K positioning | K1: up to ASIL B systems · K3: up to ASIL D (lockstep) |
| WDOG | Feed from **main loop only**, conditionally on task health |
| WDOG keys | Unlock `0xD928C520`, refresh `0xB480A602`; window mode catches early feeds |
| Reset cause | Check RCM status at boot — know *why* you're restarting |
| CRC uses | Flash image, NVM records, E2E protection of CAN signals |
| ECC | HW corrects 1-bit, detects 2-bit errors; double-bit ⇒ fault, not garbage |
| Detect chain | detect → contain → safe state → report |
| E2E on CAN | Payload CRC + alive counter defeats stale/frozen-sender bugs |

## Exercise

Do a miniature safety analysis of module 8's radiator-fan example (coolant
sensor → PWM fan): (1) list five distinct faults across sensor, harness,
MCU, and fan (e.g. NTC open circuit, task overrun, fan stalled); (2) for
each, name the *detection mechanism* from this module and the *reaction*,
and define the system's safe state — justify whether it's fan-off or
fan-full-on (hint: what does an engine prefer when uncertain?); (3) then
write `outputs_task_10ms()` in C: it must apply the commanded duty only
when sensor data is valid and fresh, otherwise drive the safe state and
latch a fault flag — and return `bool` health suitable for the conditional
watchdog feed shown above.
