---
title: "S32K Automotive Embedded Interview Questions and Answers (179 Questions)"
description: "179 S32K Automotive Embedded interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the S32K Automotive Embedded Mastery Path."
---

# S32K Automotive Embedded Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free S32K Automotive Embedded Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · The Automotive Embedded Landscape &amp; the S32K Family](level-1/01-automotive-landscape-s32k.md)

**Q1.** What most distinguishes an automotive MCU from a hobby Cortex-M board?

**Answer:** Qualification for harsh temperature, long life and functional safety, plus automotive peripherals such as CAN and LIN

Both may be Cortex-M cores; the ecosystem, qualification and safety features differ.

**Q2.** Which peripheral is the backbone of in-vehicle networking on S32K parts?

**Answer:** FlexCAN

CAN connects ECUs in the vehicle; LIN serves cheap sub-networks.

**Q3.** What does ISO 26262 define?

**Answer:** Functional safety for road-vehicle electrical and electronic systems, including ASIL levels

ASIL A to D rate risk, with D the most demanding.

**Q4.** What is AUTOSAR?

**Answer:** A standardised software architecture for automotive ECUs (Classic and Adaptive)

It standardises layers like MCAL, BSW and RTE.

### [02 · Toolchain Setup — S32 Design Studio](level-1/02-toolchain-s32-design-studio.md)

**Q5.** What does S32 Design Studio provide?

**Answer:** An Eclipse-based IDE with GCC toolchain, SDK and debugging for S32K

It bundles project creation, configuration tools and debug support.

**Q6.** Why use an SDK or RTD instead of writing register code everywhere?

**Answer:** Tested drivers speed development and reduce bugs, while register-level knowledge still matters for debugging

Know what the driver does under the hood.

**Q7.** What does a debugger probe such as P&amp;E or J-Link allow?

**Answer:** Flashing, halting, stepping and inspecting registers over SWD

SWD uses two pins plus ground.

**Q8.** Why commit the generated configuration files with the project?

**Answer:** So the build is reproducible and others get the same pin and clock setup

Configuration is part of the source.

### [03 · Clocks, Power &amp; Boot](level-1/03-clocks-power-boot.md)

**Q9.** Which clock source does the S32K1 boot on?

**Answer:** FIRC, the 48 MHz fast internal RC oscillator

Software then enables the crystal and PLL for full speed.

**Q10.** What clock rates result from this S32K1 divider chain?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){int sosc=8; int vco=sosc*40; int spll=vco/2; printf("%d %d %d %.2f\n",spll,spll/2,spll/4,(double)spll/6);}
```

**Answer:** 160 80 40 26.67

8 MHz x 40 = 320 MHz VCO, /2 = 160 MHz SPLL output; the course divides by 2 for the 80 MHz core, 4 for the 40 MHz bus and 6 for the roughly 26.67 MHz flash clock. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q11.** Why does CAN need a crystal-grade clock rather than an internal RC oscillator?

**Answer:** Bit timing tolerance is tight, so an inaccurate clock causes errors

Internal RC drifts with temperature.

**Q12.** What is the purpose of the PCC (peripheral clock controller)?

**Answer:** Gate and select the clock for each peripheral before using it

A peripheral without a clock ignores register writes.

### [04 · GPIO &amp; Pin Muxing](level-1/04-gpio-pin-muxing.md)

**Q13.** Why must a pin be muxed before use?

**Answer:** Each pin can serve several functions and the PORT module selects which one is connected

PORTx-&gt;PCR[n] MUX field.

**Q14.** What happens if you write a GPIO output but the pin is muxed to another function?

**Answer:** The pin does not change because GPIO is not connected to it

Check the mux first.

**Q15.** Which register typically sets bits atomically without read-modify-write?

**Answer:** PSOR/PCOR/PTOR (set, clear, toggle)

Atomic bit operations avoid race conditions with interrupts.

**Q16.** What does this bit manipulation print?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){uint32_t r=0; r|=(1u<<3)|(1u<<5); r&=~(1u<<3); r^=(1u<<0); printf("0x%02X\n",r);}
```

**Answer:** 0x21

Set bits 3 and 5, clear bit 3, toggle bit 0. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

### [05 · UART Communication](level-1/05-uart-communication.md)

**Q17.** What baud rate results from SBR = 26 with a 48 MHz clock and OSR+1 = 16?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){double b=48e6/(16.0*26); double e=(b-115200)/115200*100; printf("%.0f %.2f\n",b,e);}
```

**Answer:** 115385 0.16

Baud = clock / (OSR x SBR); the 0.16% error is well within UART tolerance. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q18.** What does 8N1 mean?

**Answer:** 8 data bits, no parity, 1 stop bit

Ten bits per byte on the wire including start.

**Q19.** Why prefer interrupt or DMA driven UART RX over polling?

**Answer:** So bytes are not lost while the CPU does other work

Overrun errors occur if the data register is not read in time.

**Q20.** How long does it take to send 100 bytes at 115200 baud (8N1)?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){double t=100*10/115200.0*1000; printf("%.2f ms\n",t);}
```

**Answer:** 8.68 ms

Each byte is 10 bits including start and stop. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

### [06 · ADC &amp; Sensor Inputs](level-1/06-adc-sensor-inputs.md)

**Q21.** What voltage does an ADC count of 2048 represent (12-bit, 5 V reference)?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){printf("%.3f\n",2048*5.0/4095);}
```

**Answer:** 2.501

Counts = V/Vref x 4095, so V = counts x Vref / 4095. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q22.** Why average several ADC samples?

**Answer:** To reduce random noise

Oversampling trades time for accuracy.

**Q23.** What limits ADC accuracy when measuring a high-impedance source?

**Answer:** Insufficient sampling time to charge the sample capacitor

Increase the sample time or buffer the input.

**Q24.** What integer temperature (tenths of C) comes from this linear sensor formula?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){int mv=1500; int t10=(mv-500)*10/10; printf("%d\n",t10);}
```

