# GPU


## glossary
* TM: transactional memory
* warps(wavefronts): groups of scalar threads in lockstep on SIMD hardware to exploit their regularities and spatial localities; N(32 threads) A(64 threads)
* SIMT: single-instruction multiple-thread
* SAXPY: single-precision scalar value A times vector value X plus vector value Y
* CTA(thread block): cooperative thread array (存放warps)
* scratchpad memory/shared memory: Threads within a CTA can communicate with each other efficiently via it
* LDS: AMD, local data store, similar to scratchepad memory
* GDS: AMD, global data store scratchpad memory sared by all cores on the GPU
* SM: streaming multiprocessor contains a single shared memory which divided up among all CTAs running on that SM
* CDP: CUDA Dynamic Parallelism
* DWF: Dynamic Warp Formation
* PTX: Parallel Thread Excution ISA
* SASS: Streaming ASSembler (the actual instruction set architecture supported by the hardware)
* HSAIL: AMD, Heterogeneous System Architecture intermediate language
* Southern Islands architecture: AMD, hardware-level ISA

