# RISC-V 12条指令实现详解

## 指令编码表分析

根据你提供的指令编码表，这12条指令可以分为4类：

### 1. R型指令 (寄存器-寄存器运算)
```
指令: add_w, sub_w, and, mul_w
格式: [31:25]funct7 [24:20]rk [19:15]rj [14:10]funct3 [9:5]rd [4:0]opcode
特点: 
- opcode = 0011011 (add_w, sub_w, and, mul_w)
- 两个源寄存器，一个目标寄存器
- 结果写回寄存器堆
```

### 2. I型指令 (立即数运算)
```
指令: slli_w, slti, addi_w, ori, ld_w
格式: [31:20]imm [19:15]rj [14:12]funct3 [11:7]rd [6:0]opcode
特点:
- slli_w: 左移立即数位 (opcode = 0010010)
- slti: 立即数比较 (opcode = 0010010) 
- addi_w: 立即数加法 (opcode = 0010010)
- ori: 立即数或运算 (opcode = 0010010)
- ld_w: 加载字 (opcode = 0101000)
```

### 3. S型指令 (存储指令)
```
指令: st_w
格式: [31:25]imm[11:5] [24:20]rk [19:15]rj [14:12]funct3 [11:7]imm[4:0] [6:0]opcode
特点:
- opcode = 0101001
- 地址 = rj + 符号扩展立即数
- 数据 = rk寄存器值
```

### 4. B型指令 (分支指令)
```
指令: b, beq
格式: [31:26]imm[15:10] [25:20]rj [19:15]rk [14:10]imm[9:5] [9:5]imm[25:16] [4:0]opcode
特点:
- b: 无条件跳转 (opcode = 0101000)
- beq: 相等分支 (opcode = 0101100)
```

## 数据通路实现要点

### 1. 控制单元设计
```verilog
// 控制信号生成
always_comb begin
    case(opcode)
        7'b0011011: begin // R型
            alu_op = funct3;
            gr_we = 1'b1;
            mem_read = 1'b0;
            mem_write = 1'b0;
            branch = 1'b0;
        end
        7'b0010010: begin // I型(非ld)
            alu_op = funct3;
            gr_we = 1'b1;
            mem_read = 1'b0;
            mem_write = 1'b0;
            branch = 1'b0;
        end
        7'b0101000: begin // ld_w
            alu_op = 3'b000; // ADD
            gr_we = 1'b1;
            mem_read = 1'b1;
            mem_write = 1'b0;
            branch = 1'b0;
        end
        7'b0101001: begin // st_w
            alu_op = 3'b000; // ADD
            gr_we = 1'b0;
            mem_read = 1'b0;
            mem_write = 1'b1;
            branch = 1'b0;
        end
        7'b0101100: begin // beq
            alu_op = 3'b001; // SUB
            gr_we = 1'b0;
            mem_read = 1'b0;
            mem_write = 1'b0;
            branch = 1'b1;
        end
    endcase
end
```

### 2. ALU设计
```verilog
// ALU运算单元
always_comb begin
    case(alu_op)
        3'b000: alu_result = src1 + src2;        // ADD
        3'b001: alu_result = src1 - src2;        // SUB
        3'b010: alu_result = src1 << src2[4:0];  // SLL
        3'b011: alu_result = ($signed(src1) < $signed(src2)) ? 1 : 0; // SLT
        3'b100: alu_result = src1 ^ src2;        // XOR
        3'b101: alu_result = src1 >> src2[4:0];  // SRL
        3'b110: alu_result = src1 | src2;        // OR
        3'b111: alu_result = src1 & src2;        // AND
    endcase
end
```

### 3. 立即数生成器
```verilog
// 立即数扩展
always_comb begin
    case(inst_type)
        I_TYPE: imm = {{20{inst[31]}}, inst[31:20]};
        S_TYPE: imm = {{20{inst[31]}}, inst[31:25], inst[11:7]};
        B_TYPE: imm = {{19{inst[31]}}, inst[31], inst[7], inst[30:25], inst[11:8], 1'b0};
        default: imm = 32'b0;
    endcase
end
```

### 4. 分支判断单元
```verilog
// 分支条件判断
always_comb begin
    case(branch_type)
        2'b00: branch_taken = 1'b0;              // 无分支
        2'b01: branch_taken = 1'b1;              // b指令
        2'b10: branch_taken = (rj_value == rk_value); // beq指令
        default: branch_taken = 1'b0;
    endcase
end
```

## 指令执行流程

### 典型R型指令 (add_w rd, rj, rk)
1. **IF**: 取指令
2. **ID**: 译码，读取rj和rk寄存器
3. **EX**: ALU执行加法运算
4. **MEM**: 跳过存储器访问
5. **WB**: 结果写回rd寄存器

### 典型I型指令 (addi_w rd, rj, imm)
1. **IF**: 取指令
2. **ID**: 译码，读取rj寄存器，生成立即数
3. **EX**: ALU执行rj + imm
4. **MEM**: 跳过存储器访问
5. **WB**: 结果写回rd寄存器

### 加载指令 (ld_w rd, rj, imm)
1. **IF**: 取指令
2. **ID**: 译码，读取rj寄存器，生成立即数
3. **EX**: ALU计算地址 = rj + imm
4. **MEM**: 从数据存储器读取数据
5. **WB**: 数据写回rd寄存器

### 存储指令 (st_w rk, rj, imm)
1. **IF**: 取指令
2. **ID**: 译码，读取rj和rk寄存器，生成立即数
3. **EX**: ALU计算地址 = rj + imm
4. **MEM**: 将rk值写入数据存储器
5. **WB**: 无写回操作

### 分支指令 (beq rj, rk, offset)
1. **IF**: 取指令
2. **ID**: 译码，读取rj和rk寄存器，比较，计算分支目标
3. **EX**: 确定是否跳转
4. **MEM**: 更新PC值
5. **WB**: 无写回操作

## 数据前推实现

为了解决数据相关问题，需要实现数据前推机制：

```verilog
// 数据前推逻辑
always_comb begin
    // EX阶段前推
    if (ex_mem_gr_we && (ex_mem_rd != 0) && (ex_mem_rd == id_ex_rj))
        forward_a = 2'b10; // 从EX/MEM寄存器前推
    else if (mem_wb_gr_we && (mem_wb_rd != 0) && (mem_wb_rd == id_ex_rj))
        forward_a = 2'b01; // 从MEM/WB寄存器前推
    else
        forward_a = 2'b00; // 无前推
    
    // 类似的逻辑用于rk寄存器
end
```

这样就能完整支持你要求的12条RISC-V指令了！