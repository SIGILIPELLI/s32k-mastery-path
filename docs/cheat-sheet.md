---
title: "S32K Automotive Embedded Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free S32K Automotive Embedded cheat sheet: 40 quick-reference tables from every level of the S32K Automotive Embedded Mastery Path, each linked to its full lesson."
---

# S32K Automotive Embedded Cheat Sheet

Quick-reference tables from **40 lessons** of the free S32K Automotive Embedded Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · The Automotive Embedded Landscape &amp; the S32K Family](level-1/01-automotive-landscape-s32k.md)

| Term | Meaning |
|------|---------|
| ECU | Electronic Control Unit — one embedded computer in the car |
| Domain | Grouping of ECUs: body, chassis, powertrain, ADAS, infotainment |
| CAN | Controller Area Network — the dominant in-vehicle bus (module 7) |
| AEC-Q100 | Automotive IC qualification standard; Grade 1 = −40 to +125 °C |
| ISO 26262 | Functional-safety standard for road vehicles |
| ASIL | Automotive Safety Integrity Level, A (lowest) to D (highest) |
| AUTOSAR | Standardized automotive software architecture (Level 3 topic) |
| S32K1xx | Cortex-M4F generation — S32K144 is this course's reference part |
| S32K3xx | Cortex-M7 generation, lockstep cores, HSE security, ASIL D |
| S32K144EVB | ~$50 eval board: RGB LED, 2 buttons, potentiometer, CAN transceiver, OpenSDA debugger |
| Renode | Free open-source emulator with S32K support |

### [02 · Toolchain Setup — S32 Design Studio](level-1/02-toolchain-s32-design-studio.md)

| Item | Notes |
|------|-------|
| S32 Design Studio | Free NXP IDE (registration required); includes GCC, SDK, debugger |
| New project | *File → New → S32DS Project from Example* → filter `S32K144` |
| OpenSDA | The EVB's built-in USB debug probe — no external hardware needed |
| `arm-none-eabi-gcc` | Standard ARM cross-compiler; flags: `-mcpu=cortex-m4 -mthumb -mfloat-abi=hard` |
| `.elf` / `.hex` / `.map` | Debug image / flash image / memory-layout report |
| Linker script | Defines flash (`0x0000_0000`) and SRAM (`0x1FFF_8000`, 64 KB) regions |
| Startup code | Vector table, `.data` copy, `.bss` zero, then `main()` |
| Flash config field | `0x400`–`0x40F` — never hand-edit; wrong values can lock the chip |

### [03 · Clocks, Power &amp; Boot](level-1/03-clocks-power-boot.md)

| Item | Notes |
|------|-------|
| Boot clock | FIRC, 48 MHz — what you're running on until you configure SCG |
| SOSC | External crystal (8 MHz on EVB); required reference for accurate CAN |
| SPLL | PLL from SOSC: 8 MHz → 160 MHz out → 80 MHz core via DIVCORE |
| Clock-switch pattern | enable → wait valid flag → switch RCCR → verify CSR |
| PCC | Per-peripheral clock gate (CGC) + functional clock select (PCS) |
| Access before CGC set | Hard fault — the #1 beginner crash on S32K |
| WDOG at reset | **Enabled**, ~1 ms timeout — startup must handle it or the chip reboot-loops |
| WDOG unlock / refresh keys | `0xD928C520` / `0xB480A602` |
| Run modes | RUN (80 MHz) · HSRUN (112) · VLPR (low power) · STOP/VLPS (sleep) |

### [04 · GPIO &amp; Pin Muxing](level-1/04-gpio-pin-muxing.md)

| Item | Notes |
|------|-------|
| PORT vs GPIO | PORTx->PCR[n] = pin *function/pulls*; PTx = pin *data/direction* |
| Make pin GPIO | `PORTx->PCR[n] = PORT_PCR_MUX(1)` (after PCC clock enable!) |
| Output | `PTx->PDDR |= bit`, then PSOR/PCOR/PTOR to drive |
| PSOR / PCOR / PTOR | Atomic set / clear / toggle — prefer over PDOR read-modify-write |
| Input | PDDR bit = 0 (default); read `PTx->PDIR` |
| Internal pull-up | PCR bits `PE=1, PS=1` |
| EVB RGB LED | PTD15 red, PTD16 green, PTD0 blue — **active-low** |
| EVB buttons | SW2 = PTC12, SW3 = PTC13 — read high when pressed (board wiring) |
| Debounce | Sample every ~10 ms, accept after 2–3 stable samples |

### [05 · UART Communication](level-1/05-uart-communication.md)

