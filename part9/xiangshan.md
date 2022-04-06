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
* s2: 获得读结果，选择并写回。 VIPT

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

虚地址前传的数据通路：load pipeline

* s0: 生成mask和虚拟地址
* s1: 虚拟地址和mask送到buffer和queue中查询
* s2: 产生结果，和dcache读出的数据做合并

虚地址前传的控制通路：load pipeline

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
        + 只访问前两级页表，1G和2M，如果访问到叶子节点（大页或空页），则返回给L1 TLB；否则送往miss queue，访问4K页表
    - miss queue
        + 接受page cache和page walker的请求
        + 同时接受N个请求，因为对页表的访问集中于最后一级页表。
        + 承担miss的缓冲，等到miss消失，到重新访问page cache
        + 承担访问最后一级页表
    - prefetcher
        + 由page cache的结果引起预取产生，不会返回给L1 TLB。
        + 下一行预取算法 
* repeater: L1到L2物理距离较长，加拍。
* filter：在repeater的基础上强化，接受DTLB请求，发送给L2。过滤重复请求，项数一定程度决定L2的并行度
* PMP
* pma

* feedbackSlow包括的信息
    - rsIdx，重发指令位置
    - sourceType，重发原因
    - sqIdx，前传的时候load发现store的数据没准备好


### load pipeline

* s0：
    - 指令和操作数从rs中读出
    - 加法器将操作数和立即数相加，计算虚拟地址
    - 虚拟地址送入TLB进行查询
    - 虚拟地址送入数据缓存进行tag查询？还是VIPT
* s1:
    - TLB产生物理地址
    - 完成快速异常检查
    - 物理地址送进数据缓存进行data查询
    - 虚拟地址和物理地址送入store buffer和queue进行store到load的前传
    - 根据L1 dcache返回的命中向量和初步异常判断结果，产生wakeup信号
    - 如果这一级出现了导致重发的事件，需要通知保留站这条指令重发 (feedbackFast)，一旦fast触发，后边流水不在进行
* s2:
    - 异常检查
    - 根据以及缓存以及前传返回结果做数据选择
    - 根据load要求，对分会结果做剪裁
    - 更新load queue对应项状态
        + datavalid
        + writebacked
        + miss
        + pending
        + released
    - 整数结果写到cdb common data bus
    - 浮点结果写到浮点模块
* s3:
    - dcache的反馈结果，更新load queue中对应项的状态
        + 重发可能导致的miss, datavalid为false，标志会重发，而不是在load queue等待refill
    - 根据 dcache 的反馈结果, 反馈到保留站, 通知保留站这条指令是否需要重发 (feedbackSlow)

load miss 的处理：

* s1，根据tag比较结果，可以知道是否miss，miss，则禁用当前指令的提前唤醒
* s2，如果miss，load不会写回结果，更新load queue状态，这条miss的load指令等refill，分配mshr
* s3，分配结果这时才能知道，因此再次更新load queue状态，如果分配失败，则请求保留站重发指令
* 分配了mshr的话，后续再load queue中侦听refill结果

保留站重发指令的情况：

* TLB miss:
    - feedbackSlow。 TLB的refill需要时间，因此需要延迟
* bank conflict:
    - feedbackFast。指两条load流水线之间bank冲突，以及load和store的cacheline的冲突，不设置延迟可以立即重发
* MSHR分配失败：
    - feedbackSlow。不设置延迟可以立即重发
* Store Data invalid：
    - feedbackSlow。load前传时的store数据没准备好，会随着feedbackSlow一起把sqIdx反馈给保留站，到时候有了就可以重发，所以延迟不固定

预取指令：这里是指binding-prefetch，软件预取指令：

* 再发现miss时会向dcache的missqueue发出请求，出发对下层的cache访问。
* 期间屏蔽异常，不会重发

提前唤醒：

* 提前唤醒的指令必须要能正常的执行，意思是可以提前唤醒依赖于这个load的后续指令。
* s1的阶段发出快速唤醒信号。

前传失败：

* 再s2的时候，附上mark，repalyInst，需要从取指重发。在ROB的尾部会触发重定向？

### load queue

* 80项循环队列
* 每周期最多
    - 从dispatch接受4条指令
    - 从load接受2条指令结果，更新状态
    - 从dcache接受refill结果，更新load queue中等待此次refill的指令状态
    - 写回2条miss的load指令，和正常访存流水线征用2个写回端口
* entry 内容
    - 物理地址
    - 虚拟地址(debug用作调试的)
    - 重填数据位
    - 状态位
        + allocated: 表明该项被dispatch分配
        + datavalid：表明load已经拿到所需数据
        + writebacked: load已经写回寄存器堆，通知ROB和RS
        + miss：表明dcache未命中，等待refill
        + pending: 表明mmio地址，执行被推迟，等待指令成为ROB最后一条指令
        + release: 表明cacheline被dcache release
        + error: 表明执行过程中检测到错误

