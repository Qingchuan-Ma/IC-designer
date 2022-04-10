
把load依赖的store set称为load's store set
目标就是发现和使用这个store set来预测load能安全执行的最早时间
这里不用bypass来解决memory-order violation的一个原因就是有的时候store的addr还没计算出来，不能用于bypass

goals:

* 预测load指令执行是否会导致memory-order violation
* delay执行这些loads

metric:

* 预测器导致的memory-order的violation
* 预测器导致的false dependencies

首先让所有load都推测执行，如果检测到一个当store执行时检测memory-order violation，那么把这个store加载load's store set

### Baseline

* alpha 21264但是double cache和issue width
* 128 instrution queue entries.

![](../assets/mem_dep_pre1.png)

assume baseline检测violation的方式如下:

* 通过表存放所有在执行的loads指令的地址
* 每次store都会check该表看是否有推测的load依赖于我
* 如果有，则需要恢复到该loads的PC位置，重新取该load指令
* 假设N周期恢复，也就是下边的memory trap penalty
* benchmarks是 SPEC95

### Motivation

#### No speculation

load 不会推测执行：

* violation无了
* 但是false dependencies很多
    - loads必须等store计算完addr和data才能发射；本文不考虑data和addr分开流水的情况

#### Naive speculation

只要寄存器依赖已经解决，load推测执行；

* violation造成trap
    - memory trap penalty: 取决于程序中这个store执行所需的时间
* 有的设计可以不用flush 推测load之后的所有指令，只需要flush跟load相关的指令即可。（这个可以通过保留站的机制实现，只需要relay一些指令(香山的s2 feedbackfast就是这么处理的)，但是在论文中的baseline中，他是尽可能快的de-allocated from instruction queue）
    - 这样可以让小的queue也能够找到更多的ILP，因为发射的指令出去的快说明进来的快
    - 论文的目的就是尽可能消除violation，使得memory trap penalty不重要，同时还能更早的de-allovation
    
#### Perfect Memory dependence prediction

* 不产生violation
* 不产生false dependencies

### Related work

* Digital Equipment Corporation: synchronizing？
* IBM: store barrier cache
* Moshovos: comprehensive description of memory dependence prediction， first paper指出memory dependenciese在ooo CPU中是有问题的

* paper work的前提就是历史记录的dependencies能够精确预测未来的dependencies

### Store Sets

* 预测one load依赖于多个store
* 也预测多个load依赖于相同store

store sets： 每个load指令会有一个相联的store指令

当load被fetch，处理器会决定那些store被fetch但是没有被发射的处于store set中那些指令，加上dependence的annoation

* 允许一个store对应多个load，这个很显然
* 多个store对应一个load
    - if (expr) then x = a; else x = b; ..... c = x;
    - load可能是packed的结构体，例如rgba
    - WAW也认为是dependecies

需要对比这两种的重要性，做了三种实验：

* 每个load有自己的store set，及可能保存多的store，但是store只能存在一个store set里，如果一个store导致了violation，从store set中删掉，保存到last导致violation的load's store set里
    - 可以存2，4，8 stores情况下： 8和没有限制性能基本一致
* 允许每个load指定最多一个store dependence。如果有新的violation直接替代，允许一个store存在多个store sets
* 不限制store set size，也不限制store能出现在几个store sets里


* 每个load可能4中情况：
    - 没有预测（该load的store sets为空）
    - 预测正确
    - 依赖错误
    - violation
    
* 为了解决false dep，给store set的每个store加一个2-bit的counter
    - violation，counter设置成最大
    - 如果false则减1；反之加1
    - 只有1x才强制order
    
### Implementation

* 假定store set里的stores对同一个地址写他是顺序的；即消除了内存WAW冒险
* Store Set Identifier Table(SSIT)
    - 使用common tag保存store sets中的load和stores
* Last Fetched Store Table(LFST)
    - 保存每个store set最近fetched的store的信息

* 只要是fetch到load指令，访问SSIT，获取SSID
    - 如果合理，则访问LFST，获取他的store set最近获取的store指令的inum
* 只要是fetch到store指令，访问SSIT，找到valid SSID
    - 找到了，说明属于一个store set
    - 然后访问其LFST，获取最近的store指令
    - 本store指令也会是这个store的dependent
    - 所以他会更新这个表，把自己的inum加到这个表
    - 表明本store才是最后fetched的store

* 该办法不允许store存在多个store set当中
    - 因为store也会索引SSIT，找到一个SSID
    - 如果load store pair发生一次vialation
        + store处无，load处无，则都分配同一个
        + store处无，load处有，则把load对应的SSID copy到store PC index处；然后查看LFST的最新inum
        + store处有，load处无，则把store对应的SSID copy到load PC index处
        + store处有，load处有；但两个不SSID不对应，需要merging
            * 两个sets中的一个会声明为winner
            * loser的store set会合并成winner的store-set
* winner仲裁算法：
    - 最简单的是选更小store set number的；发现这个还挺好使的
    - store-set merging可能会潜在地创建一个加载对另一个加载的存储的错误依赖关系。我们没有发现这种错误的依赖关系会对性能造成影响
        + 因为如果两个load share一个store set，相当可能一个load的conflicting stores和另外一个load的conflicting stores在cpu上运行不再同一个时刻，也就永远不会相交，也就产生不了错误依赖关系

* 解决一下多个PC的index映射到同一个SSIT entry的问题
    - 一般处理就三种：周期性清除、用个counter、加上tag
    - 实验发现周期性清除效果还不错，而且还简单
    - tag可以减少SSIT entry数量，但是会增加tag array和比较逻辑

* 建议：
    - 4K entry SSIT and a 128 entry LFST
    - cyclic clearing to invalidate the SSIT every million cycles.
    
### Conclusion

* limiting the total number of loads that can have their own store set
* limiting stores to being in at most one store set at once
* constraining stores within a store set to execute in order. 