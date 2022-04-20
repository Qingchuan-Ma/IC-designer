## Frontend

## uBTB

记在纸上了

### FTB

一部分记在纸上了，之后填上来


512 sets + 4 ways

traits BPUUtils:

* getFallThroughAddr: 根据start, carry, pft_addr获取ft_addr


FTB entry：

* valid
* 1个brSlot
* 1个tailSlot 该slot可能是JMP也可能是BR
* pftAdder: partial Fall-Through Addr
* carry: 表明这个addr是否进位还是单纯 concat
* isCall
* isRet
* isJalr
* last_may_be_rvi_call
* always_taken
* method:
  * getSlotForBr: 如果idx为1，选tailSlot，否则选brSlot
  * allSlotsForBr: 返回所有slots list
  * setByBrTarget: 根据target地址设置idx的slot LowerStat
  * setByJmpTarget: 根据target地址直接设置tailSlot的LowerStat，sharing为false，意思是该slot之后target计算都用offsetlen（Jump的offset），不用br_offset计算
  * getTargetVec: 获得两个目标地址Vec
  * getOffsetVec: 获得两个分支指令的Offset Vec
  * isJal: 返回!isJalr，意思是如果不是Jal就是Jalr
  * getFallThrough: 调用getFallThroughAddr获得地址
  * hasBr: 在给定的offset之内看有没有branch
  * getBrMaskByOffset: 给定offset之内给出一个mask，例如11表明两个slot都是valid，01表明后者是valid，前者不是
  * getBrRecordedVec: 在给定offset，看有没有slot有记录该offset，返回存有该offset的 Vec
  * brIsSaved: 给定一个offset，判断该offset是都存在于某个slot，返回上述函数的归约或
  * brValids: 返回是branch且valid的valid list
  * noEmptySlotForNewBr: 返回是否有非valid的slot用于装新的br
  * newBrCanNotInsert: 给定一个offset，返回tailSlot是否为valid并且offset不超
  * jmpValid: tailSlot.valid && !tailSlot.sharing
  * brOffset: 返回所有offset Vec
  

FtbSlot:

* valid
* offset: 表明该分支离start的距离？还是离block的开头的距离
* lower: 分支预测目标地址低位
* sharing: 还没看出来是干嘛的，为1选suboffsetlen，为0选offsetlen，为sharing表明是个branch，不是jump
* tarStat: 表明目标地址的高位是overflow还是underflow，相对应+1 -1再与lower concat
* method:
  * setLowerStatByTarget: 根据target设置lower, tarStat和 sharing
  * getTarget: 根据lower tarStat和sharing获取target address
  * fromAnotherSlot: 把别的slot搬运到本slot


FTBEntryWithTag:

* entry
* tag (20bits) 物理地址？


FTBMeta:

* writeWay: 2bit用于指定哪个way
* hit
* pred_cycle: 干嘛的还未知

FTB: 一样继承BasePredictor

* 

