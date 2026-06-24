# 数字电路课程设计：8 位多寄存器 CPU 仿真与 4 位 ACC＋ALU 实物系统

本仓库记录数字电路课程设计的方案设计、Multisim 仿真、PCB 制作、实物调试与总结过程。

项目由两个相互关联、但实现范围不同的部分组成：

1. **CPU 仿真部分**：在 Multisim 中实现 8 位多寄存器、自定义指令集的简化 CPU；
2. **实物部分**：实现采用三位操作码控制的 4 位 ACC＋ALU 运算系统。

> 最终方案：CPU 不设置数据 RAM，中间数据保存在通用寄存器中；GCD 结果通过 `ST` 指令写入 LED 输出寄存器。实物部分采用 74LS 系列器件、三位操作码和 PCB 实现。

## 快速导航

| 内容 | 文件或目录 |
|---|---|
| CPU 最终技术方案 | [`docs/cpu-technical-spec.md`](docs/cpu-technical-spec.md) |
| CPU 分阶段实现指南 | [`docs/cpu-implementation-guide.md`](docs/cpu-implementation-guide.md) |
| ACC＋ALU 最终设计指南 | [`docs/acc-alu-design-guide.md`](docs/acc-alu-design-guide.md) |
| ACC＋ALU 项目简介 | [`docs/acc-alu-project-overview.md`](docs/acc-alu-project-overview.md) |
| Multisim 工程 | [`circuit/`](circuit/) |
| 基础电路截图 | [`screenshots/base-system/`](screenshots/base-system/) |
| ACC 初版截图 | [`screenshots/acc-prototype/`](screenshots/acc-prototype/) |

## 一、项目目标

- 完成 8 位多寄存器 CPU 的模块划分与 Multisim 仿真；
- 设计 16 位自定义指令集和控制逻辑；
- 实现 PC、ROM、寄存器组、ALU、状态标志和跳转控制；
- 使用寄存器保存中间数据，不设置数据 RAM；
- 使用 GCD 程序验证循环、比较、减法和跳转；
- 使用 `ST` 指令将结果写入 LED 输出寄存器；
- 实现采用 `OP2 OP1 OP0` 控制的 4 位 ACC＋ALU；
- 完成器件选型、原理图、PCB、焊接和实物调试。

## 二、CPU 仿真系统

CPU 仿真部分包括：

- 4 位程序计数器 PC；
- 16×16 bit 指令 ROM；
- 指令译码与控制单元；
- 8 个 8 位通用寄存器 R0～R7；
- 8 位 ALU；
- Zero、Borrow 状态标志；
- JMP、BZ、BB 跳转逻辑；
- 8 位 LED 输出寄存器。

```text
                 ┌──────────── 跳转地址 ────────────┐
                 │                                  ▼
PC → ROM → 指令译码 → 寄存器组 → ALU → 寄存器写回 → PC选择
                              │       │
                              │       └→ Zero / Borrow
                              └────────→ LED_OUT（ST）
```

### CPU 最终规格

| 项目 | 规格 |
|---|---|
| 数据宽度 | 8 bit |
| 指令宽度 | 16 bit |
| 通用寄存器 | R0～R7，共 8 个 |
| R0 | 固定为 0 |
| PC | 4 bit |
| 指令 ROM | 16×16 bit |
| 数据 RAM | 不使用 |
| ALU | ADD、SUB、AND、OR |
| 状态标志 | Zero、Borrow，仅由 SUB 更新 |
| 跳转 | JMP、BZ、BB |
| 输出 | `ST` 写 LED_OUT |
| 验证程序 | 辗转相减法 GCD |

## 三、自定义指令集

| opcode | 指令 | 功能 |
|---|---|---|
| `0000` | `NOP` | 空操作 |
| `0001` | `LI rd, imm8` | 将立即数写入寄存器 |
| `0010` | 保留 | 当前方案不使用 |
| `0011` | `ST rs` | 将寄存器数据写入 LED_OUT |
| `0100` | `ADD rd, rs1, rs2` | 两个寄存器相加 |
| `0101` | `SUB rd, rs1, rs2` | 相减并更新 Zero、Borrow |
| `0110` | `AND rd, rs1, rs2` | 按位与 |
| `0111` | `OR rd, rs1, rs2` | 按位或 |
| `1000` | `JMP addr` | 无条件跳转 |
| `1001` | `BZ addr` | Zero=1 时跳转 |
| `1010` | `BB addr` | Borrow=1 时跳转 |

```text
JumpEnable = isJMP
             OR (isBZ AND Zero)
             OR (isBB AND Borrow)
```

## 四、GCD 验证程序

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

### ROM 机器码

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

预期结果：

```text
gcd(12, 8) = 4
LED_OUT = 0000_0100
```

## 五、ACC＋ALU 实物系统

实物采用累加器型数据通路：

