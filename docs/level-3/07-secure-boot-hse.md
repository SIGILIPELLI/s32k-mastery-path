---
description: "Secure Boot & the HSE Security Engine — Module 4 of Level 2 built a bootloader that flashes new firmware over UDS. That bootloader has an unstated…"
---

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

## How It Actually Works

Secure boot's root of trust has to originate somewhere the attacker cannot rewrite, and on S32K that's typically a small block of one-time-programmable (OTP) fuses or a masked boot ROM burned at fabrication — this is why it's called a *hardware* root of trust: unlike flash, OTP fuses are physically permanent (blown by an irreversible high-voltage process, not erasable/reprogrammable), so the public key hash or boot-configuration bits stored there cannot be altered by any software exploit after manufacturing, only trusted or not trusted from that point forward.

The HSE (Hardware Security Engine) present on more capable S32K variants is a genuinely separate processor core with its own private memory, running independently of the main application core specifically so that cryptographic key material and verification logic are never exposed to application-core memory space or debug access — signature verification (checking the application image's signature against the OTP-anchored public key using a hardware-accelerated ECC or RSA engine) happens entirely inside the HSE's isolated execution environment, and only a pass/fail result (not the key material) crosses back to the boot sequence deciding whether to jump to the application, the same `VTOR`-reprogram-and-jump mechanism covered in the bootloader module.

Anti-rollback protection (preventing a valid-but-old, vulnerable firmware image from being reinstalled) is enforced by a monotonic counter stored in a write-once or write-increment-only region — often battery-backed registers or dedicated OTP fuse rows that can only be incremented, never decremented, by hardware design — so the boot verification logic can reject an image whose embedded version is lower than the counter's current value, and no software path exists to decrement that counter back down.

*(Described from general automotive HSE/secure-boot architecture concepts and NXP S32K security documentation; not measured on physical silicon in this course.)*

## 🔀 Related lessons on other tracks

- [Embedded Python — Security Hardening — TLS & Secure Storage](https://sigilipelli.github.io/embedded-python-mastery-path/level-4/06-security-hardening/)
- [Embedded Linux — 02 · Secure Boot Chain (HAB/AHAB)](https://sigilipelli.github.io/embedded-linux-mastery-path/level-4/02-secure-boot-hab-ahab/)
- [Embedded — Secure Boot & Encrypted OTA at Scale](https://sigilipelli.github.io/embedded-mastery-path/level-4/02-secure-boot-ota-scale/)

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
