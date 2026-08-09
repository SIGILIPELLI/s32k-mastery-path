# FlexCAN Deep Dive — Mailboxes & Filtering

Level 1's module 7 gave you the CAN protocol and a working TX/RX pair.
That is enough for a demo node and nowhere near enough for an ECU sharing
a 500 kbit/s bus with twenty other modules. Production firmware answers
harder questions: *which* mailbox catches *which* frames, what happens
when frames arrive faster than the CPU drains them, and what the node does
when the bus turns hostile. This module walks the FlexCAN peripheral
mailbox-by-mailbox and register-by-register, then wires the error path
that keeps a faulty node from taking the whole bus down with it.

## The message buffer RAM

FlexCAN's core is a block of RAM organized as **message buffers (MBs)**.
On the S32K144, **CAN0 has 32 MBs; CAN1 and CAN2 have 16** — and the count
depends on payload size, because the RAM is fixed. With 8-byte payloads an
MB costs 16 bytes; switch to CAN FD 64-byte payloads and one MB costs 72
bytes, so the same RAM holds far fewer. Choosing `payload` in the driver
config is a *budgeting* decision, not a cosmetic one.

Each MB is four-plus words:

```text
word 0 : CS      — CODE, SRR, IDE, RTR, DLC, TIME STAMP
word 1 : ID      — 11-bit standard ID (bits 28:18) or 29-bit extended
word 2+: DATA    — payload, big-endian (data[0] is first byte on the wire)
```

The **CODE** field in CS is the whole state machine. The values you will
actually meet:

| CODE | Direction | Meaning |
|------|-----------|---------|
| `0b0000` | RX | INACTIVE — MB is not participating |
| `0b0100` | RX | EMPTY — armed and waiting for a match |
| `0b0010` | RX | FULL — a frame landed; read it and re-arm |
| `0b0110` | RX | OVERRUN — a second frame arrived before you read the first |
| `0b1010` | RX | RANSWER — auto-answer a remote frame (rarely used) |
| `0b1000` | TX | INACTIVE — MB is free for a new transmission |
| `0b1100` | TX | DATA — transmit this frame once the bus is free |
| `0b1001` | TX | ABORT — cancel a pending transmission |

Two hardware rules bite people who go straight to registers:

- **Reading CS locks the MB.** Hardware will not overwrite a locked MB.
  You unlock it by reading the free-running **TIMER** register, or by
  locking a different MB. Forget the unlock and that mailbox quietly stops
  receiving. `FLEXCAN_DRV_Receive` does this for you.
- **Writing CODE last commits the MB.** Fill ID and data first, then write
  CS — otherwise the engine can transmit a half-built frame.

## Filtering: masks that actually match

An RX mailbox matches an incoming ID through a mask, and the convention is
the one people get backwards:

```text
mask bit = 1  →  this ID bit MUST match the mailbox's ID
mask bit = 0  →  "don't care" — accept either value
```

FlexCAN offers several mask scopes, selected by configuration:

| Register | SDK call | Scope |
|----------|----------|-------|
| `RXMGMASK` | `FLEXCAN_DRV_SetRxMbGlobalMask` | One mask for all RX MBs |
| `RX14MASK` / `RX15MASK` | — | Private masks for MB14 and MB15 |
| `RXIMR[n]` | `FLEXCAN_DRV_SetRxIndividualMask` | One mask per mailbox |
| `RXFGMASK` | `FLEXCAN_DRV_SetRxFifoGlobalMask` | Mask for RX FIFO filter elements |

Individual masks require `MCR[IRMQ]`; the driver sets it when you select
the mask type. A worked example — one mailbox accepting the whole `0x31x`
block of our Level 1 node family:

```c
#include "flexcan_driver.h"

#define INST_CAN0   0u
#define MB_CMD_RX   1u

void can_filters_init(void)
{
    /* Per-mailbox masks instead of one global mask */
    FLEXCAN_DRV_SetRxMaskType(INST_CAN0, FLEXCAN_RX_MASK_INDIVIDUAL);

    /* Standard IDs sit in bits 28:18 of the mask word.
       0x7F0 << 18 => match the top 7 bits, ignore the low 4 =>
       IDs 0x310..0x31F all land in this mailbox.               */
    FLEXCAN_DRV_SetRxIndividualMask(INST_CAN0, FLEXCAN_MSG_ID_STD,
                                    MB_CMD_RX, (0x7F0u << 18));

    flexcan_data_info_t rxInfo = {
        .msg_id_type = FLEXCAN_MSG_ID_STD,
        .data_length = 8u,
        .is_remote   = false,
    };
    FLEXCAN_DRV_ConfigRxMb(INST_CAN0, MB_CMD_RX, &rxInfo, 0x310u);
}
```

