# Production ECU Hardware Architecture

Every module so far has treated the S32K as a given — a chip on a
development board. A production ECU is a PCB, a connector, an enclosure,
and a set of components around the MCU that a firmware engineer needs to
understand even without doing the hardware design personally, because
firmware decisions (which pins, which peripherals, what timing) are
constrained by — and constrain — the hardware around them. This module
covers the production ECU architecture patterns that shape everything
from Level 3's SBC integration to Level 4's OTA flash budget.

## A production body-controller ECU block diagram

```text
┌────────────────────────────────────────────────────────────┐
│  Connector (sealed, automotive-grade, e.g. TE MCP or similar) │
├──────────────┬─────────────────┬─────────────────────────────┤
│ Power input   │ CAN/LIN pins     │ Digital I/O (switches,       │
│ (Vbat, GND)   │ (to harness)     │ actuator drives)              │
├──────────────┴─────────────────┴─────────────────────────────┤
│  Reverse-battery protection + transient suppression (TVS)      │
├────────────────────────────────────────────────────────────┤
│  Power supply: buck regulator (Vbat -> 5V/3.3V rails)           │
│  + SBC (Level 3 module 5: regulator + watchdog + transceivers)  │
├────────────────────────────────────────────────────────────┤
│  S32K MCU                                                       │
│  + external crystal/oscillator                                  │
│  + HSE-provisioned flash (Level 3 module 7)                     │
├────────────────────────────────────────────────────────────┤
│  Output drive stage: high-side/low-side MOSFET drivers for      │
│  actuators (door lock motor, relay coils), with current sense   │
│  feedback to the MCU's ADC                                       │
└────────────────────────────────────────────────────────────┘
```

## Power architecture: why a body controller isn't just "5V in"

```c
/* Firmware-visible consequence of the power architecture: brown-out
   and undervoltage conditions are common in a vehicle (cranking event
   during engine start can sag Vbat to ~6V briefly) and must be handled
   as a normal operating condition, not an exceptional fault */
void Pmic_UndervoltageHandler(void)
{
    /* During cranking, Vbat can sag well below nominal 12V. A body
       controller must NOT brown-out reset every time the engine starts —
       the SBC's regulator and the MCU's own low-voltage detect need
       margin, and firmware needs a defined graceful-degradation
       behavior, not a hard reset */
    if (Adc_ReadVbat_mV() < CRANKING_UNDERVOLTAGE_THRESHOLD_MV) {
        SuspendNonEssentialActuators();  /* reduce load, ride through the sag */
    }
}
```

Cranking voltage sag is the canonical automotive power example: a design
that only validates against a clean, regulated bench supply will pass
every bench test and then reset unpredictably during every cold engine
start in the field, because the actual vehicle power rail behaves
nothing like a lab supply.

## Output drive stages and the firmware/hardware boundary

```c
/* High-side driver for a door lock motor — the MCU's GPIO/PWM output
   controls a gate driver IC, which switches the actual motor current;
   the MCU never switches motor current directly */
void DoorLock_DriveMotor(bool lock_direction, uint8_t duty_percent)
{
    Dio_WriteChannel(DOOR_MOTOR_DIRECTION_PIN, lock_direction);
    Pwm_SetDutyCycle(DOOR_MOTOR_PWM_CHANNEL, duty_percent);

    /* Current sense feedback: the ADC reading here is how firmware
       detects a stalled motor (door mechanism jammed) or a short —
       a hardware protection feature the firmware must actively use,
       not just a hardware safety net running independently */
    uint16_t motor_current_ma = Adc_ReadMotorCurrent();
    if (motor_current_ma > STALL_CURRENT_THRESHOLD_MA) {
        Pwm_SetDutyCycle(DOOR_MOTOR_PWM_CHANNEL, 0u);
        Dem_ReportErrorStatus(DEM_EVENT_MOTOR_STALL, DEM_EVENT_STATUS_FAILED);
    }
}
```