load refill:

* 一次refill会把数据传递到所有等待这一cacheline的load queue项
* 如果之前进行了前传，则会在refill的时候合并前传的结果
* 每个周期，load queue会分别从奇偶两列中选出最老的已经refill但没写回的指令，在下一个周期将其通过写回端口写回。
* 会和load流水线抢占写回端口。load流水线优先级更高


load commit:

* load取得结果，写回rob和rf。每周期最多两条指令被写回
* load queue写回包括
    - miss之后的refill写回
    - mmio load写回
* rob在指令提交后，会产生lcomimt信号，通知load queue这些指令已经成功提交
* load queue会把allocated 状态更新为false表示已经完成，同时更新队尾指针

redirect:

* 重定向到load queue之后两排内更新load queue状态
    - 1： 根据robidx找到错误路径指令，allocated设置为false
    - 2：格努就上一排的结果，统计多少指令被取消，更新enqPtr

违例检查：这是内存模型的要求

* 实现办法是使用重定向，从取指开始重新执行后续的指令

### store pipeline

* robidx和sqidx相同，分别对应一条store指令的数据和地址
* store addr流水：
    - 前两个流水负责store的控制信息和地址传给store queue，
    - 后两个流水及负责等待访存依赖检查
    - s0: 
        + 计算虚拟地址，虚拟地址送入TLB
    - s1: 
        + TLB产生物理地址
        + 快读检查异常
        + 访存依赖检查
        + 物理地址进入store queue
    - s2:
        + 访存依赖检查
        + 完成全部异常检查和PMA查询，根据结果更新store queue
    - s3:
        + 完成访存依赖检查
        + 通知ROB可以commit
* store data流水：
    - 保留站提供data之后，
    - 立刻将data 写入store queue对应的项


TLB miss的处理：

* 和load流水线一样，使用rsFeedback端口反馈是否要重发
  
PMA和异常检查：

* mmio等检查结果在data更新到store queue一周期之后才全部完成，此时store流水线会查询最终结果写到store queue

### store queue

* 64项
* 每周期至多
    - 从dispatch处接受4条指令
    - 从 store addr 流水线接收 2 条指令的地址和控制信息
    - 从 store data 流水线接收 2 条指令的数据
    - 将 2 条指令的数据写入 committed store buffer
    - 为 2 条load流水线提供 store 到 load 前递结果
* entry 内容
    - 物理地址
    - 虚拟地址
    - 数据
    - 数据有效mask
    - 状态
        + allocated
        + addrvalid
        + datavalid
        + committed
        + mmio
        + pending: mmio，执行被推迟等待成为ROB最后一条指令
* store addr
    - s1在没有TLV miss等问题更新 addrvalid
    - s2更新mmio和pending，PMA检查是否处于MMIO时序紧张
* store data
    - 设置datavalid状态

store commit:

* 同load commite，产生scommit信号，通知store queue这些store已经成功提交，committed更新为true

sbuffer:

* 已经提交的store指令不会取消，按顺序从queue读到buffer。
* 一周期读数据，下一周期写到sbuffer，只有真的写到sbuffer之前，store queue一直都是有效

flush:

* sbuffer drain，在这个过程中store queue会不断把已经提交的store指令的数据写到sbuffer中

### sbuffer

* 16项以cacheline为单位的数据
* 每周期最多
    - 接受2条store queue写入的store指令
    - 向dcache写一个cacheline
* 使用PLRU进行替换写入cache
* entry 内容
    - 虚实地址
    - 数据
    - 数据有效掩码
    - 状态
        + state_valid：表明该项有效
        + state_inflight： 表示已经发出写请求
        + w_timeout： 等待重发计时器timout，需要重发写请求
        + w_sameblock_inflight：表明有相同的block在写如dcache，这个时候的指令不会被替换

sbuffer项dcache的写：

* 第一拍选，同时更改装填
* 第二拍写入请求发到dcache
* 只有等dcache报告写入成功了之后，sbuffer才会标记为无效

向dcache写入项的选择，优先级高到低

* dcache请求重发到了等待时间了
* sbuffer刷新
* 一个sbuffer存在的时间超过阈值
* sbuffer满了，部分项选择换出
    - PRLU，在queue写到sbuffer或者queue和某项merge，PRLU会更新

flush:

* 前面讲过了

### 原子指令

状态机：

