# RiscV
# riscv-understanding

hi, i'm Shubhi Rai, first sem Bsc Hons computer science. this repo is my attempt to understand
risc-v before a workshop on Monday where i might get shortlisted.

# RISC-V Architecture & Assembly Systems Engineering: Comprehensive Reference Manual

> **Document Type:** Technical Systems Reference & Architecture Study Guide  
> **Target ISA Specification:** RISC-V Unprivileged ISA Specification (Base RV32I / RV64I)  
> **Course / Academic Focus:** Computer Systems Architecture, ISA Design, and Assembly Programming  
> **Primary Source Reference:** *RISC-V Architecture Tutorial: Complete Guide from Fundamentals to Advanced Implementation* by Nikhil Kumar Rajput  

---

## Executive Summary & Abstract

Modern computer systems rely on a clean hardware-software boundary defined by the **Instruction Set Architecture (ISA)** [cite: 2]. As proprietary closed architectures like x86 and ARM face trade-offs between legacy bloat, licensing restrictions, and energy consumption, the open-standard **RISC-V** architecture has emerged as an open, modular alternative [cite: 2].

This document provides a comprehensive, ground-up guide to RISC-V architecture, low-level execution pipelines, binary instruction encodings, calling conventions, stack frame lifecycle management, cache-memory hierarchies, and standard extensions [cite: 2]. It bridges theory with bare-metal implementation through runnable assembly code, line-by-line machine code breakdowns, and systems projects designed to demonstrate mastery of modern processor design [cite: 2].

---

