RISC-V: A Simple Complete Guide

By Shubhi Rai, BSc (Hons) Computer Science, Semester 1
Made to prepare for a RISC-V workshop. This guide explains RISC-V from zero, with working assembly code.

---

What is this document?

This is a complete beginner-friendly guide to the RISC-V processor architecture. It covers:

- Why RISC-V exists and why it matters
- RISC vs CISC (the two CPU design styles)
- How a CPU actually runs instructions, step by step
- How assembly instructions are encoded as binary numbers
- The 32 registers and what each one is for
- Memory, addressing, and caches
- How functions call each other (the stack)
- CPU pipelines and their problems
- The optional RISC-V extensions (M, A, F, D, C, V)
- One working project: a calculator
- How to run RISC-V code (Venus simulator + Linux toolchain)

---

1. Why RISC-V Matters

Every computer sits between two worlds: software (programs) and hardware (the chip). The rulebook that connects them is called the Instruction Set Architecture (ISA). If a programmer writes code for an ISA, any chip built for that ISA can run it.

For 40+ years, two companies' rulebooks dominated:

- x86 (Intel/AMD): A "Complex" ISA (CISC). It carries 40 years of old baggage. Instructions are 1 to 15 bytes long, and just figuring out what an instruction means takes a lot of chip area and power.
- ARM (Arm Ltd.): A "Reduced" ISA (RISC). Very efficient — that's why phones use it — but it is proprietary. You must pay big license fees, pay royalties on every chip sold, and you are not allowed to change the ISA.

Enter RISC-V

In 2010, professors Krste Asanović and David Patterson at UC Berkeley designed a new ISA from scratch to fix these problems:

1. Free and open. Managed by RISC-V International. Anyone can build and sell RISC-V chips with zero license fees and zero royalties.
2. No legacy baggage. Designed clean, so chips are smaller, cheaper, and use less power.
3. Modular. There is a tiny base set (RV32I, under 50 instructions) that never changes. Everything else is an optional add-on "extension" you pick only if you need it.

---

2. RISC vs CISC

	CISC (e.g. x86)	RISC (e.g. RISC-V)	
Idea	Give the chip rich, powerful instructions that do multi-step jobs	Keep instructions tiny and simple; let the compiler combine them	
Instruction size	Variable: 1 to 15 bytes	Fixed: always 4 bytes (or 2 with the C extension)	
Memory access	ALU instructions can read/write memory directly	Load-store model: only `lw`/`sw` touch memory; ALU works only on registers	
Decoding	Needs complex, multi-cycle decoders	Simple hardwired decoding — one cycle	
Pipelining	Hard, because every instruction takes a different time	Easy, because every instruction looks the same	
Silicon spent on	Decoder logic, microcode	Actual useful work: registers and ALUs	

The load-store model in action

x86 (one instruction does everything):

```
add eax, [ebx + 4]    ; read memory, add it to eax, all in one go
```

RISC-V (split into clear steps):

```
lw   t0, 4(s0)    # step 1: LOAD from memory into register t0
add  a0, a0, t0   # step 2: add two registers
```

RISC-V needs more instructions, but each one is simple and fast, so the chip stays small and quick.

---

3. The Abstraction Layers of a Computer

A computer is a stack of layers, each hiding the one below it:

```
[7] High-level languages (Python, C, Rust)
[6] Assembly language (human-readable mnemonics)
[5] ISA (the machine-code bits — RISC-V lives here)
[4] Digital logic (adders, muxes, flip-flops)
[3] Transistors and logic gates
[2] Semiconductor physics
```

The ISA is the contract: the compiler writer only needs to know the ISA, and the chip designer only needs to implement the ISA. Neither needs to know about the other.

How a CPU runs one instruction (the fetch-decode cycle)

Every RISC-V CPU loops forever through 5 stages:

1. Fetch (IF): The Program Counter (PC) says which memory address holds the next instruction. The 32-bit instruction is read, and PC is incremented by 4.
2. Decode (ID): The control unit reads the fixed fields of the instruction: opcode, source registers rs1/rs2, destination register rd. The register values are fetched.
3. Execute (EX): The ALU does the math (add, subtract, compare, compute a branch address...).
4. Memory (MEM): Only for load/store instructions (`lw`, `sw`, ...) — data is read from or written to RAM. Other instructions skip this stage.
5. Writeback (WB): The result (from ALU or from memory) is written into the destination register rd.

Then the cycle repeats with the next instruction.

---

4. Instruction Formats and Binary Encoding

All base RISC-V instructions are exactly 32 bits (4 bytes), and they start at 4-byte-aligned addresses. Even better, the rs1, rs2, and rd fields are in the same bit positions in every format — this makes the decoder hardware much simpler.

