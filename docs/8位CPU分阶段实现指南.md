# 8 位多寄存器 CPU 分阶段实现指南

> 目标：按照“先小模块、后整机”的方式，在 Multisim 中逐步完成 CPU，避免一次搭完整系统后难以排错。

---

## 一、总体实施顺序

整个 CPU 建议分为 6 个阶段：

1. **PC + 指令 ROM**
2. **LI 指令 + 寄存器写回**
3. **ADD / SUB + Zero / Borrow 标志位**
4. **JMP / BZ / BB 跳转系统**
5. **ST + LED_IO 内存映射输出**
6. **运行完整 GCD 程序**

每完成一个阶段，都要先单独测试，确认正确后再进入下一阶段。

---

# 阶段 1：实现 PC + 指令 ROM

## 1. 本阶段要实现什么

本阶段只实现 CPU 的“取指”部分。

CPU 每来一个时钟，PC 加 1；ROM 根据 PC 的值输出对应地址中的 16 位指令。

需要实现：

- 程序计数器 PC
- `PC + 1` 加法器
- PC 寄存器
- 指令 ROM
- 16 位指令观察端口

---

## 2. PC 的作用

PC 保存当前正在执行的指令地址。

例如：

```text
PC = 0  → ROM 输出第 0 条指令
PC = 1  → ROM 输出第 1 条指令
PC = 2  → ROM 输出第 2 条指令
```

正常情况下：

```text
PC_next = PC + 1
```

在时钟上升沿：

```text
PC ← PC_next
```

---

## 3. 推荐参数

```text
PC 宽度：5 bit
可寻址范围：0～31
ROM 深度：32 words
ROM 数据宽度：16 bit
```

---

## 4. 需要搭建的电路

```text
        ┌─────────┐
时钟 ──▶│ PC寄存器 │──────▶ PC[4:0]
        └────┬────┘
             │
             ▼
        ┌─────────┐
        │  PC + 1 │
        └────┬────┘
             │
             └────────────▶ PC寄存器输入

PC[4:0] ───────────────▶ ROM地址端

ROM数据端 ─────────────▶ inst[15:0]
```

---

## 5. 测试内容

在 ROM 中预先写入几条不同的数据，例如：

```text
地址 0：120C
地址 1：1408
地址 2：1EF0
地址 3：5650
```

单步输入时钟，观察：

```text
PC：0 → 1 → 2 → 3
ROM输出：
120C → 1408 → 1EF0 → 5650
```

---

## 6. 本阶段完成标准

满足以下条件，即可进入阶段 2：

- PC 每个时钟稳定加 1。
- PC 不会乱跳或一次增加多个数。
- ROM 地址端正确连接 PC。
- ROM 能输出对应地址的 16 位指令。
- 能用数码显示器、逻辑探针或十六进制显示器观察 PC 和指令。

---

## 7. 本阶段先不要实现

暂时不要接：

- 寄存器堆
- ALU
- RAM
- 跳转电路
- LED 输出

此时 CPU 只需要完成：

```text
取指
```

---

# 阶段 2：实现 LI 指令 + 寄存器写回

## 1. 本阶段要实现什么

本阶段实现第一条真正可以执行的指令：

```asm
LI rd, imm8
```

作用：

```text
Reg[rd] ← imm8
```

例如：

```asm
LI R1, 12
```

执行后：

```text
R1 = 12
```

---

## 2. 本阶段新增模块

需要新增：

- opcode 译码
- 目标寄存器地址 `rd`
- 立即数 `imm8`
- 8 个 8 位寄存器
- 寄存器写使能 `RegWrite`
- 写回数据选择器

---

## 3. LI 指令格式

```text
[15:12] opcode
[11:9]  rd
[8]     unused
[7:0]   imm8
```

对于 LI：

```text
opcode = 0001
```

因此指令译码结果为：

```text
opcode == 0001
→ RegWrite = 1
→ 写回数据选择 imm8
```

---

## 4. 寄存器堆结构

共 8 个寄存器：

```text
R0～R7
```

每个寄存器宽度：

```text
8 bit
```

其中：

```text
R0 恒为 0
```

因此写入时必须满足：

```text
RegWrite = 1
并且 rd ≠ 000
```

即：

```text
if RegWrite and rd != 0:
    Reg[rd] = write_data
```

---

## 5. 推荐电路结构

```text
inst[15:12] ──▶ opcode译码 ──▶ RegWrite

inst[11:9] ────────────────▶ 寄存器写地址 rd

inst[7:0] ─────────────────▶ 写回数据 imm8

RegWrite + 时钟 ───────────▶ 寄存器写入
```

---

## 6. 测试程序

ROM 中放入：

```asm
0: LI R1, 12
1: LI R2, 8
```

