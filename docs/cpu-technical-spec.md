# 8 位多寄存器自定义指令集 CPU 技术方案

> **适用场景：** 数字电路课程设计  
> **仿真平台：** NI Multisim  
> **仿真主体：** 8 位多寄存器简化 CPU  
> **实物主体：** 4 位 ACC＋ALU 系统  
> **最终方案：** CPU 不设置数据 RAM，运算数据全部保存在寄存器中

---

## 1. 项目定位

本项目设计一款适合数字电路课程设计规模的 **8 位多寄存器自定义指令集 CPU**，并在实物部分实现 **4 位 ACC＋ALU 系统**。

仿真 CPU 用于展示处理器的基本组成和指令执行过程，包括：

- 程序计数器 PC；
- 指令存储器 ROM；
- 指令译码与控制单元；
- 通用寄存器组；
- 算术逻辑单元 ALU；
- Zero、Borrow 状态标志；
- 条件跳转逻辑；
- LED 输出寄存器。

本设计不直接实现 RISC-V，也不设置数据 RAM。这样既能体现 CPU 的基本数据通路，又能控制 Multisim 电路规模与调试难度。

### 1.1 仿真部分

```text
PC → Instruction ROM → Control Unit
                   ↓
            Register File → ALU → Register Write-back
                   │          │
                   │          └→ Zero / Borrow → Jump Logic
                   └────────────→ LED_OUT（ST）
```

### 1.2 实物部分

实物部分不实现完整 CPU，而是在 4 位 ALU 基础上加入：

- ACC 累加寄存器；
- 外部输入与 ACC 反馈选择；
- 手动时钟；
- 按键消抖与脉冲整形；
- LED 显示与驱动。

```text
外部输入 ──┐
           ├──→ MUX ──→ 4-bit ALU ──→ ACC ──→ LED
ACC反馈 ───┘                         │
                                    └──反馈到下一次运算
```

### 1.3 推荐题目名称

> 基于 Multisim 的 8 位多寄存器自定义指令集 CPU 仿真与 4 位 ACC＋ALU 实物设计

---

## 2. 设计范围

### 2.1 Multisim 仿真部分

仿真部分完成：

1. 8 位数据通路；
2. 16 位自定义指令格式；
3. 8 个通用寄存器 R0～R7；
4. ADD、SUB、AND、OR 四种 ALU 运算；
5. Zero、Borrow 状态标志；
6. JMP、BZ、BB 跳转控制；
7. `ST` 指令写 LED 输出寄存器；
8. GCD 程序运行与结果显示。

仿真部分不包含：

- 数据 RAM；
- `LD` 访存指令；
- 普通存储地址译码；
- 数据 Cache 等复杂结构。

### 2.2 ACC＋ALU 实物部分

实物部分完成：

1. 4 位算术与逻辑运算；
2. 4 位 ACC 数据保存；
3. ACC 结果反馈；
4. 外部输入与反馈输入选择；
5. 手动时钟控制；
6. 按键消抖与脉冲整形；
7. LED 数据与状态显示；
8. PCB、焊接和实物调试。

实物部分不包含：

- PC；
- 指令 ROM；
- 完整通用寄存器组；
- CPU 控制单元；
- GCD 指令执行过程。

---

## 3. 总体规格

### 3.1 Multisim CPU 规格

| 项目 | 规格 |
|---|---|
| 数据宽度 | 8 bit |
| 指令宽度 | 16 bit |
| 寄存器数量 | 8 个，R0～R7 |
| 特殊寄存器 | R0 固定为 0 |
| PC 宽度 | 4 bit |
| 指令存储器 | 16×16 bit ROM |
| 数据 RAM | 不使用 |
| ALU 功能 | ADD、SUB、AND、OR |
| 状态标志 | Zero、Borrow |
| 跳转 | JMP、BZ、BB |
| 输出 | `ST` 写 LED_OUT |
| 执行方式 | 简化单周期展示 |
| 验证程序 | 辗转相减法 GCD |

### 3.2 ACC＋ALU 实物规格

| 项目 | 规格 |
|---|---|
| 数据宽度 | 4 bit |
| 基本输入 | 外部 4 位数据 |
| 反馈输入 | ACC[3:0] |
| 运算选择 | 按最终 ALU 原理图设置 |
| ACC | 74LS175 |
| 数据选择 | 74LS157 |
| 时钟整形 | 74LS14 |
| 缓冲驱动 | 74LS07 |
| 显示 | LED |
| 实现方式 | DIP 直插 74LS 系列器件与 PCB |

