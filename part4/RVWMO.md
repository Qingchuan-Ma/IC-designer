# RVWMO

## Memory Model Axioms (内存模型公理)

一个RV程序执行符合RVWMO一致性模型仅当存在一个全局内存顺序符合PPO并且满足load value axiom, the atomicity axiom, and the progress axiom.

Load Value Axiom

每个load i的每个字节都返回由一个store写入到该字节的值，该store中在全局内存顺序中的以下stores是最新的:
1. 写那个字节并且比i在全局内存顺序上更前
2. 写那个字节并且在程序顺序上更前

换句话说就是先从write buffer中找，然后再去内存上找

Atomicity Axiom

如果r和w是由对齐的LR和SC指令在hart h上执行的成对的load和store操作，s是对字节x的store，r返回一个s写的值，那么s必须在全局内存上比w前，并且除了h以外的hart不能向字节x发出一个在全局内存上s和w之间的store操作。

Progress Axiom
在全局内存顺序中，任何内存操作都不能在其他内存操作的无限序列之前进行。
(不知道意味着什么，没懂)

## Preserved Program Order

如果a和b满足程序顺序，并且满足PPO => 满足全局顺序。

内存操作a在保留内存顺序上比内存操作b前（因此在全局内存上也前）是指，a在程序顺序上比b前，a和b都访问常规主存而非IO区，并且以下任何一条成立：

* Overlapping-Address Orderings:
    1. b是一个store操作，并且a和b访问重叠内存地址（相同地址强制Load->Store）
    2. a和b都是loads，x是a和b都读出的字节，在a和b的程序顺序中没有任何对x的store，并且a和b返回x的值由不同的内存操作写出来（也就是说，ab不能由同一个store产生或者ab之间有store(第3点除外)，这样就可以乱序）
    3. a是由一个AMO或者SC指令生成，b是一个load，并且b返回a写的值（也就是说，这种情况下是不能乱序的）
* Explicit Synchronization
    4. 有FENCE指令强制a在b之前
    5. a有一个acquire annotation
    6. b有一个release annotation
    7. a和b都有 RC_sc annotations
    8. a和b是paired
* Syntactic Dependencies
    9. b和a语法地址相关
    10. b和a语法数据相关
    11. b是store，并且b和a有语法控制相关
* Pipeline Dependencies
    12. b是load，并且在a和b在程序顺序之间存在某个store m使得m有地址或者数据依赖于a并且b返回m写的值。
    13. b是store，并且a和b在程序顺序之间存在某条指令m使得m有地址依赖于a

## Syntactic Dependencies 语法相关

寄存器指GPR，一部分或整个CSR。通过csr跟踪语法依赖关系的粒度

当以下任何一条成立时，寄存器r(非x0)是一条指令i的一个源寄存器：

* 在i的opcode中，rs1, rs2 or rs3被设为r
* i是一条CSR指令，并且在i的opcode中，csr被设置成了r，除非i是CSRRW或者CSRRWI并且rd设置成了x0
* r是一个CSR并且是i的一个隐式源寄存器
* r是一个与i的另一个源寄存器同名的CSR

内存指令还进一步指定哪些源寄存器是地址源寄存器，哪些是数据源寄存器。

在以下任何一条成立时，寄存器r(非x0)是一条指令i的一个目标寄存器：

* 在i的opcode中，rd被设为r
* i是一条CSR指令，并且在i的opcode中，csr被设置成了r，除非i是CSRRS或者CSRRC并且rs1被设置成了x0或者i是CSRRSI或者CSRRCI并且uimm[4:0]设置成了0
* r是一个CSR并且是i的一个隐式源寄存器
* r是一个与i的另一个源寄存器同名的CSR

大多数非内存指令都有一个依赖关系，从每个源寄存器到每个目标寄存器

当下面两条之一成立时，指令j有一个语法依赖于指令i via i的目标寄存器s和j的源寄存器r：

* s和r相同，并且在i和j之间，在程序顺序中没有任何指令将r作为目标寄存器（i在j前）
* 程序顺序，在i和j之间存在一条m指令，使得下面所有都成立：
    1. j有一个语法依赖于m via 目标寄存器q和源寄存器r
    2. m有一个语法依赖于i via 目标寄存器s和源寄存器p
    3. m传递从p到q的依赖

i: xxx s rs; (1. s=p) 
m: xxx q p; (2. q=r) (3. rd=q, rs=p)
j: xxx rd r;

假设a和b是两个内存操作，i和j分别生成a和b：

* b语法地址依赖于a当r是j的一个地址源寄存器并且j语法依赖于i via 源寄存器r
* b语法数据依赖于a当b是一个store操作，r是j的数据源寄存器，并且j语法依赖于i via 源寄存器r
* b语法控制依赖于a当程序顺序在i和j之间存在指令m使得m是一个分支或者间接跳转并且m语法依赖于i

*Generally speaking, non-AMO load instructions do not have data source registers, and unconditional non-AMO store instructions do not have destination registers. However, a successful SC instruction is considered to have the register specified in rd as a destination register, and hence it is possible for an instruction to have a syntactic dependency on a successful SC instruction that precedes it in program order.*

## Memory Model Primitive

在RV32GC和RV64GC的指令中，每条对齐的内存指令只会产生一个内存操作，只有两个例外。首先，一条不成功的SC指令不会引起任何内存操作。第二，如果XLEN<64, FLD和FSD指令可能各自引起多个内存操作，如13.3节所述。一个对齐的AMO产生了一个单一的内存操作，它同时是一个加载操作和一个存储操作。

如果在程序顺序上，LR指令在SC指令之前，并且中间没有其他LR或SC指令，则LR指令和SC指令被称为配对;相应的内存操作也被认为是成对的(除了在SC失败的情况下，那里没有存储操作生成)。决定SC是否必须成功、可能成功或必须失败的完整条件列表在章节9.2中定义。

acquire-RCpc，acquire-RCsc，release-RCpc， release-RCsc，这些定义由PPO 5~7 定义
带ac的AMO或者LR指令有acquire-RCsc注释
带rl的AMO或者SC指令有release-RCsc注释
带aq和rl的AMO，LR或者SC指令两个都有

RCpc: release consistency with processor consistent synchronization operations
RCsc: release consistency with sequentially consistent synchronization operations 

*Kourosh Gharachorloo, Daniel Lenoski, James Laudon, Phillip Gibbons, Anoop Gupta, and John Hennessy. Memory consistency and event ordering in scalable shared-memory multiprocessors. In In Proceedings of the 17th Annual International Symposium on Computer architecture, pages 15–26, 1990*

“RCpc”目前仅在RVTSO隐式地分配给每个内存访问时使用。此外，尽管ISA目前不包含native load-acquire or store-release指令，也不包含其RCpc变体，但RVWMO模型本身被设计为向前兼容，在未来的扩展中，可能会将上述任何一种或全部添加到ISA中

# RVTSO

* 所有load操作的行为就像它们有一个acquire-RCpc注释一样
* 所有store操作的行为就像它们有一个release-RCpc注释一样
* 所有的AMOs的行为就像他们有acquire-RCsc和release-RCsc

这些规则使得除了4-7之外的所有PPO规则都是多余的。它们还使没有同时设置PW和SR的任何非i/O fence变得多余。最后，他们还暗示没有内存操作将被reorder past an AMO in either direction，就是不允许越过或提前于AMO操作。

Load Value Axiom还是满足，说明可以bypass


===================================================

1. SC什么时候能够写成功：
2. Processor Consistency是什么
3. RCpc和RCsc区别是什么

