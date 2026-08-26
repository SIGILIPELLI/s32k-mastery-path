# Manufacturing, EOL Test & Traceability

Every "provisioned at manufacturing" reference across Level 3 and 4 —
HSE keys (module 7), SecOC keys (module 5), the boot_info version
counter (module 4) — has pointed at this module. A blank S32K3 arriving
from the fab has no application firmware, no keys, and no way to prove
it works correctly, and a vehicle assembly line cannot accept an ECU
without both. **End-of-Line (EOL) test** and **manufacturing
provisioning** are the process that turns a bare board into a verified,
secured, traceable unit — and it happens once, at scale, under time
pressure (a modern assembly line tests an ECU in seconds, not minutes),
which shapes every design decision in this module.

## The manufacturing sequence

```text
1. Bare PCB assembly (SMT line) -- no firmware, no keys
2. In-Circuit Test (ICT)         -- verifies solder joints, shorts,
                                     opens; no MCU code runs yet
3. Initial flash programming      -- via SWD/JTAG (production debug
                                     access, before it gets locked down
                                     per Level 3 module 7)
4. Key/certificate provisioning   -- HSE root keys, SecOC symmetric
                                     keys, written into protected NVM
5. Functional EOL test             -- the board runs real firmware and
                                     is verified against a test spec
6. Calibration                     -- XCP-based (Level 3 module 9)
                                     writing final calibration values
7. Life-cycle transition + debug lockdown  -- production life-cycle
                                     state, closes JTAG/SWD access
8. Serialization + traceability record  -- VIN-adjacent unique ID,
                                     stored and linked to test results
```

The ordering matters in ways that aren't obvious: **debug lockdown must
happen after** flash programming and key provisioning (both need debug
access) but **before** the unit ships (an unlocked debug port on a
shipped ECU is exactly the secure-boot bypass Level 3 module 7 warned
about). A manufacturing line that gets this ordering wrong either can't
program units at all, or ships units with an open attack surface.

## EOL functional test

```c
/* EOL test firmware mode — a special build or a special boot mode
   that runs manufacturing self-tests, NOT the production application */
typedef struct {
    bool can_loopback_pass;
    bool lin_loopback_pass;
    bool adc_reference_pass;    /* known reference voltage measured within tolerance */
    bool flash_crc_pass;
    bool watchdog_service_pass; /* SBC watchdog actually resets when unfed */
    uint16_t vbat_reading_mv;
} eol_test_result_t;

eol_test_result_t Eol_RunSelfTest(void)
{
    eol_test_result_t result = {0};

    result.can_loopback_pass = Can_SelfTest_Loopback();  /* TX->RX on the
                                    same board, verifies FlexCAN + transceiver */
    result.adc_reference_pass = Adc_VerifyReference(EXPECTED_REF_MV, TOLERANCE_MV);
    result.flash_crc_pass = (Fls_ComputeCrc(APP_FLASH_START, APP_FLASH_LEN)
                              == expected_crc);
    result.vbat_reading_mv = Adc_ReadVbat_mV();

    /* Watchdog test deliberately stops servicing and confirms the SBC
       actually resets the board within its expected window — verifying
       a Level 3 module 5 safety mechanism actually works on THIS
       physical unit, not just in the design */
    result.watchdog_service_pass = Eol_VerifyWatchdogResets();

    return result;
}
```

The watchdog self-test line matters more than it looks: a safety
mechanism validated only in design review and never verified on the
actual manufactured unit is an unverified assumption running in every
vehicle that ships. EOL test is the one point in a unit's life where
deliberately triggering a fault condition (starving the watchdog) is
both safe and necessary — no vehicle test track will ever safely
reproduce this specific fault condition to confirm the mechanism works.

## Traceability

