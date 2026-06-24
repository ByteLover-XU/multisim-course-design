# 8 位多寄存器 CPU 分阶段实现指南

> 最终参数：4 位 PC、16×16 bit 指令 ROM、8 位数据通路、8 个通用寄存器、不使用数据 RAM、使用 `ST` 写 LED_OUT。

---

## 一、总体实施顺序

1. PC＋指令 ROM；
2. LI＋寄存器写回；
3. ADD、SUB、AND、OR；
4. Zero、Borrow 状态寄存器；
5. JMP、BZ、BB；
6. ST＋LED_OUT；
7. 装入 GCD 机器码并单步验证。

每个阶段必须先单独测试，再接入总系统。

---

# 阶段 1：PC＋指令 ROM

## 目标

```text
PC → ROM → inst[15:0]
```

## 参数

```text
PC：4 bit
地址：0～15
ROM：16 words × 16 bit
```

## 结构

```text
PC+1 ──────────┐
               ├──→ PC输入MUX → PC寄存器 → ROM地址
跳转地址 ──────┘
```

## 测试

- RESET 后 PC=0；
- 正常情况下每个有效时钟沿 PC 加 1；
- PC=15 后按 4 位二进制自然回到 0；
- ROM 输出与 PC 地址一致。

测试数据：

| 地址 | 数据 |
|---:|---|
| 0 | `120C` |
| 1 | `1408` |
| 2 | `0000` |
| 3 | `0000` |

---

# 阶段 2：LI＋寄存器写回

## LI 格式

```text
[15:12] opcode = 0001
[11:9]  rd
[8]     unused
[7:0]   imm8
```

## 寄存器组要求

- R0～R7，共 8 个 8 位寄存器；
- R0 固定为 0；
- 对 R0 的写请求必须屏蔽；
- 两个读端口；
- 一个同步写端口。

## 测试程序

```asm
0: LI R1, 12
1: LI R2, 8
```

预期：

```text
R1 = 0000_1100
R2 = 0000_1000
R0 = 0000_0000
```

---

# 阶段 3：ALU

## R 型格式

```text
[15:12] opcode
[11:9]  rd
[8:6]   rs1
[5:3]   rs2
[2:0]   unused
```

## 运算

```text
ADD：A + B
SUB：A - B
AND：A & B
OR ：A | B
```

## 测试用例

| A | B | 运算 | 预期 |
|---|---|---|---|
| 3 | 5 | ADD | 8 |
| 12 | 8 | SUB | 4 |
| `1100` | `1010` | AND | `1000` |
| `0101` | `0011` | OR | `0111` |

---

# 阶段 4：Zero 与 Borrow

## 最终定义

```text
Zero   = 1，当 SUB 结果为 0
Borrow = 1，当无符号 SUB 中 A < B
```

## 更新规则

```text
只有 SUB 指令更新 Zero 和 Borrow。
其他指令保持状态寄存器原值。
```

这是 GCD 程序正确运行的必要条件，因为 BZ 和 BB 必须读取前一条 SUB 保存的状态。

## 测试

| A | B | 结果 | Zero | Borrow |
|---:|---:|---:|---:|---:|
| 12 | 8 | 4 | 0 | 0 |
| 8 | 12 | 252（8 位回绕） | 0 | 1 |
| 8 | 8 | 0 | 1 | 0 |

> 如果实际电路从加法器 Cout 生成 Borrow，应根据芯片逻辑确认是否需要取反。

---

# 阶段 5：JMP、BZ、BB

## 跳转逻辑

```text
isJMP = opcode == 1000
isBZ  = opcode == 1001
isBB  = opcode == 1010

JumpEnable = isJMP
             OR (isBZ AND Zero)
             OR (isBB AND Borrow)
```

## PC 下一值

```text
PC_next =
    inst[3:0]，JumpEnable = 1
    PC + 1，   JumpEnable = 0
```

## 测试

- JMP：无条件装载目标地址；
- BZ：Zero=1 时跳转，Zero=0 时顺序执行；
- BB：Borrow=1 时跳转，Borrow=0 时顺序执行；
- 跳转指令不得改写 Zero 和 Borrow。

---

# 阶段 6：ST＋LED_OUT

