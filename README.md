# 数字电路课程设计：8 位 CPU 仿真与 4 位 ACC＋ALU 实物系统

本仓库记录数字电路课程设计的 Multisim 仿真、ACC＋ALU 实物、PCB 和技术文档。

项目分为两部分：

1. **CPU 仿真**：8 位多寄存器、自定义指令集 CPU；
2. **实物系统**：三位操作码控制的 4 位 ACC＋ALU。

## 快速导航

| 内容 | 路径 |
|---|---|
| CPU 技术方案 | [`docs/cpu-technical-spec.md`](docs/cpu-technical-spec.md) |
| CPU 实现指南 | [`docs/cpu-implementation-guide.md`](docs/cpu-implementation-guide.md) |
| ACC＋ALU 设计指南 | [`docs/acc-alu-design-guide.md`](docs/acc-alu-design-guide.md) |
| ACC＋ALU 项目简介 | [`docs/acc-alu-project-overview.md`](docs/acc-alu-project-overview.md) |
| Multisim 工程 | [`circuit/`](circuit/) |
| 截图 | [`screenshots/`](screenshots/) |

## 一、CPU 仿真

### 最终规格

| 项目 | 规格 |
|---|---|
| 数据宽度 | 8 bit |
| 指令宽度 | 16 bit |
| 通用寄存器 | R0～R7，R0 固定为 0 |
| PC | 4 bit |
| ROM | 16×16 bit |
| 数据 RAM | 不使用 |
| ALU | ADD、SUB、AND、OR |
| 状态标志 | Zero、Borrow，仅由 SUB 更新 |
| 跳转 | JMP、BZ、BB |
| 输出 | `ST` 写 LED_OUT |

### GCD 测试程序

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
LED_OUT = 0000_0100
```

## 二、ACC＋ALU 实物系统

### 实际数据通路

根据顶层和 ALU 子模块：

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

SUB 使用补码加法实现：

```text
A + (~ACC) + 1 = A - ACC
```

### 三位操作码

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

### 已确认的硬件检查项

1. U26、U27 的 74LS253 低有效 `/1G`、`/2G` 必须接 GND；
2. U4A 的 74LS74 低有效 `/PRE`、`/CLR` 必须接确定电平，未使用时接 VCC；
3. `110`、`111` 不能在按时钟后误写 ACC，推荐实现 HOLD；
4. 当前 Carry 只在 ADD 时有效，不能解释为 SUB 的 Borrow；
5. 当前顶层未看到独立 Zero 输出，Zero 暂列为未实现或待补充；
6. 仿真中的 74HC138、74HC04 与实物 BOM 型号必须分别注明，不能把普通 74HC 与 74LS 视为可随意混用。

## 三、仓库结构

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
    └── acc-prototype/
```

后续可补充：

```text
hardware/
├── schematic/
├── pcb/
└── bom/

photos/
reports/
```

## 四、说明

- CPU 仿真最终方案不包含数据 RAM；
- 实物部分为三位操作码控制的 4 位 ACC＋ALU；
- Multisim 工程是二进制文件，不适合多人同时修改；
- 文件名统一使用英文小写和连字符；
- 小组电路、PCB 和实物资料属于小组共同成果。