```c
/* Every manufactured unit gets a unique identity linked to its test
   record — this is what lets a field failure be traced back to a
   specific manufacturing batch, test result, and even which component
   reels/lots were in use that shift */
typedef struct {
    uint8_t  serial_number[12];
    uint32_t manufacturing_date;
    uint16_t test_station_id;
    uint8_t  firmware_version[16];
    eol_test_result_t eol_result;
} unit_traceability_record_t;

Std_ReturnType Eol_CommitTraceabilityRecord(const unit_traceability_record_t *rec)
{
    /* Written to HSE-protected NVM (module 7) alongside the keys —
       tamper-evident, so a traceability record can't be silently
       altered after the fact */
    return Fee_Write(TRACEABILITY_BLOCK_ID, (const uint8_t *)rec, sizeof(*rec));
}
```

Traceability is what makes a field recall tractable: if a specific
component lot is later found defective, the traceability record lets an
OEM identify exactly which manufactured units used that lot, rather than
recalling every unit ever produced. ISO 26262's process requirements
extend into manufacturing precisely because a safety case built on
"the design is correct" is incomplete without "and every manufactured
unit was verified to match that design."

## Automotive-MCU concerns

- **EOL test time directly affects manufacturing cost at automotive
  volumes.** A test sequence that takes 30 seconds per unit, multiplied
  across hundreds of thousands of units per year, is a real cost line —
  EOL test design constantly trades thoroughness against cycle time, and
  understanding which tests are non-negotiable (safety mechanism
  verification) versus which can be sampled or shortened is a real
  engineering judgment call, not purely a test engineer's decision.
- **Key provisioning at scale needs a secure, auditable process, not
  just a script.** A provisioning station with access to root signing
  keys is a high-value attack target — physical security, key rotation
  procedures, and per-station audit logs are part of the manufacturing
  security architecture, and a TARA (module 5) covering the vehicle
  should extend to the manufacturing supply chain, not stop at the
  shipped product.
- **A unit that fails EOL test needs a defined disposition path, not
  ad hoc handling.** Rework (reflash and retest), scrap, or root-cause
  quarantine each have different cost and traceability implications — a
  manufacturing line without a clear failure-disposition process risks
  either scrapping recoverable units unnecessarily or reworking units
  that should have been quarantined for root-cause analysis.
- **Debug lockdown verification must itself be tested, not assumed.** A
  life-cycle transition command that silently fails to actually close
  JTAG/SWD access (a firmware bug, or an incomplete OTP fuse burn) ships
  a unit with the exact secure-boot bypass the whole mechanism exists to
  prevent — EOL test should include a step that attempts debug access
  post-lockdown and confirms it is actually refused, not just that the
  lockdown command returned success.

## Cheat sheet

| Stage | Purpose |
|-------|---------|
| ICT | Verifies solder/assembly correctness before any firmware runs |
| Initial flash programming | Via SWD/JTAG, before debug lockdown |
| Key/certificate provisioning | HSE root keys, SecOC keys, written to protected NVM |
| EOL functional test | Runs real self-test firmware; verifies CAN/LIN/ADC/flash/watchdog |
| Calibration | XCP-based (Level 3 module 9) final tuning values |
| Life-cycle transition | Closes debug access — must happen AFTER programming, BEFORE shipping |
| Traceability record | Serial number + test results + firmware version, tamper-evident storage |
| Disposition path | Rework / scrap / quarantine for units failing EOL test |

## Exercise

Design an EOL test sequence for the Level 3 body controller. (1)
Implement `Eol_RunSelfTest` covering at minimum CAN loopback, an ADC
reference check, and a flash CRC check, and specify pass/fail tolerances
for each. (2) Design the watchdog self-test: specify exactly how you'd
deliberately stop servicing the SBC watchdog during EOL test and confirm
a reset occurs within the expected window, without damaging the unit or
leaving it in a bad state if the test fails. (3) Define a
`unit_traceability_record_t` for your design and specify where and how
it's stored so it survives a later field investigation. (4) Write your
manufacturing sequence in order, explicitly marking where debug
lockdown occurs and justifying why it cannot occur earlier (breaks
programming) or be skipped (leaves a secure-boot bypass) — then design
one verification step that confirms lockdown actually took effect rather
than trusting the lockdown command's return value alone.
