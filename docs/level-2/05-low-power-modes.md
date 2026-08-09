# Low-Power Modes & Wakeup Sources

A parked car's battery is not switched off. Dozens of ECUs stay connected
to the 12 V rail for weeks, and the vehicle must still start after a month
at an airport car park. That gives every module a **quiescent current
budget** — often a few tens of microamps per ECU, negotiated at vehicle
level and audited on a bench with a precision ammeter. Firmware owns most
of that number. This module covers the S32K1's power modes, how to enter
and leave them, every wakeup source that matters, and FlexCAN's Pretended
Networking — the feature that lets a sleeping node listen for one specific
frame without waking the CPU.

## The mode map

The S32K1 exposes its modes through the **SMC** (System Mode Controller),
with the **PMC** (Power Management Controller) handling regulators and
voltage detection. Unlike the Kinetis parts it descends from, the S32K1
has **no LLS or VLLS modes** — the deepest state is VLPS, and it retains
RAM and register state.

| Mode | Core | Peripherals | Typical use |
|------|------|-------------|-------------|
| **HSRUN** | Full speed (up to 112 MHz on S32K14x) | All | Short compute bursts; some peripherals restricted |
| **RUN** | Up to 80 MHz | All | Normal operation |
| **VLPR** | ≤ 4 MHz, SIRC-sourced | Most, reduced | Awake but idle — low-rate polling |
| **WAIT** | Clock gated (WFI) | All running | Cheapest possible sleep; instant wake |
| **VLPW** | Clock gated | Reduced | WAIT entered from VLPR |
| **STOP1 / STOP2** | Stopped | Partial (PSTOPO selects) | Keep some buses alive while the core sleeps |
| **VLPS** | Stopped | Minimal; SIRC/LPO available | Deep sleep — the "parked car" mode |

Two transitions are worth memorizing. **You cannot enter VLPR from a fast
clock** — the core must already be at or below the VLPR limit, so you
switch the clock configuration *first*. And **the mode you are allowed to
enter is gated by `SMC->PMPROT`**, which is write-once after reset: if
firmware does not permit VLPS early in boot, no later code can enter it.

## Entering a mode

The SDK's power manager takes a table of configurations and an index:

```c
#include "power_manager.h"

static const power_manager_user_config_t runCfg = {
    .powerMode       = POWER_MANAGER_RUN,
    .sleepOnExitValue = false,
};

static const power_manager_user_config_t sleepCfg = {
    .powerMode       = POWER_MANAGER_VLPS,
    .sleepOnExitValue = false,   /* true = go straight back to sleep
                                    after the waking ISR returns      */
};

static const power_manager_user_config_t *const pwrCfgs[] = {
    &runCfg, &sleepCfg,
};

#define PWR_MODE_RUN    0u
#define PWR_MODE_SLEEP  1u

void power_init(void)
{
    POWER_SYS_Init(&pwrCfgs[0], 2u, NULL, 0u);
}

void app_enter_sleep(void)
{
    outputs_force_safe();          /* module 9: safe state before sleep */
    can_transceiver_standby(true); /* SBC / transceiver into standby    */

    (void)POWER_SYS_SetMode(PWR_MODE_SLEEP, POWER_MANAGER_POLICY_AGREEMENT);
    /* execution resumes HERE after a wakeup interrupt has been taken */

    can_transceiver_standby(false);
    app_log_wakeup_source();
}
```

`POWER_MANAGER_POLICY_AGREEMENT` lets registered driver callbacks veto the
transition — a driver mid-transfer says "not now." `POWER_MANAGER_POLICY_FORCIBLE`
ignores them. Use agreement unless you have a specific reason not to;
forcibly stopping a peripheral mid-transaction is how you corrupt an
EEPROM write.

