# LIN & FlexIO

Not every signal in a vehicle justifies a CAN transceiver, a termination
resistor pair, and a dedicated controller. A window switch, a mirror
motor, a seat position sensor: single master, a handful of slaves, low
bandwidth, cost matters more than throughput. **LIN** (Local
Interconnect Network) is the answer — a single-wire, master/slave,
UART-based protocol running at up to 20 kbit/s, and it is everywhere in
body electronics. The S32K doesn't have a dedicated LIN peripheral; LIN
runs over **LPUART** with a LIN transceiver IC handling the single-wire
electrical layer, and where you need more LIN channels than LPUART
instances, or a completely custom bit-level protocol, the S32K's
**FlexIO** peripheral steps in as a software-configurable I/O engine.

## LIN frame anatomy

```text
Break (≥13 bits dominant) | Sync (0x55) | PID (ID+parity, 1 byte) | Data (0-8 bytes) | Checksum
```

The master sends **Break + Sync + PID** (the "header"); either the master
or the addressed slave then sends the **response** (data + checksum). No
arbitration exists — LIN has exactly one master, so there is nothing to
arbitrate. The **PID** packs a 6-bit frame ID with 2 parity bits computed
from specific ID bits, catching header corruption before a slave
misidentifies which frame it's supposed to answer.

```c
/* LIN header transmission over LPUART, S32K-style */
static uint8_t lin_pid_calc(uint8_t id)
{
    uint8_t p0 = ((id>>0)^(id>>1)^(id>>2)^(id>>4)) & 0x01u;
    uint8_t p1 = (~((id>>1)^(id>>3)^(id>>4)^(id>>5))) & 0x01u;
    return (uint8_t)((id & 0x3Fu) | (p0 << 6) | (p1 << 7));
}

void Lin_SendHeader(LPUART_Type *base, uint8_t frame_id)
{
    /* Break: LPUART configured for a longer-than-normal low period.
       Many S32K LIN drivers toggle the pin as GPIO to force >=13 bit
       times dominant, then hand control back to LPUART for Sync/PID. */
    Lin_SendBreak(base, 13u);
    LPUART_WriteByte(base, 0x55u);                 /* Sync */
    LPUART_WriteByte(base, lin_pid_calc(frame_id)); /* PID */
}
```

**Classic checksum** covers only the data bytes; **enhanced checksum**
(LIN 2.x, the version used on essentially all modern designs) also
includes the PID. Mixing the two on the same network — a LIN 1.3 slave
answering a LIN 2.1 master's enhanced-checksum expectation — produces a
checksum that looks corrupted every single frame, which is a common
integration bug when reusing an old slave node design.

## FlexIO as a software UART/LIN engine

FlexIO is a small array of shifters and timers you configure to emulate
serial protocols in hardware without a dedicated peripheral — useful when
you've used up every LPUART instance, or need a protocol S32K silicon
doesn't natively support:

```c
/* FlexIO configured as an additional LIN-capable UART channel.
   Conceptually: one shifter for TX, one for RX, one timer sets baud. */
typedef struct {
    uint8_t  shifter_tx;
    uint8_t  shifter_rx;
    uint8_t  timer;
    uint32_t baud_rate;
} flexio_uart_cfg_t;

void FlexIO_UART_Init(FLEXIO_Type *base, const flexio_uart_cfg_t *cfg)
{
    /* TIMCMP sets baud via FlexIO clock / (2 * baud) - 1, matching the
       same divide-down math as an LPUART baud generator */
    base->TIMCMP[cfg->timer] = (FLEXIO_CLOCK_HZ / (2u * cfg->baud_rate)) - 1u;
    base->TIMCTL[cfg->timer] = FLEXIO_TIMCTL_TIMOD(0x1u)   /* dual 8-bit baud/bit timer */
                             | FLEXIO_TIMCTL_PINSEL(cfg->shifter_tx);
    base->SHIFTCTL[cfg->shifter_tx] = FLEXIO_SHIFTCTL_TIMSEL(cfg->timer)
                                     | FLEXIO_SHIFTCTL_PINCFG(0x3u); /* pin = output */
    base->SHIFTCFG[cfg->shifter_tx] = FLEXIO_SHIFTCFG_SSTOP(0x2u)   /* stop bit = 1 */
                                     | FLEXIO_SHIFTCFG_SSTART(0x1u); /* start bit = 0 */
}
```

