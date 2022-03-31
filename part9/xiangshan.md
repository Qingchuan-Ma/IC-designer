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