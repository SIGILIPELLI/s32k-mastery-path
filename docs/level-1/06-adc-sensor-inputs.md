# 06 · ADC & Sensor Inputs

Cars are full of analog: coolant temperature, throttle position,
battery voltage, pedal position, current through a motor. The **ADC**
(analog-to-digital converter) is how firmware sees any of it. The S32K144
has two 12-bit successive-approximation ADCs (ADC0, ADC1) with up to a
million samples per second — far more than a body ECU ever needs. This
module covers single conversions, reading the EVB's potentiometer, scaling
raw counts into engineering units, and the automotive habits (plausibility
limits, filtering) that separate ECU code from demo code.

## From volts to counts

A 12-bit ADC maps 0 V…VREFH (5 V on the EVB, since the board runs the MCU
at 5 V) onto integer **counts** 0…4095:

```text
counts = round(Vin / Vref × 4095)         resolution = Vref / 4096
```

At Vref = 5 V that's ~1.22 mV per count. Two truths to internalize early:

- The ADC measures a **ratio** to Vref, not absolute volts. If Vref sags 2%,
  every "measurement" sags 2%. Precision designs use a dedicated reference;
  ratiometric sensors (like potentiometers powered from the same rail)
  cleverly cancel this error out.
- 12 bits of *resolution* is not 12 bits of *accuracy* — noise, source
  impedance, and layout eat real bits. Treat the last count or two as noise
  unless proven otherwise, and filter (below).

## A single conversion (register-level)

The EVB's potentiometer is wired to **PTC14 = ADC0 channel SE12**. Analog is
the *default* pin function (ALT0), so unusually, no mux write is needed —
but the ADC clock and configuration are:

```c
#include "S32K144.h"

void adc0_init(void)
{
    /* Clock: functional clock from FIRCDIV2 (48 MHz), then gate on */
    PCC->PCCn[PCC_ADC0_INDEX] = PCC_PCCn_PCS(3) | PCC_PCCn_CGC_MASK;

    /* 12-bit mode, input clock divided to stay within ADC spec */
    ADC0->CFG1 = ADC_CFG1_ADIV(2)       /* clock / 4 */
               | ADC_CFG1_MODE(1);      /* MODE 1 = 12-bit */
    ADC0->CFG2 = ADC_CFG2_SMPLTS(12);   /* sample time in ADC clocks */
    ADC0->SC2  = 0u;                    /* software trigger */
}

uint16_t adc0_read(uint8_t channel)     /* pot = channel 12 (SE12) */
{
    ADC0->SC1[0] = ADC_SC1_ADCH(channel);            /* write starts it  */
    while ((ADC0->SC1[0] & ADC_SC1_COCO_MASK) == 0u) /* conversion done? */
        { }
    return (uint16_t)ADC0->R[0];                     /* read clears COCO */
}
```

The core rhythm: **write the channel into SC1[0] (that starts a
conversion) → poll COCO (COnversion COmplete) → read the result register**.
A 12-bit conversion completes in a few microseconds — polling is perfectly
fine at ECU sample rates (100 Hz–1 kHz). Hardware triggering from a timer
and DMA transfer of results exist for high-rate work (Level 2).

Before trusting real measurements, run the ADC's built-in **calibration**
once after clock setup — the SDK wraps this as
`ADC_DRV_AutoCalibration(0u)`; at register level it's a documented sequence
using the CLPx/CLPS calibration registers. Skipping calibration costs you
real accuracy (offset/gain error), and it's free.

!!! note "SDK-style equivalent"
    ```c
    adc_converter_config_t cfg;
    ADC_DRV_InitConverterStruct(&cfg);       /* sane defaults      */
    cfg.resolution = ADC_RESOLUTION_12BIT;
    ADC_DRV_ConfigConverter(0u, &cfg);
    ADC_DRV_AutoCalibration(0u);

    adc_chan_config_t ch = { .channel = 12u, .interruptEnable = false };
    ADC_DRV_ConfigChan(0u, 0u, &ch);         /* starts a conversion */
    ADC_DRV_WaitConvDone(0u);
    uint16_t raw = 0; ADC_DRV_GetChanResult(0u, 0u, &raw);
    ```

## Scaling to engineering units

Raw counts are useless on a CAN bus or in control logic — convert to real
units immediately, in one well-named function, using integer math (floats
are available on the M4F, but integer habits keep ISRs fast and port to
FPU-less parts):

```c
/* Potentiometer position in tenths of a percent (0..1000) */
uint16_t pot_permille(uint16_t raw)
{
    return (uint16_t)(((uint32_t)raw * 1000u) / 4095u);
}

/* Battery voltage in millivolts, measured via an external
   10k:2k2 divider (factor 12200/2200), Vref = 5000 mV        */
uint16_t vbat_mV(uint16_t raw)
{
    uint32_t pin_mV = ((uint32_t)raw * 5000u) / 4095u;
    return (uint16_t)((pin_mV * 12200u) / 2200u);
}
```

