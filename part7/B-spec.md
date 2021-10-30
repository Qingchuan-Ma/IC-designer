# Notion
* .w: 32 bits word
* .uw: unsigned 32 bits word
* .b: 8 bits
* .h: 16 bits


# Address generation instructions (Zba)

| 助记符  |指令意思 | 支持RV32 |
| ----  | ---- | ---- |
|   add.uw rd, rs1, rs2     |   Add unsigned word   |   否 |
|   sh1add rd, rs1, rs2     |   Shift left by 1 and add |   是 |
|   sh1add.uw rd, rs1, rs2  |   Shift unsigned word left by 1 and add | 否 |
|   sh2add rd, rs1, rs2     |   Shift left by 2 and add |   是|
|   sh2add.uw rd, rs1, rs2  |   Shift unsigned word left by 2 and add | 否 |
|   sh3add rd, rs2, rs2     |   Shift left by 3 and add | 是| 
| sh3add.uw rd, rs1, rs2    |   Shift unsigned word left by 3 and add |否|
| slli.uw rd, rs1, imm      |   Shift-left unsigned word (Immediate) | 否 |


# Basic bit-manipulation (Zbb)


| 助记符  |指令意思 | 支持RV32 |
| ----  | ---- | ---- |
|andn rd, rs1, rs2 |AND with inverted operand  |是
|clz rd, rs |Count leading zero bits|是
|clzw rd, rs |Count leading zero bits in word|否
|cpop rd, rs |Count set bits |是
|cpopw rd, rs |Count set bits in word |否
|ctz rd, rs |Count trailing zero bits |是
| ctzw rd, rs |Count trailing zero bits in word |否
|max rd, rs1, rs2 |Maximum |是
| maxu rd, rs1, rs2 |Unsigned maximum |是
|  min rd, rs1, rs2 |Minimum |是
|  minu rd, rs1, rs2 |Unsigned minimum |是
|  orc.b rd, rs1, rs2 |Bitwise OR-Combine, byte granule |是
|  orn rd, rs1, rs2 |OR with inverted operand |是
|  rev8_rd_, rs |Byte-reverse register |是
|  rol rd, rs1, rs2 |Rotate left (Register) |是
| rolw rd, rs1, rs2 |Rotate Left Word (Register)|否 
|  ror rd, rs1, rs2 |Rotate right (Register) |是
|  rori rd, rs1, shamt |Rotate right (Immediate) |是
| roriw rd, rs1, shamt |Rotate right Word (Immediate) |否
| rorw rd, rs1, rs2 |Rotate right Word (Register)|否
|sext.b rd, rs |Sign-extend byte |是
|  sext.h rd, rs |Sign-extend halfword |是
|xnor rd, rs1, rs2 |Exclusive NOR |是
|  zext.h rd, rs |Zero-extend halfword |是


# Carry-less multiplication (Zbc)

| 助记符  |指令意思 | 支持RV32 |
| ----  | ---- | ---- |
|clmul rd, rs1, rs2 |Carry-less multiply (low-part) |是
|  clmulh rd, rs1, rs2 |Carry-less multiply (high-part) |是
|  clmulr rd, rs1, rs2 |Carry-less multiply (reversed)|是

# Single-bit instructions ((Zbd)

| 助记符  |指令意思 | 支持RV32 |
| ----  | ---- | ---- |
|bclr rd, rs1, rs2 |Single-Bit Clear (Register) |是
|  bclri rd, rs1, imm |Single-Bit Clear (Immediate) |是
|  bext rd, rs1, rs2 |Single-Bit Extract (Register) |是
|  bexti rd, rs1, imm |Single-Bit Extract (Immediate) |是
|  binv rd, rs1, rs2 |Single-Bit Invert (Register) |是
|  binvi rd, rs1, imm |Single-Bit Invert (Immediate) |是
|  bset rd, rs1, rs2 |Single-Bit Set (Register) |是
|  bseti rd, rs1, imm |Single-Bit Set (Immediate) |是

# Specification

![](../assets/rv-b-spec.pdf)


