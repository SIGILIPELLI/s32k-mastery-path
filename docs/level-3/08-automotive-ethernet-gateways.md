# Automotive Ethernet & Gateways

Every bus so far — CAN, CAN FD, LIN — tops out in the low megabits.
Camera feeds for surround-view, radar point clouds, and high-bandwidth
diagnostic reflashing over a factory network all exceed what any CAN
variant can carry. **Automotive Ethernet** (100BASE-T1/1000BASE-T1,
single unshielded twisted pair instead of the 4/8-wire cabling of
commercial Ethernet) fills that gap, and S32K3-class MCUs increasingly
sit as **gateway nodes** — bridging a CAN/LIN-based body domain to an
Ethernet backbone, translating signals between the two worlds. This
module covers the S32K's Ethernet MAC/DMA path and the gateway patterns
that make CAN-to-Ethernet bridging a real architectural decision, not a
protocol footnote.

## Why single-pair automotive Ethernet, not the office kind

```text
Commercial Ethernet (100BASE-TX)     Automotive (100BASE-T1)
─────────────────────────────        ─────────────────────────
2 pairs (4 wires) or more            1 pair (2 wires)
RJ45 connector                       Sealed automotive connectors
No EMC requirement for engine bay    Must survive harsh EMC + vibration
Not rated for -40°C to 125°C         Automotive temperature range
```

100BASE-T1 (formerly "BroadR-Reach") uses PAM-3 encoding over one
twisted pair to fit into the weight, connector-count, and EMC budget a
vehicle harness can afford — full-duplex 100 Mbit/s (or 1000BASE-T1 for
higher-bandwidth links) on wiring that would carry only a fraction of
that with legacy encoding.

## S32K Ethernet MAC/DMA path

```c
/* S32K3 ENET (Ethernet MAC) uses a descriptor-ring DMA model, similar
   in shape to FlexCAN's DMA but for streamed frames rather than
   fixed-size mailboxes */
typedef struct {
    uint32_t status;
    uint32_t length;
    uint8_t  *buf_addr;
} enet_bd_t;  /* buffer descriptor */

#define ENET_RX_RING_SIZE 8u
static enet_bd_t rx_ring[ENET_RX_RING_SIZE] __attribute__((aligned(64)));

void Enet_Init(ENET_Type *base)
{
    base->RDSR = (uint32_t)rx_ring;         /* receive descriptor ring start */
    base->MRBR = ENET_MAX_FRAME_LEN;        /* max receive buffer size */
    for (uint32_t i = 0u; i < ENET_RX_RING_SIZE; i++) {
        rx_ring[i].buf_addr = &rx_buffers[i][0];
        rx_ring[i].status = ENET_BD_EMPTY | ((i == ENET_RX_RING_SIZE - 1u) ? ENET_BD_WRAP : 0u);
    }
    base->ECR |= ENET_ECR_ETHEREN_MASK;     /* enable the MAC */
}
```

The descriptor ring means the DMA controller — not the core — moves
Ethernet frame data in and out of RAM; the core only handles descriptor
housekeeping and the ISR when a ring's frame is ready. This is the same
DMA-conflict risk surface from Level 2's CAN/ADC DMA work, now carrying
much larger buffers at higher rates, so ring sizing and buffer alignment
matter more here than they did there.

## The CAN-to-Ethernet gateway pattern

```c
/* Gateway: a CAN frame arrives, gets translated to a SOME/IP-style
   or raw UDP payload, and forwarded onto the Ethernet segment.
   This is the PduR's job in a full AUTOSAR stack (module 1); shown
   here at the signal level for clarity. */
typedef struct {
    uint32_t can_id;
    uint8_t  can_data[8];
} can_frame_t;

typedef struct {
    uint16_t service_id;
    uint16_t method_id;
    uint8_t  payload[64];
    uint16_t payload_len;
} someip_msg_t;

void Gateway_CanToEthernet(const can_frame_t *frame)
{
    someip_msg_t msg;
    if (!GatewayRoutingTable_Lookup(frame->can_id, &msg.service_id, &msg.method_id)) {
        return; /* unmapped CAN ID: not every signal crosses the gateway */
    }
    memcpy(msg.payload, frame->can_data, sizeof(frame->can_data));
    msg.payload_len = sizeof(frame->can_data);
    Someip_Send(&msg);   /* over UDP/TCP via the ENET MAC */
}
```

