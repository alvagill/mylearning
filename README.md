# 芯片学习记录

电子信息工程 26 级学生，跟随「一生一芯」（YSYX）计划学习处理器芯片设计，目标是往智能芯片算法与硬件设计方向发展。

本仓库是自己的学习笔记与阶段总结，按 D / E / F 三个阶段组织，记录知识要点、踩坑经验与可复用的调试方法。

## 目录结构

```
mylearning/
├── D阶段/                      # 软件与计算机系统基础
│   ├── D1 D2 C语言底层与表达式求值/   # C 语言底层、编译流程、表达式求值与调试器
│   ├── D3 NEMU模拟器与RISC-V指令实现/  # NEMU 模拟器与 RISC-V 指令实现
│   ├── D4 程序的机器级表示/            # 程序的机器级表示
│   ├── D5 AM抽象层/                    # 机器级表示到 AM 抽象层
│   ├── D6 从RTL到可流片版图/           # 从 RTL 代码到可流片版图
│   ├── D7 NEMU调试基础设施与验证体系/   # NEMU 调试基础设施与验证体系
│   └── D8 NEMU调试基建与DiffTest/      # NEMU 调试基建与 DiffTest 协同架构
├── E阶段/                      # 数字电路与处理器实现
│   ├── E1 C语言学习/            # C 语言 API 与《Learn C the Hard Way》总结
│   ├── E2 Verilog 数字电路设计.md
│   ├── E3 Linux系统安装和基本使用.md
│   ├── E4 处理器的仿真和验证.md
│   ├── E5 功能完备的迷你RISC-V处理器.md
│   ├── E6 简易运行时环境.md
│   ├── E7 简易总线和SoC.md
│   └── E8 后端物理设计和流片准备/  # 含 NPC 全流程指令与工程手册
└── F阶段/                      # 架构与数字逻辑进阶
    ├── 处理器架构/              # minirv 单周期处理器、GPR.circ（Logisim 工程）
    └── 数字逻辑电路基础/         # 门电路到 sCPU 的 Logisim 实践
```

## 各阶段内容

### D 阶段：软件视角的计算机系统

| 笔记 | 内容概要 |
| :--- | :--- |
| [一生一芯 D1 D2 阶段核心知识总结](D阶段/D1%20D2%20C语言底层与表达式求值/一生一芯%20D1%20D2%20阶段核心知识总结.md) | 编译四阶段与工具链、宏与条件编译、C 标准中的 UB / 未指定行为 / 实现定义行为、`volatile`、ABI、程序生命周期；表达式求值、监视点与自动化测试 |
| D2 阶段：10.30 / 11.1 笔记 | 课堂随记，补充习题与实验过程 |
| [NEMU 模拟器与 RISC-V 指令实现](D阶段/D3%20NEMU模拟器与RISC-V指令实现/一生一芯%20D3阶段知识点：NEMU%20模拟器与%20RISC-V%20指令实现.md) | NEMU 的执行流程、指令译码与实现要点 |
| [NEMU 代码导读](D阶段/D3%20NEMU模拟器与RISC-V指令实现/一生一芯-P7-NEMU代码导读.md) | P7 讲义配套的代码结构梳理 |
| [程序的机器级表示](D阶段/D4%20程序的机器级表示/一生一芯-P9-P10-程序的机器级表示.md) | P9/P10 讲义：寄存器约定、栈帧、过程调用与 ABI |
| [AM 抽象层](D阶段/D5%20AM抽象层/一生一芯-P11-AbstractMachine.md) | P11 讲义：AbstractMachine 的接口设计与裸机运行环境 |
| [D5 阶段进阶总结](D阶段/D5%20AM抽象层/一生一芯%20D5阶段进阶学习总结：从机器级表示到%20AM%20抽象层.md) | 从机器级表示到 AM 抽象层的串联梳理 |
| [D6 从 RTL 代码到可流片版图](D阶段/D6%20从RTL到可流片版图/“一生一芯”D6%20从RTL代码到可流片版图知识总结.md) | Verilog 事件模型与阻塞/非阻塞赋值、数据竞争、Yosys 逻辑综合三阶段与工艺映射、后端到版图流程 |
| [D7 NEMU 调试基础设施与验证体系](D阶段/D7%20NEMU调试基础设施与验证体系/“一生一芯”%20NEMU%20调试基础设施与验证体系学习总结.md) | itrace / iringbuf / mtrace / ftrace 实现与 ELF 符号解析、klib 本地验证与 Canary 测试、DiffTest 与 Spike 对标、死循环检测、`getopt_long` |
| [D8 NEMU 调试基建与 DiffTest 协同架构](D阶段/D8%20NEMU调试基建与DiffTest/“一生一芯”%20NEMU%20调试基建与%20DiffTest%20协同架构总结.md) | 表驱动式 SDB 命令注册、`readline`/`strtok` 与信号处理、总线请求快照与 RTL 调试口对齐、DUT/REF 与 `.so` 动态加载、MMIO 跳过机制 |

