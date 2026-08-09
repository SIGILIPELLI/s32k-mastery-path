# Project — Multi-Node CAN Network

Level 1's capstone was one node talking to a bench tool. Everything in
Level 2 — mailboxes and filters, DMA, an RTOS, low power, flash, the
bootloader, UDS — exists because real ECUs share a bus with modules
written by other teams, booting at different times and failing
independently. This project builds that: **three nodes, one 500 kbit/s
bus, a defined message catalogue, and a fault-handling matrix that states
what each node does when any other goes quiet.** The firmware is the easy
half; the communication matrix and the timeout behaviour are the half that
gets reviewed.

## The system

A simplified cooling-fan subsystem:

```text
                     CAN 500 kbit/s, 120 Ω at each end
     ┌────────────┬─────────────────┬─────────────────┬────────────┐
    120Ω          │                 │                 │           120Ω
           ┌──────┴──────┐   ┌──────┴──────┐   ┌──────┴──────┐
           │ SENSOR node │   │ACTUATOR node│   │GATEWAY node │
           │   S32K144   │   │   S32K144   │   │   S32K144   │
           ├─────────────┤   ├─────────────┤   ├─────────────┤
           │ NTC via ADC │   │ FTM PWM→fan │   │ Arbitrates  │
           │ 10 ms sample│   │ Tacho input │   │ Owns UDS    │
           │ TX 0x310    │   │ RX 0x311    │   │ TX 0x100/311│
           │ RX 0x100    │   │ TX 0x312    │   │ RX 0x310/312│
           └─────────────┘   └─────────────┘   └─────────────┘
```

The sensor measures, the gateway decides, the actuator drives. No node
trusts another's liveness: each supervises the frames it consumes and has
a defined fallback when they stop.

## Message catalogue

Lower ID wins arbitration, so the ID allocation *is* the priority scheme.

| ID | Name | Producer | Consumers | Period | DLC |
|----|------|----------|-----------|--------|-----|
| `0x100` | `NetworkMode` | Gateway | Sensor, Actuator | 500 ms | 2 |
| `0x200` | `NodeFault` | any | Gateway | event, ≥ 100 ms gap | 4 |
| `0x310` | `SensorStatus` | Sensor | Gateway | 100 ms | 8 |
| `0x311` | `ActuatorCommand` | Gateway | Actuator | 100 ms | 4 |
| `0x312` | `ActuatorStatus` | Actuator | Gateway | 100 ms | 6 |
| `0x7DF`/`0x7E0`/`0x7E8` | UDS (module 9) | tester / Gateway | Gateway / tester | on request | ≤ 8 |

**`0x310` SensorStatus** — Level 1's layout carried forward:

| Bits | Signal | Type | Scale | Notes |
|------|--------|------|-------|-------|
| 0–15 | `CoolantTemp` | int16 | 0.1 °C | −40.0…150.0 °C |
| 16–31 | `SupplyVoltage` | uint16 | 1 mV | Node's own rail |
| 32 | `SensorValid` | bool | — | Plausibility result |
| 36–39 | `AliveCounter` | uint4 | — | Wraps 0→15 |
| 48–55 | `Checksum` | uint8 | — | XOR of bytes 0–6 plus the ID |

**`0x311` ActuatorCommand** carries `FanTargetPct` (uint8, 0–100),
`AliveCounter` (uint4), `RequestedMode` (uint2: 0 normal, 1 degraded,
2 safe) and a checksum. **`0x312` ActuatorStatus** returns `FanActualPct`
(uint8), `FanSpeedRpm` (uint16 from the tacho capture), `AliveCounter`,
an `OutputStageFault` bit and a checksum. **`0x100` NetworkMode** carries
a 2-bit mode (0 run, 1 degraded, 2 safe, 3 sleep-request) plus an alive
counter; **`0x200` NodeFault** carries the reporting node's ID, a fault
code and a fault counter.

## Expected message flow

One healthy 100 ms cycle. The offsets are deliberate — if every node
transmitted on its own 100 ms boundary they would collide every cycle and
the lowest-priority frame would jitter systematically.

```text
 t (ms)  0    10   20   30   40   50   60   70   80   90   100
         │    │    │    │    │    │    │    │    │    │    │
 Sensor  ├─0x310                                            ├─0x310
         │  ADC sampled every 10 ms, frame sent on the tick
 Gateway │    ├─0x100 (every 5th cycle)
         │         ├─0x311  built from the 0x310 seen at t=0
 Actuator│              ├─0x312  reports the PWM applied at t≈22 ms
```

The end-to-end latency the gateway must budget for:

