# 8 位多寄存器自定义指令集 CPU 技术方案

> **适用场景：** 数字电路课程设计  
> **仿真平台：** Multisim  
> **仿真主体：** 完整 8 位简化 CPU  
> **实物主体：** 4 位 ALU 核心模块

---

## 1. 项目定位

本项目设计一款 **8 位多寄存器自定义指令集 CPU**，作为数字电路课程设计中的 Multisim 仿真主体。

CPU 参考现代处理器的基本数据通路，包含：

- 程序计数器 PC；
- 指令存储器 ROM；
- 控制单元；
- 通用寄存器堆；
- ALU；
- 数据 RAM；
- 状态标志；
- 内存映射 LED 输出。

本项目不直接实现 RISC-V 指令集，而是设计一套适合数字电路课程设计规模的自定义指令集。这样既能体现处理器的基本架构思想，又能控制电路规模和搭建难度。

### 1.1 仿真部分

Multisim 中实现完整的 8 位简化 CPU，包括：

```text
PC → Instruction ROM → Control Unit
   → Register File → ALU → RAM / LED_IO
```

### 1.2 实物部分

实物部分不实现完整 CPU，只实现其中最核心、最适合面包板或 PCB 落地的 **4 位 ALU**。

实物 ALU 对应仿真 CPU 中 8 位 ALU 的低 4 位切片，用于验证：

- ADD；
- SUB；
- AND；
- OR；
- Zero；
- Carry/Borrow。

### 1.3 推荐题目名称

> 基于 Multisim 的 8 位多寄存器自定义指令集 CPU 仿真与 4 位 ALU 实物验证

或：

> 一种 8 位简化类 RISC-V CPU 的仿真设计及其 ALU 核心模块实物实现

---

## 2. 设计范围划分

### 2.1 Multisim 仿真部分

仿真部分需要完成：

1. 8 位数据通路；
2. 16 位自定义指令格式；
3. 8 个通用寄存器 R0～R7；
4. ADD、SUB、AND、OR 四种 ALU 运算；
5. Zero、Borrow 状态标志；
6. JMP、BZ、BB 跳转控制；
7. 16×8 数据 RAM；
8. 内存映射 LED 输出；
9. 运行 GCD 程序并输出结果。

### 2.2 实物部分

实物部分只实现 **4 位 ALU**。

实物不包括：

- PC；
- 指令 ROM；
- 完整寄存器堆；
- 控制单元；
- 数据 RAM；
- GCD 程序执行流程。

实物包括：

1. 4 位输入 `A[3:0]`；
2. 4 位输入 `B[3:0]`；
3. 2 位运算选择 `OP[1:0]`；
4. ADD、SUB、AND、OR；
5. 4 位结果输出 `F[3:0]`；
6. Carry/Borrow 标志；
7. Zero 标志；
8. LED 显示结果与标志位。

---

## 3. 总体规格

### 3.1 Multisim 仿真 CPU 规格

| 项目 | 规格 |
|---|---|
| 数据宽度 | 8 bit |
| 指令宽度 | 16 bit |
| 寄存器数量 | 8 个通用寄存器，R0～R7 |
| 特殊寄存器 | R0 恒为 0 |
| PC 宽度 | 4 bit，支持 16 条指令 |
| 指令存储器 | 16×16 bit ROM |
| 数据存储器 | 16×8 bit RAM |
| RAM 地址范围 | `0x00～0x0F` |
| RAM 地址使用方式 | CPU 保留 8 位地址计算，RAM 只译码低 4 位 |
| ALU 功能 | ADD / SUB / AND / OR |
| 状态标志 | Zero、Borrow |
| I/O | 内存映射 LED 输出 |
| LED_IO 地址 | `0xF0` |
| 执行方式 | 单周期或简化多周期，推荐单周期展示 |
| 验证程序 | 辗转相减法求最大公约数 GCD |

### 3.2 实物 ALU 规格