**Answer:** 1000

A 10 mV/degC sensor with a 500 mV offset gives 100.0 C at 1500 mV, i.e. 1000 tenths. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

### [07 · CAN Fundamentals](level-1/07-can-fundamentals.md)

**Q25.** Which CAN frame wins arbitration?

**Answer:** The one with the lowest identifier (dominant bits override recessive)

Lower ID means higher priority.

**Q26.** What is bit stuffing?

**Answer:** Inserting an opposite-polarity bit after five identical bits to keep edges for synchronisation

The receiver removes stuffed bits.

**Q27.** How many bits does a classic CAN base frame need on the wire at most for 8 data bytes before stuffing?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){int bits=1+11+1+1+1+4+64+15+1+1+1+7+3; printf("%d\n",bits);}
```

**Answer:** 111

SOF, ID, RTR, IDE, r0, DLC, data, CRC, delimiters, ACK, EOF and interframe space add up to 111 before stuffing. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q28.** What does the ACK slot mean?

**Answer:** At least one node received the frame without error (receivers drive it dominant)

The transmitter only knows someone acknowledged.

### [08 · Timers &amp; PWM](level-1/08-timers-pwm.md)

**Q29.** What PWM compare value gives a 25% duty with a 1000-tick period?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){uint32_t mod=999, duty=25; uint32_t cv=(mod+1)*duty/100; printf("%u\n",cv);}
```

**Answer:** 250

CnV = (MOD+1) x duty. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q30.** What is the PWM frequency for a 48 MHz timer clock, prescaler 1 and MOD 999?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){printf("%.0f\n",48e6/1000);}
```

**Answer:** 48000

f = clock / (MOD+1). (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q31.** What does an input-capture channel measure?

**Answer:** The time of an edge, for pulse width or frequency

The difference between captures gives period.

**Q32.** Why use a hardware timer for periodic tasks instead of a delay loop?

**Answer:** It is precise, frees the CPU and works with interrupts or DMA

LPIT 10 ms tick example.

### [09 · Safety &amp; Robustness Basics](level-1/09-safety-robustness.md)

**Q33.** What does a watchdog do?

**Answer:** Resets the MCU if software fails to refresh it in time

Refresh only when the system is verifiably healthy.

**Q34.** Why is petting the watchdog from a timer ISR a bad idea?

**Answer:** It can keep the system 'alive' while the main loop is dead

Refresh from the supervised task logic.

**Q35.** What does a CRC over flash at startup detect?

**Answer:** Corruption of the program image

A basic safety mechanism.

**Q36.** Why validate sensor readings for plausibility?

**Answer:** A shorted or open sensor can produce out-of-range values that must trigger a safe state

Check range and rate of change.

### [10 · Capstone — Design a CAN Sensor Node](level-1/10-capstone-can-sensor-node.md)

**Q37.** What is the capstone goal?

**Answer:** Design a CAN sensor node: sample, scale, frame, transmit and handle faults

Bring the level together.

**Q38.** How often should the node transmit its sensor message?

**Answer:** At a fixed, documented period so receivers can detect loss

Periodic messages make timeouts meaningful.

**Q39.** How many bytes does a message with a 16-bit signal, a 4-bit counter and a 4-bit status need?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){int bits=16+4+4; printf("%d\n",(bits+7)/8);}
```

**Answer:** 3

24 bits is exactly 3 bytes. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q40.** What should the node do if the ADC fails self-check?

**Answer:** Send a defined fault status and avoid transmitting stale valid-looking data

Fail safe and visibly.

## Level 2 · Intermediate

### [FlexCAN Deep Dive — Mailboxes &amp; Filtering](level-2/01-flexcan-deep-dive.md)

**Q41.** What is a FlexCAN message buffer (mailbox)?

**Answer:** A hardware slot holding one CAN frame for transmit or receive

Rx FIFO can aggregate several.

**Q42.** What do acceptance filters and masks do?

**Answer:** Select which received IDs are stored, reducing CPU load

Mask bits set to 1 must match.