```
Bit:     31      25 24     20 19     15 14  12 11      7 6       0
R-Type: | funct7  |  rs2  |  rs1  | f3 |   rd   | opcode |
I-Type: |     imm[11:0]   |  rs1  | f3 |   rd   | opcode |
S-Type: | imm[11:5]|  rs2  |  rs1  | f3 |imm[4:0]| opcode |
B-Type: |imm[12|10:5]| rs2  |  rs1  | f3 |imm[4:1|11]| opcode |
U-Type: |        imm[31:12]        |   rd   | opcode |
J-Type: |   imm[20|10:1|11|19:12]  |   rd   | opcode |
```

What each format is for

- R-Type (Register-Register): math on two registers: `add, sub, sll, slt, xor, srl, sra, or, and`
- I-Type (Immediate/Load): math with a constant: `addi, andi, ori`; loads: `lw, lh, lb, lhu, lbu`; and `jalr`
- S-Type (Store): `sw, sh, sb` — writes a register to memory. The offset is split into two pieces so rs1 and rs2 stay in their standard positions.
- B-Type (Branch): conditional jumps: `beq, bne, blt, bge, bltu, bgeu`. The offset is in units of 2 bytes, since instructions are 2-byte aligned.
- U-Type (Upper immediate): `lui, auipc` — load a 20-bit constant into the top half of a register, used to build big numbers/addresses.
- J-Type (Jump): `jal` — unconditional jump with link (for function calls).

Example: hand-encoding `add x3, x1, x2`

Build it field by field:

Field	Value	Why	
opcode	`0110011`	all R-type integer math	
rd	`00011`	x3 = 3	
funct3	`000`	selects ADD/SUB	
rs1	`00001`	x1 = 1	
rs2	`00010`	x2 = 2	
funct7	`0000000`	0000000 = ADD, 0100000 = SUB	

Concatenate: `0000000 00010 00001 000 00011 0110011`

Group in nibbles: `0000 0000 0010 0000 1000 0001 1011 0011`

Machine code: `0x002081B3` — this is the exact 4 bytes the CPU fetches.

---

5. The 32 Registers and the ABI

RV32I has 32 general-purpose registers (x0–x31), each 32 bits, plus the PC. The ABI (Application Binary Interface) gives them standard names and jobs, so code from different compilers and libraries works together:

Register	ABI name	Job	Survives a function call?	
x0	zero	always reads as 0; writes are ignored	permanent	
x1	ra	return address	No (caller saves)	
x2	sp	stack pointer (16-byte aligned)	Yes (callee saves)	
x3	gp	global data pointer	unspecified	
x4	tp	thread pointer	unspecified	
x5–x7	t0–t2	temporaries	No	
x8	s0 / fp	saved reg / frame pointer	Yes	
x9	s1	saved register	Yes	
x10–x11	a0–a1	function arguments + return values	No	
x12–x17	a2–a7	more arguments	No	
x18–x27	s2–s11	more saved registers	Yes	
x28–x31	t3–t6	more temporaries	No	

Why x0 (zero) is brilliant

Because x0 is hardwired to 0, we get useful "pseudo-instructions" for free:

- `mv rd, rs`  →  `addi rd, rs, 0` (copy)
- `nop`  →  `addi x0, x0, 0` (do nothing)
- `neg rd, rs`  →  `sub rd, x0, rs` (0 − rs)
- `beqz rs, label`  →  `beq rs, x0, label` (branch if zero)

Caller-saved vs callee-saved

- Caller-saved (t0–t6, a0–a7, ra): scratch space. If you call a function, it may destroy them. If you still need the value, save it yourself before the call.
- Callee-saved (s0–s11, sp): safe across calls. If a function wants to use s1, it must push the old value, use it, and pop it back before returning — so the caller never notices.

---

6. Memory, Addressing, and Caches

6.1 Base + Offset addressing

RISC-V has exactly one addressing mode:

```
address = value in a register + 12-bit signed offset
```

CISC machines have fancy modes like `[base + index*4 + 100]`. RISC-V drops them — you compute addresses with normal math instructions:

```
# load array[i], base in a0, index in a1
slli t0, a1, 2     # t0 = i * 4  (shift left by 2 = multiply by 4)
add  t1, a0, t0    # t1 = base + i*4
lw   a2, 0(t1)     # load the word
```

6.2 Signed vs unsigned loads

Loading something smaller than 32 bits must fill the upper register bits somehow:

- `lb` (load byte signed): copies bit 7 across bits 8–31 → negative numbers stay negative (two's complement).
- `lbu` (load byte unsigned): fills bits 8–31 with 0 → value is 0–255.
- `lh` / `lhu`: same idea for 16-bit halfwords.

6.3 The memory hierarchy

Fast memory is tiny, big memory is slow. So CPUs layer it:

```
Registers (~1 cycle)
  └─ L1 cache (~32–64 KB)
       └─ L2 cache (~256 KB–1 MB)
            └─ L3 cache (~8–32 MB)
                 └─ Main RAM (GBs, ~100x slower than registers)
```

When you access memory sequentially, the hardware prefetches the next chunk (cache line) automatically — this is called spatial locality, and it's the single biggest free performance trick.

---

7. The Stack and Function Calls

The stack is a region of RAM used for temporary storage during function calls.

Stack rules in RISC-V

1. It grows downward. Allocating = `addi sp, sp, -N` (sp goes to a lower address); freeing = `addi sp, sp, +N`.
2. sp must always be 16-byte aligned at function boundaries (required by the ABI, and needed for vector/float data).
3. No push/pop instructions. x86 has them; RISC-V does it manually with `addi` + `sw` + `lw`.

```
high address
| caller's frame                    |
+-----------------------------------+  <- old sp
| saved ra                  [sp+12] |
| saved s0                  [sp+8]  |
| local variables           [sp+0]  |
+-----------------------------------+  <- new sp (= old sp - 16)
low address   (stack grows down)
```

A complete non-leaf function (one that calls other functions)

```
my_function:
    # 1. PROLOGUE — set up the frame
    addi sp, sp, -16      # reserve 16 bytes
    sw   ra, 12(sp)       # save return address
    sw   s0, 8(sp)        # save s0 (callee-saved)

    # 2. BODY
    mv   s0, a0           # keep our argument safe in s0
    li   a0, 42           # argument for the child call
    jal  ra, child        # call child (overwrites ra!)
    add  a0, a0, s0       # a0 = child's result + our saved arg

    # 3. EPILOGUE — undo the prologue, in reverse order
    lw   s0, 8(sp)        # restore s0
    lw   ra, 12(sp)       # restore ra  <- without this, ret jumps to child!
    addi sp, sp, 16       # free the frame
    ret                   # = jalr x0, ra, 0 (jump back to caller)
```

Why save ra? `jal` overwrites ra with the return address inside the child. If our function then called another child, ra would get overwritten again and `ret` would send us back to the wrong place — infinite loop or crash. Saving ra on the stack is the fix.

---

8. Pipelines, Hazards, and Execution Order

8.1 The 5-stage pipeline

Instead of finishing instruction 1 before starting instruction 2, the CPU overlaps them — each stage works on a different instruction at the same time:

```
Cycle:      1     2     3     4     5     6     7
Instr 1:   [IF]  [ID]  [EX]  [MEM] [WB]
Instr 2:         [IF]  [ID]  [EX]  [MEM] [WB]
Instr 3:               [IF]  [ID]  [EX]  [MEM] [WB]
```

Result: one instruction finishes every cycle instead of every five.

8.2 Pipeline hazards (and the fixes)

- Structural hazard: two instructions need the same hardware at once (e.g., one fetching from RAM while another loads data). Fix: separate instruction memory from data memory (Harvard architecture).
- Data hazard: instruction B needs the result of instruction A, but A hasn't finished yet.
  - Fix 1 — forwarding/bypassing: route the ALU output straight back into the next instruction's inputs instead of waiting for writeback.
  - Fix 2 — stall: if B needs data loaded by a `lw` one instruction earlier, forwarding can't help (the data isn't there yet). The CPU inserts a one-cycle "bubble" (a nop). The assembler can also just reorder instructions to hide it.
- Control hazard: a branch changes the PC, but the CPU already fetched the next 1–2 instructions assuming no branch.
  - Fix — branch prediction: the CPU guesses taken/not-taken. Right guess = no cost. Wrong guess = the speculative instructions are flushed and fetch restarts at the correct target.

8.3 In-order vs out-of-order

- In-order: instructions run strictly in program order. If one stalls on a slow RAM read, everything behind it waits. Simple, low power — used in microcontrollers.
- Out-of-order (OoO): the CPU looks ahead in the instruction stream and runs later instructions whose inputs are ready, even if an earlier one is stalled. Needs extra hardware: register renaming, reservation stations, and a reorder buffer (ROB) to retire instructions in the original order.

---

9. The Standard Extensions (M, A, F, D, C, V)

RISC-V never changes its base — you just add letters:

Ext	Name	What it adds	
I	Base Integer	The mandatory base (RV32I / RV64I)	
M	Multiply/Divide	`mul, mulh, mulhu, div, rem` in hardware	
A	Atomics	Multi-core synchronization: `lr.w`, `sc.w`, `amoadd.w`	
F	Single-precision float	32-bit IEEE 754 math, new registers f0–f31	
D	Double-precision float	64-bit IEEE 754 math	
C	Compressed	16-bit versions of common instructions, 25–30% smaller code	
V	Vector	SIMD for ML, DSP, matrices — variable-length vectors	
G	General	Shorthand for I + M + A + F + D (+ Zicsr + Zifencei)	