| Item | Notes |
|------|-------|
| Frame 8N1 | start(0) + 8 data LSB-first + stop(1) = 10 bit-times per byte |
| Baud formula | `baud = func_clk / ((OSR+1) × SBR)` — compute actual, keep error < ~2% |
| EVB console | LPUART1, PTC6 RX / PTC7 TX (ALT2), via OpenSDA USB → 115200-8N1 |
| PCC for LPUART | Needs PCS (functional clock, e.g. FIRCDIV2) **and** CGC |
| TDRE flag | TX register empty → OK to write `DATA` |
| RDRF flag | RX register full → read `DATA` (read clears it) |
| Enable | `CTRL = TE | RE` after BAUD is set |
| SDK calls | `LPUART_DRV_Init`, `LPUART_DRV_SendDataPolling`, `LPUART_DRV_ReceiveDataPolling` |
| Garbage output | Baud/clock mismatch first, then mux, then wiring |

### [06 · ADC &amp; Sensor Inputs](level-1/06-adc-sensor-inputs.md)

| Item | Notes |
|------|-------|
| Resolution | 12-bit: 0–4095 counts over 0–Vref (5 V on EVB → ~1.22 mV/count) |
| EVB potentiometer | PTC14 = ADC0 channel SE12; analog is ALT0, no mux write needed |
| Start conversion | Write `ADC0->SC1[0] = ADC_SC1_ADCH(ch)` |
| Done? | Poll `COCO` in SC1[0]; read result from `ADC0->R[0]` |
| Calibration | Run once at init (`ADC_DRV_AutoCalibration`) — free accuracy |
| Scaling | Integer math: `(raw × range) / 4095`; convert to units immediately |
| NTC thermistor | Nonlinear → lookup table + interpolation |
| EMA filter | `filt += (raw − filt) >> 3` |
| Plausibility | Near-rail readings = wiring fault → debounce, substitute, DTC |

### [07 · CAN Fundamentals](level-1/07-can-fundamentals.md)

| Item | Notes |
|------|-------|
| Physical layer | Twisted pair CAN_H/CAN_L, differential, 120 Ω termination × 2 |
| Dominant / recessive | 0 beats 1 on the wire — basis of arbitration and ACK |
| Frame | ID + DLC + 0–8 data bytes + CRC + ACK |
| ID | Message identity **and** priority — lower wins arbitration |
| Arbitration | Bitwise, non-destructive; losers retry automatically |
| ACK | Some receiver validated the CRC — not an end-to-end delivery receipt |
| Bus-off | Nodes with persistent errors remove themselves — fault containment |
| Signals / DBC | Payload = packed scaled signals; `raw = (phys − offset)/scale` |
| Alive counter | Incrementing field letting receivers detect frozen senders |
| FlexCAN | S32K's CAN peripheral: mailboxes with ID+mask filters, HW retransmission |
| Bit timing | 500 kbit/s typical; sample point aligned across nodes; crystal clock only |
| No hardware? | Linux SocketCAN `vcan0` + `candump`/`cansend` |

### [08 · Timers &amp; PWM](level-1/08-timers-pwm.md)

| Item | Notes |
|------|-------|
| LPIT | 4-channel countdown timer → periodic interrupts; the ECU heartbeat |
| Tick math | `TVAL = period × func_clk − 1` (10 ms @ 48 MHz = 479 999) |
| ISR rules | Clear the flag (write-1-to-clear MSR), stay short, `volatile` shared data |
| NVIC | Peripheral interrupt must also be enabled in the ARM core's NVIC |
| FTM PWM | freq = clk/(PS×(MOD+1)); duty = CnV/(MOD+1); edge-aligned = MSB|ELSB |
| Update duty | Write `CONTROLS[n].CnV` |
| PWM frequency choice | LEDs ~500 Hz; motors/fans 16–25 kHz (audibility); heaters ~Hz |
| Input capture | Timestamp edges with the FTM counter → period & duty measurement |
| Unsigned wrap | `uint16_t delta = now − prev;` handles counter wraparound for free |

### [09 · Safety &amp; Robustness Basics](level-1/09-safety-robustness.md)

