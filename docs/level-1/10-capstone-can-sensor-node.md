# 10 · Capstone — Design a CAN Sensor Node

Every module so far gave you one capability: clocks, GPIO, UART, ADC, CAN,
timers/PWM, safety habits. Real ECU work is combining all of them into one
coherent, reviewable design — and that's what this capstone asks you to do.
We design (not just code) a small but complete **CAN sensor node**: it
reads a temperature every 10 ms, filters it, transmits a CAN status frame
every 100 ms, receives a command frame, drives a PWM output, and feeds a
watchdog. This is the shape of a genuinely large fraction of body and
powertrain ECUs on the road today.

## The requirement, restated precisely

Build firmware for a node that:

1. Samples a temperature sensor via **ADC** every **10 ms** (module 6).
2. **Filters** the reading (EMA) and applies **plausibility checks**.
3. Transmits a **CAN frame** with a defined signal layout every **100 ms**
   (module 7), including validity and an alive counter.
4. **Receives** a command frame that sets a target PWM duty.
5. Drives a **PWM output** (module 8) toward that target, clamped to safe
   limits and only while inputs are valid.
6. Feeds a **watchdog** (module 9) only when every task is healthy.

## Architecture

```text
                    ┌─────────────────────────────────────────┐
                    │              CAN Sensor Node             │
                    │                                           │
  Thermistor ──ADC──▶  sensor_task_10ms()                       │
   (module 6)        │        │ filtered temp, valid flag       │
                    │        ▼                                 │
                    │  app_state (temp_dC, valid, alive_ctr,    │
                    │             pwm_target_pm, cmd_fresh)     │
                    │        │              ▲                   │
                    │        ▼              │                   │
   CAN bus ◀──TX────┤  can_tx_task_100ms()  │  can_rx_isr()  ◀──── CAN bus
   (0x310, status)  │                        │  (0x311, command)   │
                    │                        │                   │
                    │        ▼                                   │
                    │  output_task_10ms() ──PWM──▶ FTM ──▶ Fan/actuator
                    │        │                                   │
                    │        ▼                                   │
                    │  health = AND of all task results          │
                    │        │                                   │
                    │        ▼                                   │
                    │  if (health) wdog_feed()                   │
                    └─────────────────────────────────────────┘
                             ▲
                        LPIT0 10 ms tick (module 8)
                        drives the whole schedule
```

Everything hangs off the module 8 10 ms tick; 100 ms work runs every 10th
tick. This is the same "cooperative scheduler off one timer" pattern from
module 8, now carrying real payload.

## CAN message layout

**Status frame — ID 0x310, node → bus, every 100 ms, DLC 8:**

| Bits | Signal | Type | Scale/Offset | Range | Notes |
|------|--------|------|--------------|-------|-------|
| 0–15 | `CoolantTemp` | int16 | 0.1 °C, offset 0 | −40.0…150.0 °C | From module 6's NTC lookup |
| 16–31 | `SupplyVoltage` | uint16 | 1 mV | 0…20 000 mV | Node's own rail health |
| 32 | `SensorValid` | bool | — | 0/1 | Plausibility result |
| 33 | `WatchdogOK` | bool | — | 0/1 | Set 0 the tick *before* a deliberate non-feed (diagnostic aid) |
| 36–39 | `AliveCounter` | uint4 | — | 0–15, wraps | Increments every transmission |
| 40–47 | `PwmActualPct` | uint8 | 1% | 0–100 | What the node is actually driving |

**Command frame — ID 0x311, bus → node, DLC 8:**

| Bits | Signal | Type | Scale/Offset | Range | Notes |
|------|--------|------|--------------|-------|-------|
| 0–7 | `PwmTargetPct` | uint8 | 1% | 0–100 | Desired output |
| 8–11 | `CmdAliveCounter` | uint4 | — | 0–15 | Sender's alive counter |

The node treats the command **stale after 300 ms without a new
`CmdAliveCounter` value** (three missed 100 ms cycles) and falls back to
the safe state (below) — the module 7/9 timeout habit, applied for real.

## State machine

