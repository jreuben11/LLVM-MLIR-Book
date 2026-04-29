# Chapter 107 — Embedded Targets

*Part XV — Targets*

Embedded systems represent LLVM's most demanding compilation environment: microcontrollers with kilobytes of flash and RAM, no operating system, real-time response requirements, and hardware peripherals accessed directly through memory-mapped registers. Clang and LLVM have become the dominant toolchain for professional embedded development, displacing GCC across the ARM Cortex-M ecosystem, gaining ground on RISC-V bare-metal, and supporting exotic architectures like AVR and MSP430 through community backends. The embedded context exercises aspects of the compiler that desktop targets rarely stress: linker script interaction, section placement, startup code generation, code size optimization, multi-library (multilib) selection, and the subtle interactions between compiler-rt builtins, newlib/picolibc, and the vector table. This chapter covers the full landscape of LLVM embedded support, from the ABI mechanics of Cortex-M Thumb-2 through code size optimization strategies applicable to all constrained targets.

---

## 107.1 Embedded Target Landscape

### 107.1.1 What Defines an Embedded Target

An embedded target differs from a hosted target in several fundamental ways:

- **No OS**: No operating system provides system calls, memory management, or a C runtime startup. The compiler's output must include or link against startup code that initializes the hardware state before `main()`.
- **Constrained memory**: Program flash (ROM) from 16 KB to a few MB; SRAM from 2 KB to hundreds of KB. Code size matters as much as performance.
- **Bare-metal ABI**: Calling conventions exist but no dynamic linker, no `dlopen`, no process boundaries. The linker script defines the entire address space.
- **Real-time constraints**: Interrupt latency, deterministic execution timing — the compiler must not silently introduce unbounded delays via software floating-point, unexpected recursion, or heap allocation.
- **Hardware access**: Peripheral registers accessed via volatile memory-mapped I/O; the compiler must not eliminate or reorder these accesses.

### 107.1.2 LLVM-Supported Embedded Targets

| Architecture | LLVM Target | Common Hardware |
|-------------|------------|----------------|
| ARM Cortex-M0/M0+ | ARM (Thumb) | STM32F0, RP2040, nRF51 |
| ARM Cortex-M3 | ARM (Thumb-2) | STM32F1/F2/L1, LPC17xx |
| ARM Cortex-M4/M7 | ARM (Thumb-2+FPU) | STM32F4/F7/H7, i.MX RT |
| ARM Cortex-M23/M33 | ARM (ARMv8-M) | STM32L5, nRF9160, PSOC64 |
| ARM Cortex-M55/M85 | ARM (ARMv8.1-M + MVE) | i.MX RT1160, STM32H5 |
| RISC-V RV32I/E/IMAC | RISC-V | GD32VF103, SiFive FE310, ESP32-C3 |
| AVR | AVR | ATmega328P (Arduino), ATtiny, XMEGA |
| MSP430 | MSP430 | MSP430FR5969, MSP430G2553 |
| Xtensa | Xtensa | ESP32, ESP32-S2/S3 |

### 107.1.3 Toolchain Components

A complete embedded toolchain requires:

```
Clang/LLVM        -- compiler, optimizer, IR
LLD               -- linker (replaces binutils ld)
compiler-rt       -- software FP builtins, libunwind
newlib/picolibc   -- C standard library (no glibc)
LLDB              -- debugger (via GDB RSP protocol)
```

The `--sysroot` flag points Clang at the cross-compilation root:

```bash
clang --sysroot=/path/to/arm-none-eabi-sysroot \
      --target=thumbv7em-none-eabihf \
      -mcpu=cortex-m4 \
      -mfloat-abi=hard \
      -mfpu=fpv4-sp-d16 \
      -c main.c -o main.o
```

---

## 107.2 ARM Cortex-M (Thumb-2)

### 107.2.1 Cortex-M Profiles

ARM's M-profile processors form a tiered family:

| Profile | ISA | FPU | TrustZone | Typical Use |
|---------|-----|-----|-----------|------------|
| M0/M0+ | ARMv6-M (Thumb subset) | None | No | Ultra-low-power, cost-sensitive |
| M3 | ARMv7-M (Thumb-2) | None | No | General MCU, STM32F1 class |
| M4 | ARMv7E-M (Thumb-2+DSP) | Optional single | No | Motor control, audio DSP |
| M7 | ARMv7E-M (Thumb-2+DSP) | Optional double | No | High-performance MCU |
| M23 | ARMv8-M Baseline | None | Yes | Low-power with security |
| M33 | ARMv8-M Mainline | Optional single | Yes | IoT gateway, secure MCU |
| M55 | ARMv8.1-M + MVE | Double + MVE | Yes | ML inference on MCU |
| M85 | ARMv8.1-M + MVE | Double + MVE + Cache | Yes | Edge AI, high-performance |

### 107.2.2 Target Triples and Clang Flags

The ARM target triple for bare-metal Cortex-M:

```
thumbv6m-none-eabi      -- Cortex-M0/M0+
thumbv7m-none-eabi      -- Cortex-M3
thumbv7em-none-eabi     -- Cortex-M4/M7 (software FP)
thumbv7em-none-eabihf   -- Cortex-M4/M7 (hardware FP, hard-float ABI)
thumbv8m.base-none-eabi -- Cortex-M23
thumbv8m.main-none-eabi -- Cortex-M33 (no FPU)
thumbv8m.main-none-eabihf -- Cortex-M33 (hardware FPU)
```

The `eabi`/`eabihf` suffix specifies the float ABI:
- **`eabi`** (soft/softfp): float args passed in integer registers, or software FP
- **`eabihf`** (hard): float args passed in FPU registers (`s0`–`s15`), requires FPU

Key Clang flags:

```bash
# Cortex-M4 with hardware single-precision FPU
clang --target=thumbv7em-none-eabihf \
      -mcpu=cortex-m4 \
      -mfloat-abi=hard \
      -mfpu=fpv4-sp-d16 \
      -mthumb \
      -Os \
      -ffreestanding \
      -fno-exceptions \
      -fno-rtti \
      -c source.c -o source.o

# Cortex-M33 with TrustZone, secure world
clang --target=thumbv8m.main-none-eabihf \
      -mcpu=cortex-m33 \
      -mcmse \
      -mfloat-abi=hard \
      -mfpu=fpv5-sp-d16 \
      -c secure_main.c -o secure_main.o
```

### 107.2.3 AAPCS and EABI Register Model

The ARM Embedded ABI (AAPCS) defines:

- **R0–R3**: Argument/result registers (scratch)
- **R4–R11**: Callee-saved (push/pop in prologue/epilogue)
- **R12 (IP)**: Intra-procedure scratch
- **R13 (SP)**: Stack pointer (13-bit aligned in exception handlers)
- **R14 (LR)**: Link register (return address)
- **R15 (PC)**: Program counter
- **CPSR**: Status register (N, Z, C, V flags plus execution state)
- **S0–S15 / D0–D7**: FPU scratch (with VFP/FPv4)
- **S16–S31 / D8–D15**: FPU callee-saved

In Thumb-2, most instructions can access all 16 registers (R0–R15). Thumb (16-bit encoding) instructions typically only access R0–R7 (low registers).

### 107.2.4 Vector Table and Reset Handler

The Cortex-M startup sequence is critical and compiler-generated code must not interfere with it:

```c
// startup.c — minimal Cortex-M startup
extern unsigned _data_start, _data_end, _data_load;
extern unsigned _bss_start, _bss_end;
extern int main(void);

void Reset_Handler(void) {
    // Copy .data section from flash to RAM
    unsigned *src = &_data_load;
    unsigned *dst = &_data_start;
    while (dst < &_data_end) *dst++ = *src++;

    // Zero .bss section
    for (unsigned *p = &_bss_start; p < &_bss_end; ) *p++ = 0;

    // Call C library init (if using newlib/picolibc)
    // __libc_init_array();

    main();
    for (;;);  // trap if main returns
}

// Weak defaults for interrupt handlers
void Default_Handler(void) __attribute__((weak, alias("Reset_Handler")));
void NMI_Handler(void)      __attribute__((weak, alias("Default_Handler")));
void HardFault_Handler(void) __attribute__((weak, alias("Default_Handler")));

// Vector table — placed in .isr_vector section by linker script
__attribute__((section(".isr_vector")))
void (*const vector_table[])(void) = {
    (void (*)(void))(&_stack_top),  // initial stack pointer
    Reset_Handler,
    NMI_Handler,
    HardFault_Handler,
    // ... more handlers
};
```

### 107.2.5 Linker Script

```ld
/* Cortex-M4 linker script — STM32F407 */
MEMORY {
  FLASH (rx)   : ORIGIN = 0x08000000, LENGTH = 1024K
  SRAM  (rwx)  : ORIGIN = 0x20000000, LENGTH = 128K
  SRAM2 (rwx)  : ORIGIN = 0x2001C000, LENGTH = 16K
  CCM   (rwx)  : ORIGIN = 0x10000000, LENGTH = 64K   /* Core-Coupled Memory */
}

ENTRY(Reset_Handler)

SECTIONS {
  /* Vector table must be first */
  .isr_vector : {
    KEEP(*(.isr_vector))
  } > FLASH

  /* Code and read-only data */
  .text : {
    *(.text*)
    *(.rodata*)
    . = ALIGN(4);
    _data_load = .;    /* load address of .data in flash */
  } > FLASH

  /* Initialized data: loaded from flash, runs from SRAM */
  .data : AT(_data_load) {
    _data_start = .;
    *(.data*)
    . = ALIGN(4);
    _data_end = .;
  } > SRAM

  /* Zero-initialized data */
  .bss (NOLOAD) : {
    _bss_start = .;
    *(.bss*)
    *(COMMON)
    . = ALIGN(4);
    _bss_end = .;
  } > SRAM

  /* Stack: grows downward from top of SRAM */
  .stack (NOLOAD) : {
    . = ALIGN(8);
    _stack_top = ORIGIN(SRAM) + LENGTH(SRAM);
  } > SRAM

  /* CCM: time-critical code or data */
  .ccmram (NOLOAD) : {
    *(.ccmram*)
  } > CCM
}
```

Using LLD with this script:
```bash
/usr/lib/llvm-22/bin/ld.lld \
    --script=stm32f407.ld \
    --entry=Reset_Handler \
    -Map=output.map \
    crt0.o main.o libfoo.a \
    -lc -lm \
    --start-group --end-group \
    -o firmware.elf

/usr/lib/llvm-22/bin/llvm-objcopy \
    -O binary firmware.elf firmware.bin
```

### 107.2.5b Thumb-2 Instruction Encoding

Understanding Thumb-2 encoding helps when analyzing compiler output and hand-optimizing critical paths:

```asm
; 16-bit Thumb instructions (T16):
; These occupy only 2 bytes each — crucial for code size
  MOV   r0, #42      ; 0x202A — 2 bytes
  ADD   r0, r1       ; 0x4408 — 2 bytes (T2 ADD)
  LDR   r0, [r1]     ; 0x6808 — 2 bytes (word load, T1)
  PUSH  {r4-r7, lr}  ; 0x2DE0 — 2 bytes (push up to r7 + lr)
  BX    lr           ; 0x4770 — 2 bytes (return)

; 32-bit Thumb-2 instructions (T32):
; These are 4 bytes; needed for full register file or large immediates
  MOV   r8, r0       ; 0x4640 — 2 bytes (MOV high reg, T1)
  MOVW  r8, #0xABCD  ; 4 bytes (16-bit immediate)
  MOVT  r8, #0x1234  ; 4 bytes (16-bit top half — together: r8 = 0x1234ABCD)
  LDR   r8, [r0, r1] ; 4 bytes (T2 with Rn > r7 or Rm > r7)
  BL    target       ; 4 bytes (branch-link, 24-bit offset)
  IT    EQ           ; 2 bytes — IF-THEN block for conditional execution
  MOVEQ r0, #1       ; 2 bytes — executes only if EQ flag set
```

The IT (If-Then) block is a Thumb-2 mechanism for conditional execution without branches: the `IT` instruction specifies condition codes for the following 1–4 instructions. Clang uses IT blocks when converting conditional IR branches to conditional move sequences, saving a branch instruction:

```c
// C code:
int abs_val(int x) { return x < 0 ? -x : x; }

// Thumb-2 output (-Os):
abs_val:
    CMP   r0, #0          ; test x < 0
    IT    LT
    RSBLT r0, r0, #0      ; x = 0 - x  (only if LT)
    BX    lr              ; return r0
; 6 bytes total vs 8 bytes with a branch
```

### 107.2.6 ARMv8-M TrustZone (CMSE)

ARM's TrustZone for Cortex-M (Secure and Non-Secure worlds) uses the `__attribute__((cmse_nonsecure_entry))` attribute to mark functions callable from the Non-Secure world:

```c
// secure_gateway.c (compiled for Secure world)
#include <arm_cmse.h>

typedef void (*ns_callback_t)(void) __attribute__((cmse_nonsecure_call));

// This function is a Secure Gateway — callable from Non-Secure world
__attribute__((cmse_nonsecure_entry))
int secure_crypto_op(const uint8_t *ns_input, uint32_t len, uint8_t *ns_output) {
    // Verify pointers are in Non-Secure memory
    if (!cmse_is_nsfptr(ns_input) || !cmse_is_nsfptr(ns_output))
        return -1;

    // Perform crypto operation in Secure world
    aes_encrypt(ns_input, len, ns_output);
    return 0;
}
```

Clang generates a **veneer** (trampoline) in a dedicated `.gnu.sgstubs` section for each `cmse_nonsecure_entry` function. The veneer contains the `SG` (Secure Gateway) instruction that marks the only valid entry point from the Non-Secure world, followed by a branch to the actual function.

---

## 107.3 Cortex-M Code Size Optimization

### 107.3.1 -Os vs -Oz

For embedded targets, size often dominates performance. Clang provides two size optimization levels:

- **`-Os`**: Optimize for size, but not if it significantly hurts performance. Equivalent to `-O2` with a size bias: disables unrolling, reduces inlining aggressiveness.
- **`-Oz`**: Aggressive size optimization. Disables loop unrolling, SLP vectorization, inlining (except `always_inline`), tail duplication. Enables function merging. May introduce size-for-speed tradeoffs that hurt runtime.

For Cortex-M4 applications, `-Os` is the common choice; `-Oz` is used for severely flash-constrained systems.

### 107.3.2 Machine Outliner

The Machine Outliner ([Chapter 92 — The Machine Outliner](../part-14-backend/ch92-machine-outliner.md)) is particularly effective for Cortex-M because:

1. Thumb-2 instructions are 16 or 32 bits; repeated 16-bit instruction sequences are common
2. Cortex-M function call overhead is only 2–4 instructions (BL + BX LR)
3. Small repeated patterns (initialization loops, error handling paths) appear frequently in MCU firmware

