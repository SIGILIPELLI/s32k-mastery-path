# UDS Diagnostics Intro (ISO 14229)

When a workshop plugs a tester into the OBD connector and reads a fault
code, when an end-of-line station writes a VIN into a fresh ECU, and when
the bootloader from module 8 is asked to accept a new image — all three
are the same protocol. **UDS**, Unified Diagnostic Services, is the
request/response language every automotive ECU speaks, standardised as
ISO 14229 and carried over CAN by ISO 15765-2 (ISO-TP). An ECU without a
UDS server cannot be diagnosed, configured or reflashed, and no OEM will
accept it.

## The request/response shape

```text
Request        SID  [sub-function]  [data ...]
Positive resp  SID + 0x40           [data ...]
Negative resp  0x7F  SID  NRC
```

A request for the active session (`22 F1 86`) is answered `62 F1 86 03`; a
rejected one is answered `7F 22 31` — service `0x22`, NRC `0x31`
requestOutOfRange. Three rules trip up first implementations. **The
response SID is the request SID plus `0x40`** — not a separate identifier.
**A sub-function's bit 7 is `suppressPosRspMsgIndicationBit`**: if the
tester sets it, send no positive response, but still send a negative one
if the request fails. And **silence is never a valid answer** — any
request you understand enough to address must produce a response or a
documented NRC, because a tester timeout is reported as "ECU not
responding", which reads like a hardware fault.

| NRC | Name | When |
|-----|------|------|
| `0x11` | serviceNotSupported | SID not implemented |
| `0x12` | subFunctionNotSupported | SID exists, sub-function does not |
| `0x13` | incorrectMessageLengthOrInvalidFormat | Length check failed |
| `0x22` | conditionsNotCorrect | Vehicle moving, voltage too low |
| `0x24` | requestSequenceError | e.g. key before seed |
| `0x31` | requestOutOfRange | Unknown DID, address outside range |
| `0x33` | securityAccessDenied | Service needs unlocking first |
| `0x35` / `0x36` / `0x37` | invalidKey / exceedNumberOfAttempts / requiredTimeDelayNotExpired | SecurityAccess failures |
| `0x78` | requestCorrectlyReceivedResponsePending | Working on it; restarts P2 |
| `0x7E` / `0x7F` | subFunction- / serviceNotSupportedInActiveSession | Wrong session |

## Sessions and the S3 timer

The ECU is always in exactly one **diagnostic session**, and the session
decides which services are legal.

| ID | Session | Typical rights |
|----|---------|----------------|
| `0x01` | default | Read-only: `0x22` DIDs, `0x19` read DTCs, `0x3E` |
| `0x02` | programming | Download services `0x34`/`0x36`/`0x37` — usually the bootloader |
| `0x03` | extendedDiagnostic | Writes (`0x2E`), routines (`0x31`), actuator tests (`0x2F`) |
| `0x04` | safetySystemDiagnostic | Safety routines behind security access |

Three timers govern the conversation. **P2** — the ECU answers within
typically **50 ms**. **P2\*** — the extended budget, typically **5000 ms**,
granted only after replying `0x78` responsePending. **S3** — after
**5000 ms** with no diagnostic request the ECU falls back to the default
session by itself; `0x3E` TesterPresent exists purely to reset it. That
fallback is a safety feature: if a technician unplugs the tester
mid-actuator-test, the ECU must return to default, which means returning
actuators to normal control.

```c
#define UDS_S3_MS   5000u
#define UDS_P2_MS     50u

typedef enum {
    UDS_SESSION_DEFAULT  = 0x01u,
    UDS_SESSION_PROGRAM  = 0x02u,
    UDS_SESSION_EXTENDED = 0x03u,
} uds_session_t;

typedef struct {
    uds_session_t session;
    uint32_t      s3_ms;
    bool          unlocked;
    uint8_t       key_attempts;
    uint32_t      seed;
} uds_ctx_t;

static uds_ctx_t g_uds = { .session = UDS_SESSION_DEFAULT };

void uds_task_10ms(void)
{
    if (g_uds.session == UDS_SESSION_DEFAULT) { return; }

    if (g_uds.s3_ms > 10u) {
        g_uds.s3_ms -= 10u;
    } else {
        g_uds.session  = UDS_SESSION_DEFAULT;
        g_uds.unlocked = false;          /* security drops with session */
        app_stop_all_actuator_tests();   /* back to normal control      */
    }
}
```

