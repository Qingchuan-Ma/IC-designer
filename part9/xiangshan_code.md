
## CircularQueuePtr

传入参数 entires: entry的个数
flag: 应该是判断加完之后是否循环的一个标志把
value: 指针值
this: 返回Int，应该是value
重定义了 + - === =/=

+: 传入v: Uint，如果是幂，则cat(flag, value) + v
-: 传入v: Uint，flipped_new_ptr = value + entries - v; flag = !flag; value = value





## Frontend

### Parameters

allHistLen = (0, 4, 10, 16,   from SC
              4, 8, 13, 16,   from ITTAGE
              8, 13,32, 119,  from TAGE
              4,              from uBTB
              )

HistoryLength = allHistLen.max + numBr * FtqSize + 9 = 119 + 2*64 + 9 = 256

TageTableInfos
sets hist tag
(4096,8,8)
(4096,13,8)
(4096,32,8)
(4096,119,8)

SCTableInfos
ntables, ctrbits, history
(4, 6, 0)
(4, 6, 4)
(4, 6, 10)
(4, 6, 16)


ITTageTableInfos
sets hist tag
(256,4,9)
(256,8,9)
(512,13,9)
(512,16,9)
(512,32,9)


foldedGHistInfo是个set，不会建立重复的
history, ?
from Tage

(8, 8)
(8, 8)
(8, 7)
(13, 11)
(13, 8)
(13, 7)
(32, 11)
(32, 8)
(32, 7)
(119, 11)
(119, 8)
(119, 7)
from SC
set[FoldedHistoryInfo]() 空
(4, 1)
(10, 1)
(16, 1)
from ITTage
(4, 4)
(4, 4)
(4, 4)
(8, 8)
(8, 8)
(8, 8)
(13, 9)
(13, 9)
(13, 8)
(16, 9)
(16, 9)
(16, 8)
(32, 9)
(32, 9)
(32, 8)
from uBTB
(4, 8)

### FrontendBundle


CGHPtr: circ global histroy Ptr

* 父类 CircularQueuePtr[CGHPtr]
* 重载了cloneType

GlobalHistroy:抽象类，需要update函数

ShiftingGlobalHistory: 

* predHist = 256长度的uint
* update 根据传入的shift, 是否taken产生新的shiftGlobalHistory
* update 根据2个vec[bool] br_valid, real_taken_mask，更新shiftGlobalHistroy
  * br_valid[0:1] 代表branch valid；如果br_valid[1]，则last_valid_idx = 2；如果br_valid[0]，则last_valid_idx=1；否则last_valid_idx=0；
  * real_taken_mask[0:1]，如果real_taken_mask[1]，则first_taken_idx=1；如果real_taken_mask[0]，则first_taken_idx=2；否则如果first_taken_idx=?这个real_taken_mask必须有一个1？
  * 选择last_valid_idx和real_taken_mask里小的那个。更新。这个更新完全没理解。。。
* read: 给定n，强转为Bool，返回bools[n]
* 重定义===  =/=

CircularGlobalHistory:

* buffer: 长度为256的bool
* update返回自己，啥都没做

AllFoldedHistories:

* hist: 根据传入的(len, complen)的seq建立一个vec，内容是所有的FoldedHistroy
* method:
  * getHistWithInfo: 根据info获取foldedHostory
  * autoConnectFrom: 传入另一个AllFoldedHistroy，来重写this的history
  * update: 传入ghvector和CGHptr, shitf(num), taken

FoldedHistory: 

* parameter: len, complen(folded len), max_update_num(2)
* folded_hist: 长度为complen
* methods:
  * need_oldest_bits: 如果len大于compLen，则为1
  * info: 返回(len, compLen)
  * oldest_bit_to_get_from_ghr:  (len-1, len-2)
  * oldest_bit_pos_in_folded: (len-1 % compLen, len-2 % compLen)
  * oldest_bit_wrap_around: (len-1 / compLen > 0, len-2 / compLen)
  * oldest_bit_start: len-1 % compLen
  * get_oldest_bits_from_ghr: 传入ghr和histPtr，输出(ghr(len+histPtr), ghr(len+histPtr-1))
  * circular_shift_left: 循环左移shamt的动作
  * update: 慢通路，传入ghr, histPtr, num, taken
  * update: 快通路，传入ob, num, taken
    * bitsets_xor: len, bitsets (2维数组，数组的内容是tuple2[int,bool])
      * 
  



TableAddr:
* tag
* idx
* offset
* method:
  * tagBits: 返回vaddrbits-idxbits-instuOffsetBits
  * fromUInt: 从一个虚拟地址Uint返回TableAddr
  * getTag: 返回tag
  * getIdx: 返回idx
  * getBank: 假如banks=2，返回idx[0:0]
  * getBankIdx: 假如bank2=2，返回idx[idxbits-1:1]

### uBTB

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
* tag (20bits) 虚拟地址？


FTBMeta:

* writeWay: 2bit用于指定哪个way
* hit
* pred_cycle: 干嘛的还未知

FTB: 一样继承BasePredictor

* ftbAddr (一个bank的TableAddr，idxBits是9bits)

FTBBank:

