# CAN FD Networks & DBC Workflows

Every CAN frame you built in Level 1 was Classic CAN: 8 data bytes, one
bit rate for the whole frame. Real vehicle networks moved past that
because 8 bytes cannot carry a modern powertrain torque-request signal
set, and re-splitting a signal across multiple frames adds latency and
synchronization bugs. **CAN FD** (Flexible Data-rate) raises the payload
to 64 bytes and lets the data phase run at a higher bit rate than
arbitration — and the S32K's FlexCAN peripheral supports it directly, no
external transceiver logic needed beyond a CAN FD-capable transceiver
(e.g. TJA1057 will not do FD; you need a TJA1443 or similar). Alongside
the protocol change, this module covers the tooling change every OEM
project requires: signals are never hand-encoded from a raw frame ID.
They are defined in a **DBC file** and generated into C.

## CAN FD frame shape

```text
Classic CAN:  ID | DLC(0-8) | up to 8 data bytes | CRC-15
CAN FD:       ID | DLC(0-15, maps to 0-64 bytes) | up to 64 data bytes | CRC-17/21
              + BRS bit (Bit Rate Switch) + ESI bit (Error State Indicator)
```

DLC no longer equals byte count above 8: DLC 9=12 bytes, 10=16, 11=20,
12=24, 13=32, 14=48, 15=64. **Arbitration phase always runs at the
nominal bit rate** (e.g. 500 kbit/s, for bus-wide arbitration compatibility
with any Classic CAN nodes); if BRS is set, the data phase switches to a
faster bit rate (commonly 2 or 4 Mbit/s) for the payload and CRC, then
drops back to nominal for the last bit and ACK.

```c
/* S32K FlexCAN FD message buffer configuration (RTD-style, extends module 2) */
typedef struct {
    uint32_t id;
    bool     fd_enable;   /* EDL bit — Extended Data Length */
    bool     brs_enable;  /* BRS bit — Bit Rate Switch, needs fd_enable */
    uint8_t  dlc;         /* 0-15, per the FD DLC-to-length mapping above */
    uint8_t  data[64];
} can_fd_frame_t;

Std_ReturnType Can_FD_Write(uint8 hth, const can_fd_frame_t *frame)
{
    /* CTRL2[EDL] and CTRL2[BRS] per message buffer, not global —
       an FD-capable controller can still send Classic frames to
       legacy nodes on the same bus */
    if (frame->fd_enable && !CanFdCapable(hth)) {
        return E_NOT_OK;
    }
    return Can_47_FlexCAN_Write(hth, (const Can_PduType *)frame);
}
```

## DBC: the signal database

A `.dbc` file is the actual interface contract for a vehicle network —
not the C code, not the ARXML port, the DBC. It defines every frame's ID,
length, and the bit-level layout of every signal packed inside it:

```text
BO_ 1024 BCM_DoorStatus: 8 Vector__XXX
 SG_ FL_Door_Locked : 0|1@1+ (1,0) [0|1] ""  Receiver_ECU
 SG_ FR_Door_Locked : 1|1@1+ (1,0) [0|1] ""  Receiver_ECU
 SG_ Vehicle_Speed  : 8|16@1+ (0.01,0) [0|327.67] "km/h"  Receiver_ECU
```

`BO_` declares the frame (ID 1024, 8 bytes). Each `SG_` declares a signal:
start bit, bit length, byte order (`1` = little-endian/Intel, `0` = big-
endian/Motorola), signedness, then `(factor,offset)` and `[min|max]`
physical range. `Vehicle_Speed` at raw value 6000 decodes to
`6000 * 0.01 + 0 = 60.00 km/h` — that scale-and-offset arithmetic is
exactly what a code generator turns into C, and exactly what you must
never re-derive by hand from a spec PDF, because a single off-by-one in
a start bit silently reads the wrong 16 bits.

```c
/* Generated from the DBC above (illustrative — real generators
   produce this from cantools, PCAN's DBC-to-C, or a vendor's own tool) */
static inline uint16_t BCM_DoorStatus_encode_speed(float speed_kph)
{
    return (uint16_t)(speed_kph / 0.01f);   /* inverse of (factor,offset) */
}

static inline void BCM_DoorStatus_pack(uint8_t *data, bool fl, bool fr, float speed)
{
    uint16_t raw_speed = BCM_DoorStatus_encode_speed(speed);
    data[0] = (fl ? 0x01u : 0u) | (fr ? 0x02u : 0u);
    data[1] = (uint8_t)(raw_speed & 0xFFu);
    data[2] = (uint8_t)((raw_speed >> 8) & 0xFFu);
}
```

## Automotive-MCU concerns

- **Never hand-decode DBC bit offsets in application code.** Generate the
  pack/unpack functions (cantools' `cantools generate_c_source` is the
  common open-source path) and treat the DBC file as the single source
  of truth. A hand-transcribed start-bit error is invisible in review —
  the code compiles and runs, it just silently reads the wrong signal.
