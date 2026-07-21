# 04 · GPIO & Pin Muxing

Time to make the chip *do* something visible. On the S32K, driving a pin
involves two separate hardware modules — a split that confuses newcomers
for about ten minutes and then becomes second nature: the **PORT** module
decides *what a pin is* (its function, pull resistors, interrupt behavior),
and the **GPIO** module decides *what a GPIO-configured pin does* (drive
high, drive low, read). This module covers both, drives the S32K144EVB's
RGB LED, reads its user buttons, and shows the same job in SDK-style and
register-level code side by side.

## The PORT/GPIO split

Each pin on the S32K144 belongs to a port (A–E) and appears in **two**
peripherals:

| Module | Register block | Role |
|--------|---------------|------|
| PORT (PORTA…PORTE) | `PORTx->PCR[n]` | Pin Control Register per pin: **MUX** (which peripheral owns the pin), pull-up/down enable, passive filter, interrupt config |
| GPIO (PTA…PTE) | `PTx->PDDR/PDOR/PSOR/PCOR/PTOR/PDIR` | Direction, output data, set/clear/toggle, input read |

A pin does nothing as GPIO until its PCR **MUX field is set to 1** (ALT1 =
GPIO). Other MUX values hand the pin to peripherals — the same physical pin
might be ADC input (ALT0/analog), GPIO (ALT1), LPUART TX (e.g. ALT2), or
FlexCAN (another ALT), depending on the pin. The mapping lives in the
chip's IO Signal Description table — you look it up, you don't guess.

And remembering module 3: **enable the PORT clock in the PCC first**, or
the first `PCR` write hard-faults.

## Driving the EVB's RGB LED (register-level)

On the S32K144EVB the RGB LED sits on **PTD15 (red), PTD16 (green), PTD0
(blue)**, wired *active-low* — the pin sinks current, so writing **0 turns
the LED on**.

```c
#include "S32K144.h"

#define LED_RED_PIN    15u   /* PTD15, active-low */
#define LED_GREEN_PIN  16u   /* PTD16, active-low */

static void leds_init(void)
{
    /* 1. Clock the PORT D configuration module (PCC!)              */
    PCC->PCCn[PCC_PORTD_INDEX] |= PCC_PCCn_CGC_MASK;

    /* 2. PORT: make the pins GPIO (MUX = 1)                        */
    PORTD->PCR[LED_RED_PIN]   = PORT_PCR_MUX(1);
    PORTD->PCR[LED_GREEN_PIN] = PORT_PCR_MUX(1);

    /* 3. GPIO: outputs, and start OFF (active-low → drive high)    */
    PTD->PSOR = (1u << LED_RED_PIN) | (1u << LED_GREEN_PIN);
    PTD->PDDR |= (1u << LED_RED_PIN) | (1u << LED_GREEN_PIN);
}

static void led_red(bool on)
{
    if (on)  PTD->PCOR = (1u << LED_RED_PIN);   /* clear = sink = ON  */
    else     PTD->PSOR = (1u << LED_RED_PIN);   /* set   = high = OFF */
}
```

The set/clear/toggle registers deserve a pause, because they're a genuinely
good hardware idea:

| Register | Effect of writing bit n = 1 |
|----------|------------------------------|
| `PDOR` | Directly sets the whole output register (read-modify-write needed for one pin) |
| `PSOR` | **Sets** output bit n high — other bits untouched, single write |
| `PCOR` | **Clears** output bit n — atomic, no read-modify-write |
| `PTOR` | **Toggles** output bit n |

Why it matters: `PDOR |= ...` is a read-modify-write — if an interrupt
changes another pin between your read and your write, one of the changes is
lost. `PSOR/PCOR/PTOR` are single atomic stores. In ECU code where
interrupts are always in play, **prefer PSOR/PCOR/PTOR always** — it's a
correctness habit, not a style preference.

A blink, with a crude delay for now (module 8 replaces this properly):

```c
int main(void)
{
    leds_init();
    for (;;) {
        PTD->PTOR = (1u << LED_RED_PIN);          /* toggle */
        for (volatile uint32_t i = 0; i < 500000u; i++) { }  /* ~busy-wait */
    }
}
```

## Reading the user buttons

The EVB's buttons **SW2 (PTC12)** and **SW3 (PTC13)** are wired through the
board's own resistors so that the pin reads **1 while pressed, 0 when idle**
(check your board's schematic — polarity and pulls are a board decision,
not a chip decision; many other boards wire buttons active-low with
pull-ups instead).

