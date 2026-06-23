# 8 位多寄存器 CPU 分阶段实现指南

> 本指南以最终技术方案为准：**4 位 PC、16×16 bit 指令 ROM、8 位数据通路、8 个通用寄存器、不使用数据 RAM、使用 ST 写 LED_OUT。**

---

## 一、总体实施顺序

CPU 按以下 6 个阶段搭建：

1. PC＋指令 ROM；
2. LI 指令＋寄存器写回；
3. ADD/SUB/AND/OR＋Zero/Borrow；
4. JMP/BZ/BB 跳转；
5. ST＋LED_OUT；
6. 运行 GCD 程序。

每完成一个阶段，都要先单独测试，再进入下一阶段。不要一次性搭完整系统。

---

# 阶段 1：PC＋指令 ROM

## 1. 本阶段目标

实现 CPU 的取指通路：

```text
PC → ROM → inst[15:0]
```

每个有效时钟沿：

```text
PC ← PC + 1
```

## 2. 最终参数

```text
PC 宽度：4 bit
地址范围：0～15
ROM 深度：16 words
ROM 数据宽度：16 bit
```

## 3. 电路结构

```text
               ┌──────────┐
PC[3:0] ──────▶│  +1      │
               └────┬─────┘
                    │
                    ▼
               ┌──────────┐
CLK ──────────▶│ PC寄存器 │────▶ PC[3:0]
RESET ────────▶│          │
               └────┬─────┘
                    │
                    ▼
               ┌──────────┐
               │ ROM16×16 │────▶ inst[15:0]
               └──────────┘
```

## 4. 测试

在 ROM 中写入不同测试数据：

```text
地址 0：120C
地址 1：1408
地址 2：0000
地址 3：0000
```

检查：

- 复位后 PC=0；
- 每个时钟沿 PC 加 1；
- ROM 输出与当前 PC 地址一致；
- PC 从 15 回到 0 时不出现未知状态。

---

# 阶段 2：LI 指令＋寄存器写回

## 1. 本阶段目标

实现：

```asm
LI rd, imm8
```

执行效果：

```text
Reg[rd] ← imm8
```

## 2. I 型格式

```text
[15:12] opcode = 0001
[11:9]  rd
[8]     unused
[7:0]   imm8
```

## 3. 需要增加的模块

- opcode 译码；
- rd 选择；
- imm8 提取；
- 8×8 位寄存器组；
- 写回数据选择；
- RegWrite 控制。

## 4. 寄存器规则

- R0 固定为 0；
- R1～R7 可以写入；
- 对 R0 的写入请求必须被屏蔽；
- 至少提供两个读端口和一个写端口。

## 5. 测试程序

```asm
0: LI R1, 12
1: LI R2, 8
2: NOP
```

预期：

```text
R1 = 0000_1100
R2 = 0000_1000
R0 = 0000_0000
```

---

# 阶段 3：ALU＋状态标志

## 1. 本阶段目标

实现：

```asm
ADD rd, rs1, rs2
SUB rd, rs1, rs2
AND rd, rs1, rs2
OR  rd, rs1, rs2
```

## 2. R 型格式

```text
[15:12] opcode
[11:9]  rd
[8:6]   rs1
[5:3]   rs2
[2:0]   unused
```

## 3. ALU 功能

```text
ADD：A + B
SUB：A - B
AND：A & B
OR ：A | B
```

## 4. 状态标志

```text
Zero = 1，当结果为 0
Borrow = 1，当 SUB 中 A < B
```

GCD 程序主要使用 SUB 产生的 Zero 和 Borrow。

## 5. 测试

### ADD

```text
A = 3
B = 5
结果 = 8
```

### SUB，无借位

```text
A = 12
B = 8
结果 = 4
Borrow = 0
Zero = 0
```

### SUB，有借位

```text
A = 8
B = 12
Borrow = 1
```

### SUB，结果为零

```text
A = 8
B = 8
结果 = 0
Zero = 1
```

---

# 阶段 4：JMP、BZ、BB 跳转

## 1. 本阶段目标

实现三种跳转：

```asm
JMP addr
BZ  addr
BB  addr
```

## 2. 跳转条件

```text
isJMP = opcode == 1000
isBZ  = opcode == 1001
isBB  = opcode == 1010

JumpEnable =
    isJMP
    OR (isBZ AND Zero)
    OR (isBB AND Borrow)
```

## 3. PC 选择

```text
PC_plus_1 = PC + 1

PC_next =
    inst[3:0]，JumpEnable = 1
    PC_plus_1，JumpEnable = 0
```

也可以从 `inst[7:0]` 中提取地址，但最终只接低 4 位到 PC。

## 4. MUX 位置

MUX 应位于 PC 寄存器输入端：

```text
PC+1 ──────────┐
               ├──▶ MUX ───▶ PC寄存器D端
jump_addr ─────┘
                  ▲
                  │
              JumpEnable
```

## 5. 测试

### JMP

让地址 1 的 JMP 跳到地址 5，检查 PC 是否直接装载 5。

### BZ

- Zero=0：PC 顺序加 1；
- Zero=1：PC 跳到目标地址。