| Item | Notes |
|------|-------|
| ISO 26262 | Functional-safety standard; hazard → S/E/C assessment → ASIL |
| ASIL | A (lowest) … D (highest); QM = no safety requirement |
| Safe state | Defined degraded/off condition every fault path must reach |
| S32K positioning | K1: up to ASIL B systems · K3: up to ASIL D (lockstep) |
| WDOG | Feed from **main loop only**, conditionally on task health |
| WDOG keys | Unlock `0xD928C520`, refresh `0xB480A602`; window mode catches early feeds |
| Reset cause | Check RCM status at boot — know *why* you're restarting |
| CRC uses | Flash image, NVM records, E2E protection of CAN signals |
| ECC | HW corrects 1-bit, detects 2-bit errors; double-bit ⇒ fault, not garbage |
| Detect chain | detect → contain → safe state → report |
| E2E on CAN | Payload CRC + alive counter defeats stale/frozen-sender bugs |

### [10 · Capstone — Design a CAN Sensor Node](level-1/10-capstone-can-sensor-node.md)

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

## Level 2 · Intermediate

### [FlexCAN Deep Dive — Mailboxes &amp; Filtering](level-2/01-flexcan-deep-dive.md)

| Item | Notes |
|------|-------|
| MB count | S32K144: CAN0 = 32 MBs, CAN1/CAN2 = 16 — fewer with 64-byte FD payloads |
| MB layout | CS (CODE/IDE/RTR/DLC) · ID · DATA; write CODE **last** |
| RX CODEs | EMPTY `0b0100` → FULL `0b0010` → OVERRUN `0b0110` if not drained |
| Lock rule | Reading CS locks the MB; read TIMER to unlock |
| Mask polarity | 1 = must match, 0 = don't care |
| Standard IDs | Shift by 18 in ID and mask words (`0x7FF << 18`) |
| Mask scopes | `RXMGMASK` global · `RXIMR[n]` individual (`MCR[IRMQ]`) · `RXFGMASK` FIFO |
| RX FIFO | 6 deep, occupies MB0–5, filter table from MB6, size via `CTRL2[RFFN]` |
| FIFO flags | `IFLAG1` bit 5 available · bit 6 warning · bit 7 **overflow = frame lost** |
| Error counters | `ECR` TXERRCNT/RXERRCNT; ≥128 error-passive, TX >255 bus-off |
| Bus-off recovery | 128 × 11 recessive bits; `MCR[BOFFREC]` **cleared** = automatic |
| Callback discipline | Copy and return — no parsing, no blocking, no logging in the ISR |

### [eDMA — Direct Memory Access](level-2/02-edma.md)

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

### [LPSPI &amp; LPI2C — Off-Chip Peripherals](level-2/03-lpspi-lpi2c.md)

| Item | LPSPI | LPI2C |
|------|-------|-------|
| Wires | SCK, SOUT, SIN, PCS0–3 | SDA, SCL (open drain + pull-ups) |
| Addressing | One chip select per device | 7-bit (or 10-bit) address on the wire |
| Duplex | Full — every TX byte returns an RX byte | Half — direction set per transfer |
| Typical rate | 1–10 MHz | 100 kHz / 400 kHz / 1 MHz |
| Ack / errors | None on the wire | Per-byte ACK/NACK, arbitration detection |
| Init | `LPSPI_DRV_MasterInit` | `LPI2C_DRV_MasterInit` |
| Blocking transfer | `LPSPI_DRV_MasterTransferBlocking` | `..._MasterSendDataBlocking` / `..._MasterReceiveDataBlocking` |
| Repeated start | n/a — use `isPcsContinuous` | `sendStop = false` on the write phase |
| DMA | `LPSPI_USING_DMA` + rx/tx channel fields | `LPI2C_USING_DMA` + `dmaChannel` |
| Signature failure | Wrong CPOL/CPHA → bit-shifted garbage | Slave holds SDA low → bus hang |
| Recovery | Re-init, re-transfer | Nine SCL pulses + manual STOP |
| Typical devices | SBC, external EEPROM, gate drivers | Temperature/pressure sensors, EEPROMs |

### [FreeRTOS on S32K](level-2/04-freertos-s32k.md)

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

### [Low-Power Modes &amp; Wakeup Sources](level-2/05-low-power-modes.md)

| Item | Notes |
|------|-------|
| Controllers | **SMC** (mode control), **PMC** (regulators, LVD) |
| Mode set | HSRUN · RUN · VLPR · WAIT · VLPW · STOP1/STOP2 · VLPS |
| No LLS/VLLS | S32K1 removed them — VLPS is the deepest mode, state is retained |
| `SMC->PMPROT` | Write-once at boot; permits the modes you may later enter |
| VLPR limit | Core ≤ 4 MHz from SIRC — switch clocks *before* the mode change |
| SDK entry | `POWER_SYS_Init` then `POWER_SYS_SetMode(idx, POWER_MANAGER_POLICY_AGREEMENT)` |
| Resume point | Code after `SetMode` runs *after* the wakeup ISR |
| Periodic wake | LPTMR from the 1 kHz LPO tap (`LPTMR_CLOCKSOURCE_1KHZ_LPO`) — alive in VLPS |
| Pin wake | PORT interrupts are asynchronous; filter needs `PORTx->DFCR` on LPO |
| CAN self-wake | `MCR[SLFWAK]` + `CTRL1[WAKMSK]` — wakes on *any* bus activity |
| Pretended Networking | `MCR[PNET_EN]`, `CTRL1_PN`, `FLT_ID1`, `WU_MTC` — wake on a *matching* frame |
| Biggest saving | Transceiver standby, then pin states, then the MCU mode |

