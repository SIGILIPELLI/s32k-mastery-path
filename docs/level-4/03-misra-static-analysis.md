# MISRA C & Static Analysis in Practice

Module 2 listed MISRA C compliance as mandatory at ASIL D. This module
is the practical side: what MISRA rules actually catch, why several of
the "annoying" ones exist for real historical reasons, and how a
deviation gets formally justified instead of just silenced. **MISRA C**
(from the Motor Industry Software Reliability Association) is a subset
of the C language — not a style guide, a subset — designed to eliminate
constructs that are legal C but have caused real, expensive automotive
field failures.

## Rule categories

```text
Mandatory  — no deviation permitted, ever (very small set)
Required   — deviation permitted only with formal, documented justification
Advisory   — deviation permitted with lighter-weight justification
```

A **deviation** is not "ignore the warning" — it's a written record: which
rule, which specific line, why the violation is safe in this exact
context, and who approved it. An ASIL D safety case includes the
deviation log as evidence; a codebase with silenced warnings and no
deviation records is a compliance gap regardless of whether the code
itself is actually safe.

## Rules that matter most in this codebase's own conventions

```c
/* Rule 10.1/10.3 (essential type model) — implicit conversions between
   incompatible essential types are forbidden */
uint16_t raw_speed = 6000u;
float32_t speed_kph = raw_speed * 0.01f;   /* VIOLATION: implicit uint16->float */
float32_t speed_kph_ok = (float32_t)raw_speed * 0.01f;  /* explicit cast: compliant */
```

```c
/* Rule 8.13 — a pointer parameter that is not modified through should
   be declared const; catches "read-only" APIs that accidentally leave
   a write path open (relevant directly to Level 3 module 6's MPU
   read-only mailbox pattern — the C-level intent should match the
   MPU-level enforcement) */
Std_ReturnType ReadDoorStatus(const DoorStatus_t *status);  /* compliant */
Std_ReturnType ReadDoorStatus(DoorStatus_t *status);        /* VIOLATION if never written */
```

```c
/* Rule 17.7 — the return value of a non-void function must be used or
   explicitly discarded with (void). This is the exact rule that
   Level 2's UDS module was implicitly teaching by insisting every
   Rte_Read_*/Rte_Call_* return be checked */
(void)Rte_Write_DoorControl_PPort_DoorLockStatus(desired); /* explicit discard, compliant */
Rte_Write_DoorControl_PPort_DoorLockStatus(desired);        /* VIOLATION: silently ignored */
```

```c
/* Rule 21.3 — no dynamic memory allocation (malloc/free/calloc/realloc).
   Real reason: heap fragmentation over a 15-year vehicle life with
   unpredictable allocation patterns is a failure mode with no
   acceptable mitigation in a safety-relevant embedded system */
uint8_t rx_buffer[256];              /* compliant: static allocation */
uint8_t *rx_buffer = malloc(256);    /* VIOLATION */
```

```c
/* Rule 15.5 — a function should have a single point of exit. Real
   reason: multiple return points in a long function are a leading
   cause of a resource-cleanup or lock-release path being missed on
   one path but not another during a later edit */
Std_ReturnType Process(uint8_t *buf, uint16_t len)
{
    Std_ReturnType result = E_NOT_OK;
    if (buf != NULL) {
        if (len <= MAX_LEN) {
            DoWork(buf, len);
            result = E_OK;
        }
    }
    return result;  /* single exit point, compliant */
}
```

## Static analysis tool workflow

```text
1. Compile   -> catches syntax/type errors the language itself defines
2. MISRA analyzer (e.g. Polyspace, Parasoft, PC-lint, Coverity's MISRA
   module) -> flags rule violations against the chosen MISRA C version
3. Triage    -> each flagged violation: fix the code, OR file a deviation
4. Deviation record -> rule ID, file:line, justification, approver,
   stored alongside the code (often as a structured comment + a
   deviation database entry, not just a comment)
5. Re-run on every commit -> a violation count that silently creeps up
   over a project's life is exactly the failure mode this gate prevents
```

