# Project — Diagnostic-Capable Body Controller

This project combines every module of Level 3 into one coherent design:
a body control module (BCM) that manages doors, lighting, and a window
lift, structured in AUTOSAR-style layers, running on a CAN FD network
described by a real DBC, backed by an SBC watchdog, MPU-partitioned by
integrity level, secure-booted, and reachable for diagnostics and
calibration through the gateway you'd find in a production vehicle
architecture. Nothing here is new protocol content — it is the
architecture-level integration that a real automotive software team does
after each subsystem works in isolation.

## System architecture

```text
┌───────────────────────────────────────────────────────────┐
│  Application SWCs (module 1)                                │
│  DoorControl | LightingManager | WindowLift | DiagManager    │
├───────────────────────────────────────────────────────────┤
│  RTE (generated glue)                                        │
├───────────────┬───────────────────────────────────────────┤
│  BSW Services  │  ECU Abstraction                             │
│  NvM, Dcm, Dem │  CanIf, PduR (routes Dcm<->UDS, XCP<->calib) │
├───────────────┴───────────────────────────────────────────┤
│  MCAL (RTD, module 2): Can, Port, Dio, Mcu, Fee, Lpuart      │
├───────────────────────────────────────────────────────────┤
│  MPU partitions (module 6): QM (lighting) | ASIL-B (doors)   │
├───────────────────────────────────────────────────────────┤
│  S32K3 + SBC (module 5, external watchdog + fault pin)       │
│  HSE (module 7: secure boot, signs every flashed image)      │
└───────────────────────────────────────────────────────────┘
      │ CAN FD (module 3)         │ LIN (module 4)
      ▼                            ▼
  Vehicle network,            Window motor,
  gateway to Ethernet          mirror actuator
  diagnostic bench (module 8)
```

## Partitioning by integrity level

Door lock control interacts with occupant safety (an unintended unlock
above a speed threshold, or a lock engaging on a child in a doorway, are
real hazard scenarios); lighting control does not carry the same
consequence. This project treats `DoorControl` as the higher-integrity
partition and `LightingManager` as QM, enforced by the MPU from module 6:

```c
/* Partition boundary enforced identically to module 6's pattern —
   DoorControl's RAM region and LightingManager's RAM region are
   mutually inaccessible; a bug in one cannot corrupt the other */
static const mpu_region_cfg_t bcm_mpu_regions[] = {
    { .start_addr = DOORCTRL_RAM_BASE, .end_addr = DOORCTRL_RAM_END,
      .master_id = MPU_MASTER_CORE, .read_enable = true,
      .write_enable = true, .execute_enable = false },
    { .start_addr = LIGHTING_RAM_BASE, .end_addr = LIGHTING_RAM_END,
      .master_id = MPU_MASTER_CORE, .read_enable = true,
      .write_enable = true, .execute_enable = false },
    { .start_addr = SHARED_MAILBOX_BASE, .end_addr = SHARED_MAILBOX_END,
      .master_id = MPU_MASTER_CORE, .read_enable = true,
      .write_enable = false, .execute_enable = false }, /* Lighting's view: read-only */
};
```

`DoorControl` publishes vehicle-speed-gated lock state into the shared
mailbox; `LightingManager` (QM) can read it (e.g. to flash hazards on
lock/unlock) but cannot write it — the same single-writer discipline
from module 6, now applied for a concrete reason instead of an abstract
example.

## The CAN FD network (DBC-defined)

```text
BO_ 1024 BCM_DoorStatus: 8 BCM
 SG_ FL_Door_Locked  : 0|1@1+  (1,0)    [0|1]      ""      Gateway
 SG_ FR_Door_Locked  : 1|1@1+  (1,0)    [0|1]      ""      Gateway
 SG_ Vehicle_Speed   : 8|16@1+ (0.01,0) [0|327.67] "km/h"  BCM

BO_ 1025 BCM_LightingStatus: 4 BCM
 SG_ Headlamp_State  : 0|2@1+  (1,0)    [0|3]      ""      Gateway
 SG_ Hazard_Active   : 2|1@1+  (1,0)    [0|1]      ""      Gateway

BO_ 1280 BCM_WindowCmd: 4 Gateway
 SG_ Window_Position_Req : 0|8@1+ (1,0) [0|100]    "%"     BCM
```