The routing table lookup is the architecturally important line: a
gateway that blindly forwards every CAN ID onto Ethernet turns a
bandwidth-constrained, physically-segmented CAN bus into one exposed on
a much higher-bandwidth network — every signal that crosses a gateway is
a deliberate decision, both for bandwidth and for the security boundary
it represents.

## Automotive-MCU concerns

- **A gateway is a trust boundary, not just a protocol translator.** CAN
  bus physical access historically required opening the vehicle; Ethernet
  gateways sometimes bridge to segments reachable from an infotainment
  unit or a diagnostic port with looser physical security. Never forward
  safety-relevant CAN signals onto Ethernet without the same
  authentication/integrity controls (module 4 Level 4's SecOC) applied
  at the boundary — a gateway is exactly the node an ISO/SAE 21434 threat
  analysis focuses on first.
- **DMA descriptor ring exhaustion under burst traffic looks like a
  dropped-frame mystery.** If the core doesn't service completed RX
  descriptors fast enough during a burst (e.g. a diagnostic flash session
  saturating the link), the ring wraps and new frames overwrite
  not-yet-processed ones. Size the ring and the servicing ISR's priority
  for your actual worst-case burst, not average throughput.
- **100BASE-T1 link training and autonegotiation differ from commercial
  Ethernet — do not assume standard PHY bring-up code works unmodified.**
  Automotive PHYs (e.g. TJA1101-class) use a master/slave clocking role
  configured per-node, not autonegotiated the way 802.3 commercial PHYs
  do; a master/master or slave/slave misconfiguration between two nodes
  on the same segment prevents link-up entirely, and looks identical to
  a wiring fault.
- **Clock domain crossing between the CAN and Ethernet sides of a gateway
  needs an explicit buffering/synchronization strategy.** CAN frames
  arrive asynchronously relative to the Ethernet frame's time-sensitive
  networking (TSN) schedule if one is in use; a gateway design that
  assumes "just copy the bytes across" without a jitter buffer produces
  timing-variable latency that a real-time consumer downstream may not
  tolerate.

## Cheat sheet

| Term | Meaning |
|------|---------|
| 100BASE-T1 | Single twisted pair, 100 Mbit/s automotive Ethernet PHY standard |
| 1000BASE-T1 | Single pair, 1 Gbit/s variant, higher-bandwidth backbone links |
| PAM-3 | Encoding scheme automotive single-pair Ethernet uses to fit the wiring budget |
| ENET | S32K's Ethernet MAC/DMA peripheral |
| Buffer descriptor (BD) ring | DMA-managed circular list of RX/TX buffers, core only handles completed entries |
| SOME/IP | Common automotive service-oriented middleware protocol carried over Ethernet |
| PduR | AUTOSAR's protocol data unit router — the "real" layer that does gateway routing in a full stack |
| Master/slave PHY role | Automotive PHY clocking role, configured not autonegotiated — mismatch prevents link-up |

## Exercise

Design a CAN-to-Ethernet gateway node, and implement what your hardware
allows. (1) Define a routing table mapping a small set of CAN IDs to
Ethernet-side message identifiers (SOME/IP service/method IDs or a
simple UDP payload scheme), and implement the lookup-and-forward function
shown above, explicitly rejecting unmapped IDs rather than passing them
through by default. (2) If S32K3 hardware with ENET is available, bring
up the descriptor ring RX path and confirm you can receive and parse a
raw Ethernet frame sent from a PC on the same link; if not, implement and
unit-test the ring management logic against a simulated descriptor array.
(3) Size your RX descriptor ring for a defined worst-case burst (e.g. a
2 MB diagnostic transfer at your link's line rate) and calculate how many
descriptors you need given your servicing ISR's measured or estimated
latency. (4) Write a short threat-boundary note: which CAN signals in
your design are safety-relevant, and what integrity/authentication
control (even if only "not yet implemented — flagged for module 5 of
Level 4") applies to each before it crosses the gateway.
