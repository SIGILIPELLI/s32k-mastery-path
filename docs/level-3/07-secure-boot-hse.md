# Secure Boot & the HSE Security Engine

Module 4 of Level 2 built a bootloader that flashes new firmware over
UDS. That bootloader has an unstated assumption: the image it's flashing
is legitimate. Once a vehicle is reachable over-the-air or through an
OBD dongle (Level 4 module 4 covers OTA properly), that assumption is a
liability — an attacker who can get arbitrary code flashed onto a body
controller can, at minimum, unlock every door on the vehicle remotely.
**Secure boot** closes this gap: before the application image runs, the
MCU cryptographically verifies it was signed by the OEM, and refuses to
boot anything else. On S32K3, this is implemented using the **HSE
(Hardware Security Engine)** — a separate, isolated core inside the same
package that owns the cryptographic keys and root-of-trust, deliberately
kept out of reach of the application core even if the application core
is fully compromised.

## Why a separate core, not a library

```text
Software-only signature check          HSE-based secure boot
──────────────────────────────         ──────────────────────
Verification code runs on the          Verification runs on an
same core as the app it verifies       isolated, separate core
Private key material must be           HSE holds keys in its own
reachable to derive a public key       protected key storage;
check, or hardcoded — extractable      app core never sees them
if the app core is compromised
A code-execution bug in the app        A code-execution bug in the
can potentially patch out the          app core cannot reach or
verification call itself               disable the HSE's own logic
```

The HSE is not "more secure crypto code" — it is a structurally different
trust boundary. Even a full exploit of the application core's firmware
cannot forge a signature, because the signing/verification logic and key
material never execute on that core at all.

## The boot chain

```text
1. ROM (immutable, silicon mask)
      verifies HSE firmware signature -> boots HSE
2. HSE core
      verifies application core's boot image signature
      using OEM public key stored in HSE-protected flash
3. If valid: releases application core from reset, boot proceeds
   If invalid: application core held in reset — no code executes
```

```c
/* Application-side request to HSE, over the S32K3's internal HSE
   host interface (a mailbox-style shared-memory + interrupt protocol) */
typedef struct {
    uint32_t image_addr;
    uint32_t image_len;
    uint8_t  signature[64];   /* e.g. ECDSA P-256 signature */
} hse_verify_req_t;

hse_status_t Hse_VerifyImage(const hse_verify_req_t *req)
{
    Hse_Host_SendRequest(HSE_SRV_ID_VERIFY, req, sizeof(*req));
    return Hse_Host_WaitResponse(HSE_VERIFY_TIMEOUT_MS);
    /* HSE_STATUS_OK only if the signature validates against the OEM
       public key already provisioned into HSE-protected NVM at
       manufacturing time (module 9 covers that provisioning step) */
}
```

Application code never sees the private key, never re-implements the
signature algorithm, and cannot bypass this by patching application
flash — the check that matters already ran, in the HSE, before the
application core's reset was released.

## Chain of trust extends past first boot

```c
/* A secure-boot-aware application still verifies anything it loads
   at runtime — an OTA update image, a calibration blob (module 9's
   XCP), a diagnostic-flashed module — using the same HSE service,
   not a re-implementation */
Std_ReturnType Bootloader_ValidateNewImage(uint32_t addr, uint32_t len,
                                             const uint8_t *sig)
{
    hse_verify_req_t req = { .image_addr = addr, .image_len = len };
    memcpy(req.signature, sig, sizeof(req.signature));

    if (Hse_VerifyImage(&req) != HSE_STATUS_OK) {
        Dem_ReportErrorStatus(DEM_EVENT_SECURE_BOOT_FAIL, DEM_EVENT_STATUS_FAILED);
        return E_NOT_OK; /* never jump to or flash-commit an unverified image */
    }
    return E_OK;
}
```

Secure boot only protects what runs at reset. Any mechanism that can
load and execute code afterward — the bootloader's own UDS flash service,
an OTA agent, a debug interface left enabled in production — is a second
attack surface that needs the same verification discipline, or the
secure boot chain is trust-anchoring a system with an unlocked side door.

## Automotive-MCU concerns

- **Debug ports are a secure boot bypass if left open.** JTAG/SWD access
  to a production part that hasn't had its debug interface locked down
  (S32K3 supports debugger authentication/disable via HSE-managed life
  cycle state) lets an attacker halt the core, dump memory, or single-
  step past a check — secure boot does nothing against a debugger with
  full core control. Production life-cycle transition (closing debug
  access) is part of the same security story, not a separate concern.
- **Key provisioning is a one-time, high-stakes manufacturing step.**
  The OEM public key (or a key hierarchy: a root key signs intermediate
  keys) gets burned into HSE-protected NVM during manufacturing (module
  9). A provisioning mistake — wrong key, or a key not properly locked
  against rewriting — either bricks legitimate updates or leaves the
  root of trust replaceable by an attacker.
- **HSE service calls are not free — budget the latency.** A full image
  signature verification over a multi-hundred-KB application image takes
  measurable time (tens to low-hundreds of milliseconds depending on
  image size and algorithm), which adds directly to boot time. A
  secure-boot design with an aggressive boot-time requirement needs this
  budgeted explicitly, not discovered during integration testing.
- **Rollback protection is a separate mechanism from signature
  verification.** A validly-signed *old* firmware image with a known
  vulnerability is still a valid signature — secure boot alone does not
  prevent downgrade attacks. Anti-rollback (a monotonic version counter
  the HSE checks alongside the signature) is required to close that gap,
  and is exactly the kind of control an ISO/SAE 21434 cybersecurity
  analysis (Level 4 module 5) would flag if missing.

## Cheat sheet

| Term | Meaning |
|------|---------|
| HSE | Hardware Security Engine — isolated core on S32K3 holding keys/crypto, separate trust domain |
| ROM boot | Immutable silicon logic that verifies and boots the HSE firmware first |
| Chain of trust | ROM → HSE → application image, each stage verifying the next before releasing it |
| Life cycle state | HSE-managed production state controlling debug access, key locking |
| Anti-rollback | Monotonic counter preventing reinstall of an old, validly-signed but vulnerable image |
| Signature algorithm | Commonly ECDSA (e.g. P-256) for automotive secure boot |
| Key provisioning | One-time manufacturing step writing OEM keys into HSE-protected NVM |
| Relevant standards | ISO/SAE 21434 (cybersecurity process), NIST SP 800-57 (key management, referenced generically) |

## Exercise

Design (and where an S32K3 board with HSE is available, implement) a
secure boot chain for a body controller image. (1) Define the trust
chain explicitly: what signs what, where each key lives, and what the
failure action is at each stage if verification fails (should always be
"hold in reset" or "reject," never "boot anyway with a warning"). (2)
Write the `Hse_VerifyImage` call pattern for your bootloader's UDS
`0x34`/`0x36`/`0x37` download sequence from Level 2, adding a signature
check before the final flash-commit step — reject and log via DEM on
failure rather than silently discarding the image. (3) Design an anti-
rollback scheme: a version counter stored in a location the application
cannot rewrite on its own, checked alongside the signature. (4) Write out
your production life-cycle plan: at what manufacturing step do you
provision keys, and at what step do you close debug access — and what
happens if a board needs field debugging after that step (a real
supply-chain and field-support trade-off every OEM secure boot
deployment has to resolve).