The current-sense stall detection is a direct example of firmware and
hardware jointly implementing a safety/reliability function — the
hardware provides the sense path, but firmware must actively read and
act on it. A hardware review that assumes "the current limit circuit
handles it" and a firmware review that assumes "hardware protects
against overcurrent" can each individually miss that neither side
actually implements shutoff without the other.

## Automotive-MCU concerns

- **Reverse-battery and load-dump protection are hardware features
  firmware must not accidentally defeat.** A firmware update that
  changes GPIO drive strength or adds a pin function without re-checking
  the schematic's protection network assumptions can create a new path
  for a transient to reach the MCU that the original hardware design
  didn't anticipate — pin reassignment is not purely a software decision
  on a production board.
- **Thermal derating limits what firmware can safely command.** An
  actuator driver IC's continuous current rating typically assumes a
  specific ambient temperature and PCB copper area for heat dissipation;
  firmware commanding sustained high duty cycle in a hot engine bay
  environment can exceed the driver's real thermal limit even while
  staying under its absolute maximum current rating — thermal derating
  curves from the driver's datasheet are a firmware input, not just a
  hardware concern.
- **EMC (electromagnetic compatibility) requirements shape firmware
  timing decisions, not just PCB layout.** Switching a high-current
  output at a fixed, harmonically "clean" frequency can radiate more
  interference at automotive-relevant frequencies than a dithered or
  spread-spectrum PWM frequency — some S32K FlexTimer configurations
  support frequency dithering specifically to help meet CISPR 25 EMC
  limits, and choosing not to use it is a decision with real electrical
  test consequences.
- **Connector pin assignment changes are expensive late in a project,
  and firmware pin mapping should be defined through configuration
  (Level 3 module 2's RTD pattern), not scattered magic numbers.** A
  late-stage harness or connector change (common in automotive programs)
  that only requires updating a `Port_Cfg.c` table is a minor firmware
  change; the same change against hand-coded `PORTC->PCR[6]` scattered
  across multiple files is a much larger, higher-risk edit under time
  pressure.

## Cheat sheet

| Hardware element | Firmware-visible consequence |
|--------------------|---------------------------------|
| Reverse-battery protection | Prevents reversed Vbat from damaging the MCU; do not reassign pins without checking the protection network |
| TVS / load dump protection | Absorbs voltage transients (e.g. alternator load dump); firmware should not assume clean power |
| SBC regulator | Powers MCU; brownout/undervoltage during cranking is normal, not exceptional |
| Buck regulator rails | 5V/3.3V typical; firmware ADC references must match the actual regulated rail, not a nominal assumption |
| High/low-side driver + current sense | Firmware must actively read current sense and shut off on stall/overcurrent — hardware alone doesn't do it |
| Crystal/oscillator | Clock accuracy for CAN bit timing (Level 3 module 3) and freshness counters (module 5) depends on it |
| EMC/PWM frequency | Dithered/spread-spectrum PWM can be required to meet CISPR 25 limits |
| Connector pin mapping | Should route through generated config (module 2), not hand-coded register writes |

## Exercise

Review a production ECU schematic (a real automotive reference design if
available — NXP's S32K3 evaluation board schematics are a reasonable
public substitute — or design a minimal one on paper). (1) Trace the
power path from the vehicle battery connector through to the MCU's
3.3V rail, identifying every protection component (reverse-battery,
TVS, regulator) and noting what firmware-visible behavior each one
implies (e.g. "brownout during cranking is expected, not a fault"). (2)
Design the current-sense stall-detection logic for one actuator output,
specifying the ADC channel, the stall threshold, and the shutoff
response — and identify explicitly which half of this mechanism is
hardware and which is firmware. (3) Identify one EMC-relevant firmware
decision in your design (a PWM frequency choice, a GPIO slew rate
setting if configurable) and note what would need to change if a bench
EMC pre-scan flagged excess emissions at that frequency. (4) Write out a
pin reassignment scenario (a connector change moving one signal to a
different physical pin) and confirm your firmware's configuration
structure (module 2's RTD pattern) would contain that change to a single
generated file rather than requiring edits across your application code.
