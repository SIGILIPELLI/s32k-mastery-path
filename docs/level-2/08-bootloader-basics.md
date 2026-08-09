# Bootloader Basics & Reprogramming

Every ECU on a vehicle can be reflashed without opening the housing. That
is not a convenience feature — it is a warranty, recall and cybersecurity
requirement. The code that makes it possible is a **bootloader**: a small,
self-contained program that owns the first sectors of flash, decides
whether the application is trustworthy, and, when a tester asks, erases
and rewrites everything except itself. Module 6 gave you the flash
controller; this module turns it into a reprogramming path that survives a
power cut in the middle of a download.

## What the bootloader must guarantee

1. **It always runs first** — the reset vector belongs to the bootloader.
2. **It never erases itself** — its sectors are write-protected in
   hardware, so a bug in the reprogramming logic cannot remove the only
   path back to a working ECU.
3. **It only starts an image it has verified** — a CRC or signature
   checked *before* the jump, not after.
4. **It is recoverable from any interruption** — power lost mid-download
   leaves the ECU in the bootloader, never booting half an application.

Promise 4 separates a demo from a product, and it falls out of one
ordering rule: **erase the validity marker first, program the image, and
program the marker last.**

## Flash layout and the linker

| Region | Address range | Size | Contents |
|--------|---------------|------|----------|
| Boot block | `0x0000_0000`–`0x0000_7FFF` | 32 KB | Bootloader; its vector table at `0x0000_0000` |
| Application | `0x0000_8000`–`0x0007_EFFF` | 476 KB | App vector table at `0x0000_8000`, then code |
| App descriptor | `0x0007_F000`–`0x0007_FFFF` | 4 KB | Magic, version, length, CRC — its own sector |
| FlexNVM | `0x1000_0000`–`0x1000_FFFF` | 64 KB | Calibration, DTCs — survives reprogramming |

Two details make or break this layout. **The descriptor gets its own 4 KB
sector**: P-Flash erases a whole sector at a time, so if the marker shared
a sector with code, invalidating it would erase code and the safe ordering
above would be impossible. And **the application's vector table must be
aligned** — `S32_SCB->VTOR` ignores low address bits, and an S32K144 table
needs at least 1024-byte alignment, which `0x0000_8000` satisfies
comfortably. In the application's linker file, move `m_interrupts` and
`m_text` to `0x00008000` and shrink their lengths; nothing in the
application source changes.

## The application descriptor

```c
#define APP_BASE        0x00008000u
#define APP_DESC_ADDR   0x0007F000u
#define APP_MAGIC       0xA5C3F00Du

typedef struct {
    uint32_t magic;      /* APP_MAGIC, written only when complete      */
    uint32_t version;    /* packed major/minor/patch/build             */
    uint32_t length;     /* bytes of image covered by the CRC          */
    uint32_t crc32;      /* CRC-32 over [APP_BASE, APP_BASE + length)  */
    uint32_t reserved[3];
    uint32_t descCrc32;  /* CRC-32 over the fields above               */
} app_descriptor_t;      /* 32 bytes — a multiple of the 8-byte phrase */

static bool app_image_valid(void)
{
    const app_descriptor_t *d = (const app_descriptor_t *)APP_DESC_ADDR;

    if (d->magic != APP_MAGIC)                    { return false; }
    if ((d->length == 0u) ||
        (d->length > (APP_DESC_ADDR - APP_BASE))) { return false; }

    static const crc_user_config_t crcCfg = {
        .crcWidth           = CRC_BITS_32,
        .polynomial         = 0x04C11DB7u,   /* CRC-32 / Ethernet      */
        .seed               = 0xFFFFFFFFu,
        .writeTranspose     = CRC_TRANSPOSE_BITS_AND_BYTES,
        .readTranspose      = CRC_TRANSPOSE_BITS_AND_BYTES,
        .complementChecksum = true,
    };
    (void)CRC_DRV_Init(INST_CRC, &crcCfg);
    CRC_DRV_WriteData(INST_CRC, (const uint8_t *)APP_BASE, d->length);
    return (CRC_DRV_GetCrcResult(INST_CRC) == d->crc32);
}
```

Using the CRC peripheral rather than a software loop is faster and, more
usefully, means you and the tester are running the same polynomial.

