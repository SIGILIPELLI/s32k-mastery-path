# OTA & Reprogramming Stacks

Level 2's bootloader flashed firmware over a wired UDS session; Level 3
module 7 added a signature check before commit. A modern vehicle needs
one more layer: reprogramming an ECU **without a technician plugged into
the OBD port at all**, delivered over cellular/Wi-Fi to the vehicle and
distributed internally over the vehicle's CAN/Ethernet backbone. **OTA
(Over-The-Air)** update stacks are this delivery mechanism, and the parts
of it that land on the S32K side are the pieces that matter for firmware
engineering: dual-bank flash management, an atomic activation scheme, and
a rollback path that survives a failed update without bricking the
vehicle.

## Where OTA sits relative to what you've already built

```text
Cloud OTA campaign management     <- not this module's concern
        │  (cellular/Wi-Fi to a Telematics Control Unit)
        ▼
Vehicle-internal distribution     <- module 8 (Ethernet gateway) territory
        │  (gateway relays image chunks to target ECUs over CAN/Ethernet)
        ▼
Target ECU's UDS bootloader        <- Level 2 module 8, this is where OTA lands
        │  (0x34/0x36/0x37 download sequence, now HSE-signed per Level 3 module 7)
        ▼
Dual-bank flash activation          <- this module's core content
```

An OTA stack changes almost nothing about the download protocol itself —
it's still UDS over whatever transport reaches the ECU. What it adds is
everything around "what happens if this reprogram session gets
interrupted at 60% while the vehicle is in motion," which a wired,
technician-supervised reflash session could mostly ignore.

## Dual-bank flash: the mechanism that makes OTA safe

```text
Flash Bank A (currently running)     Flash Bank B (inactive)
  Active application v1.2      -->     New image v1.3 downloaded here
                                        while A keeps running normally

  Download interrupted? Bank A is untouched -- vehicle still fully
  functional on v1.2, no partial-flash corruption possible.

  Download + verify complete?  -->  Boot vector swaps to Bank B
                                     Bank A becomes the new inactive bank
```

```c
/* Dual-bank activation: the swap is a single, small, atomic write —
   never a large in-place overwrite of the running application */
typedef struct {
    uint32_t active_bank;      /* 0 = Bank A, 1 = Bank B */
    uint32_t bank_b_valid;     /* set only after full verify passes */
    uint32_t version_counter;  /* anti-rollback, per Level 3 module 7 */
} boot_info_t;

/* Stored in a small, separately-erasable flash sector — NOT inside
   either application bank, so a corrupted application image can
   never corrupt the record of which bank is active */
static boot_info_t boot_info __attribute__((section(".boot_info")));

Std_ReturnType Ota_ActivateNewImage(void)
{
    if (!Hse_VerifyImage(&pending_image_req) == HSE_STATUS_OK) {
        return E_NOT_OK;  /* module 7's check, mandatory before any activation */
    }
    if (pending_image_version <= boot_info.version_counter) {
        return E_NOT_OK;  /* anti-rollback: reject an old, even if validly signed, image */
    }

    boot_info_t new_info = boot_info;
    new_info.active_bank     = 1u - boot_info.active_bank;
    new_info.bank_b_valid    = 1u;
    new_info.version_counter = pending_image_version;

    /* The swap: one small structure write, not a reflash of megabytes.
       This single write is the entire "point of no return," and it is
       sized so a power loss during it corrupts at most this one sector. */
    Fee_Write(BOOT_INFO_BLOCK_ID, (const uint8_t *)&new_info, sizeof(new_info));

    return E_OK;
}
```

The core safety property: **the currently-running bank is never the
write target during a download.** Whatever goes wrong mid-transfer —
power loss, a dropped CAN connection, a crashed telematics unit — the
vehicle boots back into the bank it was already running, unaffected.

## Rollback on boot failure