对应机器码：

```text
0: 120C
1: 1408
```

---

## 7. 预期结果

执行第 0 条指令后：

```text
R1 = 12
```

执行第 1 条指令后：

```text
R2 = 8
```

同时：

```text
R0 = 0
```

---

## 8. 本阶段完成标准

满足以下条件，即可进入阶段 3：

- opcode 能正确识别 `0001`。
- `rd` 能正确选择 R1、R2 等寄存器。
- `imm8` 能正确取出低 8 位。
- 寄存器只在时钟边沿写入一次。
- `LI R1,12` 后 R1 显示 12。
- `LI R2,8` 后 R2 显示 8。
- 写入 R0 时，R0 仍然保持 0。

---

## 9. 常见错误

### 错误 1：寄存器不停变化

原因：

```text
写使能没有和时钟配合
```

### 错误 2：所有寄存器一起写入

原因：

```text
rd 没有经过 3-8 译码器
```

### 错误 3：R0 被写成其他数

原因：

```text
没有屏蔽 rd = 000
```

---

# 阶段 3：实现 ADD / SUB + 标志位

## 1. 本阶段要实现什么

本阶段实现 ALU 的核心运算：

```asm
ADD rd, rs1, rs2
SUB rd, rs1, rs2
```

并产生两个状态标志：

```text
Zero
Borrow
```

---

## 2. 本阶段新增模块

需要新增：

- 两个寄存器读端口
- `rs1` 地址选择
- `rs2` 地址选择
- ALU
- ALU 操作选择 `ALUOp`
- ALU 结果写回
- Zero 标志寄存器
- Borrow 标志寄存器

---

## 3. R 型指令格式

```text
[15:12] opcode
[11:9]  rd
[8:6]   rs1
[5:3]   rs2
[2:0]   unused
```

---

## 4. ALU 功能

### ADD

```text
ALU_out = rs1 + rs2
```

### SUB

```text
ALU_out = rs1 - rs2
```

---

## 5. Zero 标志

定义：

```text
Zero = 1，当 ALU_out = 0
Zero = 0，当 ALU_out ≠ 0
```

可用方法：

- 对 ALU_out 的 8 位做或运算。
- 再将结果取反。

即：

```text
Zero = NOT(ALU_out[7] OR ... OR ALU_out[0])
```

---

## 6. Borrow 标志

Borrow 只在 SUB 时有效。

定义：

```text
如果 rs1 < rs2，则 Borrow = 1
否则 Borrow = 0
```

例如：

```text
12 - 8 = 4
Borrow = 0
```

```text
4 - 8
Borrow = 1
```

---

## 7. 写回路径

ALU 运算结果要写回 `rd`：

```text
write_data = ALU_out
```

因此需要一个写回选择器：

```text
WBSel = IMM → 写回立即数
WBSel = ALU → 写回 ALU 结果
```

---

## 8. 测试程序

```asm
LI  R1, 12
LI  R2, 8
SUB R3, R1, R2
```

机器码：

```text
120C
1408
5650
```

---

## 9. 预期结果

执行 SUB 后：

```text
R3 = 4
Zero = 0
Borrow = 0
```

再测试：

```asm
SUB R3, R2, R1
```

预期：

```text
Borrow = 1
```

再测试：

```asm
SUB R3, R1, R1
```

预期：

```text
R3 = 0
Zero = 1
Borrow = 0
```

---

## 10. 本阶段完成标准

满足以下条件，即可进入阶段 4：

- `rs1` 能选出正确寄存器。
- `rs2` 能选出正确寄存器。
- ADD 结果正确。
- SUB 结果正确。
- ALU 结果能写回 `rd`。
- 结果为 0 时 Zero = 1。
- 无符号减法发生借位时 Borrow = 1。
- LI 和 ALU 两种写回来源可以正确切换。

---

## 11. 本阶段建议额外完成

虽然 GCD 主要使用 ADD 和 SUB，但可以在这一阶段顺便加入：

```text
AND
OR
```

这样 ALU 一次完成全部四种功能：

```text
ADD / SUB / AND / OR
```

---

# 阶段 4：实现 JMP / BZ / BB 跳转系统

## 1. 本阶段要实现什么

本阶段让 CPU 不再只能顺序执行，而是可以跳转和循环。

实现：

```asm
JMP addr
BZ  addr
BB  addr
```

---

## 2. 三种跳转指令

### JMP

无条件跳转：

```text
PC_next = addr
```

### BZ

当 Zero = 1 时跳转：

```text
if Zero == 1:
    PC_next = addr
else:
    PC_next = PC + 1
```

### BB

当 Borrow = 1 时跳转：

```text
if Borrow == 1:
    PC_next = addr
else:
    PC_next = PC + 1
```

