# V-SPEC


## glossary

* ELEN: The maximum size in bits of a vector element that any operation can produce or consume；任何操作可以产生或消耗的向量元素的最大位数，大于等于8，必须是2的幂
* VLEN: The number of bits in a single vector register；寄存器位数，VLEN>=ELEN，必须2的幂，小于2^16
* SEW: selected element width 一个向量寄存器被分成了VLEN/SEW个元素
* LMUL: vector length multiplier 如果大于1，说明要和别的寄存器合并成1个，必须实现1,2,4,8；
* VLMAX = LMUL\*VLEN/SEW: 在指定SEW和LMUL后，一条指令中能够处理的最大元素个数
* SEWMIN: 最小值8；ELEN是最大的支持SEW。所以LMUL>=SEWMIN/ELEN
```
ELEN=32的标准扩展LMUL必须实现1/2和1/4；其实就是说64bit的寄存器，存2个32bit的数，SEW=32，LMUL如果是1的话，那么一条指令能够处理2个数，意思是指令是以寄存器为单位控制的。
ELEN=64的标准扩展LMUL必须实现1/2,1/4和1/8
```
* EEW: effective element width
* EMUL: effective LMUL
```
总会有EEW/EMUL = SEW/LMUL
```
* AVL: application vector length
* Prestart: 元素index小于vstart的那些元素，不产生异常也不更新目标向量寄存器
* Body: 元素index大于等于vstart的那些元素，但是小于vl中的目前向量长度设置
* Tail: 不产生异常；不更新目标寄元素除非vtype.vta=1.这种情况下目标元素不可知
* Inactive: 不产生异常；不更新目标寄存器元素除非vtype.vma=1.这种情况下目标元素不可知
```
for element index x
prestart(x) = (0 <= x < vstart)
body(x) = (vstart <= x < vl)
tail(x) = (vl <= x < max(VLMAX,VLEN/SEW))
mask(x) = unmasked || v0.mask[x] == 1
active(x) = body(x) && mask(x)
inactive(x) = body(x) && !mask(x)
```


## Vector Extension Programmer’s Model
* 在local hart上执行满足程序顺序
* 指令层面（多核）满足RVWMO
* Instructions affected by the vector length register vl have a control dependency on vl, rather than a data dependency.
* masked vector instructions have a control dependency on the source mask register, rather than a data dependency
* 复位后，vstart/vxrm/vxsat可以是任意值， vtype/vl必须有能被正确读出并写用vsetvl指令恢复的值。

### mstatus[10:9]
向量上下文状态字段VS：mstatus[10:9]（同样包括sstatus[10:9]）。VS的定义类似于浮点上下文状态FS：

* 当 VS 字段被设置为 Off ，试图执行任何向量指令或访问向量CSRs时，会引发非法指令异常；
* 当被设置为 Initial 或 Clean ，执行更改向量状态的指令，包括向量 CSRs 指令，该字段会变为 Dirty 。


### vstart
如上表所示，为prestart的index数


### vxsat
vxsat[0]表示是否一个定点指令需要饱和来适应到目标格式，该位在vcsr中也有镜像。


### vxrm

假设要舍入的数为v，d是需要舍入的位数，则舍入结果为(v >> d) + r，下表为不同舍入模式下的r结果

|vxrm[1:0]| Abbreviation| Rounding Mode| Rounding increment, r|
|---|---|---|---|
|0 0| rnu| round-to-nearest-up (add +0.5 LSB) |v[d-1]
|0 1| rne| round-to-nearest-even |v[d-1] & (v[d-2:0]≠0 | v[d])
|1 0| rdn| round-down (truncate) |0
|1 1| rod| round-to-odd (OR bits into LSB, aka "jam") |!v[d] & v[d-1:0]≠0

```
roundoff_unsigned(v, d) = (unsigned(v) >> d) + r
roundoff_signed(v, d) = (signed(v) >> d) + r
```

### vcsr
拥有vxrm和vxsat两个镜像。

### vl
长度为XLEN的只读寄存器，只能通过vset{i}vl{i}指令和fault-only-fisrt向量load指令变体更新，保存一个无符号整数，用于指定元素数量。



### vtype
长度为XLEN的只读寄存器，只能通过vset{i}vl{i}指令更新，包括vill, vma, vta, vsew[2:0], and vlmul[2:0]域


