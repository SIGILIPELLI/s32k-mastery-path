# FreeRTOS on S32K

Level 1's capstone ran a cooperative scheduler off one LPIT tick: a
`for(;;)` loop calling `sensor_task_10ms()`, `output_task_10ms()`, and a
100 ms CAN transmit. That design is genuinely good — it is deterministic,
trivially reviewable, and used in shipping ECUs. It stops being good when
one task needs to *wait* (an SPI transfer, a flash erase, a UDS response)
while another has a hard 1 ms deadline. **FreeRTOS** buys you preemption
and blocking primitives at the cost of stack budgets, priority reasoning,
and a new class of bug. This module covers getting it running on the S32K,
the task structure a real ECU uses, and the two mistakes that cause almost
every FreeRTOS field failure.

## Getting FreeRTOS onto the part

S32 Design Studio ships FreeRTOS alongside the S32 SDK: when you create a
project, select **FreeRTOS** as the RTOS instead of bare-metal, and the
Cortex-M4F port plus `FreeRTOSConfig.h` are generated for you. Three
hardware resources get claimed:

| Resource | Used for |
|----------|----------|
| `SysTick` | The RTOS tick (typically 1 ms — `configTICK_RATE_HZ = 1000`) |
| `PendSV` | Context switch, at the lowest interrupt priority |
| `SVCall` | Starting the first task |

The SDK drivers themselves are RTOS-agnostic because they sit on top of
**OSIF** (`osif.h`), a thin adaptation layer. `OSIF_TimeDelay()` becomes
`vTaskDelay()` under FreeRTOS and a busy-wait bare-metal;
`OSIF_SemaWait()` becomes a real semaphore instead of a spin. This is why
a blocking call like `LPSPI_DRV_MasterTransferBlocking` yields the CPU
under an RTOS but burns it bare-metal — same source code, different
behaviour, and worth knowing before you measure CPU load.

## Task design for an ECU

Priorities in an ECU follow deadlines, not importance. A 1 ms current-loop
task outranks a safety-critical 100 ms diagnostic task, because missing a
1 ms deadline is what breaks. A workable structure for a node like the
Level 1 capstone:

| Task | Priority | Period / trigger | Stack (words) |
|------|----------|------------------|---------------|
| `can_rx_task` | 4 (highest) | Blocks on a queue fed by the FlexCAN ISR | 256 |
| `control_task` | 3 | 1 ms periodic | 256 |
| `sensor_task` | 2 | 10 ms periodic | 256 |
| `comm_task` | 1 | 100 ms — CAN TX, UDS | 384 |
| `diag_task` | 0 (lowest) | Idle-time NVM writes, statistics | 512 |

```c
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"

static void sensor_task(void *pv)
{
    TickType_t last = xTaskGetTickCount();
    (void)pv;

    for (;;) {
        /* Fixed 10 ms period regardless of how long the body takes —
           vTaskDelay() would drift, vTaskDelayUntil() does not.      */
        vTaskDelayUntil(&last, pdMS_TO_TICKS(10));

        uint16_t raw = adc0_read(ADC_CH_TEMP);
        app_publish_temp(ntc_temp_dC(adc_filtered(raw)));
        task_alive_set(TASK_BIT_SENSOR);      /* watchdog supervision */
    }
}

void app_start(void)
{
    xTaskCreate(sensor_task,  "sensor",  256u, NULL, 2u, NULL);
    xTaskCreate(control_task, "control", 256u, NULL, 3u, NULL);
    xTaskCreate(can_rx_task,  "canrx",   256u, NULL, 4u, NULL);
    xTaskCreate(comm_task,    "comm",    384u, NULL, 1u, NULL);
    vTaskStartScheduler();                     /* never returns        */
}
```

`vTaskDelayUntil` versus `vTaskDelay` is not a style preference.
`vTaskDelay(10)` sleeps ten ticks *after the body finishes*, so a task
whose body varies between 1 ms and 4 ms runs at an 11–14 ms period that
drifts against everything else on the bus. `vTaskDelayUntil` holds the
period. Every periodic ECU task uses it.