!!! warning "A CRC is an integrity check, not a security check"
    CRC-32 catches corrupted downloads and half-written images. It does
    nothing against an attacker, who can simply recompute it. Production
    ECUs verify a **signature** — CSEc on S32K1, HSE on S32K3, or a
    software ECDSA check — before the jump. Build the CRC path first
    because you need it anyway, then add the signature.

## Jumping to the application

```c
typedef void (*app_entry_t)(void);

static void boot_jump_to_app(void)
{
    const uint32_t *vt       = (const uint32_t *)APP_BASE;
    uint32_t        appStack = vt[0];                  /* initial MSP   */
    app_entry_t     appEntry = (app_entry_t)vt[1];     /* Reset_Handler */

    INT_SYS_DisableIRQGlobal();

    /* Every peripheral back to reset state — the application must not
       inherit a half-configured FlexCAN or a running LPIT.            */
    FLEXCAN_DRV_Deinit(INST_CAN0);
    LPIT_DRV_Deinit(INST_LPIT0);
    LPUART_DRV_Deinit(INST_LPUART1);

    /* Clear pending NVIC state: a latched interrupt would fire the
       instant the application enables interrupts, into a handler it
       has not installed yet.                                         */
    for (uint32_t i = 0u; i < 8u; i++) {
        S32_NVIC->ICER[i] = 0xFFFFFFFFu;
        S32_NVIC->ICPR[i] = 0xFFFFFFFFu;
    }
    S32_SysTick->CSR = 0u;

    S32_SCB->VTOR = APP_BASE;
    __asm volatile ("dsb");
    __asm volatile ("isb");
    __asm volatile ("msr msp, %0" : : "r" (appStack));
    __asm volatile ("msr psp, %0" : : "r" (appStack));

    INT_SYS_EnableIRQGlobal();
    appEntry();                                /* never returns        */
}
```

Because the stack pointer is rewritten, this function must have no live
locals afterwards — `appEntry()` is a tail call by construction. The other
trap is the **watchdog**: if the bootloader armed WDOG with a short
timeout and application startup exceeds it, the ECU resets in a loop that
looks exactly like a bad image. Disable WDOG before the jump, or give the
application a documented startup budget and re-arm accordingly.

## Deciding where to go

```c
#define BOOT_REQ_MAGIC   0x5A5AA5A5u

/* A linker section startup code does NOT zero, so it survives a
   software reset while RAM keeps its contents.                    */
volatile uint32_t g_bootRequest __attribute__((section(".noinit")));

void boot_main(void)
{
    clocks_init();
    (void)nvm_init();                    /* module 6: FLASH_DRV_Init   */

    bool stayInBoot = false;
    if (g_bootRequest == BOOT_REQ_MAGIC) {      /* 1. app requested it */
        g_bootRequest = 0u;
        stayInBoot = true;
    } else if (!app_image_valid()) {            /* 2. nothing to run   */
        stayInBoot = true;
    }
    if (!stayInBoot) { boot_jump_to_app(); }

    outputs_force_safe();                /* module 9: safe state first */
    can_init_boot_ids();
    boot_server_run();                   /* returns when download ends */

    S32_SCB->AIRCR = S32_SCB_AIRCR_VECTKEY(0x5FAu)
                   | S32_SCB_AIRCR_SYSRESETREQ_MASK;
    for (;;) { }
}
```

Notice what is *not* on that list: a "press a key within three seconds"
window. On a vehicle nobody is pressing a key, and a fixed delay on every
power-up is dead time an OEM will reject. The application requests the
bootloader; the bootloader does not wait around hoping.

## The reprogramming sequence

The wire protocol is UDS — module 9 covers the framing — but the flash
side is the same whatever the transport, and the ordering *is* the safety
argument:

```c
status_t boot_erase_application(void)
{
    /* Invalidate FIRST. From here on the ECU will not start the
       application, so an interruption below is always recoverable.  */
    status_t st = FLASH_DRV_EraseSector(&g_ssd, APP_DESC_ADDR,
                                        FEATURE_FLS_PF_BLOCK_SECTOR_SIZE);
    if (st != STATUS_SUCCESS) { return st; }

    for (uint32_t a = APP_BASE; a < APP_DESC_ADDR;
         a += FEATURE_FLS_PF_BLOCK_SECTOR_SIZE) {
        st = FLASH_DRV_EraseSector(&g_ssd, a,
                                   FEATURE_FLS_PF_BLOCK_SECTOR_SIZE);
        if (st != STATUS_SUCCESS) { return st; }
        wdog_feed();                     /* erases take milliseconds  */
    }
    return STATUS_SUCCESS;
}

status_t boot_write_block(uint32_t addr, const uint8_t *data, uint32_t len)
{
    if ((addr < APP_BASE) || ((addr + len) > APP_DESC_ADDR)) {
        return STATUS_ERROR;             /* never write our own block */
    }
    if (((addr % 8u) != 0u) || ((len % 8u) != 0u)) {
        return STATUS_ERROR;             /* 8-byte phrase rule        */
    }
    return FLASH_DRV_Program(&g_ssd, addr, len, data);
}
```

The bootloader must not fetch instructions from the P-Flash block it is
erasing. On S32K144 the boot region and the application share one block,
so **the command-launch routine has to execute from RAM**; the SDK
provides `START_FUNCTION_DEFINITION_RAMSECTION` /
`END_FUNCTION_DEFINITION_RAMSECTION` for exactly this. Confirm in the map
file that the symbol actually landed in SRAM — skip this and the first
erase hard-faults or hangs, with the debugger pointing at an address that
no longer contains code.

## Automotive concerns

- **Protect the boot block in hardware.** `FPROT0–3` guard 1/32-sized
  P-Flash regions; configure them in the bootloader's own startup so
  protection is re-established every boot. A range check in C is a good
  first line; `FPVIOL` is the one that holds when the C is wrong.
- **Flash wear applies to the reprogramming path.** P-Flash endurance is
  in the low thousands of cycles (module 6). A rig that reflashes every
  ten minutes for a week will exhaust a part — count cycles in FlexNVM.
- **Voltage during programming is a hard requirement.** Writing at a
  sagging rail produces marginal cells that read back fine on the bench
  and fail at −40 °C two years later. Refuse below the PMC's low-voltage
  threshold and reject explicitly rather than trying and hoping.
- **CAN error handling still applies while flashing.** A bus-off during a
  download must abort cleanly and leave the descriptor invalid. Never
  "resume" across a bus-off — you cannot prove which blocks arrived.
  Module 1's rate-limited recovery applies unchanged.
- **The bootloader is safety-relevant even when idle.** The ISO 26262
  argument is that a corrupt application can never start and that outputs
  sit in their safe state the whole time the bootloader runs — module 9's
  rule, applied to the code that runs before your application exists.
- **Never reprogram a moving vehicle.** The precondition check
  (stationary, ignition on, engine off) belongs in the application, the
  only party that knows. The bootloader trusts nobody and refuses unless
  the request arrived through the guarded path.
- **Keep a compatibility field.** `version` encodes which bootloader
  generation the image expects, so a mismatched flash is rejected by the
  ECU rather than discovered in the field.

## Cheat sheet

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

## Exercise

Build a bootloader you would trust with a field update. (1) Split your
Level 1 capstone into a 32 KB bootloader at `0x0000_0000` and the capstone
relocated to `0x0000_8000`, with nothing in the bootloader but the CRC
check; prove the relocation by confirming the LPIT tick interrupt still
fires in the application. (2) Add the descriptor sector plus a host script
that CRCs a `.bin` and appends the descriptor, so producing a flashable
image is one command. (3) Implement erase and program over CAN with a
three-command protocol — erase, write block, finalise — then reflash with
a build whose LED blinks at a different rate; the changed blink is your
end-to-end proof. (4) Now attack it: remove power during the erase, during
the block writes, and between the last block and the descriptor write. All
three must come back in the bootloader and accept a fresh download; if any
boots the application, your ordering is wrong. (5) Set `FPROT` to protect
the boot block and deliberately ask `boot_write_block` to write
`0x0000_1000` — confirm `FSTAT[FPVIOL]` instead of a successful write.
Without hardware, do steps (1)–(2) as a linker and checksum exercise
verifiable from the map file, and write the power-loss state table from
step (4) as a design note; that table is what a reviewer actually reads.
