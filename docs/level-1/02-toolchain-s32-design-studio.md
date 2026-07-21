# 02 · Toolchain Setup — S32 Design Studio

Automotive firmware is plain C compiled with an ARM cross-compiler — no
magic. What the vendor tooling adds is device headers, startup code, linker
files, driver libraries, and a debugger connection. This module sets up
**S32 Design Studio (S32DS)**, NXP's free Eclipse-based IDE for the S32
family, walks the build/debug flow, and then demystifies what the IDE does
for you by showing the same build with plain GCC and a Makefile. Even if you
have no board yet, install the tools — you can build every example in this
course and inspect the generated artifacts.

## Installing S32 Design Studio

1. Create a free account at [nxp.com](https://www.nxp.com/) (registration is
   required for the download — this is normal for automotive vendor tools).
2. Search for **"S32 Design Studio for S32 Platform"** and download the
   installer for your OS (Windows and Linux are first-class; on macOS, use a
   VM or use the plain-GCC route below).
3. During installation you'll receive a free activation code by email.
4. Launch S32DS and install the **S32K1xx development package** (and the
   S32K3 package if you want it) via *Help → S32DS Extensions and Updates*.

What you get: the **NXP GCC** ARM toolchain (a validated GCC build),
device headers and startup code for every S32K part, the **S32 SDK**
(S32K1) / **RTD** (S32K3) driver sources, example projects, a pin/clock
configuration tool, and a debugger that talks to the EVB's built-in OpenSDA
probe over USB.

## Your first project from an SDK example

The fastest correct start is an SDK example, not an empty project:

1. *File → New → S32DS Project from Example*.
2. Filter for your board (e.g. `S32K144`), pick something small like
   `hello_world` or a GPIO/blinking-LED demo.
3. Press the hammer icon (**Build**). You should end with a console line
   reporting code/data sizes and an `.elf` file in `Debug/`.
4. With an EVB connected over USB: click **Debug** (the bug icon), and S32DS
   flashes the chip and halts at `main()`. Press **Resume** — the board LED
   blinks. Use pause/step/registers/memory views freely; this visibility is
   the whole point of a debugger.

The three artifacts a build produces:

| File | What it is |
|------|-----------|
| `project.elf` | The full program with debug info — what the debugger loads |
| `project.hex` / `.srec` / `.bin` | Raw flash images — what production programmers use |
| `project.map` | The linker's report: where every function and variable landed in memory |

Open the `.map` file once now. Seeing `main` at an address like
`0x0000_0510` (flash) and your variables at `0x2000_xxxx` (SRAM) makes the
memory map real.

## The alternative: plain GCC + Makefile

Everything S32DS does can be done with the standard
[Arm GNU Toolchain](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads)
(`arm-none-eabi-gcc`) — this is also how CI servers build firmware. A
minimal Makefile for an S32K144 project looks like this:

```makefile
# --- toolchain ---
CC      = arm-none-eabi-gcc
OBJCOPY = arm-none-eabi-objcopy

# --- flags: Cortex-M4F, no OS, no hosted libc assumptions ---
CFLAGS  = -mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16 \
          -O2 -g -Wall -Wextra -ffunction-sections -fdata-sections \
          -Iinclude
LDFLAGS = -T S32K144_flash.ld -nostartfiles -Wl,--gc-sections \
          -Wl,-Map=app.map

SRCS = src/main.c src/startup.c src/system.c

app.elf: $(SRCS)
	$(CC) $(CFLAGS) $(LDFLAGS) $^ -o $@

app.hex: app.elf
	$(OBJCOPY) -O ihex $< $@
```

The two files that make bare-metal different from PC programming are the
linker script and the startup code:

**The linker script** (`S32K144_flash.ld`) tells the linker where memory is.
The S32K144 has flash at address `0x0000_0000` (512 KB) and SRAM at
`0x1FFF_8000`–`0x2000_6FFF` (64 KB, split around `0x2000_0000`). Conceptually:

```text
MEMORY
{
  m_interrupts (RX) : ORIGIN = 0x00000000, LENGTH = 0x0400  /* vector table */
  m_flash_cfg  (RX) : ORIGIN = 0x00000400, LENGTH = 0x0010  /* flash config field! */
  m_text       (RX) : ORIGIN = 0x00000410, LENGTH = ~512K   /* code + constants */
  m_data       (RW) : ORIGIN = 0x1FFF8000, LENGTH = 64K     /* stack, data, bss */
}
```

!!! warning "The flash configuration field"
    Addresses `0x400`–`0x40F` in S32K1 flash are special: the chip reads
    them at boot to decide flash security. Program the wrong value there and
    you can **permanently lock yourself out of the chip**. Vendor linker
    files place the correct unsecured pattern there — this is a big reason
    to start from SDK examples rather than a blank slate.

**The startup code** (`startup.c` / `startup_S32K144.S`) is what runs before
`main()`:

1. The Cortex-M core reads the **initial stack pointer** from address `0x0`
   and the **reset handler address** from `0x4` (the vector table).
2. The reset handler copies initialized data from flash to RAM (`.data`),
   zeroes `.bss`, optionally disables the watchdog (module 3!), sets up the
   FPU, and finally calls `main()`.

You rarely write this from scratch — but when a project crashes before
`main()`, this is where you'll be looking, so know that it exists and what
it does.

## S32DS vs plain GCC — when to use which

| | S32DS | GCC + Makefile |
|---|---|---|
| Setup effort | One installer | Assemble toolchain, linker file, startup, headers yourself |
| Debugging | Integrated, one click | GDB + OpenOCD/pyOCD, manual setup |
| Pin/clock config tools | Built-in graphical tools | Not available — write registers yourself (you'll learn this anyway) |
| CI / automation | Headless builds possible but clunky | Natural fit |
| In this course | Recommended for beginners | Shown so you understand what's underneath |

## Cheat sheet

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

## Exercise

Install S32DS (or the plain Arm GNU toolchain) and build any S32K144
example project. Then answer these from the build outputs alone, without
running anything: (1) From the `.map` file, at what address is `main`, and
how many bytes of `.text`, `.data`, and `.bss` does the program use?
(2) Find the vector table in the map — what symbol is placed at address 0?
(3) In the startup file, find the loop that zeroes `.bss` and the line that
calls `main`. If you're hardware-free, this exercise works completely —
which is exactly the point: most firmware understanding comes from reading
build artifacts, not from watching LEDs.