**Q43.** Does ID 0x123 pass filter 0x120 with mask 0x7F0?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){uint32_t id=0x123, f=0x120, m=0x7F0; printf("%d %d\n",(id&m)==(f&m),(0x133&m)==(f&m));}
```

**Answer:** 1 0

Only masked bits are compared. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q44.** What does the error passive state mean?

**Answer:** The node has seen many errors and is restricted in how it signals errors

Bus-off disconnects the node until recovery.

### [eDMA — Direct Memory Access](level-2/02-edma.md)

**Q45.** What does eDMA do?

**Answer:** Moves data between memory and peripherals without CPU involvement

Triggered by peripheral requests.

**Q46.** What is a TCD?

**Answer:** Transfer control descriptor holding source, destination, sizes and counts for a DMA channel

Chained TCDs form scatter-gather.

**Q47.** Why use DMA for ADC sampling into a buffer?

**Answer:** Samples arrive at fixed rates and the CPU can sleep or compute

Use half/complete interrupts.

**Q48.** What hazard exists with DMA and caches or shared memory?

**Answer:** The CPU can read stale data unless synchronised or buffers are placed correctly

More relevant on cached cores like S32K3.

### [LPSPI &amp; LPI2C — Off-Chip Peripherals](level-2/03-lpspi-lpi2c.md)

**Q49.** Which bus fits a short-range, multi-device sensor chain using chip selects?

**Answer:** SPI (LPSPI)

Full-duplex, clocked by the master.

**Q50.** What addressing does I2C use?

**Answer:** A 7-bit (or 10-bit) device address on a two-wire bus

LPI2C is the peripheral.

**Q51.** Why do I2C lines need pull-up resistors?

**Answer:** Devices only pull low (open-drain)

Pull-up value affects rise time.

**Q52.** What does SPI mode (CPOL/CPHA) define?

**Answer:** Clock idle level and which edge samples data

Master and slave must match.

### [FreeRTOS on S32K](level-2/04-freertos-s32k.md)

**Q53.** What does an RTOS scheduler give embedded firmware?

**Answer:** Deterministic task scheduling by priority with timing services

FreeRTOS is common.

**Q54.** What is priority inversion?

**Answer:** A high-priority task waits for a lock held by a low-priority one that a medium task preempts

Priority inheritance mutexes help.

**Q55.** Why must ISRs use the FromISR API variants?

**Answer:** Normal calls may block, which is illegal in interrupt context

Also respect priority limits.

**Q56.** How do you size a task stack?

**Answer:** Measure high-water mark during worst-case tests and add margin

uxTaskGetStackHighWaterMark.

### [Low-Power Modes &amp; Wakeup Sources](level-2/05-low-power-modes.md)

**Q57.** What does a low-power mode trade?

**Answer:** Current consumption against wake-up latency and available peripherals

Pick the deepest mode that meets timing.

**Q58.** Which is a typical wake-up source from deep sleep?

**Answer:** CAN activity, LPTMR or a GPIO edge

Check the mode's allowed sources.

**Q59.** What must be done before entering a low-power mode?

**Answer:** Finish or drain pending transfers and configure wake sources

Otherwise data may be lost.

**Q60.** Average current of an ECU awake 10 ms per second (20 mA) and asleep otherwise (50 uA)?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){double a=(0.010*20+0.990*0.05); printf("%.4f mA\n",a);}
```

**Answer:** 0.2495 mA

Duty-weighted average. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

### [Flash, FlexNVM &amp; EEPROM Emulation](level-2/06-flash-flexnvm-eeprom.md)

**Q61.** Why must flash be erased before programming?

**Answer:** Programming can only change bits from 1 to 0

Erase works on whole sectors.

**Q62.** What does EEPROM emulation on FlexNVM provide?

**Answer:** Byte-wise non-volatile storage with wear levelling on top of flash

Endurance is much better than raw flash.

**Q63.** Why must code writing flash not run from the same flash block on some parts?

**Answer:** The flash is busy during program/erase and cannot supply instructions

Run from RAM or another block.

**Q64.** How many sectors of 4 KiB are needed for 100 KiB?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){int n=(100*1024+4095)/4096; printf("%d\n",n);}
```

**Answer:** 25

Round up. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

### [S32K1 to S32K3 Migration](level-2/07-s32k1-to-s32k3.md)

**Q65.** What is a big difference between S32K1 and S32K3 cores?

**Answer:** S32K3 uses Cortex-M7 (higher performance, caches) while S32K1 uses Cortex-M4F or M0+

Check the reference manual for the exact part.

**Q66.** What do migrations from S32K1 to S32K3 typically require?

**Answer:** New drivers/RTD, different clock and pin tools and attention to caches and memory map

Plan time for peripherals.

**Q67.** Why not rely on register-level code when migrating?

**Answer:** Peripheral register layouts differ, so abstraction layers ease the move

HAL/RTD helps.

**Q68.** What new feature appears on many S32K3 parts?

**Answer:** Hardware security engine (HSE) and multicore options

Depends on the variant.

### [Bootloader Basics &amp; Reprogramming](level-2/08-bootloader-basics.md)

**Q69.** What is a bootloader's job?

**Answer:** Run first, check and program the application image, then jump to it

Needs a protected region.

**Q70.** How does a bootloader hand over to the application?

**Answer:** Set the vector table (VTOR) and stack pointer from the app image and branch to reset handler

Disable interrupts first.

**Q71.** Why verify an image before jumping?

**Answer:** A partially programmed or corrupted image could crash or behave unsafely

CRC or signature.

**Q72.** What is a CRC-8 (poly 0x07) of bytes 0x31 0x32 0x33?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){uint8_t d[]={0x31,0x32,0x33}; uint8_t c=0; for(int i=0;i<3;i++){c^=d[i]; for(int b=0;b<8;b++) c=(c&0x80)?(c<<1)^0x07:(c<<1);} printf("0x%02X\n",c);}
```

**Answer:** 0xC0

A bit-by-bit CRC-8 as used for image or message checks. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

### [UDS Diagnostics Intro (ISO 14229)](level-2/09-uds-diagnostics.md)

**Q73.** What is the positive response SID for UDS service 0x22?

**Answer:** 0x62 (request SID + 0x40)

Negative responses use 0x7F, SID, NRC.

**Q74.** What does NRC 0x78 mean?

**Answer:** Request correctly received, response pending

The ECU needs more time.

**Q75.** Which UDS service switches diagnostic sessions?

**Answer:** 0x10 DiagnosticSessionControl

Default, programming and extended sessions.