!!! warning "Sleep is a state transition, not a function call"
    The line after `POWER_SYS_SetMode` executes *after* the wakeup ISR has
    already run. Anything the ISR needs must be valid before you sleep,
    and anything you re-initialize afterwards (clocks, PLL relock,
    peripherals that lost configuration) must be done before the first
    real work. Write sleep and wake as one reviewed pair.

## Wakeup sources

Any enabled NVIC interrupt can wake the core from STOP and VLPS, but the
peripheral generating it must still have a clock in that mode. The
practical list:

| Source | Works from | Notes |
|--------|-----------|-------|
| **PORT pin interrupt** | STOP, VLPS | Asynchronous edge detect; the digital filter needs `PORTx->DFCR` set to the LPO clock to survive stop |
| **LPTMR** | STOP, VLPS | The standard periodic wake — runs from the 1 kHz LPO tap or SIRC |
| **RTC alarm / seconds** | STOP, VLPS | Long timebases, 32 kHz source |
| **LPUART RX edge** | STOP, VLPS | Wake on start bit; the first byte is usually lost |
| **CMP (analog comparator)** | STOP, VLPS | Threshold crossing without the ADC |
| **FlexCAN self-wake** | STOP, VLPS | `MCR[SLFWAK]`, `CTRL1[WAKMSK]` — wakes on *any* bus activity |
| **FlexCAN Pretended Networking** | STOP, VLPS | Wakes only on a *matching* frame — see below |

A minimal periodic wake with the LPTMR:

```c
#include "lptmr_driver.h"

static const lptmr_config_t lptmrCfg = {
    .workMode       = LPTMR_WORKMODE_TIMER,
    .dmaRequest     = false,
    .interruptEnable = true,
    .freeRun        = false,
    .clockSelect    = LPTMR_CLOCKSOURCE_1KHZ_LPO,  /* alive in VLPS */
    .prescaler      = LPTMR_PRESCALE_2,
    .bypassPrescaler = true,
    .counterUnits   = LPTMR_COUNTER_UNITS_MICROSECONDS,
    .compareValue   = 500000u,                     /* wake every 500 ms */
};

void wake_timer_init(void)
{
    LPTMR_DRV_Init(0u, &lptmrCfg, false);
    INT_SYS_EnableIRQ(LPTMR0_IRQn);
    LPTMR_DRV_StartCounter(0u);
}
```

## Pretended Networking: sleeping through the noise

