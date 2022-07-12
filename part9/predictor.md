
* MPKI: miss prediction kilo instruction



* Championship Branch Prediction 1 [2004](https://jilp.org/cbp/)
    - 2BcgSkew
    - PPM-like, Tag-based Predictor
    - Idealized Piecewise Linear Branch Prediction
    - Frankenpredictor
    - -[O-GEHL](https://www.irisa.fr/caps/people/seznec/ISCA05.pdf)(SC的原型)
    - Adaptive Information Processing: An Effective Way to Improve Perceptron Predictors

* Championship Branch Prediction 2 [2006](https://hpca23.cse.tamu.edu/taco/camino/cbp2/index.html)
    - L-TAGE Branch Predictor
    - Perceptron Branch Predictor
    - Fused Two-Level Branch Prediction

* Championship Branch Prediction 3 [2011](https://jilp.org/jwac-2/program/JWAC-2-program.htm)
    - [A 64 Kbytes ISL-TAGE branch predictor](https://hal.inria.fr/hal-00639040/document)(引入了SC和IUM和LOOP，ISL)
    - 这个是上一篇的延申，[A New Case for the TAGE Branch Predictor](https://hal.inria.fr/hal-00639193/document)(这个提出了一个Local Statistical Corrector Predictor 缩写TAGE-LSC)
    - Revisiting Local History for Improving Fused Two-Level Branch Predictor
    - OH-SNAP: Optimized Hybrid Scaled Neural Analog Predictor
    - Penalty-Sensitive L-TAGE Predictor

* Championship Branch Prediction 4 [2014](https://jilp.org/cbp2014/program.html)
    - [TAGE-SC-L branch predictors](https://jilp.org/cbp2014/paper/AndreSeznec.pdf)


* Championship Branch Prediction 5 [2016](https://jilp.org/cbp2016/program.html)
    - [TAGE-SC-L Branch Predictors Again](http://www.jilp.org/cbp2016/paper/AndreSeznecLimited.pdf)






## TAGE

* 优点
    - 用tag消除了aliasing的问题
    - 全局历史粒度，usefulness
* https://zhuanlan.zhihu.com/p/397105511
* provider predictor
* alternate predictor
* u 更新策略
    - 当altpred和pred的预测结果不一样
        + pred正确，provider的u增加
        + 反之减少
    - 周期性的u高位变0，然后低位变0
        + 原因是一些entries，几乎不更新的情况，导致存在一些永远被假定为有效的预测
* p 更新策略
    - provider正确，则p++
    - 反之p--
* 分配策略
    - 如果总体的预测不对，先更新provider的prediction counter
    - 如果历史记录不是最长的
        + 在历史记录更前的Ti中找到一个u=0的，分配。
        + 如果没有，则所有Ti的所有u都减1
        + 如果有两个Tk和Tj(j < k)都有空的，则优先选Tj，可以用LFSR来做这个选择


## SC

* 多个逻辑表，indexed with 多个历史长度和TAGE流出的预测结果
* 最后的预测结果为 sign(the centered predictions read on SC tables + centered output of hitting bank in TAGE * 8); centered的意思是 cnt*2 + 1
* 当SC和TAGE的预测结果不同，并且sum的value高于某个动态阈值，则反转
* 动态阈值的技术，Analysis of the O-GEHL branch predictor.

## IUM

* 当取到一条conditional branch，IUM记录预测结果和TAGE预测其中的项的id(table_idx和index)。如果该分支被解析成misprediction，IUM会讲他的头指针退回到这条分支并且用正确direction更新该项
* 当在正确的路径上取值时，与执行中分支B相联的的IUM项具有匹配的预测项，同时TAGE也提供了预测想，那么以IUM中的项为准。因此可以使用正在执行还没提交的分支信息进行预测，减少因更新延迟而带来的错误预测
* 实现：每个inflight branch一个entry的全相联表