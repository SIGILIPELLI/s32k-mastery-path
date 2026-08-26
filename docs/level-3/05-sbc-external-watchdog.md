# Safety Mechanisms — SBC & External Watchdogs

The S32K has an internal watchdog (`WDOG`), and you've likely already
serviced it in Level 1/2. It is not enough on its own for a safety-
relevant ECU, for a simple reason: **a watchdog inside the same silicon
it's supervising cannot detect a fault that takes down the whole chip** —
a brown-out that corrupts flash, a clock failure, a core lockup that also
freezes the watchdog timer's own increment logic. Production automotive
ECUs pair the MCU with a **System Basis Chip (SBC)** — a companion IC
(NXP's UJA116x/FS26 families are common S32K partners) that combines a
CAN/LIN transceiver, voltage regulator, and an **external watchdog**
running on its own independent clock and supply monitoring, so it can
declare the MCU dead and force a reset even when the MCU itself cannot.

## Why two watchdogs

```text
Internal WDOG (in MCU)          External Watchdog (in SBC)
─────────────────────           ──────────────────────────
Detects: task hang, runaway     Detects: MCU clock failure, MCU
loop, missed deadline           brown-out, internal WDOG itself
                                 failing to reset the chip
Own clock domain: shares MCU    Own clock domain: independent
oscillator (usually)            oscillator on the SBC
Failure mode covered: software  Failure mode covered: MCU hardware
bug                              failure the software can't detect
```

An ISO 26262 safety case for anything beyond ASIL A typically requires
this kind of independent monitoring path — a single point of failure in
the MCU's own watchdog logic cannot be allowed to also disable the
mechanism meant to catch it.

## Windowed watchdog service pattern

Most external SBC watchdogs (and the S32K's internal `WDOG` in its
strict mode) are **windowed**: servicing too early is a fault, just like
servicing too late.

```text
|<-- closed window -->|<-- open window -->|
0                    Wmin                Wmax
     service here = FAULT      service here = OK
```

```c
/* External SBC watchdog service over SPI, e.g. NXP FS26-style pattern */
#define WDOG_WINDOW_MIN_MS   8u
#define WDOG_WINDOW_MAX_MS  12u

typedef struct {
    uint32_t last_service_ms;
} sbc_wdog_ctx_t;

Std_ReturnType Sbc_Wdog_Service(sbc_wdog_ctx_t *ctx, uint32_t now_ms)
{
    uint32_t elapsed = now_ms - ctx->last_service_ms;

    if ((elapsed < WDOG_WINDOW_MIN_MS) || (elapsed > WDOG_WINDOW_MAX_MS)) {
        return E_NOT_OK; /* servicing outside the window is itself a fault */
    }

    /* SBC watchdogs frequently require a rotating challenge/response
       token over SPI, not a fixed magic byte — a fixed value would let
       a stuck-at fault on the SPI line "accidentally" service it forever */
    uint8_t token = Sbc_Wdog_NextToken();
    Spi_SyncTransmit(SBC_SPI_CHANNEL, &token, 1u);

    ctx->last_service_ms = now_ms;
    return E_OK;
}
```

The rotating-token detail is not decorative: a fixed "pet the dog" byte
means a shorted or stuck-high SPI MOSI line can accidentally produce the
correct service pattern by coincidence, defeating the very fault the
watchdog exists to catch. A challenge/response scheme (the SBC issues a
seed, the MCU must compute and return a derived value) makes an
accidental service statistically implausible.

## SBC fault outputs and the safe state

```c
/* SBC drives a dedicated fault/error pin the MCU monitors on a GPIO,
   independent of any communication bus that might itself be down */
void Sbc_FaultPin_ISR(void)
{
    if (Gpio_ReadPin(SBC_FAULT_PIN) == GPIO_LOW) {
        /* SBC has declared a fault condition (overtemp, undervoltage,
           watchdog timeout it detected independently) */
        EnterSafeState();  /* module 6 covers what "safe state" means
                               in terms of MPU-partitioned actuator cutoff */
    }
}
```

The SBC's fault pin is deliberately a separate physical signal, not a
CAN or SPI message — because the failure it needs to report can include
"the MCU can no longer talk on any bus," and a fault report that depends
on the failed component's own communication path is not independent.

## Automotive-MCU concerns

- **Window violation handling must not be "just retry."** A windowed
  watchdog rejecting a too-early service is telling you the scheduler
  ran faster than expected — possibly because an interrupt storm is
  starving lower-priority tasks. Logging and investigating a window
  violation, not silently re-servicing, is what catches this class of
  bug before it becomes a field failure.
- **SPI-based SBC communication is itself a dependency the safety
  mechanism relies on.** If the watchdog service token travels over the
  same SPI bus used for other peripherals, a bus contention bug can
  delay the service past the window — verify the SBC watchdog SPI
  transaction has priority, or lives on its own SPI instance, in a
  timing-critical design.
- **Independent clock sources must actually be independent.** Some SBC
  reference designs allow the SBC to derive its watchdog timing from the
  same crystal as the MCU for cost reasons — check the schematic. If they
  share an oscillator, a crystal fault takes down both watchdogs
  simultaneously, defeating the independence the architecture is meant
  to provide.
- **Power-on reset sequencing between SBC and MCU has real ordering
  constraints.** The SBC typically must complete its own power-up
  self-test and release the MCU's reset line only after its voltage
  rails are stable; a design that lets the MCU boot before the SBC has
  validated supply rails can start executing on marginal voltage,
  producing intermittent flash-read corruption that looks like a
  software bug.

## Cheat sheet

| Term | Meaning |
|------|---------|
| SBC | System Basis Chip — transceiver + regulator + watchdog + fault monitoring in one IC |
| Internal WDOG | MCU-integrated watchdog; detects software hangs, shares MCU clock/power domain |
| External watchdog | SBC-hosted; independent clock/supply, detects MCU-level hardware failures |
| Windowed watchdog | Service must land inside [Wmin, Wmax]; too early is a fault, not just too late |
| Challenge/response service | SBC issues a token, MCU must compute/return a derived value — defeats stuck-line false-service |
| Fault pin | Dedicated GPIO from SBC to MCU, independent of any shared communication bus |
| Common NXP SBC families | UJA116x, FS26 (safety SBC family) |
| Relevant standard | ISO 26262-5 (hardware), independent monitoring path requirement for higher ASIL |

## Exercise

Design (and, if you have SBC hardware, implement) a windowed external
watchdog service loop. (1) Define your window bounds and justify them
against your actual task scheduling period — the window must be wide
enough to tolerate normal scheduling jitter but narrow enough to catch a
meaningfully-late or early service. (2) Implement `Sbc_Wdog_Service` with
a simple rotating-token scheme (even a basic LFSR-derived token is
enough to demonstrate the principle) and show, with a deliberately
disabled token check, why a fixed-value service defeats the mechanism.
(3) Wire a GPIO interrupt to simulate an SBC fault pin assertion and
implement `EnterSafeState()` as a stub that at minimum disables all
actuator outputs and logs the fault reason. (4) Write out, as a design
note, what would happen in your system if the SBC and MCU shared a
single crystal oscillator — identify the specific failure mode this
would fail to detect.
