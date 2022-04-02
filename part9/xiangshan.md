## Introduction

* 雁西湖架构
	- RV64GC
* 南湖架构
	- RV64GCBK
* 昆明湖


以下都是指西湖架构的结构

## 整体结构

前端：

* 分支预测
* 取指
* 指令Buffer

后端：

* 译码
* 重命名
* ROB
* reservation
* rf
* alu

访问子系统：

* 2 load pipeline
* 2 store addr pipeline
* 2 store data pipeline
* load queue
* store queue
* store buffer

cache:

* harvard I/D
* L2/L3(HuanCun)
* TLB
* prefetch

## 前端

### branch prediction unit(BPU)

BPU包括覆盖预测逻辑和流水线握手逻辑，以及分支历史管理。

如果一个预测流水级存在有效预测结果，或者后续预测流水级产生不同的预测结果，则和FTQ进行握手，可能是冲刷，也可能是入队

* 分支历史管理 gbh
	- 每次新的预测就会更新全局历史
	- 预测过程的时候，如果后边的流水推测和前边的结果不一样，会冲刷流水重新预测（怎么重新预测，没懂）
	- 预测结束，会把预测的时候用的全局历史存到FTQ，预测错恢复时读出送回BPU

预测宽度 PW：

每次预测提供给取指单元的最大指令流宽度，南湖架构中和取值宽度相同(32 byte)，未命中PC=PC+PW

覆盖预测器（overriding predictor）：

先获得一个预测结果先用着，如果延迟大的、相对更准确的如果和前边的不同，则冲刷流水线

分支预测器：

* next line predictor(NLP): uBTB
	- 无空泡的预测
* APD:
	- Fetch Target Buffer (2 cycle)
	- TAGE (2 cycle)
		+ TAGE-SC (3 cycle)
	- RAS (2 cycle)
	- IT-TAGE (3 cycle)


NLP(uBTB)：

* index = pc xor gbh
* 预测块地址 nextAddr
* 是否跳转 taken
* 偏移 cfioffset
* 指令信息
	- 是否在条件分支跳转 takenOnBr
	- 块内包含分支指令数目 brNumOH
* aliasing problem
	- 在训练的时候如果预测块内没有分支，则不写uBTB
	- pc直接索引一个存储结构，存放pc是否被写入过该表
	- 如果没有写过，则PC=PC+PW
	- 这样就不再需要tag匹配

FTB：