* 空闲
* TLB
* PM：地址异常检查
* sbuffer flush请求（drain store buffer和store queue？）
* 等待请求完成
* 向dcache发原子操作
* 等dcache返回原子操作结果
* 写回结果
  
store指令，只有addr和data都准备好，才会开始

lr执行之后：

* 阻塞对当前核当前 cacheline 的所有 probe 请求并阻塞后续 lr 的执行, 持续一段时间 (lrscCycles - LRSCBackOff cycles)
    - 这个时候可以完成constrained LR/SC loop
    - 当前hart可以不被打扰执行一个成功的SC sequential consistency
* 持续阻塞后续 lr 的执行, 但允许对这一 cacheline 的 probe 请求执行, 持续一段时间 (LRSCBackOff cycles)
* 恢复到正常状态


异常处理：

直接从pm到finish，不触发flush，不产生实际访存请求

### D cache

* 128KB, 8-way组相联，PLRU替换策略，SECDED校验
* 和l2 cache配合一起处理aliasing问题
* 内部模块
    - Load pipeline
    - Main pipeline
    - Refill pipeline
    - Atomics Unit
    - Miss queue
    - Probe Queue, 处理一致性请求
    - Writeback Queue

#### Load pipeline

* 如果miss，进入miss queue，然后refill，如果需要替换，则用writeback Queue来将替换块写回
* 和load unit的load各个流水级意义对应，逻辑上被视为同一个流水线，非阻塞
* s0:
    - v index查tag
* s1:
    - 获得tag查询结果，把hit和miss送入下一级寄存器
    - 获取meta查询结果
    - 检查tag error
    - 物理地址查询data
    - 根据PLRU，选出替换的way（在miss的时候需要）
    - 检查bank冲突
    - 生成fastwakeup和load pipeline一致
* s2:
    - 更新PLRU
    - 获取data结果
    - 如果miss，尝试分配MSHR，如果queue满则告诉s3重新wakeup
    - 检查data error
    - 

#### Main pipeline

Store buffer重发原则：

* 如果Dcache hit
    - 在main pipeline中完成对Dcache的写入
* 如果miss且miss queue满
    - 一段时间后重发
* 如果miss且miss queue成功接受
    - miss queue继续执行后续操作
    - 完成后通知sbuffer，并通过refill pipeline更新dcache
* 如果有被替换的块，在writeback queue写回
    - 只有真的替换的块到了Dcache之后，被替换出去的才会被writeback queue往下写，valid?

* 该流水为dcache主流水线，用于处理store, probe,原子操作和替换操作
* 负责所有需要征用writeback queue向下层cache发起请求/写回的指令执行
* s0: 
    - 仲裁优先级最高的（以下优先级从高到低）
        + probe_req
        + replace_req
        + store_req
        + atomic_req
    - 根据请求信息判断资源是否准备好
    - tag,meta读请求
* s1: 
    - 获得tag，meta读请求的结果
    - 进行tag匹配检查，判断是否命中，命中会被寄存，下一级就知道需要访问miss queue了
    - 如果需要替换，获得PLRU提供的替换选择结果（miss的时候这个有用）
    - 根据读出的meta进行权限检查
    - 提前判断是否需要执行 miss queue 访问
* s2:
    - 获得读data的结果，与要写入的数据拼合
    - 如果miss，尝试写入miss queue
* s3:
    - 更新meta,data,tag
    - 如果需要项下层写回，生成write_back queue访问请求，尝试写writeback queue
    - 在relase操作生效更新meta时，向load queue给出release信号进行违例判断(应该还是RVWMO的某个要求？)
    - 对于原子操作的支持
        + AMO在这一级两拍，第一拍做运算，再把结果写回dcache
        + LR/SR指令会检查其reservation set queue



#### Refill pipeline

* 进入miss queue之前会选出替换way，因此在写回的时候，不需要再明确替换路，所以只需要一个周期就可以把L2的数据填入DCache，不需要再访问一边DCache
* Refill和Main都会对DCache写，因此会有冲突，保证Main Pipe前后读写数据的一致性，即读写不会被Refill的写大端，因此某些情况会被阻塞
    - Refill的请求和s1有set冲突，为了放送时序要求，值根据set阻塞即可
    - Refill的请求和s2/s2有冲突，并且way一样（根据way_en）
    - 说明Main对Dcache先写，但是读操作呢？
        + 比如load的s1 tag匹配，但是同时refill把该way覆盖掉了，load的s2又获取了数据，不就出问题了么


#### Miss queue