---

## 3. 指令格式

```text
[15:12] opcode
[11:8]  unused
[7:0]   addr
```

---

## 4. PC 下一值选择

PC 输入不能再只接 `PC + 1`，需要增加多路选择器：

```text
                   ┌──────────┐
PC + 1 ───────────▶│          │
                   │ PC MUX   ├────▶ PC寄存器输入
addr ─────────────▶│          │
                   └──────────┘
```

选择逻辑：

| 指令情况 | PC_next |
|---|---|
| 普通指令 | PC + 1 |
| JMP | addr |
| BZ 且 Zero=1 | addr |
| BZ 且 Zero=0 | PC + 1 |
| BB 且 Borrow=1 | addr |
| BB 且 Borrow=0 | PC + 1 |

---

## 5. 推荐测试 1：JMP 自循环

ROM：

```asm
0: JMP 0
```

预期：

```text
PC 始终为 0
```

---

## 6. 推荐测试 2：BZ

```asm
0: LI  R1, 4
1: SUB R2, R1, R1
2: BZ  4
3: LI  R3, 1
4: LI  R3, 2
```

因为：

```text
R1 - R1 = 0
Zero = 1
```

所以应跳到地址 4。

最终：

```text
R3 = 2
```

---

## 7. 推荐测试 3：BB

```asm
0: LI  R1, 4
1: LI  R2, 8
2: SUB R3, R1, R2
3: BB  5
4: LI  R4, 1
5: LI  R4, 2
```

因为：

```text
4 < 8
Borrow = 1
```

所以应跳到地址 5。

最终：

```text
R4 = 2
```

---

## 8. 本阶段完成标准

满足以下条件，即可进入阶段 5：

- JMP 能无条件修改 PC。
- BZ 只在 Zero = 1 时跳转。
- BB 只在 Borrow = 1 时跳转。
- 条件不满足时 PC 正常加 1。
- PC 在跳转后不会额外再加 1。
- 能够搭建简单循环。

---

## 9. 常见错误

### 错误 1：跳转地址偏一位

表现：

```text
想跳到 4，实际到了 5
```

原因：

```text
先做了 PC+1，又把跳转地址加了一次
```

本方案使用绝对地址：

```text
PC_next = addr
```

不是：

```text
PC + addr
```

### 错误 2：BZ 永远跳转

原因：

```text
Zero 没有保存，或者组合逻辑接错
```

### 错误 3：SUB 后下一条分支读到旧标志

原因：

```text
标志寄存器更新时序与 PC 更新时序冲突
```

需要保证 SUB 执行后，下一条指令能读到新的 Zero/Borrow。

---

# 阶段 5：实现 ST + LED_IO

## 1. 本阶段要实现什么

本阶段实现 CPU 的输出功能。

CPU 使用普通 ST 指令，把数据写到特殊地址：

```text
0xF0
```

当地址等于 `0xF0` 时，不写 RAM，而是写 LED 输出寄存器。

---

## 2. ST 指令功能

```asm
ST rs, [base + off]
```

地址计算：

```text
address = Reg[base] + offset
```

写入数据：

```text
write_data = Reg[rs]
```

---

## 3. 内存映射 I/O

规定：

```text
0xF0 = LED_OUT
```

写入规则：

```text
if ST and address == 0xF0:
    LED_OUT = write_data
else if ST:
    RAM[address] = write_data
```

---

## 4. 本阶段新增模块

需要新增：

- 地址加法器
- base 寄存器读端口
- offset 扩展
- ST 写数据通路
- 地址比较器
- LED_OUT 寄存器
- RAM 写使能屏蔽逻辑

---

## 5. 推荐测试程序

```asm
LI R1, 4
LI R7, 240
ST R1, [R7 + 0]
```

机器码：

```text
1204
1EF0
33C0
```

---

## 6. 预期结果

```text
R1 = 4
R7 = 240 = 0xF0
address = 0xF0
LED_OUT = 00000100
```

---

## 7. LED 输出方式

最简单可以直接连接 8 个 LED：

```text
LED_OUT[7:0]
```

显示：

```text
00000100
```

如果使用数码管，也可以把低 4 位送入七段译码器。

---

## 8. 本阶段完成标准

满足以下条件，即可进入阶段 6：

- ST 能正确读取源寄存器数据。
- base + offset 地址计算正确。
- 地址等于 `0xF0` 时 LED_OUT 更新。
- 地址等于 `0xF0` 时 RAM 不应被写入。
- LED_OUT 在下一次写入前保持原值。
- 测试程序能稳定显示 4。

---

# 阶段 6：运行完整 GCD 程序

## 1. 本阶段要实现什么