**Q76.** What frame does the server send for a 0x22 request of DID 0xF190 returning 'ABC'?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){uint8_t r[]={0x62,0xF1,0x90,'A','B','C'}; for(int i=0;i<6;i++) printf("%02X ",r[i]); printf("\n");}
```

**Answer:** 62 F1 90 41 42 43

Positive response = SID+0x40, DID echo, then data. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

### [Project — Multi-Node CAN Network](level-2/10-project-multi-node-can.md)

**Q77.** In a multi-node CAN network what keeps message IDs manageable?

**Answer:** A shared database such as a DBC file describing messages and signals

One source of truth.

**Q78.** How can a receiver detect a missing sender?

**Answer:** A timeout on the expected periodic message

Plus alive counters.

**Q79.** What is an alive counter for?

**Answer:** Detecting stale or repeated frames

Increment each transmission and wrap.

**Q80.** Why leave bus load headroom (for example below 50%)?

**Answer:** To keep latency bounded and allow diagnostics and growth

Measure worst case.

## Level 3 · Advanced

### [AUTOSAR Classic — MCAL, BSW &amp; RTE](level-3/01-autosar-classic.md)

**Q81.** What does MCAL do in AUTOSAR Classic?

**Answer:** Provides hardware-specific drivers for the microcontroller peripherals

Lowest BSW layer.

**Q82.** What is the RTE?

**Answer:** The runtime environment connecting software components to each other and to the BSW

It is generated from the configuration.

**Q83.** Why is the AUTOSAR layered model useful?

**Answer:** Application software can be reused across ECUs and microcontrollers

Hardware is abstracted.

**Q84.** Where do you configure AUTOSAR modules?

**Answer:** In configuration tools producing generated C code

Configuration is part of the product.

### [Real-Time Drivers (RTD) &amp; MCAL Config](level-3/02-rtd-mcal-config.md)

**Q85.** What are RTD drivers?

**Answer:** NXP's real-time drivers (MCAL-style) for S32K3, configured in the S32 Config Tools

Generated configuration plus driver code.

**Q86.** Why keep the configuration under version control?

**Answer:** Generated code must be reproducible and reviewable

Tool versions matter too.

**Q87.** What is a common mistake with MCAL clock configs?

**Answer:** Enabling a peripheral without its clock or with the wrong source

Symptoms look like dead hardware.

**Q88.** What does the MCAL initialisation order matter for?

**Answer:** Dependencies such as clock and port must be ready before CAN or ADC

Follow the vendor's sequence.

### [CAN FD Networks &amp; DBC Workflows](level-3/03-can-fd-dbc.md)

**Q89.** What can CAN FD do that classic CAN cannot?

**Answer:** Send up to 64 data bytes and switch to a faster bit rate in the data phase

BRS bit switches rate.

**Q90.** How many data bytes does CAN FD DLC code 13 represent?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){int m[16]={0,1,2,3,4,5,6,7,8,12,16,20,24,32,48,64}; printf("%d\n",m[13]);}
```

**Answer:** 32

DLC 9 to 15 map to 12, 16, 20, 24, 32, 48, 64. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q91.** What is a DBC file?

**Answer:** A database describing CAN messages, signals, scaling and nodes

Used by tools and code generators.

**Q92.** Extract a 12-bit little-endian signal starting at bit 4 from bytes {0xA5,0x3C}.

```c
#include <stdio.h>
#include <stdint.h>
int main(void){uint8_t d[2]={0xA5,0x3C}; uint16_t raw=d[0]|(d[1]<<8); uint16_t s=(raw>>4)&0xFFF; printf("0x%03X %u\n",s,s);}
```

**Answer:** 0x3CA 970

Little-endian assembling then shift and mask. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

### [LIN &amp; FlexIO](level-3/04-lin-flexio.md)

**Q93.** What is LIN?

**Answer:** A low-cost, single-master serial bus for sub-networks like windows and seats

Up to 20 kbit/s.

**Q94.** What does FlexIO provide?

**Answer:** Flexible configurable I/O that can emulate UART, SPI, I2C and custom protocols

Hardware-timed pin sequences.

**Q95.** Which node schedules frames on a LIN bus?

**Answer:** The master, using a schedule table

Slaves respond to headers.

**Q96.** What does the LIN enhanced checksum include?

**Answer:** The protected ID plus the data bytes

Classic checksum covers data only.

### [Safety Mechanisms — SBC &amp; External Watchdogs](level-3/05-sbc-external-watchdog.md)

**Q97.** What does an SBC (system basis chip) provide?

**Answer:** Power supplies, watchdog and CAN/LIN transceivers in one safety-oriented device

Often includes an external watchdog.

**Q98.** Why use an external watchdog in addition to the MCU's own?

**Answer:** It stays independent if the MCU clock or core fails

Independent monitoring is a safety mechanism.

**Q99.** What is a challenge/response (Q&amp;A) watchdog?

**Answer:** The monitor asks questions the software must answer correctly within time

It checks program flow, not just aliveness.

**Q100.** What should happen if the watchdog expires?

**Answer:** Enter a safe state, record the cause and reset

Log for diagnostics.

### [MPU &amp; Freedom from Interference](level-3/06-mpu-freedom-from-interference.md)

**Q101.** What does the MPU enforce?

**Answer:** Access permissions for memory regions per task or core

Faults on violations.

**Q102.** What is freedom from interference?

**Answer:** Lower-ASIL software cannot corrupt higher-ASIL software

A key ISO 26262 concept.

**Q103.** Why give each task its own stack region?

**Answer:** A stack overflow in one task is caught instead of silently corrupting another

MPU or guard regions.

**Q104.** What should the MPU fault handler do?

**Answer:** Log, enter a safe state and recover or reset according to safety goals

Define the reaction.

### [Secure Boot &amp; the HSE Security Engine](level-3/07-secure-boot-hse.md)

**Q105.** What does secure boot do?

**Answer:** Verifies the signature of the firmware before running it

A root of trust anchors the chain.

**Q106.** What is the HSE?

**Answer:** The hardware security engine providing key storage and crypto operations

Keys never leave the HSE.

**Q107.** Why not store signing keys in application flash?

**Answer:** They could be read and used to forge firmware

Use a secure keystore.

**Q108.** What does a signature prove?

**Answer:** Authenticity and integrity of the image from the key holder

Not encryption.

