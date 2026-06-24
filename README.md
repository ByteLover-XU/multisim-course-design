# 数字电路课程设计：8 位多寄存器 CPU 仿真与 4 位 ACC＋ALU 实物系统

本仓库记录数字电路课程设计的方案设计、Multisim 仿真、PCB 制作、实物调试与总结过程。

项目由两个相互关联、但实现范围不同的部分组成：

1. **CPU 仿真部分**：在 Multisim 中实现 8 位多寄存器、自定义指令集的简化 CPU；
2. **实物部分**：实现采用三位操作码控制的 4 位 ACC＋ALU 运算系统。

> 最终方案：CPU 不设置数据 RAM，中间数据保存在通用寄存器中；GCD 结果通过 `ST` 指令写入 LED 输出寄存器。实物部分采用三位操作码和 PCB 实现。

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

从当前顶层原理图可确定的数据通路为：

```text
外部 A ───────────────→ ALU A端 ───────────┐
ACC 输出 ─────────────→ ALU B端 ───────────┤
                                              ▼
                                          ALU_RESULT
                                              │
外部 B ───────────────┐                       │
                      ├──→ ACC输入选择 ───────┘
ALU_RESULT ───────────┘
                              │
                              ▼
                             ACC → LED
```

因此，当前电路的含义是：

- 外部 **B** 用于 LOAD；
- 外部 **A** 作为 ALU 外部操作数；
- ACC 反馈作为 ALU 另一操作数；
- ALU_RESULT 写回 ACC。

### 三位操作码

| OP2 | OP1 | OP0 | 功能 | 当前顶层数据通路对应含义 |
|---:|---:|---:|---|---|
| 0 | 0 | 0 | CLEAR | `ACC ← 0000` |
| 0 | 0 | 1 | LOAD | `ACC ← B` |
| 0 | 1 | 0 | ADD | `ACC ← A + ACC` |
| 0 | 1 | 1 | SUB | 由 ALU 内部减法方向决定 |
| 1 | 0 | 0 | AND | `ACC ← A AND ACC` |
| 1 | 0 | 1 | OR | `ACC ← A OR ACC` |
| 1 | 1 | 0 | 保留 | 当前行为未定义 |
| 1 | 1 | 1 | 保留 | 当前行为未定义 |

> `SUB` 的精确表达式必须以 ALU 子模块内部连接为准：如果内部实现 `A-B`，则当前顶层为 `A-ACC`；如果内部实现 `B-A`，则为 `ACC-A`。

### 译码逻辑

当前译码模块使用三线—八线译码器：

- `Y0` 直接作为低有效 `CLEAR_N`；
- `Y1～Y5` 经反相后生成高有效 `LOAD`、`ADD_EN`、`SUB_EN`、`AND_EN`、`OR_EN`；
- `110`、`111` 为保留码。

在保留码下，如果按下时钟，ACC 是否保持取决于 ALU 和 ACC 子模块的默认行为。最终硬件应确保 `110`、`111` 不会把未知值写入 ACC，推荐实现 HOLD 或屏蔽时钟。

### 显示与状态

当前顶层截图中明确显示：

- ACC 四位输出；
- Carry 输出。

当前顶层未看到独立 Zero 输出，因此 Zero 应标记为“可选扩展”或补充实际 Zero 电路后再写为已实现。

### 器件说明

Multisim 译码截图使用了 74HC138 和 74HC04 仿真模型；实际 PCB/BOM 若使用 74LS138、74LS04，应在文档中区分“仿真模型”和“实物型号”。普通 74HC 与 74LS 不应描述为可以随意混用。

## 六、项目成果状态

| 内容 | 状态 |
|---|---|
| CPU 总体方案与指令集 | 已完成 |
| PC、ROM、寄存器组和 ALU | 已完成 |
| Zero、Borrow 与跳转逻辑 | 已完成 |
| 无 RAM 数据通路 | 已确定 |
| GCD 程序及 ROM 机器码 | 已完成 |
| 三位操作码 ACC＋ALU | 已完成 |
| ACC 四位显示与 Carry | 已完成 |
| Zero 显示 | 当前顶层未见，待确认或补充 |
| `110`、`111` 安全 HOLD | 待确认 |
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
- ACC＋ALU 的减法方向需以 ALU 子模块内部电路为准；
- 小组电路、PCB 和实物资料属于小组共同成果；
- 个人报告和个人总结由各成员独立完成。
