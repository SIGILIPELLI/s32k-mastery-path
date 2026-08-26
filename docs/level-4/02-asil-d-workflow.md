# ASIL-D Development Workflow

Every mechanism from Level 3 — MPU partitioning, secure boot, SBC
watchdogs, lockstep — exists to satisfy requirements from a specific
standard: **ISO 26262**, Road vehicles — Functional safety. This module
covers the process wrapped around those mechanisms: how a requirement
becomes a hazard analysis, how a hazard analysis assigns an **ASIL**, and
what changes in your day-to-day workflow once a component is rated
ASIL D versus QM. This is process content, not new C code — and it's
exactly the content a real automotive engineering role expects you to
know cold before touching a body-controller or powertrain project.

## Where ASIL comes from: HARA

An **ASIL (Automotive Safety Integrity Level)** is not assigned to a
component by a manager's judgment call — it is derived from a **Hazard
Analysis and Risk Assessment (HARA)**, defined in ISO 26262-3, using
three factors:

```text
ASIL = f(Severity, Exposure, Controllability)

Severity (S0-S3):        How bad is the injury if the hazard occurs?
Exposure (E0-E4):        How often is the vehicle in the situation
                          where this hazard could occur?
Controllability (C0-C3): Can the driver/situation mitigate it?

Higher S + higher E + lower C  ->  higher ASIL
```

```text
Example: unintended door unlock while driving above 30 km/h
  Severity: S2 (moderate-to-severe injury possible, e.g. occupant fall risk)
  Exposure: E4 (driving above 30 km/h is a very common situation)
  Controllability: C2 (driver has limited ability to prevent consequence
                        once unlock occurs)
  -> Combination maps to ASIL B or C depending on the exact HARA table entry
```

```text
Example: airbag fails to deploy in a qualifying crash
  Severity: S3 (life-threatening/fatal)
  Exposure: E4 (crash conditions, while individually rare, are
                categorized at the vehicle-population level, not per-trip)
  Controllability: C3 (driver/occupant has no ability to control the outcome)
  -> ASIL D
```

**QM (Quality Managed)** is the fifth outcome — a HARA can conclude a
hazard doesn't warrant ASIL treatment at all, in which case standard
quality-managed development (still rigorous, but without ISO 26262's
additional process burden) applies. The point worth internalizing: ASIL
is a property of a *hazard*, decomposed down to requirements and then to
components — not an adjective you attach to "this ECU" as a whole. A
single body controller commonly contains QM, ASIL B, and ASIL D
functions simultaneously, which is exactly why Level 3 module 6's MPU
partitioning exists — to let mixed-ASIL software coexist safely on one
MCU.

## What changes at ASIL D versus QM

| Aspect | QM | ASIL D |
|--------|----|--------|
| Requirements traceability | Recommended | Mandatory, bidirectional, tool-supported |
| Unit testing | Coverage targets, team's choice | MC/DC (Modified Condition/Decision Coverage) required |
| Static analysis | Recommended | MISRA C mandatory, deviations formally justified (Level 4 module 3) |
| Code review | Standard peer review | Independent reviewer, documented checklist |
| Tool qualification | Not required | Compiler/static-analyzer must be qualified per ISO 26262-8 §11 |
| Hardware metrics | Not applicable | SPFM ≥ 99%, LFM ≥ 90% (typical ASIL D targets from -5) |
| Independence | Same team can design+review | Design and safety assessment roles must be organizationally independent |

**MC/DC** is the one line worth expanding: it requires that every
condition within a decision has been shown, independently, to affect
that decision's outcome — a stricter bar than branch coverage, which
only requires each branch taken at least once.

```c
/* Branch coverage is satisfied by 2 test cases; MC/DC needs more,
   because it must show EACH condition independently affects the result */
if ((speed > 30) && (door_open) && !(seatbelt_fastened)) {
    TriggerWarning();
}
/* MC/DC requires test vectors where flipping ONE condition at a time
   (holding the others fixed) flips the overall decision — for 3
   conditions in a simple AND, that's 4 carefully chosen vectors,
   not 2^3 = 8 exhaustive ones, but each is individually justified */
```