### [Automotive Ethernet &amp; Gateways](level-3/08-automotive-ethernet-gateways.md)

**Q109.** What does an automotive Ethernet gateway do?

**Answer:** Routes frames between networks such as CAN and Ethernet

Mind latency and filtering.

**Q110.** What is 100BASE-T1?

**Answer:** Automotive 100 Mbit/s Ethernet over a single twisted pair

Lower weight cabling.

**Q111.** Why filter traffic at a gateway?

**Answer:** To stop faulty or malicious messages crossing domains

Security and robustness.

**Q112.** What is a risk in gateway routing?

**Answer:** Added latency and bus overload if signals are forwarded blindly

Shape and prioritise.

### [Measurement &amp; Calibration (XCP)](level-3/09-xcp-measurement-calibration.md)

**Q113.** What is XCP used for?

**Answer:** Measuring and calibrating ECU variables over CAN, Ethernet and other transports

Uses an A2L description.

**Q114.** What is an A2L file?

**Answer:** A description of ECU variables, addresses and calibration parameters

Generated from the ELF/map.

**Q115.** Why disable XCP in production firmware?

**Answer:** It exposes memory read/write and is a security risk

Or protect it with seed and key.

**Q116.** What is calibration?

**Answer:** Tuning parameters in a running ECU without reflashing

Page switching helps.

### [Project — Diagnostic-Capable Body Controller](level-3/10-project-body-controller.md)

**Q117.** What does a diagnostic-capable body controller add over a simple node?

**Answer:** UDS services, DTC storage and session/security handling

Workshop and end-of-line use.

**Q118.** What is a DTC?

**Answer:** A diagnostic trouble code identifying a detected fault

Stored with status and freeze frame.

**Q119.** Why debounce faults before setting a DTC?

**Answer:** To avoid logging transient glitches

Counter-based debouncing.

**Q120.** What does service 0x14 do?

**Answer:** Clears diagnostic information

0x19 reads DTCs.

## Level 4 · Master

### [Multi-Core S32K3 &amp; Lockstep](level-4/01-multicore-lockstep.md)

**Q121.** What is lockstep?

**Answer:** Two cores execute the same instructions and a checker compares outputs to detect faults

Used for ASIL-D.

**Q122.** What happens when lockstep detects a mismatch?

**Answer:** The fault reaction occurs, typically reset or safe state

Defined by the safety concept.

**Q123.** What is asymmetric multicore for?

**Answer:** Running different workloads, such as safety and non-safety, on separate cores

With protected shared memory.

**Q124.** Why need inter-core communication care?

**Answer:** Shared memory needs synchronisation and protection against corruption

Use mailbox or semaphores.

### [ASIL-D Development Workflow](level-4/02-asil-d-workflow.md)

**Q125.** What does ASIL D require compared with ASIL A?

**Answer:** More rigorous development, verification and diagnostic coverage

Per ISO 26262.

**Q126.** What does a safety goal describe?

**Answer:** A top-level safety requirement derived from hazard analysis

Decomposed into requirements.

**Q127.** What is traceability?

**Answer:** Linking requirements to design, code and tests

Needed for audits.

**Q128.** What does FMEA analyse?

**Answer:** How components can fail and the effects

Part of safety analysis.

### [MISRA C &amp; Static Analysis in Practice](level-4/03-misra-static-analysis.md)

**Q129.** What is MISRA C?

**Answer:** A coding guideline subset of C for safety- and reliability-critical systems

Rules and directives.

**Q130.** Why run static analysis?

**Answer:** Find defects and rule violations without executing the code

Run in CI.

**Q131.** What should you do with a justified rule violation?

**Answer:** Document a deviation with rationale and review it

Formal deviation process.

**Q132.** What does this MISRA-style unsigned wraparound print?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){uint8_t a=250; a=(uint8_t)(a+10); printf("%u\n",a);}
```

**Answer:** 4

Unsigned arithmetic wraps modulo 256; MISRA asks you to make intent explicit. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

### [OTA &amp; Reprogramming Stacks](level-4/04-ota-reprogramming.md)

**Q133.** Why do OTA updates need rollback?

**Answer:** A failed update must not brick the ECU

Dual banks help.

**Q134.** What is an A/B (dual-bank) layout?

**Answer:** Two image slots so the new image is verified before switching

Swap atomically.

**Q135.** What should an update package contain?

**Answer:** The image, version, compatibility info and a signature

Verify before install.

**Q136.** Why check battery and vehicle state before updating?

**Answer:** Power loss or driving mid-update is unsafe

Policy conditions.

### [Cybersecurity — ISO/SAE 21434 &amp; SecOC](level-4/05-cybersecurity-secoc.md)

**Q137.** What does SecOC add to a CAN message?

**Answer:** A truncated MAC and freshness value so receivers can authenticate it

Protects integrity and replay.

**Q138.** What is a freshness value?

**Answer:** A counter or timestamp preventing replay of old messages

Synchronised between nodes.

**Q139.** What is ISO/SAE 21434?

**Answer:** Cybersecurity engineering for road vehicles

Threat analysis and risk assessment.

**Q140.** Why is CRC not security?

**Answer:** Anyone can recompute it; a MAC needs a secret key

CRC detects accidents, not attacks.

### [AUTOSAR OS &amp; Timing Analysis](level-4/06-autosar-os-timing.md)

**Q141.** What does a schedule table or alarm do in AUTOSAR OS?

**Answer:** Activates tasks at defined times

Statically configured.

**Q142.** What is WCET?

**Answer:** Worst-case execution time of a task

Needed for schedulability.

**Q143.** Why is response-time analysis used?

**Answer:** To prove tasks meet deadlines under worst-case interference

Rate monotonic etc.

**Q144.** Is a task set with utilisations 0.3, 0.2, 0.2 under the 3-task RM bound (0.78)?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){double u=0.3+0.2+0.2; printf("%.2f %s\n",u,u<=0.7798?"ok":"check");}
```