| 项目 | 规格 |
|---|---|
| 数据宽度 | 4 bit |
| 输入 | `A[3:0]`、`B[3:0]` |
| 运算选择 | `OP[1:0]` |
| 支持运算 | ADD / SUB / AND / OR |
| 输出 | `F[3:0]` |
| 标志位 | Zero、Carry/Borrow |
| 显示方式 | LED 显示结果和标志位 |
| 实现方式 | DIP 直插 74HC/74LS 系列芯片 |
| 对应关系 | 8 位仿真 ALU 的低 4 位切片 |

---

## 4. 总体结构

### 4.1 仿真 CPU 总体结构

```text
┌──────────────┐
│      PC      │
│    4 bit     │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Instruction  │
│  ROM 16×16   │
└──────┬───────┘
       │ inst[15:0]
       ▼
┌──────────────────┐
│   Control Unit   │
│  opcode decoder  │
└──────┬───────────┘
       │ control signals
       ▼
┌────────────────────────────────────┐
│           Register File            │
│       R0～R7, each 8 bit            │
│       R0 is fixed to zero          │
└──────────┬──────────────┬──────────┘
           │              │
           ▼              ▼
       ┌──────────────────┐
       │       ALU        │
       │ ADD/SUB/AND/OR   │
       └────────┬─────────┘
                │
        ┌───────┴────────┐
        ▼                ▼
┌──────────────┐  ┌──────────────┐
│ Data RAM     │  │ LED_IO Reg   │
│ 16×8         │  │ addr = 0xF0  │
└──────────────┘  └──────────────┘
```

### 4.2 实物 ALU 结构

```text
A[3:0] ─────────┐
                ▼
          ┌───────────┐
B[3:0] ─▶ │ 4-bit ALU │ ─▶ F[3:0] ─▶ LED
          │           │ ─▶ Zero ─────▶ LED
OP[1:0] ─▶│           │ ─▶ Carry/
          └───────────┘    Borrow ───▶ LED
```

---

## 5. 指令集设计

| opcode | 指令 | 功能 |
|---|---|---|
| `0000` | `NOP` | 空操作，可选 |
| `0001` | `LI rd, imm8` | `rd ← imm8` |
| `0010` | `LD rd, [base + off]` | `rd ← RAM[(base + off) & 0x0F]` |
| `0011` | `ST rs, [base + off]` | 地址为 `0xF0` 时写 LED_IO，否则写 RAM |
| `0100` | `ADD rd, rs1, rs2` | `rd ← rs1 + rs2` |
| `0101` | `SUB rd, rs1, rs2` | `rd ← rs1 - rs2`，更新 Zero/Borrow |
| `0110` | `AND rd, rs1, rs2` | `rd ← rs1 & rs2` |
| `0111` | `OR rd, rs1, rs2` | `rd ← rs1 \| rs2` |
| `1000` | `JMP addr` | `PC ← addr[3:0]` |
| `1001` | `BZ addr` | 若 `Zero=1`，则跳转 |
| `1010` | `BB addr` | 若 `Borrow=1`，则跳转 |

### 5.1 标志位定义

```text
Zero = 1，当 ALU 结果为 0
Borrow = 1，当无符号 SUB 中 rs1 < rs2
```

### 5.2 存储地址规则

CPU 内部仍然进行 8 位地址计算：

```text
addr8 = Reg[base] + offset6
```

普通 RAM 只使用低 4 位地址：

```text
RAM_addr = addr8[3:0]
```

LED_IO 使用独立特殊地址：

```text
0xF0 : LED_OUT
```

---

## 6. 指令格式

### 6.1 R 型指令

适用于：

```asm
ADD rd, rs1, rs2
SUB rd, rs1, rs2
AND rd, rs1, rs2
OR  rd, rs1, rs2
```

格式：

```text
[15:12] opcode
[11:9]  rd
[8:6]   rs1
[5:3]   rs2
[2:0]   unused
```

编码：

```text
inst = opcode << 12 | rd << 9 | rs1 << 6 | rs2 << 3
```

### 6.2 I 型指令

适用于：