```text
        power-on
           │
           ▼
     ┌───────────┐   sensor implausible          ┌────────────┐
     │   INIT    │──────────────────────────────▶│   FAULT    │
     └─────┬─────┘   3+ consecutive samples       └─────┬──────┘
           │ clocks/pins/adc/can/timers OK               │ output = safe
           ▼                                              │ (PWM = 0%)
     ┌───────────┐   sensor valid again (5 stable)        │
     │  NORMAL   │◀─────────────────────────────────────┘
     └─────┬─────┘
           │ command stale > 300 ms
           ▼
     ┌───────────┐
     │ CMD_STALE │  PWM holds last-known-good target, clamped;
     └───────────┘  status frame's WatchdogOK-style visibility
                    lets the bus see this state (add a bit if
                    you extend the layout)
```

`INIT` runs once: clock/pin/ADC/CAN/timer init, outputs forced to the safe
state *before* anything else, only then does the LPIT tick start driving
tasks — matching module 9's "initialize outputs to safe first" rule.

## Main-loop pseudocode

```c
typedef struct {
    int16_t  temp_dC;
    bool     temp_valid;
    uint8_t  pwm_target_pct;   /* from last valid command       */
    uint8_t  pwm_actual_pct;   /* what we're actually driving    */
    uint8_t  cmd_alive_last;
    uint32_t cmd_age_ticks;    /* 10 ms units since last new cmd */
    uint8_t  tx_alive;         /* our own outgoing alive counter */
} app_state_t;

static app_state_t st;

int main(void)
{
    clocks_init();
    pins_init();
    outputs_force_safe();      /* PWM = 0% before anything else  */
    uart_init(); adc_init(); can_init(); lpit_init(); wdog_init();

    uint32_t last_tick = 0;
    uint8_t  div100 = 0;

    for (;;) {
        if (g_tick_10ms != last_tick) {
            last_tick = g_tick_10ms;

            bool ok_sensor = sensor_task_10ms(&st);   /* module 6 logic  */
            bool ok_cmd    = command_watch_task_10ms(&st);
            bool ok_output = output_task_10ms(&st);    /* module 8 PWM    */

            if (++div100 >= 10u) {
                div100 = 0;
                can_tx_status_task_100ms(&st);          /* module 7        */
            }

            if (ok_sensor && ok_cmd && ok_output) {
                wdog_feed();                            /* module 9 rule   */
            }
        }
        can_rx_poll(&st);      /* or: handled in a FlexCAN RX interrupt */
        console_poll();        /* module 5, for bring-up visibility     */
    }
}
```

`command_watch_task_10ms` is the timeout logic: if the RX handler updated
`cmd_alive_last` this tick, reset `cmd_age_ticks` to 0 and adopt the new
target (clamped 0–100); otherwise increment it, and once it exceeds 30
(ticks × 10 ms = 300 ms), force `pwm_target_pct = 0` and report unhealthy —
exactly the module 7 "signal timeout" habit made concrete.

## SDK-style code skeleton

```c
/* --- sensor.c (module 6 pattern) --- */
bool sensor_task_10ms(app_state_t *s)
{
    uint16_t raw = adc0_read(ADC_CH_TEMP);
    if (raw < RAW_MIN_PLAUSIBLE || raw > RAW_MAX_PLAUSIBLE) {
        if (++s_fault_count > 5u) { s->temp_valid = false; }
    } else {
        s_fault_count = 0u;
        s->temp_dC   = ntc_temp_dC(adc_filtered(raw));
        s->temp_valid = true;
    }
    return s->temp_valid;
}

/* --- can_app.c --- */
void can_tx_status_task_100ms(app_state_t *s)
{
    uint8_t data[8] = {0};
    int16_t  t   = s->temp_dC;
    uint16_t vbat = vbat_mV(adc0_read(ADC_CH_VSENSE));

    data[0] = (uint8_t)(t & 0xFF);         data[1] = (uint8_t)(t >> 8);
    data[2] = (uint8_t)(vbat & 0xFF);      data[3] = (uint8_t)(vbat >> 8);
    data[4] = (uint8_t)((s->temp_valid ? 1u : 0u)
                       | ((uint8_t)(++s->tx_alive & 0x0Fu) << 4));
    data[5] = s->pwm_actual_pct;

    can_send_status(data);                 /* FLEXCAN_DRV_Send, ID 0x310 */
}

void can_on_command_received(const uint8_t data[8])   /* RX ISR/callback */
{
    uint8_t target = data[0];
    uint8_t alive  = data[1] & 0x0Fu;
    if (target > 100u) { return; }         /* reject implausible frame  */
    g_cmd_target = target;
    g_cmd_alive  = alive;
    g_cmd_updated = true;                  /* consumed by command_watch */
}

/* --- outputs.c (module 8 pattern) --- */
bool output_task_10ms(app_state_t *s)
{
    uint8_t target = s->temp_valid ? s->pwm_target_pct : 0u;  /* safe state */
    /* Slew-limit toward target so the actuator never steps instantly    */
    if (s->pwm_actual_pct < target)      s->pwm_actual_pct++;
    else if (s->pwm_actual_pct > target) s->pwm_actual_pct--;
    pwm_set_permille((uint16_t)s->pwm_actual_pct * 10u);
    return true;   /* an output stage that always reaches a defined state
                       is, itself, "healthy" for watchdog purposes        */
}
```