## Queues: the ISR-to-task handoff

Module 1 ended with the rule "copy the CAN frame and return." Here is
where the frame goes. A queue is the natural boundary between interrupt
context and task context, and FreeRTOS provides a separate API for the
ISR side:

```c
static QueueHandle_t g_canRxQueue;      /* holds flexcan_msgbuff_t     */

/* --- FlexCAN event callback: interrupt context --- */
static void can_event_cb(uint8_t instance, flexcan_event_type_t event,
                         uint32_t buffIdx, flexcan_state_t *state)
{
    BaseType_t higherPriorityTaskWoken = pdFALSE;
    (void)buffIdx; (void)state;

    if (event == FLEXCAN_EVENT_RXFIFO_COMPLETE) {
        (void)xQueueSendFromISR(g_canRxQueue, (const void *)&g_rxMsg,
                                &higherPriorityTaskWoken);
        FLEXCAN_DRV_RxFifo(instance, &g_rxMsg);       /* re-arm        */
    }
    /* If the queue unblocked a higher-priority task, switch to it on
       exit from this ISR rather than waiting for the next tick.      */
    portYIELD_FROM_ISR(higherPriorityTaskWoken);
}

/* --- task context --- */
static void can_rx_task(void *pv)
{
    flexcan_msgbuff_t msg;
    (void)pv;

    for (;;) {
        if (xQueueReceive(g_canRxQueue, &msg, portMAX_DELAY) == pdTRUE) {
            app_dispatch_frame(&msg);       /* parse, scale, store     */
            task_alive_set(TASK_BIT_CANRX);
        }
    }
}

void can_rx_init(void)
{
    g_canRxQueue = xQueueCreate(16u, sizeof(flexcan_msgbuff_t));
    configASSERT(g_canRxQueue != NULL);
}
```

Note the queue depth. Sixteen frames at 500 kbit/s is roughly 4 ms of
worst-case back-to-back traffic — enough to absorb a burst while
`can_rx_task` is preempted, and cheap in RAM. Sizing it is a real
calculation: *(worst-case burst rate) × (worst-case task response time)*.

## Interrupt priorities: the mistake everyone makes once

This is the single most common FreeRTOS-on-Cortex-M failure, and on the
S32K the numbers are specific. The Cortex-M4 in the S32K144 implements
**4 priority bits**, so priorities are `0` (highest) through `15`
(lowest), stored in the upper bits of each NVIC priority byte.

The rule: **an ISR may only call `...FromISR()` APIs if its priority
number is greater than or equal to `configMAX_SYSCALL_INTERRUPT_PRIORITY`**
— that is, if it is *logically lower priority*. An ISR at a higher
priority than that threshold can preempt the kernel's critical sections;
calling a FreeRTOS API from it corrupts the scheduler in a way that
manifests hours later as a random hard fault.

```c
/* FreeRTOSConfig.h — S32K144, __NVIC_PRIO_BITS = 4 */
#define configPRIO_BITS                          4
#define configKERNEL_INTERRUPT_PRIORITY          (15 << (8 - configPRIO_BITS))
#define configMAX_SYSCALL_INTERRUPT_PRIORITY     ( 5 << (8 - configPRIO_BITS))
#define configASSERT(x)  if ((x) == 0) { taskDISABLE_INTERRUPTS(); for(;;); }
```

With that configuration: the FlexCAN interrupt must be set to priority
**5–15** if its handler touches a queue. An interrupt that must *not* be
delayed by the kernel — a hardware fault line, say — can sit at 0–4, but
it may not call any FreeRTOS function at all. Set every interrupt priority
explicitly at init; the reset default of 0 puts every peripheral in the
forbidden zone.

Keep `configASSERT` enabled in development builds. The FreeRTOS port
contains assertions that catch exactly this misconfiguration, and they
turn a week of debugging into an immediate stop at a known line.

## Automotive concerns

- **Static allocation, no heap.** Set `configSUPPORT_DYNAMIC_ALLOCATION`
  to 0 and use `xTaskCreateStatic` / `xQueueCreateStatic`. Automotive
  coding standards prohibit dynamic allocation after init; static creation
  also makes RAM usage a link-time fact rather than a runtime hope.
