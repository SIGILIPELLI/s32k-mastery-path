# Flash, FlexNVM & EEPROM Emulation

Some data has to survive a power cycle: end-of-line calibration, the
vehicle identification number, learned adaptation values, odometer-like
counters, and diagnostic trouble codes that a workshop will read out three
months later. The S32K has no separate EEPROM die — it has **program
flash**, a configurable **FlexNVM** region, and a small **FlexRAM** window
that the flash controller can back with FlexNVM to emulate EEPROM. Getting
this right is unglamorous and unforgiving: flash wears out, writes take
milliseconds, and a power loss halfway through one is a real event in a
vehicle, not a theoretical one.

## The S32K1 flash architecture

On an S32K144:

| Region | Size | Sector size | Purpose |
|--------|------|-------------|---------|
| **P-Flash** (program flash) | 512 KB | 4 KB | Code and constants; execute-in-place |
| **FlexNVM** (D-Flash) | 64 KB | 2 KB | Data storage, EEPROM backup, or CSEc keys |
| **FlexRAM** | 4 KB | — | Plain RAM *or* the EEPROM emulation window |

The controller is **FTFC**, driven through a command interface: load a
command and its parameters into `FCCOB0…FCCOBB`, clear `FSTAT[CCIF]` to
launch it, and poll `CCIF` until it sets again. Error bits live in the
same register — `ACCERR` (illegal command or alignment), `FPVIOL`
(protection violation), `MGSTAT0` (verify failure). The S32 SDK wraps all
of it, but knowing the sequence matters when you are staring at a debugger.

The **program granularity is a phrase — 8 bytes, 8-byte aligned**. You
cannot program a single byte, and you cannot program the same phrase twice
without erasing it first. Erase granularity is a whole sector.

## Program and erase from the application

```c
#include "flash_driver.h"

static flash_ssd_config_t g_ssd;

/* The driver calls this during long operations so you can feed the
   watchdog — a sector erase takes milliseconds.                    */
static void flash_wait_cb(void)
{
    wdog_feed();
}

status_t nvm_init(void)
{
    static const flash_user_config_t userCfg = {
        .PFlashBase  = 0x00000000u,
        .PFlashSize  = 0x00080000u,      /* 512 KB */
        .DFlashBase  = 0x10000000u,      /* FlexNVM window */
        .EERAMBase   = 0x14000000u,      /* FlexRAM window */
        .CallBack    = (flash_callback_t)flash_wait_cb,
    };
    return FLASH_DRV_Init(&userCfg, &g_ssd);
}

status_t nvm_write_block(uint32_t addr, const uint8_t *data, uint32_t len)
{
    status_t st;

    /* Both address and length must respect the 8-byte phrase rule */
    if (((addr % 8u) != 0u) || ((len % 8u) != 0u)) {
        return STATUS_ERROR;
    }

    st = FLASH_DRV_EraseSector(&g_ssd, addr, FEATURE_FLS_DF_BLOCK_SECTOR_SIZE);
    if (st != STATUS_SUCCESS) { return st; }

    st = FLASH_DRV_Program(&g_ssd, addr, len, data);
    if (st != STATUS_SUCCESS) { return st; }

    /* Verify at the tighter user margin, not just "it reads back OK" */
    uint32_t failAddr = 0u;
    return FLASH_DRV_ProgramCheck(&g_ssd, addr, len, data, &failAddr, 1u);
}
```

Three constraints that are easy to violate and hard to debug:

- **You cannot fetch instructions from a flash block while erasing or
  programming it.** Reads of that block return undefined data or stall.
  The SDK provides RAM-section macros for the command-launch routine; on a
  part where P-Flash and D-Flash are separate blocks you can run from
  P-Flash while writing D-Flash, which is exactly why data usually lives
  in FlexNVM.
- **Interrupts must not execute from the block being written.** If your
  vector table and ISRs are in P-Flash and you are erasing P-Flash,
  disable interrupts around the operation — or place the handler in RAM.
- **The watchdog keeps counting.** A sector erase can take several
  milliseconds. That is what the SDK's callback exists for; wire it up on
  day one rather than after the first mysterious reset.

## FlexNVM as emulated EEPROM

Emulated EEPROM (EEE) works like this: a slice of FlexNVM is set aside as
**EEPROM backup**, and a slice of FlexRAM becomes the **EEE window**.
Writes to the FlexRAM window are byte- or word-granular and take effect
immediately in RAM; the flash controller transparently journals them into
the backup region and restores them at reset. You get EEPROM semantics —
write a byte, it persists — on top of sector-erasable flash.

The partition is set once with `FLASH_DRV_DEFlashPartition`, and this is
**not** a routine operation:

```c
/* Partition FlexNVM: EEE window size and how much D-Flash backs it.
   The code values come from the reference manual's partition tables. */
status_t nvm_partition_once(void)
{
    if (g_ssd.EEESize != 0u) {
        return STATUS_SUCCESS;        /* already partitioned — leave it */
    }
    return FLASH_DRV_DEFlashPartition(&g_ssd,
                                      EEE_DATA_SIZE_CODE,
                                      DE_PARTITION_CODE,
                                      0x00u,      /* no CSEc keys      */
                                      false,      /* SFE               */
                                      true);      /* load EEE data     */
}

status_t nvm_eee_write(uint32_t offset, const uint8_t *data, uint32_t len)
{
    return FLASH_DRV_EEEWrite(&g_ssd, g_ssd.EERAMBase + offset, len, data);
}
```

!!! warning "Partitioning is effectively one-way in the field"
    Repartitioning erases FlexNVM and, on a device with security enabled,
    can require a full mass erase. Partition at end-of-line production or
    in the bootloader — never in application code that might run on a
    part that is already in service with customer data on it.

## Wear, and how to plan for it