---

## 4. CPU 总体结构

```text
┌──────────────┐
│      PC      │
│    4 bit     │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ ROM 16×16    │
└──────┬───────┘
       │ inst[15:0]
       ▼
┌──────────────────┐
│ Control Unit     │
└──────┬───────────┘
       │ control signals
       ▼
┌──────────────────────────────┐
│ Register File R0～R7, 8 bit  │
└──────────┬───────────┬───────┘
           │           │
           ▼           ▼
       ┌──────────────────┐
       │       ALU        │
       │ ADD/SUB/AND/OR   │
       └───────┬──────────┘
               │
        ┌──────┴───────────┐
        ▼                  ▼
 Register Write-back    LED_OUT
                       （ST 写入）
```

CPU 不设置数据 RAM。所有中间数据均由 R0～R7 保存。

---

## 5. 指令集设计

| opcode | 指令 | 功能 |
|---|---|---|
| `0000` | `NOP` | 空操作 |
| `0001` | `LI rd, imm8` | `rd ← imm8` |
| `0010` | 保留 | 当前方案不使用 |
| `0011` | `ST rs` | `LED_OUT ← Reg[rs]` |
| `0100` | `ADD rd, rs1, rs2` | `rd ← rs1 + rs2` |
| `0101` | `SUB rd, rs1, rs2` | `rd ← rs1 - rs2`，更新标志 |
| `0110` | `AND rd, rs1, rs2` | `rd ← rs1 & rs2` |
| `0111` | `OR rd, rs1, rs2` | `rd ← rs1 \| rs2` |
| `1000` | `JMP addr` | `PC ← addr[3:0]` |
| `1001` | `BZ addr` | `Zero=1` 时跳转 |
| `1010` | `BB addr` | `Borrow=1` 时跳转 |
| `1011～1111` | 保留 | 当前方案不使用 |

### 5.1 状态标志

```text
Zero = 1，当 SUB 或指定 ALU 运算结果为 0
Borrow = 1，当无符号 SUB 中 rs1 < rs2
```

GCD 程序中，Zero 用于判断两个数是否相等，Borrow 用于判断大小关系。

### 5.2 跳转判断

```text
isJMP = (opcode == 1000)
isBZ  = (opcode == 1001)
isBB  = (opcode == 1010)

BZ_take = isBZ AND Zero
BB_take = isBB AND Borrow

JumpEnable = isJMP OR BZ_take OR BB_take
```

PC 下一值：

```text
PC_plus_1 = PC + 1

PC_next =
    jump_addr[3:0], 当 JumpEnable = 1
    PC_plus_1,      当 JumpEnable = 0
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
inst = (opcode << 12) | (rd << 9) | (rs1 << 6) | (rs2 << 3)
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
inst = (opcode << 12) | (rd << 9) | imm8
```

### 6.3 S 型输出指令

适用于：

```asm
ST rs
```

格式：

```text
[15:12] opcode
[11:9]  rs
[8:0]   unused
```

执行：

```text
if opcode == ST:
    LED_OUT ← Reg[rs]
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

PC 只使用目标地址低 4 位：

```text
PC_next = addr[3:0]
```

---

## 7. 主要模块

### 7.1 程序计数器 PC

- 宽度：4 bit；
- 复位后为 0；
- 正常执行时加 1；
- 跳转成立时装载目标地址；
- 通过 MUX 在 `PC+1` 与 `jump_addr` 之间选择。

### 7.2 指令 ROM

- 深度：16 words；
- 数据宽度：16 bit；
- 地址输入：PC[3:0]；
- 数据输出：inst[15:0]；
- 保存 GCD 测试程序。

### 7.3 通用寄存器组

- 8 个 8 位寄存器；
- R0 固定输出 0；
- 支持两个读端口；
- 支持一个写端口；
- `LI` 和 ALU 指令可写回；
- 中间数据全部保存在寄存器中。

### 7.4 ALU

输入：

```text
A = Reg[rs1]
B = Reg[rs2]
```

输出：

```text
ADD: A + B
SUB: A - B
AND: A & B
OR : A | B
```

状态：

```text
Zero = (result == 0)
Borrow = (A < B)，仅在 SUB 时有效
```

### 7.5 LED 输出寄存器

- 宽度：8 bit；
- `ST` 有效时锁存指定寄存器数据；
- LED 显示最终 GCD 结果；
- 与数据 RAM 无关，不进行地址译码。

### 7.6 控制单元

主要控制信号：

| 信号 | 作用 |
|---|---|
| `RegWrite` | 通用寄存器写使能 |
| `ALUSel` | 选择 ALU 运算 |
| `FlagWrite` | 状态标志写使能 |
| `LedWrite` | LED_OUT 写使能 |
| `JumpEnable` | 跳转是否成立 |
| `PCSel` | 选择 PC+1 或跳转地址 |

---

## 8. GCD 验证程序

示例输入：

```text
R1 = 12
R2 = 8
```

推荐程序流程：

```asm
0: LI  R1, 12
1: LI  R2, 8
2: SUB R3, R1, R2
3: BZ  9
4: BB  7
5: ADD R1, R3, R0
6: JMP 2
7: SUB R2, R2, R1
8: JMP 2
9: ST  R1
```

说明：

- `SUB R3, R1, R2` 同时完成比较；
- Zero=1 表示 R1=R2，GCD 已求出；
- Borrow=1 表示 R1<R2；
- R0 固定为 0，因此 `ADD R1, R3, R0` 等效于把 R3 写入 R1；
- 最终使用 `ST R1` 将结果写入 LED_OUT。

对于 `gcd(12, 8)`，最终 LED_OUT 应显示：

```text
0000_0100
```

---

## 9. ACC＋ALU 实物设计

### 9.1 模块组成

```text
外部数据输入
      │
      ▼
