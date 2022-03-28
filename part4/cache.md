## Basic

* block = line
* block 有很多 word
* 组内并行，组外index
* n-way set associative: n路就是n line/block
* fully associative has just one set
* write-through: cache和memory都更新
* write-back: 更新cache
* write-buffer: 两种都需要
* miss rate
    - compulsory
    - capacity: set 不够
    - conflict: line 不够
    - coherency
* Average memory access time = Hit time + Miss rate * Miss penalty
* six basic cache optimizations
    - Larger block size to reduce miss rate
    - Bigger caches to reduce miss rate
    - Higher associativity to reduce miss rate
    - Multilevel caches to reduce miss penalty
        + Hit timeL1 + Miss rateL1 * (Hit timeL2 + Miss rateL2 * Miss penaltyL2)
    - Giving priority to read misses over writes to reduce miss penalty
        + 读优先度一般都高于写，从write buffer中先读
    - Avoiding address translation during indexing of the cache to reduce hit time
        + VIPT
* Ten Advanced Optimizations of Cache Performance
    - classify:
        + Reducing the hit time
        + Increasing cache bandwidth
        + Reducing the miss penalty
        + Reducing the miss rate
        + Reducing the miss penalty or miss rate via parallelism
    - Small and Simple First-Level Caches to Reduce Hit Time and Power
        + high associative reason
            * many processors take at least 2 clock cycles to access the cache and thus the impact of a longer hit time may not be critical
            * VIPT
            * multithreading, conflict misses can increase, making higher associativity more attractive.
    - Way Prediction to Reduce Hit Time
        + extra bits are kept in the cache to predict the way (or block within the set) of the next cache access
    - Pipelined Access and Multibanked Caches to Increase Bandwidth
        + 更深的流水线，缩短了时钟周期、提高带宽；代价increased latency 和分支预测的greater penalty
        + sequential interleaving
    - Nonblocking Caches to Increase Cache Bandwidth
        + nonblocking cache or lockup-free cache
        + 非阻塞：甚至容忍多重缺失
        + 决定支持多少个未处理缺失(outstanding misses)，需要考虑多重因素
            * miss 的时间和空间局域性，决定了一次缺失会触发多少次对下级的新访问
            * 对访问请求作出回应的存储器或缓存的带宽
            * 为了允许在最低级别的缓存中出现更多的未处理缺失(miss time 最长)，需要在较高级别上支持至少数目相等的缺失，这是因为这些缺失必须在最高级别缓存上启动
            * 存储器系统的延迟
        + 具体实现困难
            * arbitrating contention between hits and misses
            * tracking outstanding misses so that we know when loads or stores can proceed
    - Critical Word First and Early Restart to Reduce Miss Penalty
        + 前者：先送word，至于block再说
        + 后者：正常送block，但是到了就立刻发给处理器
        + 优化有限
    - Merging Write Buffer to Reduce Miss Penalty
        + 如果wirte buffer中的地址block和现在要写的地址block号重合，直接修改write buffer中的数据，Write merging
        + IO区域不允许写合并
    - Compiler Optimizations to Reduce Miss Rate
        + 循环嵌套顺序交换，提高空间局域性
        + 分块访问，提高时间局域性。一次算不完，算多次，例如矩阵乘法分成子矩阵乘法
    - Hardware Prefetching of Instructions and Data to Reduce Miss Penalty or Miss Rate
        + 需要利用未被充分利用的存储带宽，可以需要编译器帮助，减少无用预取
    - Compiler-Controlled Prefetching to Reduce Miss Penalty or Miss Rate
        + 编译器插入预取指令
        + 可以分为register or cache，faulting or non-faulting
            * 正常的load可以看作faulting register prefecth instruction
            * cache + non-faulting也称non-binding
    - Using HBM to Extend the Memory Hierarchy
        + DRAM-based L4 Cache with tags stored in HBM
            * Loh and Hill
            * alloy cache
        + two full DRAM accesses: one to get the initial tag and a follow-on access to the main memory

![](../assets/cache0.png)

### Non-blocking cache implementation

两个挑战：

* 在命中和未命中之间进行仲裁
    - 命中可能会与从内存层次结构的下一层返回的未命中发生碰撞，所以需要解析碰撞
        + 可以设置hit的优先级比miss高
        + ordering colliding misses
* 跟踪未完成的miss，以便我们知道何时可以进行装载或存储
    - miss的返回可能不是按miss之间的顺序返回的，有些可能返回更早
    - 当一个miss返回时
        + 处理器必须知道那个load和store导致了miss
        + 必须知道数据应该放在哪里
    - 最近的处理器保存在a set of registers (Miss Status Handling Registers(MSHRs))
        + 如果我们允许n个outstanding misses，就有n个MSHR，每一个MSHR保存这个miss要去cache的那个位置以及tag位，还要表明哪个load或store导致这个miss
        + 全相联的方式组织所有MSHR，如果第二次miss，发现tag相同，说明二次miss就不再分配MSHR，而是在load miss queue和store miss queue分配一个入口，达到多个miss合并
        + 当miss发生，分配一个MSHR来处理这个miss，记录miss信息，把内存请求和MSHR的index挂钩
        + 内存系统返回数据的时候使用这个MSHR的信息将数据送到合适的block然后通知load和store数据已经可用
* 其他的问题
    - 多核情况下的内存系统的一致性模型和缓存一致性问题
    - cache miss不再是原子，因此可能导致死锁

## Replacement Policies

When a new line is to be inserted into the cache, which line should be evicted to make space for the new line

Belady’s policy: it evicts the line that will be re-used furthest in the future

### Taxonomy

* Coarse-Grained policies, treat all lines identically when they are inserted into the cache and only differentiate among lines based on their their behavior while they reside in the cache. 
* Fine-Grained policies distinguish among lines when they are inserted into the cache (in addition to observing their behavior while they reside in the cache).

* Insertion Policy: How does the replacement policy initialize the replacement state of a new line when it is inserted into the cache?
* Promotion Policy: How does the replacement policy update the replacement state of a line when it hits in the cache?
* Aging Policy: How does the replacement policy update the replacement state of a line when a competing line is inserted or promoted?
* Eviction Policy: Given a fixed number of choices, which line does the replacement policy evict?

* Coarse-Grained policies (insertion一视同仁)
    - Recency-Based Policies
        + fixed lifetime
        + extended lifetime
    - Frequency-BasedPolicies
    - Hybrid Policies
* Fine-Grained policies (smarter insertion policy)
    - Classification-Based Policies
        + cache-friendly
        + cache-averse
    - Reuse Distance-Based Policies

The primary goal of replacement policies is to increase cache hit rates, and many design factors contribute toward achieving a higher hit rate. We outline three such factors as follows：

* Granularity: inserstion是否使用过去数据还是一视同仁
* History: 需要使用多少过去数据
* Access Patterns: 替换策略对某些访问模式的专门化程度如何?它对访问模式的更改或不同访问模式的混合是否健壮?


### Coarse-Grained policies

three commonlu observed cache access patterns:
* recency-friendly accesses: working set small enough to fit cache size
* thrashing accesses: working set bigger than cache size
* scans: never repeat

#### RECENCY-BASED POLICIES

* LRU
* MRU
* EELRU: 跟踪Recency链表的命中次数，如果分布单调递减，认为没有thrash，反之有
* Seg-LRU


LIP