## ISO-TP: fitting 4 KB into 8-byte frames

A CAN frame carries 8 bytes; a `0x22` response with a VIN needs 20.
ISO 15765-2 segments the message using the first byte or two of each
frame:

| PCI | Name | Layout |
|-----|------|--------|
| `0x0L` | Single Frame | `0L` plus up to 7 data bytes (`L` = length) |
| `0x1LLL` | First Frame | `1` plus 12-bit length, then 6 data bytes |
| `0x2S` | Consecutive Frame | `2` plus 4-bit sequence number, wrapping 0–15 |
| `0x3F BS ST` | Flow Control | FlowStatus, BlockSize, SeparationTime |

The flow is fixed: First Frame, then one Flow Control from the receiver,
then Consecutive Frames numbered `1, 2, … F, 0, 1, …`. In your Flow
Control, **FlowStatus** is `0` ContinueToSend, `1` Wait, `2` Overflow —
use `2` when the declared length exceeds your buffer rather than
truncating silently. **BlockSize** `0` means send everything without
pausing. **STmin** is `0x00–0x7F` milliseconds or `0xF1–0xF9` for
100–900 µs; an ECU with a slow receive path asks for a larger STmin
instead of dropping frames. That is your back-pressure control.

Addresses on a typical 11-bit bus are `0x7DF` functional (tester → every
ECU), `0x7E0` physical request and `0x7E8` physical response for ECU #1.
Functional addressing is why length checks matter: every ECU on the bus
sees `0x7DF`, so a malformed frame must be rejected by all of them without
side effects.

## A minimal server on S32K

The dispatcher sits on the FlexCAN plumbing from module 1 — `0x7E0` and
`0x7DF` are two more filter entries, and responses go out through a
dedicated TX mailbox so they never contend with periodic application
frames.

```c
#define SID_SESSION_CONTROL   0x10u
#define SID_READ_DID          0x22u
#define SID_SECURITY_ACCESS   0x27u
#define SID_WRITE_DID         0x2Eu
#define SID_ROUTINE_CONTROL   0x31u
#define SID_TESTER_PRESENT    0x3Eu

static uint8_t  g_resp[UDS_BUF_SIZE];
static uint16_t g_respLen;

static void uds_negative(uint8_t sid, uint8_t nrc)
{
    g_resp[0] = 0x7Fu;  g_resp[1] = sid;  g_resp[2] = nrc;
    g_respLen = 3u;
}

void uds_handle_request(const uint8_t *req, uint16_t len, bool functional)
{
    g_respLen = 0u;
    if (len == 0u) { return; }

    g_uds.s3_ms = UDS_S3_MS;              /* any request resets S3     */

    switch (req[0]) {
    case SID_SESSION_CONTROL: uds_session_control(req, len); break;
    case SID_READ_DID:        uds_read_did(req, len);        break;
    case SID_WRITE_DID:       uds_write_did(req, len);       break;
    case SID_SECURITY_ACCESS: uds_security_access(req, len); break;
    case SID_ROUTINE_CONTROL: uds_routine_control(req, len); break;
    case SID_TESTER_PRESENT:
        if (len != 2u) { uds_negative(req[0], 0x13u); break; }
        g_resp[0] = 0x7Eu; g_resp[1] = req[1] & 0x7Fu; g_respLen = 2u;
        break;
    default:
        uds_negative(req[0], 0x11u);
        break;
    }

    /* suppressPosRspMsgIndicationBit */
    if ((len >= 2u) && ((req[1] & 0x80u) != 0u) && (g_resp[0] != 0x7Fu)) {
        g_respLen = 0u;
    }
    /* Never answer a functional request with serviceNotSupported — the
       bus would fill with identical negatives from every ECU.        */
    if (functional && (g_resp[0] == 0x7Fu) &&
        ((g_resp[2] == 0x11u) || (g_resp[2] == 0x7Fu))) {
        g_respLen = 0u;
    }

    if (g_respLen != 0u) { isotp_send(0x7E8u, g_resp, g_respLen); }
}
```

`0x22` ReadDataByIdentifier is the service you implement first and use
forever. DIDs are 16-bit, and the standard reserves a block of them:

```c
#define DID_ACTIVE_SESSION   0xF186u
#define DID_SW_VERSION       0xF189u
#define DID_VIN              0xF190u
#define DID_COOLANT_TEMP     0x0100u   /* our own, from the capstone  */

static void uds_read_did(const uint8_t *req, uint16_t len)
{
    if (len != 3u) { uds_negative(SID_READ_DID, 0x13u); return; }

    uint16_t did = ((uint16_t)req[1] << 8) | req[2];

    g_resp[0] = SID_READ_DID + 0x40u;
    g_resp[1] = req[1];
    g_resp[2] = req[2];

    switch (did) {
    case DID_ACTIVE_SESSION:
        g_resp[3] = (uint8_t)g_uds.session;
        g_respLen = 4u;
        break;
    case DID_VIN:
        (void)nvm_read(NVM_ID_VIN, &g_resp[3], 17u);  /* module 6      */
        g_respLen = 20u;                              /* multi-frame   */
        break;
    case DID_COOLANT_TEMP: {
        int16_t t = app_get_temp_dC();                /* Level 1 node  */
        g_resp[3] = (uint8_t)((uint16_t)t >> 8);
        g_resp[4] = (uint8_t)((uint16_t)t & 0xFFu);
        g_respLen = 5u;
        break;
    }
    default:
        uds_negative(SID_READ_DID, 0x31u);
        break;
    }
}
```

## SecurityAccess and RoutineControl

`0x27` is a two-step handshake: an odd sub-function requests a seed, the
next even sub-function delivers the key. The algorithm is OEM-specific and
confidential; what is standardised is the shape and the lockout.

```c
static void uds_security_access(const uint8_t *req, uint16_t len)
{
    if (len < 2u) { uds_negative(SID_SECURITY_ACCESS, 0x13u); return; }
    if (g_uds.session == UDS_SESSION_DEFAULT) {
        uds_negative(SID_SECURITY_ACCESS, 0x7Fu);      /* wrong session */
        return;
    }

    if ((req[1] & 1u) != 0u) {                         /* requestSeed   */
        g_uds.seed = g_uds.unlocked ? 0u : sec_generate_seed();
        g_resp[0] = SID_SECURITY_ACCESS + 0x40u;
        g_resp[1] = req[1];
        g_resp[2] = (uint8_t)(g_uds.seed >> 24);
        g_resp[3] = (uint8_t)(g_uds.seed >> 16);
        g_resp[4] = (uint8_t)(g_uds.seed >> 8);
        g_resp[5] = (uint8_t)(g_uds.seed);
        g_respLen = 6u;
    } else {                                           /* sendKey       */
        if (len != 6u) { uds_negative(SID_SECURITY_ACCESS, 0x13u); return; }
        if (g_uds.seed == 0u) {
            uds_negative(SID_SECURITY_ACCESS, 0x24u);  /* key w/o seed  */
            return;
        }
        uint32_t key  = ((uint32_t)req[2] << 24) | ((uint32_t)req[3] << 16)
                      | ((uint32_t)req[4] << 8)  |  (uint32_t)req[5];
        uint32_t seed = g_uds.seed;
        g_uds.seed = 0u;                               /* one shot only */

        if (key == sec_expected_key(seed)) {
            g_uds.unlocked = true;  g_uds.key_attempts = 0u;
            g_resp[0] = SID_SECURITY_ACCESS + 0x40u;
            g_resp[1] = req[1];  g_respLen = 2u;
        } else {
            g_uds.key_attempts++;
            uds_negative(SID_SECURITY_ACCESS,
                         (g_uds.key_attempts >= 3u) ? 0x36u : 0x35u);
            if (g_uds.key_attempts >= 3u) { sec_start_lockout(10000u); }
        }
    }
}
```

`0x31` RoutineControl is how the bootloader gets invoked: sub-function
`0x01` startRoutine with a routine identifier such as `0xFF00`
eraseMemory. The reprogramming path itself is `10 02` — the application
enters the programming session, writes module 8's `.noinit` magic and
issues a software reset, which is exactly the guarded path the bootloader
refused to boot without.

## Automotive concerns

- **Answer within P2 or say `0x78`.** A flash erase takes hundreds of
  milliseconds. Send `0x78` immediately, run the long operation in a task,
  and send the real response when it finishes. Going quiet for 400 ms is
  indistinguishable, to a tester, from a dead ECU.