### [Flash, FlexNVM &amp; EEPROM Emulation](level-2/06-flash-flexnvm-eeprom.md)

| Item | Notes |
|------|-------|
| Controller | **FTFC**; commands via `FCCOB0…B`, launched by clearing `FSTAT[CCIF]` |
| Error bits | `FSTAT`: `ACCERR` (bad command/alignment), `FPVIOL` (protection), `MGSTAT0` (verify) |
| S32K144 sizes | P-Flash 512 KB / 4 KB sectors · FlexNVM 64 KB / 2 KB sectors · FlexRAM 4 KB |
| Program unit | 8-byte phrase, 8-byte aligned — no byte writes, no rewriting a phrase |
| Erase unit | One sector |
| Read-while-write | Cannot fetch from the block being written — data in FlexNVM, code in P-Flash |
| Watchdog | Wire `flash_user_config_t.CallBack` to feed during long operations |
| SDK calls | `FLASH_DRV_Init`, `EraseSector`, `Program`, `ProgramCheck`, `EEEWrite`, `DEFlashPartition` |
| EEE | FlexRAM window backed by FlexNVM; byte-granular writes, journaled by hardware |
| Partitioning | `FLASH_DRV_DEFlashPartition` — end-of-line or bootloader only |
| Endurance | ~10³ P-Flash / ~10⁴ D-Flash cycles order-of-magnitude — confirm in the data sheet |
| Wear strategy | Write on change + deadband · RAM shadow flushed on a trigger · rotating CRC'd records |
| Protection | `FPROT0–3` (P-Flash), `FDPROT` (D-Flash), `FEPROT` (EEPROM window) |

### [S32K1 to S32K3 Migration](level-2/07-s32k1-to-s32k3.md)

| Item | S32K1 | S32K3 |
|------|-------|-------|
| Core | Cortex-M4F @ ≤112 MHz | Cortex-M7 @ 160 MHz+, lockstep options |
| Drivers | S32 SDK `FLEXCAN_DRV_Init` | RTD: `Can_Init` (MCAL) or `FlexCAN_Ip_Init` (IP) |
| Pins | `PORT->PCR[n]` + `PTx` | `SIUL2` `MSCR[n]` / `IMCR[n]` |
| Clocks | `SCG` + `PCC` | `MC_CGM` + `MC_ME` |
| Watchdog | `WDOG` | `SWT` |
| PWM | `FTM` | `eMIOS` |
| Periodic timer | `LPIT` | `PIT` / `STM` |
| ADC trigger | `PDB` | `BCTU` |
| Unchanged | LPUART, LPSPI, LPI2C, FlexCAN, eDMA, TRGMUX | same |
| Code flash base | `0x0000_0000` | `0x0040_0000` (ITCM now at 0) |
| Boot | Vector table at 0 | BAF validates the **IVT** first |
| Erased flash | Reads `0xFF` | **ECC error on read** |
| Safety | ECC, WDOG, clock monitor | + FCCU, MEMU, STCU2, lockstep |
| Security | CSEc | **HSE** firmware-based subsystem |

### [Bootloader Basics &amp; Reprogramming](level-2/08-bootloader-basics.md)

| Item | Notes |
|------|-------|
| Boot block | `0x0000_0000`; owns the reset vector; protected via `FPROT0–3` |
| App base | `0x0000_8000` — vector table first, alignment ≥ 1024 bytes |
| Descriptor | Own 4 KB sector: magic, version, length, CRC-32, descriptor CRC |
| Safe ordering | Erase descriptor → erase app → program app → program descriptor **last** |
| Validity check | `CRC_DRV_Init` → `CRC_DRV_WriteData` → `CRC_DRV_GetCrcResult` before the jump |
| Jump steps | Deinit peripherals → clear `S32_NVIC->ICER/ICPR` → set `S32_SCB->VTOR` → `msr msp` → call `vt[1]` |
| Stay-in-boot | `.noinit` RAM magic from the app · invalid image · never a fixed timeout |
| Software reset | `S32_SCB->AIRCR = S32_SCB_AIRCR_VECTKEY(0x5FA) \| SYSRESETREQ_MASK` |
| Reset cause | `RCM->SRS` separates POR / PIN / WDOG / software — log it in both images |
| Read-while-write | Erase/program sequence must run from RAM (`START_FUNCTION_DEFINITION_RAMSECTION`) |
| Program unit | 8-byte phrase, 8-byte aligned (module 6) |
| Watchdog | Feed inside the erase loop; disable or re-arm generously before the jump |
| Security | CRC catches corruption only — add CSEc/HSE signature verification |