!!! warning "The shift is not optional"
    Standard IDs live in the *upper* bits of the ID word. A mask written
    as plain `0x7F0` masks nothing useful and you will receive either
    everything or nothing. Write `(mask << 18)` for 11-bit IDs; extended
    29-bit IDs use the word unshifted.

## RX FIFO: when frames outrun the CPU

A mailbox holds exactly one frame. If your node consumes ten different IDs
arriving in bursts, dedicating ten mailboxes and servicing ten interrupts
is wasteful. The **RX FIFO** is the answer: a six-deep hardware queue
occupying the MB0–MB5 region, fed by an **ID filter table** starting at
MB6 and growing with `CTRL2[RFFN]` (8, 16, … up to 128 elements — each
extra step consumes more mailbox RAM).

Filter element formats:

| Format | Elements per word | Matches |
|--------|-------------------|---------|
| A | 1 | One full standard or extended ID + IDE/RTR bits |
| B | 2 | Two partial IDs (full 11-bit standard, or upper bits of extended) |
| C | 4 | Four 8-bit ID slices — coarse, mask-like |
| D | — | Reject all |

```c
static flexcan_state_t    canState;
static flexcan_id_table_t rxFifoFilters[8] = {
    { .isRemoteFrame = false, .isExtendedFrame = false, .id = 0x310u },
    { .isRemoteFrame = false, .isExtendedFrame = false, .id = 0x311u },
    { .isRemoteFrame = false, .isExtendedFrame = false, .id = 0x320u },
    { .isRemoteFrame = false, .isExtendedFrame = false, .id = 0x321u },
    /* Unused slots: repeat a used ID rather than leaving 0x000 — an
       accidental 0x000 filter accepts the highest-priority ID on the
       bus and floods your FIFO.                                     */
    { false, false, 0x321u }, { false, false, 0x321u },
    { false, false, 0x321u }, { false, false, 0x321u },
};

void can_fifo_init(void)
{
    flexcan_user_config_t cfg;
    FLEXCAN_DRV_GetDefaultConfig(&cfg);
    cfg.max_num_mb        = 16u;
    cfg.is_rx_fifo_needed = true;
    cfg.num_id_filters    = FLEXCAN_RX_FIFO_ID_FILTERS_8;
    cfg.flexcanMode       = FLEXCAN_NORMAL_MODE;
    cfg.payload           = FLEXCAN_PAYLOAD_SIZE_8;

    FLEXCAN_DRV_Init(INST_CAN0, &canState, &cfg);
    FLEXCAN_DRV_ConfigRxFifo(INST_CAN0, FLEXCAN_RX_FIFO_ID_FORMAT_A,
                             rxFifoFilters);
    FLEXCAN_DRV_SetRxFifoGlobalMask(INST_CAN0, FLEXCAN_MSG_ID_STD,
                                    (0x7FFu << 18));   /* exact match */
}
```

The FIFO raises three separate flags in `IFLAG1`: **bit 5** frames
available, **bit 6** FIFO warning (nearly full), **bit 7** overflow — a
frame was *lost*. Treat bit 7 as a defect, not a status: it means software
missed a deadline. Count it, export it, and treat it as evidence during
timing analysis.

## Interrupt-driven handling

Polling `FLEXCAN_DRV_GetTransferStatus` is fine for a 100 ms status frame
and hopeless on a busy bus. Install a callback:

```c
static volatile uint32_t g_rxFifoOverflows;
static flexcan_msgbuff_t g_rxMsg;

static void can_event_cb(uint8_t instance, flexcan_event_type_t eventType,
                         uint32_t buffIdx, flexcan_state_t *state)
{
    (void)state;
    switch (eventType) {
    case FLEXCAN_EVENT_RXFIFO_COMPLETE:
        app_queue_frame(&g_rxMsg);                    /* copy out fast   */
        FLEXCAN_DRV_RxFifo(instance, &g_rxMsg);       /* re-arm the FIFO */
        break;
    case FLEXCAN_EVENT_RXFIFO_OVERFLOW:
        g_rxFifoOverflows++;                          /* a frame was lost */
        break;
    case FLEXCAN_EVENT_RX_COMPLETE:
        app_mailbox_ready(buffIdx);
        break;
    case FLEXCAN_EVENT_TX_COMPLETE:
        app_tx_done(buffIdx);
        break;
    default:
        break;                                        /* nothing to do   */
    }
}

void can_irq_init(void)
{
    FLEXCAN_DRV_InstallEventCallback(INST_CAN0, can_event_cb, NULL);
    FLEXCAN_DRV_RxFifo(INST_CAN0, &g_rxMsg);          /* first arm       */
}
```

The callback runs in interrupt context. Do the minimum: copy the frame
into an application queue, set a flag, return. Parsing signals, running
control logic, or anything blocking belongs in the task that consumes the
queue — module 4's FreeRTOS lesson makes that split explicit.

## Error counters and bus-off