```c
/* A properly recorded deviation — visible in code AND in the project's
   deviation log, not just suppressed silently */
/* PRQA S 0310 1 -- Rule 11.3 deviation: this cast is required to access
   a memory-mapped peripheral register via a vendor-supplied header
   macro; verified safe because the target address is always correctly
   aligned by the linker script. Approved: J.Reviewer, DR-0042. */
volatile uint32_t *reg = (volatile uint32_t *)FLEXCAN0_BASE_ADDR;
```

## Automotive-MCU concerns

- **A tool's default rule set is a starting point, not a finished
  configuration.** Different MISRA versions (MISRA C:2004, C:2012,
  C:2012 Amendment 1-4) have materially different rule numbering and
  content — confirm which version your project's safety case cites
  before comparing violation counts across tools or teams, since "Rule
  17.7" means different things across editions in some corner cases.
- **Suppressing a warning at the tool level without a deviation record
  is a compliance gap even if the code is genuinely fine.** An assessor
  reviewing an ASIL D safety case checks for the deviation log's
  existence and completeness, not just for a clean tool report — a clean
  report with unrecorded suppressions is arguably worse than a report
  with an honestly documented violation count.
- **MISRA compliance does not guarantee correctness, and treating it as
  a substitute for review is a real project risk.** A function can be
  100% MISRA-compliant and still contain a logic bug — Rule 15.5's
  single-exit-point rule doesn't check whether the logic inside is
  right, only that its control flow shape avoids one specific class of
  maintenance hazard. Static analysis is one gate among several (Level 4
  module 2's independent review, unit test MC/DC), not a replacement for
  any of them.
- **Legacy code migration to MISRA compliance is a real, expensive,
  often-underestimated task.** A pre-existing Level 1/2-style codebase
  written without MISRA in mind commonly surfaces hundreds of violations
  on first analysis; budgeting this as "a quick lint pass" instead of a
  genuine engineering effort with prioritization (fix Mandatory first,
  triage Required, defer low-risk Advisory) is a common project-planning
  mistake.

## Cheat sheet

| Rule (MISRA C:2012) | Category | What it catches |
|----------------------|----------|-------------------|
| 10.1/10.3 | Required | Implicit conversions between incompatible essential types |
| 8.13 | Advisory | Pointer parameter should be `const` if never written through |
| 17.7 | Required | Discarding a non-void return value without explicit `(void)` |
| 21.3 | Required | Dynamic memory allocation (`malloc`/`free`/etc.) forbidden |
| 15.5 | Advisory | Function should have a single point of exit |
| 11.3 | Required | Casting between incompatible pointer types (e.g. raw register access) |
| Deviation category | Meaning |
|----------------------|---------|
| Mandatory | No deviation ever permitted |
| Required | Deviation only with formal, documented, approved justification |
| Advisory | Deviation with lighter justification, still recorded |

## Exercise

Run a MISRA-style review pass on your own Level 3 body-controller code
(by hand if no analyzer is available, or with a real static analysis
tool if one is). (1) Find or construct at least 3 genuine violations
from the rule list above in your existing code (or write short examples
if your code happens to already be clean) and fix each one, showing the
before/after. (2) Pick one violation you believe is a legitimate case for
a deviation rather than a fix (e.g. the register-access pointer cast
example) and write a properly formatted deviation record: rule, location,
justification, and what evidence would satisfy a reviewer that it's
safe. (3) Audit your codebase specifically for Rule 17.7 (discarded
return values) — this is the rule most directly connected to the "always
check `Std_ReturnType`" discipline from Level 3 module 1, and is worth
treating as a near-zero-tolerance check regardless of ASIL level. (4) If
a real static analyzer (Cppcheck with a MISRA addon, or a commercial
tool trial) is available, run it against your project and compare its
findings against what you found by hand — note any it caught that you
missed, and any it flagged that you'd argue is a false positive worth a
deviation.
