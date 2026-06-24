# 基于三位操作码控制的 4 位 ACC＋ALU 项目简介

## 一、项目概述

本项目实现一个采用 `OP2 OP1 OP0` 三位操作码控制的 4 位 ACC＋ALU 运算系统。

系统能够完成 CLEAR、LOAD、ADD、SUB、AND、OR 六种功能，并使用 ACC 保存结果、参与后续运算。本项目不是完整 CPU，不包含 PC、指令 ROM 和完整取指流程。

## 二、实际数据通路

根据顶层与 ALU 子模块：

```text
外部 A → ALU A端
ACC    → ALU B端
外部 B → ACC 的 LOAD 输入
ALU_RESULT → ACC 的运算结果输入
```

因此：

```text
LOAD：ACC ← B
ADD ：ACC ← A + ACC
SUB ：ACC ← A - ACC
AND ：ACC ← A AND ACC
OR  ：ACC ← A OR ACC
```

减法由 `A + (~ACC) + 1` 实现，因此方向已经确认是 `A - ACC`。

## 三、操作码

| OP2 | OP1 | OP0 | 功能 | 执行结果 |
|---:|---:|---:|---|---|
| 0 | 0 | 0 | CLEAR | `ACC ← 0000` |
| 0 | 0 | 1 | LOAD | `ACC ← B` |
| 0 | 1 | 0 | ADD | `ACC ← A + ACC` |
| 0 | 1 | 1 | SUB | `ACC ← A - ACC` |
| 1 | 0 | 0 | AND | `ACC ← A AND ACC` |
| 1 | 0 | 1 | OR | `ACC ← A OR ACC` |
| 1 | 1 | 0 | 保留 | 必须确保 ACC 保持 |
| 1 | 1 | 1 | 保留 | 必须确保 ACC 保持 |

## 四、关键硬件检查

1. U26、U27 的 74LS253 低有效输出使能 `/1G`、`/2G` 必须接 GND。
2. U4A 的 74LS74 低有效 `/PRE`、`/CLR` 必须接确定电平；未使用时接 VCC。
3. `110`、`111` 不能在按下时钟后误写 ACC，推荐实现 `ACC_next=ACC`。
4. 当前 Carry 只在 ADD 时有效，不能当作 SUB 的 Borrow。
5. 当前顶层未看到独立 Zero 输出，Zero 应标为未实现或待补充。

## 五、基本操作

```text
OP=000：CLEAR
B=0011，OP=001：LOAD → ACC=0011
A=0010，OP=010：ADD  → ACC=0101
A=0111，OP=011：SUB  → ACC=0010
```

## 六、验收清单

- [ ] OP0、OP1、OP2 位序正确；
- [ ] CLEAR、LOAD、ADD、SUB、AND、OR 正确；
- [ ] SUB 结果符合 `A-ACC`；
- [ ] `110`、`111` 不会破坏 ACC；
- [ ] 74LS253 使能端已接 GND；
- [ ] 74LS74 异步端已接确定电平；
- [ ] 单次按键只更新一次 ACC；
- [ ] Carry 只按 ADD 进位解释；
- [ ] Zero 的实现状态已明确；
- [ ] PCB 电源和地无短路；
- [ ] 实物运行稳定。
