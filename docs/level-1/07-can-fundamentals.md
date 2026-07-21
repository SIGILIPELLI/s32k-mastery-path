# 07 · CAN Fundamentals

If one module in this course is "the automotive one," this is it. **CAN**
(Controller Area Network), invented by Bosch in the 1980s, is still the
backbone of in-vehicle networking: robust, cheap, deterministic enough,
and standardized into every automotive MCU on the market. Understanding
frames, arbitration, and acknowledgment is a job-interview staple and the
foundation for everything from module 10's capstone to UDS diagnostics and
AUTOSAR COM in later levels. The S32K's CAN peripheral is **FlexCAN** —
we'll cover it conceptually and with a minimal SDK-style TX/RX example.

## Why a bus, and why CAN won

Point-to-point wiring between every ECU pair would need a harness thicker
than your arm. CAN replaces it with **one twisted pair shared by all
nodes**: CAN_H and CAN_L, differentially driven (noise hits both wires
equally and cancels), terminated with 120 Ω at both ends. Every node sees
every frame; there are **no addresses** — frames carry a message **ID**
that says *what the data is* ("engine RPM"), not who it's for. Any
interested node just listens for the IDs it cares about.

The electrical trick that makes everything work: the bus has a **dominant**
state (logical 0, actively driven) and a **recessive** state (logical 1,
passive). If any node transmits dominant while another transmits recessive,
**dominant wins on the wire**. Remember this — arbitration and ACK both
fall out of it.

Classic CAN runs up to **1 Mbit/s** (500 kbit/s is the most common vehicle
rate) with up to **8 data bytes** per frame.

## The frame

A classic CAN data frame, simplified:

```text
| SOF | ID (11 or 29 bit) + RTR | DLC | DATA (0..8 bytes) | CRC | ACK | EOF |
```

| Field | Meaning |
|-------|---------|
| ID | Message identifier — also the **priority** (lower ID = higher priority) |
| RTR | Remote frame flag (requests data — rarely used in modern designs) |
| DLC | Data length code: how many data bytes (0–8) |
| DATA | The payload — signals packed into bits/bytes (see below) |
| CRC | 15-bit checksum computed by hardware over the frame |
| ACK | One bit slot where **every receiver that validated the CRC drives dominant** |
| EOF | End of frame; then 3 bits of interframe space |

Standard IDs are 11-bit (0x000–0x7FF); extended frames carry 29-bit IDs —
both coexist on one bus. Note what ACK means and doesn't mean: a dominant
ACK tells the transmitter *someone* received the frame intact — not that
the intended consumer did. Higher-layer protocols add end-to-end guarantees
where needed.

## Arbitration: collision resolution with zero wasted bits

Any node may start transmitting when the bus is idle. If two start
simultaneously, both send their ID bit-by-bit **while reading the bus
back**:

```text
Node A sends ID 0x24C:  0 1 0 0 1 0 0 1 1 0 0
Node B sends ID 0x24F:  0 1 0 0 1 0 0 1 1 1 …
                                          ^
        B sent recessive(1), reads dominant(0) → B lost, stops, retries later
        A never noticed anything: its frame continues undamaged
```

The lower ID wins every collision, losing nodes retry automatically, and no
bus time is ever destroyed. This is **CSMA/CR** — collision *resolution*,
not Ethernet-style collision destruction — and it's why CAN IDs are
assigned by system engineers with care: **ID = priority**. Brake messages
get low IDs; seat-heater status does not. Consequence worth knowing: under
heavy load, low-priority messages can starve — network designers keep bus
load under ~50–70% partly for this reason.

Error handling is equally clever: every node checks every frame (CRC, form,
stuffing), any node that spots an error transmits an **error flag** that
destroys the frame for everyone, and per-node error counters demote
consistently-faulty nodes (error-passive, then **bus-off** — a babbling
node removes *itself* from the bus). Fault containment is built into the
protocol — one of the clearest examples of automotive design philosophy
you'll ever see.

## Signals: DBC-style thinking

Nobody designs "8 raw bytes." Payloads are defined as packed **signals** in
a database file (a **.dbc**), which tools and code generators consume. A
typical definition for our capstone's frame, in DBC-like notation:

```text
BO_ 0x310 SENSOR_NODE_STATUS: 8 SensorNode
 SG_ CoolantTemp   : 0|16@1- (0.1, 0)  [-40|150]  "degC"   BodyECU
 SG_ SupplyVoltage : 16|16@1+ (0.001,0) [0|20]    "V"      BodyECU
 SG_ SensorValid   : 32|1@1+  (1,0)     [0|1]     ""       BodyECU
 SG_ AliveCounter  : 40|4@1+  (1,0)     [0|15]    ""       BodyECU
```

Read: frame 0x310, 8 bytes; a 16-bit signed temperature starting at bit 0,
scale 0.1 °C/bit; a voltage in mV; a validity flag; an alive counter that
increments each transmission so receivers can detect a frozen sender.
`raw = (physical − offset) / scale` — the same scaling thinking as module
6, now applied to the wire. Level 3 covers real DBC workflows; from today,
*think in signals, not bytes*.

## FlexCAN on the S32K

The S32K144 has up to three FlexCAN instances; **CAN0 is wired through a
transceiver to a connector on the EVB** — with a second node and two
termination resistors, it's a real bus. The concepts:

- **Message buffers (mailboxes):** FlexCAN provides 16–32 RAM slots, each
  configurable as TX or RX. An RX mailbox is programmed with an ID (plus a
  mask, so one mailbox can accept a *range* of IDs); matching frames land
  in it and set an interrupt flag. TX mailboxes take your frame and handle
  arbitration/retransmission autonomously.
- **Bit timing:** the nominal bit is divided into time quanta (sync,
  propagation, phase segments); registers set the prescaler and segment
  lengths, and the **sample point** (~75–87.5% into the bit) must match the
  other nodes'. Rule of thumb: derive CAN clock from the **crystal (SOSC)**,
  never an RC oscillator — CAN's tolerance math assumes crystal accuracy.
- **CAN FD:** most S32K parts support CAN FD — same arbitration, but up to
  **64 data bytes** and a faster data-phase bit rate (e.g. 500k/2M). Level 3
  material; everything above still applies.

A minimal SDK-style TX/RX (S32K1 SDK shapes — details vary by SDK version):

```c
#include "flexcan_driver.h"

flexcan_state_t canState;
const flexcan_user_config_t canCfg = {
    .max_num_mb      = 16u,
    .num_id_filters  = FLEXCAN_RX_FIFO_ID_FILTERS_8,
    .is_rx_fifo_needed = false,
    .flexcanMode     = FLEXCAN_NORMAL_MODE,
    .payload         = FLEXCAN_PAYLOAD_SIZE_8,
    .fd_enable       = false,
    .bitrate         = { /* 500 kbit/s timing from 8 MHz SOSC:        */
        .propSeg = 7, .phaseSeg1 = 4, .phaseSeg2 = 1,
        .preDivider = 0, .rJumpwidth = 1,
    },
};

void can_init(void)
{
    FLEXCAN_DRV_Init(0u, &canState, &canCfg);

    /* RX mailbox 1: accept standard ID 0x311 (our command frame) */
    flexcan_data_info_t rxInfo = {
        .msg_id_type = FLEXCAN_MSG_ID_STD, .data_length = 8u,
    };
    FLEXCAN_DRV_ConfigRxMb(0u, 1u, &rxInfo, 0x311u);
}

void can_send_status(const uint8_t data[8])
{
    flexcan_data_info_t txInfo = {
        .msg_id_type = FLEXCAN_MSG_ID_STD, .data_length = 8u,
    };
    FLEXCAN_DRV_Send(0u, 0u /* mailbox 0 */, &txInfo, 0x310u, data);
}

flexcan_msgbuff_t rxMsg;
bool can_poll_command(void)     /* non-blocking check for ID 0x311 */
{
    if (FLEXCAN_DRV_GetTransferStatus(0u, 1u) == STATUS_SUCCESS) {
        /* previous receive completed; rxMsg holds ID, DLC, data  */
        FLEXCAN_DRV_Receive(0u, 1u, &rxMsg);   /* re-arm mailbox  */
        return true;
    }
    return false;
}
```

At register level the same story is MCR (module config), CTRL1/CBT (bit
timing), and the message-buffer RAM with per-mailbox CODE/ID/DATA words —
the reference manual's FlexCAN chapter is long but well organized; Level 2
walks it mailbox-by-mailbox.

No board? You can still get hands-on: Linux's SocketCAN with a **virtual
CAN interface** (`vcan0`) plus `candump`/`cansend` teaches frames, IDs and
filtering with zero hardware; USB adapters (candleLight, PCAN) later make
your laptop a real bus node — that's also the capstone's bench-test tool.

## Cheat sheet

| Item | Notes |
|------|-------|
| Physical layer | Twisted pair CAN_H/CAN_L, differential, 120 Ω termination × 2 |
| Dominant / recessive | 0 beats 1 on the wire — basis of arbitration and ACK |
| Frame | ID + DLC + 0–8 data bytes + CRC + ACK |
| ID | Message identity **and** priority — lower wins arbitration |
| Arbitration | Bitwise, non-destructive; losers retry automatically |
| ACK | Some receiver validated the CRC — not an end-to-end delivery receipt |
| Bus-off | Nodes with persistent errors remove themselves — fault containment |
| Signals / DBC | Payload = packed scaled signals; `raw = (phys − offset)/scale` |
| Alive counter | Incrementing field letting receivers detect frozen senders |
| FlexCAN | S32K's CAN peripheral: mailboxes with ID+mask filters, HW retransmission |
| Bit timing | 500 kbit/s typical; sample point aligned across nodes; crystal clock only |
| No hardware? | Linux SocketCAN `vcan0` + `candump`/`cansend` |

## Exercise

Design the message set for a two-node system: a **pedal sensor node**
(measures pedal position twice, dual-channel) and a **motor controller**
that consumes it. Specify: (1) two or three frames with IDs chosen by
priority reasoning you write down, (2) full DBC-style signal layouts —
position channels, validity flags, alive counter, and a motor-status
return frame, (3) the motor controller's timeout rule (what does it do if
the pedal frame stops arriving for 100 ms?), and (4) one arbitration trace:
show bit-by-bit how your two lowest IDs would resolve a simultaneous start.
If you have Linux, bring up `vcan0` and replay your design with `cansend`
and `candump` to see the frames as tools display them.