```text
A 输入 ───────────────┐
                      ├──→ ACC输入选择 → 74LS175 ACC → LED
ALU_RESULT ───────────┘                    │
                                           ▼
                                      4 位 ALU
                                      ▲      ▲
                                   ACC反馈   B输入
```

### 三位操作码

实物 PCB 上使用 `OP2 OP1 OP0` 三位操作码：

| OP2 | OP1 | OP0 | 功能 | 执行结果 |
|---:|---:|---:|---|---|
| 0 | 0 | 0 | CLEAR | `ACC ← 0000` |
| 0 | 0 | 1 | LOAD | `ACC ← A` |
| 0 | 1 | 0 | ADD | `ACC ← ACC + B` |
| 0 | 1 | 1 | SUB | `ACC ← ACC - B` |
| 1 | 0 | 0 | AND | `ACC ← ACC AND B` |
| 1 | 0 | 1 | OR | `ACC ← ACC OR B` |
| 1 | 1 | 0 | 保留 | 后续扩展 |
| 1 | 1 | 1 | 保留 | 后续扩展 |

操作码负责选择功能，时钟/执行按键负责将所选结果写入 ACC。若最终原理图另设独立复位按键，它作为辅助硬件清零，不改变上述操作码定义。

### 主要器件

| 器件 | 作用 |
|---|---|
| 74LS175 | 4 位 ACC 数据保存 |
| 74LS157 | A 输入与 ALU_RESULT 选择 |
| 74LS14 | 按键消抖和信号整形 |
| 74LS07 | 缓冲及 LED 驱动 |
| 74LS 系列逻辑器件 | ALU 运算、译码和状态判断 |
| LED、限流电阻 | 数据与状态显示 |
| 拨码开关 | A、B 和 OP2～OP0 输入 |
| 轻触按键 | 执行时钟及辅助控制 |

本项目最终采用 74LS 系列器件。普通 74HC 与 74LS 不应被描述为可以随意混用；器件替换必须核对输入门限、输出电平和驱动能力。

## 六、项目成果状态

| 内容 | 状态 |
|---|---|
| CPU 总体方案与指令集 | 已完成 |
| PC、ROM、寄存器组和 ALU | 已完成 |
| Zero、Borrow 与跳转逻辑 | 已完成 |
| 无 RAM 数据通路 | 已确定 |
| GCD 程序及 ROM 机器码 | 已完成 |
| 三位操作码 ACC＋ALU | 已完成 |
| 嘉立创原理图与 PCB | 已完成，工程资料待继续整理上传 |
| 器件清单 | 已完成，待补充到 `hardware/bom/` |
| 实物焊接与测试 | 已完成，照片待补充到 `photos/` |

## 七、当前仓库结构

```text
multisim-course-design/
├── README.md
├── circuit/
│   ├── cpu-simulation-v0.5.ms14
│   ├── acc-module-test.ms14
│   ├── alu-module.ms14
│   └── acc-alu-system-v1.ms14
├── docs/
│   ├── cpu-technical-spec.md
│   ├── cpu-implementation-guide.md
│   ├── acc-alu-design-guide.md
│   └── acc-alu-project-overview.md
└── screenshots/
    ├── base-system/
    │   ├── acc-module.png
    │   ├── alu-module.png
    │   ├── decoder-module.png
    │   └── system-top.png
    └── acc-prototype/
        └── acc.png
```

后续建议补充：

```text
hardware/
├── schematic/
├── pcb/
└── bom/

photos/
reports/
```

## 八、个人工作

本人主要负责 ACC＋ALU 实物系统的工程实现，包括：

- 设计并接入 ACC 累加寄存器；
- 分析 ACC、ALU、三位操作码和时钟控制关系；
- 整理芯片、阻容器件、接口和辅助材料清单；
- 在嘉立创 EDA 中完成相关原理图、封装检查和 PCB 设计；
- 参与 PCB 焊接、上电测试和问题排查；
- 学习 PC、ROM、寄存器组、自定义指令和 CPU 数据通路。

CPU 整体仿真方案由小组协作完成，本人重点承担 ACC＋ALU 实物系统的落地与验证。

## 九、命名规范

- 文件名统一使用英文小写；
- 单词之间使用连字符 `-`；
- Multisim 工程保留 `.ms14` 扩展名；
- 截图保留 `.png` 扩展名；
- 版本和日期放在文件名末尾；
- 避免使用 `test`、`final`、`Design1`、`第一次` 等含义不清的名称。

## 十、说明

- 仿真部分为 8 位多寄存器 CPU；
- 实物部分为采用三位操作码控制的 4 位 ACC＋ALU；
- CPU 最终方案不包含数据 RAM；
- 实物最终方案采用 74LS 系列和 PCB；
- 小组电路、PCB 和实物资料属于小组共同成果；
- 个人报告和个人总结由各成员独立完成。