```asm
LI rd, imm8
```

格式：

```text
[15:12] opcode
[11:9]  rd
[8]     unused
[7:0]   imm8
```

编码：

```text
inst = opcode << 12 | rd << 9 | imm8
```

### 6.3 M 型指令

适用于：

```asm
LD rd, [base + off]
ST rs, [base + off]
```

格式：

```text
[15:12] opcode
[11:9]  rA
[8:6]   base
[5:0]   offset6
```

编码：

```text
inst = opcode << 12 | rA << 9 | base << 6 | offset6
```

地址计算：

```text
addr8 = Reg[base] + offset6
```

访问规则：

```text
if ST and addr8 == 0xF0:
    LED_OUT = Reg[rs]
else if ST:
    RAM[addr8[3:0]] = Reg[rs]

if LD:
    Reg[rd] = RAM[addr8[3:0]]
```

### 6.4 J/B 型指令

适用于：

```asm
JMP addr
BZ  addr
BB  addr
```

格式：

```text
[15:12] opcode
[11:8]  unused
[7:0]   addr
```

PC 只使用地址低 4 位：

```text
PC_next = addr[3:0]
```

编码：

```text
inst = opcode << 12 | addr
```

---

## 7. 核心模块设计

### 7.1 PC 模块

正常情况：

```text
PC_next = PC + 1
```

跳转情况：

```text
PC_next = branch_target[3:0]
```

| 情况 | PC_next |
|---|---|
| 普通指令 | `PC + 1` |
| JMP | `addr[3:0]` |
| BZ 且 Zero=1 | `addr[3:0]` |
| BB 且 Borrow=1 | `addr[3:0]` |
| 条件不满足 | `PC + 1` |

### 7.2 指令 ROM

```text
宽度：16 bit
深度：16 words
地址：PC[3:0]
输出：inst[15:0]
```

ROM 中存放手写机器码。

### 7.3 寄存器堆

```text
R0 = 0
R1～R7 = general purpose registers
```

| 寄存器 | 推荐用途 |
|---|---|
| R0 | 恒为 0 |
| R1 | 算法变量 a |
| R2 | 算法变量 b |
| R3 | 临时结果 diff |
| R4 | 循环计数器，可选 |
| R5 | 数据指针，可选 |
| R6 | 常数 1，可选 |
| R7 | I/O 基地址 `0xF0` |

写入规则：

```text
if RegWrite and rd != 0:
    Reg[rd] = write_data
```

### 7.4 ALU

仿真 CPU 中使用 8 位 ALU：

```text
ADD
SUB
AND
OR
```

输出：

```text
ALU_out[7:0]
Zero
Borrow
```

标志位：

```text
Zero = (ALU_out == 0)
Borrow = SUB 时发生借位
```

对于无符号减法：

```text
若 rs1 < rs2，则 Borrow = 1
```

实物部分只实现 4 位：

```text
A[3:0], B[3:0] → F[3:0]
```

### 7.5 数据 RAM 与 LED_IO

RAM 固定为：

```text
16×8 bit
```

地址范围：

```text
RAM[0x0]～RAM[0xF]
```

写入规则：

```text
if ST and addr8 == 0xF0:
    LED_OUT = write_data
else if ST:
    RAM[addr8[3:0]] = write_data
```

读出规则：

```text
if LD:
    read_data = RAM[addr8[3:0]]
```

---

## 8. Multisim 器件修改方案

根据 Multisim 中可用器件，采用以下替换：

| 原设想 | 最终方案 | 注意事项 |
|---|---|---|
| 74LS154 | 74HC154 | 输出低有效 |
| 74LS150 | 74150N | 输出通常反相 |
| 74LS688 | 基本门电路译码 | 用于识别 `0xF0` |
| 32×16 ROM | 16×16 ROM | 与 4 bit PC 配套 |
| 5 bit PC | 4 bit PC | 跳转只使用 `addr[3:0]` |
| 256×8 RAM | 16×8 RAM | 只实现低 16 字节 |

