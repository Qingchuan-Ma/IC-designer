
* MPKI: miss prediction kilo instruction



* Championship Branch Prediction 1 [2004](https://jilp.org/cbp/)
    - 2BcgSkew
    - PPM-like, Tag-based Predictor
    - Idealized Piecewise Linear Branch Prediction
    - Frankenpredictor
    - O-GHEL
    - Adaptive Information Processing: An Effective Way to Improve Perceptron Predictors

* Championship Branch Prediction 2 [2006](https://hpca23.cse.tamu.edu/taco/camino/cbp2/index.html)
    - L-TAGE Branch Predictor
    - Perceptron Branch Predictor
    - Fused Two-Level Branch Prediction

* Championship Branch Prediction 3 [2011](https://jilp.org/jwac-2/program/JWAC-2-program.htm)
    - A 64 Kbytes ISL-TAGE branch predictor
    - Revisiting Local History for Improving Fused Two-Level Branch Predictor
    - OH-SNAP: Optimized Hybrid Scaled Neural Analog Predictor
    - Penalty-Sensitive L-TAGE Predictor

* Championship Branch Prediction 4 [2014](https://jilp.org/cbp2014/program.html)


* Championship Branch Prediction 5 [2016](https://jilp.org/cbp2016/program.html)


## TAGE

* 优点
    - 消除了aliasing的问题
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