---

10. Project 1: Caesar Cipher (in-place stream engine)

(Content: an in-place Caesar cipher implementation — shifts each letter of a string by a fixed key while streaming through memory. Demonstrates byte loads/stores (`lb`/`sb`), loops, and pointer arithmetic in assembly.)

11. Project 2: Multi-Operation Calculator

A calculator that reads an operator (`+ - * /`) and two integers, computes the result, and checks for division by zero in software before dividing.

Why this project matters

- Shows dispatching: a chain of `beq` on the operator's ASCII code picks which operation to run.
- Shows input validation: testing the divisor against zero before `div`, so the program fails gracefully with a message instead of a wrong answer.

The code (`riscv_calculator.s`)

```asm
# RISC-V Calculator — runs on the Venus simulator (needs RV32IM for mul/div)
.data
val_a:       .word 120
val_b:       .word 15
op_choice:   .byte 42          # ASCII: '+'=43 '-'=45 '*'=42 '/'=47

str_header:  .asciiz "--- RISC-V Compute Engine Result ---\n"
str_ans:     .asciiz "Computed Result: "
str_rem:     .asciiz "\nRemainder:       "
str_div_err: .asciiz "\nError: Division by zero!\n"
newline:     .asciiz "\n"

.text
.globl main
main:
    addi sp, sp, -24
    sw   ra, 20(sp)
    sw   s0, 16(sp)
    sw   s1, 12(sp)
    sw   s2, 8(sp)

    # print header (ecall 4 = print string)
    li   a0, 4
    la   a1, str_header
    ecall

    # load operands: a0 = first number, a1 = second, a2 = operator
    lw   a0, val_a
    lw   a1, val_b
    lb   a2, op_choice

    jal  ra, execute_calculation
    mv   s0, a0            # s0 = result
    mv   s1, a1            # s1 = remainder (division only)
    mv   s2, a2            # s2 = error flag (0 = ok, 1 = error)

    bnez s2, handle_calc_error

    # print result
    li   a0, 4
    la   a1, str_ans
    ecall
    li   a0, 1             # ecall 1 = print integer
    mv   a1, s0
    ecall

    # print remainder only if operator was '/'
    lb   t0, op_choice
    li   t1, 47            # ASCII '/'
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
    lw   s2, 8(sp)
    lw   s1, 12(sp)
    lw   s0, 16(sp)
    lw   ra, 20(sp)
    addi sp, sp, 24
    li   a0, 10            # ecall 10 = exit
    ecall

# execute_calculation(a0=num1, a1=num2, a2=operator)
# returns: a0 = result, a1 = remainder, a2 = error flag
execute_calculation:
    li   t0, 43            # '+'
    beq  a2, t0, do_add
    li   t0, 45            # '-'
    beq  a2, t0, do_sub
    li   t0, 42            # '*'
    beq  a2, t0, do_mul
    li   t0, 47            # '/'
    beq  a2, t0, do_div
    li   a0, 0             # unknown operator -> error
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
    mul  a0, a0, a1        # M extension
    li   a1, 0
    li   a2, 0
    ret

do_div:
    beqz a1, div_by_zero_error   # check BEFORE dividing
    div  t0, a0, a1              # M extension
    rem  t1, a0, a1
    mv   a0, t0
    mv   a1, t1
    li   a2, 0
    ret

div_by_zero_error:
    li   a0, 0
    li   a1, 0
    li   a2, 1             # set error flag
    ret
```

(Verified on Venus: 120 * 15 = 1800.)

---

12. Running the Code

Option A: Venus (browser, easiest)
1. Open https://venus.cs61c.org/
2. Paste the code in the Editor tab
3. Click "Assemble & Simulate from Editor"
4. Press Run, or Step through it instruction-by-instruction while watching registers and memory change

Option B: Real toolchain on Linux + QEMU

```bash
# install
sudo apt update
sudo apt install gcc-riscv64-unknown-elf binutils-riscv64-unknown-elf qemu-user

# assemble + link (rv32im because we use mul/div)
riscv64-unknown-elf-as -march=rv32im -mabi=ilp32 -o program.o program.s
riscv64-unknown-elf-ld -m elf32lriscv -o program.elf program.o

# run
qemu-riscv32 ./program.elf
```

---

Study reference: "RISC-V Architecture Tutorial" by Nikhil Kumar Rajput, plus the official RISC-V Unprivileged ISA Specification.
