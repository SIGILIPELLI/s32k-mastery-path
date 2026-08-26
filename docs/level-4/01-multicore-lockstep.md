# Multi-Core S32K3 & Lockstep

Everything through Level 3 assumed one core. S32K3-family parts (unlike
S32K1) ship with **multiple Arm Cortex-M7 cores**, and at least one pair
of them is typically configured as a **lockstep core**: two physical
cores executing the identical instruction stream in near-perfect
synchrony, with a hardware comparator flagging any divergence within a
few clock cycles. This is the hardware mechanism that lets an S32K3-based
design claim **ASIL D** — the highest automotive integrity level — for
functions where a single silicon defect (a bit flip from cosmic-ray-
induced SEU, an internal short) must be caught before it can produce an
unsafe output, not just eventually detected.

## Lockstep vs. symmetric multi-core: different jobs

```text
Lockstep pair (safety)              Symmetric multi-core (performance)
─────────────────────                ──────────────────────────────────
Core 0 + Core 0-checker run          Core 0, Core 1, Core 2 run
the SAME code, same instant,         DIFFERENT code, independently,
comparator flags any divergence      each with its own workload
Purpose: catch a random hardware     Purpose: partition workload for
fault (SEU, transistor fault)        throughput or functional isolation
Software sees ONE logical core       Software sees N independent cores
```

An S32K3 typically provides one lockstep pair (appearing to software as
a single, higher-reliability core) alongside one or more independent
cores for non-lockstepped workloads — e.g. a lockstep core running the
ASIL-D torque-arbitration logic, with a separate core running QM
diagnostics, logging, or a communication stack.

## What the lockstep comparator actually catches

```text
Core 0 (main)  ---- executes instruction N ---->  result A
Core 0' (checker, delayed by a few cycles) ---->  result A'
Comparator: A == A'?
   Yes -> continue, transparent to software
   No  -> lockstep fault -> immediate reset / safe state, NOT a software trap
```

This is a **hardware fault detection mechanism, not a software one** —
your application code never sees a "lockstep mismatch" exception to
handle gracefully, because the premise of the mechanism is that the core
itself may no longer be trustworthy the instant a mismatch occurs. The
response is a forced reset or transition toward the safe state described
in Level 3 module 5, driven at the hardware/platform level.

```c
/* Application code configures WHAT happens on a lockstep fault via the
   platform's fault collection unit, but does not implement the
   comparison itself — that logic is fixed silicon */
void Fccu_ConfigureLockstepFaultResponse(void)
{
    /* FCCU: Fault Collection and Control Unit routes a lockstep
       mismatch to a defined reaction — here, immediate short-circuit
       reset, the strictest available response for ASIL-D functions */
    FCCU->CFG_FLT[FCCU_FAULT_LOCKSTEP] = FCCU_REACTION_SHORT_CIRCUIT_RESET;
    FCCU->CTRL |= FCCU_CTRL_OPR_MASK;   /* transition FCCU to operational */
}
```

## Inter-core communication for the non-lockstep cores

Where multiple independent cores genuinely run different workloads, they
need a defined, MPU-protected (Level 3 module 6) shared-memory channel —
never an assumption that "it's all one address space so any pointer
works":

```c
/* Shared mailbox in RAM both cores can address, MPU-restricted so
   only the intended core can write each half */
typedef struct {
    volatile uint32_t seq_number;   /* incremented by writer, read by reader for staleness check */
    volatile float32_t torque_request;
} intercore_mailbox_t;

/* Core 0 (producer) */
void Core0_PublishTorqueRequest(float32_t value)
{
    g_mailbox.torque_request = value;
    __DSB();                          /* ensure the write completes before... */
    g_mailbox.seq_number++;           /* ...the sequence number visibly updates */
}

/* Core 1 (consumer) */
bool Core1_ReadTorqueRequest(float32_t *out)
{
    uint32_t seq_before = g_mailbox.seq_number;
    __DSB();
    *out = g_mailbox.torque_request;
    __DSB();
    return (seq_before == g_mailbox.seq_number); /* reject if writer updated mid-read */
}
```

