

## 指令类型


R
    R4-type
I
S
U

B，J是对imm的处理的变体


G = IMAFDZicsr_Zifencei


## RV32I


### Integer Computational Instructions


Integer Register-Immediate Instructions

* I-imm[11:0], OP-IMM, I-type
* func7 I-shamt[4:0], OP-IMM, I-type  (shamt: shift amount)
* U-imm[32:12], LUI/AUIPC, U-type

Integer Register-Register Operations

* func7, OP, R-type

NOP Instruction

* addi x0, x0, 0

### Control Transfer Instructions


Unconditional Jumps

* J-imm[20:1], JAL, J-type
* I-imm[11:0], JALR, I-type

Conditional Branches

* B-imm[12:0], BRANCH, B-type


### Load and Store Instructions

* I-imm[11:0], LOAD, I-type
* S-imm[11:0], STORE, S-type

### Memory Ordering Instructions

// 细看
* ,fun3(FENCE), MISC-MEM,


### Environment Call and Breakpoints

* funct12, SYSTEM, I-type
    - ECALL/EBREAK

### HINT Instructions


## Zifencei extension

* 在MISC-MEM中加了funct3为FENCE.I的指令

## M extension

### Multiplication Operations

* funct7, OP, R-type
* funct7, OP-32, R-type

### Division Operations

* funct7, OP, R-type
* funct7, OP-32, R-type

### Zmmul Extenison (不需要Division)

implements the multiplication subset of the M extension

MUL, MULH, MULHU, MULHSU, and (for RV64 only) MULW.

The Zmmul extension enables low-cost implementations that require multiplication operations but not division

## A extension

### Specifying Ordering of Atomic Instructions

* FENCE, to order accesses to one or both of these two address domains(I/O and memory domain).
* each atomic instruction has two bits, aq and rl, used to specify additional memory ordering constraints as viewed by other RISC-V harts.
    - ACQUIRE -> Load, Store
    - Load, Store -> RELEASE
    - ACQUIRE 和 RELEASE 则是SC顺序


*If both the aq and rl bits are set, the atomic memory operation is sequentially consistent and cannot be observed to happen before any earlier memory operations or after any later memory operations in the same RISC-V hart and to the same address domain.*

### Load-Reserved/Store-Conditional Instructions

* funct5, aq, rl, AMO, R-type
    - funct5: LR.W/D(SC.W/D)


### Eventual Success of Store-Conditional Instructions


### Atomic Memory Operations

* funct5, aq, rl, AMO, U-type

## Zicsr extension


### CSR Instructions

* source, rs1, funct3, rd, SYSTEM   I-type

### Counter


## F extension

### F Register State

FLEN=32 for the F single-precision floating-point register

### Floating-Point Control and Status Register

### NaN Generation and Propagation

canonical NaN  0x7fc00000

### Subnormal Arithmetic

Operations on subnormal numbers are handled in accordance with the IEEE 754-2008 standard

### Single-Precision Load and Store Instructions

* I-imm[11:0], LOAD-FP, I-type
* S-imm[11:0], STORE-FP, S-type
 
### Single-Precision Floating-Point Computational Instructions

* funct5, fmt(floating-point format), rm(round mode), OP-FP, R-type
* rs3, fmt, rs2, rs1, rm, rd, F[N]MADD/F[N]MSUB, R4-type

### Single-Precision Floating-Point Conversion and Move Instructions


* I2F/F2I, fmt, OP-FP, R-type
* FSGNJ, fmt, OP-FP, R-Type （符号注入） 用 f[rs1]指数和有效数以及 f[rs2]的符号的符号位，来构造一个新的双精度浮点数，并将其写入 f[rd]。
* FMV.X.W/FMV.W.X, fmt, OP-FP, R-type  (Floating-point Move Word to Integer)

### Single-Precision Floating-Point Compare Instructions

* FCMP, fmt, OP-FP, R-type


### Single-Precision Floating-Point Classify Instruction

* FCLASS, fmt, OP-FP, R-type

## D extension

### D Register State

FLEN=64

### NaN Boxing of Narrower Value

### Double-Precision Load and Store Instructions

* I-imm[11:0], LOAD-FP, I-type
* S-imm[11:0], STORE-FP, S-type

### Double-Precision Floating-Point Computational Instructions

* funct5, fmt(floating-point format), rm(round mode), OP-FP, R-type
* rs3, fmt, rs2, rs1, rm, rd, F[N]MADD/F[N]MSUB, R4-type

### Double-Precision Floating-Point Conversion and Move Instructions


* I2D/D2I, fmt, OP-FP, R-type
* FCVT.S.D, fmt, OP-FP, R-type
* FSGNJ, fmt, OP-FP, R-type
* FMV.D.X, fmt, OP-FP, R-type

### Double-Precision Floating-Point Compare Instructions


* FCMP, fmt, OP-FP, R-type

### Double-Precision Floating-Point Classify Instruction

* FCLASS, fmt, OP-FP, R-type

## 