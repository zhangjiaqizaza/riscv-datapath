# RISC-V 处理器数据通路图

```mermaid
flowchart LR
    %% IF阶段
    PC[PC] --> InstMem[指令存储器]
    PC --> Add4[+4]
    
    %% 流水线寄存器用竖线表示
    InstMem --> PipeIF["|"]
    Add4 --> PipeIF
    
    %% ID阶段
    PipeIF --> Decoder[Decoder]
    Decoder --> |rj| RegFile[寄存器堆]
    Decoder --> |rk| RegFile
    Decoder --> |rd| RegFile
    
    %% 寄存器堆输出
    RegFile --> |rdata1| RjVal[rj_value]
    RegFile --> |rdata2| RkVal[rk_value]
    
    %% 控制信号
    Decoder --> ALUOpSig[alu_op]
    Decoder --> RegWESig[gr_we]
    
    %% ID/EX流水线寄存器
    RjVal --> PipeID["|"]
    RkVal --> PipeID
    ALUOpSig --> PipeID
    RegWESig --> PipeID
    
    %% EX阶段
    PipeID --> |src1| ALU[ALU]
    PipeID --> |src2| ALU
    ALU --> |res| ALUResult[alu_result]
    
    %% EX/MEM流水线寄存器
    ALUResult --> PipeEX["|"]
    PipeID --> PipeEX
    
    %% MEM阶段
    PipeEX --> |addr| DataMem[数据存储器]
    PipeEX --> |wdata| DataMem
    DataMem --> |rdata| MemData
    
    %% MEM/WB流水线寄存器
    MemData --> PipeMEM["|"]
    PipeEX --> PipeMEM
    
    %% WB阶段
    PipeMEM --> |wdata| RegFile
    
    %% 控制信号传递（蓝色线）
    ALUOpSig -.-> ALU
    RegWESig -.-> RegFile
```

## 数据通路说明

### 五级流水线架构：

1. **IF (指令提取)**：
   - 程序计数器(PC)提供指令地址
   - 指令存储器读取指令
   - 计算下一条指令地址(PC+4)

2. **ID (指令译码)**：
   - 控制单元译码指令，生成控制信号
   - 立即数生成器从指令中提取立即数
   - 寄存器堆读取源寄存器数据
   - 分支比较器进行分支判断

3. **EX (执行)**：
   - ALU执行算术和逻辑运算
   - 选择器选择ALU的输入源
   - 计算分支目标地址

4. **MEM (存储访问)**：
   - 数据存储器进行读写操作
   - 分支选择器决定下一条指令地址

5. **WB (写回)**：
   - 选择写回数据(ALU结果或存储器数据)
   - 将结果写回寄存器堆

### 主要功能单元：

- **ALU**: 执行算术和逻辑运算
- **立即数生成器**: 从指令中提取并扩展立即数
- **寄存器堆**: 32个通用寄存器的存储
- **分支判断单元**: 比较寄存器值并生成分支条件
- **数据存储器**: 存储和读取数据

### 流水线寄存器：
- **IF/ID**: 保存指令和PC值
- **ID/EX**: 保存译码后的数据和控制信号
- **EX/MEM**: 保存执行结果和内存访问信息
- **MEM/WB**: 保存内存数据和写回信息