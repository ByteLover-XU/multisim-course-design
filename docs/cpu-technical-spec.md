# 8 位多寄存器自定义指令集 CPU 技术方案

> **仿真平台：** NI Multisim  
> **数据宽度：** 8 bit  
> **指令宽度：** 16 bit  
> **程序空间：** 16×16 bit ROM  
> **最终方案：** 不设置数据 RAM，中间数据全部保存在通用寄存器中

---

## 1. 项目定位

本项目实现一款适合数字电路课程设计规模的 8 位多寄存器简化 CPU。

CPU 仿真包括：

- 4 位程序计数器 PC；
- 16×16 bit 指令 ROM；
- 指令译码与控制单元；
- 8 个 8 位通用寄存器 R0～R7；
- 8 位 ALU；
- Zero、Borrow 状态寄存器；
- JMP、BZ、BB 跳转控制；
- 8 位 LED 输出寄存器。

本设计不实现数据 RAM、Cache 和完整商业 ISA，重点展示简化 CPU 的取指、译码、运算、写回和跳转过程。

---

## 2. 总体规格

| 项目 | 规格 |
|---|---|
| 数据宽度 | 8 bit |
| 指令宽度 | 16 bit |
| 通用寄存器 | R0～R7，共 8 个 |
| R0 | 固定为 0，禁止写入 |
| PC | 4 bit |
| ROM | 16×16 bit |
| 数据 RAM | 不使用 |
| ALU | ADD、SUB、AND、OR |
| 状态标志 | Zero、Borrow |
| 标志更新 | 仅 SUB 更新 |
| 跳转 | JMP、BZ、BB |
| 输出 | `ST` 写 LED_OUT |
| 验证程序 | 辗转相减法 GCD |

---

## 3. 总体结构

```text
┌──────────────┐
│  PC[3:0]     │
└──────┬───────┘
       ▼
┌──────────────┐
│ ROM 16×16    │
└──────┬───────┘
       │ inst[15:0]
       ▼
┌──────────────────┐
│ Control Unit     │
└──────┬───────────┘
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
                          ST
```

CPU 不设置数据 RAM。GCD 的输入、中间值和最终值均由寄存器保存。

---

## 4. 指令集

| opcode | 指令 | 功能 |
|---|---|---|
| `0000` | `NOP` | 空操作 |
| `0001` | `LI rd, imm8` | `Reg[rd] ← imm8` |
| `0010` | 保留 | 当前不使用 |
| `0011` | `ST rs` | `LED_OUT ← Reg[rs]` |
| `0100` | `ADD rd, rs1, rs2` | `rd ← rs1 + rs2` |
| `0101` | `SUB rd, rs1, rs2` | `rd ← rs1 - rs2`，更新标志 |
| `0110` | `AND rd, rs1, rs2` | `rd ← rs1 & rs2` |
| `0111` | `OR rd, rs1, rs2` | `rd ← rs1 | rs2` |
| `1000` | `JMP addr` | 无条件跳转 |
| `1001` | `BZ addr` | Zero=1 时跳转 |
| `1010` | `BB addr` | Borrow=1 时跳转 |
| `1011～1111` | 保留 | 当前不使用 |

---

## 5. 指令格式

### 5.1 R 型

适用于 ADD、SUB、AND、OR：

```text
[15:12] opcode
[11:9]  rd
[8:6]   rs1
[5:3]   rs2
[2:0]   unused
```

```text
inst = (opcode << 12)
     | (rd << 9)
     | (rs1 << 6)
     | (rs2 << 3)
```

### 5.2 I 型

适用于 `LI rd, imm8`：

```text
[15:12] opcode
[11:9]  rd
[8]     unused
[7:0]   imm8
```

```text
inst = (opcode << 12) | (rd << 9) | imm8
```

### 5.3 S 型输出指令

适用于 `ST rs`：

```text
[15:12] opcode
[11:9]  rs
[8:0]   unused
```

### 5.4 J/B 型

适用于 JMP、BZ、BB：

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

## 6. 寄存器组

- 8 个 8 位寄存器；
- R0 固定输出 `0000_0000`；
- 对 R0 的写入请求必须屏蔽；
- 两个组合读端口；
- 一个同步写端口；
- LI 和 ALU 指令可以写回；
- ST、跳转和 NOP 不写通用寄存器。

---

## 7. ALU 与状态标志

### 7.1 ALU 运算

```text
ADD: A + B
SUB: A - B
AND: A & B
OR : A | B
```

### 7.2 状态定义