### E 阶段：从 Verilog 到可流片的 NPC

| 笔记 | 内容概要 |
| :--- | :--- |
| [E1 C 语言学习笔记](E阶段/E1%20C语言学习/E1%20C语言学习笔记.md) | 文件 IO、标准库常用函数速查 |
| [LCTHW 知识点总结](E阶段/E1%20C语言学习/LCTHW_知识点总结.md) | 《Learn C the Hard Way》系统性总结 |
| [E2 Verilog 数字电路设计](E阶段/E2%20Verilog%20数字电路设计.md) | Verilog 语法、可综合写法与时序逻辑 |
| [E3 Linux 系统安装和基本使用](E阶段/E3%20Linux系统安装和基本使用.md) | Linux 环境搭建与常用命令 |
| [E4 处理器的仿真和验证](E阶段/E4%20处理器的仿真和验证.md) | iverilog / Verilator 仿真流程与 DiffTest 验证 |
| [E5 功能完备的迷你 RISC-V 处理器](E阶段/E5%20功能完备的迷你RISC-V处理器.md) | 数据通路、控制信号、关键 Bug 与调试方法论 |
| [E6 简易运行时环境](E阶段/E6%20简易运行时环境.md) | 裸机运行时、链接脚本与启动流程 |
| [E7 简易总线和 SoC](E阶段/E7%20简易总线和SoC.md) | SimpleBus 握手、地址译码、UART 与 MMIO |
| [E8 后端物理设计和流片准备](E阶段/E8%20后端物理设计和流片准备/E8%20后端物理设计和流片准备.md) | 综合、布局布线、时序收敛与流片检查 |
| [NPC 全流程指令与工程手册](E阶段/E8%20后端物理设计和流片准备/一生一芯%20NPC%20全流程指令与工程手册.md) | 从 RTL 到门级网表、后端交付的完整命令清单 |

### F 阶段：架构与数字逻辑进阶

| 笔记 | 内容概要 |
| :--- | :--- |
| [minirv 处理器](F阶段/处理器架构/minirv处理器.md) | RV32I 寄存器、指令格式与立即数扩展，单周期处理器设计 |
| [GPR.circ](F阶段/处理器架构/GPR.circ) | Logisim Evolution 绘制的通用寄存器堆电路 |
| [从门电路到 sCPU](F阶段/数字逻辑电路基础/从门电路到%20sCPU：Logisim%20位切片%20CPU%20设计实战.md) | MUX / D 触发器 / 分线器使用避坑，标准 GPR 与位切片架构，sCPU 数据通路 |
| [Fundamentals of Digital Logic](F阶段/数字逻辑电路基础/Fundamentals%20of%20Digital%20Logic.md) | 数字逻辑基础要点摘录 |

## 阅读与使用说明

- 笔记均为 Markdown，建议使用支持数学公式（LaTeX / MathJax）的编辑器（Typora、Obsidian、VS Code + Markdown Preview Enhanced）。
- `.circ` 文件需要用 [Logisim Evolution](https://github.com/logisim-evolution/logisim-evolution) 打开。
- 笔记中的示例命令以 Linux 环境为准（E3 记录了环境搭建过程）；仿真部分同时涉及 iverilog 与 Verilator，DiffTest 需要与 NEMU 参考模型对照运行。
- 内容为个人学习整理，可能存在理解偏差，欢迎指出错误。

## 学习进度

- D 阶段：进行中，D8（NEMU 调试基建与 DiffTest 协同）
- E 阶段：已完成
- F 阶段：已完成
