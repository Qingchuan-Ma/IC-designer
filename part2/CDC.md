1. 是否允许错过一些采样
    1. 异步fifo，判空和判满都不需要采样所有的信号，只需要采到的信号是真实的就行，因此使用格雷码
2. 每一个信号都必须被采样

tips:

1. 送入其他时钟域之前寄存

# 快到慢必须采样

## 单信号

1. open-loop solution (three edge)
    
    ![](../assets/cdc1.png)
    
2. closed-loop solution
    
    ![](../assets/cdc2.png)
    

## 多信号

### Multi-bit signal consolidation

### Multi-cycle path formulations (MCP)

    ![](../assets/cdc3.png)

1. Closed-loop - MCP formulation with feedback
    

    ![](../assets/cdc4.png)
    
2. Closed-loop - MCP formulation with acknowledge feedback
    
    ![](../assets/cdc5.png)

### Pass multiple CDC bits using gray codes

1. conversion
    
    ![](../assets/cdc6.png)
    
    assign bin = gray
    
    ![](../assets/cdc7.png)
    
    assign gray = (bin>>1) ^ bin
    
2. counter
    
    ![](../assets/cdc8.png)
    
    ![](../assets/cdc9.png)
    
3. 1-deep / 2-register FIFO synchronizer
    
    ![](../assets/cdc10.png)