### [UDS Diagnostics Intro (ISO 14229)](level-2/09-uds-diagnostics.md)

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

### [Project — Multi-Node CAN Network](level-2/10-project-multi-node-can.md)

| Item | Notes |
|------|-------|
| Topology | 3 nodes, one 500 kbit/s bus, 120 Ω at **both** physical ends only |
| ID = priority | `0x100` mode · `0x200` fault · `0x310`–`0x312` cyclic · `0x7Ex` diag |
| Periods | 100 ms cyclic, 500 ms network mode, event faults with a ≥ 100 ms gap |
| Phasing | Offset producers by 10–20 ms so cyclic frames do not collide every cycle |
| Timeout | 300 ms = three missed cycles, for every consumed signal |
| Protection | Alive counter (staleness) **and** ID-inclusive checksum (corruption) |
| Latency budget | ≈ 41 ms sample → actuator applied; recompute if a queue is added |
| Bus load | ≈ 0.7 % — compute it before anyone proposes a trace frame |
| Gateway RX | RX FIFO, 8-element format-A table, no `0x000` padding entries |
| Callback rule | Copy and re-arm; checksum, alive and range checks belong in the task |
| Fail-safe | Fan to 100 % on lost or invalid temperature — safe direction, not zero |
| Shared header | `can_matrix.h` compiled by all three nodes — one source of truth |
| Bench tools | 3 × S32K144 EVB (or 2 EVBs + PCAN-USB), `candump -t z`, `cansend` |

## Level 3 · Advanced

### [AUTOSAR Classic — MCAL, BSW &amp; RTE](level-3/01-autosar-classic.md)

| Layer | Owns | Example S32K module |
|-------|------|----------------------|
| Application (SWC) | Feature logic | `DoorControl`, `LightingManager` |
| RTE | Generated glue, all inter-SWC and SWC-BSW calls | `Rte_Write_*`, `Rte_Read_*`, `Rte_Call_*` |
| Services (BSW) | OS/HW-independent services | `NvM`, `Dcm`, `ComM`, `Dem` |
| ECU Abstraction | Decouple app from HW instance/channel | `CanIf`, `PduR`, `IoHwAb` |
| MCAL | Direct register access, standardized API | `Can`, `Port`, `Dio`, `Mcu`, `Fee` |

| Return type | Meaning | Where used |
|-------------|---------|------------|
| `E_OK` / `E_NOT_OK` | Generic success/failure | Every BSW/MCAL call |
| `Std_ReturnType` | AUTOSAR's uniform return type | RTE and BSW APIs |
| `RTE_E_INVALID` | Sender-receiver signal not yet written | Rte_Read_* checks |

| Config artifact | Generated by | Consumed by |
|------------------|---------------|-------------|
| ARXML | S32 Configuration Tools / DaVinci Configurator | RTE generator, MCAL generator |
| `Can_ConfigType` | Configuration tool, from ARXML | `Can_Init()` |

### [Real-Time Drivers (RTD) &amp; MCAL Config](level-3/02-rtd-mcal-config.md)

| Concept | RTD term | Notes |
|---------|----------|-------|
| Configuration tool | S32 Configuration Tools (S32CT) | Generates `_Cfg.c`/`_Cfg.h`, never hand-edit output |
| Init order | `Mcu` → `Port` → peripheral `_Init` → `_SetMode`/`_Start` | Wrong order = silent failure, not a build error |
| Error reporting | DET (Development Error Tracing) | On for bring-up, off for production; costs cycles |
| Error reporting | DEM (Diagnostic Event Manager) | Production runtime fault logging, feeds `Dcm`/`0x19` DTC reads |
| Vendor API | `Can_47_FlexCAN_*` | NXP-specific extensions layered on standard `Can_*` |
| ISR naming | Fixed by startup file, e.g. `CAN0_ORed_0_31_MB_IRQHandler` | Renaming silently breaks the vector table entry |
| Config data source | ARXML | Single source of truth; regenerate all dependents together |
| Standard MCAL modules | `Mcu`, `Port`, `Dio`, `Can`, `Lpuart`, `Adc`, `Fee`, `Spi` | AUTOSAR standardized API names across vendors |

