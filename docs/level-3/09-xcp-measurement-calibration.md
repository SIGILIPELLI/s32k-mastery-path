# Measurement & Calibration (XCP)

UDS (Level 2 module 9) exists so a technician can diagnose a fault after
the fact. It is the wrong tool for a calibration engineer tuning a PID
controller's gains while the ECU runs on a dyno, watching an internal
variable update at 1 kHz. **XCP** (Universal Measurement and Calibration
Protocol, ASAM standard) is built for exactly that: reading internal
variables in near-real-time ("measurement") and rewriting constants in
flash or RAM while the ECU keeps running ("calibration"), both driven
from a PC tool like ETAS INCA or Vector CANape, transported over CAN,
CAN FD, or Ethernet.

## Why XCP instead of just reading variables over UDS

```text
UDS 0x22 (ReadDataByIdentifier)      XCP measurement (DAQ)
──────────────────────────────       ──────────────────────
Request/response, one DID at a       Continuous streaming of many
time, polled                         signals synchronized to a
                                      time base, no per-signal request
Designed for diagnostics/config      Designed for high-rate tuning
No concept of "every task cycle"     DAQ lists tied to ECU event
                                      channels (e.g. every 10ms task)
```

A calibration engineer watching 40 internal variables at 1 kHz over
polled request/response reads would saturate the bus with overhead
before getting useful data. XCP's **DAQ (Data AcQuisition)** lists let
the tool configure once — "these addresses, this event channel" — and
then the ECU pushes data continuously with minimal per-sample overhead.

## Addresses, not signal names

The defining feature of XCP that trips people up coming from UDS: **XCP
addresses raw memory, not identifiers.** The PC tool needs to know the
exact address of every variable it wants to read or write, which comes
from an **A2L file** (ASAM MCD-2MC) — a description file mapping symbolic
names to memory addresses, generated from your build's debug symbols or
map file.

```text
/* Fragment of an A2L MEASUREMENT block */
/begin MEASUREMENT
  EngineTorqueRequest   /* name */
  "Requested engine torque, Nm"
  FLOAT32_IEEE          /* datatype */
  NO_COMPU_METHOD
  1 0                   /* resolution, accuracy */
  -500 2000             /* lower, upper limit */
  ECU_ADDRESS 0x20004120
/end MEASUREMENT
```

```c
/* The variable itself — an ordinary global, nothing XCP-specific
   about the C code. XCP's job is purely to expose the address. */
volatile float32_t EngineTorqueRequest;   /* 0x20004120 in the linked image */
```

**The A2L file and the linked binary must match exactly.** If the A2L
was generated against a build where `EngineTorqueRequest` sat at
`0x20004120` and you relink with one more global declared earlier in the
same section, the address shifts — and the calibration tool now reads or
*writes* an unrelated variable at the old address with no error, because
XCP has no idea the mapping is stale. This is the single most common XCP
integration bug: mismatched A2L-to-binary versions, and it fails silently.

## XCP driver skeleton on S32K

```c
/* Minimal XCP slave command dispatch — CONNECT and standard read/write,
   transported here over CAN (XCP-on-CAN uses a pair of CAN IDs) */
typedef enum {
    XCP_CMD_CONNECT       = 0xFFu,
    XCP_CMD_SHORT_UPLOAD  = 0xF4u,  /* read memory at address */
    XCP_CMD_DOWNLOAD      = 0xF0u,  /* write memory at address */
    XCP_CMD_SET_MTA       = 0xF6u,  /* set memory transfer address */
} xcp_cmd_t;

void Xcp_ProcessCommand(const uint8_t *cmd_data, uint8_t len)
{
    switch (cmd_data[0]) {
    case XCP_CMD_CONNECT:
        Xcp_SendResponse(XCP_RES_POSITIVE, xcp_resource_mask, sizeof(xcp_resource_mask));
        break;
    case XCP_CMD_SET_MTA: {
        uint32_t addr;
        memcpy(&addr, &cmd_data[4], sizeof(addr));
        xcp_mta = (uint8_t *)addr;   /* raw pointer from the wire — see concerns below */
        Xcp_SendResponse(XCP_RES_POSITIVE, NULL, 0);
        break;
    }
    case XCP_CMD_SHORT_UPLOAD: {
        uint8_t n_bytes = cmd_data[1];
        Xcp_SendResponse(XCP_RES_POSITIVE, xcp_mta, n_bytes); /* reads live memory */
        break;
    }
    case XCP_CMD_DOWNLOAD: {
        uint8_t n_bytes = cmd_data[1];
        memcpy(xcp_mta, &cmd_data[2], n_bytes);  /* writes live memory! */
        Xcp_SendResponse(XCP_RES_POSITIVE, NULL, 0);
        break;
    }
    default:
        Xcp_SendResponse(XCP_RES_ERROR, NULL, 0);
    }
}
```

