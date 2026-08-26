# HIL Testing & CI for ECU Firmware

Every exercise so far has ended with "verify on real hardware." A
production automotive project cannot run its entire regression suite by
hand on a bench every time a line of code changes — a body controller
codebase touched by a dozen engineers needs its safety-relevant behavior
re-verified automatically, on every commit, against real timing and real
electrical signals, not just a desktop simulation. **Hardware-in-the-Loop
(HIL)** testing is how this is done: the real ECU (or real S32K target),
its real I/O pins wired to a test system that simulates the rest of the
vehicle — sensors, actuators, other ECUs' CAN traffic — driven by an
automated test sequence, integrated into **CI (Continuous Integration)**
so it runs on every relevant code change without a human initiating it.

## Where HIL sits in the test pyramid

```text
Unit tests (module 3's MC/DC-covered tests)   -- fastest, runs on host PC,
                                                   no hardware, mocks every
                                                   peripheral
Software-in-the-Loop (SIL)                     -- same test cases, cross-
                                                   compiled for target,
                                                   run in an instruction
                                                   set simulator
Hardware-in-the-Loop (HIL)                     -- real S32K silicon, real
                                                   timing, simulated
                                                   electrical environment
Vehicle-level test                              -- slowest, most expensive,
                                                   real vehicle, real roads
                                                   or a test track
```

Each layer catches a different class of bug: unit tests catch logic
errors cheaply; HIL catches the class this whole course has emphasized —
timing violations, register-level misconfiguration, real ADC noise
behavior, actual CAN bus electrical faults — that a host-PC simulation
cannot reproduce because it doesn't touch real silicon.

## A HIL rig for the body controller

```text
┌──────────────────────────────────────────────────────────┐
│  HIL Test System (e.g. dSPACE, NI VeriStand, or a custom   │
│  rig built from a second S32K board + relays + a CAN         │
│  interface)                                                 │
│                                                              │
│  Simulates: vehicle speed signal, door switch inputs,         │
│  battery voltage, CAN traffic from other ECUs                 │
└───────────────┬──────────────────────────────────────────┘
                │  Real wiring: analog signals, digital I/O, CAN bus
                ▼
┌──────────────────────────────────────────────────────────┐
│  Device Under Test: real S32K3 body controller, real         │
│  firmware image, running unmodified production code           │
└──────────────────────────────────────────────────────────┘
```

```python
# Example HIL test script (illustrative — real rigs use vendor-specific
# APIs, but the pattern generalizes across dSPACE ControlDesk, NI
# VeriStand, or a Python-based custom rig using python-can + a DAQ)

def test_auto_lock_at_speed_threshold(hil):
    hil.set_analog_input("vehicle_speed_sim", 10.0)   # km/h, below threshold
    hil.wait_ms(100)
    assert hil.read_can_signal("BCM_DoorStatus", "FL_Door_Locked") == 0

    hil.set_analog_input("vehicle_speed_sim", 20.0)   # above 15 km/h threshold
    hil.wait_ms(100)
    assert hil.read_can_signal("BCM_DoorStatus", "FL_Door_Locked") == 1

def test_secoc_replay_rejected(hil):
    valid_frame = hil.capture_can_frame("BCM_DoorStatus")
    hil.wait_ms(500)
    hil.inject_can_frame(valid_frame)          # replay a captured frame
    assert hil.read_dem_event("SECOC_AUTH_FAIL") == True   # module 5's freshness check catches it
```

Note the second test: it directly exercises module 5's SecOC freshness
mechanism against a *real* replayed CAN frame on a *real* bus, which is
exactly the class of test a unit test mocking the CAN peripheral cannot
meaningfully perform — the timing and bus-level behavior are the point.

## CI pipeline structure

```yaml
# Illustrative CI pipeline (structure generalizes across Jenkins,
# GitLab CI, or a vendor ALM tool's pipeline definition)
stages:
  - build:            # compile with the qualified toolchain (module 2's TCL)
      script: build_s32k_image.sh --config production
  - static_analysis:   # module 3's MISRA gate
      script: run_misra_analysis.sh --fail-on-required
  - unit_test:         # host-based, MC/DC coverage checked
      script: run_unit_tests.sh --coverage mcdc --min 100
  - sil_test:          # instruction-set simulator regression
      script: run_sil_suite.sh
  - hil_test:          # runs only on a schedule / merge to main —
      script: run_hil_suite.sh   # HIL rigs are a shared, limited resource
      when: merge_to_main
```

