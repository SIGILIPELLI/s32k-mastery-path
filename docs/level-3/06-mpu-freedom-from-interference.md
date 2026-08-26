# MPU & Freedom from Interference

A body controller that runs door locks and airbag-adjacent seatbelt
pretensioner diagnostics on the same MCU has a problem the C compiler
cannot see: a bug in the door-lock task's array indexing must be
*physically incapable* of corrupting the pretensioner task's memory. This
property is called **freedom from interference (FFI)**, and it's an
ISO 26262 requirement whenever software of different ASIL levels — or
QM (non-safety) and safety-rated software — shares one MCU. The S32K's
**Memory Protection Unit (MPU)** is the hardware mechanism that enforces
it: memory regions with per-region, per-master access permissions, so a
lower-ASIL task's write to an out-of-bounds pointer faults immediately
instead of silently landing in a higher-ASIL task's data.

## What the MPU actually protects against

```text
Without MPU: one flat address space
  Task A (QM)        Task B (ASIL B)
  [.......buggy write.......]--------> lands in Task B's RAM, undetected

With MPU: regions with enforced permissions
  Task A (QM) region      | Task B (ASIL B) region
  [read/write, Task A only]|[read/write, Task B only]
  buggy write from A into B's region -> MPU fault, caught immediately
```

The MPU does not make buggy code correct — it makes an interference bug
**detectable at the moment it happens**, converting a silent data
corruption into an immediate, diagnosable fault. That conversion is
precisely what an ISO 26262 FFI argument needs: you cannot claim freedom
from interference by asserting "the code is careful," you need a
mechanism that catches the violation.

## S32K MPU region configuration

```c
/* S32K MPU: regions defined by start/end address + per-bus-master
   permission bits. Simplified illustrative layout. */
typedef struct {
    uint32_t start_addr;
    uint32_t end_addr;
    uint8_t  master_id;      /* which bus master this rule applies to (core, DMA, ...) */
    bool     read_enable;
    bool     write_enable;
    bool     execute_enable;
} mpu_region_cfg_t;

static const mpu_region_cfg_t mpu_regions[] = {
    /* QM task A: RW its own RAM, no access to Task B's region */
    { .start_addr = 0x1FFF0000u, .end_addr = 0x1FFF3FFFu,
      .master_id = MPU_MASTER_CORE, .read_enable = true,
      .write_enable = true, .execute_enable = false },

    /* ASIL-B task B: RW its own RAM only */
    { .start_addr = 0x1FFF4000u, .end_addr = 0x1FFF7FFFu,
      .master_id = MPU_MASTER_CORE, .read_enable = true,
      .write_enable = true, .execute_enable = false },

    /* Shared mailbox region: both tasks read, only Task B writes —
       enforces a single writer for cross-task communication */
    { .start_addr = 0x1FFF8000u, .end_addr = 0x1FFF80FFu,
      .master_id = MPU_MASTER_CORE, .read_enable = true,
      .write_enable = false, .execute_enable = false }, /* Task A's view */
};

void Mpu_Init(void)
{
    for (uint32_t i = 0u; i < (sizeof(mpu_regions)/sizeof(mpu_regions[0])); i++) {
        MPU->WORD[i][0] = mpu_regions[i].start_addr;
        MPU->WORD[i][1] = mpu_regions[i].end_addr;
        MPU->WORD[i][2] = Mpu_BuildPermissionWord(&mpu_regions[i]);
        MPU->WORD[i][3] |= MPU_WORD_VLD_MASK;  /* validate the region */
    }
    MPU->CESR |= MPU_CESR_VLD_MASK;  /* enable the MPU globally */
}
```

Critically, the S32K MPU's permissions are **per bus master**, not just
per core mode — a DMA channel is a separate master ID from the core, so
a DMA descriptor misconfigured to write outside its intended buffer is
caught by the same mechanism, which matters directly for module 8's
Ethernet DMA and any ADC/CAN DMA usage from Level 2.

## The fault handler is part of the safety mechanism