Two classic automotive sensor shapes you'll meet constantly:

- **Ratiometric position sensors** (throttle, pedal): output is a fraction
  of their supply. Safety-relevant ones are *dual-channel* — two independent
  tracks, one often inverted, and firmware cross-checks them (sum ≈
  constant). Disagreement ⇒ sensor fault, limp-home mode.
- **NTC thermistors** (coolant, air, battery temperature): resistance falls
  with temperature, read through a divider. The volts→°C curve is nonlinear,
  so production code uses a **lookup table with interpolation**, not a
  formula:

```c
/* 11-entry table: raw ADC (with 10k pullup to 5V) -> temperature in 0.1 °C */
typedef struct { uint16_t raw; int16_t temp_dC; } ntc_point_t;
static const ntc_point_t ntc_tbl[11] = {
    {3900, -400}, {3600, -250}, {3200, -100}, {2750,   0}, {2300, 100},
    {1900,  250}, {1500,  400}, {1150,  550}, { 850, 700}, { 600, 850},
    { 420, 1000},
};   /* values illustrative — derive from your thermistor's datasheet */

int16_t ntc_temp_dC(uint16_t raw)   /* linear interpolation between points */
{
    if (raw >= ntc_tbl[0].raw)  return ntc_tbl[0].temp_dC;
    if (raw <= ntc_tbl[10].raw) return ntc_tbl[10].temp_dC;
    for (uint8_t i = 1; i < 11u; i++) {
        if (raw >= ntc_tbl[i].raw) {
            const ntc_point_t *a = &ntc_tbl[i-1], *b = &ntc_tbl[i];
            return (int16_t)(a->temp_dC +
                (int32_t)(b->temp_dC - a->temp_dC) *
                (a->raw - raw) / (a->raw - b->raw));
        }
    }
    return ntc_tbl[10].temp_dC;   /* unreachable */
}
```

## Filtering and plausibility — the ECU habits

Single ADC readings jitter. The standard cheap fix is an **exponential
moving average** in integer math:

```c
static uint16_t filt = 0;
uint16_t adc_filtered(uint16_t raw)      /* alpha = 1/8 */
{
    filt = (uint16_t)(filt + ((raw - filt) >> 3));
    return filt;
}
```

And *before* filtering, check **plausibility** — automotive firmware never
trusts a sensor blindly:

```c
#define RAW_MIN_PLAUSIBLE  50u     /* ~0 V ⇒ wiring short to ground   */
#define RAW_MAX_PLAUSIBLE  4045u   /* ~5 V ⇒ open circuit / short to V+ */

if ((raw < RAW_MIN_PLAUSIBLE) || (raw > RAW_MAX_PLAUSIBLE)) {
    sensor_fault_count++;
    if (sensor_fault_count > 5u) { enter_failsafe(); }   /* substitute value, set DTC */
} else {
    sensor_fault_count = 0u;
    temp_dC = ntc_temp_dC(adc_filtered(raw));
}
```

Rail-level readings almost always mean a broken harness, not a real
temperature of −40 °C — designing sensor circuits so faults land *outside*
the valid range is deliberate. Module 9 builds this into a fuller
defensive-firmware picture, and the capstone uses this exact pattern.

## Cheat sheet

| Item | Notes |
|------|-------|
| Resolution | 12-bit: 0–4095 counts over 0–Vref (5 V on EVB → ~1.22 mV/count) |
| EVB potentiometer | PTC14 = ADC0 channel SE12; analog is ALT0, no mux write needed |
| Start conversion | Write `ADC0->SC1[0] = ADC_SC1_ADCH(ch)` |
| Done? | Poll `COCO` in SC1[0]; read result from `ADC0->R[0]` |
| Calibration | Run once at init (`ADC_DRV_AutoCalibration`) — free accuracy |
| Scaling | Integer math: `(raw × range) / 4095`; convert to units immediately |
| NTC thermistor | Nonlinear → lookup table + interpolation |
| EMA filter | `filt += (raw − filt) >> 3` |
| Plausibility | Near-rail readings = wiring fault → debounce, substitute, DTC |

## Exercise

Write a `sensor.c` module for a coolant-temperature input: every call to
`sensor_task_100Hz()` reads ADC0 SE12, applies plausibility limits (min/max
raw with a 5-sample fault debounce), an EMA filter, and the NTC lookup
table, and stores the result via `int16_t sensor_get_temp_dC(void)` plus a
`bool sensor_is_valid(void)`. Then print both over UART (module 5) once per
second. No hardware? Desk-test the logic in a plain C file on your PC: feed
it a scripted sequence of raw values — stable 2750, noisy ±30, then a jump
to 10 (harness short) — and print what the module reports at each step.
That host-testing trick is real practice: the capstone reuses this module
unchanged.