最终方案没有数据 RAM。

```asm
ST rs
```

执行：

```text
LED_OUT ← Reg[rs]
```

## 格式

```text
[15:12] opcode = 0011
[11:9]  rs
[8:0]   unused
```

## 测试

```asm
0: LI R1, 4
1: ST R1
```

预期：

```text
LED_OUT = 0000_0100
```

注意：

- 不搭建数据 RAM；
- 不使用 LD；
- 不使用 MemRead、MemWrite；
- ST 只负责写 LED_OUT；
- LED 必须串联限流电阻。

---

# 阶段 7：GCD 程序

## 汇编程序

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

## ROM 机器码

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

## 单步检查

| PC | 指令 | 重点观察 |
|---:|---|---|
| 0 | LI R1,12 | R1=12 |
| 1 | LI R2,8 | R2=8 |
| 2 | SUB R3,R1,R2 | R3、Zero、Borrow |
| 3 | BZ 9 | 是否读取前一条 SUB 的 Zero |
| 4 | BB 7 | 是否读取前一条 SUB 的 Borrow |
| 5 | ADD R1,R3,R0 | R1 是否复制 R3 |
| 6 | JMP 2 | 是否返回比较 |
| 7 | SUB R2,R2,R1 | 是否更新 R2 |
| 8 | JMP 2 | 是否返回比较 |
| 9 | ST R1 | LED_OUT 是否显示 4 |

## 示例执行

```text
初始：R1=12，R2=8

12-8=4：R1更新为4
4-8：Borrow=1，R2更新为8-4=4
4-4=0：Zero=1，跳到ST

LED_OUT=4
```

---

# 八、控制信号汇总

| 指令 | RegWrite | FlagWrite | LedWrite | 跳转 |
|---|---:|---:|---:|---:|
| NOP | 0 | 0 | 0 | 0 |
| LI | 1 | 0 | 0 | 0 |
| ADD | 1 | 0 | 0 | 0 |
| SUB | 1 | 1 | 0 | 0 |
| AND | 1 | 0 | 0 | 0 |
| OR | 1 | 0 | 0 | 0 |
| ST | 0 | 0 | 1 | 0 |
| JMP | 0 | 0 | 0 | 1 |
| BZ | 0 | 0 | 0 | Zero |
| BB | 0 | 0 | 0 | Borrow |

---

# 九、关键时序检查

- PC、寄存器组、状态寄存器和 LED_OUT 只在有效时钟沿更新；
- ROM、译码、寄存器读取、ALU 和 MUX 属于组合逻辑；
- 时钟沿到来前，写回数据和 `PC_next` 必须稳定；
- 跳转指令只读取状态，不更新状态；
- SUB 写回目标寄存器和状态寄存器应在同一有效时钟沿完成；
- 不允许机械按键抖动导致 PC 或寄存器一次按键更新多次。

---

# 十、常见错误

1. PC 不是 4 bit，导致 ROM 地址范围不一致；
2. ROM 不是 16×16 bit；
3. R0 被写入；
4. ADD、AND、OR 错误更新 Zero 或 Borrow；
5. BZ、BB 在跳转时覆盖状态；
6. Borrow 极性与实际芯片 Cout 关系错误；
7. PC MUX 放置位置错误；
8. 仍保留 RAM、LD、MemRead 或 MemWrite；
9. ST 仍按存储器写指令接线，而不是写 LED_OUT；
10. ROM 机器码与指令字段位置不一致；
11. Multisim 按键未消抖；
12. 多人同时修改同一个 `.ms14` 文件造成二进制冲突。

---

# 十一、最终检查清单

- [ ] PC=4 bit；
- [ ] ROM=16×16 bit；
- [ ] R0 固定为 0；
- [ ] LI 正确；
- [ ] ADD、SUB、AND、OR 正确；
- [ ] 只有 SUB 更新 Zero、Borrow；
- [ ] Borrow 极性正确；
- [ ] JMP、BZ、BB 正确；
- [ ] ST 写 LED_OUT；
- [ ] 不存在数据 RAM 和 LD；
- [ ] ROM 机器码全部核对；
- [ ] `gcd(12,8)` 输出 4；
- [ ] 单步按键一次只执行一个周期。