- **Session state is safety state.** Extended and programming sessions
  drive actuators and stop normal control loops. The S3 timeout, a session
  change and any reset must all return outputs to the safe state through
  *one* function, not three copies of the logic.
- **Guard with preconditions, then say `0x22`.** Vehicle speed above zero,
  engine running, battery below threshold: each is a reason to reject with
  conditionsNotCorrect. One precondition function per service keeps the
  rules reviewable.
- **Security access must survive a power-cycle attack.** The lockout timer
  and attempt counter belong in FlexNVM (module 6), not only in RAM —
  otherwise three attempts plus a reset gives unlimited tries. Keep the
  write frequency low so the counter does not become a flash-wear problem.
- **ISO-TP buffers are an attack surface.** A First Frame can claim 4095
  bytes. Check the declared length against your buffer *before* allocating
  and answer FlowStatus Overflow; never index a Consecutive Frame payload
  without re-checking the running offset.
- **Diagnostics share the bus with control traffic.** A 4 KB response at
  BlockSize 0 saturates a 500 kbit/s bus and delays safety-relevant
  frames. Keep diagnostic IDs low-priority (`0x7E8` already is), use a
  dedicated TX mailbox, and set STmin so the transfer paces itself.
- **DTC services write to flash.** `0x14` ClearDiagnosticInformation and
  `0x19` ReadDTCInformation sit directly on module 6's fault memory.
  Debounce and coalesce before persisting; a rapidly toggling fault must
  not become thousands of flash writes.

## Cheat sheet

| Item | Notes |
|------|-------|
| Positive response | request SID + `0x40`; negative is `7F SID NRC` |
| Suppress bit | Sub-function bit 7 — no positive response, negatives still sent |
| Sessions | `0x01` default · `0x02` programming · `0x03` extended · `0x04` safety |
| Timers | P2 ≈ 50 ms · P2\* ≈ 5000 ms after `0x78` · S3 = 5000 ms back to default |
| TesterPresent | `3E 00` → `7E 00`; `3E 80` suppresses the response |
| Core services | `0x10` session · `0x11` reset · `0x14`/`0x19` DTCs · `0x22`/`0x2E` DIDs · `0x27` security · `0x31` routine · `0x34`/`0x36`/`0x37` download |
| Common DIDs | `0xF186` active session · `0xF189` SW version · `0xF190` VIN · `0xF18C` serial |
| Key NRCs | `0x13` length · `0x22` conditions · `0x31` range · `0x33` locked · `0x35` bad key · `0x78` pending |
| ISO-TP PCI | `0x0L` SF · `0x1LLL` FF · `0x2S` CF · `0x3F BS ST` FC |
| FlowStatus | `0` continue · `1` wait · `2` overflow (message exceeds your buffer) |
| STmin | `0x00–0x7F` ms · `0xF1–0xF9` = 100–900 µs |
| CAN IDs | `0x7DF` functional · `0x7E0` physical request · `0x7E8` response |
| S32K plumbing | Extra FlexCAN filter entries for `0x7E0`/`0x7DF`, dedicated TX mailbox for `0x7E8` |

## Exercise

Add a UDS server to your Level 1 capstone node and prove it against a real
tester. (1) Implement ISO-TP on its own first: Single Frame both ways,
then First/Consecutive/Flow Control with configurable BlockSize and STmin.
Test it with a 40-byte message before any UDS code exists — a transport
bug found later will look like a UDS bug and cost you a day.
(2) Implement `0x3E`, `0x10` and `0x22` with the DIDs above, including
`0xF190` VIN so you exercise multi-frame responses; verify with
`cansend`/`candump` or a Python `udsoncan` script that `03 22 F1 90`
produces a First Frame, your Flow Control, and the rest. (3) Implement the
S3 timer and demonstrate it — enter the extended session, stop sending
TesterPresent, and confirm via `0xF186` that the ECU is back in default
exactly 5 s later. (4) Implement `0x2E` writing the VIN to FlexNVM, gated
behind `0x27` with a deliberately trivial key algorithm; confirm `0x33`
when locked and `0x36` after three bad keys, then power-cycle and confirm
the lockout survived. (5) Add `0x78` around a `0x31` routine that
deliberately takes 500 ms and check with a timestamped `candump` that the
pending frames arrive inside P2. Without hardware, build the whole server
against a SocketCAN `vcan0` interface — every step runs unchanged there,
and the ISO-TP state machine is the part worth getting right.
