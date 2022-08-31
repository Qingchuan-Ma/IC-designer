## value prediction

### EVES
André Seznec. 2018. Exploring value prediction with the EVES predictor. In 1st Championship Value Prediction

E-Stride Predictor + EVTAGE

E-Stride:

1. Last Commit Value + (inflight + 1) * stride
2. 3-way skewd asociative, 16 set
3. line: 64-bit last value, 2-bit useful counter, 20-bit stride, 14-bit tags, 5-bit confidence counter
4. 1-bit notfirstOcc indicating that the stride has been set for the entry since its allocation or the last misprediction. 
5. https://www.microarch.org/cvp1/cvp1online/contestants.html code_website



### DLVP
Rami Sheikh, Harold W. Cain, and Raguram Damodaran. 2017. Load Value Prediction via Path-based Address Prediction: Avoiding Mispredictions due to Conflicting Stores. In 2017 50th Annual IEEE/ACM International Symposium on Microarchitecture (MICRO)

fetch generate address prediction, 
access icache, hit return predicted value, miss generate a prefetch
must return before rename, otherwise probe fail


### Composite value predictor
fusion of EVES and DLVP
Rami Sheikh and Derek Hower. 2019. Efficient Load Value Prediction Using Multiple Predictors and Filters. In 2019 IEEE International Symposium on High Performance Computer Architecture (HPCA).

1. Context-agnostic Predictors, E-STRIDE
    1. Last Value Prediction(LVP), The PC of the load highly correlates with the load value
    2. Stride Address Prediction, The PC of the load highly correlates with the load address.
2. Context-aware Predictors:
    1. Context Value Predictions, VTAGE, branch-history-based
    2. Context Address Prediction, DLVP, load-path-based


### Efficient Pipeline Prefetch

Ricardo Alves, Stefanos Kaxiras, and David Black-Schaffer. 2021. Early Address Prediction: Efficient Pipeline Prefetch and Reuse. ACM Transactions on Architecture and Code Optimization 18, 3, Article 39 (June 2021), 22 pages.

### Register File Prefetching
Sudhanshu Shukla, Sumeet Bandishte, Jayesh Gaur, and Sreenivas Subramoney. 2022. Register File Prefetching. In Proceedings of The 49th Annual

1. (value prediction) + (RFP)(context-agnostic address prediction + register file prefecthing from lsq and dcache)
2. address prediction is simple stride prediction
3. value prediction is EVES, Here, an RFP is performed for a given load only if the load is not value predictable
4. do not flush when misprediction, only cancle wake-up speculative dependence inst.