```bash
# Enable outliner for Cortex-M
clang --target=thumbv7em-none-eabihf \
      -mcpu=cortex-m4 -mfloat-abi=hard \
      -Oz -moutline \
      -c firmware.c -o firmware.o
```

The outliner is enabled by default at `-Oz` for supported targets including ARM.

### 107.3.3 LTO for Dead Code Elimination

Link-Time Optimization is essential for embedded: the linker cannot eliminate unused functions from archives (only full compilation units), but LTO can see the full call graph and eliminate dead code:

```bash
# Compile with LTO
clang --target=thumbv7em-none-eabihf \
      -mcpu=cortex-m4 -mfloat-abi=hard \
      -Os -flto=thin \
      -c module.c -o module.bc

# Link with LTO
/usr/lib/llvm-22/bin/ld.lld \
      --lto-O2 \
      --script=firmware.ld \
      crt0.o module.bc main.bc \
      -lc_nano \
      -o firmware.elf
```

Pitfall: LTO with `--whole-archive` can reintroduce dead code because `--whole-archive` forces all archive members to be included. Use LLD's `--gc-sections` instead:

```bash
/usr/lib/llvm-22/bin/ld.lld \
    --gc-sections \
    --script=firmware.ld \
    -o firmware.elf *.o
```

`--gc-sections` requires that Clang emitted code with `-ffunction-sections -fdata-sections`, placing each function and variable in its own section so the linker can discard unused ones.

### 107.3.4 Per-Function Size Attributes

```c
// Force size optimization on critical size paths
__attribute__((optimize("Os")))
void rarely_called_large_function(void) { /* ... */ }

// Force outliner on function
__attribute__((minsize))
void another_function(void) { /* ... */ }

// Disable size optimization on a hot path
__attribute__((optimize("O3")))
void hot_isr(void) { /* ... */ }
```

---

## 107.4 RISC-V Embedded (RV32I/E)

### 107.4.1 RV32I and RV32E

RISC-V's 32-bit embedded profiles:

- **RV32I**: 32-bit base integer ISA, 32 registers (x0–x31)
- **RV32E**: Embedded reduced variant, 16 registers (x0–x15); saves register file area at cost of fewer caller-saved registers
- **RV32IMAC**: I + M (multiplication) + A (atomics) + C (compressed 16-bit instructions) — the typical embedded profile

The `C` (compressed) extension is critical for embedded: 16-bit encodings of common instructions reduce code size by ~25–30%, comparable to Thumb-2.

### 107.4.2 Target Triples and ABIs

| Triple | ABI | Description |
|--------|-----|-------------|
| `riscv32-unknown-elf` | `ilp32` | Standard 32-bit, no FPU |
| `riscv32-unknown-elf` | `ilp32f` | 32-bit, single-precision FPU |
| `riscv32-unknown-elf` | `ilp32d` | 32-bit, double-precision FPU |
| `riscv32-unknown-elf` | `ilp32e` | 32-bit, RV32E reduced register file |
| `riscv32-unknown-none-elf` | `ilp32` | Bare-metal, no C library assumed |

```bash
# Compile for SiFive FE310 (RV32IMAC, no FPU)
clang --target=riscv32-unknown-elf \
      -march=rv32imac \
      -mabi=ilp32 \
      -mcmodel=medlow \
      -Os \
      -ffreestanding \
      -c main.c -o main.o

# RV32E for cost-sensitive MCU
clang --target=riscv32-unknown-elf \
      -march=rv32e \
      -mabi=ilp32e \
      -Os \
      -c main.c -o main.o
```

### 107.4.3 Linker Relaxation on RISC-V

RISC-V's two-instruction `LUI + ADDI`/`JALR` sequences for far references can be relaxed to single instructions when the target is close enough. See [Chapter 98 — The RISC-V Backend Architecture](ch98-the-riscv-backend-architecture.md) for the full relaxation mechanism. For embedded, this is especially important because:

1. Compressed (`C`) instructions encode many common patterns in 16 bits
2. The GP (global pointer) relaxation converts two-instruction globals accesses to a single `lw x0, offset(gp)` instruction
3. On typical MCU code, linker relaxation reduces code size by 5–10%

Enable GP relaxation in the startup code by setting `__global_pointer$`:

```ld
/* In linker script: expose __global_pointer$ */
PROVIDE(__global_pointer$ = ADDR(.sdata) + 0x800);
```

```c
// In startup.S: load GP
.globl _start
_start:
    .option push
    .option norelax
    la gp, __global_pointer$
    .option pop
    la sp, _stack_top
    call main
```

### 107.4.4 RISC-V Zce and Code Size Extensions

The RISC-V Zce extension group (Zca, Zcb, Zcmp, Zcmt) adds further code size reduction beyond the C extension:

- **Zcb**: Additional 16-bit encodings for common operations (`c.zext.b`, `c.sext.b`, `c.not`)
- **Zcmp**: Push/pop for register save/restore in function prologues/epilogues (one 16-bit instruction replaces 4–8 32-bit instructions)
- **Zcmt**: Jump table compression for switch statements

```bash
# Enable Zcmp for smaller prologues/epilogues
clang --target=riscv32-unknown-elf \
      -march=rv32imac_zca_zcb_zcmp \
      -mabi=ilp32 \
      -Os \
      -c func.c -o func.o
```

---

## 107.5 AVR Target

### 107.5.1 AVR Architecture Peculiarities

The AVR is an 8-bit Harvard architecture processor — unique among LLVM's embedded targets in several ways:

- **Program memory (flash)** and **data memory (SRAM)** have separate address spaces. A `const` variable in C may live in flash, but reading it from C code requires special `pgmspace.h` macros on avr-gcc.
- **8-bit register file**: R0–R31, all 8-bit. Pairs R0:R1, R24:R25, R26:R27, R28:R29, R30:R31 can be used as 16-bit register pairs. The X (R26:R27), Y (R28:R29), Z (R30:R31) pairs are pointer registers.
- **16-bit address bus for data**: SRAM is limited to 64 KB in most AVR variants.
- **No hardware multiplier** on ATtiny; on ATmega328P there is an 8×8→16 hardware MUL.

### 107.5.2 LLVM AVR Backend

