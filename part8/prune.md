# 剪枝

## 原理

剪枝即减去神经网络中不重要的权值，希望节省空间、减少计算、提高推理速度。剪枝要关注三个问题，分别是剪多少，剪哪里和怎么剪。回答剪多少的问题，就要知道修剪的颗粒度和修剪的比例；回答剪哪里的问题，就要寻找某个方法指示可以被修剪的部位；回答怎么剪的问题，就是为前一问题的答案选择合适的剪枝流程。

颗粒度是判断重要性和修剪的单位。就像修剪灌木的单位可以是半片叶子、整片叶子或者一枝分叉一样，神经网络的剪枝也有不同颗粒度，大致可分为层级、通道级、向量级和稀疏化矩阵，他们分别对应剪枝一层网络、一层网络中的卷积核/通道、卷积核中的一个向量，以及卷积核的每个权重。其中稀疏化矩阵属于非结构化剪枝，其他层级都属于结构化剪枝。

<img src="./compression_pic\prune1.jpg" alt="prune1" style="zoom: 50%;" />



​                                                                        **图2.1**、预定义和自动化剪枝的区别（以通道剪枝为例）

修剪的比例有两种确定方式，分别称为预定义(Predefined)方式和自动(Automatic)方式。如图2.1所示，左侧为预定义剪枝，右侧为自动化剪枝。预定义剪枝中剪枝比例被人为地、经验性地设置，而并且剪枝比例在各通道相同。自动化剪枝中，剪枝比例是由算法自行决定的，每个通道以不同比例被修剪。非结构化剪枝也可以视作自动化剪枝，因为矩阵的稀疏是由算法决定的，人无法提前知道要减去哪些权重。

剪枝要剪掉不重要的单位，可以使用三种标准判断单位重要性，分别是性质重要性、适应重要性和重构误差最小。这些方法或人为设计某个标准，或者利用网络中参数在训练过程中的变化来指示重要性，或者使修剪前后网络特征图的差距最小化，以判断重要性。

使用某方法判断网络中各部分的重要性，然后对应减去不重要的单位，即为基本剪枝步骤。剪枝流程可分为三阶段剪枝和单阶段剪枝。三阶段剪枝中，训练与剪枝过程是分开的，之后通常还需要微调，用来恢复准确率，即形成训练-剪枝-微调的剪枝流程。单阶段剪枝中，训练中引入正则化项，用来指示可修剪部位，即训练和剪枝同时进行，有时单阶段剪枝也需要微调。

剪枝可能需要迭代进行。首先，剪枝可能直接对整个网络的剪枝，也可能迭代式逐层修剪；另外，剪枝可能一次成功，也可能需要多次剪枝才能达到合适指标。

由于在当前的神经网络中，卷积层的层数和计算量远大于全连接层，所以本文主要对卷积层剪枝进行说明。

## 剪枝的颗粒度和结构化划分

颗粒度大小和结构化与否是两种剪枝分类标准。颗粒度划分以剪枝的层级进行分类，结构化划分以剪枝是否有规律进行分类。颗粒度和结构化分类之间有关系，颗粒度最小的稀疏化矩阵属于非结构化剪枝，而其他更大颗粒度的剪枝都属于结构化剪枝。

### 颗粒度划分

从剪枝的颗粒度上划分，可以从粗到细大致分为四个层级：层级剪枝、通道级/核级剪枝、向量级剪枝和稀疏化矩阵，其中前两种可用现有库实现，且无需特殊硬件支持，在实际使用中能体现出加速效果；后两种需要专门的软硬件支持，实现难度较大。

图2.2展示了不同颗粒度的剪枝，图中每个大方块代表三维卷积核，透明或者黄色部分表示被减掉的结构，从左到右依次是层级剪枝、核级剪枝、通道级剪枝、向量级剪枝和稀疏化矩阵，其中核级剪枝和通道级剪枝在实质上是一样的。

![image-20211102143744258](.\compression_pic\image-20211102143744258.png)

​                                                                               **图2.2**： 不同颗粒度的剪枝