```c
/* Boot-time health check: the new image gets a limited number of
   chances to prove itself before the bootloader reverts automatically */
void Bootloader_Main(void)
{
    if (boot_info.bank_b_valid && (boot_info.boot_attempt_count < MAX_BOOT_ATTEMPTS)) {
        boot_info.boot_attempt_count++;
        Fee_Write(BOOT_INFO_BLOCK_ID, (const uint8_t *)&boot_info, sizeof(boot_info));
        JumpToBank(boot_info.active_bank);
    } else {
        /* New image failed to confirm itself within the attempt budget —
           automatically fall back, without waiting for a service visit */
        boot_info.active_bank = 1u - boot_info.active_bank;
        boot_info.bank_b_valid = 0u;
        Fee_Write(BOOT_INFO_BLOCK_ID, (const uint8_t *)&boot_info, sizeof(boot_info));
        JumpToBank(boot_info.active_bank);
    }
}

/* The new application MUST call this once it has confirmed it is
   running correctly (e.g. passed its own self-test, established
   comms) — otherwise the attempt counter above will eventually revert it */
void App_ConfirmBootSuccess(void)
{
    boot_info.boot_attempt_count = 0u;
    Fee_Write(BOOT_INFO_BLOCK_ID, (const uint8_t *)&boot_info, sizeof(boot_info));
}
```

This "confirm or auto-revert" pattern is what prevents a vehicle from
being stuck boot-looping a broken update in a customer's driveway — the
system degrades back to the last-known-good image automatically, and the
failure surfaces as a failed OTA campaign report, not a stranded vehicle.

## Automotive-MCU concerns

- **Flash bank sizing directly constrains what you can OTA.** A dual-bank
  scheme needs enough flash for two full application images plus the
  small boot_info sector — on a flash-constrained S32K1 part this is a
  real budget decision, not a given, and is a major reason larger,
  OTA-capable ECUs often specify S32K3 parts with more flash headroom.
- **The download session itself must tolerate a vehicle power state
  change.** A customer turning the ignition off mid-download must not
  corrupt Bank B — either resume from a checkpoint on next power-up, or
  restart cleanly, but never assume the download completes in one
  uninterrupted session. This is a materially different assumption than
  Level 2's workshop-supervised reflash, where a technician keeps power
  stable for the whole session.
- **The boot_info sector is a single point of failure if not itself
  protected.** It must live in its own flash sector, written with the
  same wear-leveling discipline as any other flash-backed data (Level 1
  concerns about flash wear apply directly, now at higher write
  frequency if boot attempts increment it every cycle), and ideally
  mirrored or checksum-protected so a torn write to it doesn't strand
  the ECU unable to determine which bank is valid.
- **OTA doesn't remove the reprogramming preconditions from Level 2's UDS
  bootloader.** Vehicle-speed and ignition-state gating (never allow a
  safety-relevant ECU reflash while driving) still applies, and now needs
  to be enforced by both the vehicle-side OTA orchestration (a
  precondition check before initiating) and the ECU's own bootloader (a
  precondition check that doesn't trust the orchestrator alone).

## Cheat sheet

| Term | Meaning |
|------|---------|
| Dual-bank flash | Two application image slots; download targets the inactive one, active bank never touched mid-update |
| boot_info sector | Small, separately-erasable flash sector recording active bank + version + boot attempt count |
| Activation swap | Single small atomic write flipping the active bank pointer — the actual "point of no return" |
| Auto-rollback | Bootloader reverts to the previous bank if the new image fails to self-confirm within an attempt budget |
| App_ConfirmBootSuccess | New image's explicit signal that it booted and self-tested correctly |
| Anti-rollback (version counter) | Rejects reinstalling an old, validly-signed image (Level 3 module 7) |
| Session resumability | Download must tolerate an interrupted vehicle power cycle, not assume one continuous session |
| Preconditions | Vehicle speed / ignition state gating, enforced independently by orchestrator AND ECU bootloader |

## Exercise

Extend your Level 2 UDS bootloader with a dual-bank OTA-style activation
scheme. (1) Define a `boot_info_t` structure and its own dedicated flash
sector, separate from either application bank, and implement the
activation swap as a single small write. (2) Implement the auto-rollback
boot-attempt counter and `App_ConfirmBootSuccess`, then simulate a
"broken update" by deliberately never calling confirm from the new
image — verify the bootloader reverts to the previous bank within your
defined attempt budget. (3) Simulate a power loss mid-download (stop the
UDS transfer sequence partway through and reset the board) and confirm
the previously active bank is completely unaffected and still boots
correctly. (4) Write out, as a design note, exactly which preconditions
(vehicle speed, ignition state, battery voltage) your bootloader should
independently verify before allowing bank activation — even if your test
rig can't produce those signals, specify what the check would look like
and why the ECU cannot simply trust the orchestrator's claim that
conditions are safe.