```text
sensor sample → 0x310 TX      ≤ 10 ms   (sampling phase)
0x310 on the wire             ≈ 0.26 ms (8-byte frame at 500 kbit/s)
gateway processes → 0x311     = 20 ms   (fixed offset)
0x311 on the wire             ≈ 0.26 ms
actuator applies PWM          ≤ 10 ms   (its own tick)
                              ─────────
worst case                    ≈ 41 ms
```

Write that number down — it is the first thing a systems engineer asks
for, and it changes the moment anyone adds a queue. Bus load deserves the
same treatment:

| Frame | Rate | Worst-case bits | Bits/s |
|-------|------|-----------------|--------|
| `0x310` (8 B) | 10 Hz | ~130 | 1300 |
| `0x311` (4 B) | 10 Hz | ~98 | 980 |
| `0x312` (6 B) | 10 Hz | ~114 | 1140 |
| `0x100` (2 B) | 2 Hz | ~82 | 164 |
| **Total** | | | **≈ 3.6 kbit/s → 0.7 % of 500 kbit/s** |

Worst-case bits include standard-frame overhead plus maximum bit stuffing.
Under 1 % sounds like infinite headroom; the point of computing it is that
when someone proposes a 20 ms trace frame carrying 64 bytes, you answer
with a number instead of an opinion.

## Shared definitions

Every node compiles the same header. That is not tidiness — it is how you
guarantee three teams agree on byte 4.

```c
/* can_matrix.h — shared by all three nodes, single source of truth */
#define ID_NETWORK_MODE      0x100u
#define ID_NODE_FAULT        0x200u
#define ID_SENSOR_STATUS     0x310u
#define ID_ACTUATOR_COMMAND  0x311u
#define ID_ACTUATOR_STATUS   0x312u

#define PERIOD_MS_STATUS     100u
#define TIMEOUT_MS_STATUS    300u    /* three missed cycles            */

typedef enum {
    NET_MODE_RUN = 0u, NET_MODE_DEGRADED = 1u,
    NET_MODE_SAFE = 2u, NET_MODE_SLEEP = 3u,
} net_mode_t;

/* The ID is folded in, so a frame delivered under the wrong identifier
   — a mis-set filter, a copy-paste in another node — fails the check
   instead of being silently accepted.                               */
static inline uint8_t can_checksum(uint16_t id, const uint8_t *d, uint8_t n)
{
    uint8_t c = (uint8_t)(id & 0xFFu) ^ (uint8_t)(id >> 8);
    for (uint8_t i = 0u; i < n; i++) { c ^= d[i]; }
    return c;
}

typedef struct {
    uint8_t  lastAlive;
    uint32_t age_ms;
    bool     valid;
} sig_health_t;

/* Called from each node's 10 ms task, once per consumed signal. */
static inline bool sig_supervise(sig_health_t *h, uint32_t timeout_ms)
{
    h->age_ms += 10u;
    if (h->age_ms > timeout_ms) { h->valid = false; }
    return h->valid;
}
```

## Receiving with the RX FIFO

The gateway consumes three IDs, so it uses module 1's RX FIFO rather than
three mailboxes and three interrupts:

```c
static flexcan_state_t   canState;
static flexcan_msgbuff_t rxMsg;

static flexcan_id_table_t gwFilters[8] = {
    { .isRemoteFrame = false, .isExtendedFrame = false, .id = ID_SENSOR_STATUS   },
    { .isRemoteFrame = false, .isExtendedFrame = false, .id = ID_ACTUATOR_STATUS },
    { .isRemoteFrame = false, .isExtendedFrame = false, .id = ID_NODE_FAULT      },
    /* Pad with a repeat of a used ID — never 0x000, which would accept
       the highest-priority frame on the bus (module 1).              */
    { false, false, ID_NODE_FAULT }, { false, false, ID_NODE_FAULT },
    { false, false, ID_NODE_FAULT }, { false, false, ID_NODE_FAULT },
    { false, false, ID_NODE_FAULT },
};

static void gw_can_cb(uint8_t instance, flexcan_event_type_t event,
                      uint32_t buffIdx, flexcan_state_t *state)
{
    (void)buffIdx; (void)state;
    switch (event) {
    case FLEXCAN_EVENT_RXFIFO_COMPLETE:
        gw_queue_frame(&rxMsg);                    /* copy out, return */
        (void)FLEXCAN_DRV_RxFifo(instance, &rxMsg);
        break;
    case FLEXCAN_EVENT_RXFIFO_OVERFLOW:
        g_rxOverflows++;                           /* a frame was lost */
        break;
    default:
        break;
    }
}

void gw_can_init(void)
{
    flexcan_user_config_t cfg;
    FLEXCAN_DRV_GetDefaultConfig(&cfg);
    cfg.max_num_mb        = 16u;
    cfg.is_rx_fifo_needed = true;
    cfg.num_id_filters    = FLEXCAN_RX_FIFO_ID_FILTERS_8;
    cfg.flexcanMode       = FLEXCAN_NORMAL_MODE;
    cfg.payload           = FLEXCAN_PAYLOAD_SIZE_8;

    (void)FLEXCAN_DRV_Init(INST_CAN0, &canState, &cfg);
    (void)FLEXCAN_DRV_ConfigRxFifo(INST_CAN0,
                                   FLEXCAN_RX_FIFO_ID_FORMAT_A, gwFilters);
    FLEXCAN_DRV_SetRxFifoGlobalMask(INST_CAN0, FLEXCAN_MSG_ID_STD,
                                    (0x7FFu << 18));   /* exact match  */
    FLEXCAN_DRV_InstallEventCallback(INST_CAN0, gw_can_cb, NULL);
    (void)FLEXCAN_DRV_RxFifo(INST_CAN0, &rxMsg);
}
```