`Vehicle_Speed` is the interlock signal — `DoorControl`'s auto-lock logic
from module 1 reads it, and the DBC formalizes its scale exactly as
module 3 requires: no hand-decoded bit offsets anywhere in application
code, generated pack/unpack functions only.

## Diagnostics, secure flashing, and calibration coexist on one gateway

```c
/* PduR-style routing: the same CAN FD interface carries UDS diagnostic
   traffic (0x7E0/0x7E8, Level 2) and, in dev builds only, XCP calibration
   traffic (module 9) on separate CAN IDs — production builds compile
   XCP out entirely per module 9's concern about remote memory write */
void PduR_RxIndication(uint32_t can_id, const uint8_t *data, uint8_t len)
{
    if (can_id == UDS_REQUEST_ID) {
        Dcm_RxIndication(data, len);
    }
#if (XCP_ENABLED == STD_ON)  /* never STD_ON in a production build */
    else if (can_id == XCP_REQUEST_ID) {
        Xcp_ProcessCommand(data, len);
    }
#endif
    else {
        CanIf_RxIndication(can_id, data, len); /* normal signal traffic */
    }
}
```

Reflashing this BCM in the field goes through Level 2's UDS bootloader,
now gated by module 7's HSE signature check before any image commits —
the same UDS `0x34`/`0x36`/`0x37` sequence, with a mandatory verify step
inserted before flash-commit that was absent in the Level 2 version.

## Safety mechanisms tying it together

- **SBC windowed watchdog (module 5)** services on the 10 ms scheduler
  tick; a window violation or an unexpected MPU fault escalation both
  route to the SBC fault pin path, which is the one mechanism trusted to
  work even if the application core has locked up entirely.
- **MPU violation handling (module 6)** on `DoorControl`'s partition
  forces that task to a halted state and reports via `Dem`, rather than
  attempting recovery — consistent with the "never resume a task whose
  memory model is already inconsistent" rule from that module.
- **Secure boot (module 7)** ensures the image running this whole stack
  was verified by the HSE before the application core ever executed a
  single instruction of it.

## Cheat sheet

| Layer | Module | Role in this project |
|-------|--------|------------------------|
| Application/RTE | 1 | `DoorControl`, `LightingManager`, `WindowLift` SWCs |
| MCAL | 2 | RTD `Can`, `Port`, `Fee` drivers under the SWCs |
| Network | 3 | DBC-defined CAN FD frames for door/light/window signals |
| LIN | 4 | Window motor / mirror actuator as a LIN slave off the BCM |
| Watchdog | 5 | SBC windowed watchdog + fault pin, independent of the app core |
| Partitioning | 6 | MPU: `DoorControl` (higher integrity) vs `LightingManager` (QM) |
| Secure boot | 7 | HSE-verified image before application core release |
| Gateway | 8 | Optional Ethernet bridge to a diagnostic/test bench |
| Calibration | 9 | XCP in dev builds only, compiled out for production |

## Stretch goals

Extend the project past the baseline design. (1) Add a fourth SWC,
`SeatMemory`, storing calibratable seat positions in an XCP-writable RAM
page (module 9) that a bench tool can tune live, while keeping it fully
isolated by the MPU from `DoorControl`'s partition. (2) Implement the
Ethernet gateway path from module 8: bridge `BCM_DoorStatus` onto a
SOME/IP message on an Ethernet segment, and write the threat-boundary
note module 8 asked for — is door status safety-relevant enough to need
SecOC-style authentication before it crosses that gateway? (3) Simulate
an MPU violation in `LightingManager` (a deliberate out-of-bounds write)
under load, and confirm on a scope or via UART trace that the SBC
watchdog window is unaffected — proving your two safety mechanisms are
actually independent, not just independently documented. (4) Write the
anti-rollback design from module 7 concretely for this BCM: define the
version-counter storage location and the exact check your secure boot
sequence performs before accepting an otherwise-validly-signed image.