* 16项entry，接受load\store\atomic的请求，从L2 refill，并把load 数据返回给load queue
* load具体操作/store类似：
    - 分配空entry，记录相关信息
    - 根据way_en所在块是否有效，判断是否需要替换，如果需要则给main发replace_req
    - 同时向L2发Acquire
        + 如果是整个block的覆盖写，则发AcquirePerm(不需要读合并的操作了)
        + 否则发给AcquireBlock
    - 等L2 Grant或GrantData（后者带数据，前者可能用于store）
    - 收到GrantData每个beat把数据转发给Load queue
    - 在收到 Grant / GrantData 第一个 beat 后向 L2 返回 GrantAck;
    - 在收到 Grant / GrantData 最后一个 beat, 并且 replace 请求已经完成后, 向 Refill Pipe 发送 refill 请求, 并等待应答, 完成数据回填;
        + store miss 在最终完成回填后要向 Store Buffer 返回应答, 表示 store 已完成.
    - 释放entry
* 原子指令：
    - 分配空entry，记录相关信息
    - 向L2发送AcquireBlock指令
    - L2返回GrantData
* 也支持一定程度的合并请求
* 合并条件：
    - 块地址相同时：
        + A的acquire还没握手，且A是load；B是load或store
        + A的acquire已经法出去了，但没有收到Grant，或者收到了Grant但是没有转发给Load queue，且A是load或store；B是load
* 拒绝条件：
    - 块地址相相同，但是不满足合并条件
    - 块地址不同，但是处于同一个slot（即同一个set和way）
        + 假设两个位于同一个 set 但不同 tag 的 load 请求, 先后在 Load Pipeline 上 miss 了, 但是两个 load 决定替换相同的 way, 并分别分配了一项 Miss Entry, 最终导致后 refill 的块把先 refill 的块覆盖掉了
* 空项分配条件
    - 在没有 Miss Entry 想要合并或者拒绝新的 miss 请求的情况下
        + 如果 Miss Queue 有空项, 分配新的 Miss Entry;
        + 如果 Miss Queue 已满, 拒绝新的 miss 请求, 该请求会在一定时间后 replay.


#### probe queue

* 16项，接受来自L2的probe请求，发给main pipe，修改probe的权限
* 处理流程：
    - 分配一项空entry
    - 向main发probe请求
    - 等待main返回应答
    - 释放entry

#### Alias Problem

* L2 Cache 的目录会维护每一个物理块对应的别名位 (即虚地址索引超出页偏移的部分)
* 保证某一个物理地址在 L1 Cache 中只有一种别名位. 当 L1 Cache 在某个物理地址上想要获取不同的别名位, 即不同的 virtual index 时, L2 Cache 会将另一个 virtual index 对应的 cache 块 probe 下来
* probe的时候会有B通道的data域传递2bit的额外index位，用这个和页偏移concat，得到真的index

#### Writeback Queue

* 18项写回队列。release，或者对probe请求作出应答(ProbeAck)，支持Release和ProbeAck之间相互合并减少请求数目
* 队列满不接受任何请求，不满则接受，可能合并也可能分配空项
* 状态位
    - s_invalid: 空项
    - s_sleep: 准备release，但暂时sleep等待refill唤醒，说明refill了之后他就可以被替换走了。
        + 处于sleep的块需要和dcache中的同步
    - s_release_req: 正在发送release或probe ack
    - s_release_resp: 等待release ack请求
* 不考虑请求合并
    - ProbeAck: s_invalid -> s_release_req -> s_invalid
    - Release: s_invalid -> s_sleep -> s_release_req -> s_release_resp -> s_invalid
* 考虑合并
    - release合并ProbeAck: release的过程中可能被动放弃
        + s_sleep阶段直接release变probeack
        + s_release_req
            * release还没握手，直接变
            * release处理完再处理probeAck
    - probeAck合并release
* miss queue的请求如果发现和writeback中某一项地址相同，则miss请求会被阻塞

#### 自定义L1操作

寄存器分类：

* 指令寄存器,CSR_CACHE_OP
* 指令状态寄存器,CSR_OP_FINISH
* 一级缓存控制寄存器,配置参数

执行流程：

* 使用csr指令写入参数配置寄存器，清空OP_FINISH寄存器
* 向CSR_CACHE_OP写入指令码
* 直到CSR_OP_FINISH==1
* 使用csr指令读结果寄存器获得结果

### 访存依赖预测

* Load Wait Table
* Store Sets

如果预测到一条 load 指令可能违例, 则这条 load 指令需要在保留站中等待到之前的 store 都发射之后才能发射.

SSIT 和 LFST。第一次听说，得去看下原始论文

### TileLink

*满足cache一致性的总线协议*

一致性请求分类：
* Acquire 获取权限
* Probe 被动释放权限
    - 一般来自L2 Cache的Probe请求，
    - 在主流水线中修改被Probe的块的权限
    - 返回应答，同时写回Dirty数据
* Relase 主动释放权限

