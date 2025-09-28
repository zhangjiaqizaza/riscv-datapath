# LoongArch32 单周期处理器设计

## 实验概述

本实验旨在设计并实现一个支持 LoongArch32 指令集（12条指令子集）的单周期处理器。处理器使用 Verilog HDL 描述，并在 Vivado 环境中进行仿真和 FPGA 验证。

## 支持的指令集

| 指令类型 | 指令助记符 | 说明 |
| :--- | :--- | :--- |
| **R-Type** | `add.w` | 加法 |
| | `sub.w` | 减法 |
| | `and` | 按位与 |
| | `mul.w` | 有符号乘法 |
| **I-Type** | `addi.w` | 立即数加法 |
| | `slti` | 有符号小于立即数置位 |
| | `ori` | 立即数按位或 |
| | `slli.w` | 逻辑左移 |
| | `ld.w` | 加载字 |
| | `st.w` | 存储字 |
| **B-Type** | `b` | 无条件跳转 |
| | `beq` | 等于跳转 |

## 处理器数据通路

下图展示了支持所有12条指令的完整单周期数据通路。数据通路在 `add.w` 指令的基础上，增加了立即数生成、分支跳转、数据访存等逻辑。

```mermaid
flowchart TD
    %% ========== 取指阶段 ==========
    PC[PC<br>程序计数器] -->|pc| IMEM[Instruction Memory<br>指令存储器]
    IMEM -->|inst| CU[Control Unit<br>控制单元]
    IMEM -->|inst| IG[Immediate Generator<br>立即数生成器]

    %% ========== 译码与寄存器读 ==========
    IMEM -->|31:27 rj| RF_raddr1
    IMEM -->|24:20 rd/rk| RF_raddr2
    subgraph RF[Register File 寄存器堆]
        RF_raddr1[raddr1] --> RF_rdata1
        RF_raddr2[raddr2] --> RF_rdata2
        RF_waddr --> RF_wen
        RF_wdata_in --> RF_wen
    end
    RF_rdata1 -->|data1| ALU_src1_MUX
    RF_rdata2 -->|data2| ALU_src2_MUX
    RF_rdata2 -->|data2| DMEM_wdata

    %% ========== 控制信号 ==========
    CU -->|RegWrite| RF_wen
    CU -->|MemtoReg| WB_MUX_sel[MemtoReg]
    CU -->|ALU_Op| ALU_op[ALU Operation]
    CU -->|ALUSrc| ALU_src2_sel[ALUSrc]
    CU -->|MemWrite| DMEM_wen
    CU -->|MemRead| DMEM_ren
    CU -->|Branch| PC_src_sel[PC Source]

    %% ========== 立即数生成 ==========
    IG -->|imm32| ALU_src2_MUX

    %% ========== 执行阶段 ==========
    ALU_src1_MUX[ A ] -->|src1| ALU
    ALU_src2_MUX{ALUSrc MUX} -->|src2| ALU
    ALU_src2_MUX -->|0: data2| ALU
    ALU_src2_MUX -->|1: imm32| ALU
    ALU -->|result| ALU_result[ALU Result]
    ALU -->|Zero| Zero_Flag

    %% ========== 访存阶段 ==========
    ALU_result -->|addr| DMEM[Data Memory<br>数据存储器]
    DMEM -->|rdata| WB_MUX

    %% ========== 写回阶段 ==========
    ALU_result -->|0: ALU Result| WB_MUX{MemtoReg MUX}
    DMEM -->|1: MEM Read Data| WB_MUX
    WB_MUX -->|wdata| RF_wdata_in

    %% ========== PC 更新逻辑 ==========
    PC_Plus4["PC+4<br>Adder"] -->|pc+4| PC_src_MUX{PC Source MUX}
    Branch_Target["Branch Target<br>Adder"] -->|target| PC_src_MUX
    PC_src_MUX -->|next_pc| PC

    %% ========== 分支判断 ==========
    Zero_Flag --> PC_src_Logic[Branch Logic]
    CU -->|Branch| PC_src_Logic
    PC_src_Logic -->|PC Source| PC_src_sel

    %% ========== 测试接口 ==========
    PC -->|debug_wb_pc| DEBUG
    RF_wen -->|debug_wb_rf_wen| DEBUG
    IMEM -->|4:0 rd| debug_wb_rf_wnum[debug_wb_rf_wnum]
    debug_wb_rf_wnum -->|debug_wb_rf_wnum| DEBUG
    RF_wdata_in -->|debug_wb_rf_wdata| DEBUG

    %% ========== 样式 ==========
    classDef controlPath fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef dataPath fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef memory fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef debug fill:#ccff90,stroke:#33691e,stroke-width:2px

    class CU,ALU_op,ALU_src2_sel,WB_MUX_sel,PC_src_sel,DMEM_wen,DMEM_ren,PC_src_Logic controlPath
    class PC,IMEM,RF,DMEM memory
    class ALU,PC_Plus4,Branch_Target,IG,ALU_src1_MUX,ALU_src2_MUX,WB_MUX,PC_src_MUX dataPath
    class DEBUG,debug_wb_rf_wnum debug