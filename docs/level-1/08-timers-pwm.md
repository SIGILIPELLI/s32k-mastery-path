# 08 · Timers & PWM

Real ECU firmware doesn't `delay()`. It runs on a heartbeat: read sensors
every 10 ms, transmit CAN every 100 ms, blink an indicator at 1.5 Hz — all
concurrently, none blocking the others. Hardware timers provide that
heartbeat, and PWM — the other face of a timer — is how firmware speaks
"analog" to the power world: LED brightness, fan speed, motor drive. On the
S32K1xx the everyday cast is **LPIT** for periodic interrupts and **FTM**
for PWM and input capture. This module covers all three roles.

## LPIT: the periodic heartbeat

The **LPIT** (Low-Power Periodic Interrupt Timer) has four channels that
count down from a load value and fire an interrupt at zero, forever. One
channel at 10 ms is the classic ECU tick (and exactly what the capstone
uses).

```c
#include "S32K144.h"

volatile uint32_t g_tick_10ms = 0;      /* shared with ISR → volatile */

void lpit_init(void)                     /* 10 ms tick from 48 MHz FIRCDIV2 */
{
    PCC->PCCn[PCC_LPIT_INDEX] = PCC_PCCn_PCS(3)     /* FIRCDIV2 = 48 MHz */
                              | PCC_PCCn_CGC_MASK;

    LPIT0->MCR = LPIT_MCR_M_CEN_MASK;               /* enable module clock */

    /* Load value: 10 ms × 48 MHz = 480 000 ticks (minus 1) */
    LPIT0->TMR[0].TVAL = 480000u - 1u;
    LPIT0->MIER |= LPIT_MIER_TIE0_MASK;             /* channel 0 interrupt  */
    LPIT0->TMR[0].TCTRL = LPIT_TMR_TCTRL_T_EN_MASK; /* start counting       */

    S32_NVIC->ISER[LPIT0_Ch0_IRQn / 32u] =          /* enable IRQ in NVIC   */
        (1u << (LPIT0_Ch0_IRQn % 32u));
}

void LPIT0_Ch0_IRQHandler(void)
{
    LPIT0->MSR = LPIT_MSR_TIF0_MASK;    /* write-1-to-clear the flag — always! */
    g_tick_10ms++;                      /* ISR does the minimum and returns    */
}
```

The main loop then schedules everything off that one tick — the standard
bare-metal ECU pattern:

```c
int main(void)
{
    /* ...clock, pin, uart, adc, can, lpit init... */
    uint32_t last = 0;
    uint8_t  div100ms = 0;

    for (;;) {
        if (g_tick_10ms != last) {          /* a 10 ms boundary passed */
            last = g_tick_10ms;
            sensor_task_10ms();             /* module 6                */
            if (++div100ms >= 10u) {        /* every 10th tick = 100 ms */
                div100ms = 0;
                can_tx_task_100ms();        /* module 7                */
            }
        }
        console_poll();                     /* module 5, every pass    */
    }
}
```

Three ISR rules, learned now, valid forever: **clear the flag** (or the
interrupt refires endlessly), **keep it short** (set a flag/counter, do
real work in the loop), and **`volatile`** for anything shared between ISR
and main loop. This "timer tick + cooperative tasks" structure *is* a tiny
scheduler — an RTOS (Level 2) and AUTOSAR OS (Level 4) are industrial-grade
versions of exactly this idea.

## FTM: PWM output

**PWM** (pulse-width modulation) switches a pin between 0 and 1 at fixed
frequency, controlling the **duty cycle** — the fraction of time high.
Switch faster than the load can respond and the load sees the *average*:
50% duty ≈ half brightness, half fan speed. Power stays efficient because
the transistor is always fully on or fully off.

The **FTM** (FlexTimer Module — the S32K144 has four, FTM0–FTM3) is a
16-bit counter with 8 channels; in edge-aligned PWM mode the counter runs
0→MOD repeatedly and each channel's pin goes high at 0 and low when the
counter passes the channel's **CnV** value:

```text
PWM frequency = FTM clock / (prescaler × (MOD + 1))
duty          = CnV / (MOD + 1)
```

Example: 8 MHz SOSCDIV1 clock, prescaler 1, MOD = 15999 → 500 Hz. CnV =
4000 → 25% duty. Register-level, on the EVB's blue LED (PTD0 = FTM0_CH2,
ALT2 — and remember the LED is active-low, so duty is inverted visually):