Every CAN node maintains two counters in `ECR`: **TXERRCNT** and
**RXERRCNT**. Transmit errors cost 8, receive errors cost 1, successes
decrement. The thresholds are protocol-defined, not vendor-specific:

| State | Condition | Behaviour |
|-------|-----------|-----------|
| Error active | both counters < 128 | Sends *active* (dominant) error flags |
| Error passive | either counter ≥ 128 | Sends *passive* (recessive) error flags — cannot destroy other nodes' frames |
| Bus off | TXERRCNT > 255 | Node disconnects itself from the bus |

`ESR1[FLTCONF]` reports the current state and `ESR1[BOFFINT]` fires on the
transition. Recovery from bus-off requires **128 occurrences of 11
consecutive recessive bits** — about 1408 bit times, roughly 2.8 ms of
quiet bus at 500 kbit/s. `MCR[BOFFREC]` controls whether that happens
automatically: **cleared = automatic recovery enabled**, set = software
must intervene. The double negative catches everyone once.

```c
static volatile uint32_t g_busOffCount;

static void can_error_cb(uint8_t instance, flexcan_event_type_t eventType,
                         flexcan_state_t *state)
{
    (void)state;
    if (eventType == FLEXCAN_EVENT_ERROR) {
        uint32_t esr1 = FLEXCAN_DRV_GetErrorStatus(instance);
        if ((esr1 & CAN_ESR1_BOFFINT_MASK) != 0u) {
            g_busOffCount++;
            app_set_dtc(DTC_CAN_BUS_OFF);      /* module 9: report it  */
            app_enter_can_safe_state();        /* consumers time out   */
        }
    }
}
```

The practice around bus-off is worth spelling out, because it appears in
most OEM software specifications:

- **Never spin waiting for recovery.** The rest of the ECU must keep
  running its safe-state logic while CAN is down.
- **Rate-limit re-initialization.** A node that resets its CAN controller
  in a tight loop against a shorted bus becomes exactly the babbling idiot
  the protocol was designed to contain. Back off progressively — 10 ms,
  100 ms, 1 s — then latch a fault.
- **Distinguish "bus is broken" from "I am broken."** Persistent TX errors
  alongside a healthy RX counter point at your own transceiver, stub, or
  termination, not at the network.
- **Abort, do not orphan.** If a periodic frame is superseded before it
  ever wins arbitration, `FLEXCAN_DRV_AbortTransfer` (requires `MCR[AEN]`)
  cancels it so the bus never carries stale data.
- **Disable self-reception** (`MCR[SRXDIS]`) unless you genuinely want to
  hear your own frames — otherwise every transmission also consumes an RX
  slot and confuses timeout logic.

## Cheat sheet

| Item | Notes |
|------|-------|
| MB count | S32K144: CAN0 = 32 MBs, CAN1/CAN2 = 16 — fewer with 64-byte FD payloads |
| MB layout | CS (CODE/IDE/RTR/DLC) · ID · DATA; write CODE **last** |
| RX CODEs | EMPTY `0b0100` → FULL `0b0010` → OVERRUN `0b0110` if not drained |
| Lock rule | Reading CS locks the MB; read TIMER to unlock |
| Mask polarity | 1 = must match, 0 = don't care |
| Standard IDs | Shift by 18 in ID and mask words (`0x7FF << 18`) |
| Mask scopes | `RXMGMASK` global · `RXIMR[n]` individual (`MCR[IRMQ]`) · `RXFGMASK` FIFO |
| RX FIFO | 6 deep, occupies MB0–5, filter table from MB6, size via `CTRL2[RFFN]` |
| FIFO flags | `IFLAG1` bit 5 available · bit 6 warning · bit 7 **overflow = frame lost** |
| Error counters | `ECR` TXERRCNT/RXERRCNT; ≥128 error-passive, TX >255 bus-off |
| Bus-off recovery | 128 × 11 recessive bits; `MCR[BOFFREC]` **cleared** = automatic |
| Callback discipline | Copy and return — no parsing, no blocking, no logging in the ISR |

## Exercise

Extend your Level 1 capstone node to use the RX FIFO instead of dedicated
mailboxes, and give it a real error path. Specifically: (1) configure an
eight-element format-A filter table for four IDs your node consumes, and
prove with `candump` that a fifth, unlisted ID never reaches your
dispatcher; (2) deliberately leave one filter entry as `0x000` on a busy
bus, observe what floods in, then fix it — that is a mistake worth making
once, cheaply; (3) instrument `FLEXCAN_EVENT_RXFIFO_OVERFLOW` with a
counter exported in your status frame, then cause an overflow on purpose
by stalling your dispatcher for 5 ms during a burst; (4) implement bus-off
handling with the three-strikes backoff above, and test it by disconnecting
the second node's termination. Finally, write one paragraph explaining how
your design distinguishes a broken bus from a broken node — that paragraph
is the part a reviewer will actually read.