* 512 sets, 4 ways的单bank
* io:
  * s1_fire
  * req_pc: 虚拟地址
  * read_resp: 返回FTBEntry
  * u_req_pc: 更新用虚拟地址
  * update_hits
  * update_access
  * update_pc
  * updata_write_data
  * update_write_alloc
  * try_to_write_way
  * try_to_write_pc
* 




BasePredictorIO:

* in: BasePredictorInput的一个decoupledIO
  * s0_pc
  * folded_history
  * ghist
  * resp_in: 是1个branchPredictResp
* out: BasePredictorOutput
  * last_stage_meta
  * BranchPredictResp
    * s1: 一个BranchPredictionBundle
    * s2, s3同上
      * pc
      * valid
      * hasRedirect
      * ftq_idx
      * is_minimal
      * minimal_pred: MinimalBranchPrediction
      * full_pred: FullBranchPrediction
      * folded_history
      * afhob:  AllAheadFoldedHistoryOldestBits
      * lastBrNumOj
      * histPtr
      * rasSp
      * rasTop
      * ftb_entry
* ctrl: BPUCtrl
* s0_fire: 第0级寄存器使能
* s1_fire: 第1级寄存器使能
* s2_fire: 第2级寄存器使能
* s3_fire: 第3级寄存器使能
* s2_redirect: redirect使能
* s3_redirect
* s1_ready:
* s2_ready:
* s3_ready:
* update: BranchPredictionUpdate的valid和bits
* redirect: BranchPredictionRedirect的valid和bits






### TAGE


tagetableinfos: tuple3
set history tag
(4096,8,8)
(4096,13,8)
(4096,32,8)
(4096,119,8)

tageNTable2 = 4 意思是有4个表
TotalBits = set*(1+tag+tagCtrBits+1) .reduce(_ + _)
BtSize = 2048 (base table)
bypassentries = 8(base table的bypass table)

tagCtrBits: cnter 3-bit

get_unshuffle_bits：传入idx，返回idx[0]
get_phy_br_idx: 传入unhashed_idx 和 br_logic idx 返回 unhashed_idx[0] ^ br_lidx
get_lgc_br_idx: 传入unhashed_idx 和 br_phy idx 返回 unhashed_idx[0] ^ br_pidx


TageReq:
* pc： 虚拟地址
* ghist: global history
* allFoldedHistories

TageResp:
* ctr
* u?

TageUpdate:
* pc
* allfoldedhistories
* ghist
* // update tag and ctr
* mask
* takens
* alloc
* oldCtrs
* // update u
* uMask
* us
* reset_u

TageMeta:
* providers
* providerResp
* altUsed
* altDiffers
* basecnts
* allocates
* takens
* scMeta
* pred_cycle
* use_alt_on_na


TageBTable: basic table
* bimAddr
* s1_cnt: 就是一个vec，把读出的两个way映射成了一个vec，每个元素2-bit
* set: 2048, way: 2, data: 2-bit(无tag)
* 还有一个WrBypass, 8项，idx为11位。存放的pidx
* 给一个u_idx，hit，则找到oldCtrs；反之使用oldCtrs = io.update_cnt
* 更新时，使用oldCtrs进行更新。
* 两个问题：为什么要区分lidx和pidx；为什么要使用bypass
  
WrBypass:

* numEntries
* idxWidth
* numWays
* tagWidth
* 本质是一个CAMTemplate 内容为idx+tag，项数量为numEntries，readWidth = 1和一个Mem(numEntries, Vec(2, data))
* 只有write的请求，给出write_idx+tag（cam的内容，用来找hit_idx的）
  * 如果cam hit，则数据更新hit_idx的mem里；同时返回数据；hit是单周期就可以返回的
  * 如果没有hit，则数据写到enq_idx的mem里；write_idx+tag写到cam里内容里，enq_idx写到cam的idx里



CAMTemplate: data, set, readWidth

* r: req, resp
* w: valid, bits(index data)
* 本质是一个reg vec，长度是set数量，内容大小是data.getWidth
* r的时候给出一个readWidth为长度的vec数据内容，返回一个Vec(readWidth, Vec(set, Bool()))；单周期内返回
* w的时候带上valid和bits(index data)，把数据写进去即可


TageTable: 
* nRows: 传入参数
* histLen: 传入参数
* tagLen: 传入参数
* tableIdx: 传入参数
* TageEntry(valid, tag, ctr)
* io:
  * TageReq
  * TageResp
  * TageUpdate
* SRAM_SIZE = 256
* nRowsPerBr = nRows / 2
* nBanks = 8
* bankSize = nRowsPerBr / nBanks
* bankFoldWidth: 如果bankSize > SRAM_SIZE 那么bankSize / SRAM_SIZE；否则为1
* uFoldedWidth = nRowsPerBr / SRAM_SIZE
* get_bank_mask：给idx，返回一个bool vec，表明访问哪个bank
* get_bank_idx：给idx，返回idx >> 3；这个就是去除bank之后的idx
* get_way_in_bank: 给idx，如果需要折叠，则返回idx中的折叠bit
* perBankWrbypassEntries = 8 跟base Tage一样
* getUnhashedIdx:  pc >> instOffsetBits
* us:   FoldedSRAMTemplate


FoldedSRAMTemplate:

setIdx -> ridx | width | way