Validation happens in the task, never in the callback:

```c
bool gw_accept_sensor_status(const flexcan_msgbuff_t *m)
{
    if (m->dataLen != 8u) { return false; }
    if (m->data[6] != can_checksum(ID_SENSOR_STATUS, m->data, 6u)) {
        g_checksumErrors++;
        return false;                     /* corrupted or mis-routed  */
    }
    uint8_t alive = (m->data[4] >> 4) & 0x0Fu;
    if (alive == g_sensorHealth.lastAlive) {
        return false;                     /* stale repeat, not fresh  */
    }
    g_sensorHealth.lastAlive = alive;
    g_sensorHealth.age_ms    = 0u;
    g_sensorHealth.valid     = true;

    g_temp_dC = (int16_t)((uint16_t)m->data[0] |
                          ((uint16_t)m->data[1] << 8));
    return ((m->data[4] & 0x01u) != 0u);  /* SensorValid bit          */
}
```

## Fault-handling matrix

This table is the real deliverable. Every row is a failure a reviewer will
ask about, and every cell is a decision you have to defend.

| Failure | Detected by | Detection time | Response |
|---------|-------------|----------------|----------|
| Sensor node silent | Gateway: `0x310` age > 300 ms | ≤ 300 ms | Fan to 100 % (fail-safe cooling), send `0x200`, set DTC |
| Sensor reports invalid | `SensorValid` = 0 | 1 cycle | Same — an implausible reading is not a low reading |
| Gateway silent | Actuator: `0x311` age > 300 ms | ≤ 300 ms | Hold last command 1 s, then ramp to the safe default |
| Actuator silent | Gateway: `0x312` age > 300 ms | ≤ 300 ms | Send `0x200`, set DTC, keep transmitting `0x311` |
| Checksum failures | Any consumer | immediate | Discard and count; 5 consecutive = treat as silence |
| Alive counter frozen | Any consumer | 2 cycles | Producer's task has hung — treat as silence |
| Bus-off | `ESR1[BOFFINT]` | ≤ 1 frame | Safe state, rate-limited recovery (module 1), latch after 3 |
| Node reset | Its own `RCM->SRS` | at boot | Outputs safe before CAN init; report cause in `0x200` |
| Both terminators removed | Rising `ECR` counts | ≤ 1 s | All nodes reach bus-off independently and go safe |

The important asymmetry: **the actuator holds the last command for 1 s
before defaulting, while the gateway acts within 300 ms.** A brief gateway
glitch should not slam the fan; a genuinely absent gateway must not leave
the fan under stale control forever. Where that boundary sits is a system
decision, and writing it down is what makes it reviewable.

## Automotive concerns

- **Nodes boot at different times, and that is normal.** The gateway sees
  nothing from the actuator for a few hundred milliseconds after key-on.
  Start every supervisor in a "not yet valid" state with a startup grace
  period, distinct from "was valid, went silent" — they deserve different
  DTCs.
- **Alive counter plus checksum is the minimum for anything that moves an
  actuator.** The checksum catches corruption; the alive counter catches a
  producer whose task has hung while its DMA or mailbox keeps re-sending
  the same bytes. Neither alone is sufficient — together they are what an
  end-to-end protection profile formalises.
- **Include the ID in the checksum.** A mailbox configured with the wrong
  ID, or a copy-pasted transmit routine in another node, produces
  perfectly valid-looking data under the wrong identifier. Folding the ID
  in makes that failure loud instead of silent.
- **DMA and CAN compete for attention.** If the sensor node scans its ADC
  by eDMA (module 2), channel priorities and the FlexCAN interrupt
  priority interact — a DMA error interrupt at the wrong priority can
  preempt the CAN callback and cost you a FIFO slot. Assign channels and
  priorities once, in a table, and review them together.
- **Flash writes stall the node.** If the gateway stores a DTC while a
  `0x311` transmission is due, the flash operation blocks for milliseconds
  and the frame jitters. Queue DTC writes and flush them in the
  lowest-priority task or at shutdown (module 6), never inline in a fault
  handler.
