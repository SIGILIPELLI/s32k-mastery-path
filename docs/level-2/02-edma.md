# eDMA — Direct Memory Access

Every peripheral lesson so far has moved data with the CPU: poll a flag,
read a register, store a byte. That works until the numbers get awkward —
a 1 kHz ADC scan of eight channels is 8000 interrupts per second, and each
one costs you context save, restore, and a hole in your timing analysis.
The **eDMA** engine moves data without the core: you describe the transfer
once, a peripheral request triggers it, and the CPU finds the buffer
already full. This module covers channels, TCDs, the DMAMUX, linking DMA
to the ADC and LPUART, ping-pong buffering — and the honest answer to when
DMA is *not* worth it.

## Channels, DMAMUX and TCDs

Three pieces of hardware cooperate:

- **eDMA engine** — 16 channels on the S32K144, each with its own
  **TCD** (Transfer Control Descriptor) in the `DMA->TCD[n]` register
  block. The TCD is the transfer: source, destination, offsets, sizes,
  and the two loop counters.
- **DMAMUX** — routes one of many peripheral request lines onto a channel.
  `DMAMUX->CHCFG[n]` holds a `SOURCE` field and an `ENBL` bit. A channel
  with no source can still be triggered by software.
- **Channel arbitration** — when several channels request at once, the
  engine picks by fixed priority or round-robin, configured through
  `DMA->CR` and the per-channel priority registers.

The TCD's two nested loops are the concept worth internalizing:

```text
minor loop : NBYTES bytes moved per request
             SADDR += SOFF, DADDR += DOFF after each transfer
major loop : CITER requests until the loop completes
             then SADDR += SLAST, DADDR += DLASTSGA, optional interrupt
```

So "read 2 bytes from the ADC result register on every conversion, 8
conversions per scan, then interrupt" is: `NBYTES = 2`, `SOFF = 0` (the
result register does not move), `DOFF = 2` (the buffer does), `CITER = 8`,
`DLASTSGA = -16` (rewind the buffer pointer), interrupt on major loop
completion.

## A software-triggered memory transfer

Start with the simplest case to get the initialization shape right:

```c
#include "edma_driver.h"

#define DMA_CH_MEMCPY   0u

static edma_state_t     dmaState;
static edma_chn_state_t chnState[1];

static const edma_channel_config_t chnCfg = {
    .channelPriority = EDMA_CHN_DEFAULT_PRIORITY,
    .virtualChannel  = DMA_CH_MEMCPY,
    .source          = EDMA_REQ_DISABLED,   /* software-triggered only */
    .callback        = NULL,
    .callbackParam   = NULL,
    .enableTrigger   = false,
};

static const edma_user_config_t dmaCfg = {
    .chnArbitration = EDMA_ARBITRATION_ROUND_ROBIN,
    .haltOnError    = false,   /* one bad channel must not stop the rest */
};

static const edma_channel_config_t *const chnCfgArr[1] = { &chnCfg };
static edma_chn_state_t            *const chnStateArr[1] = { &chnState[0] };

void dma_init(void)
{
    EDMA_DRV_Init(&dmaState, &dmaCfg, chnStateArr, chnCfgArr, 1u);
}

void dma_block_copy(const uint8_t *src, uint8_t *dst, uint32_t bytes)
{
    EDMA_DRV_ConfigMultiBlockTransfer(DMA_CH_MEMCPY, EDMA_TRANSFER_MEM2MEM,
                                      (uint32_t)src, (uint32_t)dst,
                                      EDMA_TRANSFER_SIZE_1B,
                                      1u,        /* bytes per request   */
                                      bytes,     /* total byte count    */
                                      true);     /* stop when finished  */
    EDMA_DRV_StartChannel(DMA_CH_MEMCPY);
    EDMA_DRV_TriggerSwRequest(DMA_CH_MEMCPY);
}
```

## ADC → memory, triggered by hardware

This is the pattern that earns its keep in an ECU. The LPIT triggers the
PDB, the PDB triggers an ADC scan, the ADC's completion raises a DMA
request, and DMA parks the result in a buffer. The CPU is not involved
until the scan is complete:

```text
LPIT ch0 (1 ms)  →  PDB0 pre-trigger  →  ADC0 conversion
                                              │ DMA request
                                              ▼
                                   eDMA ch1 → g_adcBuf[8]
                                              │ major-loop interrupt
                                              ▼
                                        sensor task
```