```c
void MPU_ISR(void)
{
    uint32_t err_detail = MPU->EAR[0]; /* error address register */
    uint8_t  err_master = (MPU->EDR[0] & MPU_EDR_EMN_MASK) >> MPU_EDR_EMN_SHIFT;

    Dem_ReportErrorStatus(DEM_EVENT_MPU_VIOLATION, DEM_EVENT_STATUS_FAILED);
    /* Do not attempt to "fix and continue" the faulting task — an MPU
       violation means that task's internal state is no longer trusted.
       The safe response is documented isolation, not silent recovery. */
    Task_ForceStop(err_master);
    EnterSafeStateIfSafetyRelevant(err_master);
}
```

The instinct to catch the fault and resume is exactly wrong for a safety
mechanism: an MPU violation proves the faulting task's own memory model
is already inconsistent, so "recovering" and continuing that task risks
running on corrupted state. The correct action is isolating the faulting
component and — if it was safety-relevant — transitioning the whole ECU
toward its defined safe state (module 5's SBC fault-pin path is often
the escalation target for a fault this severe).

## Automotive-MCU concerns

- **Stack overflow is the most common real-world MPU catch.** A guard
  region placed immediately past each task's stack, configured
  no-access, turns a stack overflow from silent heap/adjacent-stack
  corruption into an immediate, addressable fault — cheap to configure,
  and it catches a genuinely common class of embedded bug.
- **MPU region count is limited hardware, budget it like RAM.** The S32K
  MPU has a fixed number of regions (commonly 8-16 depending on part);
  a design with 20 logical partitions cannot get 20 independent MPU
  regions and needs a partitioning strategy — group by ASIL level and
  QM/non-QM boundary first, not by feature.
- **DMA masters must be included in the FFI argument, not assumed
  benign.** A DMA channel configured to move CAN RX data into a QM
  buffer, if its descriptor is corrupted (e.g. by the exact class of bug
  the MPU exists to catch elsewhere), can write anywhere its master ID
  is permitted to. Give DMA masters the same regional restrictions as
  the core, scoped to only the buffers each DMA channel legitimately
  needs.
- **MPU misconfiguration is itself a hazard, not just an inconvenience.**
  A region boundary off by one word can leave a byte of a "protected"
  buffer unprotected, or worse, deny legitimate access and cause a
  functional failure at the least convenient moment. Region boundaries
  belong in generated/reviewed configuration (module 2's RTD pattern),
  not ad hoc constants.

## Cheat sheet

| Term | Meaning |
|------|---------|
| FFI | Freedom from Interference — ISO 26262 requirement when mixed-ASIL software shares an MCU |
| MPU | Memory Protection Unit — hardware-enforced per-region, per-master access rules |
| Bus master | Core, DMA channel, or other initiator; MPU permissions apply per master ID |
| Region | Address range + permission set; S32K has a limited, fixed count |
| Guard region | No-access region placed past a stack to catch overflow immediately |
| MPU violation response | Isolate + report via DEM, never silently resume the faulting task |
| Relevant standard | ISO 26262-6 §7 (software architectural design, freedom from interference) |
| ASIL decomposition | Splitting a function across QM + safety partitions relies on FFI being provable |

## Exercise

Partition a Level 2-style multi-task application using the S32K MPU. (1)
Define at least three MPU regions: a QM task's RAM, a higher-integrity
task's RAM, and a shared read-mostly mailbox region with asymmetric
permissions (one writer, one reader). (2) Configure a guard region past
one task's stack and deliberately trigger a stack overflow (a runaway
recursive function is the simplest way) — confirm the MPU fault fires
before the overflow silently corrupts adjacent memory, and compare
against the same test with the guard region disabled. (3) Add a DMA
channel to your configuration (reuse a Level 2 CAN or ADC DMA setup) and
restrict its MPU permissions to only its intended destination buffer;
corrupt its descriptor deliberately and confirm the MPU catches the
resulting out-of-bounds write. (4) Write the fault ISR so it reports via
a DEM-style event log and forcibly halts only the faulting task, then
document — as your FFI argument — why each of your three original
regions cannot interfere with either of the others.
