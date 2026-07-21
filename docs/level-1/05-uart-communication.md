# 05 · UART Communication

Before an ECU can talk to the car, it needs to talk to *you*. A UART serial
console is the firmware engineer's stethoscope: boot messages, sensor
readings, error dumps — during bring-up of a new board, `printf` over UART
is often the only window into a live system, and in the lab it stays
invaluable long after CAN is up. This module covers the S32K's **LPUART**
peripheral, baud-rate math, framing, and a polled "hello ECU" you can
extend for the rest of the course.

## UART in 60 seconds

UART (Universal Asynchronous Receiver/Transmitter) sends bytes serially
with no shared clock — both sides just agree on a **baud rate** in advance.
A classic **8N1** frame (8 data bits, no parity, 1 stop bit) on the wire:

```text
idle(high) │ start(0) │ b0 b1 b2 b3 b4 b5 b6 b7 │ stop(1) │ idle...
             1 bit      8 data bits (LSB first)    1 bit
```

10 bit-times per byte — so at 115200 baud you move at most 11 520 bytes/s.
The receiver detects the falling start edge, then samples the middle of
each bit period. Because each side times bits from its own clock, total
clock mismatch must stay under roughly ±2–3% per frame — which is why baud
math (below) matters, and one reason CAN (module 7) uses continuous
resynchronization instead of trusting frame-length timing alone.

On the S32K144EVB, **LPUART1** (pins **PTC6 = RX, PTC7 = TX**, mux ALT2) is
routed through the OpenSDA debug chip, which presents it to your PC as a
normal USB-serial port — connect at 115200-8N1 with any terminal (PuTTY,
`screen`, S32DS terminal view).

## Baud-rate math

LPUART divides its **functional clock** (chosen in the PCC, module 3) down
to the bit rate using two fields in the `BAUD` register: **SBR** (a 13-bit
divider) and **OSR** (oversampling ratio, commonly 16×):

```text
baud = functional_clock / ((OSR + 1) × SBR)
```

With the functional clock on FIRCDIV2 at 48 MHz, OSR+1 = 16:

| Target baud | SBR = 48 MHz / (16 × baud) | Actual baud | Error |
|-------------|-----------------------------|-------------|-------|
| 9 600 | 312.5 → 313 | 9 585 | −0.16 % |
| 115 200 | 26.04 → 26 | 115 385 | +0.16 % |
| 921 600 | 3.26 → 3 | 1 000 000 | **+8.5 % — broken!** |

That last row is the lesson: SBR must be an integer, so not every baud rate
is reachable from every clock — always compute the *actual* rate and check
the error is well under ~2%. (For very high rates you'd lower OSR or pick a
different functional clock.)

## Polled "hello ECU" (register-level)

```c
#include "S32K144.h"

void lpuart1_init(void)          /* 115200-8N1 from 48 MHz FIRCDIV2 */
{
    /* Clocks: PORT C for pins, LPUART1 with functional clock = FIRCDIV2 */
    PCC->PCCn[PCC_PORTC_INDEX]  |= PCC_PCCn_CGC_MASK;
    PCC->PCCn[PCC_LPUART1_INDEX] = PCC_PCCn_PCS(3)      /* FIRCDIV2 */
                                 | PCC_PCCn_CGC_MASK;

    /* Pins: PTC6 = LPUART1_RX, PTC7 = LPUART1_TX (ALT2) */
    PORTC->PCR[6] = PORT_PCR_MUX(2);
    PORTC->PCR[7] = PORT_PCR_MUX(2);

    /* Baud: OSR = 15 (16x), SBR = 26  → 115385 baud (+0.16%) */
    LPUART1->BAUD = LPUART_BAUD_OSR(15) | LPUART_BAUD_SBR(26);

    /* 8N1 is the reset default of CTRL; just enable TX and RX */
    LPUART1->CTRL = LPUART_CTRL_TE_MASK | LPUART_CTRL_RE_MASK;
}

void uart_putc(char c)
{
    /* Wait until TX Data register Empty flag says there's room */
    while ((LPUART1->STAT & LPUART_STAT_TDRE_MASK) == 0u) { }
    LPUART1->DATA = (uint8_t)c;
}

void uart_puts(const char *s)
{
    while (*s != '\0') {
        if (*s == '\n') uart_putc('\r');   /* terminals like CRLF */
        uart_putc(*s++);
    }
}

int main(void)
{
    clocks_init();          /* module 3 */
    lpuart1_init();
    uart_puts("hello ECU\n");
    for (;;) { }
}
```

