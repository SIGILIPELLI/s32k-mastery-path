# Real-Time Drivers (RTD) & MCAL Config

Module 1 drew the AUTOSAR layer diagram; this module lives entirely in
its bottom layer. NXP's **Real-Time Drivers (RTD)** package is the
AUTOSAR-compliant MCAL for S32K3, generated from configuration you build
in **S32 Configuration Tools (S32CT)** rather than hand-written against
`FLEXCAN0->MCR`. The shift that matters: on S32K1 bare-metal (Level 1/2)
a driver call and its configuration are the same C statement. On RTD, the
call is generic (`Can_Init`) and everything project-specific — pin
assignments, baud rate, mailbox count, interrupt priority — lives in a
generated `_Cfg.c`/`_Cfg.h` pair you never hand-edit, because the next
regeneration overwrites it.

## The generated-config pattern

```c
/* Port_Cfg.c — generated, do not edit by hand */
const Port_PinConfigType Port_Config_PinConfig[] = {
    { .pinPortIdx = PORTC, .pinNum = 6u,
      .mux = PORT_MUX_ALT2 /* CAN0_TX */, .direction = PORT_PIN_OUT },
    { .pinPortIdx = PORTC, .pinNum = 7u,
      .mux = PORT_MUX_ALT2 /* CAN0_RX */, .direction = PORT_PIN_IN },
};

const Port_ConfigType Port_Config_0 = {
    .pins    = Port_Config_PinConfig,
    .numPins = sizeof(Port_Config_PinConfig) / sizeof(Port_Config_PinConfig[0])
};
```

```c
/* main.c — application code, hand-written, references generated symbols only */
#include "Port.h"
#include "Can.h"
#include "Mcu.h"

int main(void)
{
    Mcu_Init(&Mcu_Config_0);
    Mcu_InitClock(McuClockSettingConfig_0);
    while (Mcu_GetPllStatus() != MCU_PLL_LOCKED) { /* wait for lock */ }

    Port_Init(&Port_Config_0);
    Can_Init(&Can_Config_0);
    Can_SetControllerMode(CanConf_CanController_CanController_0, CAN_T_START);

    for (;;) {
        Can_MainFunction_Write();   /* RTD polling hooks — module 6 covers ISR-driven alt. */
        Can_MainFunction_Read();
    }
}
```

The discipline this enforces: **application code never contains a
magic register value.** If a baud rate is wrong, the fix happens in
S32CT and a regenerate, not a hunt through `.c` files for a stray
`0x00DB0006`. That traceability — every configuration parameter has one
authoritative source — is what an ISO 26262 software safety case
(Level 4 module 2) will demand evidence of.

## MCAL initialization order matters

RTD modules have real dependency ordering; getting it wrong produces
faults that look like silicon bugs:

```text
1. Mcu_Init()          — clock sources configured, NOT yet switched
2. Mcu_InitClock()     — PLL configured and started
3. Mcu_DistributePllClock() / poll GetPllStatus() — wait for lock
4. Port_Init()         — pin muxing (must precede peripheral init)
5. Peripheral _Init()  — Can_Init, Lpuart_Init, Adc_Init, ...
6. Peripheral enable   — Can_SetControllerMode(..., CAN_T_START)
```

Calling `Can_Init()` before `Port_Init()` configures the FlexCAN
peripheral correctly but leaves its TX/RX pins in their GPIO reset state
— the controller runs, transmits nothing onto the physical bus, and the
failure looks exactly like a transceiver or wiring fault. This ordering
bug is common enough that it is the first thing to check when a "new"
RTD-based board doesn't talk on the bus at all.

## Interrupt configuration through RTD

```c
/* Can_Cfg.h (generated) exposes ISR entry points; RTD wires them into
   the NVIC/INTC vector table via the linker's startup_S32K3xx.c */
void CAN0_ORed_0_31_MB_IRQHandler(void)
{
    Can_47_FlexCAN_MainFunction_Read();  /* vendor extended API, module-specific ISR handler */
}
```