## Table of Contents
1. [The Architectural Renaissance: Why RISC-V Matters](#1-the-architectural-renaissance-why-risc-v-matters)
2. [CISC vs. RISC: Microarchitectural Trade-Offs](#2-cisc-vs-risc-microarchitectural-trade-offs)
3. [The Computing Abstraction Hierarchy & The Execution Engine](#3-the-computing-abstraction-hierarchy--the-execution-engine)
4. [Instruction Formats & Binary Machine-Code Encoding](#4-instruction-formats--binary-machine-code-encoding)
5. [The Register File Architecture & The ABI Standard](#5-the-register-file-architecture--the-abi-standard)
6. [Memory Systems, Addressing Modes & Cache Locality](#6-memory-systems-addressing-modes--cache-locality)
7. [Calling Conventions, Activation Frames & Stack Mechanics](#7-calling-conventions-activation-frames--stack-mechanics)
8. [Processor Pipelines, Hazards & Execution Models](#8-processor-pipelines-hazards--execution-models)
9. [Standard Modular Extensions (M, A, F, D, C, V)](#9-standard-modular-extensions-m-a-f-d-c-v)
10. [Featured Systems Project 1: The Caesar Cipher In-Place Stream Engine](#10-featured-systems-project-1-the-caesar-cipher-in-place-stream-engine)
11. [Featured Systems Project 2: High-Precision Multi-Operation Calculator](#11-featured-systems-project-2-high-precision-multi-operation-calculator)
12. [Toolchains, Emulation & Debugging Workflows](#12-toolchains-emulation--debugging-workflows)

---

## 1. The Architectural Renaissance: Why RISC-V Matters

For over four decades, the semiconductor landscape has been dominated by proprietary, closed Instruction Set Architectures [cite: 2]:
- **x86 / x86-64 (Intel, AMD):** A Complex Instruction Set Computer (CISC) architecture laden with four decades of backward-compatibility requirements [cite: 2]. Decoding variable-length x86 instructions requires substantial silicon real estate and power.
- **ARM (Arm Ltd.):** A Reduced Instruction Set Computer (RISC) architecture prevalent in mobile and embedded computing [cite: 2]. While energy-efficient, ARM is proprietary: companies must pay substantial upfront license fees, recurring royalties per fabricated chip, and are generally forbidden from altering the underlying ISA.

### 1.1 The Inception of RISC-V
Conceived in 2010 by Krste Asanović, David Patterson, and their research team at the University of California, Berkeley, **RISC-V** (pronounced *"risk-five"*) was designed to solve these systemic constraints [cite: 2]:
- **Free, Open, and Royalty-Free:** The ISA specification is managed by RISC-V International [cite: 2]. Anyone can design, manufacture, and sell chips without licensing fees or nondisclosure agreements [cite: 2].
- **Clean-Slate Simplicity:** Engineered without legacy design artifacts, yielding smaller die sizes, lower power usage, and simplified microarchitectural design [cite: 2].
- **Base Plus Modular Extensions:** Rather than forcing every implementation to support thousands of instructions, RISC-V defines a minimal, immutable base integer ISA (e.g., `RV32I` with fewer than 50 base instructions) that can be augmented with optional standardized modular extensions [cite: 2].

---

## 2. CISC vs. RISC: Microarchitectural Trade-Offs

The fundamental difference between CISC and RISC lies in the division of responsibilities between hardware decoders and software compilers [cite: 2].

| Architectural Attribute | CISC (Complex Instruction Set Computer) | RISC (Reduced Instruction Set Computer) |
| :--- | :--- | :--- |
| **Design Philosophy** | Rich instruction set; hardware executes multi-step tasks [cite: 2]. | Minimal, primitive instructions; compiler arranges complex logic [cite: 2]. |
| **Instruction Length** | Variable (1 to 15 bytes in x86) [cite: 2]. | Fixed uniform width (32-bit standard; 16-bit compressed) [cite: 2]. |
| **Memory Access** | Orthogonal: Memory operands can appear inside ALU operations [cite: 2]. | **Strict Load-Store:** Only `LW`/`SW` touch RAM; ALU operates on registers [cite: 2]. |
| **Instruction Decoding** | Complex, multi-cycle decoders; microcode lookup engines [cite: 2]. | Direct hardwired decoders; fast, deterministic single-cycle decode [cite: 2]. |
| **Pipeline Predictability** | Variable execution cycles per instruction cause pipeline stalls [cite: 2]. | Highly uniform instruction execution enables balanced pipelining [cite: 2]. |
| **Transistor Allocation** | Significant silicon dedicated to decoding logic and micro-op caches. | Majority of silicon dedicated to general-purpose registers and ALUs. |

### The Load-Store Operational Model
In a CISC architecture like x86, an operation adding memory contents to a register can be written as:
```nasm
add eax, [ebx + 4]   ; Single instruction: Reads memory, adds to EAX, updates flags
```
In RISC-V, this operation must be explicitly decoupled into distinct load, compute, and store phases [cite: 2]:
```asm
lw   t0, 4(s0)       # 1. LOAD: Read memory word into scratch register t0
add  a0, a0, t0      # 2. COMPUTE: Perform register-register addition in the ALU
# Result resides cleanly in register a0; memory is untouched
```
While RISC-V requires more instructions, each instruction executes cleanly across streamlined pipeline stages without holding up the processor [cite: 2].

---

## 3. The Computing Abstraction Hierarchy & The Execution Engine

Computing systems operate across a stack of layered abstractions [cite: 2]:
```
[Level 7] High-Level Software (Python, C, Rust, Go)
              ↓ (Compiled / Interpreted)
[Level 6] Assembly Language (Human-readable symbolic mnemonics)
              ↓ (Assembler: Translates text to machine instructions)
[Level 5] Instruction Set Architecture (ISA: RV32I machine-code bits)
              ↓ (Microarchitecture: Pipeline datapaths, hazard units)
[Level 4] Digital Logic Circuits (Multiplexers, ALUs, Flip-Flops, Adders)
              ↓
[Level 3] Transistors & CMOS Gates (Logic switching at silicon voltage levels)
              ↓
[Level 2] Semiconductor Physics & Materials Science
```
The **ISA serves as the universal contract** between software and hardware: the compiler targets the ISA without caring how the silicon is constructed, and the hardware engineer designs the circuit to execute the ISA without caring what software will run [cite: 2].

### 3.1 The Von Neumann Instruction Execution Cycle
Every RISC-V processor continuously cycles through five primary execution phases [cite: 2]:

```
+---------------+     +---------------+     +---------------+     +---------------+     +---------------+
|     FETCH     | --> |    DECODE     | --> |    EXECUTE    | --> |    MEMORY     | --> |   WRITEBACK   |
| (Read PC/RAM) |     | (Parse Fields)|     | (ALU Compute) |     | (RAM Read/Wr) |     | (Commit to Rd)|
+---------------+     +---------------+     +---------------+     +---------------+     +---------------+
```

1. **Instruction Fetch (IF):** The Program Counter (`PC`) delivers its memory address to the instruction cache [cite: 2]. The 32-bit instruction word is fetched, and the `PC` is incremented by 4 (`PC + 4`) [cite: 2].
2. **Instruction Decode (ID):** The control unit parses the fixed-position opcode, source register indexes (`rs1`, `rs2`), and destination index (`rd`) [cite: 2]. Operands are fetched from the register file.
3. **Execute (EX):** The Arithmetic Logic Unit (ALU) operates on register values or immediate offsets to compute results or branch target addresses [cite: 2].
4. **Memory Access (MEM):** For memory instructions (`lw`, `lh`, `lb`, `sw`, `sh`, `sb`), data RAM is read or written [cite: 2]. Arithmetic operations bypass this stage without taking action.
5. **Writeback (WB):** The final computed result (from the ALU or memory load) is committed into the destination register (`rd`) [cite: 2].

---

## 4. Instruction Formats & Binary Machine-Code Encoding

RISC-V simplifies hardware decoders by adhering to **fixed 32-bit width instructions** aligned to 4-byte boundaries [cite: 2]. Furthermore, source register locations (`rs1`, `rs2`) and destination register locations (`rd`) are locked in the same bit positions across formats [cite: 2].

```
Bit:      31        25 24        20 19        15 14   12 11        7 6          0
        +-------------+------------+------------+-------+------------+------------+
R-Type: |  funct7 (7) |   rs2 (5)  |   rs1 (5)  |f3 (3) |   rd (5)   | opcode (7) |
        +-------------+------------+------------+-------+------------+------------+
I-Type: |         imm[11:0] (12)   |   rs1 (5)  |f3 (3) |   rd (5)   | opcode (7) |
        +-------------+------------+------------+-------+------------+------------+
S-Type: | imm[11:5](7)|   rs2 (5)  |   rs1 (5)  |f3 (3) |imm[4:0] (5)| opcode (7) |
        +-------------+------------+------------+-------+------------+------------+
B-Type: |imm[12|10:5] |   rs2 (5)  |   rs1 (5)  |f3 (3) |imm[4:1|11] | opcode (7) |
        +-------------+------------+------------+-------+------------+------------+
U-Type: |                    imm[31:12] (20-bit Upper)  |   rd (5)   | opcode (7) |
        +-----------------------------------------------+------------+------------+
J-Type: |             imm[20 | 10:1 | 11 | 19:12] (20)  |   rd (5)   | opcode (7) |
        +-----------------------------------------------+------------+------------+
```

### 4.1 Detailed Breakdown of Format Purposes
- **R-Type (Register-Register):** Arithmetic and logical operations taking two register operands (`add`, `sub`, `sll`, `slt`, `xor`, `srl`, `sra`, `or`, `and`) [cite: 2].
- **I-Type (Immediate & Loads):** Register-immediate arithmetic (`addi`, `andi`, `ori`), memory load instructions (`lw`, `lh`, `lb`, `lhu`, `lbu`), and indirect jumps (`jalr`) [cite: 2].
- **S-Type (Store):** Memory store operations (`sw`, `sh`, `sb`) [cite: 2]. The 12-bit immediate offset is split across bits `[31:25]` and `[11:7]` so that `rs1` and `rs2` remain in their standardized bit slots [cite: 2].
- **B-Type (Branch):** Conditional branches (`beq`, `bne`, `blt`, `bge`, `bltu`, `bgeu`) [cite: 2]. Encodes a signed immediate representing a 2-byte aligned PC-relative jump offset [cite: 2].
- **U-Type (Upper Immediate):** 20-bit immediate values loaded into the upper portion of a register (`lui`, `auipc`), used to build 32-bit constants and absolute memory addresses.
- **J-Type (Unconditional Jump):** Jump and link operations (`jal`) encoding a signed 20-bit PC-relative offset.

### 4.2 Machine-Code Encoding Case Study: `add x3, x1, x2`
Let us hand-assemble `add x3, x1, x2` into its 32-bit binary representation [cite: 2]:
1. **Opcode:** For R-type integer arithmetic, the opcode is `0110011` [cite: 2].
2. **rd (Destination):** `x3` is integer 3 $
ightarrow$ `00011` [cite: 2].
3. **funct3:** For ADD/SUB, `funct3` is `000` [cite: 2].
4. **rs1 (Source 1):** `x1` is integer 1 $
ightarrow$ `00001` [cite: 2].
5. **rs2 (Source 2):** `x2` is integer 2 $
ightarrow$ `00010` [cite: 2].
6. **funct7:** To distinguish ADD from SUB, `funct7` for ADD is `0000000` (`0100000` designates SUB) [cite: 2].

Now combine all bit patterns:
```
funct7    rs2    rs1    funct3  rd     opcode
0000000 | 00010 | 00001 | 000  | 00011 | 0110011
```
Group into 4-bit nibbles for hexadecimal translation [cite: 2]:
```
0000 0000 0010 0000 1000 0001 1011 0011
   0    0    2    0    8    1    B    3  ==> Machine Code: 0x002081B3
```

---

## 5. The Register File Architecture & The ABI Standard

The base RV32I architecture defines **32 general-purpose registers** (`x0` through `x31`), each 32 bits wide, along with the independent Program Counter (`pc`) [cite: 2]. To ensure modularity between different compilers, operating systems, and assembly libraries, the **Application Binary Interface (ABI)** standardizes register assignments [cite: 2].

```
+----------+----------+-------------------------------------+--------------------------+
| Register | ABI Name | Primary Functional Category         | Preserved Across Calls?  |
+----------+----------+-------------------------------------+--------------------------+
| x0       | zero     | Constant 0 (Hardware Hardwired)     | Unalterable (Permanent)  |
| x1       | ra       | Return Address                      | No  (Caller-Saved)       |
| x2       | sp       | Stack Pointer (16-byte aligned)     | Yes (Callee-Saved)       |
| x3       | gp       | Global Data Pointer                 | Unspecified              |
| x4       | tp       | Thread Pointer                      | Unspecified              |
| x5-x7    | t0-t2    | Temporaries                         | No  (Caller-Saved)       |
| x8       | s0 / fp  | Saved Register 0 / Frame Pointer    | Yes (Callee-Saved)       |
| x9       | s1       | Saved Register 1                    | Yes (Callee-Saved)       |
| x10-x11  | a0-a1    | Function Arguments / Return Values  | No  (Caller-Saved)       |
| x12-x17  | a2-a7    | Function Arguments (3 through 8)    | No  (Caller-Saved)       |
| x18-x27  | s2-s11   | Saved Registers (2 through 11)      | Yes (Callee-Saved)       |
| x28-x31  | t3-t6    | Temporaries (3 through 6)           | No  (Caller-Saved)       |
+----------+----------+-------------------------------------+--------------------------+
```
*(Reference: Nikhil Kumar Rajput, Table 5: Complete RISC-V Register Usage Convention [cite: 2])*

### 5.1 The `x0` (zero) Architectural Advantage
The `x0` register is hardwired to electrical ground ($0	ext{V}$) [cite: 2]. Reads always yield $0$, and writes are silently ignored by the register file write-enable logic [cite: 2]. This eliminates the need for specialized instructions [cite: 2]:
- **Register Copy (`mv rd, rs`):** Translated as `addi rd, rs, 0` [cite: 2].
- **No-Op (`nop`):** Translated as `addi x0, x0, 0` [cite: 2].
- **Sign Inversion (`neg rd, rs`):** Translated as `sub rd, x0, rs` ($0 - rs$) [cite: 2].
- **Comparison to Zero (`beqz rs, label`):** Translated as `beq rs, x0, label` [cite: 2].

### 5.2 Caller-Saved vs. Callee-Saved Mechanics
Understanding register ownership is essential for reliable assembly programming [cite: 2]:
- **Caller-Saved Registers (`t0-t6`, `a0-a7`, `ra`):** Considered temporary [cite: 2]. If the calling function holds data in a `t` register and calls another function, that sub-function can overwrite it without warning [cite: 2]. The caller must save it to the stack beforehand if the value is needed later [cite: 2].
- **Callee-Saved Registers (`s0-s11`, `sp`):** Considered non-volatile [cite: 2]. If a called function needs to use `s1`, it must save the caller's original `s1` value to the stack, use the register, and restore the original value before returning via `ret` [cite: 2].

---

## 6. Memory Systems, Addressing Modes & Cache Locality

### 6.1 Base + Offset Addressing
RISC-V relies exclusively on **Base + Offset** addressing [cite: 2]:
$$	ext{Memory Address} = 	ext{Register Value} + 	ext{Sign-Extended 12-bit Immediate Offset}$$
Complex addressing modes found in CISC processors (such as scaled index addressing `[base + index * scale + disp]`) are intentionally omitted from RISC-V hardware [cite: 2]. Instead, software compilers generate explicit arithmetic instructions to compute scaled array pointers [cite: 2]:

```asm
# Array indexing in RISC-V: Load array[i] where base is in a0, index i in a1
slli t0, a1, 2       # t0 = i * 4 (Shift Left Logical by 2 multiplies by word size)
add  t1, a0, t0      # t1 = base_address + (i * 4)
lw   a2, 0(t1)       # Load 32-bit word directly from computed address
```

### 6.2 Data Sign Extension Semantics
When loading data types smaller than the 32-bit register width (bytes or halfwords), the processor must define how upper bits `[31:8]` or `[31:16]` are populated [cite: 2]:
- `lb` (Load Byte Signed): Reads 8 bits and sign-extends the highest bit (bit 7) across bits `[31:8]`, preserving two's complement negative values [cite: 2].
- `lbu` (Load Byte Unsigned): Reads 8 bits and zero-extends bits `[31:8]`, preserving positive magnitude (range 0 to 255) [cite: 2].
- `lh` / `lhu` (Load Halfword Signed / Unsigned): Operates identically on 16-bit quantities [cite: 2].

### 6.3 Memory Hierarchy & Cache Locality
```
Registers (32 x 32-bit words)     < 0.5 ns
  └── L1 Data/Instruction Cache (32 KB - 64 KB)     ~ 1-2 ns
        └── L2 Shared Cache (256 KB - 1 MB)          ~ 5-10 ns
              └── L3 System Cache (8 MB - 32 MB)     ~ 20-40 ns
                    └── Main DRAM Memory (8 GB - 64 GB)    ~ 60-100 ns
```
Accessing data sequentially maximizes spatial locality, allowing the hardware prefetcher to load cache lines and avoid high-latency stalls [cite: 2].

---

## 7. Calling Conventions, Activation Frames & Stack Mechanics

The runtime stack is a contiguous block of memory utilized for temporary storage, dynamic local variables, and preserving return addresses across nested function executions [cite: 2].

### 7.1 Stack Rules in RISC-V
1. **Grows Downward:** Memory allocations decrement the Stack Pointer (`addi sp, sp, -N`); deallocations increment it (`addi sp, sp, N`) [cite: 2].
2. **16-Byte Boundary Alignment:** The standard RISC-V ABI requires `sp` to remain aligned to multiples of 16 bytes at any function boundary to maintain compatibility with vector and floating-point data structures [cite: 2].
3. **No Hardware Push/Pop Instructions:** Unlike x86, which provides microcoded `push` and `pop` instructions, RISC-V performs stack modifications via explicit `addi`, `sw`, and `lw` operations [cite: 2].

```
High Memory Address
       |                                                 |
       +-------------------------------------------------+
       | Caller's Call Frame                             |
       +-------------------------------------------------+ <--- Previous SP
       | Return Address (ra)                     [sp+12] |
       | Saved Register (s0 / fp)                [sp+8]  |
       | Local Variable / Scratch Storage        [sp+4]  |
       | Local Variable / Scratch Storage        [sp+0]  |
       +-------------------------------------------------+ <--- Current SP (sp - 16)
       | (Stack grows downwards toward lower addresses)  |
       v                                                 v
Low Memory Address
```

### 7.2 Non-Leaf Function Calling Lifecycle
A non-leaf function is any function that invokes another child function [cite: 2]. Because executing `jal ra, <child>` immediately overwrites `ra` with the return address inside the current function, failure to preserve `ra` on the stack results in an infinite loop or segmentation fault [cite: 2].

```asm
# Canonical Non-Leaf Function Implementation
nested_function_example:
    # 1. PROLOGUE: Allocate frame and preserve volatile context
    addi sp, sp, -16          # Reserve 16-byte aligned stack frame
    sw   ra, 12(sp)           # Save return address
    sw   s0, 8(sp)            # Save callee-saved s0

    # 2. FUNCTION BODY
    mv   s0, a0               # Store input argument safely in s0
    li   a0, 42               # Prepare argument for child function
    jal  ra, external_service # Execute nested child call; ra is overwritten
    add  a0, a0, s0           # a0 = child_return_val + original input

    # 3. EPILOGUE: Restore context and tear down frame
    lw   s0, 8(sp)            # Restore s0
    lw   ra, 12(sp)           # Restore original ra
    addi sp, sp, 16           # Free stack frame
    ret                       # Jump to ra (jalr x0, ra, 0)
```

---

## 8. Processor Pipelines, Hazards & Execution Models

### 8.1 The Standard 5-Stage Classic Pipeline
A scalar RISC-V processor executes instructions across five overlapping stages [cite: 2]:

```
Cycle:      1      2      3      4      5      6      7
Instr 1:   [IF]   [ID]   [EX]   [MEM]  [WB]
Instr 2:          [IF]   [ID]   [EX]   [MEM]  [WB]
Instr 3:                 [IF]   [ID]   [EX]   [MEM]  [WB]
Instr 4:                        [IF]   [ID]   [EX]   [MEM]  [WB]
Instr 5:                               [IF]   [ID]   [EX]   [MEM]  [WB]
```

### 8.2 Pipeline Hazards and Mitigation Strategies
- **Structural Hazards:** Hardware resource conflicts (e.g., instructions attempting to access single-ported RAM simultaneously). Solved by separating instruction memory and data memory (Harvard architecture) [cite: 2].
- **Data Hazards:** Occur when an instruction depends on the result of an earlier instruction that has not yet reached writeback:
  - *Mitigation:* **Data Forwarding (Bypassing):** Forwarding multiplexers route the output of the ALU directly back into the inputs of the EX stage for the next instruction, bypassing register writeback.
  - *Load-Use Delay:* If an instruction immediately follows a `lw` and depends on that data, forwarding cannot travel backward in time; the hardware pipeline must insert an automatic stall bubble.
- **Control Hazards:** Branch decisions (`beq`, `bne`) alter the PC during stage 3 (EX), causing instructions fetched during stages 1 and 2 to become invalid [cite: 2].
  - *Mitigation:* **Branch Prediction:** Hardware branch predictors guess the branch direction (taken vs. not taken); on a misprediction, the speculative instructions are flushed [cite: 2].

### 8.3 In-Order vs. Out-of-Order Execution
- **In-Order Processors:** Instructions execute strictly according to compiler program order [cite: 2]. If an instruction stalls on a DRAM read, the entire processor core halts [cite: 2]. Common in low-power microcontrollers and edge systems [cite: 2].
- **Out-of-Order Processors (OoO):** The processor dynamically examines instruction windows [cite: 2]. Instructions with available operands execute immediately, even if earlier instructions are stalled on cache misses [cite: 2]. Dependencies are resolved using **Register Renaming**, **Reservation Stations**, and a **Reorder Buffer (ROB)** [cite: 2].

---

## 9. Standard Modular Extensions (M, A, F, D, C, V)

Rather than changing the entire architecture between chip generations, RISC-V uses standardized modular extensions [cite: 2]:

| Extension | Extension Name | Hardware Capability & Instruction Set Addition |
| :--- | :--- | :--- |
| **I** | Base Integer | Minimal, standard base set (RV32I / RV64I) required for all implementations [cite: 2]. |
| **M** | Multiply & Divide | Hardware multiplier/divider: `mul`, `mulh`, `mulhu`, `div`, `rem` [cite: 2]. |
| **A** | Atomic Operations | Inter-core synchronization: `lr.w` (Load-Reserved), `sc.w` (Store-Conditional), `amoadd.w` [cite: 2]. |
| **F** | Single-Precision FP | 32-bit IEEE 754 floating-point calculations with dedicated registers `f0-f31` [cite: 2]. |
| **D** | Double-Precision FP | 64-bit IEEE 754 floating-point calculations [cite: 2]. |
| **C** | Compressed Instructions | 16-bit compressed encodings for common operations, reducing code size by 25–30% [cite: 2]. |
| **V** | Vector Processing | Variable-length SIMD operations for deep learning, DSP, and matrix computing [cite: 2]. |
| **G** | General-Purpose Base | Shorthand denoting: `IMAFD_Zicsr_Zifencei`. |

---

---
## 10. Featured Systems Project 2: High-Precision Multi-Operation Calculator

This project implements a multi-operation calculator that evaluates expressions, uses software-based division-by-zero checks, and handles arithmetic dispatching [cite: 2].

### Architectural Significance
- **Modular Function Dispatching:** Demonstrates branching on character-encoded operational selectors (`+`, `-`, `*`, `/`) [cite: 2].
- **Error Handling:** Protects hardware pipelines by testing divisor operands against `x0` prior to arithmetic execution, preventing hardware division faults [cite: 2].

### Complete Annotated Assembly Code (`riscv_calculator.s`)
```asm
# ==============================================================================
# RISC-V RV32I Reference Implementation: Arithmetic Calculator
# Demonstrates: ALU arithmetic, branching tables, error handling, syscalls
# ==============================================================================

# ==============================================================================
# RISC-V RV32IM Reference Implementation: Arithmetic Calculator
# Target: Venus RISC-V Simulator
# ==============================================================================

.data
val_a:          .word   120
val_b:          .word   15
op_choice:      .byte   42      # ASCII: '+'=43, '-'=45, '*'=42, '/'=47

str_header:     .asciiz "--- RISC-V Compute Engine Result ---\n"
str_ans:        .asciiz "Computed Result: "
str_rem:        .asciiz "\nRemainder:       "
str_div_err:    .asciiz "\nError: Hardware Division By Zero Aborted!\n"
newline:        .asciiz "\n"

.text
.globl main

main:
    addi sp, sp, -24
    sw   ra, 20(sp)
    sw   s0, 16(sp)
    sw   s1, 12(sp)
    sw   s2, 8(sp)

    # Print Header (ecall 4: print string)
    li   a0, 4
    la   a1, str_header
    ecall

    # Load Operands into Argument Registers
    lw   a0, val_a              # a0 = operand 1
    lw   a1, val_b              # a1 = operand 2
    lb   a2, op_choice          # a2 = operator ASCII code

    jal  ra, execute_calculation
    mv   s0, a0                 # s0 = primary result
    mv   s1, a1                 # s1 = remainder (if division)
    mv   s2, a2                 # s2 = error flag (1 if error, 0 if clean)

    # Verify if calculation returned an error
    bnez s2, handle_calc_error

    # Print clean calculation output
    li   a0, 4
    la   a1, str_ans
    ecall

    li   a0, 1                  # Syscall 1: Print Integer
    mv   a1, s0
    ecall

    # Check if division operation returned a remainder
    lb   t0, op_choice
    li   t1, 47                 # ASCII '/'
    bne  t0, t1, finish_execution

    li   a0, 4
    la   a1, str_rem
    ecall

    li   a0, 1
    mv   a1, s1
    ecall
    j    finish_execution

handle_calc_error:
    li   a0, 4
    la   a1, str_div_err
    ecall

finish_execution:
    li   a0, 4
    la   a1, newline
    ecall

    # Restore stack frame
    lw   s2, 8(sp)
    lw   s1, 12(sp)
    lw   s0, 16(sp)
    lw   ra, 20(sp)
    addi sp, sp, 24

    # Exit program (ecall 10: exit)
    li   a0, 10
    ecall

# ------------------------------------------------------------------------------
# Function: execute_calculation(int a, int b, char op) -> (res, rem, err)
# Outputs: a0 = result, a1 = remainder, a2 = error status (0=OK, 1=FAIL)
# ------------------------------------------------------------------------------
execute_calculation:
    li   t0, 43                 # '+'
    beq  a2, t0, do_add
    li   t0, 45                 # '-'
    beq  a2, t0, do_sub
    li   t0, 42                 # '*'
    beq  a2, t0, do_mul
    li   t0, 47                 # '/'
    beq  a2, t0, do_div

    # Unknown operator: Return error
    li   a0, 0
    li   a1, 0
    li   a2, 1
    ret

do_add:
    add  a0, a0, a1
    li   a1, 0
    li   a2, 0
    ret

do_sub:
    sub  a0, a0, a1
    li   a1, 0
    li   a2, 0
    ret

do_mul:
    mul  a0, a0, a1             # RV32M instruction
    li   a1, 0
    li   a2, 0
    ret

do_div:
    beqz a1, div_by_zero_error  # Test for division by zero
    div  t0, a0, a1             # RV32M instruction
    rem  t1, a0, a1             # RV32M instruction
    mv   a0, t0
    mv   a1, t1
    li   a2, 0
    ret

div_by_zero_error:
    li   a0, 0
    li   a1, 0
    li   a2, 1                  # Assert error flag
    ret
```
-------------------------------------------------------------------------------------------------------------
Output Screenshot
<img width="1920" height="1080" alt="Screenshot from 2026-09-29 11-47-28" src="https://github.com/user-attachments/assets/061d0291-92a7-4628-9c24-886412cc25e4" />
```

## 12. Toolchains, Emulation & Debugging Workflows

### 12.1 Interactive Browser Simulation via Venus
The Venus simulator runs in the browser, making it ideal for immediate testing without local toolchain installation [cite: 2]:
1. Navigate to the Venus simulator interface (`https://venus.cs61c.org/`) [cite: 2].
2. Paste assembly source code into the **Editor** pane [cite: 2].
3. Click **Assemble & Simulate** [cite: 2].
4. Step through execution using the **Step** button while monitoring register updates and memory panels [cite: 2].

### 12.2 Local Linux Toolchain Compilation & QEMU Execution
For local Linux systems, the standard GNU cross-compiler and QEMU emulator can be used [cite: 2]:

```bash
# 1. Install toolchain and QEMU on Ubuntu/Debian
sudo apt update
sudo apt install gcc-riscv64-unknown-elf binutils-riscv64-unknown-elf qemu-user

# 2. Assemble and link assembly source into an ELF binary
riscv64-unknown-elf-as -march=rv32im -mabi=ilp32 -o program.o program.s
riscv64-unknown-elf-ld -m elf32lriscv -o program.elf program.o

# 3. Execute binary using QEMU
qemu-riscv32 ./program.elf
```
*(Reference: Nikhil Kumar Rajput, Section 11: Development Tools and Practice Resources [cite: 2])*