Receiving, polled — echo everything back:

```c
for (;;) {
    if ((LPUART1->STAT & LPUART_STAT_RDRF_MASK) != 0u) {  /* byte arrived? */
        char c = (char)LPUART1->DATA;                     /* read clears flag */
        uart_putc(c);                                     /* echo */
    }
    /* ...other non-blocking work continues here... */
}
```

The two flags to memorize: **TDRE** (transmit data register empty — safe to
write `DATA`) and **RDRF** (receive data register full — a byte is waiting).
Polled I/O like this is fine for bring-up and low traffic; the moment RX
must never miss bytes while the CPU is busy, you graduate to interrupts
(module 8's mindset) or DMA (Level 2).

!!! note "SDK-style equivalent"
    ```c
    lpuart_state_t uartState;
    const lpuart_user_config_t uartCfg = {
        .baudRate = 115200u, .parityMode = LPUART_PARITY_DISABLED,
        .stopBitCount = LPUART_ONE_STOP_BIT,
        .bitCountPerChar = LPUART_8_BITS_PER_CHAR,
        .transferType = LPUART_USING_INTERRUPTS,
    };
    LPUART_DRV_Init(1u, &uartState, &uartCfg);
    LPUART_DRV_SendDataPolling(1u, (const uint8_t *)"hello ECU\r\n", 11u);
    ```
    Same registers underneath; the driver additionally computes SBR/OSR for
    you from the configured clock — which only works if the clock manager
    knows the true functional clock frequency. Wrong clock config ⇒ wrong
    baud ⇒ garbage characters: the classic symptom chain.

## Why UART matters in real ECU work

- **Bring-up.** A new board's first milestone is always "prints over UART" —
  it proves clocks, pin mux, flash, and startup all work before more complex
  peripherals exist.
- **Debug channel that survives.** Production ECUs often keep a UART header
  (unpopulated) on the PCB for factory tests and field returns analysis.
- **It's the transport under other things.** LIN (Level 3) is essentially a
  UART with rules; many GPS modules, Bluetooth modules, and modem AT
  interfaces are UARTs.
- **But it's not CAN.** UART is point-to-point, unaddressed, unacknowledged,
  and error-unchecked. The differences are exactly what module 7 is about —
  after this module, you'll appreciate *why* CAN's extra machinery exists.

If garbage appears on your terminal: 1) baud mismatch (check actual-rate
math and the true functional clock), 2) wrong pin mux, 3) crossed TX/RX,
4) missing common ground — in that order of likelihood.

## Cheat sheet

| Item | Notes |
|------|-------|
| Frame 8N1 | start(0) + 8 data LSB-first + stop(1) = 10 bit-times per byte |
| Baud formula | `baud = func_clk / ((OSR+1) × SBR)` — compute actual, keep error < ~2% |
| EVB console | LPUART1, PTC6 RX / PTC7 TX (ALT2), via OpenSDA USB → 115200-8N1 |
| PCC for LPUART | Needs PCS (functional clock, e.g. FIRCDIV2) **and** CGC |
| TDRE flag | TX register empty → OK to write `DATA` |
| RDRF flag | RX register full → read `DATA` (read clears it) |
| Enable | `CTRL = TE | RE` after BAUD is set |
| SDK calls | `LPUART_DRV_Init`, `LPUART_DRV_SendDataPolling`, `LPUART_DRV_ReceiveDataPolling` |
| Garbage output | Baud/clock mismatch first, then mux, then wiring |

## Exercise

Build a tiny command console: your firmware prints a `> ` prompt, reads a
line (polled RX, echo as you type, handle backspace by printing `"\b \b"`),
and executes three commands — `led on`, `led off` (module 4's red LED) and
`id`, which prints a device name and firmware version string. Structure it
as a `console_poll()` function called from the main loop that never blocks
(it processes at most one received character per call and keeps its own
line buffer + state). Bonus (paper exercise if no hardware): at 115200 baud,
how many *milliseconds* does echoing a 40-character line cost in pure wire
time, and why does a non-blocking design make that cost irrelevant to the
rest of the loop?