The sequence-number pattern exists because two independent cores reading
and writing shared RAM without any synchronization primitive can produce
a **torn read** — the reader observing half of an old value and half of
a new one, particularly for multi-word structures the core cannot write
atomically. `__DSB()` (Data Synchronization Barrier) ensures memory
ordering; it does not by itself prevent a torn read, which is why the
sequence check is still necessary.

## Automotive-MCU concerns

- **A lockstep core is exactly as fast as one core, not two.** The
  checker core does not contribute throughput — it exists purely for
  fault detection. Do not budget lockstep-pair performance as if it were
  a second independent core; the software-visible throughput is
  single-core.
- **Startup synchronization between lockstep cores is a real bring-up
  hazard.** The two physical cores must begin executing in a defined
  cycle-accurate relationship (a fixed delay offset, not simultaneous) for
  the comparator to align correctly; a boot sequence that doesn't follow
  the vendor's documented lockstep-enable procedure produces spurious
  mismatch faults that look exactly like real hardware defects during
  bring-up.
- **Not every fault is coverable by lockstep.** Lockstep catches faults
  that manifest as a divergence in the two cores' outputs — it does not
  catch a systematic design fault present identically in both cores'
  logic (a shared silicon design defect) nor a software bug, since both
  lockstep cores execute the identical, equally-buggy code. Lockstep is
  a hardware random-fault mitigation, not a software correctness
  mechanism — ISO 26262 Part 5's hardware metrics (SPFM/LFM) are what
  lockstep coverage feeds into, not the software safety case.
- **Inter-core mailboxes without a staleness check produce a specific,
  hard-to-reproduce bug class.** A consumer core that doesn't verify the
  sequence number can act on a torn read exactly once in a very long
  run — the kind of bug that survives unit testing and appears in the
  field under specific timing conditions. Always pair a multi-word
  shared structure with either a sequence-number check, a hardware
  semaphore peripheral if the part provides one, or a single-word atomic
  handoff.

## Cheat sheet

| Term | Meaning |
|------|---------|
| Lockstep pair | Two physical cores executing identically, hardware-compared; software sees one core |
| FCCU | Fault Collection and Control Unit — routes detected faults (including lockstep mismatch) to a configured reaction |
| SPFM / LFM | Single-Point Fault Metric / Latent Fault Metric — ISO 26262-5 hardware metrics lockstep coverage feeds |
| Torn read | Reading a multi-word shared value mid-write by another core; sequence numbers or atomics prevent acting on it |
| DSB | Data Synchronization Barrier — Arm instruction ensuring memory ordering, not atomicity by itself |
| Symmetric multi-core | Independent cores running different workloads, for throughput/partitioning, not fault detection |
| Reaction on lockstep fault | Immediate reset/safe-state; not a recoverable software exception |
| Relevant standard | ISO 26262-5 (hardware architectural metrics), -6 §7 (FFI, now across cores) |

## Exercise

Design (and, on real multi-core S32K3 hardware, implement) an inter-core
communication scheme for a two-core split: one lockstep core running a
safety-relevant control function, one independent core running
diagnostics/logging. (1) Define the FCCU reaction for a lockstep fault
explicitly — what physical response happens, and confirm it is
documented as "cannot be caught by application code," not as an
exception handler. (2) Implement the sequence-number mailbox pattern
above for at least one multi-word value crossing between the two cores,
and write a test that deliberately reads mid-write (by inserting a delay
on the reader side) to confirm the staleness check actually rejects the
torn value. (3) If real hardware is available, deliberately trigger a
documented lockstep self-test (most S32K3 parts provide a lockstep
built-in self-test, LBIST/MBIST-adjacent) during startup and confirm it
reports a pass before your application code begins running safety-
relevant logic. (4) Write a short note distinguishing, for your specific
design, which faults your lockstep pair covers (random hardware faults)
versus which it explicitly does not (a software bug present on both
cores identically) — this distinction is exactly what an ISO 26262
safety case reviewer will ask you to articulate.