`SET_MTA` takes an address literally off the wire and `DOWNLOAD` writes
to it — this is the protocol working exactly as designed, and also
exactly why it must never ship enabled in a production, non-development
build.

## Automotive-MCU concerns

- **An XCP slave that accepts writes anywhere in address space is a
  full remote memory-write primitive.** A production ECU must either
  disable XCP entirely (compiled out) or restrict `SET_MTA`/`DOWNLOAD`
  to a whitelisted calibration-page address range checked in firmware —
  never trust the A2L file, which lives on the PC tool, as the only
  access control.
- **Calibration writes to flash need the same wear-leveling discipline
  as UDS `0x2E`.** A calibration engineer sweeping a gain value across
  100 test points, if each XCP `DOWNLOAD` triggers an immediate flash
  write, wears the sector in minutes. Real calibration flows write to a
  shadow RAM "working page" during a live tuning session and commit to
  flash only explicitly — check your A2L's memory segment definitions
  distinguish RAM (`ECU_ADDRESS` in RAM) calibration pages from flash.
- **DAQ event channel timing must be honest about jitter.** An event
  channel tied to a 10 ms task that sometimes overruns will deliver DAQ
  samples with irregular timestamps; a calibration engineer analyzing a
  control loop's step response needs to know the DAQ timestamp is the
  actual sample time, not the nominal period, or their tuning conclusions
  are built on a false timing assumption.
- **A2L/binary version mismatch has no protocol-level detection.**
  XCP has an `EPK` (EPROM identifier) field precisely to let a tool
  verify the connected ECU's software version matches the loaded A2L —
  skipping the EPK check (or not implementing it) is how "the gains I
  just set did nothing" bug reports happen, because the write landed on
  a stale address.

## Cheat sheet

| Term | Meaning |
|------|---------|
| XCP | Universal Measurement and Calibration Protocol (ASAM), address-based, transport-agnostic |
| A2L | ASAM MCD-2MC file mapping symbol names to memory addresses; must match the exact linked binary |
| DAQ | Data AcQuisition — continuous streamed measurement, tied to ECU event channels |
| MTA | Memory Transfer Address — the pointer XCP reads/writes are relative to, set by `SET_MTA` |
| EPK | EPROM identifier — lets a tool verify the ECU's software version matches its loaded A2L |
| Calibration page | RAM working copy of tunable constants, distinct from the committed flash copy |
| Transport | XCP-on-CAN, XCP-on-CAN FD, XCP-on-Ethernet — protocol is transport-independent |
| Common PC tools | ETAS INCA, Vector CANape |

## Exercise

Implement a minimal XCP slave on an S32K node and connect a real or
simulated calibration tool to it. (1) Implement `CONNECT`, `SET_MTA`,
`SHORT_UPLOAD`, and `DOWNLOAD` for a small set of global variables, and
hand-write a matching A2L fragment for each — confirm by inspecting your
linker map file that the addresses in the A2L exactly match the linked
binary. (2) Deliberately relink with one extra global declared before an
existing measurement variable, without regenerating the A2L, and confirm
(by address arithmetic, or on real hardware with a tool like `python-can`
issuing raw XCP-on-CAN frames) that the stale A2L now points at the wrong
variable — this demonstrates the exact silent-failure mode described
above. (3) Add an address-range whitelist check to your `DOWNLOAD`
handler so writes outside a defined calibration region are rejected, and
verify a write to a whitelisted address succeeds while one outside it is
rejected. (4) If a CAN interface and either INCA/CANape or an open XCP
tool (e.g. `pyxcp`) is available, connect to your slave and log a
measurement variable's DAQ stream while modifying it via calibration —
confirm end to end.