* 如果vill设置为1，任何尝试依赖于vtype的指令都会产生非法指令异常，且其余的XLEN-1的所有值均被强制置0
* vma和vta （Mask destination tail elements are always treated as tail-agnostic, regardless of the setting of vta）

|vta| vma| Tail Elements | Inactive Elements|
|---|---|---|---|
|0|0 |undisturbed| undisturbed|
|0 |1 |undisturbed |agnostic|
|1| 0| agnostic |undisturbed|
|1| 1| agnostic |agnostic|

* undisturbed: 目标元素保持原来所保持的值
* agnostic: 目标元素不可知，可能是1覆写，也可能是保持原值。

```
ta # Tail agnostic
tu # Tail undisturbed
ma # Mask agnostic
mu # Mask undisturbed
vsetvli t0, a0, e32, m4, ta, ma # Tail agnostic, mask agnostic
vsetvli t0, a0, e32, m4, tu, ma # Tail undisturbed, mask agnostic
vsetvli t0, a0, e32, m4, ta, mu # Tail agnostic, mask undisturbed
vsetvli t0, a0, e32, m4, tu, mu # Tail undisturbed, mask undisturbed
```
### vlenb
长度为XLEN的只读寄存器，保存VLEN/8，即向量寄存器的字节数。

## Vector Instruction Formats

### 缩写

LOAD-FP:

* vm: vector mask
* mop: address mode
* mew: extended memory element width （目前没用到，reserved）
* nf: species the number of fields in each segment, for segment load/stores
* lumop: load unit-stride vector addressing modes

STORE-FP:

* sumop

OP-V:

* funct6


### Vector Operands
A destination vector register group can overlap a source vector register group only if one of the following holds:

* The destination EEW equals the source EEW.
* The destination EEW is smaller than the source EEW and the overlap is in the lowest-numbered part of the source register group (e.g., when LMUL=1, vnsrl.wi v0, v0, 3 is legal, but a destination of v1 is not).
* The destination EEW is greater than the source EEW, the source EMUL is at least 1, and the overlap is in the highestnumbered part of the destination register group (e.g., when LMUL=8, vzext.vf4 v0, v6 is legal, but a source of v0,  v2, or v4 is not).

### Vector Masking
* inactive不产生异常，v0是mask vector register，EEW=1;
* inst[25]的vm域，如果为0，则表示只有v.mask[i]=1的地方才有结果；如果为0，则说明unmasked
```
vop.v* v1, v2, v3, v0.t # enabled where v0.mask[i]=1, vm=0
vop.v* v1, v2, v3 # unmasked vector operation, vm=1
```

### Configuration-Setting Instruction
```
vsetvli rd, rs1, vtypei # rd = new vl, rs1 = AVL, vtypei = new vtype setting
vsetivli rd, uimm, vtypei # rd = new vl, uimm = AVL, vtypei = new vtype setting
vsetvl rd, rs1, rs2 # rd = new vl, rs1 = AVL, rs2 = new vtype value
```

```
e8 # SEW=8b
e16 # SEW=16b
e32 # SEW=32b
e64 # SEW=64b

mf8 # LMUL=1/8
mf4 # LMUL=1/4
mf2 # LMUL=1/2
m1 # LMUL=1, assumed if m setting absent
m2 # LMUL=2
m4 # LMUL=4
m8 # LMUL=8

Examples:
        vsetvli t0, a0, e8 # SEW= 8, LMUL=1
        vsetvli t0, a0, e8, m2 # SEW= 8, LMUL=2
        vsetvli t0, a0, e32, mf2 # SEW=32, LMUL=1/2
```

|rd |rs1| AVL value |Effect on vl|
|---|---|----|-----|
|-| !x0| Value in x[rs1]| Normal stripmining|
|!x0| x0| ~0(最大无符号整数)| Set vl to VLMAX|
|x0| x0| Value in vl register| Keep existing vl (of course, vtype may change)|

vset{i}vl{i}在首先根据vtype参数设置完VLMAX之后，就会根据下面的约束设置vl：

* vl = AVL if AVL <= VLMAX
* 如果AVL < 2\*VLMAX的话，cel(AVL / 2) <= vl <= VLMAX
* vl = VLMAX 如果 AVL >= 2\*VLMAX
* 给定相同AVL和VLMAX，结果是决定性的
* 特殊情况
    - vl = 0 if AVL = 0
    - vl > 0 if AVL > 0
    - vl ≤ VLMAX
    - vl ≤ AVL
    - 一个从vl中读取值用作AVL参数时，会导致vl中出现相同值，前提是结果的VLMAX等于读取vl时的VLMAX值