### BB

- Borrow=0：PC 顺序加 1；
- Borrow=1：PC 跳到目标地址。

---

# 阶段 5：ST＋LED_OUT

## 1. 本阶段目标

CPU 最终方案不使用数据 RAM。

`ST` 的作用直接定义为：

```asm
ST rs
```

执行：

```text
LED_OUT ← Reg[rs]
```

## 2. S 型格式

```text
[15:12] opcode = 0011
[11:9]  rs
[8:0]   unused
```

## 3. 电路结构

```text
Reg[rs] ─────────────▶ LED_OUT寄存器D端
                          ▲
                          │
                       LedWrite
```

## 4. 测试

```asm
0: LI R1, 4
1: ST R1
```

预期：

```text
LED_OUT = 0000_0100
```

## 5. 注意事项

- 不要再搭建 RAM；
- 不需要地址加法器；
- 不需要 RAM 地址译码；
- 不需要 LD 指令；
- ST 只负责最终结果输出；
- LED 必须串联限流电阻。

---

# 阶段 6：运行 GCD 程序

## 1. 程序目标

```text
gcd(12, 8) = 4
```

## 2. 推荐程序

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

## 3. 执行逻辑

```text
R1 = R2：
    Zero = 1
    跳到 ST

R1 > R2：
    Borrow = 0
    R3 = R1 - R2
    R1 ← R3

R1 < R2：
    Borrow = 1
    R2 ← R2 - R1
```

## 4. 单步检查表

| PC | 指令 | 重点观察 |
|---|---|---|
| 0 | LI R1,12 | R1 是否写入 12 |
| 1 | LI R2,8 | R2 是否写入 8 |
| 2 | SUB R3,R1,R2 | ALU、Zero、Borrow |
| 3 | BZ 9 | Zero 跳转 |
| 4 | BB 7 | Borrow 跳转 |
| 5 | ADD R1,R3,R0 | R3 是否复制到 R1 |
| 6 | JMP 2 | PC 是否返回比较 |
| 7 | SUB R2,R2,R1 | 更新较大的数 |
| 8 | JMP 2 | PC 是否返回比较 |
| 9 | ST R1 | LED_OUT 是否显示 GCD |

## 5. 最终结果

```text
LED_OUT = 0000_0100
```

---

# 七、控制信号汇总

| 信号 | LI | ADD/SUB/AND/OR | JMP/BZ/BB | ST |
|---|---:|---:|---:|---:|
| RegWrite | 1 | 1 | 0 | 0 |
| FlagWrite | 0 | SUB 时 1 | 0 | 0 |
| LedWrite | 0 | 0 | 0 | 1 |
| JumpEnable | 0 | 0 | 按条件 | 0 |
| PCSel | 顺序 | 顺序 | 顺序/跳转 | 顺序 |

---

# 八、常见错误

## 1. PC 和 ROM 规格不一致

最终必须统一为：

```text
PC：4 bit
ROM：16×16 bit
```

## 2. 仍然保留 RAM 线路

最终方案没有数据 RAM。应删除：

- RAM 芯片；
- LD 指令译码；
- RAM 地址线；
- MemRead、MemWrite；
- RAM 数据回写 MUX。

## 3. R0 被写入

R0 必须保持为 0，否则 GCD 程序中的复制操作会出错。

## 4. Borrow 极性理解错误

本项目统一定义：

```text
Borrow = 1，当无符号 A < B
```

如果使用具体芯片的 Carry 输出判断借位，要根据芯片逻辑确认是否需要取反。

## 5. 标志位更新时机错误

Zero 和 Borrow 应由用于比较的 SUB 指令更新，跳转指令只读取标志，不重新计算。

## 6. PC 在同一时钟沿错误更新

组合逻辑先产生 `PC_next`，PC 寄存器只在有效时钟沿更新一次。

## 7. Multisim 二进制工程冲突

多人不要同时修改同一个 `.ms14` 文件。建议：

- 每人负责独立模块；
- 合并前先拉取远程更新；
- 每次提交附带截图；
- 用清楚的文件名和版本说明。

---

# 九、最终检查清单

## 取指

- [ ] PC 为 4 bit；
- [ ] ROM 为 16×16 bit；
- [ ] 复位后 PC=0；
- [ ] PC+1 正常。

## 寄存器

- [ ] R0 固定为 0；
- [ ] LI 能正确写入；
- [ ] 两个读端口正确；
- [ ] 写回目标寄存器正确。

## ALU 与标志

- [ ] ADD 正确；
- [ ] SUB 正确；
- [ ] AND 正确；
- [ ] OR 正确；
- [ ] Zero 正确；
- [ ] Borrow 正确。

## 跳转

- [ ] JMP 正确；
- [ ] BZ 正确；
- [ ] BB 正确；
- [ ] PC MUX 选择正确。

## 输出

- [ ] 不存在数据 RAM；
- [ ] ST 能写 LED_OUT；
- [ ] LED 显示正确。

## 完整程序

- [ ] GCD 程序逐条执行正确；
- [ ] `gcd(12,8)` 输出 4；
- [ ] 复位后可以重新运行。
