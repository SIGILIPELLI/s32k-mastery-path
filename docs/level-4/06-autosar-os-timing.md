# AUTOSAR OS & Timing Analysis

Every module since Level 1 has assumed tasks and interrupts "just run
when they're supposed to." At ASIL D, that assumption needs proof, not
faith — a torque-arbitration task that occasionally misses its deadline
by 2ms under worst-case interrupt load is a defect, even if it passes
every functional test that happened not to trigger that load. **AUTOSAR
OS** is the standardized real-time operating system underneath the RTE
(Level 3 module 1), and this module covers its scheduling model plus the
timing analysis technique — **schedulability analysis** — that proves a
task set meets its deadlines before you ever run it on hardware.

## AUTOSAR OS task and interrupt model

```text
Interrupt Category 2 (ISR2)  -- highest priority, OS-managed, can call
                                 a subset of OS services (e.g. ActivateTask)
Task (preemptable, priority) -- scheduled by priority, can be preempted
                                 by a higher-priority task or any ISR
Task (non-preemptable/basic) -- runs to completion once started
Interrupt Category 1 (ISR1)  -- lowest OS overhead, cannot call OS
                                 services at all, fastest response
```

```c
/* AUTOSAR OS task declaration — configuration-driven like RTD's MCAL,
   not a raw RTOS task-create call */
TASK(DoorControl_10ms)
{
    DoorControl_MainFunction_10ms();  /* the runnable from Level 3 module 1 */
    TerminateTask();                    /* AUTOSAR OS: tasks must explicitly terminate */
}

/* Configured (via ARXML, generated into Os_Cfg.c) not hand-coded:
   priority, whether preemptable, stack size, which core (module 1's
   multi-core: a task is pinned to one specific core, never migrates) */
```

The **Basic Conformance Class** (BCC1-4) an AUTOSAR OS configuration
targets determines what's available: BCC1 tasks each activate once and
cannot be reactivated while still running (simplifying reasoning about
concurrent instances); higher classes allow more flexibility at the cost
of more complex worst-case timing reasoning.

## Schedulability: proving deadlines are met, not hoping

For a fixed-priority preemptive scheduler (which AUTOSAR OS is), the
standard tool is **Rate Monotonic Analysis (RMA)** and its generalization,
**Response Time Analysis (RTA)**:

```text
Response Time Analysis: worst-case response time R_i of task i,
accounting for every higher-priority task and interrupt that can
preempt it during its own execution window:

R_i = C_i + sum over all higher-priority tasks j:  ceil(R_i / T_j) * C_j

Where: C_i = task i's worst-case execution time (WCET)
       T_j = task j's period
       This is solved iteratively: start R_i = C_i, recompute until stable
```

```text
Example: three periodic tasks on one core
  Task            Period (T)   WCET (C)   Priority
  DoorControl     10 ms        2 ms       High
  LightingManager 20 ms        4 ms       Medium
  Diagnostics     50 ms        6 ms       Low

R_DoorControl = 2ms                              (nothing preempts the highest priority)
R_Lighting    = 4 + ceil(R/10)*2  -> converges to 6ms  (one DoorControl instance can preempt)
R_Diagnostics = 6 + ceil(R/10)*2 + ceil(R/20)*4  -> converges to 16ms (both can preempt)

All three R_i <= their T_i -> schedulable. If Diagnostics' WCET grew to
20ms, R_Diagnostics could exceed 50ms -> a missed deadline, computable
BEFORE it ever happens on hardware.
```

This is precisely why **WCET (Worst-Case Execution Time)** matters as a
measured, not estimated, number — an RTA calculation is only as trustworthy
as its WCET inputs, and WCET must account for cache effects, pipeline
stalls, and the S32K's own memory wait states under worst-case flash
access patterns, not just "how long it took in one test run."

## Priority inversion and how AUTOSAR OS prevents it