### 8.1 PC 改为 4 bit

```text
PC[3:0]
```

正常执行：

```text
PC_next = PC + 1
```

跳转：

```text
PC_next = inst[3:0]
```

### 8.2 ROM 固定为 16×16

推荐结构：

```text
PC[3:0]
   │
   ▼
74HC154
   │
   ├── /Y0  → ROM 第 0 行
   ├── /Y1  → ROM 第 1 行
   ├── ...
   └── /Y15 → ROM 第 15 行
   │
   ▼
二极管矩阵 / 固定接线矩阵
   │
   ▼
inst[15:0]
```

### 8.3 74HC154 的低有效问题

```text
选中第 i 行：/Yi = 0
未选中第 i 行：/Yi = 1
```

若后级需要高有效：

```text
Select_i = NOT(/Yi)
```

写使能：

```text
WE_i = MemWrite AND Select_i
```

### 8.4 74150N 读出

16×8 RAM 每一位需要一片 16 选 1 多路选择器：

```text
bit0：74150N × 1
bit1：74150N × 1
...
bit7：74150N × 1
```

总计：

```text
74150N × 8
```

若输出反相：

```text
74150N 输出 → 74HC04 → RAM_read_data[i]
```

### 8.5 0xF0 门电路译码

```text
0xF0 = 1111_0000
```

地址译码：

```text
IOAddr =
A7 & A6 & A5 & A4 &
~A3 & ~A2 & ~A1 & ~A0
```

写使能：

```text
IOWrite  = MemWrite & IOAddr
RAMWrite = MemWrite & ~IOAddr
```

---

## 9. 16×8 RAM 的替代实现

若没有合适的现成 RAM，可采用：

```text
寄存器阵列 + 译码器 + 多路选择器
```

### 9.1 RAM 写入结构

16×8 RAM 可视为 16 个 8 位寄存器：

```text
RAM[0]  = 8-bit Register
RAM[1]  = 8-bit Register
...
RAM[15] = 8-bit Register
```

地址：

```text
addr[3:0]
```

译码：

```text
addr[3:0] → 74HC154
```

高有效行选择：

```text
Select_i = NOT(/Yi)
```

写使能：

```text
WE_i = RAMWrite & Select_i
```

所有寄存器的数据输入连接同一条 `write_data[7:0]` 总线，只有被选中的寄存器在时钟沿写入。

### 9.2 RAM 读出结构

每个数据位使用一个 16 选 1 多路选择器：

```text
16 个寄存器的 bit0 → 74150N → read_data[0]
16 个寄存器的 bit1 → 74150N → read_data[1]
...
16 个寄存器的 bit7 → 74150N → read_data[7]
```

### 9.3 RAM 替代方案框图

```text
                  addr[3:0]
                      │
             ┌────────┴────────┐
             ▼                 ▼
         74HC154          74150N × 8
       写地址译码          读数据选择
             │                 │
             ▼                 ▼
    16 个 8 位寄存器      read_data[7:0]
             ▲
             │
      write_data[7:0]
```

---

## 10. 控制信号设计

| 控制信号 | 作用 |
|---|---|
| RegWrite | 是否写寄存器 |
| MemRead | 是否读 RAM |
| MemWrite | 是否执行存储写操作 |
| RAMWrite | 是否写普通 RAM |
| IOWrite | 是否写 LED_IO |
| ALUOp | ALU 运算类型 |
| WBSel | 写回数据来源 |
| PCSel | PC 下一值选择 |
| FlagWrite | 是否更新 Zero/Borrow |

### 10.1 各指令控制信号

