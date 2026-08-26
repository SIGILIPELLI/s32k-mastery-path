# Capstone — Production-Grade Zone ECU

This capstone combines the entire course. A **zone ECU** is the
architecture pattern much of the industry is moving toward: instead of
one ECU per function (a separate box for doors, another for lighting,
another for seats), one physically-local "zone" controller aggregates
many nearby functions, running mixed-ASIL software on a multi-core
S32K3, gatewaying to a vehicle-wide Ethernet backbone, secured,
OTA-updatable, and manufactured at automotive volume. Building this
design end-to-end — even on paper where hardware access runs out — is
the exercise that proves every module in this course connects into one
coherent system, not ten isolated topics.

## Zone ECU scope

```text
Functions aggregated in this zone (front-left):
  - Door lock/unlock + window lift (from the Level 3 project)
  - Exterior lighting (turn signal, position lamp)
  - Seat position memory (calibratable via XCP)
  - Wheel-adjacent sensor aggregation (relayed toward ADAS/gateway)
```

## Full architecture

```text
┌────────────────────────────────────────────────────────────────┐
│  S32K3, multi-core (module 1)                                     │
│  Core 0 (lockstep): DoorControl (ASIL B), SeatMemory boundary check│
│  Core 1: LightingManager (QM), diagnostics, XCP calibration server │
│  MPU (Level 3 mod. 6) partitions Core 0's ASIL-B RAM from Core 1's │
├────────────────────────────────────────────────────────────────┤
│  AUTOSAR OS (module 6): RTA-verified schedulable task set          │
│  Priority ceiling protocol on shared flash-write resource           │
├────────────────────────────────────────────────────────────────┤
│  HSE (Level 3 mod. 7): secure boot, signs OTA images (module 4)    │
│  boot_info dual-bank record, anti-rollback counter                  │
├────────────────────────────────────────────────────────────────┤
│  SBC (Level 3 mod. 5): windowed watchdog, independent fault pin     │
├────────────────────────────────────────────────────────────────┤
│  CAN FD (Level 3 mod. 3) — SecOC-authenticated door/lock signals    │
│  (module 5), unauthenticated lighting signals (TARA-justified)      │
│  LIN (Level 3 mod. 4) — window motor, seat motor slaves               │
│  Ethernet gateway (Level 3 mod. 8) — relays zone status to a          │
│  central vehicle computer over SOME/IP                               │
└────────────────────────────────────────────────────────────────┘
```

## OTA + secure boot + anti-rollback, combined

```c
/* The full activation gate for this capstone: EVERY check from
   Level 3 module 7 and Level 4 module 4 applied together, because a
   zone ECU aggregating this many functions is exactly the kind of
   high-value target a real OTA activation gate must not shortcut */
Std_ReturnType ZoneEcu_ActivateOtaImage(const ota_image_t *img)
{
    if (Hse_VerifyImage(&img->verify_req) != HSE_STATUS_OK) {
        Dem_ReportErrorStatus(DEM_EVENT_SECURE_BOOT_FAIL, DEM_EVENT_STATUS_FAILED);
        return E_NOT_OK;
    }
    if (img->version <= boot_info.version_counter) {
        Dem_ReportErrorStatus(DEM_EVENT_ANTI_ROLLBACK_REJECT, DEM_EVENT_STATUS_FAILED);
        return E_NOT_OK;
    }
    if (Adc_ReadVehicleSpeed_Kph() > 0u) {
        return E_NOT_OK;  /* precondition: never activate while driving */
    }

    boot_info_t new_info = boot_info;
    new_info.active_bank = 1u - boot_info.active_bank;
    new_info.version_counter = img->version;
    Fee_Write(BOOT_INFO_BLOCK_ID, (const uint8_t *)&new_info, sizeof(new_info));
    return E_OK;
}
```

## MPU + AUTOSAR OS: partitioning matches priority

```c
/* Because DoorControl runs on the lockstep core (module 1) and
   LightingManager on the independent core, the MPU partitioning from
   Level 3 module 6 now also serves as physical core separation — a
   stronger guarantee than same-core MPU regions alone, since a
   lockstep fault on Core 0 cannot corrupt Core 1's memory even in the
   pathological case the MPU mechanism itself somehow failed */
static const mpu_region_cfg_t zone_mpu_regions[] = {
    { .start_addr = DOORCTRL_RAM_BASE, .end_addr = DOORCTRL_RAM_END,
      .master_id = MPU_MASTER_CORE0, .read_enable = true,
      .write_enable = true, .execute_enable = false },
    { .start_addr = SEATMEM_CAL_RAM_BASE, .end_addr = SEATMEM_CAL_RAM_END,
      .master_id = MPU_MASTER_CORE1, .read_enable = true,
      .write_enable = true, .execute_enable = false },  /* XCP-writable, module 9 Lvl 3 */
};
```

## Manufacturing and test, end to end

The zone ECU's EOL test (Level 4 module 9) must exercise every safety
mechanism this capstone assembles: SecOC key provisioning, HSE root key
burn, watchdog self-test, MPU-partition verification (confirm Core 1
genuinely cannot write into Core 0's region on the manufactured silicon,
not just in the design), and debug lockdown — all within a manufacturing
line's cycle-time budget, which is exactly the tension module 9
described between thoroughness and cost.

## Cheat sheet — full course map

| Concern | Level 3 module | Level 4 module |
|---------|------------------|-------------------|
| Layered architecture | 1 (AUTOSAR Classic) | 6 (OS/timing on top of it) |
| Driver configuration | 2 (RTD/MCAL) | 8 (hardware it configures) |
| Networking | 3 (CAN FD/DBC), 4 (LIN), 8 (Ethernet) | 5 (SecOC on top of it) |
| Safety mechanisms | 5 (SBC watchdog), 6 (MPU) | 1 (lockstep), 2 (ASIL process) |
| Security | 7 (secure boot/HSE) | 4 (OTA), 5 (SecOC), 9 (key provisioning) |
| Tooling/process | — | 2 (ASIL workflow), 3 (MISRA), 7 (HIL/CI) |
| Field/manufacturing | — | 4 (OTA rollback), 9 (EOL test) |

## Stretch goals

(1) Extend the zone concept to a second zone (e.g. front-right, mirroring
this one) and design the vehicle-wide gateway architecture connecting
both zones plus a central compute module over Ethernet — specify which
signals cross each gateway and, per Level 3 module 8's threat-boundary
discipline, which need SecOC authentication at the zone boundary. (2)
Write a complete HARA-to-code traceability chain (Level 4 module 2) for
the seat memory function's XCP-writable calibration page: a hazard
(unexpected seat movement while occupied), through FSR/TSR/SSR, down to
the MPU region and address-range whitelist check that implements it. (3)
Perform a full Response Time Analysis (module 6) across all tasks on
both cores of this zone ECU, including the AUTOSAR OS priority ceiling
protocol resource for the shared flash-write path used by both
`DoorControl`'s NVM writes and the OTA bootloader's image writes — show
they cannot deadlock or unboundedly block each other. (4) Design the
full EOL test sequence and CI pipeline (Level 4 modules 7 and 9) for
this capstone as a single document: every gate from build through
HIL-based fault injection through manufacturing key provisioning, in
the order each must run and why that order is load-bearing.