```text
Zero   = 1，当 SUB 结果为 0
Borrow = 1，当无符号 SUB 中 A < B
```

### 7.3 标志更新规则

最终规则统一为：

```text
只有 SUB 指令更新 Zero 和 Borrow。
ADD、AND、OR、LI、ST、NOP 和跳转指令保持原标志不变。
```

原因：GCD 程序执行顺序为 `SUB → BZ → BB`，BZ 和 BB 必须读取同一次 SUB 保存的标志。

如果实际减法电路通过加法器 Cout 判断借位，应根据具体芯片的 Cout 极性确认 Borrow 是否需要取反。

---

## 8. 跳转控制

```text
isJMP = (opcode == 1000)
isBZ  = (opcode == 1001)
isBB  = (opcode == 1010)

BZ_take = isBZ AND Zero
BB_take = isBB AND Borrow

JumpEnable = isJMP OR BZ_take OR BB_take
```

```text
PC_plus_1 = PC + 1

PC_next =
    jump_addr[3:0]，JumpEnable = 1
    PC_plus_1，     JumpEnable = 0
```

PC 输入端通过 MUX 在 `PC+1` 和跳转地址之间选择。

---

## 9. 控制信号

| 信号 | 功能 |
|---|---|
| `RegWrite` | 通用寄存器写使能 |
| `ALUSel` | ALU 运算选择 |
| `FlagWrite` | Zero、Borrow 写使能 |
| `LedWrite` | LED_OUT 写使能 |
| `JumpEnable` | 跳转成立 |
| `PCSel` | 选择 PC+1 或跳转地址 |

### 控制表

| 指令类型 | RegWrite | FlagWrite | LedWrite | JumpEnable |
|---|---:|---:|---:|---:|
| NOP | 0 | 0 | 0 | 0 |
| LI | 1 | 0 | 0 | 0 |
| ADD | 1 | 0 | 0 | 0 |
| SUB | 1 | 1 | 0 | 0 |
| AND | 1 | 0 | 0 | 0 |
| OR | 1 | 0 | 0 | 0 |
| ST | 0 | 0 | 1 | 0 |
| JMP | 0 | 0 | 0 | 1 |
| BZ | 0 | 0 | 0 | `Zero` |
| BB | 0 | 0 | 0 | `Borrow` |

---

## 10. GCD 验证程序

目标：

```text
gcd(12, 8) = 4
```

### 汇编程序

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

### 已核对的 ROM 机器码

| 地址 | 指令 | 机器码 |
|---:|---|---|
| 0 | `LI R1, 12` | `120C` |
| 1 | `LI R2, 8` | `1408` |
| 2 | `SUB R3, R1, R2` | `5650` |
| 3 | `BZ 9` | `9009` |
| 4 | `BB 7` | `A007` |
| 5 | `ADD R1, R3, R0` | `42C0` |
| 6 | `JMP 2` | `8002` |
| 7 | `SUB R2, R2, R1` | `5488` |
| 8 | `JMP 2` | `8002` |
| 9 | `ST R1` | `3200` |
| 10～15 | `NOP` | `0000` |

### 程序逻辑

```text
R1 = R2：Zero=1，跳到 ST
R1 > R2：R1 ← R1 - R2
R1 < R2：Borrow=1，R2 ← R2 - R1
```

最终：

```text
R1 = 4
LED_OUT = 0000_0100
```

---

## 11. 关键时序要求

在一个时钟周期内：

1. PC 和 ROM 产生当前指令；
2. 译码器生成控制信号；
3. 寄存器组和 ALU 产生组合结果；
4. `PC_next`、写回数据和新标志在时钟沿前稳定；
5. 有效时钟沿到来时，同时更新 PC、目标寄存器、状态寄存器或 LED_OUT。

不能在同一个时钟沿之后再使用刚更新的 PC 作为当前指令地址完成同一次执行。

---

## 12. 最终验收标准

- [ ] PC 为 4 bit；
- [ ] ROM 为 16×16 bit；
- [ ] R0 始终为 0；
- [ ] LI 正确；
- [ ] ADD、SUB、AND、OR 正确；
- [ ] 只有 SUB 更新 Zero、Borrow；
- [ ] Borrow 极性与实际电路一致；
- [ ] JMP、BZ、BB 正确；
- [ ] ST 能写 LED_OUT；
- [ ] 电路中不存在数据 RAM 和 LD 通路；
- [ ] ROM 机器码与指令格式一致；
- [ ] `gcd(12,8)` 最终输出 4。