| 指令 | RegWrite | MemRead | MemWrite | ALUOp | WBSel | FlagWrite | PCSel |
|---|---:|---:|---:|---|---|---:|---|
| NOP | 0 | 0 | 0 | X | X | 0 | PC+1 |
| LI | 1 | 0 | 0 | X | IMM | 0 | PC+1 |
| LD | 1 | 1 | 0 | ADD | MEM | 0 | PC+1 |
| ST | 0 | 0 | 1 | ADD | X | 0 | PC+1 |
| ADD | 1 | 0 | 0 | ADD | ALU | 1 | PC+1 |
| SUB | 1 | 0 | 0 | SUB | ALU | 1 | PC+1 |
| AND | 1 | 0 | 0 | AND | ALU | 1 | PC+1 |
| OR | 1 | 0 | 0 | OR | ALU | 1 | PC+1 |
| JMP | 0 | 0 | 0 | X | X | 0 | addr |
| BZ | 0 | 0 | 0 | X | X | 0 | Zero ? addr : PC+1 |
| BB | 0 | 0 | 0 | X | X | 0 | Borrow ? addr : PC+1 |

---

## 11. GCD 验证程序

### 11.1 功能目标

计算：

```text
gcd(12, 8) = 4
```

算法：

```text
while a != b:
    if a > b:
        a = a - b
    else:
        b = b - a

output a
```

验证内容：

- LI；
- SUB；
- ADD；
- Zero；
- Borrow；
- BZ；
- BB；
- JMP；
- ST；
- LED_IO。

### 11.2 汇编程序

```asm
; Program: gcd(12, 8)
; Result : 4
; LED_IO address = 0xF0

0:  LI   R1, 12
1:  LI   R2, 8
2:  LI   R7, 240

3: LOOP:
    SUB  R3, R1, R2
4:  BZ   DONE
5:  BB   B_GREATER

6:  ADD  R1, R3, R0
7:  JMP  LOOP

8: B_GREATER:
    SUB  R2, R2, R1
9:  JMP  LOOP

10: DONE:
    ST   R1, [R7 + 0]
11: JMP  DONE
```

### 11.3 执行过程

初始：

```text
R1 = 12
R2 = 8
```

第一次循环：

```text
R3 = 12 - 8 = 4
Zero = 0
Borrow = 0
R1 = 4
```

第二次循环：

```text
R3 = 4 - 8
Borrow = 1
R2 = 8 - 4 = 4
```

第三次循环：

```text
R3 = 4 - 4 = 0
Zero = 1
```

最终：

```text
LED_OUT = R1 = 4
LED_OUT = 00000100
```

### 11.4 对应机器码

| 地址 | 汇编 | 机器码 |
|---:|---|---|
| 0 | `LI R1, 12` | `0x120C` |
| 1 | `LI R2, 8` | `0x1408` |
| 2 | `LI R7, 240` | `0x1EF0` |
| 3 | `SUB R3, R1, R2` | `0x5650` |
| 4 | `BZ 10` | `0x900A` |
| 5 | `BB 8` | `0xA008` |
| 6 | `ADD R1, R3, R0` | `0x42C0` |
| 7 | `JMP 3` | `0x8003` |
| 8 | `SUB R2, R2, R1` | `0x5488` |
| 9 | `JMP 3` | `0x8003` |
| 10 | `ST R1, [R7+0]` | `0x33C0` |
| 11 | `JMP 10` | `0x800A` |

ROM 初始化内容：

```text
120C
1408
1EF0
5650
900A
A008
42C0
8003
5488
8003
33C0
800A
```

---

## 12. 16×8 RAM 访存测试程序

### 12.1 写 RAM 再读出

示例：

```asm
LI R1, 0x25
LI R4, 0x03
ST R1, [R4 + 0]
LD R2, [R4 + 0]
```

预期：

```text
RAM[3] = 0x25
R2 = 0x25
```

### 12.2 RAM 地址范围

RAM 实际只使用地址低 4 位。

例如：

```text
addr8 = 0x13
RAM_addr = 0x3
```

因此实际访问：

```text
RAM[3]
```

---

## 13. Multisim 搭建建议

建议分阶段实现。

### 阶段 1：4 bit PC + 16×16 ROM

目标：

- PC 在时钟驱动下递增；
- 74HC154 正确选择 ROM 行；
- ROM 输出对应机器码。

### 阶段 2：LI + 寄存器写回