值得注意的是，颗粒度划分并非只有四个层级，还有块级剪枝、组级剪枝等。块级剪枝的颗粒度比层级更大，直接减掉多层网络；组级剪枝颗粒度大于稀疏化矩阵，对权值分组并同时稀疏。

剪枝单位为层，每次可能剪掉一层或多层。对于卷积层，即减掉本层所有卷积核，对于全连接层，即减掉本层所有神经元和连接。

#### 层剪枝

剪枝单位为层，每次可能剪掉一层或多层。对于卷积层，即减掉本层所有卷积核，对于全连接层，即减掉本层所有神经元和连接。

#### 通道级/核级剪枝

通道级剪枝，每次剪掉一个或多个通道，使输出通道数减少；核级剪枝，每次减掉一个或多个卷积核。由于前后卷积层的关系，这两个层级在本质上相同。

![image-20211102143930054](.\compression_pic\image-20211102143930054.png)

​                                                                            **图2.3**：前后卷积层中核与通道的关系

如图2.3所示，输入的三个维度分别为图像的宽、高和通道数。Filters的四个维度分别为核的宽、高、通道数和卷积核个数，字母下标表示层数。观察可知，每个三维卷积核对应产生一张特征图，特征图的通道数由上一层卷积核个数决定，同时与本层卷积核通道数相同。假如移去第i层的红色卷积核，则第i+1层的红色特征图就不会被产生，同时本层卷积核的红色通道失去作用，可以被剪枝。

换句话说，由于前后两层的卷积核、特征图和卷积核通道之间存在相关关系，减掉i层卷积核，相当于减掉(i+1)层的卷积核通道；反过来讲，减掉(i+1)层卷积核的通道，相当于剪掉i层卷积核。因此，核级剪枝实际上为通道级剪枝。

#### 向量级剪枝

剪枝单位是卷积核上某向量。如果把卷积核看作三维立方体，则向量是当前通道上贯穿卷积核的小柱体。具体地，对于3×3×3卷积核，剪枝向量形状为1×1×3。

#### 稀疏化矩阵

稀疏化矩阵又称为细粒度剪枝，它对单个权重值剪枝。传统非结构化稀疏方法是对网络引入复杂的正则项，使网络权重中的部分值趋向于0，然后设置阈值，剪去其中趋于0的权重数值。

### 结构化和非结构化

有规律的剪枝可以称为结构化剪枝，如层级、通道级和向量级剪枝。无规律的剪枝称为非结构化剪枝，如稀疏化矩阵。

结构化剪枝中，层级和通道级剪枝可用现有的库和通用硬件来实现，而向量级剪枝和非结构化剪枝需要专用库和硬件设备，才能在实际应用中达到加速效果。虽然结构化剪枝的灵活性不如非结构化剪枝，但是结构化剪枝不需要特殊支持，因此结构化剪枝更常用。

#### 结构化剪枝

结构化剪枝的颗粒度比较粗，通道级剪枝能有效改善模型速度和大小，保证通用性，然而也容易造成精度大幅下降，同时模型仍残留冗余。

#### 非结构化剪枝

非结构化剪枝可以更好的保留模型精度，但是被剪除的网络连接在分布上没有任何连续性，没有规律的空间存取对实际加速起到了反作用，实际运行时加速效果不理想。

## 剪枝流程

剪枝流程都可以归为三阶段剪枝或单阶段剪枝。值得注意的是，这两种模式并非绝对不变，每种方法都可以在以下模式的基础上有所改动。 

  三阶段剪枝和单阶段剪枝的本质差别在于网络训练和选择剪枝是否放在同一步进行：如果不是同步进行，则是三阶段剪枝；如果是同步进行，则属于单阶段剪枝。

### 三阶段剪枝

图2.4为三阶段剪枝的基本流程，三阶段剪枝是最为典型的剪枝流程。首先，要训练一个大的网络，即使有冗余也要保证高准确率，如果有训练好的网络而且不需要修改，则可以直接使用训练好的模型；接着，在大模型的基础上选用某种标准，对颗粒度（通常为通道级）进行选择和剪枝；最后使用原模型权值初始化，微调剪枝网络。