```c
#include "adc_driver.h"
#include "pdb_driver.h"

#define DMA_CH_ADC   1u
#define ADC_SCAN_LEN 8u

static volatile uint16_t g_adcBuf[ADC_SCAN_LEN];
static volatile bool     g_adcScanReady;

static void adc_dma_cb(void *param, edma_chn_status_t status)
{
    (void)param;
    if (status == EDMA_CHN_ERROR) {
        app_set_dtc(DTC_DMA_TRANSFER_ERROR);
        return;
    }
    g_adcScanReady = true;      /* consumed by the 1 ms sensor task */
}

void adc_dma_init(void)
{
    adc_converter_config_t cfg;
    ADC_DRV_InitConverterStruct(&cfg);
    cfg.resolution           = ADC_RESOLUTION_12BIT;
    cfg.trigger              = ADC_TRIGGER_HARDWARE;
    cfg.dmaEnable            = true;
    cfg.continuousConvEnable = false;
    ADC_DRV_ConfigConverter(0u, &cfg);
    ADC_DRV_AutoCalibration(0u);

    EDMA_DRV_InstallCallback(DMA_CH_ADC, adc_dma_cb, NULL);
    EDMA_DRV_SetChannelRequestAndTrigger(DMA_CH_ADC,
                                         (uint8_t)EDMA_REQ_ADC0, false);

    /* Peripheral → memory: source fixed at ADC0->R[0], dest walks
       the buffer, 2 bytes per request, 16 bytes per major loop.  */
    EDMA_DRV_ConfigMultiBlockTransfer(DMA_CH_ADC, EDMA_TRANSFER_PERIPH2MEM,
                                      (uint32_t)&(ADC0->R[0]),
                                      (uint32_t)g_adcBuf,
                                      EDMA_TRANSFER_SIZE_2B,
                                      2u,
                                      sizeof(g_adcBuf),
                                      false);   /* keep the request live */
    EDMA_DRV_StartChannel(DMA_CH_ADC);
}
```

Note `disableReqOnCompletion = false`: the channel stays armed so the next
scan flows without CPU intervention. That is the difference between "DMA
saved me some interrupts" and "DMA runs this data path autonomously."

## Ping-pong buffers with scatter-gather

A single buffer has an obvious race: the application reads it while DMA is
already overwriting it. The classic fix is **ping-pong** — two buffers,
DMA fills one while software drains the other. eDMA supports it natively
with **scatter-gather**: each TCD's `DLASTSGA` points at the next TCD, so
the engine reloads a new descriptor at the end of every major loop.

```c
static uint16_t bufA[ADC_SCAN_LEN], bufB[ADC_SCAN_LEN];

/* Two descriptors that point at each other; each raises the callback
   when its half is complete, so software always owns the other one.  */
static edma_transfer_config_t tcdA, tcdB;

static void pingpong_init(void)
{
    tcdA.srcAddr             = (uint32_t)&(ADC0->R[0]);
    tcdA.destAddr            = (uint32_t)bufA;
    tcdA.srcTransferSize     = EDMA_TRANSFER_SIZE_2B;
    tcdA.destTransferSize    = EDMA_TRANSFER_SIZE_2B;
    tcdA.srcOffset           = 0;
    tcdA.destOffset          = 2;
    tcdA.minorByteTransferCount = 2u;
    tcdA.interruptEnable     = true;
    tcdA.scatterGatherEnable = true;
    tcdA.scatterGatherNextDescAddr = (uint32_t)&tcdB;
    /* tcdB is the mirror image, pointing back at &tcdA */
}
```

The rule that makes ping-pong safe: **the callback must only publish a
pointer, never copy under a deadline.** Set `g_readyBuf = bufA`, return,
and let the consumer task read it before the *next* major loop completes.
If your consumer cannot guarantee that, you need three buffers or a
slower trigger — not a longer ISR.

## When DMA is the wrong answer

DMA is not free. It costs setup complexity, a channel, and a class of bug
that is genuinely hard to debug. Skip it when:

- The transfer is short and infrequent. A 2-byte SPI register read at
  10 Hz does not need a DMA channel; a polled transfer is smaller, easier
  to review, and just as fast in wall-clock terms.