### [CAN FD Networks &amp; DBC Workflows](level-3/03-can-fd-dbc.md)

| Concept | Classic CAN | CAN FD |
|---------|--------------|--------|
| Max payload | 8 bytes | 64 bytes |
| DLC 9-15 meaning | N/A (max 8) | 12, 16, 20, 24, 32, 48, 64 bytes |
| Data phase bit rate | Same as arbitration | Can switch faster if BRS=1 |
| CRC | CRC-15 | CRC-17 (≤16 bytes) / CRC-21 (>16 bytes) |
| Key control bits | — | EDL (FD frame), BRS (bit rate switch), ESI (error state) |
| S32K register area | `CTRL1`, `MB[n].CS` | `CTRL2[ISOCANFDEN]`, `FDCTRL`, `MB[n].CS[EDL,BRS]` |

| DBC token | Meaning |
|-----------|---------|
| `BO_ <id> <name>: <dlc> <sender>` | Frame definition |
| `SG_ <name> : <start>\|<len>@<order><sign> (<factor>,<offset>) [<min>\|<max>] "<unit>" <receiver>` | Signal definition |
| `@1` / `@0` | Little-endian (Intel) / big-endian (Motorola) |
| `+` / `-` | Unsigned / signed |
| `physical = raw * factor + offset` | Decode formula |

### [LIN &amp; FlexIO](level-3/04-lin-flexio.md)

| LIN field | Size | Notes |
|-----------|------|-------|
| Break | ≥13 bit times dominant | Frames the header start |
| Sync | 1 byte, always `0x55` | Lets slaves measure the bit rate |
| PID | 1 byte | 6-bit ID + 2 parity bits (`lin_pid_calc`) |
| Data | 0-8 bytes | Sent by master or the addressed slave |
| Checksum | 1 byte | Classic = data only; Enhanced (2.x) = PID + data |

| Concept | Where |
|---------|-------|
| LIN electrical layer | External LIN transceiver (e.g. TJA1027), single wire + ground |
| LIN logical layer | LPUART peripheral, Break sent via GPIO toggle or LPUART break feature |
| Diagnostic frames | `0x3C` master request, `0x3D` slave response (LIN-specific, not UDS) |
| FlexIO use here | Extra software-defined UART/LIN channel beyond fixed LPUART count |
| FlexIO building blocks | Shifters (data path) + Timers (baud/bit timing) |

### [Safety Mechanisms — SBC &amp; External Watchdogs](level-3/05-sbc-external-watchdog.md)

| Term | Meaning |
|------|---------|
| SBC | System Basis Chip — transceiver + regulator + watchdog + fault monitoring in one IC |
| Internal WDOG | MCU-integrated watchdog; detects software hangs, shares MCU clock/power domain |
| External watchdog | SBC-hosted; independent clock/supply, detects MCU-level hardware failures |
| Windowed watchdog | Service must land inside [Wmin, Wmax]; too early is a fault, not just too late |
| Challenge/response service | SBC issues a token, MCU must compute/return a derived value — defeats stuck-line false-service |
| Fault pin | Dedicated GPIO from SBC to MCU, independent of any shared communication bus |
| Common NXP SBC families | UJA116x, FS26 (safety SBC family) |
| Relevant standard | ISO 26262-5 (hardware), independent monitoring path requirement for higher ASIL |

### [MPU &amp; Freedom from Interference](level-3/06-mpu-freedom-from-interference.md)

| Term | Meaning |
|------|---------|
| FFI | Freedom from Interference — ISO 26262 requirement when mixed-ASIL software shares an MCU |
| MPU | Memory Protection Unit — hardware-enforced per-region, per-master access rules |
| Bus master | Core, DMA channel, or other initiator; MPU permissions apply per master ID |
| Region | Address range + permission set; S32K has a limited, fixed count |
| Guard region | No-access region placed past a stack to catch overflow immediately |
| MPU violation response | Isolate + report via DEM, never silently resume the faulting task |
| Relevant standard | ISO 26262-6 §7 (software architectural design, freedom from interference) |
| ASIL decomposition | Splitting a function across QM + safety partitions relies on FFI being provable |

### [Secure Boot &amp; the HSE Security Engine](level-3/07-secure-boot-hse.md)

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

### [Automotive Ethernet &amp; Gateways](level-3/08-automotive-ethernet-gateways.md)