* 提供预测块内分支指令的信息
* 提供预测块的结束地址 end
* index = start，start 准则：
	- 是上一个预测块的end
	- 或者BPU外部重定向的目标地址(预测错误？
* FTB项内最多两条分支指令，第一条一定是条件分支
* end 下面三种条件之一
	- end = start + PW
	- end = start开始预测宽度内第三条分支指令PC
	- end = 一条无条件跳转分支的下一条指令的PC，并满足在start开始的预测范围内（意思是先把end设置成无条件跳转的PC+4）
* end只存低位
* 总是跳转位，第一次遇到时置1
	- 为1，总是跳转，不用其结果训练条件分支方向预测器
	- 如果有一次不跳转，置0，之后方向由条件分支方向预测器预测
	- 这个应该就是前边说的BPU会忽略那些从未跳转的条件分支指令，这些指令不会包括在分支历史中

TAGE-SC：


* 折叠亦或的方式不一样，hash func不一样
* 一次预测两个，分支历史也是相同的，存放两个预测结果。
* 时序考虑，alt_pre一直是base_pre
* use策略：
	- 失败pred--，饱和了的时候，会把use清零
	- 和TAGE文章里的还不太一样？
* SC统计矫正器（perceptron） 
	- 如果有大概率误判会反转结果

ITTAGE：

* jalr无条件跳转，间接跳转
* FTB记录jalr最近一次跳转，对那些固定的跳转地址有效
* RAS处理函数调用
* 不符合上边两者的交给ITTAGE预测
* 在TAGE表项基础上加入所预测的跳转地址
* FTB一次取两个分支，只存最多一条间接跳转，所以ITTAGE每周期最多预测一条间接跳转指令的目标地址

RAS：

* call push
* ret pop
* entry: address, count
	- 当重复压同一个地址（递归），count++
	- 反之count--
* 每次预测，栈顶项和栈指针都会存如FTQ的存储结构，用于误判恢复

预测器的训练

* 训练内容
	- 自身预测信息
		+ 预测后打包传入FTQ存储，用于恢复
	- 预测块中指令的译码和执行结果，FTQ读回

### fetch target queue(FTQ)

* 暂存BPU预测的取指目标，根据这些目标给IFU发送取值请求。

* 暂存BPU预测器的预测信息，用于训练，因此需要维护指令从预测到提交的完整生命周期。

* mem后缀是寄存器堆
	- pc相关信息
	- pd相关信息
	- ftb相关信息，训练新的FTB项
* sram后缀是sram
	- RAS和分支历史
	- 其余BPU预测信息

* 指令以预测块为单位，预测后送入FTQ，只有指令所在的预测块中所有指令全部在后端提交完成，FTQ才会释放预测块对应项，这个时间段内：
	- bpuPtr指针加1，写入预测信息；如果是覆盖预测逻辑，则恢复bpuPtr和ifuPtr
	- FTQ向IFU发出取值请求，ifuPtr指针加1，等待预译码信息
	- 预译码信息可以知道预测是否正确，并不是全部知道
	
这个得看代码，bpu原来是n

* bpuPtr，纵向的指针，指向n+1，预测块内的指令进入生存空间
* ifuPtr，应该是横向的指针
* ifuWbPtr，
* commPtr，指向n+1后，预测块内的指令完成生存周期

FTQ其他功能：

* 指令预取，如果BPU提供了指令请求，但是IFU没准备好，可以转发给预取逻辑，先进行预取

### instruction fetch unit

* 4级流水
* 分支预测-icache缓存解耦
* FTQ的取指令请求包括32 B的其实地址和下一个跳转目标地址
* IF1简单计算？
* IF2

### instruction cache

* nset 256组
* nWays 8路
* nTLBEntries， ITLB项数，32普通页+9大页
* tagECC, meta SRAM的校验方式，奇偶校验
* dataECC, data SRAM的校验方式，奇偶校验
* replacer，替换策略，随机替换，PLRU  cache replacement polices书里(part4 cache)没提到过这个，得去了解一下，伪LRU？
* hasPrefetch, 预取开关
* nPrefetchEntries预取的entry数量，同时可支持的最大cache line数量，默认是4

MainPipe 三个流水：

* s0阶段FTQ发过来两个cache line取指令，送入ITLB和Cache
* s1阶段，返回一个set的全部数据，ITLB返回物理地址。拿物理地址和cache的tag对比。给出hit或miss结果，replacer也会选出要替换的line
* s2阶段，hit直接返回数据给IFU，miss则暂停流水线，将miss请求发给missunit，等missunit填充完并返回数据之后再将数据返回给IFU。意思是不接受non-blocking的miss，这个阶段会把ITLB翻译得到的物理地址发给PMP模块进行权限访问查询，如果权限错误，会触发exception（Instruction Access Fault）

* 地址翻译是[VIPT](../part4/虚拟内存.md)
* 由于两个cacheline，因此ITLB端口应该要有多个，s0周期内需要返回是否命中，命中则会在下一排返回对应的物理地址，不命中会阻塞Mainpipe

cache miss:

* MissUnit向下游L2发送Tilelink Aquire，
    - 等MissUnit收到Grant
        + 如果要替换cacheline，会发送Release请求给ReplacePipe，ReplacePipe重新读一遍数据，然后给L2发送Release请求（PutM?）。
* 最后MissUNIT重新填SRAM
* 填完了之后MainPipe将数据返回给IFU

exception:

* Instruction Page Fault和Access Fault，直接报告给IFU，数据认为无效
  
存储部分：

* meta: tag、state
* data: value
* 校验错误报总线错误产生中断
* 奇偶bank，虚拟地址（虚拟地址相邻的两个cahce line的index是一样的，但是tag不一样。VIPT）空间中相邻的两个cache line分别划分到不同的bank实现一次两个ache line的读取
  
一致性：

* tilelink本身是怎么保证一致性的？
* 额外的流水ReplacePipe处理tilelink的请求
    - r0阶段，接受Probe请求和Release请求，请求包括虚拟地址和物理地址，因此不用做翻译，同时还会发起对Cache的读
    - r1阶段，tag匹配，产生hit和miss，仅对Probe有效，release肯定再cache里
    - r2阶段
        + hit时，prob请求被invalid，同时把请求发给ReleaseUnit向L2发送ProbeResponse请求，cacheline权限变为toN
        + miss时，prob请求不给release，发送给releaseUnit向L2报告权限转变为NtoN，
        + release请求再r2啥都没做，被发送到ReleaseUnit向L2发送ReleaseData.
	- r3阶段，replace pipe向missunit报告被替换出去的块已经往下release了，同时missunit可以进行重填

指令预取：

* [FDIP](../part6/ilp.md)，分支预测指令预取 Iprefetch
	- PTQ，的指针位置处于预取指针和取指令指针中间，取当前指令packet的目标地址发给预取器
    	+ 如果是跳转则是跳转地址
    	+ 不是则是顺序的下一个指令packet的目标地址
  	- 预取器完成翻译访问meta sram，如果已经在指令缓存，预取被取消。如果不在则在prefetchentry申请一项，向L2发送Hint请求，对应的line预取到L2
  	- 预取器会记录已经发送的预取地址，任何请求申请prefectentry之前都会查这个记录，如果预取请求相同也会取消
  

### decode unit

* 6条/cycle送入译码单元译码
* 指令融合，fusiondecoder模块，连续两条32it的指令完成指令融合


## 后端

* CtrlBlock
    - 译码/重命名/分派宽度=6
    - 发射前读寄存器堆? 那为什么还需要用显示重命名
* IntBlock
    - 192个物理寄存器
    - 4个ALU 2个乘除 1个CSR/JMP
* FloatBlock
    - 192个物理寄存器
    - 4个FMAC 2个FMISC
* MemBlock
    - 2 Load 2个Store(store和c910一样地址和数据分开Pipe)
  

### register renaming

和BOOM的类似，具有三个模块，busyTable换成了RefTable

* RenameTable
	- 同步读：1周期的延迟，需要addr和hold
		+ hold为1，读数据保持不变
		+ 为0，真正读，读数据根最新的地址更新
	- 写接口：1周期的延迟
		+ 根据重命名的信息和ROB回滚的信息更新表
* FreeList
	- 160个空闲物理寄存器号
	- 重命名的时候，会给出RenameWidth个空闲号
	- 物理寄存器释放时，每一拍可以进CommitWidth个物理寄存器号
	- 多个逻辑寄存器映射到同一个物理寄存器
	- 允许多次引用，多次引用次数记录在RefTable
		+ 零延迟mv
		+ 因此理论最多空闲有物理寄存器总数-1个
		+ 香山里freelist大小和物理寄存器相同
	- 如果不存在引用计数功能
		+ StdFreeList
		+ 回滚只需要出队指针回滚
	- 如果存在引用技术功能
		+ MEFreeList
		+ 回滚物理寄存器数量和FreeList释放数量并不一定相同
		+ 需要维护几个写端口，方便通过引用计数机制维护实际释放的物理寄存器
* RefTable
	- 记录每个物理寄存器被引用的次数
	- allocate: 增加引用次数
	- deallocate：减少引用次数
	- freeRegs：释放物理寄存器到FreeList
	- 重命名对每个物理寄存器的分配和释放都会告诉RefCnt。当一个物理寄存器引用变为0，则释放到FreeList等待下次分配。释放会延后两派

具体操作：

* pdest来源于FreeList的分配结果。
* 如果同一拍需要重命名多个指令，则需要旁路考虑指令依赖
	- 例如i使用j的结果，那么i的psrc应该为j的pdest
* LUI和LOAD指令对优化
	- 直接把load的基地址换成了lui的立即数

### dispatch

* Dispath
	- 指令分类发送至定点、浮点和访存三类派遣队列
	- BusyTable置位
	- 当且仅当所有资源都充足
		+ queue not empty
		+ ROB not full
	- 定点指令可能会因为浮点派遣队列满而被阻塞
* Dispatch Queue
	- 分支预测导致的可能性
* Dispatch2Rs
	- 根据不同的指令类型、保留站可接受的指令类型，做路由，发送到对应的保留站



### ROB

* 派遣：
    - 分配ROB表项，保存信息 RobCommitInfo。包括重命名信息、类型、ftq指针。ROB入队宽度与重命名宽度一致
    - valid,writebacked,interrupt_safe可能会被更新。MMIO信息不回传给ROB
* 写回：
    - ROB对应的运算操作已经完成，ROB将writebacked置1
* 提交：
    - 尽量多的按顺序提交，有异常则会被阻塞，向外发出异常信息
* 取消与回滚:
    - 分支预测、仿存违例。可能需要重刷。取消的指令会回滚


### issue

* 主要包括的操作有：入队，选择，读数据，出队
* 入队：
    - 更新status payload
* 选择：
    - 选择可以唤醒的指令，AGE算法（是什么），选择issuewidth+1条指令，并在下一周期确定可供发射的指令
* 读数据：
	- dataarray 异步读，会在RS中存储值
* 出队：
    - 最后一级流水，被T0选择之后， T1读数据，T2握手出队
* 需要监听写回数据和信号，实时捕获数据
* 浮点乘加指令发射优化
    - 3个rs拆成两个指令，只要有一个成功运行，第二个操作数就绪就可以再次发生，完成全部运算

### EXU

整数部分：

* ALU： 加减、逻辑、分支、部分位扩展
* MUL：3级wallace乘法器
* DIV: SRT16除法器，见part1，每周期算4位，前后处理各2拍
* MISC: CSR/JUMP

浮点：

* Two-path Floating-point Adder
* Floating-point Multiplier
* Cascade FMA
* Float -> Int Converter
* Int -> Float Converter
* Float -> Float Converter
* Floating-point Divider and SQRT Unit

其中FMA操作延迟为5拍，其余运算均为3拍延迟。

DIV-SQRT, SRT16算法，开方用SRT4算法。两者共用预处理和后处理。

## 访存子系统（memblock）

* 2 load
* 2 store (sta, std)
* store主要是在commit之前暂存，并且bypass数据给load
* store queue commit之后会进入store buffer
    - 以cache line为单位对store的写合并，之后一并写到L1
    - L1 cache有两个64bit的读，和一个line宽度的写，和一个充填端口(和L2之间的总线宽度决定宽度)
* MMU地址转换，TLB，L2TLB，repeater, pmp pma
  

### 乱序访存机制


load hit:

* s0: 计算地址，读TLB，读dcache tag
* s1: 读dcache data
* s2: 获得读结果，选择并写回。 PIPT

sta:

* s0: 计算地址，读TLB
* s1: addr和其他控制信息写入store queue，违例检查
* s2: 违例检查
* s3: 为例检查，允许store指令提交


Replay from reservation

重发load：

* TLB miss
* L1 DCache MSHR full(dcache是接受non-bloking miss的)
* DCache bank conflict
* data invalid，store bypass的时候发现地址是对的，然是数据没准备好

重发sta:什么情况下要重发，具体需要查看代码

store to load forward(STLF)：从commited store buffer和store queue里查找地址前传数据

虚地址前传，实地址检查，TLB就可以移除数据通路，优化时序

sqidx: store queue index

store queue和commited store buffer中也需要存虚地址？

虚地址前传的数据通路：

* s0: 生成mask和虚拟地址
* s1: 虚拟地址和mask送到buffer和queue中查询
* s2: 产生结果，和dcache读出的数据做合并

虚地址前传的控制通路：

* s0: 虚拟地址送入TLB进行转换
* s1: TLB返回物理地址，物理地址和mask送入queue和buffer，进行查询（只做地址匹配）
* s2: 虚地址匹配结果和实地址匹配结果对比？对比数据还是对比queue idx？需要看代码
* 为什么会将引发错误的虚地址从 store queue 和 committed store buffer 中排除出去

有点怪，之后再细看

### mmu

* SV39, 三级页表
* Repeater 是一级 TLB 到 L2 TLB 的请求缓冲
* PMP 和 PMA 需要对所有的物理地址访问进行权限检查
  


以下都是默认设置：

* ITLB：32普通页4大页。全相连，PLRU
* DTLB：128普通页直接相联，8项全相联负责所有大小页(PLRU)
    - DTLB 的直接相联项是全相联项的 victim cache，就是替换出来的放在这个里边
* L2TLB：共享TLB
    - Page Cache
        + hit则直接返回，miss，则给page walker或queue
        + 三级页表项都缓存了，支持ecc，如果ecc错了，刷新此项，重新进行page walker
    - Page Walker
        + 
    - miss queue
    - prefetcher
* repeater: L1到L2物理距离较长，加拍。
* filter：在repeater的基础上强化，接受DTLB请求，发送给L2。过滤重复请求，项数一定程度决定L2的并行度
* PMP
* pma