The staged structure matters for a practical reason: HIL rigs are
physical, limited-capacity resources — a project cannot run full HIL
regression on every developer's every commit without a rig farm sized
for that load, so most real pipelines run cheap gates (build, static
analysis, unit test) on every commit and reserve HIL for merges or a
nightly schedule, escalating cost only as confidence in a change grows.

## Automotive-MCU concerns

- **A HIL rig's simulated signals must match real sensor characteristics,
  not idealized ones.** A wheel-speed sensor simulator that produces a
  perfectly clean square wave will not catch a firmware bug in your
  Level 1/2 debouncing or noise-filtering logic — realistic HIL rigs
  inject representative noise and edge-case timing, because a bug that
  only manifests against real sensor imperfection is exactly the bug
  class HIL exists to catch.
- **Flaky HIL tests erode trust in the whole gate faster than almost any
  other CI failure mode.** A test that intermittently fails due to rig
  timing jitter (not a real firmware defect) trains engineers to re-run
  and ignore failures — the fix is diagnosing and eliminating the rig-
  level flakiness, never adding a retry loop that papers over a genuine
  intermittent defect indistinguishable from rig noise.
- **HIL test coverage of safety mechanisms needs deliberate fault
  injection, which most rigs require dedicated engineering to build.**
  Testing that the SBC watchdog (Level 3 module 5) actually forces a
  reset requires the rig to be able to simulate a hung MCU or a missed
  service window — this is meaningfully harder to build than a simple
  signal-in/signal-out test, and is often the first fidelity a resource-
  constrained project cuts, which is exactly the coverage an ASIL D
  safety case most needs.
- **CI pipeline security matters for the same reason secure boot
  matters.** A CI system with write access to a signing key (to produce
  HSE-signed test images, Level 3 module 7) is itself part of the attack
  surface a TARA (module 5) should consider — a compromised build server
  can produce a validly-signed malicious image just as effectively as a
  compromised HSE key.

## Cheat sheet

| Test layer | Runs on | Catches |
|------------|---------|----------|
| Unit test | Host PC, mocked peripherals | Logic errors, cheap and fast |
| SIL | Instruction-set simulator | Cross-compilation issues, some timing |
| HIL | Real silicon, simulated I/O | Register misconfiguration, real timing, electrical faults |
| Vehicle test | Real vehicle | Integration issues no rig models |
| CI gate | Typically runs on | Purpose |
| Build + static analysis | Every commit | Fast, catches MISRA/compile issues early |
| Unit test (MC/DC) | Every commit | Logic coverage per module 2's requirement |
| HIL suite | Merge to main / nightly | Expensive, shared-resource, deepest confidence |
| Fault injection HIL | Scheduled, dedicated rig time | Verifies safety mechanisms actually trigger under real fault conditions |

## Exercise

Design a HIL test plan for the Level 3 body controller (implement what
your bench allows; specify the rest). (1) Write out, in the Python-style
pseudocode above, at least 3 HIL test cases covering: the vehicle-speed
auto-lock interlock, a SecOC replay rejection, and an MPU partition
violation response — for each, specify what real signal or CAN traffic
the rig must inject. (2) If you have two S32K boards, build a minimal
rig where one board simulates vehicle-speed and door-switch inputs to
the other (the device under test) and confirm the auto-lock test passes
against real hardware. (3) Design a fault-injection test for the SBC
external watchdog (Level 3 module 5): specify exactly how a rig would
need to interrupt the watchdog service (e.g. halting the DUT's core via
debug interface at a controlled moment) to verify the SBC actually
forces a reset — note what rig capability this requires beyond simple
signal injection. (4) Write a staged CI pipeline definition (YAML or
plain text) for this project matching the structure above, and justify
which stage each of your Level 3/4 verification steps (MISRA, unit test,
HIL) belongs in and why.