┌────────────┐      ACC反馈
│ 74LS157    │◀────────────┐
│ 数据选择器 │             │
└─────┬──────┘             │
      ▼                    │
┌────────────┐             │
│ 4位 ALU    │             │
└─────┬──────┘             │
      ▼                    │
┌────────────┐             │
│ 74LS175    │─────────────┘
│ ACC寄存器  │
└─────┬──────┘
      ▼
     LED
```

时钟通路：

```text
轻触按键 → 74LS14 消抖与整形 → ACC CLK
```

必要时使用 74LS07 对显示或控制信号进行缓冲和驱动。

### 9.2 工作过程

1. 设置外部输入和运算选择；
2. 选择外部输入或 ACC 反馈；
3. ALU 产生组合逻辑结果；
4. 按下时钟按键；
5. 消抖电路输出单个有效边沿；
6. 74LS175 保存 ALU 输出；
7. ACC 输出反馈至下一次运算。

### 9.3 建议测试

| 测试 | 预期结果 |
|---|---|
| ACC 清零 | ACC 输出 `0000` |
| 3 + 5 | ACC 保存 `1000` |
| 在上一结果基础上再 +2 | ACC 保存 `1010` |
| 9 - 4 | ACC 保存 `0101` |
| AND 测试 | 与真值表一致 |
| OR 测试 | 与真值表一致 |
| 连续按键 | 每次只更新一次 ACC |

---

## 10. 实施顺序

1. 单独验证 PC 与 ROM；
2. 验证 LI 和寄存器写回；
3. 验证 ALU、Zero 和 Borrow；
4. 验证 JMP、BZ、BB；
5. 验证 ST 与 LED_OUT；
6. 装入 GCD 程序并单步测试；
7. 完成 4 位 ALU 实物；
8. 接入 ACC、反馈 MUX 和时钟整形；
9. 完成 PCB 与实物测试；
10. 整理截图、BOM、报告和调试记录。

---

## 11. 最终验收标准

### CPU 仿真

- PC 能按顺序递增；
- 跳转成立时 PC 能装载目标地址；
- LI 能正确写入寄存器；
- ADD、SUB、AND、OR 结果正确；
- Zero 与 Borrow 正确；
- ST 能把寄存器数据写入 LED_OUT；
- GCD 程序能输出正确结果；
- 电路中不存在未使用的数据 RAM 通路。

### ACC＋ALU 实物

- ALU 基本运算正确；
- ACC 可清零并在有效时钟沿保存结果；
- ACC 反馈通路正确；
- 按键不会导致一次操作多次更新；
- LED 显示与实际逻辑状态一致；
- PCB 电源、地、使能端和未使用输入处理正确。

---

## 12. 最终结论

本项目最终形成：

1. 一个不包含数据 RAM、以寄存器保存中间数据的 8 位多寄存器 CPU 仿真系统；
2. 一套能够保存和反馈 ALU 运算结果的 4 位 ACC＋ALU 实物系统。

两部分共同展示了从组合逻辑、时序逻辑到简化处理器数据通路的完整设计过程。