RTD generates the ISR *names* to match S32K3's vector table exactly —
`CAN0_ORed_0_31_MB_IRQHandler` is not a name you choose, it is the name
the startup file expects. Renaming it silently breaks nothing at compile
time (an unreferenced function) but the interrupt fires into the default
handler and hard-faults or loops forever, which is a much harder bug to
trace than a linker error would have been.

## Automotive-MCU concerns

- **Regeneration silently discards hand edits.** Never patch a `_Cfg.c`
  file directly to work around a tooling limitation — the next S32CT
  export overwrites it with no diff shown. If a generated default is
  wrong, fix the configuration model, not the output.
- **DET (Development Error Tracing) has a real runtime cost.** RTD MCAL
  modules built with `<Module>_DEV_ERROR_DETECT == STD_ON` add parameter
  range checks to every API call — useful during bring-up, but each check
  costs cycles in an ISR-rate `Can_Write()`. Production builds ship with
  DET off; verify defensive code doesn't silently depend on a DET check
  catching a bug that DET-off builds will not catch.
- **Clock configuration mismatches show up as baud-rate drift, not
  errors.** If `Mcu_InitClock()`'s configured `CAN_CLK` source frequency
  doesn't match what your CAN bit-timing calculation in S32CT assumed,
  every frame you send has a slightly wrong bit rate — invisible on a
  single node, and a source of intermittent arbitration-lost/CRC errors
  once a second node with correct timing joins the bus.
- **MCAL configuration sets are not portable across silicon revisions
  without re-validation.** An S32CT configuration built against an early
  S32K344 engineering sample's errata may configure a workaround (e.g. a
  specific FlexCAN erratum's recommended timing margin) that a later mask
  set no longer needs — check the errata sheet revision your configuration
  targeted before reusing it on new hardware.

## Cheat sheet

| Concept | RTD term | Notes |
|---------|----------|-------|
| Configuration tool | S32 Configuration Tools (S32CT) | Generates `_Cfg.c`/`_Cfg.h`, never hand-edit output |
| Init order | `Mcu` → `Port` → peripheral `_Init` → `_SetMode`/`_Start` | Wrong order = silent failure, not a build error |
| Error reporting | DET (Development Error Tracing) | On for bring-up, off for production; costs cycles |
| Error reporting | DEM (Diagnostic Event Manager) | Production runtime fault logging, feeds `Dcm`/`0x19` DTC reads |
| Vendor API | `Can_47_FlexCAN_*` | NXP-specific extensions layered on standard `Can_*` |
| ISR naming | Fixed by startup file, e.g. `CAN0_ORed_0_31_MB_IRQHandler` | Renaming silently breaks the vector table entry |
| Config data source | ARXML | Single source of truth; regenerate all dependents together |
| Standard MCAL modules | `Mcu`, `Port`, `Dio`, `Can`, `Lpuart`, `Adc`, `Fee`, `Spi` | AUTOSAR standardized API names across vendors |

## Exercise

Convert a Level 1 bare-metal FlexCAN bring-up into RTD-shaped code,
without necessarily running S32CT if it isn't installed. (1) Write out,
by hand, what a generated `Can_ConfigType` structure for your existing
500 kbit/s configuration would contain — clock source, prescaler,
segment lengths, mailbox count — matching the register values you
already calculated in Level 1. (2) Restructure your `main()` to follow
the strict `Mcu → Port → Can` init order above, and add a comment at each
step naming which real hardware fault results if that step is skipped
or reordered. (3) Write a small DET-style parameter check wrapper around
one of your existing driver calls (e.g. reject a mailbox index outside
the configured range) and show, with a comment, where you would compile
it out for a production build. (4) If S32 Design Studio and S32CT are
available, generate a real single-CAN-channel RTD project for an S32K3
target and diff its generated ISR names and init call sequence against
what you predicted by hand.
