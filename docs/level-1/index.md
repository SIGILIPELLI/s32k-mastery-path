# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: get productive on NXP's S32K automotive microcontroller family in
plain C — understand why automotive MCUs are different, set up the free S32
Design Studio toolchain, bring up clocks, GPIO, UART, ADC, timers and PWM,
learn how CAN actually works, absorb the functional-safety basics every
automotive firmware engineer needs, and finish by designing a complete small
ECU: a CAN sensor node.

**Hardware is optional for this level.** Every lesson works at the
code-and-concepts level with no board at all. If you want to run things for
real, the [S32K144EVB](https://www.nxp.com/design/design-center/development-boards-and-designs/S32K144EVB)
(~$50) is the reference target used throughout, and the free
[Renode](https://renode.io/) emulator is a middle option — lesson 1 lays out
all three paths honestly.

## Modules

1. [The Automotive Embedded Landscape & the S32K Family](01-automotive-landscape-s32k.md)
2. [Toolchain Setup — S32 Design Studio](02-toolchain-s32-design-studio.md)
3. [Clocks, Power & Boot](03-clocks-power-boot.md)
4. [GPIO & Pin Muxing](04-gpio-pin-muxing.md)
5. [UART Communication](05-uart-communication.md)
6. [ADC & Sensor Inputs](06-adc-sensor-inputs.md)
7. [CAN Fundamentals](07-can-fundamentals.md)
8. [Timers & PWM](08-timers-pwm.md)
9. [Safety & Robustness Basics](09-safety-robustness.md)
10. [Capstone — CAN Sensor Node](10-capstone-can-sensor-node.md)

By the end of this level you'll be able to configure the S32K's clocks and
pins, drive LEDs and read buttons and analog sensors, print over a UART
console, explain and use CAN frames, generate PWM and periodic interrupts,
apply watchdogs and plausibility checks, and put it all together in the
design of a small but honest automotive ECU.

If you've never touched a microcontroller before, consider skimming the
[Embedded Systems Mastery Path](https://sigilipelli.github.io/embedded-mastery-path/)
Level 1 first — it covers universal basics (voltage, GPIO, serial) in a free
browser simulator. Rusty C? The
[C Mastery Path](https://sigilipelli.github.io/c-mastery-path/) has you
covered.