## Vector Loads and Stores

* unit-stride
    - unit-stride
    - unit-stride, whole register load
    - unit-stride, mask load, EEW=8 (专门用来从寄存器和内存中传输mask值的方法)
    - unit-stride fault-only-first
* indexed-unordered
* stride
* indexed-ordered
### Unit
mop+lumop决定
```
# Vector unit-stride loads and stores

# vd destination, rs1 base address, vm is mask encoding (v0.t or <missing>)
vle8.v vd, (rs1), vm # 8-bit unit-stride load
vle16.v vd, (rs1), vm # 16-bit unit-stride load
vle32.v vd, (rs1), vm # 32-bit unit-stride load
vle64.v vd, (rs1), vm # 64-bit unit-stride load

# vs3 store data, rs1 base address, vm is mask encoding (v0.t or <missing>)
vse8.v vs3, (rs1), vm # 8-bit unit-stride store
vse16.v vs3, (rs1), vm # 16-bit unit-stride store
vse32.v vs3, (rs1), vm # 32-bit unit-stride store
vse64.v vs3, (rs1), vm # 64-bit unit-stride store

# Vector unit-stride mask load
vlm.v vd, (rs1) # Load byte vector of length ceil(vl/8)

# Vector unit-stride mask store
vsm.v vs3, (rs1) # Store byte vector of length ceil(vl/8)

```

### Stride
mop决定

```
# Vector strided loads and stores

# vd destination, rs1 base address, rs2 byte stride
vlse8.v vd, (rs1), rs2, vm # 8-bit strided load
vlse16.v vd, (rs1), rs2, vm # 16-bit strided load
vlse32.v vd, (rs1), rs2, vm # 32-bit strided load
vlse64.v vd, (rs1), rs2, vm # 64-bit strided load

# vs3 store data, rs1 base address, rs2 byte stride
vsse8.v vs3, (rs1), rs2, vm # 8-bit strided store
vsse16.v vs3, (rs1), rs2, vm # 16-bit strided store
vsse32.v vs3, (rs1), rs2, vm # 32-bit strided store
vsse64.v vs3, (rs1), rs2, vm # 64-bit strided store
```
如果rs2=x0，则实现可以直接复制而不是实现真正的访问。（如果要保证所有访问，使用值为0寄存器作为rs2进行stride访问）
如果rs2!=x0，则必须实现所有访问，但是访问顺序不会有序。

### Indexed
mop决定
```
# Vector indexed loads and stores

# Vector indexed-unordered load instructions
# vd destination, rs1 base address, vs2 byte offsets
vluxei8.v vd, (rs1), vs2, vm # unordered 8-bit indexed load of SEW data
vluxei16.v vd, (rs1), vs2, vm # unordered 16-bit indexed load of SEW data
vluxei32.v vd, (rs1), vs2, vm # unordered 32-bit indexed load of SEW data
vluxei64.v vd, (rs1), vs2, vm # unordered 64-bit indexed load of SEW data

# Vector indexed-ordered load instructions
# vd destination, rs1 base address, vs2 byte offsets
vloxei8.v vd, (rs1), vs2, vm # ordered 8-bit indexed load of SEW data
vloxei16.v vd, (rs1), vs2, vm # ordered 16-bit indexed load of SEW data
vloxei32.v vd, (rs1), vs2, vm # ordered 32-bit indexed load of SEW data
vloxei64.v vd, (rs1), vs2, vm # ordered 64-bit indexed load of SEW data

# Vector indexed-unordered store instructions
# vs3 store data, rs1 base address, vs2 byte offsets
vsuxei8.v vs3, (rs1), vs2, vm # unordered 8-bit indexed store of SEW data
vsuxei16.v vs3, (rs1), vs2, vm # unordered 16-bit indexed store of SEW data
vsuxei32.v vs3, (rs1), vs2, vm # unordered 32-bit indexed store of SEW data
vsuxei64.v vs3, (rs1), vs2, vm # unordered 64-bit indexed store of SEW data

# Vector indexed-ordered store instructions
# vs3 store data, rs1 base address, vs2 byte offsets
vsoxei8.v vs3, (rs1), vs2, vm # ordered 8-bit indexed store of SEW data
vsoxei16.v vs3, (rs1), vs2, vm # ordered 16-bit indexed store of SEW data
vsoxei32.v vs3, (rs1), vs2, vm # ordered 32-bit indexed store of SEW data
vsoxei64.v vs3, (rs1), vs2, vm # ordered 64-bit indexed store of SEW data
```

