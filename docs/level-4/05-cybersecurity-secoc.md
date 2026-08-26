# Cybersecurity — ISO/SAE 21434 & SecOC

Level 3 module 7 secured the boot chain; module 4 of this level secured
reprogramming. Neither protects the thing both ultimately run on top of:
**ordinary CAN traffic while the vehicle is driving.** A classic CAN
frame has no authentication field — any node physically on the bus (or
reaching it through a compromised gateway, Level 3 module 8's concern)
can send a frame with any ID, and every receiver trusts it. **SecOC**
(Secure Onboard Communication, AUTOSAR's specification) adds a
cryptographic authenticator to safety- or security-relevant CAN/CAN FD
frames, and **ISO/SAE 21434** is the process standard — cybersecurity's
equivalent of ISO 26262 — that determines which frames need it.

## Where this fits: TARA, not just SecOC

Before choosing which frames to protect, ISO/SAE 21434 requires a
**TARA (Threat Analysis and Risk Assessment)**, structurally similar to
ISO 26262's HARA but for security threats instead of random hardware
faults:

```text
Asset (e.g. brake command CAN message)
  -> Threat scenario (e.g. spoofed brake-release frame injected via
     compromised infotainment gateway)
  -> Impact rating (safety, financial, operational, privacy)
  -> Attack feasibility (elapsed time, expertise, equipment needed)
  -> Risk value -> determines required security controls (e.g. SecOC)
```

A TARA on the Level 3 body controller would very likely flag
`BCM_DoorStatus`'s lock-state signal as requiring authentication (an
attacker unlocking doors remotely is a plausible, high-impact threat) and
might rate `BCM_LightingStatus` lower priority — the same "not everything
gets the same treatment" principle as ASIL assignment in module 2, now
applied to security instead of safety.

## SecOC frame structure

```text
Standard CAN FD payload (up to 64 bytes):
┌────────────────────────────┬─────────────┬───────────────┐
│  Original signal data       │  Freshness   │  Truncated MAC │
│  (application payload)      │  Value (FV)  │  (authenticator)│
└────────────────────────────┴─────────────┴───────────────┘
```

```c
/* SecOC authentication on transmit — computed once per PDU, using a
   shared symmetric key provisioned at manufacturing (module 9) */
typedef struct {
    uint32_t freshness_value;   /* monotonic counter, prevents replay */
    uint8_t  mac[4];            /* truncated MAC — full MAC computed,
                                    only the low bytes transmitted to
                                    save payload space */
} secoc_auth_t;

Std_ReturnType SecOC_AuthenticateTx(const uint8_t *payload, uint16_t len,
                                      secoc_auth_t *auth_out)
{
    auth_out->freshness_value = g_secoc_tx_counter++;  /* per-sender-ID counter */

    uint8_t mac_full[16];  /* e.g. CMAC-AES128 or similar, per AUTOSAR SecOC spec */
    Csm_MacGenerate(SECOC_KEY_ID, payload, len,
                     (const uint8_t *)&auth_out->freshness_value,
                     sizeof(auth_out->freshness_value), mac_full);

    memcpy(auth_out->mac, mac_full, sizeof(auth_out->mac)); /* truncate */
    return E_OK;
}

/* SecOC verification on receive */
Std_ReturnType SecOC_VerifyRx(const uint8_t *payload, uint16_t len,
                                 const secoc_auth_t *auth_in)
{
    if (auth_in->freshness_value <= g_secoc_rx_last_counter) {
        return E_NOT_OK;  /* replay: freshness value must strictly increase */
    }

    uint8_t mac_full[16];
    Csm_MacGenerate(SECOC_KEY_ID, payload, len,
                     (const uint8_t *)&auth_in->freshness_value,
                     sizeof(auth_in->freshness_value), mac_full);

    if (memcmp(mac_full, auth_in->mac, sizeof(auth_in->mac)) != 0) {
        return E_NOT_OK;  /* MAC mismatch: reject, do not act on this frame */
    }

    g_secoc_rx_last_counter = auth_in->freshness_value;
    return E_OK;
}
```

The **freshness value** is doing a specific job the MAC alone cannot: a
MAC alone lets an attacker **replay** a previously-valid, correctly-
authenticated frame (e.g. capture a legitimate "unlock" frame and resend
it later) — the MAC would still verify, because the payload is unchanged
and genuinely signed. A strictly-increasing freshness value included in
the MAC computation makes a replayed old frame fail verification, because
its freshness value is no longer greater than the last accepted one.

## Automotive-MCU concerns

- **Truncated MACs trade cryptographic strength for CAN payload budget —
  document that trade explicitly.** A full 128-bit CMAC truncated to 4
  bytes (32 bits) has real, calculable collision odds via brute force;
  AUTOSAR SecOC's specification provides guidance on acceptable
  truncation lengths per use case, and a TARA-driven decision to use a
  shorter truncation for payload reasons needs that risk documented, not
  assumed acceptable by default.
- **Freshness value synchronization after an ECU reset is a genuine
  design problem, not an edge case.** If the freshness counter resets to
  zero on every power cycle, an attacker can replay any previously-
  captured frame with a freshness value higher than zero immediately
  after a target ECU reboots. Real designs persist the freshness value
  (or a coarser synchronized time base) across resets specifically to
  close this window.
- **Key provisioning and rotation is a hard, whole-vehicle-lifecycle
  problem.** SecOC symmetric keys shared between a sender and every
  legitimate receiver ECU must be provisioned securely (via the same
  manufacturing process module 9 covers) and have a rotation/revocation
  story if a key is ever suspected compromised — "we'll never need to
  rotate it" is not a defensible position in a 15-year vehicle service
  life.
- **SecOC protects message authenticity and freshness, not
  confidentiality.** A SecOC-authenticated frame is still readable in
  plaintext by anyone on the bus — this is a deliberate, correct design
  choice for most vehicle signals (confidentiality is rarely the threat
  that matters for a door-lock state), but a TARA that identifies a
  genuine confidentiality requirement (e.g. VIN or key material in
  transit) needs an additional encryption control, not just SecOC.

## Cheat sheet

| Term | Meaning |
|------|---------|
| TARA | Threat Analysis and Risk Assessment — ISO/SAE 21434's HARA equivalent for security |
| SecOC | AUTOSAR Secure Onboard Communication — adds freshness + MAC to CAN/CAN FD PDUs |
| Freshness value (FV) | Strictly increasing counter included in the MAC; defeats replay attacks |
| MAC | Message Authentication Code (e.g. CMAC-AES128); often truncated to fit CAN payload budget |
| Csm | AUTOSAR Crypto Service Manager — the BSW module SecOC calls for MAC generation/verification |
| Replay attack | Resending a captured, validly-signed old frame; defeated by freshness, not by the MAC alone |
| Confidentiality | NOT provided by SecOC — payload remains plaintext on the bus |
| Relevant standard | ISO/SAE 21434 (process), AUTOSAR SecOC specification (mechanism) |

## Exercise

Add SecOC-style authentication to one signal from your Level 3 body
controller. (1) Run a lightweight TARA on your `BCM_DoorStatus` frame:
identify at least one plausible threat scenario, rate its impact and
feasibility, and justify whether it warrants SecOC protection. (2)
Implement `SecOC_AuthenticateTx`/`SecOC_VerifyRx` using any available
MAC primitive (a simple HMAC-SHA256 truncated to 4 bytes is a reasonable
stand-in if a CMAC-AES library isn't available) and demonstrate a
correctly authenticated frame passing verification. (3) Demonstrate a
replay attack against a MAC-only version (no freshness check) — capture
a valid authenticated frame, resend it later, and show it passes
verification; then add the freshness check and show the same replay now
fails. (4) Write a short note on freshness-value persistence: what
happens in your design if the ECU resets, and what would need to change
(e.g. persisting the counter to NVM, module 4's Fee driver) to close the
post-reset replay window.