| Term | Meaning |
|------|---------|
| 100BASE-T1 | Single twisted pair, 100 Mbit/s automotive Ethernet PHY standard |
| 1000BASE-T1 | Single pair, 1 Gbit/s variant, higher-bandwidth backbone links |
| PAM-3 | Encoding scheme automotive single-pair Ethernet uses to fit the wiring budget |
| ENET | S32K's Ethernet MAC/DMA peripheral |
| Buffer descriptor (BD) ring | DMA-managed circular list of RX/TX buffers, core only handles completed entries |
| SOME/IP | Common automotive service-oriented middleware protocol carried over Ethernet |
| PduR | AUTOSAR's protocol data unit router — the "real" layer that does gateway routing in a full stack |
| Master/slave PHY role | Automotive PHY clocking role, configured not autonegotiated — mismatch prevents link-up |

### [Measurement &amp; Calibration (XCP)](level-3/09-xcp-measurement-calibration.md)

| Term | Meaning |
|------|---------|
| XCP | Universal Measurement and Calibration Protocol (ASAM), address-based, transport-agnostic |
| A2L | ASAM MCD-2MC file mapping symbol names to memory addresses; must match the exact linked binary |
| DAQ | Data AcQuisition — continuous streamed measurement, tied to ECU event channels |
| MTA | Memory Transfer Address — the pointer XCP reads/writes are relative to, set by `SET_MTA` |
| EPK | EPROM identifier — lets a tool verify the ECU's software version matches its loaded A2L |
| Calibration page | RAM working copy of tunable constants, distinct from the committed flash copy |
| Transport | XCP-on-CAN, XCP-on-CAN FD, XCP-on-Ethernet — protocol is transport-independent |
| Common PC tools | ETAS INCA, Vector CANape |

### [Project — Diagnostic-Capable Body Controller](level-3/10-project-body-controller.md)

| Layer | Module | Role in this project |
|-------|--------|------------------------|
| Application/RTE | 1 | `DoorControl`, `LightingManager`, `WindowLift` SWCs |
| MCAL | 2 | RTD `Can`, `Port`, `Fee` drivers under the SWCs |
| Network | 3 | DBC-defined CAN FD frames for door/light/window signals |
| LIN | 4 | Window motor / mirror actuator as a LIN slave off the BCM |
| Watchdog | 5 | SBC windowed watchdog + fault pin, independent of the app core |
| Partitioning | 6 | MPU: `DoorControl` (higher integrity) vs `LightingManager` (QM) |
| Secure boot | 7 | HSE-verified image before application core release |
| Gateway | 8 | Optional Ethernet bridge to a diagnostic/test bench |
| Calibration | 9 | XCP in dev builds only, compiled out for production |

## Level 4 · Master

### [Multi-Core S32K3 &amp; Lockstep](level-4/01-multicore-lockstep.md)

| Term | Meaning |
|------|---------|
| Lockstep pair | Two physical cores executing identically, hardware-compared; software sees one core |
| FCCU | Fault Collection and Control Unit — routes detected faults (including lockstep mismatch) to a configured reaction |
| SPFM / LFM | Single-Point Fault Metric / Latent Fault Metric — ISO 26262-5 hardware metrics lockstep coverage feeds |
| Torn read | Reading a multi-word shared value mid-write by another core; sequence numbers or atomics prevent acting on it |
| DSB | Data Synchronization Barrier — Arm instruction ensuring memory ordering, not atomicity by itself |
| Symmetric multi-core | Independent cores running different workloads, for throughput/partitioning, not fault detection |
| Reaction on lockstep fault | Immediate reset/safe-state; not a recoverable software exception |
| Relevant standard | ISO 26262-5 (hardware architectural metrics), -6 §7 (FFI, now across cores) |

### [ASIL-D Development Workflow](level-4/02-asil-d-workflow.md)

| Term | Meaning |
|------|---------|
| HARA | Hazard Analysis and Risk Assessment — ISO 26262-3, derives ASIL from S/E/C |
| S / E / C | Severity / Exposure / Controllability — the three HARA input factors |
| ASIL | A (lowest) through D (highest); QM = no ISO 26262 rating required |
| FSR / TSR / SSR | Functional / Technical / Software Safety Requirement — the traceability chain |
| MC/DC | Modified Condition/Decision Coverage — required unit test rigor at higher ASIL |
| ASIL decomposition | Splitting one ASIL D requirement into independent lower-ASIL redundant paths |
| SPFM / LFM | Single-Point / Latent Fault Metric — ISO 26262-5 hardware coverage targets |
| TCL | Tool Confidence Level — ISO 26262-8 §11, qualification requirement for dev/verification tools |