```c
void ftm0_pwm_init(void)
{
    PCC->PCCn[PCC_FTM0_INDEX] = PCC_PCCn_PCS(1)      /* SOSCDIV1 = 8 MHz */
                              | PCC_PCCn_CGC_MASK;

    PORTD->PCR[0] = PORT_PCR_MUX(2);                 /* PTD0 → FTM0_CH2  */

    FTM0->MOD = 16000u - 1u;                         /* 500 Hz            */
    FTM0->CONTROLS[2].CnSC = FTM_CnSC_MSB_MASK       /* edge-aligned PWM, */
                           | FTM_CnSC_ELSB_MASK;     /* high-true pulses  */
    FTM0->CONTROLS[2].CnV = 4000u;                   /* 25% duty          */
    FTM0->SC = FTM_SC_CLKS(3)                        /* external clk = PCC choice */
             | FTM_SC_PS(0)                          /* prescale /1       */
             | FTM_SC_PWMEN2_MASK;                   /* enable ch2 output */
}

void pwm_set_permille(uint16_t pm)                   /* 0..1000 */
{
    FTM0->CONTROLS[2].CnV = ((uint32_t)(FTM0->MOD + 1u) * pm) / 1000u;
}
```

(SDK-style: `FTM_DRV_InitPwm(...)` with a `ftm_pwm_param_t` describing
frequency and per-channel duty, then `FTM_DRV_UpdatePwmChannel(...)` — same
registers underneath.)

Choosing PWM frequency is a real engineering decision:

| Load | Typical PWM frequency | Why |
|------|----------------------|-----|
| LED dimming | 200 Hz – 1 kHz | Above flicker perception; nothing else matters |
| DC motor / fan | 16–25 kHz | Above human hearing (audible whine below ~16 kHz) |
| Heaters | 1–10 Hz (!) | Thermal mass is slow; slow switching cuts EMI |
| PWM-encoded sensor lines | Per spec, e.g. 100–500 Hz | Duty *is* the data (below) |

## Input capture: measuring pulses

The FTM's third trick: a channel can **capture** the free-running counter
value on a pin edge. Capture two consecutive rising edges → period; rising
then falling → high time; divide → duty cycle. This is how you *read*
PWM-encoded signals, which automotive uses a lot because duty cycle
survives noisy wiring far better than an analog voltage: many hall-effect
speed sensors, position senders, and fan-feedback lines encode their value
as duty (e.g. 10%–90% ⇔ 0–100 units, with duty near 0% or 100% signalling
a broken wire — the same "faults land outside the valid range" idea as
module 6).

```c
/* Concept (register-level flow):
   CnSC = ELSA|ELSB edge selection, capture mode
   on each capture interrupt: delta = CnV - prev; prev = CnV;
   handle 16-bit wraparound with unsigned subtraction — it just works. */
```

## Automotive mini-gallery

Where today's three tools show up in a real car: **wiper intermittent
control** — LPIT-style tick schedules wipe cycles; the stalk position
arrives as a CAN signal. **Radiator fan** — 25 kHz FTM PWM sets speed from
coolant temperature (module 6's sensor!); input capture on the fan's tach
line verifies it actually spins (plausibility again). **Dashboard
backlight** — PWM dimming, duty from an ambient light sensor.

## Cheat sheet

| Item | Notes |
|------|-------|
| LPIT | 4-channel countdown timer → periodic interrupts; the ECU heartbeat |
| Tick math | `TVAL = period × func_clk − 1` (10 ms @ 48 MHz = 479 999) |
| ISR rules | Clear the flag (write-1-to-clear MSR), stay short, `volatile` shared data |
| NVIC | Peripheral interrupt must also be enabled in the ARM core's NVIC |
| FTM PWM | freq = clk/(PS×(MOD+1)); duty = CnV/(MOD+1); edge-aligned = MSB|ELSB |
| Update duty | Write `CONTROLS[n].CnV` |
| PWM frequency choice | LEDs ~500 Hz; motors/fans 16–25 kHz (audibility); heaters ~Hz |
| Input capture | Timestamp edges with the FTM counter → period & duty measurement |
| Unsigned wrap | `uint16_t delta = now − prev;` handles counter wraparound for free |

## Exercise

Combine everything so far: using the 10 ms LPIT tick, make the blue LED
(FTM PWM) "breathe" — duty ramps 0→100% and back over 2 s — while the
potentiometer (module 6) sets the maximum brightness of the ramp, and the
UART console (module 5) prints the current duty once per second. Constraint:
`main()`'s loop may contain **no** busy-wait delays — every rate must derive
from the tick counter. Paper version if hardware-free: write the code, then
produce a timeline table for the first 50 ms (tick number, tasks that ran,
CnV value written) and verify the breathing math hits exactly 100% at
t = 1.0 s.