<img src=".\compression_pic\image-20211102145811405.png" alt="image-20211102145811405" style="zoom:150%;" />

​                                                                               **图2.4**：基本的三阶段剪枝流程

三阶段剪枝的特点在于，判断颗粒度重要性并选择剪枝的过程，是在已经训练好的网络之上进行的。通常剪枝之后会微调，由此形成“训练-剪枝-微调”的三阶段剪枝方式。

如图2.5，三阶段剪枝也可以根据实际情况，增加迭代过程，以获得更好的压缩效果，当压缩效果和准确率达到令人满意的平衡时，迭代结束，也就获得压缩后的模型。

<img src=".\compression_pic\image-20211102145907208.png" alt="image-20211102145907208" style="zoom:150%;" />

​                                                                                 **图2.5**：迭代式的三阶段剪枝流程

三阶段剪枝多用于基于性质重要性的剪枝，但迭代过程不是必须的，比如“Pruning Filters for Efficient ConvNets”中，对于不敏感的部分可以单次剪枝而不造成准确度的明显下降。另外，基于重构的剪枝也可以认为是三阶段剪枝，因为训练和剪枝过程不同时进行。

### 单阶段剪枝

![img](.\compression_pic/clip_image002.jpg)

​                                                                                   **图2.6**：基本的单阶段剪枝流程

图2.6为单阶段剪枝的流程，可以看到训练和剪枝是同时进行的。单阶段剪枝会向网络中引入参数，然后从头同时训练网络的权值参数，在训练过程中参数自动变化，起到指示作用，减去对应结构后获得压缩模型。此流程多用于基于适应重要性的剪枝。

单阶段剪枝也可以有微调和迭代过程，图2.8就是基于适应重要性的剪枝论文“Learning Efficient Convolutional Networks through Network Slimming”使用的剪枝流程。

![img](.\compression_pic/clip_image004.jpg)

​                                                                                    **图2.7**：迭代式的单阶段剪枝

## 对剪枝的再思考

剪枝工作中普遍认同这两个观点：

1.三阶段剪枝流程中，首先训练一个大的有冗余的网络是必要的。大网络准确率会更高，具有更强表达力和更大优化空间，能安全剪枝而不明显伤害准确率。因此，从大网络中剪枝的效果优于直接从头训练一个小的网络。

2.修剪后的结构及权值都被认为重要。在剪枝之后常用原模型的权值初始化确定结构的小模型，再微调。

“Rethinking the Value of Network Pruning”对这两个观点提出质疑。针对第一个观点，论文使用图2.1中预定义的方式对网络中每层的通道均匀剪枝，然后随机初始化并从头训练。接着把他的表现与经过三阶段剪枝的网络的表现进行比较。结果发现，随机初始化-训练的小网络总能和三阶段剪枝网络的表现相当，有时小网络的表现甚至更好。这说明，三阶段剪枝中不必先训练一个大的有冗余的网络，而可以直接训练一个小网络。

针对第二个观点，论文使用自动方式结构化修剪网络，得到的网络结构分别使用随机初始化训练和在原模型的权值上微调，发现两者的表现相当，前者还可能更好。这说明，修剪之后的网络结构有价值，而权重没价值。在非结构化剪枝中，当数据集小时原模型权重价值不大，对大数据集，原模型权重具有一定价值。

最后，该论文认为，如果目标网络可以使用预训练剪枝方式，则从头训练小网络具有三个优点。第一，模型小则占用GPU内存少，训练更快；第二，无需寻找剪枝标准，也无需层层微调；第三，无需在修剪过程中引入额外超参数。

虽然三阶段剪枝并不能在准确率上取得优势，但若有训练好的模型可以直接使用，那么剪枝-微调比从头训练省时。另外，如想要找到结构多样的网络，或者不知怎样的结构可取时，可以使用自动化剪枝从大模型开始剪枝。