### [MISRA C &amp; Static Analysis in Practice](level-4/03-misra-static-analysis.md)

| Rule (MISRA C:2012) | Category | What it catches |
|----------------------|----------|-------------------|
| 10.1/10.3 | Required | Implicit conversions between incompatible essential types |
| 8.13 | Advisory | Pointer parameter should be `const` if never written through |
| 17.7 | Required | Discarding a non-void return value without explicit `(void)` |
| 21.3 | Required | Dynamic memory allocation (`malloc`/`free`/etc.) forbidden |
| 15.5 | Advisory | Function should have a single point of exit |
| 11.3 | Required | Casting between incompatible pointer types (e.g. raw register access) |
| Deviation category | Meaning |
|----------------------|---------|
| Mandatory | No deviation ever permitted |
| Required | Deviation only with formal, documented, approved justification |
| Advisory | Deviation with lighter justification, still recorded |

### [OTA &amp; Reprogramming Stacks](level-4/04-ota-reprogramming.md)

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

### [Cybersecurity — ISO/SAE 21434 &amp; SecOC](level-4/05-cybersecurity-secoc.md)

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

### [AUTOSAR OS &amp; Timing Analysis](level-4/06-autosar-os-timing.md)

| Term | Meaning |
|------|---------|
| ISR Category 1/2 | Cat1: no OS service calls, fastest; Cat2: OS-managed, can call a subset of services |
| BCC1-4 | AUTOSAR OS Basic Conformance Classes — determine task reactivation/multiplicity rules |
| WCET | Worst-Case Execution Time — measured/analyzed, not assumed; must include cache/flash wait-state worst case |
| RTA / RMA | Response Time / Rate Monotonic Analysis — proves deadlines met via iterative formula |
| Priority Ceiling Protocol | AUTOSAR OS resource mechanism bounding priority inversion via temporary priority raise |
| Schedulability | A task set is schedulable if every task's calculated worst-case response time ≤ its deadline |
| Core pinning | AUTOSAR OS tasks are statically assigned to one core (module 1), never migrate at runtime |
| Regression risk | Adding/changing one task invalidates prior RTA results until recomputed |

### [HIL Testing &amp; CI for ECU Firmware](level-4/07-hil-testing-ci.md)

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

### [Production ECU Hardware Architecture](level-4/08-production-ecu-architecture.md)

| Hardware element | Firmware-visible consequence |
|--------------------|---------------------------------|
| Reverse-battery protection | Prevents reversed Vbat from damaging the MCU; do not reassign pins without checking the protection network |
| TVS / load dump protection | Absorbs voltage transients (e.g. alternator load dump); firmware should not assume clean power |
| SBC regulator | Powers MCU; brownout/undervoltage during cranking is normal, not exceptional |
| Buck regulator rails | 5V/3.3V typical; firmware ADC references must match the actual regulated rail, not a nominal assumption |
| High/low-side driver + current sense | Firmware must actively read current sense and shut off on stall/overcurrent — hardware alone doesn't do it |
| Crystal/oscillator | Clock accuracy for CAN bit timing (Level 3 module 3) and freshness counters (module 5) depends on it |
| EMC/PWM frequency | Dithered/spread-spectrum PWM can be required to meet CISPR 25 limits |
| Connector pin mapping | Should route through generated config (module 2), not hand-coded register writes |

### [Manufacturing, EOL Test &amp; Traceability](level-4/09-manufacturing-eol-test.md)

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

### [Capstone — Production-Grade Zone ECU](level-4/10-capstone-zone-ecu.md)

| Concern | Level 3 module | Level 4 module |
|---------|------------------|-------------------|
| Layered architecture | 1 (AUTOSAR Classic) | 6 (OS/timing on top of it) |
| Driver configuration | 2 (RTD/MCAL) | 8 (hardware it configures) |
| Networking | 3 (CAN FD/DBC), 4 (LIN), 8 (Ethernet) | 5 (SecOC on top of it) |
| Safety mechanisms | 5 (SBC watchdog), 6 (MPU) | 1 (lockstep), 2 (ASIL process) |
| Security | 7 (secure boot/HSE) | 4 (OTA), 5 (SecOC), 9 (key provisioning) |
| Tooling/process | — | 2 (ASIL workflow), 3 (MISRA), 7 (HIL/CI) |
| Field/manufacturing | — | 4 (OTA rollback), 9 (EOL test) |

---

Go deeper with the full [S32K Automotive Embedded Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