### Unit-stride Fault-Only-First Loads
mop+lumop决定

* 如果第零个元素产生了异常，vl不会被修改，并且会产生trap
* 如果任何一个大于0 index的元素产生了异常，trap则不会发生，并且vl会减少到这个元素的index
* 当fault-only-first指令因中断而trap时，实现不应该减少vl，而应该设置一个vstart值
```
# Vector unit-stride fault-only-first loads

# vd destination, rs1 base address, vm is mask encoding (v0.t or <missing>)
vle8ff.v vd, (rs1), vm # 8-bit unit-stride fault-only-first load
vle16ff.v vd, (rs1), vm # 16-bit unit-stride fault-only-first load
vle32ff.v vd, (rs1), vm # 32-bit unit-stride fault-only-first load
vle64ff.v vd, (rs1), vm # 64-bit unit-stride fault-only-first load
```

### Segment Instruction
nf决定

“segment”反映了被移动的项是具有同构元素的子数组。这些操作可以用于在内存和寄存器之间转换数组，并且可以通过将结构中的每个字段解压缩到单独的向量寄存器中来支持“结构数组”数据类型的操作。

必须满足 EMUL * NFIELDS ≤ 8，意思是不能一次使用掉超过1/4的向量通用寄存器。如果EMUL>1，则每个field(可以认为是结构体内变量)需要占据的寄存器group包括多个连续的向量寄存器，这种情况下group必须存寻通常的向量寄存器对齐规则（例如EMUL=2的时候，必须是以偶数寄存器为目标寄存器）

#### Unit-Stride Segment
```
# Format
vlseg<nf>e<eew>.v vd, (rs1), vm # Unit-stride segment load template
vsseg<nf>e<eew>.v vs3, (rs1), vm # Unit-stride segment store template

# Examples
vlseg8e8.v vd, (rs1), vm # Load eight vector registers with eight byte fields.

vsseg3e32.v vs3, (rs1), vm # Store packed vector of 3*4-byte segments from vs3,vs3+1,vs3+2 to mem

# Template for vector fault-only-first unit-stride segment loads.
vlseg<nf>e<eew>ff.v vd, (rs1), vm # Unit-stride fault-only-first segment loads
```
对于segment的fault-only-first，不管是否元素index是不是0，加载或者不加载异常的这个segment是实现自己决定的。
#### Strided Segment
```
# Format
vlsseg<nf>e<eew>.v vd, (rs1), rs2, vm # Strided segment loads
vssseg<nf>e<eew>.v vs3, (rs1), rs2, vm # Strided segment stores

# Examples
vsetvli a1, t0, e8, ta, ma
vlsseg3e8.v v4, (x5), x6    # Load bytes at addresses x5+i*x6 into v4[i],
                            # and bytes at addresses x5+i*x6+1 into v5[i],
                            # and bytes at addresses x5+i*x6+2 into v6[i].

# Examples
vsetvli a1, t0, e32, ta, ma
vssseg2e32.v v2, (x5), x6   # Store words from v2[i] to address x5+i*x6
                            # and words from v3[i] to address x5+i*x6+4

```
顺序不做任何保证
#### Indexed Segment
```
# Format
vluxseg<nf>ei<eew>.v vd, (rs1), vs2, vm # Indexed-unordered segment loads
vloxseg<nf>ei<eew>.v vd, (rs1), vs2, vm # Indexed-ordered segment loads
vsuxseg<nf>ei<eew>.v vs3, (rs1), vs2, vm # Indexed-unordered segment stores
vsoxseg<nf>ei<eew>.v vs3, (rs1), vs2, vm # Indexed-ordered segment stores

# Examples
vsetvli a1, t0, e8, ta, ma
vluxseg3ei32.v v4, (x5), v3 # Load bytes at addresses x5+v3[i] into v4[i],
                            # and bytes at addresses x5+v3[i]+1 into v5[i],
                            # and bytes at addresses x5+v3[i]+2 into v6[i].

# Examples
vsetvli a1, t0, e32, ta, ma
vsuxseg2ei32.v v2, (x5), v5 # Store words from v2[i] to address x5+v5[i]
                            # and words from v3[i] to address x5+v5[i]+4
```
vs2是index的register，destination不能覆盖该register