Plain FlexCAN self-wake has an expensive problem: a vehicle bus is rarely
silent, so *any* activity wakes the node, and a node that wakes ten times
a second is not asleep at all. **Pretended Networking** (available on the
S32K1's FlexCAN0) puts the CAN protocol engine in a low-power state where
it keeps receiving and filters frames in hardware, waking the core only on
a match — by ID, by DLC, by payload content, or after N matches.

```c
/* Wake only on the network-management frame 0x0A0.
   Field names vary slightly across SDK versions — check yours. */
static const flexcan_pn_config_t pnCfg = {
    .wakeUpTimeout    = false,
    .wakeUpMatch      = true,
    .numMatches       = 1u,
    .filterComb       = FLEXCAN_FILTER_ID,
    .idFilterType     = FLEXCAN_FILTER_MATCH_EXACT,
    .idType           = FLEXCAN_MSG_ID_STD,
    .idLower          = 0x0A0u,
};

void can_sleep_with_pn(void)
{
    FLEXCAN_DRV_ConfigPN(INST_CAN0, true, &pnCfg);
    /* ...enter VLPS. On wake, the frames received during PN are in the
       wake-up message buffers: */
    flexcan_msgbuff_t wmb;
    FLEXCAN_DRV_GetWMB(INST_CAN0, 0u, &wmb);
}
```

The registers behind this — `MCR[PNET_EN]`, `CTRL1_PN`, `FLT_ID1`,
`FLT_DLC`, `PL1_LO`/`PL1_HI`, and `WU_MTC` for the match count and wake
status — are worth reading in the reference manual, because the SDK
exposes only a subset of the filter combinations.

## Automotive concerns

- **Budget the current, then verify it.** "We enter VLPS" is not a
  measurement. Put a precision ammeter in series with the ECU supply and
  read the actual sleep current; the difference between 30 µA and 3 mA is
  usually one pin left driving a pull-up, a transceiver never put into
  standby, or a debug LED nobody removed.
- **Pin state in sleep is your responsibility.** Outputs keep driving
  through STOP/VLPS. Inputs left floating oscillate and burn current. Go
  through the pin map (module 4's table) and define a sleep state for
  every pin — this is a standard review artifact.
- **The transceiver usually dominates.** A CAN transceiver in normal mode
  draws far more than the sleeping MCU. Putting it into standby — via its
  STB pin or over SPI to the SBC (module 3) — is typically the single
  biggest saving available.
- **Log why you woke.** Store the wake source and a counter in retained
  RAM or NVM. A node with 400 wakeups per night has a bug that a current
  measurement alone will never localize.
- **Debounce the wake decision.** A single edge on a wake pin can be
  noise. Waking, sampling, confirming, and going back to sleep is normal;
  waking and immediately taking action is how a car's lights turn on in a
  thunderstorm.
- **Safe state before sleep, always.** Outputs to their defined safe
  values *before* the mode change, not after the wake — module 9's rule
  applied to the one code path where forgetting it lasts for hours.
- **Watch what the watchdog does.** The WDOG can be configured to keep
  running in stop modes; if it does, your sleep interval must be shorter
  than its timeout, or your "low-power mode" is a reset loop.

## Cheat sheet

| Item | Notes |
|------|-------|
| Controllers | **SMC** (mode control), **PMC** (regulators, LVD) |
| Mode set | HSRUN · RUN · VLPR · WAIT · VLPW · STOP1/STOP2 · VLPS |
| No LLS/VLLS | S32K1 removed them — VLPS is the deepest mode, state is retained |
| `SMC->PMPROT` | Write-once at boot; permits the modes you may later enter |
| VLPR limit | Core ≤ 4 MHz from SIRC — switch clocks *before* the mode change |
| SDK entry | `POWER_SYS_Init` then `POWER_SYS_SetMode(idx, POWER_MANAGER_POLICY_AGREEMENT)` |
| Resume point | Code after `SetMode` runs *after* the wakeup ISR |
| Periodic wake | LPTMR from the 1 kHz LPO tap (`LPTMR_CLOCKSOURCE_1KHZ_LPO`) — alive in VLPS |
| Pin wake | PORT interrupts are asynchronous; filter needs `PORTx->DFCR` on LPO |
| CAN self-wake | `MCR[SLFWAK]` + `CTRL1[WAKMSK]` — wakes on *any* bus activity |
| Pretended Networking | `MCR[PNET_EN]`, `CTRL1_PN`, `FLT_ID1`, `WU_MTC` — wake on a *matching* frame |
| Biggest saving | Transceiver standby, then pin states, then the MCU mode |

## Exercise

Turn your capstone node into a sleeping ECU with a defensible current
budget. (1) Write the **pin sleep table**: every pin, its sleep state, and
one sentence of justification — this document is the deliverable, the code
is downstream of it. (2) Implement `app_enter_sleep()` / wake handling with
VLPS, an LPTMR wake every 500 ms for a heartbeat check, and a PORT pin
wake on your board's button. (3) Measure sleep current with a meter and
record three numbers: as first written, after transceiver standby, and
after fixing your pin states. If the first and last differ by less than
10×, look again — something is still driving. (4) Replace plain CAN
self-wake with Pretended Networking filtered on a single ID, and prove the
difference by flooding the bus with unrelated traffic: the self-wake
version should wake continuously, the PN version should stay asleep. (5)
Add a retained wake-reason log exported in your status frame after every
wake. No board? Do steps (1) and (5), and write the mode-transition state
machine including what must be re-initialized after VLPS — that list is
the thing people forget.