本阶段不再增加新的核心模块，而是把前五个阶段连接起来，运行完整程序。

目标程序：

```text
gcd(12, 8) = 4
```

---

## 2. 完整程序

```asm
0:  LI   R1, 12
1:  LI   R2, 8
2:  LI   R7, 240

3:  SUB  R3, R1, R2
4:  BZ   10
5:  BB   8

6:  ADD  R1, R3, R0
7:  JMP  3

8:  SUB  R2, R2, R1
9:  JMP  3

10: ST   R1, [R7 + 0]
11: JMP  10
```

---

## 3. ROM 初始化内容

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

## 4. 执行过程

### 初始状态

```text
R1 = 12
R2 = 8
```

### 第一次比较

```text
R3 = 12 - 8 = 4
Zero = 0
Borrow = 0
```

执行：

```asm
ADD R1, R3, R0
```

得到：

```text
R1 = 4
```

### 第二次比较

```text
R3 = 4 - 8
Borrow = 1
```

执行 BB，跳到地址 8：

```text
R2 = 8 - 4 = 4
```

### 第三次比较

```text
R3 = 4 - 4 = 0
Zero = 1
```

执行 BZ，跳到地址 10。

### 输出

```text
LED_OUT = R1 = 4
```

最终：

```text
LED_OUT = 00000100
```

---

## 5. 单步调试时重点观察

建议同时观察：

```text
PC
当前指令 inst[15:0]
opcode
R1
R2
R3
Zero
Borrow
RegWrite
MemWrite
PCSel
LED_OUT
```

---

## 6. 建议制作的观察面板

```text
PC：十六进制显示
指令：4 位十六进制显示
R1：2 位十六进制显示
R2：2 位十六进制显示
R3：2 位十六进制显示
Zero：LED
Borrow：LED
LED_OUT：8 个 LED
```

---

## 7. 本阶段完成标准

整个 CPU 完成的判断标准：

- PC 能正确顺序执行和跳转。
- ROM 输出的机器码正确。
- LI 能写寄存器。
- ADD/SUB 运算正确。
- Zero/Borrow 正确。
- BZ/BB/JMP 正确。
- ST 能写 LED_IO。
- GCD 程序最终输出 4。
- CPU 能连续运行，也能单步运行。

---

# 二、各阶段新增内容总表

| 阶段 | 新增模块 | 新增指令 | 测试目标 |
|---|---|---|---|
| 阶段 1 | PC、PC+1、ROM | 暂无 | PC 递增，ROM 正确输出 |
| 阶段 2 | 指令译码、寄存器堆、立即数写回 | LI | 把 12、8 写入 R1、R2 |
| 阶段 3 | 双读端口、ALU、标志寄存器 | ADD、SUB、AND、OR | 运算和 Zero/Borrow 正确 |
| 阶段 4 | PC 多路选择器、跳转控制 | JMP、BZ、BB | 实现条件跳转和循环 |
| 阶段 5 | 地址计算、地址比较、LED_OUT | ST | 向 0xF0 写数据并点亮 LED |
| 阶段 6 | 整机联调 | 全部指令 | GCD 最终输出 4 |

---

# 三、推荐实际搭建顺序

在 Multisim 中，建议按照以下顺序建立子电路：

```text
1. PC 子电路
2. ROM 子电路
3. 指令字段拆分
4. 控制器
5. 寄存器堆
6. ALU
7. 标志位
8. PC 跳转选择
9. LED_IO
10. RAM
11. 整机顶层连接
```

其中 RAM 可以最后再做，因为 GCD 程序实际上只使用了内存映射 LED 输出，没有真正使用普通 RAM 数据。

---

# 四、最重要的调试原则

## 1. 一次只增加一个功能

不要同时加入：

```text
ALU + 跳转 + RAM + LED
```

应该每加入一个模块，就单独测试。

## 2. 先测试数据，再测试控制

例如先确认：

```text
R1、R2 能正确输出
ALU 能正确算出结果
```

再检查：

```text
RegWrite
PCSel
MemWrite
```

## 3. 所有关键总线都接观察器

至少观察：

```text
PC
inst
rs1_data
rs2_data
ALU_out
write_data
```

## 4. 使用单步时钟

整机联调前，不要直接使用高速时钟。

建议使用：

```text
按钮单步时钟
```

每按一次，只执行一条指令。

## 5. 每个阶段保存一个独立版本

建议文件名：

```text
CPU_stage1_PC_ROM.ms14
CPU_stage2_LI_RegFile.ms14
CPU_stage3_ALU.ms14
CPU_stage4_Branch.ms14
CPU_stage5_LED_IO.ms14
CPU_stage6_GCD.ms14
```

这样后面出现错误时，可以快速返回上一个正确版本。