## Requirements traceability in practice

```text
Vehicle-level hazard (HARA)
   -> Functional Safety Requirement (FSR), ISO 26262-3
      -> Technical Safety Requirement (TSR), ISO 26262-4
         -> Software Safety Requirement (SSR), ISO 26262-6
            -> Unit-level requirement
               -> Test case (traced back up the chain)
```

Every line of ASIL-D-rated code should be traceable up to a specific
SSR, and every SSR should be traceable to a test case that verifies it.
Tooling (e.g. a requirements management tool integrated with your test
framework) exists specifically because maintaining this by hand across a
project with thousands of requirements is not realistically sustainable
otherwise — and an assessor will ask to see the trace, not just the code.

## Automotive-MCU concerns

- **A HARA decision has real, sometimes counter-intuitive severity
  outcomes.** A hazard's severity is about the worst plausible outcome,
  not the typical one — a window-lift pinch hazard rates higher severity
  than intuition might suggest because the worst case (a child's limb)
  is genuinely severe, even though it's rare. Do not let "this rarely
  goes wrong badly" reasoning substitute for the standard's defined
  severity/exposure/controllability framework.
- **ASIL decomposition is a real technique, not a shortcut.** A single
  ASIL D requirement can be decomposed into two independent ASIL B(D)
  paths (e.g. two independently-designed monitoring channels) whose
  combined failure probability satisfies the original ASIL D target —
  this is exactly why Level 3's MPU-enforced freedom from interference
  matters: decomposition only counts if the two paths are provably
  independent, which an MPU violation-catching mechanism helps
  demonstrate.
- **Tool qualification is not paperwork theater.** If your MISRA static
  analyzer (module 3) has a false-negative bug that lets a genuine
  violation through undetected, and you relied on that tool as your only
  gate for an ASIL D component, that is a real process gap an assessor
  will flag — ISO 26262-8 §11's tool confidence level (TCL) assessment
  exists to make you evaluate this explicitly, not assume tools are
  infallible.
- **Independence requirements affect team structure, not just
  documents.** The person who designs an ASIL D module's algorithm
  should not be the sole reviewer signing off its safety case — this is
  an organizational control, and a small team working an ASIL D project
  needs to plan for it (e.g. cross-team review) well before a milestone
  deadline forces a shortcut.

## Cheat sheet

| Term | Meaning |
|------|---------|
| HARA | Hazard Analysis and Risk Assessment — ISO 26262-3, derives ASIL from S/E/C |
| S / E / C | Severity / Exposure / Controllability — the three HARA input factors |
| ASIL | A (lowest) through D (highest); QM = no ISO 26262 rating required |
| FSR / TSR / SSR | Functional / Technical / Software Safety Requirement — the traceability chain |
| MC/DC | Modified Condition/Decision Coverage — required unit test rigor at higher ASIL |
| ASIL decomposition | Splitting one ASIL D requirement into independent lower-ASIL redundant paths |
| SPFM / LFM | Single-Point / Latent Fault Metric — ISO 26262-5 hardware coverage targets |
| TCL | Tool Confidence Level — ISO 26262-8 §11, qualification requirement for dev/verification tools |

## Exercise

Take the Level 3 body-controller project and run a lightweight HARA-style
exercise on it. (1) For the auto-lock-above-15km/h function, assign
plausible S/E/C ratings and justify each in one sentence, then state the
resulting ASIL — compare it against how the Level 3 project implicitly
treated `DoorControl` as "higher integrity." (2) Write one Functional
Safety Requirement, one Technical Safety Requirement, and one Software
Safety Requirement for that function, showing the traceability chain
explicitly narrowing from vehicle-level to code-level. (3) Take the
three-condition example decision above and write out MC/DC-satisfying
test vectors by hand — for each condition, identify the pair of test
cases that isolates its individual effect on the outcome, and confirm
you need fewer than the full 2^3 exhaustive set. (4) Write a short
independence plan: if you were the sole developer on this project, what
organizational step (external review, a second engineer, a formal
checklist) would you add to satisfy the independence requirement for the
ASIL D-rated portion.