The AVR backend ([`llvm/lib/Target/AVR/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AVR/)) is community-maintained. The target triple `avr-unknown-unknown` is used; device selection via `-mmcu` is passed to the toolchain for linker script and startup file selection. A working compilation:

```bash
# Compile for ATmega328P (Arduino Uno) — verified on LLVM 22.1.3
/usr/lib/llvm-22/bin/clang --target=avr \
      -mmcu=atmega328p \
      -Os \
      -ffreestanding \
      -c blink.c -o blink.o

# Generate Intel HEX for flashing (via objcopy)
/usr/lib/llvm-22/bin/llvm-objcopy \
      -O ihex blink.elf blink.hex
```

The `-mmcu=atmega328p` flag sets the following compilation parameters:
- Data memory starts at address 0x0100 (for ATmega328P's SRAM)
- Program memory (flash) size = 32 KB
- The `__AVR_ATmega328P__` preprocessor define for avr-libc headers

A concrete generated assembly excerpt for a simple LED blink:

```c
// blink.c
#include <avr/io.h>
void toggle_led(void) {
    PORTB ^= (1 << 5);   // Toggle Arduino pin 13 (PB5)
}
```

```asm
; Generated Thumb-like AVR assembly (-Os)
; PORTB is at I/O address 0x05 (mapped to memory address 0x25)
toggle_led:
    in   r24, 0x05       ; r24 = PORTB (I/O port read)
    ldi  r25, 32         ; r25 = (1 << 5) = 0x20
    eor  r24, r25        ; r24 ^= r25 (XOR to toggle)
    out  0x05, r24       ; PORTB = r24
    ret
```

Note how LLVM avoids spilling to the stack for this simple function — r24 and r25 are scratch registers in the avr-gcc ABI. The I/O port is accessed via `in`/`out` instructions (1-cycle port access) rather than `ld`/`st` (2-cycle memory access), which LLVM correctly selects because the address is in the I/O space range (0x00–0x3F).

### 107.5.3 AVR Register Allocation Challenges

LLVM's AVR backend has several register allocation complexities unique to the 8-bit architecture:

**Register pair allocation**: 16-bit and 32-bit values span consecutive register pairs. The LLVM register allocator for AVR must honor these constraints, allocating `R24:R25` together for a 16-bit value, or `R22:R23:R24:R25` for a 32-bit value. This is implemented via `AVRRegisterInfo::getRegAllocationHints()` which forces contiguous pair allocation. The `AVRInstrInfo` describes pseudo-instructions for 16-bit operations (like `MOVW` which moves a register pair in a single cycle) and handles promotion from these pseudo-ops to real 8-bit instruction sequences.

**The `__zero_reg__` convention**: AVR GCC (and LLVM's AVR backend for compatibility) reserves R1 as `__zero_reg__` — a register kept permanently zero. This avoids the need for `ldi r1, 0` sequences for zero operations. LLVM marks R1 as not allocatable in `AVRRegisterInfo.td` and inserts a `CLR R1` after any `MUL` instruction that clobbers R1.

**The `__tmp_reg__` register**: R0 is used as a temporary for the compiler in some addressing modes (e.g., `elpm` for reading from extended program memory on mega AVR). It is allocatable but treated as caller-saved.

**ISR prologue/epilogue**: AVR ISRs require saving and restoring the SREG (Status Register) in addition to any used registers:

```c
volatile uint8_t overflow_count = 0;

// ISR generates complete save/restore sequence
__attribute__((signal))  // = avr-libc ISR() macro
void TIMER0_OVF_vect(void) {
    overflow_count++;
}
```

```asm
; Generated ISR prologue/epilogue
TIMER0_OVF_vect:
    push  r24            ; save R24 (used by the increment)
    in    r24, 0x3f      ; save SREG (I/O addr 63)
    push  r24
    ; ISR body:
    lds   r24, overflow_count  ; load from SRAM
    inc   r24
    sts   overflow_count, r24
    ; Epilogue:
    pop   r24
    out   0x3f, r24      ; restore SREG
    pop   r24
    reti                 ; return from interrupt (re-enables I flag)
```

The critical point is that SREG must be saved/restored around any code that modifies flags, otherwise interrupted code whose condition check spans the ISR invocation would see wrong flag values.

**`__attribute__((progmem))` and Address Space 1**: AVR's Harvard architecture means `const` data in flash (PROGMEM) requires `lpm`/`elpm` instructions to read, not `ld`. Clang for AVR uses LLVM address space 1 for flash:

```c
// Flash-resident constant array
const uint8_t lut[16] __attribute__((progmem)) = {
    0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15
};

uint8_t lookup(uint8_t idx) {
    // pgm_read_byte generates lpm Z+, r0 sequence
    return pgm_read_byte(&lut[idx]);
}
```

The `pgm_read_byte()` macro in avr-libc expands to inline assembly using `lpm` instructions. LLVM's AVR backend uses address space 1 to track flash pointers in the IR, and the target lowering emits `lpm`/`elpm` instead of `ld` for loads from address space 1.

---

## 107.6 MSP430 Target

### 107.6.1 MSP430 Architecture

The Texas Instruments MSP430 is a 16-bit ultra-low-power MCU used in battery-operated embedded systems:

- **Registers**: R0 (PC), R1 (SP), R2 (SR — Status Register), R3 (CG — Constant Generator), R4–R15 (general purpose)
- **Addressing modes**: Register, Indexed, Symbolic (PC-relative), Absolute, Indirect, Indirect with Auto-Increment, Immediate
- **Instruction set**: Orthogonal; most instructions work with any addressing mode
- **16-bit native**: All registers and ALU operations are 16-bit; 8-bit and 20-bit (MSP430X) operations available

### 107.6.2 LLVM MSP430 Backend

```bash
# MSP430 target triple
clang --target=msp430-unknown-elf \
      -mcpu=msp430 \
      -Os \
      -ffreestanding \
      -c main.c -o main.o

# MSP430X (20-bit address space, FRAM devices)
clang --target=msp430-unknown-elf \
      -mcpu=msp430x \
      -mlarge \
      -Os \
      -c main.c -o main.o
```

The MSP430 target lives in [`llvm/lib/Target/MSP430/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/MSP430/).

### 107.6.2b MSP430 Addressing Mode Excellence

MSP430's orthogonal instruction set with 7 addressing modes allows the compiler to generate compact code. The auto-increment indirect mode is particularly useful for memory-to-memory copies:

```c
// Memory copy — MSP430 generates efficient code
void fast_copy(uint16_t *dst, const uint16_t *src, uint16_t count) {
    while (count--) *dst++ = *src++;
}
```

```asm
; MSP430 generated code (approximately)
fast_copy:
    tst     r14          ; test count (R14 = third arg)
    jz      .done
.loop:
    mov     @r13+, @r12  ; *dst = *src; src++ (indirect auto-increment)
    incd    r12          ; dst += 2 (word increment)
    dec     r14
    jnz     .loop
.done:
    ret
```

The `mov @r13+, 0(r12)` pattern (load from R13 with post-increment, store to R12+offset) compiles to a 3-word (6-byte) instruction sequence. MSP430's word-addressable nature means `incd` (increment by 2 for word step) is required for pointer arithmetic on `uint16_t*`.

### 107.6.3 MSP430 ABI and Interrupts

The MSP430 calling convention:

| Register | Role |
|----------|------|
| R12, R13 | First two 16-bit arguments / return value |
| R14, R15 | Additional arguments (if 4 args) |
| R11–R4 | Callee-saved |
| R3 | Constant generator (hardware-handled, not allocatable) |
| R2 (SR) | Status register: C, Z, N, V flags + CPU mode bits |

Interrupt handlers:
```c
// MSP430 interrupt handler — uses RETI, preserves SR
#include <msp430.h>

__attribute__((interrupt(TIMER0_A0_VECTOR)))
void Timer_A0_ISR(void) {
    P1OUT ^= BIT0;   // Toggle LED on P1.0
    TA0CCR0 += 1000; // Schedule next interrupt
}

int main(void) {
    WDTCTL = WDTPW | WDTHOLD;  // Stop watchdog timer
    P1DIR |= BIT0;
    TA0CCTL0 = CCIE;
    TA0CCR0 = 1000;
    TA0CTL = TASSEL_2 | MC_2;  // SMCLK, continuous mode
    __bis_SR_register(LPM0_bits | GIE);  // Enter LPM0 + enable interrupts
    return 0;
}
```

The `__attribute__((interrupt(N)))` syntax places the function address in the corresponding interrupt vector table slot at link time.

### 107.6.4 Low-Power Mode Transitions

MSP430's primary use case is ultra-low-power IoT sensing. The `__bis_SR_register()` intrinsic enters a low-power mode and re-enables interrupts atomically:

```c
// Low-power mode entry patterns for MSP430
// LPM0: CPU off, peripherals running
__bis_SR_register(LPM0_bits | GIE);  // SR |= 0x00D0 (CPU_off, GIE)

// LPM3: CPU, MCLK, SMCLK off — only ACLK active (32.768 kHz crystal)
__bis_SR_register(LPM3_bits | GIE);  // SR |= 0x00F0

// LPM4: All clocks off — only RAM retention, port interrupts wake
__bis_SR_register(LPM4_bits | GIE);
```

These lower to the MSP430 `BIS` (bit set) instruction on the SR register — a single instruction that atomically enters the low-power mode and enables the global interrupt enable bit. The MSP430 wakes from LPM when an interrupt fires; the ISR runs at full speed and returns with `RETI`, which restores SR and thus re-enters the low-power mode (or stays awake if the ISR clears the LPM bits).

LLVM MSP430 correctly handles `__bis_SR_register` as an inline intrinsic — it does not treat it as a function call, avoiding the overhead of call setup for this critical real-time pattern.

---

## 107.7 Code Size Optimization Strategies

### 107.7.1 Size-Affecting Optimizations

Key LLVM passes and their size impact:

| Pass | Size Effect | Notes |
|------|------------|-------|
| Function inlining | Usually increases | `-fno-inline-functions` / smaller inline threshold |
| Loop unrolling | Increases | `-fno-unroll-loops` |
| Loop vectorization | Increases | `-fno-vectorize` |
| SLP vectorization | Increases | `-fno-slp-vectorize` |
| Machine Outliner | Decreases | `-moutline` (enabled at `-Oz`) |
| Function merging | Decreases | `-fmerge-functions` |
| Tail call optimization | Neutral/decrease | Reduces stack usage |
| `--gc-sections` + `-ffunction-sections` | Decreases | Removes dead functions |
| LTO | Decreases (via DCE) | Removes unused library code |
| `-Oz` vs `-O2` | Significant decrease | Trades performance for size |

### 107.7.1b Concrete Size Measurement Example

To illustrate the impact of optimization flags, consider a typical embedded firmware function with a simple FIR filter:

```c
// fir.c — 16-tap FIR filter (typical DSP workload)
float fir16(const float *x, const float *h) {
    float acc = 0.0f;
    for (int i = 0; i < 16; i++) acc += x[i] * h[i];
    return acc;
}
```

Compiling for Cortex-M4 at different optimization levels:

```bash
# -O2: 64 bytes (loop unrolled twice, SIMD pairs considered)
clang --target=thumbv7em-none-eabihf -mcpu=cortex-m4 \
      -mfloat-abi=hard -mfpu=fpv4-sp-d16 -O2 \
      -c fir.c -o fir_O2.o

# -Os: 52 bytes (loop not unrolled, but structured well)
clang --target=thumbv7em-none-eabihf -mcpu=cortex-m4 \
      -mfloat-abi=hard -mfpu=fpv4-sp-d16 -Os \
      -c fir.c -o fir_Os.o

# -Oz: 44 bytes (minimal loop, no unrolling, smaller prologue)
clang --target=thumbv7em-none-eabihf -mcpu=cortex-m4 \
      -mfloat-abi=hard -mfpu=fpv4-sp-d16 -Oz \
      -c fir.c -o fir_Oz.o
```

Checking sizes:
```bash
/usr/lib/llvm-22/bin/llvm-nm --size-sort -S fir_Oz.o | grep fir16
# 00000000 0000002c T fir16   (44 bytes)

/usr/lib/llvm-22/bin/llvm-size fir_O2.o fir_Os.o fir_Oz.o
#    text    data     bss     dec     hex filename
#      64       0       0      64      40 fir_O2.o
#      52       0       0      52      34 fir_Os.o
#      44       0       0      44      2c fir_Oz.o
```

The 31% code size reduction from `-O2` to `-Oz` is typical. For a 64 KB firmware image, `-Oz` vs `-O2` commonly saves 8–15 KB — enough to fit an additional feature or allow a smaller (cheaper) microcontroller.

### 107.7.2 Identifying Size Bottlenecks

```bash
# Per-section sizes
/usr/lib/llvm-22/bin/llvm-size -A firmware.elf

# Per-symbol sizes
/usr/lib/llvm-22/bin/llvm-nm --size-sort --print-size firmware.elf | tail -20

# Bloaty: hierarchical size analysis
bloaty firmware.elf -d compileunits,symbols

# Show which translation unit is largest
/usr/lib/llvm-22/bin/llvm-nm -S --size-sort firmware.elf | \
    /usr/lib/llvm-22/bin/llvm-addr2line -e firmware.elf | \
    sort -k2 -rn | head -20
```

### 107.7.3 Compiler Flags Summary for Minimum Size

```bash
# Minimum size compilation for Cortex-M4
clang --target=thumbv7em-none-eabihf \
      -mcpu=cortex-m4 \
      -mfloat-abi=hard \
      -mfpu=fpv4-sp-d16 \
      -mthumb \
      -Oz \                          # Aggressive size
      -ffunction-sections \          # Enables --gc-sections
      -fdata-sections \
      -flto=thin \                   # Whole-program DCE
      -moutline \                    # Function outliner
      -fno-exceptions \              # No C++ exception tables
      -fno-rtti \                    # No RTTI tables
      -fno-stack-protector \         # No stack canaries (saves code + data)
      -fomit-frame-pointer \         # R11 available for allocation
      -ffreestanding \               # No hosted C library assumptions
      source.c -o source.bc

# Link with size-focused LLD flags
/usr/lib/llvm-22/bin/ld.lld \
      --script=firmware.ld \
      --gc-sections \
      --lto-O2 \
      --icf=all \                    # Identical Code Folding
      -Map=firmware.map \
      source.bc crt0.o \
      -lc_nano \
      -o firmware.elf
```

`--icf=all` (Identical Code Folding) merges functions with identical machine code, even if they have different names — particularly effective for template instantiations in C++ firmware.

---

## 107.8 Runtime Library Selection

### 107.8.1 compiler-rt Builtins

For targets without hardware FPU or with incomplete hardware support, `compiler-rt` provides software implementations of arithmetic operations. Key builtins for ARM:

| Builtin | Description |
|---------|------------|
| `__aeabi_fadd`, `__aeabi_fsub` | ARM EABI single-precision FP add/sub |
| `__aeabi_fmul`, `__aeabi_fdiv` | Single-precision mul/div |
| `__aeabi_dadd`, `__aeabi_dsub` | Double-precision operations |
| `__aeabi_f2d`, `__aeabi_d2f` | FP conversion |
| `__aeabi_f2iz`, `__aeabi_d2iz` | FP to integer |
| `__aeabi_uidivmod` | Unsigned integer division+modulus |
| `__aeabi_ldivmod` | 64-bit division |
| `__mulsi3`, `__udivsi3` | Portable (non-EABI) equivalents |

For RISC-V:
```
__adddf3, __subdf3, __muldf3  -- double-precision (soft-float)
__addsf3, __subsf3, __mulsf3  -- single-precision
__floatsidf, __fixdfsi        -- float<->int conversion
__divdi3, __udivdi3           -- 64-bit division
```

Selecting compiler-rt:
```bash
# Explicitly select compiler-rt (default for clang cross-compilation)
clang --target=thumbv7m-none-eabi \
      --rtlib=compiler-rt \
      -c main.c -o main.o
```

### 107.8.2 C Library Choices

| Library | Size | Features | Best For |
|---------|------|---------|---------|
| glibc | Large | Full POSIX | Linux hosted (not embedded) |
| newlib | Medium (~100 KB) | Full C99 + some POSIX | ARM Cortex-M4/M7 with RAM to spare |
| picolibc | Small (~20–50 KB) | C99 + TLS | Cortex-M0/M3, RISC-V bare-metal |
| newlib-nano | Small (~30 KB) | Reduced printf/scanf | `--specs=nano.specs` (ARM GCC) |
| avr-libc | AVR-specific | C99 + AVR hardware | AVR only |
| msp430-libc | MSP430-specific | Basic C | MSP430 only |

picolibc (combining picolibc and newlib-nano lineages) is the current recommended choice for LLVM-based embedded development:

```bash
# Link with picolibc
clang --target=thumbv7em-none-eabihf \
      -mcpu=cortex-m4 -mfloat-abi=hard \
      --sysroot=/usr/arm-none-eabi-picolibc \
      -Os \
      main.c crt0.c \
      -Tfirmware.ld \
      -lc -lm \
      -o firmware.elf
```

### 107.8.3 Unwinding and Exceptions

In embedded C++ firmware, exceptions are usually disabled (`-fno-exceptions`) to avoid:
- `libunwind` (adds ~50 KB)
- LSDA (Language-Specific Data Area) tables in flash
- Frame unwinding overhead on `throw`

When C++ is needed but exceptions are not:
```bash
clang --target=thumbv7em-none-eabihf \
      -fno-exceptions \
      -fno-rtti \
      -fno-use-cxa-atexit \     # Avoid atexit() registration for statics
      -fno-threadsafe-statics \ # Remove mutex in local static init
      -c cxx_module.cpp -o cxx_module.o
```

For fault analysis (hard fault handlers), a compact unwinder from ARM's EHABI is available without full exception support:

```c
// Hard fault handler with stack unwinding for debugging
__attribute__((naked))
void HardFault_Handler(void) {
    __asm volatile(
        "TST LR, #4\n"
        "ITE EQ\n"
        "MRSEQ R0, MSP\n"
        "MRSNE R0, PSP\n"
        "B HardFault_Handler_C\n"
    );
}

void HardFault_Handler_C(uint32_t *stack) {
    // stack[0..7] = R0, R1, R2, R3, R12, LR, PC, PSR
    // saved by hardware on exception entry
    volatile uint32_t pc  = stack[6];
    volatile uint32_t lr  = stack[5];
    (void)pc; (void)lr;
    for (;;); // Set breakpoint here
}
```

---

## 107.9 Debug Info for Embedded

### 107.9.1 DWARF for Constrained Flash

DWARF debug information is large — often larger than the code itself. Embedded developers must make deliberate choices:

| Flag | Size | Content |
|------|------|---------|
| `-g0` | 0 | No debug info |
| `-g1` | Small | Line tables only (no variable info) |
| `-g` / `-g2` | Medium | Full DWARF 5 (types, variables, inlining) |
| `-g3` | Large | Full DWARF + macro info |
| `-gdwarf-4` | Medium | DWARF 4 (better toolchain compatibility) |
| `-gz=zlib` | Reduced | Compressed debug sections |

For production firmware, strip debug info from the binary while keeping a separate ELF for debugging:

```bash
# Compile with full debug info
clang --target=thumbv7em-none-eabihf -g -gdwarf-4 -Os \
      -c main.c -o main.o

# Link to ELF with debug info
/usr/lib/llvm-22/bin/ld.lld --script=fw.ld main.o -o firmware_debug.elf

# Strip debug info for flashing
/usr/lib/llvm-22/bin/llvm-objcopy \
      --strip-debug firmware_debug.elf firmware_stripped.elf

# Or: extract binary for flashing, keep ELF with symbols for GDB
/usr/lib/llvm-22/bin/llvm-objcopy \
      -O binary firmware_debug.elf firmware.bin
```

### 107.9.2 LLDB Remote Debugging

LLDB can debug embedded targets via the GDB Remote Serial Protocol (RSP):

```bash
# Start GDB server (e.g., OpenOCD)
openocd -f interface/cmsis-dap.cfg -f target/stm32f4x.cfg \
        -c "gdb_port 3333"

# Connect LLDB to OpenOCD
/usr/lib/llvm-22/bin/lldb firmware_debug.elf
(lldb) gdb-remote localhost:3333
(lldb) process connect connect://localhost:3333
(lldb) b main
(lldb) continue
(lldb) frame variable
```

Alternatively, `lldb-server gdbserver` can serve as the remote side for LLDB-to-LLDB connections:

```bash
# On device (if running Linux, e.g., Raspberry Pi as debug host)
lldb-server gdbserver :3333 -- ./test_binary

# On host
lldb
(lldb) platform select remote-linux
(lldb) platform connect connect://device:3333
```

### 107.9.3 SWO/ITM Trace

ARM's Serial Wire Output (SWO) and Instrumentation Trace Macrocell (ITM) provide non-intrusive trace output:

```c
// ITM-based printf via SWO (CMSIS)
#include "core_cm4.h"

void ITM_SendString(const char *str) {
    while (*str) {
        while (ITM->PORT[0].u32 == 0);  // wait for port ready
        ITM->PORT[0].u8 = *str++;
    }
}
```

This approach avoids UART overhead and is captured by debug probes (J-Link, ULINK, CMSIS-DAP with SWO support) without pausing the target.

---

## 107.10 Xtensa and Emerging Embedded Targets

### 107.10.1 Xtensa (ESP32) and Windowed Registers

The Xtensa LX6/LX7 processor in Espressif's ESP32 family is a configurable RISC architecture with a distinctive feature: a **windowed register file**. Rather than a fixed set of saved/caller-saved registers, Xtensa implements a rotating window into a larger physical register file.

**Windowed register calling convention**:
- Xtensa provides 16 logical registers (a0–a15) visible at any time, but 64 physical registers in the register file
- `CALL4`/`CALL8`/`CALL12` instructions both call a function and rotate the window by 4, 8, or 12 registers respectively
- The caller's registers are not copied — they simply become visible through the callee's window
- `RETW` (return with windowed restore) rotates the window back
- If all 64 physical registers are consumed (window overflow), a hardware interrupt saves registers to the stack — transparent to software

```asm
; Xtensa windowed call example
; Caller uses a0-a15 (logical), actual a8-a23 (physical)
; CALL8: rotate by 8, jump to target
; Callee sees a0-a15, which are caller's a8-a23 (physical)

main:
    ; a2-a7 = first 6 args (a0 = return addr, a1 = SP)
    call8   foo            ; a10-a15 become callee's a2-a7
    ; on return: caller's a2-a15 are restored automatically
    retw

foo:
    ; a2 = first argument (was caller's a10)
    addi    a2, a2, 1
    retw                   ; return with window restore
```

The in-tree Xtensa backend ([`llvm/lib/Target/Xtensa/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/Xtensa/)) was added in LLVM 20. Espressif maintains a downstream fork with full ESP32/S2/S3/C3 support. Compiling for Xtensa with the in-tree backend:

```bash
# Compile with LLVM in-tree Xtensa target (LLVM 22.1.3, verified)
/usr/lib/llvm-22/bin/clang --target=xtensa -Os \
    -ffreestanding \
    -S /tmp/test.c -o /tmp/test_xtensa.s
# Note: -mcpu=esp32 not recognized in upstream; use vendor fork for ESP32-specific CPUs
```

The windowed register feature is enabled via `+windowed` in the Xtensa subtarget. When the windowed ABI is selected, LLVM's Xtensa backend generates `CALL8`/`CALLX8` (call through register for indirect calls) and `RETW` rather than the non-windowed `CALL0`/`CALLX0`/`RET`. The windowed ABI is the default for ESP32 (LX6) and ESP32-S3 (LX7); Xtensa-based bare-metal targets may use either ABI.

**Call range limitation**: Xtensa's `CALL8` has an 18-bit PC-relative offset (±512 KB range). For programs larger than 1 MB, the compiler must insert stubs. Espressif's downstream fork provides a flag to force indirect calls through register for all cross-unit calls. The upstream in-tree backend generates `CALLX8` (call via register) for calls that exceed the direct range, inserting a literal pool entry with the target address. The LLVM 22 in-tree Xtensa backend does not yet expose a dedicated long-calls flag; the toolchain handles this transparently via literal pool insertion.

### 107.10.2 ARM Cortex-M55 and Helium (MVE)

Cortex-M55 introduced Helium (M-Profile Vector Extensions, MVE) — a 128-bit SIMD extension for Cortex-M targeting DSP and ML workloads:

```bash
# Cortex-M55 with Helium
clang --target=thumbv8.1m.main-none-eabihf \
      -mcpu=cortex-m55 \
      -mfpu=auto \
      -mfloat-abi=hard \
      -mvector-library=SLEEF \
      -Os \
      -c dsp_filter.c -o dsp_filter.o
```

Helium vector registers are 128-bit Q0–Q7, overlapping the existing VFP D-registers. MVE provides:
- 8/16/32-bit integer SIMD: `vmulq`, `vaddq`, `vsubq`, `vcmpq`
- Single-precision float SIMD: `vfmaq`, `vfmsq`
- Predicated execution via `vctp` (vector create tail predicate) for loop tails
- Vector loads/stores with gather/scatter: `vldrw.u32 q0, [r0, r1, uxtw #2]`

LLVM's auto-vectorizer targets MVE via the ARM TTI (Target Transform Info) when the Cortex-M55 CPU is specified, though CMSIS-DSP using manual MVE intrinsics typically outperforms auto-vectorization for DSP kernels.

### 107.10.3 RISC-V P Extension (Packed SIMD for DSP)

The RISC-V P extension adds packed SIMD instructions for DSP workloads on 32-bit and 64-bit RISC-V cores (targeting microcontrollers like the Andes N25):

```bash
# RISC-V with P extension
clang --target=riscv32-unknown-elf \
      -march=rv32imac_p \
      -mabi=ilp32 \
      -Os \
      -c dsp.c -o dsp.o
```

P extension intrinsics include `__rv_add16` (packed 16-bit add), `__rv_smul16` (signed packed 16×16→32 multiply), and `__rv_kslra16` (packed 16-bit arithmetic shift right with saturation).

---

## 107.10b Clang's Bare-Metal Toolchain Driver

### 107.10b.1 The BareMetal Toolchain Class

Clang has a dedicated `BareMetal` toolchain class in [`clang/lib/Driver/ToolChains/BareMetal.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Driver/ToolChains/BareMetal.cpp) that handles triples matching `*-*-none-*` or `*-*-elf`. It:

- Skips the OS-specific sysroot search, using `--sysroot` directly
- Selects `compiler-rt` as the default runtime library (rather than libgcc)
- Configures LLD as the default linker for supported targets
- Suppresses default C library linking unless `-lc` is explicit
- Handles multilib directory resolution via `multilib.yaml`

The toolchain is selected when the target triple has `none` as the OS component or lacks OS info entirely (e.g., `thumbv7em-none-eabihf`, `riscv32-unknown-elf`, `avr-unknown-unknown`).

### 107.10b.2 Clang Driver Flags for Embedded

Key Clang driver flags specific to embedded use:

| Flag | Effect |
|------|--------|
| `-ffreestanding` | Do not assume a hosted standard library; no `main()` requirements |
| `-fno-builtin` | Do not recognize compiler builtins (for environments where stdlib functions must be explicit) |
| `--no-standard-libraries` | Do not link any default libraries |
| `--specs=picolibc.specs` | Use picolibc startup and library (ARM-GCC convention) |
| `-mno-unaligned-access` | Disable unaligned memory access (for Cortex-M0 which traps on unaligned) |
| `-fstack-usage` | Emit `.su` file with per-function stack usage |
| `-Wstack-usage=N` | Warn if any function uses more than N bytes of stack |
| `--target-feature +cmse` | Enable ARM CMSE TrustZone (same as `-mcmse`) |

The `-fstack-usage` flag is invaluable for embedded: it produces a `.su` file listing each function's static stack frame size:

```bash
clang --target=thumbv7em-none-eabihf -mcpu=cortex-m4 \
      -mfloat-abi=hard -Os -fstack-usage -c main.c

cat main.su
# main.c:15:5:process_data     48      static
# main.c:32:5:transmit_packet  128     static
# main.c:67:5:interrupt_handler 24     static
```

Combining `-fstack-usage` output with static call graph analysis allows worst-case stack depth calculation — essential for WCET (Worst Case Execution Time) analysis in safety-critical systems.

### 107.10b.3 Pragma-Based Section Placement

For fine-grained control over code and data placement without modifying the linker script, Clang supports pragma-based section attributes:

```c
// Place time-critical ISR in CCM (Core-Coupled Memory) on STM32
__attribute__((section(".ccmram")))
void ADC1_IRQHandler(void) {
    // Executes from CCM, not flash — zero wait states
    adc_value = ADC1->DR;
}

// Place lookup table in flash (explicit even when not const)
__attribute__((section(".rodata.lut"), used))
static const uint16_t sin_table[256] = { /* ... */ };

// Place DMA buffer in SRAM with 32-byte alignment for DMA engine
__attribute__((section(".dma_buffers"), aligned(32)))
static uint8_t dma_rx_buf[512];
```

The `used` attribute prevents dead-code elimination from removing a section that is only referenced indirectly (e.g., via a linker script `KEEP`). This is important for ISR vector tables and other compile-time-constant structures that the compiler would otherwise optimize away.

## 107.11 Multilib and Toolchain Configuration

### 107.11.1 Multilib

Embedded toolchains must provide pre-built libraries for every combination of ABI, FPU, and ISA variant. This combinatorial explosion is managed by **multilib**: a directory structure containing libraries built for each variant, with the compiler selecting the correct one based on flags:

```
/usr/arm-none-eabi/lib/
    libgcc.a            -- armv4t, soft-float
    thumb/v6-m/nofp/
        libgcc.a        -- armv6-m (Cortex-M0), no FPU
    thumb/v7-m/nofp/
        libgcc.a        -- armv7-m (Cortex-M3), no FPU
    thumb/v7e-m/nofp/
        libgcc.a        -- armv7e-m (Cortex-M4), no FPU
    thumb/v7e-m/fpv4-sp-d16/hard/
        libgcc.a        -- armv7e-m, FPv4 SP, hard-float ABI
    thumb/v8-m.main/nofp/
        libgcc.a        -- armv8-m.main (Cortex-M33), no FPU
    ...
```

Clang selects the appropriate multilib variant automatically when given the right `-mcpu`, `-mfloat-abi`, and `-mfpu` flags. The selection logic is in [`Clang::ConstructJob`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Driver/ToolChains/Gnu.cpp) in the ARM toolchain driver.

### 107.11.2 Clang Multilib Configuration

Clang supports a YAML-based multilib configuration introduced in LLVM 15, replacing the previous GCC-compatible multilib selection:

```yaml
# multilib.yaml for a custom embedded toolchain
MultilibVersion: 1.0
Groups:
  - Name: target_cpu
    Type: Exclusive
    Flags: [-mcpu=cortex-m0, -mcpu=cortex-m3, -mcpu=cortex-m4,
            -mcpu=cortex-m4 -mfloat-abi=hard]
Multilibs:
  - Dir: thumb/v6-m
    Flags: [-mcpu=cortex-m0]
  - Dir: thumb/v7-m
    Flags: [-mcpu=cortex-m3]
  - Dir: thumb/v7e-m/nofp
    Flags: [-mcpu=cortex-m4]
  - Dir: thumb/v7e-m/fpv4-sp-d16/hard
    Flags: [-mcpu=cortex-m4, -mfloat-abi=hard]
```

This is used by the Arm GNU Toolchain (from Arm Ltd.) and is the recommended approach for custom toolchain distributions targeting multiple Cortex-M variants.

---

## 107.12 Safety-Critical and Certified Embedded Toolchains

### 107.12.1 Functional Safety Requirements

Embedded systems in automotive (ISO 26262), aerospace (DO-178C), and medical (IEC 62304) domains require certified toolchains. Using Clang/LLVM in these contexts requires understanding:

- **Compiler qualification**: The compiler itself must be qualified or certified, demonstrating that it does not introduce faults. This typically requires a compiler validation test suite (e.g., ACVS for Ada, or a C conformance suite for C99/C11).
- **Known issues registry**: Safety-critical use requires documenting and mitigating all known compiler bugs. LLVM's bug tracker must be reviewed for issues affecting the target platform and code patterns in use.
- **Optimization restrictions**: High optimization levels introduce transformations whose correctness proofs are complex. ASIL-D (highest automotive safety level) often restricts compilers to `-O1` with a well-understood subset of passes.
- **Reproducible builds**: Safety certification requires bit-identical builds from identical source. Clang's `SOURCE_DATE_EPOCH` and fixed `-fno-integrated-as` or `-fno-use-init-array` flags are sometimes required.

Several organizations provide certified Clang/LLVM distributions: Green Hills Software, LDRA, Perforce (QNX Momentics), and Arm's own commercial toolchain offer qualified Clang variants for ISO 26262 ASIL-B/D.

### 107.12.2 MISRA C and Clang-Tidy

MISRA C (Motor Industry Software Reliability Association) defines a subset of C for safety-critical embedded systems. Clang-tidy provides MISRA C:2012 checks via the `readability-*`, `bugprone-*`, and `cert-*` check categories:

```bash
# Run MISRA-relevant clang-tidy checks
/usr/lib/llvm-22/bin/clang-tidy \
    --checks='readability-isolate-declaration,\
              bugprone-integer-overflow,\
              cert-int30-c,cert-int31-c,\
              cppcoreguidelines-avoid-magic-numbers' \
    --target=thumbv7em-none-eabihf \
    -extra-arg='-mcpu=cortex-m4' \
    source.c -- \
    --target=thumbv7em-none-eabihf
```

The `-extra-arg` passes additional compiler flags for cross-compilation. Third-party MISRA plugins (e.g., from PRQA/Helix QAC) provide more complete coverage.

### 107.12.3 Stack Depth Analysis for Safety

Safety-critical embedded systems must demonstrate bounded stack usage. The Clang `-fstack-usage` output combined with the call graph can compute worst-case stack depth:

```python
#!/usr/bin/env python3
# Compute WCSE (Worst Case Stack Estimate) from .su files
import re, sys
from collections import defaultdict

su_data = {}  # func_name -> (size, kind)
for line in open("firmware.su"):
    m = re.match(r'.*:(\w+)\s+(\d+)\s+(static|dynamic)', line)
    if m:
        su_data[m.group(1)] = int(m.group(2))

# Parse call graph from LLVM IR (via -print-callgraph)
# Walk call graph from Reset_Handler, sum stack frames
# Report: max path through call graph = WCSE

def wcse(func, call_graph, visited=None):
    if visited is None: visited = set()
    if func in visited: return 0  # cycle (recursion not allowed in MISRA C)
    visited.add(func)
    local = su_data.get(func, 0)
    children = call_graph.get(func, [])
    return local + max((wcse(c, call_graph, visited.copy()) for c in children), default=0)

# Example output: "Worst case stack depth: 1312 bytes (limit: 4096)"
```

This approach is standard in functional safety toolchains. LLVM's `-analyze -print-callgraph` option provides call graph output that can seed such analysis.

### 107.12.4 Clang Address Sanitizer on Bare Metal

ASan (AddressSanitizer) can be adapted for bare-metal use to find memory safety bugs during development (not production):

```bash
# Partial ASan for Cortex-M4 — requires sufficient RAM
clang --target=thumbv7em-none-eabihf \
      -mcpu=cortex-m4 -mfloat-abi=hard \
      -fsanitize=address \
      -fsanitize-address-use-after-scope \
      -g -O1 \
      -c buggy.c -o buggy.o
```

For bare-metal, the ASan shadow memory and interceptors must be customized: the default 1/8-of-address-space shadow map does not fit in embedded SRAM. Embedded ASan implementations typically use a flat shadow region at a fixed address and override `malloc`/`free` with instrumented versions. This is feasible for development boards with hundreds of KB of RAM but not for ultra-constrained targets.

## Chapter 107 Summary

- **Embedded compilation** differs from hosted targets in requiring bare-metal ABI, startup code, linker scripts, software FP builtins, and aggressive code size optimization.
- **ARM Cortex-M** (Thumb-2) is the dominant embedded target; LLVM supports the full M-profile family from M0 (ARMv6-M) through M55/M85 (ARMv8.1-M + Helium MVE). The `thumbvXm-none-eabi[hf]` triple family encodes the ISA variant and float ABI.
- **Linker scripts** define flash/RAM memory regions, section placement, and the symbols (`_data_start`, `_bss_end`) used by the startup routine to initialize memory before `main()`.
- **Code size optimization** combines `-Oz`, the Machine Outliner (`-moutline`), LTO (`-flto=thin`), `--gc-sections` with `-ffunction-sections`, and Identical Code Folding (`--icf=all`).
- **RISC-V RV32I/E** is the open embedded ISA; the C (compressed) extension provides Thumb-2-comparable code density; linker GP relaxation and Zce further reduce size on bare-metal targets.
- **AVR** is LLVM's only 8-bit Harvard-architecture target; register pair allocation and the separate flash/SRAM address spaces are its primary compiler challenges.
- **MSP430** is TI's ultra-low-power 16-bit MCU; its orthogonal ISA and many addressing modes generally produce good code from LLVM's SelectionDAG.
- **Runtime library selection** — compiler-rt builtins + picolibc/newlib — replaces glibc in the embedded context; `--rtlib=compiler-rt` selects the LLVM builtins over libgcc.
- **Debug info** should be compiled with `-g -gdwarf-4` and stripped from the production binary; LLDB communicates with on-chip debug probes via the GDB Remote Serial Protocol.
- **Emerging targets** include Xtensa (ESP32, in-tree since LLVM 20), Cortex-M55 Helium MVE (auto-vectorization target), and RISC-V P extension (DSP packed SIMD).
- Cross-references: [Chapter 97 — The 32-bit ARM Backend](ch97-the-32-bit-arm-backend.md) for the underlying ARM target machinery; [Chapter 92 — The Machine Outliner](../part-14-backend/ch92-machine-outliner.md) for the outliner used in `-Oz`; [Chapter 119 — compiler-rt Builtins](../part-17-runtime-libs/ch119-compiler-rt-builtins.md) for software FP builtins; [Chapter 78 — The LLVM Linker (LLD)](../part-13-lto-whole-program/ch78-the-llvm-linker.md) for LLD embedded linker usage.