FlexIO's value isn't limited to LIN: the same shifter/timer building
blocks emulate SPI, I2C, or fully custom bit patterns, which is why it
shows up again in Level 4 test-fixture and manufacturing contexts where
an ECU needs one more protocol than its fixed peripherals provide.

## Automotive-MCU concerns

- **The Break field's minimum length is a real bus timing requirement,
  not a suggestion.** LIN spec requires ≥13 nominal bit times dominant;
  sending a marginally short break works with your own slave (which may
  tolerate it) and fails with a third-party slave built to spec, because
  its break-detect threshold is tighter. Always generate Break with
  margin, and verify on a scope during bring-up — this is a frequent
  interop failure between in-house and supplier LIN nodes.
- **LIN has no bus arbitration, so a runaway slave is a network-wide
  failure.** A slave that responds to a header it wasn't addressed by
  (a PID decode bug) collides with the correct responder's data on the
  wire; unlike CAN, there is no dominant-bit arbitration to resolve
  it — the frame is simply corrupted. Validate PID decode logic against
  the full 64-value PID table, not just your own node's IDs.
- **FlexIO timer/shifter resources are shared and limited.** Every FlexIO
  channel you allocate to LIN emulation is a shifter and timer pair
  unavailable to another use (e.g. a PWM capture elsewhere in the
  project). Check the S32K3 reference manual's FlexIO resource count for
  your specific part before committing a design to "just add another
  FlexIO LIN channel."
- **LIN slaves are frequently the least-tested node on a bus.** Because
  bring-up effort concentrates on the CAN gateway and body controller,
  a cheap window-lift LIN slave's firmware often ships with only the
  vendor's default checksum mode validated. Confirm LIN version
  (1.3 vs 2.x checksum) and diagnostic frame support (`0x3C`/`0x3D`
  master/slave request-response, the LIN equivalent of UDS) explicitly
  during integration, not by assumption.

## Cheat sheet

| LIN field | Size | Notes |
|-----------|------|-------|
| Break | ≥13 bit times dominant | Frames the header start |
| Sync | 1 byte, always `0x55` | Lets slaves measure the bit rate |
| PID | 1 byte | 6-bit ID + 2 parity bits (`lin_pid_calc`) |
| Data | 0-8 bytes | Sent by master or the addressed slave |
| Checksum | 1 byte | Classic = data only; Enhanced (2.x) = PID + data |

| Concept | Where |
|---------|-------|
| LIN electrical layer | External LIN transceiver (e.g. TJA1027), single wire + ground |
| LIN logical layer | LPUART peripheral, Break sent via GPIO toggle or LPUART break feature |
| Diagnostic frames | `0x3C` master request, `0x3D` slave response (LIN-specific, not UDS) |
| FlexIO use here | Extra software-defined UART/LIN channel beyond fixed LPUART count |
| FlexIO building blocks | Shifters (data path) + Timers (baud/bit timing) |

## Exercise

Bring up a LIN master/slave pair on two S32K boards, or a master against
a bench LIN slave IC if available. (1) Implement `lin_pid_calc` and
verify it against the LIN spec's full PID table for at least 5 IDs by
hand before trusting it on the bus. (2) Send a header with an
intentionally short break (8 bit times) and confirm on a scope or logic
analyzer whether your slave still detects it — document the margin your
implementation actually has versus the ≥13-bit spec minimum. (3)
Implement both classic and enhanced checksum modes and demonstrate a
mismatched-mode fault: a classic-checksum slave answering an enhanced-
checksum master, and show the byte-level difference in what each side
computes. (4) If FlexIO peripherals are available on your board, bring
up one FlexIO-emulated UART channel at a fixed baud rate and confirm
byte-accurate transmission against an LPUART-based receiver before
attempting the full LIN protocol on it.