```c
static void buttons_init(void)
{
    PCC->PCCn[PCC_PORTC_INDEX] |= PCC_PCCn_CGC_MASK;
    PORTC->PCR[12] = PORT_PCR_MUX(1);        /* GPIO; board provides pull */
    /* PDDR bit stays 0 = input (reset default) */
}

static bool sw2_pressed(void)
{
    return (PTC->PDIR & (1u << 12)) != 0u;   /* EVB: high when pressed */
}
```

If a board gives you no external resistors, enable the internal pull-up in
the PCR instead: `PORT_PCR_MUX(1) | PORT_PCR_PE_MASK | PORT_PCR_PS_MASK`
(PE = pull enable, PS = 1 selects pull-up) — then pressed reads 0 with the
button wired to ground.

Buttons bounce (contacts chatter for a few milliseconds), so real firmware
samples them periodically — e.g. from the 10 ms tick you'll build in module
8 — and accepts a new state only after 2–3 identical samples. In a car this
is bread and butter: every door switch, stalk, and steering-wheel button
lands on some ECU's GPIO input with exactly this treatment, and the
debounced result is often broadcast on CAN for other ECUs to consume.

## SDK-style vs register-level, side by side

The S32K1 SDK expresses pin muxing as a table consumed once at startup,
usually generated by the S32DS pin tool:

```c
/* SDK-style: one array describes every pin, one call applies it */
const pin_settings_config_t g_pin_mux_InitConfigArr[] = {
    { .base = PORTD, .pinPortIdx = 15u,
      .mux = PORT_MUX_AS_GPIO, .direction = GPIO_OUTPUT_DIRECTION,
      .gpioBase = PTD, .initValue = 1u /* off (active-low) */ },
    { .base = PORTC, .pinPortIdx = 12u,
      .mux = PORT_MUX_AS_GPIO, .direction = GPIO_INPUT_DIRECTION,
      .gpioBase = PTC },
};

PINS_DRV_Init(2u, g_pin_mux_InitConfigArr);      /* apply the table   */
PINS_DRV_ClearPins(PTD, (1u << 15));             /* red ON            */
PINS_DRV_SetPins(PTD,   (1u << 15));             /* red OFF           */
uint32_t v = PINS_DRV_ReadPins(PTC);             /* read whole port C */
```

| | Register-level | SDK-style |
|---|---|---|
| Transparency | Total — you see every bit | Structure fields, mapped to bits for you |
| Verbosity for many pins | Grows fast | One table entry per pin, scales nicely |
| Portability S32K1→S32K3 | Register names change | API concept carries over (RTD has an equivalent Port/Dio layer) |
| Production practice | Used in tiny/bootloader code | Config tables are the norm (and AUTOSAR MCAL works exactly this way) |

Both compile to the same few register writes. Learn the registers once so
the SDK is never a mystery; use tables so 60-pin configs stay reviewable.

## Cheat sheet

| Item | Notes |
|------|-------|
| PORT vs GPIO | PORTx->PCR[n] = pin *function/pulls*; PTx = pin *data/direction* |
| Make pin GPIO | `PORTx->PCR[n] = PORT_PCR_MUX(1)` (after PCC clock enable!) |
| Output | `PTx->PDDR |= bit`, then PSOR/PCOR/PTOR to drive |
| PSOR / PCOR / PTOR | Atomic set / clear / toggle — prefer over PDOR read-modify-write |
| Input | PDDR bit = 0 (default); read `PTx->PDIR` |
| Internal pull-up | PCR bits `PE=1, PS=1` |
| EVB RGB LED | PTD15 red, PTD16 green, PTD0 blue — **active-low** |
| EVB buttons | SW2 = PTC12, SW3 = PTC13 — read high when pressed (board wiring) |
| Debounce | Sample every ~10 ms, accept after 2–3 stable samples |

## Exercise

Write (and build, if you have the toolchain from module 2) a program where
the RGB LED cycles red → green → blue on each press of SW2, and SW3 turns
the LED off entirely. Requirements: use only `PSOR/PCOR/PTOR` for output
(no `PDOR` read-modify-write), detect the *press edge* (a held button must
not keep cycling), and structure it as `leds_init()/buttons_init()` plus a
tiny state machine in the loop. No hardware? Write it anyway and desk-check
it: for the input sequence idle→press→hold→release→press, write down every
register write your code performs — this trace is exactly what you'd see
stepping through in the S32DS debugger.
