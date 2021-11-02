# 量化

## 量化的基本原理

量化是指对在某一区间内可以连续变化的变量做近似处理，使其只可能在有限多个离散值中变化。模型压缩中的量化指对模型中的输入与权重变量做近似，使其从浮点数变成定点数，从而达到减小计算量，适应硬件计算资源的目的[12]。也就是说，量化是针对模型的输入与权重变量进行的，通过近似，将其从浮点数转化成定点数。

简而言之，模型量化是将浮点存储（运算）转换为整型存储（运算）的一种模型压缩技术。

以线性均匀非对称量化为例。映射步长比△（scale）与零点z（zero-point）是在量化操作中的两个参数。

假设有一个浮点型变量*x*，要将其量化成8bit定点数。那么它量化后可变化的值$N_level$就是$2^8 = 256$，具体来讲就是可以取0到255这些整数值。对于给定的映射步长比△，与零点*z*，就有这样的公式。其中，$x_Q$就是量化后的新变量。
$$
a^2
$$
![img](file:///C:/Users/asus/AppData/Local/Temp/msohtmlclip1/01/clip_image008.jpg)

这里的两个函数round与clamp，其中round可以理解为四舍五入取整，clamp是分段函数，其函数图像如图3.1。

![img](file:///C:/Users/asus/AppData/Local/Temp/msohtmlclip1/01/clip_image010.jpg)

![img](file:///C:/Users/asus/AppData/Local/Temp/msohtmlclip1/01/clip_image012.jpg)

**图****3.1** clamp函数的函数图像