**Answer:** 0.70 ok

Liu and Layland: n(2^(1/n)-1) is about 0.78 for three tasks. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

### [HIL Testing &amp; CI for ECU Firmware](level-4/07-hil-testing-ci.md)

**Q145.** What is HIL testing?

**Answer:** Running the real ECU against simulated vehicle signals in a lab rig

Hardware-in-the-loop.

**Q146.** What does CI add for ECU firmware?

**Answer:** Every commit is built, statically analysed and tested automatically

Catch regressions early.

**Q147.** Why include unit tests on the host?

**Answer:** Fast feedback without hardware

Abstract hardware access.

**Q148.** What should HIL results be tied to?

**Answer:** Requirements, for traceability

Evidence for safety cases.

### [Production ECU Hardware Architecture](level-4/08-production-ecu-architecture.md)

**Q149.** What belongs on a production ECU besides the MCU?

**Answer:** Power supply, SBC, transceivers, protection and connectors

Plus EMC design.

**Q150.** Why add reverse-polarity and load-dump protection?

**Answer:** Vehicle supply can be wired wrong or spike dramatically

Automotive electrical environment.

**Q151.** What is EMC concern with fast edges?

**Answer:** Emissions and immunity, so layout and filtering matter

Test early.

**Q152.** Why pick parts with automotive qualification (AEC-Q)?

**Answer:** Proven reliability over temperature and life

Document in the BOM.

### [Manufacturing, EOL Test &amp; Traceability](level-4/09-manufacturing-eol-test.md)

**Q153.** What is EOL (end-of-line) testing?

**Answer:** Final functional tests and programming at the factory

Use diagnostics.

**Q154.** Why store traceability data (serial, versions) per unit?

**Answer:** To trace field issues back to production batches

Recall scope.

**Q155.** What does the factory typically write to each ECU?

**Answer:** Software, calibration, keys/serial and test results

Secure the keys.

**Q156.** Why make EOL tests fast and automated?

**Answer:** Throughput and repeatability

Cycle time costs money.

### [Capstone — Production-Grade Zone ECU](level-4/10-capstone-zone-ecu.md)

**Q157.** What is a zone ECU?

**Answer:** A regional controller aggregating I/O and gateway functions for part of the vehicle

Reduces wiring.

**Q158.** What does the capstone combine?

**Answer:** Multicore safety, secure boot, diagnostics, OTA and networking

A production-grade design.

**Q159.** What evidence shows readiness?

**Answer:** Requirements trace, tests, static analysis, safety analysis and security review

Document it.

**Q160.** What should you do with open safety assumptions?

**Answer:** Record them and verify them as requirements on integration

Assumptions of use.

## Mixed questions across all levels

**Q161.** What does this print?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){uint8_t a=200,b=100; uint8_t s=a+b; uint16_t w=(uint16_t)a+b; int8_t n=(int8_t)200; printf("%u %u %d\n",s,w,n);}
```

**Answer:** 44 300 -56

uint8_t arithmetic wraps modulo 256; casting 200 to int8_t gives -56 on two's complement targets. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q162.** What does this print?

```c
#include <stdio.h>
#include <stdint.h>
#define BIT(n) (1u<<(n))
int main(void){uint32_t reg=0xF0; reg&=~(BIT(7)|BIT(4)); reg|=BIT(1); printf("0x%02X %d\n",reg,(reg&BIT(5))!=0);}
```

**Answer:** 0x62 1

Clear bits 7 and 4, set bit 1; bit 5 remains set. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q163.** What is the CAN bus load for this schedule at 500 kbit/s? (frames are 111 bits, 10 per 10 ms)

```c
#include <stdio.h>
#include <stdint.h>
int main(void){double bits=111*10; double cap=500000*0.010; printf("%.1f%%\n",100*bits/cap);}
```

**Answer:** 22.2%

1110 bits in a 10 ms window with 5000 bits of capacity. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q164.** How many bits is the CAN bit time for 500 kbit/s, and how many time quanta at an 8 MHz time-quantum clock?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){double t=1e6/500000.0; printf("%.0f us %.0f tq\n",t,t*8);}
```

**Answer:** 2 us 16 tq

Bit time = 1/baud; tq count = bit time x tq clock. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q165.** What is the sample point for 16 tq with sync 1, PROP+PS1 = 12 and PS2 = 3?

```c
#include <stdio.h>
#include <stdint.h>
int main(void){printf("%.1f%%\n",100.0*(1+12)/16);}
```

**Answer:** 81.2%