- You need the data *transformed* on arrival. DMA moves bytes; if every
  sample needs scaling and plausibility checks anyway, the CPU is already
  in the loop.
- The channel budget is tight. Sixteen channels sounds like plenty until
  ADC0, ADC1, two LPUARTs, LPSPI TX/RX, and the FlexCAN RX FIFO all want
  one.

## Automotive concerns

- **Channel conflicts are a design-time problem.** Two modules that each
  "just grab channel 3" produce a fault that only appears when both
  features are enabled. Keep a **single channel allocation table** in a
  shared header — one `#define` per channel, reviewed like a pin map.
  This is the DMA equivalent of Level 1's pin-muxing discipline.
- **Arbitration and worst-case latency.** With fixed priority, a
  high-priority channel doing long minor loops can delay a safety-relevant
  transfer. Round-robin bounds the delay but removes your ability to
  prioritize. Whichever you choose, the number you must be able to defend
  is the *worst-case* completion time of the most critical channel.
- **Errors are silent unless you look.** `DMA->ES` records the error
  source (address misalignment, source/destination bus error, priority
  conflict) and `EDMA_DRV_GetChannelStatus` reports `EDMA_CHN_ERROR`.
  A DMA error means your buffer contains stale data that *looks* valid —
  always set a fault flag and invalidate the sample.
- **Alignment is a hardware requirement.** A 2-byte transfer needs
  2-byte-aligned addresses; a 4-byte transfer needs 4. Misalignment raises
  a configuration error rather than transferring garbage — which is the
  merciful outcome, but only if you check.
- **`volatile` is mandatory on DMA buffers.** The compiler has no idea the
  memory changes behind its back; without `volatile` an optimized build
  can cache a stale value in a register. This is a real, frequently seen
  field bug.
- **Feed the watchdog from real work, not DMA completion.** A DMA channel
  cheerfully keeps filling a buffer while your application logic is
  wedged. Module 9's rule holds: health comes from tasks, not hardware.

## Cheat sheet

| Item | Notes |
|------|-------|
| Channels | 16 on S32K144, each with a TCD in `DMA->TCD[n]` |
| DMAMUX | `DMAMUX->CHCFG[n]`: `SOURCE` selects the request, `ENBL` turns it on |
| Minor loop | `NBYTES` per request; `SOFF`/`DOFF` advance the pointers |
| Major loop | `CITER` requests, then `SLAST`/`DLASTSGA` rewind + optional interrupt |
| Transfer types | `EDMA_TRANSFER_PERIPH2MEM`, `MEM2PERIPH`, `MEM2MEM`, `PERIPH2PERIPH` |
| Arbitration | `EDMA_ARBITRATION_FIXED_PRIORITY` or `..._ROUND_ROBIN` |
| Keep armed | `disableReqOnCompletion = false` for continuous peripheral streams |
| Ping-pong | Two TCDs chained via `scatterGatherNextDescAddr` |
| Errors | `DMA->ES` + `EDMA_DRV_GetChannelStatus` → `EDMA_CHN_ERROR` |
| Alignment | Address alignment must match transfer size, or you get a config error |
| Buffers | Always `volatile`; publish pointers from callbacks, never copy |
| Allocation | One shared channel map header — conflicts are a design defect |

## Exercise

Build a DMA-driven sensor front end and prove it is faster than the polled
version. (1) Write the channel allocation table for a node that uses
ADC0 scanning, LPUART1 TX logging, and the FlexCAN RX FIFO — assign
channels and justify each priority. (2) Implement the LPIT → PDB → ADC0 →
eDMA chain above for a 4-channel, 1 ms scan into a ping-pong pair, with
the callback publishing a buffer pointer and nothing else. (3) Measure the
win: toggle a GPIO in the old per-conversion ISR and in the new major-loop
callback, and compare CPU time on a scope or with a cycle counter —
report the actual numbers, not an estimate. (4) Inject a fault: set the
destination buffer to an odd address, confirm the driver reports an error
rather than silently corrupting memory, and make your callback set a DTC
and invalidate the scan. If you have no board, still do step (1) and write
the TCD field values (SADDR, SOFF, NBYTES, CITER, DLASTSGA) by hand for
the 4-channel scan — being able to read a TCD dump in a debugger is the
skill this module is really teaching.