- **A UDS session changes the whole network's behaviour.** An extended
  session forcing a fan test overrides the sensor's request, so every node
  must know that `RequestedMode` can originate from diagnostics — and the
  S3 timeout (module 9) must return the *network* to `NET_MODE_RUN`, not
  just the gateway.
- **Sleep is a network decision.** `NetworkMode` = 3 is a *request*; a
  node with a pending fault or an active diagnostic session must be able
  to veto it. Module 5's wakeup sources give you the way back — but only
  if every node agreed to go down in the first place.
- **One termination is a detectable fault.** With a single 120 Ω resistor
  the bus often still works, until temperature or cable length makes it
  not. Export `ECR` error-counter statistics from every node; a node with
  persistently elevated counts on an otherwise healthy bus is telling you
  about its own wiring.

## Cheat sheet

| Item | Notes |
|------|-------|
| Topology | 3 nodes, one 500 kbit/s bus, 120 Ω at **both** physical ends only |
| ID = priority | `0x100` mode · `0x200` fault · `0x310`–`0x312` cyclic · `0x7Ex` diag |
| Periods | 100 ms cyclic, 500 ms network mode, event faults with a ≥ 100 ms gap |
| Phasing | Offset producers by 10–20 ms so cyclic frames do not collide every cycle |
| Timeout | 300 ms = three missed cycles, for every consumed signal |
| Protection | Alive counter (staleness) **and** ID-inclusive checksum (corruption) |
| Latency budget | ≈ 41 ms sample → actuator applied; recompute if a queue is added |
| Bus load | ≈ 0.7 % — compute it before anyone proposes a trace frame |
| Gateway RX | RX FIFO, 8-element format-A table, no `0x000` padding entries |
| Callback rule | Copy and re-arm; checksum, alive and range checks belong in the task |
| Fail-safe | Fan to 100 % on lost or invalid temperature — safe direction, not zero |
| Shared header | `can_matrix.h` compiled by all three nodes — one source of truth |
| Bench tools | 3 × S32K144 EVB (or 2 EVBs + PCAN-USB), `candump -t z`, `cansend` |

## Exercise

Build the network. (1) Write `can_matrix.h` and the message catalogue as a
document *before* any node code, and have someone read it back to you — if
they can implement byte 4 without asking a question, the table is done.
(2) Bring up the sensor node and the gateway first, with `candump` playing
the actuator; confirm `0x310` at 100 ms ±2 ms, `0x311` phased 20 ms behind
it, and both alive counters advancing without a skip over ten thousand
frames. (3) Add the actuator node with its own tick and tacho capture,
then measure the real end-to-end latency: toggle a GPIO in the sensor node
when it samples and another in the actuator node when it applies PWM, put
both on a scope, and explain any difference from your 41 ms budget.
(4) Work through the fault-handling matrix row by row on the bench —
unplug the sensor node, freeze the gateway in the debugger, transmit a
hand-built frame with a bad checksum, remove a terminator. Each row must
produce the documented response within the documented time, and you must
*observe* it, not infer it. (5) Instrument every node with `ECR` error
counters, RX FIFO overflow counts and checksum-failure counts exported in
`0x200`, then run the network for an hour; any non-zero counter at the end
is a defect with an hour of evidence attached.

## Stretch goals

- **Add the bootloader and UDS from modules 8 and 9 to the gateway**, then
  reflash it over CAN while the other two nodes keep running. The
  actuator's hold-then-default behaviour during the reprogramming window
  is the moment those two modules stop being theory.
- **Move the sensor node's ADC scan to eDMA** (module 2) with a ping-pong
  buffer, and show on a scope that CAN transmission jitter drops because
  the CPU no longer services per-conversion interrupts.
- **Put one node on FreeRTOS** (module 4) with separate CAN-RX, control
  and diagnostic tasks, and demonstrate that the priority assignment holds
  the 100 ms period under a deliberately overloaded low-priority task.
- **Implement network sleep and wake** (module 5): the gateway requests
  sleep with `NetworkMode` = 3, each node acknowledges or vetoes, all
  three enter VLPS with FlexCAN Pretended Networking armed, and one
  `0x100` frame wakes the network. Measure one node's current before and
  after.
- **Add a fourth node from a different codebase** — a Raspberry Pi with a
  CAN HAT simulating the actuator in Python. If your matrix is good, that
  node integrates from the document alone, which is exactly the test the
  document exists to pass.
- **Introduce CAN FD on the same topology** (module 7's migration notes
  apply): raise the data phase to 2 Mbit/s, extend `0x310` to a 64-byte
  payload carrying a sensor history, and recompute the bus load and the
  latency budget from scratch.