### Unit-stride Whole Regitser Instruction
effective vector length: evl=NFIELDS\*VLEN/EEW，不管vtype和vl的；vstart >= evl的话，不会写任何元素
```
# Format of whole register load and store instructions.
vl1r.v v3, (a0) # Pseudoinstruction equal to vl1re8.v

vl1re8.v v3, (a0) # Load v3 with VLEN/8 bytes held at address in a0
vl1re16.v v3, (a0) # Load v3 with VLEN/16 halfwords held at address in a0
vl1re32.v v3, (a0) # Load v3 with VLEN/32 words held at address in a0
vl1re64.v v3, (a0) # Load v3 with VLEN/64 doublewords held at address in a0
vl2r.v v2, (a0) # Pseudoinstruction equal to vl2re8.v v2, (a0)

vl2re8.v v2, (a0) # Load v2-v3 with 2*VLEN/8 bytes from address in a0
vl2re16.v v2, (a0) # Load v2-v3 with 2*VLEN/16 halfwords held at address in a0
vl2re32.v v2, (a0) # Load v2-v3 with 2*VLEN/32 words held at address in a0
vl2re64.v v2, (a0) # Load v2-v3 with 2*VLEN/64 doublewords held at address in a0

vl4r.v v4, (a0) # Pseudoinstruction equal to vl4re8.v

vl4re8.v v4, (a0) # Load v4-v7 with 4*VLEN/8 bytes from address in a0
vl4re16.v v4, (a0)
vl4re32.v v4, (a0)
vl4re64.v v4, (a0)

vl8r.v v8, (a0) # Pseudoinstruction equal to vl8re8.v

vl8re8.v v8, (a0) # Load v8-v15 with 8*VLEN/8 bytes from address in a0
vl8re16.v v8, (a0)
vl8re32.v v8, (a0)
vl8re64.v v8, (a0)

vs1r.v v3, (a1) # Store v3 to address in a1
vs2r.v v2, (a1) # Store v2-v3 to address in a1
vs4r.v v4, (a1) # Store v4-v7 to address in a1
vs8r.v v8, (a1) # Store v8-v15 to address in a1
```


## Vector Integer Arithmetic Instructions

* single-width integer add, subtract and reverse subtract
* widening integer add/subtract (signed/unsigned)
* integer extension
* integer add-with-carry/subtract-with-borrow
* bitwise logical
* single-width shift
* narrowing integer right shift
* integer compare
    - set if euqal(not equal)
    - set if less than, unsigned/signed
* integer min/max
* single-width integer multiply
* integer divide
* widening integer multiply
* signle-width integer multiply-add
* widening integer multiply-add
* integer merge (vmerge.vvm vd, vs2, vs1, v0 # vd[i] = v0.mask[i] ? vs1[i] : vs2[i])
* integer move


## Vector Fixed-Point Arithmetic Instructions

* single-width saturating add and subtract
* single-width averaging add and subtract
* single-width fractional multiply with rouding and saturation
* single-width scaling shift
* narrowing fixed-point clip


## Vector Floating-Point Instructions

* single-width floating-point add/subtract
* widening floating-point add/subtract
* single-width floating-point multiply/divide
* widening floating-point multiply
* single-width floating-point fused multiply-add
* widening floating-point fused multiply-add
* floating-point square-root
* floating-point reciprocal square-root estimate (1/sqrt(x))
* floating-point reciprocal estimate
* floating-point min/max
* floating-point sign-injection
* floating-point compare
* floating-point classify
* floating-point merge
* floating-point move
* single-width floating-point/integer type-convert
* widening floating-point/integer type-convert
* narrowing floating-point/integer type-convert

## Vector Reduction Instructions

* single-width integer reduction
* widening integer reduction
* single-width floating-point reduction
* widening floating-point reduction

## Vector Mask Instructions

* mask-register logical
* count population
* find-first-set mask bit
* set-before mask bit
* set-including-first mask bit
* set-only-first mask bit
* iota (把小于该index的所有mask的和存入rd中，可以使用v0 mask)
* element index

## Vector Permutation Instructions

* integer scalar move
* floating-point scalar move
* vector slide
    - slideup
    - slidedown
    - slide1up
    - slide1down
* vector register gather (vd[i] = (vs1[i] >= VLMAX) ? 0 : vs2[vs1[i]])
* vector compress (将v0为1对应的vs2 compress到vd)
* whole vector register move
