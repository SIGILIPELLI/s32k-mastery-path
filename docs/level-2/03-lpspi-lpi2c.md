# LPSPI & LPI2C — Off-Chip Peripherals

An ECU is rarely one chip. Around the S32K sit external EEPROMs holding
calibration, pressure and inertial sensors, LED drivers, motor bridge
gate drivers, and — in almost every modern automotive design — a **System
Basis Chip (SBC)** that owns the CAN transceiver, the voltage regulators,
and an external watchdog. Most of them speak **SPI** or **I²C**. This
module covers the S32K's **LPSPI** and **LPI2C** peripherals, the two
transaction patterns you will write a hundred times, and the failure modes
that make off-chip buses disproportionately common sources of field
faults.

## LPSPI: fast, simple, chip-select-driven

SPI is four wires — SCK, SOUT (MOSI), SIN (MISO), and one PCS chip select
per device — full duplex, no addressing, no acknowledgement. Every byte
you send simultaneously receives a byte. The S32K144 has three LPSPI
instances; each supports four PCS lines.

```c
#include "lpspi_master_driver.h"

#define INST_LPSPI0   0u

static lpspi_state_t spiState;

static const lpspi_master_config_t spiCfg = {
    .bitsPerSec      = 4000000u,          /* 4 MHz — stay inside the
                                             slave's spec, not the MCU's */
    .whichPcs        = LPSPI_PCS0,
    .pcsPolarity     = LPSPI_ACTIVE_LOW,
    .isPcsContinuous = false,
    .bitcount        = 8u,
    .lpspiSrcClk     = 48000000u,         /* from PCC: FIRCDIV2         */
    .clkPhase        = LPSPI_CLOCK_PHASE_1ST_EDGE,
    .clkPolarity     = LPSPI_SCK_ACTIVE_HIGH,   /* CPOL = 0, CPHA = 0   */
    .lsbFirst        = false,
    .transferType    = LPSPI_USING_INTERRUPTS,
    .callback        = NULL,
    .callbackParam   = NULL,
};

void spi_init(void)
{
    LPSPI_DRV_MasterInit(INST_LPSPI0, &spiState, &spiCfg);
}
```

`clkPolarity` and `clkPhase` together are the classic **SPI mode**. Get
them wrong and you read plausible-looking garbage — usually every byte
shifted by one bit. The datasheet timing diagram is the authority, and
mode 0 (CPOL 0 / CPHA 0) or mode 3 (CPOL 1 / CPHA 1) covers most
automotive parts.

A blocking register read from an SPI device — the shape of nearly every
SPI transaction you will write:

```c
/* Read one register: send [cmd | addr], then a dummy byte to clock
   the answer back. Full duplex means rx[0] is garbage by definition. */
status_t spi_read_reg(uint8_t addr, uint8_t *value)
{
    uint8_t tx[2] = { (uint8_t)(0x80u | addr), 0xFFu };  /* 0x80 = read */
    uint8_t rx[2] = { 0u, 0u };

    status_t st = LPSPI_DRV_MasterTransferBlocking(INST_LPSPI0, tx, rx,
                                                   2u, 10u /* ms */);
    if (st != STATUS_SUCCESS) {
        return st;                    /* timeout — do NOT use rx[]     */
    }
    *value = rx[1];
    return STATUS_SUCCESS;
}
```

!!! note "Chip select is a design decision"
    `isPcsContinuous = true` holds PCS asserted across a multi-frame
    transfer — required by devices that latch on the *rising edge of CS*,
    such as most SPI EEPROMs during a page write. Devices that expect CS
    to toggle per byte need it `false`. Guessing costs an afternoon.

## The SBC: SPI that can reset your ECU