Flash cells tolerate a finite number of program/erase cycles — for
automotive parts, typically on the order of 10³ for program flash and 10⁴
for the data flash used as EEPROM backup, with the exact guaranteed figure
and its temperature/retention conditions in your part's data sheet. Read
that table; do not carry a number from another MCU family in your head.

EEE multiplies effective endurance because the controller rotates records
across the whole backup region: a small EEE window backed by a large
FlexNVM partition survives far more writes than the raw cell endurance
suggests, roughly in proportion to the backup-to-window ratio. That ratio
is the design knob, and the reference manual gives the endurance equation
for it.

The arithmetic you actually need is a simple budget:

```text
writes per second × seconds per driving hour × hours over vehicle life
                              ≤ endurance
```

A value written once per second over a 8000-hour vehicle life is 28.8
million writes. No flash cell survives that, so the design must change,
not the cell. The standard answers:

- **Write only on change**, with a deadband. An odometer that stores every
  metre is a defect; one that stores every 100 m is a design.
- **Shadow in RAM, flush on a trigger** — ignition off, a UDS request, or
  a low-voltage warning from the PMC. RAM absorbs the churn; flash sees
  one write.
- **Rotate records yourself** if you are not using EEE: a log-structured
  region where each record carries a monotonically increasing sequence
  number and a CRC. On boot, scan for the highest valid sequence number.
  This also gives you atomicity for free — an interrupted write leaves the
  *previous* record intact and valid.

```c
typedef struct {
    uint32_t sequence;      /* increments on every write            */
    uint16_t length;
    uint16_t reserved;
    uint8_t  payload[24];
    uint32_t crc32;         /* over sequence..payload               */
} nvm_record_t;             /* 8-byte multiple — matches the phrase */
```

## Automotive concerns

- **Power loss during a write is a designed-for event**, not an accident.
  Cranking drops the battery rail; a technician pulls a connector. Any NVM
  scheme must have a defined answer to "what if power fails exactly here?"
  — record rotation with a CRC gives you one; a single in-place struct
  does not.
- **Brown-out is worse than power loss.** A sagging rail can leave the
  flash controller executing with marginal voltage. Enable the PMC's
  low-voltage detect and refuse to start a write below a defined
  threshold; the LVD reset path from module 9 handles the rest.
- **Protect what must never be erased.** `FPROT0–3` protect P-Flash
  regions, `FDPROT` the D-Flash, and `FEPROT` the EEPROM window. A
  bootloader (module 8) protects its own sectors so that a buggy
  application physically cannot erase the path back to recovery.
- **Every stored value needs a CRC and a safe default.** Corrupt
  calibration must fall back to a known-good value and set a DTC, never be
  used as-is. This is module 9's CRC rule applied to persistence.
- **Count your writes in the field.** Store a lifetime write counter
  alongside the data. When a field return arrives, "this part has been
  written 4.2 million times" ends the investigation in five minutes.
- **DTC storage is bursty.** A fault that toggles rapidly can generate
  thousands of DTC updates in a minute. Debounce and coalesce before the
  data reaches flash — the aging/confirmation logic in the diagnostic
  standard exists partly for this reason.

## Cheat sheet

| Item | Notes |
|------|-------|
| Controller | **FTFC**; commands via `FCCOB0…B`, launched by clearing `FSTAT[CCIF]` |
| Error bits | `FSTAT`: `ACCERR` (bad command/alignment), `FPVIOL` (protection), `MGSTAT0` (verify) |
| S32K144 sizes | P-Flash 512 KB / 4 KB sectors · FlexNVM 64 KB / 2 KB sectors · FlexRAM 4 KB |
| Program unit | 8-byte phrase, 8-byte aligned — no byte writes, no rewriting a phrase |
| Erase unit | One sector |
| Read-while-write | Cannot fetch from the block being written — data in FlexNVM, code in P-Flash |
| Watchdog | Wire `flash_user_config_t.CallBack` to feed during long operations |
| SDK calls | `FLASH_DRV_Init`, `EraseSector`, `Program`, `ProgramCheck`, `EEEWrite`, `DEFlashPartition` |
| EEE | FlexRAM window backed by FlexNVM; byte-granular writes, journaled by hardware |
| Partitioning | `FLASH_DRV_DEFlashPartition` — end-of-line or bootloader only |
| Endurance | ~10³ P-Flash / ~10⁴ D-Flash cycles order-of-magnitude — confirm in the data sheet |
| Wear strategy | Write on change + deadband · RAM shadow flushed on a trigger · rotating CRC'd records |
| Protection | `FPROT0–3` (P-Flash), `FDPROT` (D-Flash), `FEPROT` (EEPROM window) |

## Exercise

Design and implement a non-volatile store that you would defend in a
design review. (1) List the data your capstone node must persist —
calibration, a fault memory, a run-hour counter — and for each one write
its size, its expected write frequency, and its behaviour if the stored
copy is corrupt. (2) Compute the lifetime write count for the most
frequently written item and compare it against a plausible endurance
figure; if it does not fit, redesign it with a deadband or a RAM shadow
and show the new number. (3) Implement the rotating-record scheme above in
FlexNVM: `nvm_store()` writes the next record with an incremented
sequence and CRC, `nvm_load()` scans on boot for the highest valid
sequence, and a corrupt or absent record yields the safe default plus a
DTC. (4) Test power-loss atomicity properly — cut power during a write
(a switch on the supply, twenty attempts) and confirm the node always
boots with either the old record or the new one, never a mixture. (5) Wire
the watchdog callback and prove it by measuring how long a sector erase
actually takes on your part. Without hardware, do steps (1)–(3) against a
file-backed stub on your PC and simulate the power cut by truncating the
write mid-record; the recovery logic is identical, and it is the part that
matters.