- **Stack overflow is silent by default.** Enable
  `configCHECK_FOR_STACK_OVERFLOW = 2` and implement
  `vApplicationStackOverflowHook()` — in production it should reset via
  the watchdog path, not spin. Size stacks by measuring: run the worst
  case, then read the high-water mark with `uxTaskGetStackHighWaterMark()`
  and keep a documented margin.
- **Watchdog supervision must be per-task.** Replace the capstone's
  single conditional feed with an alive-bitmask: each task sets its bit
  each period, and a supervisor task feeds the WDOG only when *all* bits
  are set, then clears them. A task that dies silently now causes a reset
  instead of a quiet loss of function — this is the RTOS version of module
  9's rule and it is not optional.
- **Priority inversion is real.** A low-priority task holding a mutex the
  highest-priority task needs blocks it indefinitely unless you use
  `xSemaphoreCreateMutex()`, which implements priority inheritance.
  Binary semaphores do not inherit — use them for signalling, mutexes for
  mutual exclusion.
- **Determinism still has to be proven.** An RTOS does not give you
  real-time behaviour; it gives you the tools to build it. You still owe a
  worst-case response time argument for every deadline: interrupt latency
  plus higher-priority task execution plus your own body.
- **Do not fight the RTOS with critical sections.** Long
  `taskENTER_CRITICAL()` regions destroy interrupt latency for everything,
  including the CAN error handling from module 1. Keep them to a few
  instructions, or use a queue instead.

## Cheat sheet

| Item | Notes |
|------|-------|
| Hardware claimed | `SysTick` (tick), `PendSV` (switch), `SVCall` (first task) |
| SDK glue | OSIF — `OSIF_TimeDelay`/`OSIF_SemaWait` map to RTOS or bare-metal |
| Periodic task | `vTaskDelayUntil(&last, pdMS_TO_TICKS(n))` — never `vTaskDelay` |
| ISR → task | `xQueueSendFromISR(...)` then `portYIELD_FROM_ISR(woken)` |
| Queue depth | worst-case burst rate × worst-case task response time |
| Priority bits | S32K144 Cortex-M4: **4 bits**, values 0 (highest) … 15 (lowest) |
| ISR API rule | Only ISRs at priority ≥ `configMAX_SYSCALL_INTERRUPT_PRIORITY` may call `...FromISR` |
| Explicit priorities | Set every NVIC priority at init — reset default 0 is illegal for RTOS-aware ISRs |
| Allocation | `xTaskCreateStatic`/`xQueueCreateStatic`; no heap after init |
| Stack checking | `configCHECK_FOR_STACK_OVERFLOW = 2` + `uxTaskGetStackHighWaterMark` |
| Watchdog | Alive bitmask across all tasks → supervisor feeds only when complete |
| Mutual exclusion | `xSemaphoreCreateMutex` (inherits priority), not a binary semaphore |

## Exercise

Port the Level 1 capstone from its cooperative loop to FreeRTOS, and prove
the port did not cost you determinism. (1) Create the five tasks from the
table above with static allocation, using `vTaskDelayUntil` for every
periodic one. (2) Replace the CAN RX polling with the ISR → queue → task
chain, and set the FlexCAN NVIC priority explicitly to a legal value —
then deliberately set it to 0, enable `configASSERT`, and observe the
assertion fire so you have seen the failure mode once. (3) Implement the
alive-bitmask watchdog supervisor: every task sets its bit, a supervisor
task at priority 1 feeds the WDOG only when all bits are set. Kill one
task with `vTaskSuspend()` from a debug command and confirm the board
resets and that RCM reports a watchdog reset. (4) Measure and record the
high-water mark of every stack after a 10-minute run under load, and
document your chosen margin. (5) Write a short comparison: for *this*
node, is the RTOS actually better than the Level 1 loop? Argue it with
your measured numbers — the ability to answer "no" with evidence is worth
more than the port itself.