A System Basis Chip (NXP's UJA116x/FS26 family and equivalents) is the
device the S32K talks to most and forgives least. Over one SPI link it
typically provides:

- the CAN transceiver, including **standby and sleep modes** — this is how
  module 5's low-power design actually silences the bus;
- the 5 V and 3.3 V regulators feeding the MCU;
- a **window watchdog** that must be triggered over SPI within a timing
  window, or it resets the MCU and can disable outputs.

Two consequences for firmware design. First, SPI to the SBC is
**safety-relevant**: a stuck SPI bus means the watchdog is not being
served, which means a reset. Second, initialization order matters — the
SBC usually starts in a restricted mode with a short watchdog period, and
firmware must configure it before that window expires. Read the SBC's
state diagram before writing a line of code; it is the real boot sequence
of the ECU, not the one in `startup.S`.

## LPI2C: two wires, more ways to fail

I²C trades wires for complexity: SDA and SCL, open-drain with pull-ups,
7-bit addressing, and a per-byte acknowledge. The S32K144 has two LPI2C
instances that can act as master or slave.

```c
#include "lpi2c_driver.h"

#define INST_LPI2C0    0u
#define SENSOR_ADDR    0x48u          /* 7-bit address, not shifted     */

static lpi2c_master_state_t i2cState;

static const lpi2c_master_user_config_t i2cCfg = {
    .slaveAddress   = SENSOR_ADDR,
    .is10bitAddr    = false,
    .operatingMode  = LPI2C_FAST_MODE,      /* 400 kHz                  */
    .baudRate       = 400000u,
    .transferType   = LPI2C_USING_INTERRUPTS,
    .masterCallback = NULL,
    .callbackParam  = NULL,
};

void i2c_init(void)
{
    LPI2C_DRV_MasterInit(INST_LPI2C0, &i2cCfg, &i2cState);
}
```

The universal I²C register read is **write the register address without a
stop, then read** — a repeated start:

```c
status_t i2c_read_reg16(uint8_t reg, uint16_t *value)
{
    uint8_t  rx[2];
    status_t st;

    /* sendStop = false → repeated START, keeps the bus owned */
    st = LPI2C_DRV_MasterSendDataBlocking(INST_LPI2C0, &reg, 1u,
                                          false, 10u);
    if (st != STATUS_SUCCESS) { return st; }

    st = LPI2C_DRV_MasterReceiveDataBlocking(INST_LPI2C0, rx, 2u,
                                             true, 10u);
    if (st != STATUS_SUCCESS) { return st; }

    *value = (uint16_t)(((uint16_t)rx[0] << 8) | rx[1]);
    return STATUS_SUCCESS;
}
```

The driver's status codes tell you *what kind* of failure occurred, and
each deserves different handling:

| Status | Meaning | Reasonable reaction |
|--------|---------|---------------------|
| `STATUS_SUCCESS` | Transfer complete | Use the data |
| `STATUS_BUSY` | Transfer still running | Poll again (non-blocking API) |
| `STATUS_TIMEOUT` | Deadline expired | Abort, count, retry with backoff |
| `STATUS_I2C_RECEIVED_NACK` | Slave did not acknowledge | Device absent or wrong address |
| `STATUS_I2C_ARBITRATION_LOST` | Another master won | Retry; on a single-master bus this means noise |
| `STATUS_I2C_BUS_BUSY` | SDA/SCL not idle | Suspect a hung slave — see below |

## Recovering a hung I²C bus

I²C's worst failure is unique among embedded buses: if the master is reset
mid-read, the slave can be left driving SDA low, waiting to finish
clocking out a byte. No amount of software re-initialization fixes it,
because SDA is stuck low and START cannot be generated. The standard
recovery is to **bit-bang up to nine SCL pulses** so the slave finishes
its byte, then issue a manual STOP:

```c
/* Mux SCL/SDA back to GPIO (module 4), then: */
void i2c_bus_recover(void)
{
    for (uint8_t i = 0u; i < 9u; i++) {
        PTA->PCOR = (1u << I2C_SCL_PIN);   delay_us(5u);
        PTA->PSOR = (1u << I2C_SCL_PIN);   delay_us(5u);
        if ((PTA->PDIR & (1u << I2C_SDA_PIN)) != 0u) {
            break;                          /* slave released SDA       */
        }
    }
    /* Manual STOP: SDA low → high while SCL is high */
    PTA->PCOR = (1u << I2C_SDA_PIN);  delay_us(5u);
    PTA->PSOR = (1u << I2C_SCL_PIN);  delay_us(5u);
    PTA->PSOR = (1u << I2C_SDA_PIN);  delay_us(5u);
    /* Re-mux to LPI2C and re-init the driver */
}
```

Every production I²C driver should have this function. If yours does not,
the first field return will teach you why.

## Automotive concerns

- **Timeout everything.** Never call a blocking transfer with an infinite
  timeout. An external device that stops responding must degrade your
  function, not hang your scheduler. Module 9's "timeout every external
  dependency" rule applies to on-board buses exactly as it does to CAN.
- **Never trust a single read of a safety-relevant value.** Read twice and
  compare, or use the device's own CRC/parity if it has one. SPI has *no*
  error detection on the wire — a corrupted MISO byte is indistinguishable
  from a real one.
- **DMA channels are shared.** `LPSPI_USING_DMA` needs `rxDMAChannel` and
  `txDMAChannel` fields, and those channels come out of module 2's
  allocation table. An SPI driver that silently claims channel 0 will
  collide with the ADC scan of a colleague's module.
- **Bus length and EMC.** SPI at 10 MHz across a long PCB run radiates and
  fails EMC testing. Automotive designs frequently run SPI slower than the
  silicon allows for exactly this reason — slow and passing beats fast and
  re-spun.
- **Keep the SBC watchdog out of interrupt context.** Like the internal
  WDOG, trigger it from supervised application code. An SPI watchdog
  serviced from a timer ISR supervises the timer, nothing more.
- **Pull-up sizing is a system property.** Weak pull-ups plus long traces
  round off I²C edges until 400 kHz stops working over temperature. If
  I²C is intermittently unreliable, look at the scope before the code.

## Cheat sheet

| Item | LPSPI | LPI2C |
|------|-------|-------|
| Wires | SCK, SOUT, SIN, PCS0–3 | SDA, SCL (open drain + pull-ups) |
| Addressing | One chip select per device | 7-bit (or 10-bit) address on the wire |
| Duplex | Full — every TX byte returns an RX byte | Half — direction set per transfer |
| Typical rate | 1–10 MHz | 100 kHz / 400 kHz / 1 MHz |
| Ack / errors | None on the wire | Per-byte ACK/NACK, arbitration detection |
| Init | `LPSPI_DRV_MasterInit` | `LPI2C_DRV_MasterInit` |
| Blocking transfer | `LPSPI_DRV_MasterTransferBlocking` | `..._MasterSendDataBlocking` / `..._MasterReceiveDataBlocking` |
| Repeated start | n/a — use `isPcsContinuous` | `sendStop = false` on the write phase |
| DMA | `LPSPI_USING_DMA` + rx/tx channel fields | `LPI2C_USING_DMA` + `dmaChannel` |
| Signature failure | Wrong CPOL/CPHA → bit-shifted garbage | Slave holds SDA low → bus hang |
| Recovery | Re-init, re-transfer | Nine SCL pulses + manual STOP |
| Typical devices | SBC, external EEPROM, gate drivers | Temperature/pressure sensors, EEPROMs |

## Exercise

Write a defensive driver for one external device of your choosing and
prove it survives abuse. (1) Implement `dev_read_reg()` and
`dev_write_reg()` over LPSPI or LPI2C with a bounded timeout, returning
`status_t` — no `void` returns, no silent failures. (2) Add a
**consistency check**: read a known constant register (most sensors have a
device-ID register) at init, and refuse to mark the device available if it
does not match — this single check catches wrong addresses, wrong SPI
mode, and unpopulated parts. (3) Add fault handling: three consecutive
failures set a DTC, mark the reading invalid, and substitute a safe
default; a later success clears the fault after five good reads. (4) Test
it by physically disconnecting the device mid-operation, and confirm your
10 ms task still meets its deadline while the device is gone. (5) For I²C
only: pull SDA low with a jumper to a ground pin to simulate a hung slave,
and confirm your recovery routine restores the bus. If you do not have the
hardware, implement all of the above against a stub layer on your PC that
returns scripted status codes, including `STATUS_I2C_RECEIVED_NACK` and
`STATUS_TIMEOUT` — the fault logic is the part being graded.