- **Mixed Classic/FD buses need every node's tolerance confirmed.** A
  Classic-only node ignores FD frames' EDL bit correctly only if its
  FlexCAN instance is configured to *tolerate* FD frames on the bus
  (`CTRL2[ISOCANFDEN]`-equivalent) even if it never sends them; an older
  or misconfigured Classic node can instead flag a form error and go
  bus-off when it sees the first FD frame's reserved-bit pattern.
- **BRS data-phase bit rate has a real physical limit set by your
  transceiver and stub length, not just the FlexCAN prescaler.** 5 Mbit/s
  data phase looks correct in FlexCAN CTRL1FD register math and fails on
  the bench because the transceiver's loop delay or an unterminated stub
  causes reflections the receiver samples as bit errors — CRC-17/21
  failures cluster at the data-phase-to-nominal-phase boundary when this
  happens.
- **ISO 11898-1:2015 CAN FD is not backward-bit-compatible with the
  pre-ISO "Bosch CAN FD" some early tooling and silicon implemented.**
  Confirm every node on a bus implements the same CRC computation
  (`ISOCANFDEN` bit on FlexCAN selects it) — a mixed bus with one non-ISO
  node produces CRC errors on every FD frame that node touches, and the
  fault looks like noise, not a protocol version mismatch.

## Cheat sheet

| Concept | Classic CAN | CAN FD |
|---------|--------------|--------|
| Max payload | 8 bytes | 64 bytes |
| DLC 9-15 meaning | N/A (max 8) | 12, 16, 20, 24, 32, 48, 64 bytes |
| Data phase bit rate | Same as arbitration | Can switch faster if BRS=1 |
| CRC | CRC-15 | CRC-17 (≤16 bytes) / CRC-21 (>16 bytes) |
| Key control bits | — | EDL (FD frame), BRS (bit rate switch), ESI (error state) |
| S32K register area | `CTRL1`, `MB[n].CS` | `CTRL2[ISOCANFDEN]`, `FDCTRL`, `MB[n].CS[EDL,BRS]` |

| DBC token | Meaning |
|-----------|---------|
| `BO_ <id> <name>: <dlc> <sender>` | Frame definition |
| `SG_ <name> : <start>\|<len>@<order><sign> (<factor>,<offset>) [<min>\|<max>] "<unit>" <receiver>` | Signal definition |
| `@1` / `@0` | Little-endian (Intel) / big-endian (Motorola) |
| `+` / `-` | Unsigned / signed |
| `physical = raw * factor + offset` | Decode formula |

## How It Actually Works

CAN FD's speed gain comes from a genuine change in bit-timing hardware, not just a faster clock: the arbitration phase (identifier field) runs at the classic bit rate because arbitration *requires* every node to see a dominant bit within one bit-time of driving it — the round-trip propagation delay across the physical bus length sets a hard ceiling on arbitration speed. Once arbitration is won and the BRS (Bit Rate Switch) bit is sent, FlexCAN's bit-timing state machine — a second, separate set of timing-segment registers for the data phase — switches to a faster clock division specifically because no arbitration (and thus no propagation-delay-bound resynchronization) is needed during the data phase; only the single transmitting node is driving the bus, so the data-phase bit rate is limited by transceiver switching speed and cable characteristics, not arbitration physics.

CAN FD's CRC also changes because the FD frame format and higher data-phase bit rate would make the classic CRC's stuffing-bit ambiguity a real problem — CAN FD uses a longer CRC (17 or 21 bits depending on payload length) computed by dedicated CRC hardware inside FlexCAN's FD-capable message-buffer logic, and a modified bit-stuffing scheme (stuff-bit counting) that lets the receiver detect a specific class of stuffing errors the classic scheme couldn't.

A DBC file's scaling factors (`/ 0.1 offset -40` style signal definitions) aren't hardware at all — they're a pure software convention mapping the raw integer bits FlexCAN's message buffer actually stores into physical units; the ECU's application code has to explicitly apply that same scale/offset math when packing/unpacking signals, because FlexCAN's hardware only ever moves and matches raw bytes, with zero awareness of what a "temperature" or "RPM" signal means.

*(Described from ISO 11898-1 (CAN FD amendment) and the FlexCAN FD reference manual chapter; not measured on physical silicon in this course.)*

## Exercise

Take a CAN network you already defined informally (frame IDs and byte
offsets in comments, from Level 1/2) and formalize it as a real DBC file.
(1) Write a `.dbc` file describing at least 3 frames and 6 signals total,
including one multi-byte little-endian signal with a non-trivial
factor/offset (not just 1.0/0). (2) Use `cantools` (`pip install cantools`)
to validate the file (`cantools dump your.dbc`) and generate C pack/unpack
functions from it; confirm the generated code's bit math matches what you
intended by hand-computing one signal's raw encoding. (3) Extend one
frame to a CAN FD frame carrying more than 8 bytes of signals, and update
the DBC's `BO_` DLC field to the FD-mapped value. (4) On real S32K
hardware, enable `ISOCANFDEN` and `BRS`, and use `candump` (or PCAN-View)
on a CAN FD-capable interface to confirm the frame decodes correctly
against your DBC with `cantools decode`.