## Bench-test plan

A design isn't done until you know how you'd prove it works. With a
**CAN adapter (PCAN-USB, candleLight, or a second S32K board)** on the bus:

1. **Power-on behavior.** Confirm PWM output is 0% before any command
   arrives (probe the pin, or watch `PwmActualPct` in the status frame) —
   proves the safe-state-first init.
2. **Status frame timing.** Log with `candump -t z` (or PCAN-View); confirm
   0x310 arrives every 100 ms ±small jitter, and `AliveCounter` increments
   1→15→0 without skips.
3. **Command → output.** `cansend can0 311#320F` (target 50 = 0x32, alive
   nibble in the low bits of byte 1) and watch `PwmActualPct` slew toward
   50 over successive status frames — proves the slew limiter and command
   path.
4. **Command timeout.** Stop sending 0x311 entirely; confirm PWM decays to
   0% within 300 ms of the last accepted command — the core safety
   requirement of this whole design.
5. **Sensor fault injection.** Physically disconnect the thermistor (or, on
   the bench, short the ADC input to ground/VCC); confirm `SensorValid`
   flips to 0 within 5 samples and PWM immediately forces to 0%, regardless
   of what the command frame asks for.
6. **Watchdog proof.** Deliberately stall one task (e.g. spin-loop inside
   `sensor_task_10ms` for >100 ms in a debug build) and confirm the board
   resets — then check the RCM reset-cause register reports "watchdog," not
   "power-on" — proving the supervision chain from module 9 is truly wired
   end to end, not just present in the code.
7. **Bus-off recovery (stretch).** Deliberately create a bus fault (e.g.
   disconnect termination) and confirm the node re-synchronizes and resumes
   transmitting once the bus is healthy again, rather than hanging forever.

## Cheat sheet

| Item | Notes |
|------|-------|
| Schedule | One 10 ms LPIT tick drives sensor/output/watchdog; every 10th tick drives CAN TX |
| Status frame | 0x310, 100 ms, temp/vbat/valid/alive/actual-PWM |
| Command frame | 0x311, target PWM% + alive nibble |
| Command timeout | 300 ms (3 missed cycles) → force target to 0% |
| Safe state | Outputs forced to 0% at boot, and on any sensor/command fault |
| Slew limiting | Actual PWM steps 1%/10 ms toward target — no instant jumps |
| Watchdog feed rule | Only when sensor+command+output tasks all report healthy |
| Bench tools | PCAN-USB / candleLight + `candump`/`cansend`, or SocketCAN `vcan0` for dry runs |
| Proof, not assumption | Every safety claim above has a matching bench step that would catch it failing |

## Exercise — and course wrap-up

Implement this capstone as a single buildable S32DS (or Makefile) project,
combining modules 2–9 unchanged where possible. Then write a one-page
**design note** (the deliverable real teams produce before code review):
the CAN layout table above, the state diagram, and a "known limitations"
section — is there a race between the RX callback and the main loop reading
`g_cmd_target`? What happens if two nodes are accidentally flashed with the
same CAN ID? Answering those two questions honestly is the real skill this
level was building toward. From here, Level 2's FlexCAN deep dive, eDMA,
FreeRTOS, and UDS diagnostics pick up exactly where this capstone's rough
edges are — go find out how production firmware closes them.