```text
Without protection:
  Low-priority task L holds a shared resource
  High-priority task H wants the same resource, blocks
  Medium-priority task M (unrelated) preempts L
  -> H is now effectively blocked by M, a LOWER priority task than H
     was ever supposed to be blocked by. Unbounded priority inversion.

AUTOSAR OS's Priority Ceiling Protocol (a resource's ceiling = highest
priority of any task that uses it):
  L, upon taking the resource, is temporarily raised to the resource's
  ceiling priority -> M cannot preempt L while L holds it -> H's
  maximum blocking time is bounded and calculable, not unbounded.
```

```c
/* AUTOSAR OS resource — configuration-driven priority ceiling, not a
   raw mutex with unspecified priority behavior */
GetResource(SharedFlashBufferResource);
WriteSharedBuffer(data);
ReleaseResource(SharedFlashBufferResource);
/* Between Get and Release, this task's effective priority is raised
   to the resource's configured ceiling — bounding blocking time is
   the entire reason this mechanism exists over a plain OS mutex */
```

## Automotive-MCU concerns

- **WCET measurement on real S32K hardware must include worst-case cache
  and flash-wait-state behavior, not best-case.** A function measured
  with warm instruction cache and zero flash wait states can have a
  meaningfully longer WCET on a cold cache after an interrupt-heavy
  preemption sequence — timing analysis tools (e.g. AbsInt aiT, or
  vendor-provided WCET estimation in S32 Design Studio) model this;
  hand-measured "it took X µs on the bench" numbers routinely underestimate.
- **A schedulability analysis is only valid for the exact task set and
  priorities it was computed for.** Adding one new low-priority
  diagnostic task late in a project, without rerunning RTA, can silently
  push a previously-schedulable high-priority task past its deadline —
  this is a common real-world regression source, and is exactly why
  timing budgets belong in the same traceable, reviewed artifact set as
  requirements (Level 4 module 2).
- **Interrupts are not accounted for by task-level RTA unless explicitly
  modeled as such.** A CAN RX ISR firing at a high rate under bus load
  consumes core time that every task's response time calculation must
  include — treating "interrupt overhead" as a rounding error rather than
  an explicit term in the RTA formula is a common source of an analysis
  that looks fine on paper and misses deadlines in the field under real
  bus traffic.
- **Priority ceiling protocol requires correct configuration, not just
  availability.** An AUTOSAR OS resource with a ceiling priority set
  incorrectly (e.g. below the highest priority task that actually uses
  it) reintroduces the exact unbounded priority inversion the mechanism
  exists to prevent — a subtle configuration bug, not a code bug, and
  one that schedulability tooling should be used to double-check.

## Cheat sheet

| Term | Meaning |
|------|---------|
| ISR Category 1/2 | Cat1: no OS service calls, fastest; Cat2: OS-managed, can call a subset of services |
| BCC1-4 | AUTOSAR OS Basic Conformance Classes — determine task reactivation/multiplicity rules |
| WCET | Worst-Case Execution Time — measured/analyzed, not assumed; must include cache/flash wait-state worst case |
| RTA / RMA | Response Time / Rate Monotonic Analysis — proves deadlines met via iterative formula |
| Priority Ceiling Protocol | AUTOSAR OS resource mechanism bounding priority inversion via temporary priority raise |
| Schedulability | A task set is schedulable if every task's calculated worst-case response time ≤ its deadline |
| Core pinning | AUTOSAR OS tasks are statically assigned to one core (module 1), never migrate at runtime |
| Regression risk | Adding/changing one task invalidates prior RTA results until recomputed |

## Exercise

Perform a schedulability analysis on the Level 3 body-controller's task
set. (1) Define at least 3 tasks with realistic periods and estimated
WCETs (measure them if you have hardware; estimate conservatively if
not) and assign priorities using rate-monotonic ordering (shorter period
= higher priority). (2) Run the Response Time Analysis formula by hand
for each task, iterating until each R_i converges, and confirm every
task meets its deadline — if one doesn't, adjust priorities or reduce a
WCET and recompute. (3) Add a new low-priority diagnostic task to the
set and rerun the full analysis — confirm whether it changes any
existing task's response time, and explain why or why not in terms of
the RTA formula. (4) Identify one shared resource in your design (e.g.
a flash write buffer used by both a safety task and a diagnostic task)
and specify its Priority Ceiling Protocol configuration — state which
task's priority becomes the ceiling and why.