测试：

```asm
LI R1, 12
LI R2, 8
```

预期：

```text
R1 = 12
R2 = 8
```

### 阶段 3：ADD/SUB + 标志位

测试：

```asm
LI  R1, 12
LI  R2, 8
SUB R3, R1, R2
```

预期：

```text
R3 = 4
Zero = 0
Borrow = 0
```

### 阶段 4：JMP/BZ/BB

目标：

- 实现无条件跳转；
- 实现 Zero 条件跳转；
- 实现 Borrow 条件跳转；
- 能执行循环。

### 阶段 5：16×8 RAM

目标：

- ST 能写 RAM；
- LD 能读 RAM；
- RAM 只使用低 4 位地址。

### 阶段 6：ST + LED_IO

测试：

```asm
LI R1, 4
LI R7, 240
ST R1, [R7 + 0]
```

预期：

```text
LED_OUT = 00000100
```

### 阶段 7：运行完整 GCD

目标：

- ROM 装入完整机器码；
- 单步观察 PC、R1、R2、R3、Zero、Borrow；
- 最终 LED 输出 4。

---

## 14. 实物部分对应方案：4 位 ALU

### 14.1 功能

```text
A[3:0]
B[3:0]
OP[1:0]
F[3:0]
Zero
Carry/Borrow
```

支持：

```text
ADD
SUB
AND
OR
```

### 14.2 推荐器件思路

可使用：

- 74HC283 / 74LS283：4 位加法；
- 74HC86 / 74LS86：异或，用于减法控制；
- 74HC08 / 74LS08：与；
- 74HC32 / 74LS32：或；
- 74HC157 / 74LS157：结果选择；
- 74HC04 / 74LS04：反相；
- LED 与限流电阻：结果和标志显示。

### 14.3 答辩说明

> 仿真部分实现完整 8 位多寄存器 CPU；实物部分选取其中最核心的 ALU 数据通路进行硬件验证。由于 8 位 ALU 可由两个 4 位 ALU 级联构成，因此实物 4 位 ALU 可以视为仿真 CPU 中 ALU 模块的低位切片。

---

## 15. 报告建议目录

1. 设计背景与目标；
2. 总体方案设计；
3. 仿真范围与实物范围；
4. 自定义指令集设计；
5. CPU 数据通路设计；
6. 控制单元设计；
7. PC 与指令 ROM；
8. 寄存器堆；
9. ALU 与标志位；
10. 16×8 RAM；
11. 内存映射 LED_IO；
12. GCD 程序与机器码；
13. Multisim 仿真结果；
14. 4 位 ALU 实物设计；
15. 测试与分析；
16. 总结与改进。

---

## 16. 答辩展示建议

建议展示顺序：

1. 展示整体框图；
2. 说明仿真 CPU 与实物 ALU 的范围区别；
3. 展示指令集；
4. 展示 PC 和 ROM 取指；
5. 单步运行 GCD；
6. 观察 R1、R2、R3、Zero、Borrow；
7. 展示 LED 输出 `00000100`；
8. 展示 4 位 ALU 实物；
9. 说明实物 ALU 与仿真 8 位 ALU 的对应关系。

核心表述：

> 本设计不是将输入直接送入组合逻辑得到结果，而是由存放在 ROM 中的机器指令驱动 CPU 数据通路逐条运行。CPU 完成取指、译码、执行、写回和条件跳转，最终通过内存映射 I/O 将 GCD 结果输出到 LED。

---

## 17. 后续扩展方向

可以继续增加：

1. XOR 指令；
2. 拨码开关输入端口；
3. 七段数码管显示；
4. 自动运行与单步运行切换；
5. 汇编到机器码的自动转换脚本；
6. MAC 乘加外设；
7. 更大的 ROM 和 RAM；
8. 多周期控制器；
9. 中断或简单外设接口。

其中最有价值的扩展是：

```text
通用 CPU + 专用 MAC 加速器
```

这样可以进一步体现通用处理器与专用数字电路协同工作的思想。