Sample point = (sync + seg1) / total, here 13/16 (81.25%, printed as 81.2% by round-half-even). (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked.

**Q166.** Why is a CRC alone not enough to secure a CAN message?

**Answer:** Anyone can recompute it; authentication needs a keyed MAC with freshness (SecOC)

CRC detects accidental errors only.

**Q167.** What is lowest-ID-wins arbitration an example of?

**Answer:** Non-destructive bitwise arbitration, with dominant bits overriding recessive

The winner's frame is not corrupted.

**Q168.** What do you need to enable before using a peripheral on S32K1?

**Answer:** Its PCC clock (and the right clock source), then pin muxing if it uses pins

Writes to an unclocked peripheral are ignored or fault.

**Q169.** What is the main reason to use DMA for a fast ADC stream?

**Answer:** Move samples without CPU per-sample interrupts

CPU stays free.

**Q170.** What is wrong with refreshing the watchdog in a periodic timer ISR?

**Answer:** The main loop can be dead while the ISR keeps feeding it

Supervise real task progress.

**Q171.** Which UDS response means 'service not supported in the active session'?

**Answer:** NRC 0x7F

0x11 is service not supported at all.

**Q172.** Why do bootloaders verify the image before jumping to it?

**Answer:** A partial or corrupted image could crash or behave unsafely

Use CRC at minimum, a signature for authenticity.

**Q173.** What does freedom from interference require?

**Answer:** Lower-integrity software cannot corrupt higher-ASIL software (MPU, partitioning, timing protection)

A key ISO 26262 concept.

**Q174.** What does SecOC add to a PDU?

**Answer:** A truncated MAC computed over data and a freshness value

Authenticity and replay protection.

**Q175.** What does lockstep detect?

**Answer:** Random hardware faults in the core, by comparing two cores' outputs

Software bugs execute identically on both cores.

**Q176.** Why disable or protect XCP in production firmware?

**Answer:** It allows memory read/write, which is a security risk

Seed-and-key or removal.

**Q177.** What makes an A/B OTA layout safer?

**Answer:** The new image is verified in the inactive bank before switching, with rollback

Avoids bricking.

**Q178.** What is a typical production ECU evidence set for ASIL claims?

**Answer:** Requirements trace, reviews, static analysis, tests with coverage and safety analysis

Evidence, not assertion.

**Q179.** What is the right reaction to a failed ADC plausibility check?

**Answer:** Report a defined fault status, avoid sending stale values as valid, and enter the safe state

Fail safe and visibly.

## Practical coding questions

### Pack and unpack a CAN message with CRC and alive counter

Implement crc8, encode and decode for a sensor message: a signed 0.1 C temperature, an rpm value, a 4-bit alive counter and a CRC-8 in byte 7. Decode must reject corrupted frames and wrong counters. This runs on your computer as ordinary C; it models the signal packing you would do on an S32K node, but uses no hardware.

Starter code:

```
#include <stdio.h>
#include <stdint.h>
#include <string.h>

typedef struct { int16_t temp_c10; uint16_t rpm; uint8_t counter; } Sample;

// TODO: static uint8_t crc8(const uint8_t *d, int n)            CRC-8, polynomial 0x07, initial value 0
// TODO: static void encode(const Sample *s, uint8_t f[8])        bytes 0-1 temp (LE, 0.1 C), 2-3 rpm (LE),
//       byte 4 low nibble = alive counter, byte 7 = CRC-8 over bytes 0..6, unused bytes 0
// TODO: static int decode(const uint8_t f[8], Sample *s, uint8_t expected_counter)
//       returns -1 on CRC mismatch, -2 on counter mismatch, 0 on success (check CRC first)

int main(void) {
    Sample in = { -125, 3000, 15 }, out;
    uint8_t f[8];
    encode(&in, f);
    for (int i = 0; i < 8; i++) printf("%02X ", f[i]);
    printf("\n");
    printf("decode: %d temp=%.1f rpm=%u ctr=%u\n", decode(f, &out, 15), out.temp_c10 / 10.0, out.rpm, out.counter);
    printf("wrong counter: %d\n", decode(f, &out, 0));
    f[2] ^= 0x01;
    printf("corrupted: %d\n", decode(f, &out, 15));
    return 0;
}
```

Expected output:

```text
83 FF B8 0B 0F 00 00 56 
decode: 0 temp=-12.5 rpm=3000 ctr=15
wrong counter: -2
corrupted: -1
```

??? success "Show a solution"

    ```
    #include <stdio.h>
    #include <stdint.h>
    #include <string.h>

    typedef struct { int16_t temp_c10; uint16_t rpm; uint8_t counter; } Sample;

    static uint8_t crc8(const uint8_t *d, int n) {
        uint8_t c = 0;
        for (int i = 0; i < n; i++) {
            c ^= d[i];
            for (int b = 0; b < 8; b++) c = (c & 0x80) ? (uint8_t)((c << 1) ^ 0x07) : (uint8_t)(c << 1);
        }
        return c;
    }

    /* byte0-1 temp (little endian, signed, 0.1 C), byte2-3 rpm, byte4 low nibble counter, byte7 crc over 0..6 */
    static void encode(const Sample *s, uint8_t f[8]) {
        memset(f, 0, 8);
        f[0] = (uint8_t)(s->temp_c10 & 0xFF);
        f[1] = (uint8_t)((s->temp_c10 >> 8) & 0xFF);
        f[2] = (uint8_t)(s->rpm & 0xFF);
        f[3] = (uint8_t)(s->rpm >> 8);
        f[4] = s->counter & 0x0F;
        f[7] = crc8(f, 7);
    }

    static int decode(const uint8_t f[8], Sample *s, uint8_t expected_counter) {
        if (crc8(f, 7) != f[7]) return -1;
        if ((f[4] & 0x0F) != expected_counter) return -2;
        s->temp_c10 = (int16_t)(f[0] | (f[1] << 8));
        s->rpm = (uint16_t)(f[2] | (f[3] << 8));
        s->counter = f[4] & 0x0F;
        return 0;
    }

    int main(void) {
        Sample in = { -125, 3000, 15 }, out;
        uint8_t f[8];
        encode(&in, f);
        for (int i = 0; i < 8; i++) printf("%02X ", f[i]);
        printf("\n");
        printf("decode: %d temp=%.1f rpm=%u ctr=%u\n", decode(f, &out, 15), out.temp_c10 / 10.0, out.rpm, out.counter);
        printf("wrong counter: %d\n", decode(f, &out, 0));
        f[2] ^= 0x01;
        printf("corrupted: %d\n", decode(f, &out, 15));
        return 0;
    }
    ```

    Little-endian signals are assembled byte by byte, the counter uses the low nibble, and the CRC covers bytes 0 to 6. Decode checks the CRC before trusting any field, then the counter. Verified by compiling and running with gcc on the host.

### A UDS request handler

Implement respond for services 0x10, 0x22 and 0x3E with the specified negative response codes, including a DID that is only readable in the extended session. This is a host-side model of the service logic; real UDS also needs ISO-TP transport, timing (P2) and security access.

Starter code:

```
#include <stdio.h>
#include <stdint.h>
#include <string.h>

enum { SESSION_DEFAULT = 1, SESSION_EXTENDED = 3 };
static int session = SESSION_DEFAULT;

// TODO: static int respond(const uint8_t *req, int len, uint8_t *resp) -> response length
//   0x10 DiagnosticSessionControl: len must be 2 (else NRC 0x13), sub-function 1 or 3 (else NRC 0x12);
//        positive response 50 <sub> 00 32 01 F4
//   0x22 ReadDataByIdentifier: len must be 3 (NRC 0x13); DID F190 -> 62 F1 90 'V' 'I' 'N';
//        DID 0100 only in the extended session (otherwise NRC 0x7F), returns 62 01 00 2A; any other DID NRC 0x31
//   0x3E TesterPresent: 3E 00 -> 7E 00 (else NRC 0x13)
//   anything else: NRC 0x11 (service not supported). Negative response = 7F <sid> <nrc>

static void run(const char *label, const uint8_t *req, int len) {
    uint8_t resp[16];
    int n = respond(req, len, resp);
    printf("%-14s ->", label);
    for (int i = 0; i < n; i++) printf(" %02X", resp[i]);
    printf("\n");
}

int main(void) {
    run("read VIN", (uint8_t[]){0x22, 0xF1, 0x90}, 3);
    run("read 0100 dflt", (uint8_t[]){0x22, 0x01, 0x00}, 3);
    run("ext session", (uint8_t[]){0x10, 0x03}, 2);
    run("read 0100 ext", (uint8_t[]){0x22, 0x01, 0x00}, 3);
    run("bad subfunc", (uint8_t[]){0x10, 0x02}, 2);
    run("tester present", (uint8_t[]){0x3E, 0x00}, 2);
    run("short read", (uint8_t[]){0x22, 0xF1}, 2);
    run("unknown DID", (uint8_t[]){0x22, 0x12, 0x34}, 3);
    run("unknown svc", (uint8_t[]){0x85, 0x01}, 2);
    return 0;
}
```

Expected output:

```text
read VIN       -> 62 F1 90 56 49 4E
read 0100 dflt -> 7F 22 7F
ext session    -> 50 03 00 32 01 F4
read 0100 ext  -> 62 01 00 2A
bad subfunc    -> 7F 10 12
tester present -> 7E 00
short read     -> 7F 22 13
unknown DID    -> 7F 22 31
unknown svc    -> 7F 85 11
```

??? success "Show a solution"

    ```
    #include <stdio.h>
    #include <stdint.h>
    #include <string.h>

    enum { SESSION_DEFAULT = 1, SESSION_EXTENDED = 3 };
    static int session = SESSION_DEFAULT;

    static int respond(const uint8_t *req, int len, uint8_t *resp) {
        if (len < 1) return 0;
        uint8_t sid = req[0];
        switch (sid) {
        case 0x10:
            if (len != 2) { resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x13; return 3; }
            if (req[1] != 1 && req[1] != 3) { resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x12; return 3; }
            session = req[1];
            resp[0] = 0x50; resp[1] = req[1]; resp[2] = 0x00; resp[3] = 0x32; resp[4] = 0x01; resp[5] = 0xF4;
            return 6;
        case 0x22:
            if (len != 3) { resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x13; return 3; }
            if (req[1] == 0xF1 && req[2] == 0x90) {
                memcpy(resp, (uint8_t[]){0x62, 0xF1, 0x90, 'V', 'I', 'N'}, 6);
                return 6;
            }
            if (req[1] == 0x01 && req[2] == 0x00) {
                if (session != SESSION_EXTENDED) { resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x7F; return 3; }
                memcpy(resp, (uint8_t[]){0x62, 0x01, 0x00, 0x2A}, 4);
                return 4;
            }
            resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x31; return 3;
        case 0x3E:
            if (len == 2 && req[1] == 0x00) { resp[0] = 0x7E; resp[1] = 0x00; return 2; }
            resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x13; return 3;
        default:
            resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x11; return 3;
        }
    }

    static void run(const char *label, const uint8_t *req, int len) {
        uint8_t resp[16];
        int n = respond(req, len, resp);
        printf("%-14s ->", label);
        for (int i = 0; i < n; i++) printf(" %02X", resp[i]);
        printf("\n");
    }

    int main(void) {
        run("read VIN", (uint8_t[]){0x22, 0xF1, 0x90}, 3);
        run("read 0100 dflt", (uint8_t[]){0x22, 0x01, 0x00}, 3);
        run("ext session", (uint8_t[]){0x10, 0x03}, 2);
        run("read 0100 ext", (uint8_t[]){0x22, 0x01, 0x00}, 3);
        run("bad subfunc", (uint8_t[]){0x10, 0x02}, 2);
        run("tester present", (uint8_t[]){0x3E, 0x00}, 2);
        run("short read", (uint8_t[]){0x22, 0xF1}, 2);
        run("unknown DID", (uint8_t[]){0x22, 0x12, 0x34}, 3);
        run("unknown svc", (uint8_t[]){0x85, 0x01}, 2);
        return 0;
    }
    ```

    Validate length first, then sub-function or identifier, then session state, returning 7F SID NRC on failure. Positive responses add 0x40 to the service ID. Verified by compiling and running with gcc on the host; no ECU or CAN hardware was used.

---

More free practice: the [S32K Automotive Embedded Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
