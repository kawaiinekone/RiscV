# RiscV
# riscv-understanding

hi, i'm Shubhi Rai, first sem Bsc Hons computer science. this repo is my attempt to understand
risc-v before a workshop on Monday where i might get shortlisted.

## what is risc-v (my version)

risc-v is a free and open processor design. anyone can read the spec,
build one, modify it. no license fees, no nda, no "contact sales for
pricing". started at uc berkeley in 2010. the "v" is roman numeral 5.

i like it because it feels like the linux of processors. x86 and arm are
like proprietary software — you can use them but you can't see inside.

## risc vs cisc

risc = reduced instruction set. few, simple instructions.
cisc = complex instruction set. many, powerful instructions.

x86 is cisc. one instruction can do a lot, like add two numbers directly
from memory. risc-v says no — everything goes through registers. memory
is only touched by load/store instructions.

the analogy that helped me: risc is like cooking where you can only hold
a few things in your hands at once. everything else is across the kitchen
(memory). you have to walk over, grab it (load), bring it back, cook with
it, then put it back (store). cisc lets you cook directly from the fridge.

which, fine, seems slower. but the processor hardware gets way simpler,
and simple means fast and power-efficient.

## registers

risc-v has 32 integer registers, x0 to x31. x0 is always zero. always.
you can write to it but it stays zero. i thought this was dumb until i
realized it makes comparisons free.

| reg | abi name | what it's for |
|-----|----------|---------------|
| x0  | zero     | always 0, writes ignored |
| x1  | ra       | return address |
| x2  | sp       | stack pointer |
| x5-x7, x28-x31 | t0-t6 | temporaries |
| x10-x17 | a0-a7 | function args / return values |
| x8-x9, x18-x27 | s0-s11 | saved registers |

### caller vs callee-saved

this took me embarrassingly long. the t registers are caller-saved —
if you call a function, it might trash them. the s registers are callee-saved — if a function uses them, it must put them back.

group project analogy: t-regs are scrap paper on the table. anyone can
scribble on it. s-regs are your teammate's notebook. you can borrow it,
but you better return it exactly as you found it, or things break.

## calling convention + stack

honestly? the stack took me a day to get.
the stack is a region of memory, pointed to by sp (x2). it grows DOWN.
you allocate space by subtracting from sp. it works like a stack of
plates — you only ever touch the top. push = put a plate on. pop =
take the top plate off. you can't grab from the middle.

when you call a function:
1. args go in a0-a7
2. jal saves the return address in ra
3. the function does its thing
4. return value goes in a0
5. ret jumps back

```asm
# my first working function call
addi a0, zero, 5    # arg = 5
jal  ra, square     # call square(5)
# a0 now has 25

square:
    mul a0, a0, a0  # just multiply it by itself
    ret             # jump back
