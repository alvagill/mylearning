# 一生一芯 NPC 全流程指令与工程手册

## 整体数据流图（先建立全局视角）

```
[软件侧] C代码 -> .o -> .elf -> .bin -> .hex
[硬件侧] RTL(.v) --iverilog/verilator--> 仿真 --yosys--> 网表(.v) --ECOS--> 版图(GDSII)
```

---

## 第一阶段：软件侧指令（把 C 代码变成 CPU 能读的指令）

### 1. `riscv64-unknown-elf-gcc` —— 交叉编译器

**指令示例**：

```bash
riscv64-unknown-elf-gcc -march=rv32i -mabi=ilp32 -nostdlib -ffreestanding -c add.c -o add.o
```

- **在干什么**：在你的 x86 电脑上，把 C 语言编译成 RISC-V 架构的汇编，再汇编成机器码目标文件。`riscv64` 前缀说明它是交叉编译器。
- **关键参数**：
  - `-march=rv32i`：目标架构是 32 位基础整数指令集。
  - `-mabi=ilp32`：ABI 约定，int/long/pointer 都是 32 位。
  - `-nostdlib`：不链接标准库（裸机没有 printf/malloc）。
  - `-c`：只编译不链接。
- **生成文件**：`add.o`（目标文件，含机器码，但地址尚未最终确定）。

### 2. `riscv64-unknown-elf-ld` —— 链接器

**指令示例**：

```bash
riscv64-unknown-elf-ld -Ttext=0x80000000 add.o -o add.elf
```

- **在干什么**：把多个 `.o` 文件拼接起来，解决符号引用（比如调用函数时的跳转地址），并根据链接脚本分配内存地址。
- **关键参数**：
  - `-Ttext=0x80000000`：**极其关键！** 告诉链接器，代码段（`.text`）的起始地址是 `0x80000000`。这必须与 NPC 的 `RESET_PC` 完全一致。
- **生成文件**：`add.elf`（可执行文件，包含机器码、数据、ELF 头、符号表、调试信息）。

### 3. `riscv64-unknown-elf-objcopy` —— 格式提取器

**指令示例**：

```bash
riscv64-unknown-elf-objcopy -O binary add.elf add.bin
```

- **在干什么**：剥离 ELF 文件里的"外壳"（文件头、符号表、调试信息），只留下纯粹的机器码和数据。
- **关键参数**：`-O binary` 指定输出格式为纯二进制。
- **生成文件**：`add.bin`（纯机器码，CPU 真正执行的内容）。

### 4. `od` —— 八进制/十六进制转储工具

**指令示例**：

```bash
od -An -tx4 -v add.bin > program.hex
```

- **在干什么**：把不可读的二进制 `.bin` 文件，转换成一行行十六进制文本。因为 iverilog 的 `$readmemh` 只认这种文本格式。
- **关键参数**：
  - `-An`：不打印地址偏移。
  - `-tx4`：以 4 字节（32 位）为单位，输出十六进制。
  - `-v`：不压缩重复行（防止出现 `*` 号导致读取失败）。
- **生成文件**：`program.hex`（每行一个 32 位十六进制数，小端序，供仿真器读取）。

---

## 第二阶段：RTL 仿真侧指令（在电脑里模拟电路）

### 1. `iverilog` —— RTL 四值仿真编译器

**指令示例**：

```bash
iverilog -g2012 -s tb -o build/sim_npc tb/tb.v vsrc/*.v
```

- **在干什么**：读取 Verilog 源码和 Testbench，检查语法，生成一个仿真可执行文件（`.vvp`）。
- **关键参数**：
  - `-g2012`：启用 SystemVerilog 语法支持（代码里有 `10_000_000` 和数组端口必须加）。
  - `-s tb`：显式指定顶层模块为 tb，防止工具认错顶层。
  - `-o`：指定输出文件名。
- **生成文件**：`build/sim_npc`（仿真可执行文件）。

### 2. `vvp` —— iverilog 仿真运行引擎

**指令示例**：

```bash
vvp build/sim_npc
```

- **在干什么**：真正执行仿真，模拟时间流逝。执行 `tb.v` 里的 `initial` 块，产生时钟、复位，读取 `program.hex` 进入内存，并驱动 NPC 运行。
- **生成文件**：`npc.vcd`（波形文件，由 `$dumpfile` 指定）。

### 3. `gtkwave` —— 波形查看器

**指令示例**：

```bash
gtkwave npc.vcd
```

- **在干什么**：打开 `.vcd` 波形文件，提供图形化界面。红色代表 X 态（不定态），蓝色代表 Z 态（高阻态）。用于追查未复位触发器、总线死锁等问题。

### 4. `verilator` —— RTL 二值仿真器

**指令示例**：

```bash
verilator --cc --exe --build -O3 --x-assign fast --x-initial fast --noassert \
  -GRESET_PC=0x80000000 -CFLAGS "-std=c++17" --top-module NPC vsrc/*.v csrc/sim_main.cpp
```

- **在干什么**：把 Verilog 转换成 C++ 代码，再编译成可执行文件。速度极快，支持 DPI-C（用 C++ 写内存模型和 DiffTest）。
- **关键参数**：
  - `--cc --exe --build`：生成 C++ 代码，生成可执行文件，并自动调用 g++ 编译。
  - `--x-assign fast` / `--x-initial fast`：把未初始化的 X 态当作 0 处理（掩盖复位缺陷，加快仿真）。
  - `-GRESET_PC=0x80000000`：**Verilator 专属的参数传递方式！** 把参数从 C++ 传递给 RTL 模块。
- **生成文件**：`build/NPC`（可执行文件），`build/obj_dir/`（C++ 中间文件）。

### 5. `make` / `make iverilog` / `make run` —— Makefile 封装

- **在干什么**：把上面那些繁琐的编译命令写成脚本，一键执行。
- **对比**：
  - `make` / `make default`：调用 Verilator 生成 `build/NPC`，不自动运行。
  - `make iverilog`：调用 iverilog 生成 `sim.vvp`，并自动运行 vvp。
  - `make run BIN_PATH=...`：运行 `build/NPC`，传入测试程序路径。

---

## 第三阶段：逻辑综合侧指令（RTL 变门级网表）

### 1. `yosys` 交互模式与 `syn.ys` 脚本

**指令示例**：

```bash
yosys -l build/syn.log syn.ys
```

- **在干什么**：执行 `syn.ys` 脚本，把行为级 Verilog 转化为由标准单元（与门、或门、触发器）构成的门级网表。
- **`syn.ys` 内部核心命令拆解**：
  - `read_verilog -sv -DSYNTHESIS vsrc/*.v`：读入 RTL。`-sv` 支持 SystemVerilog 语法；`-DSYNTHESIS` 宏让 ``ifdef SYNTHESIS`` 包裹的仿真代码不被综合。
  - `chparam -set RESET_PC 32'h80000000 NPC`：**核心排雷指令！** Yosys 不认 Makefile 里的 `-G` 参数，必须在展开层次前（`hierarchy` 之前）显式覆写参数，否则会烤进默认的 `0x30000000`，导致死锁。
  - `hierarchy -check -top NPC`：检查并展开模块层次，指定顶层为 NPC。
  - `synth -top NPC`：执行通用逻辑综合，优化电路。
  - `dfflibmap -liberty ...`：映射 D 触发器到工艺库（`.lib`）中的具体单元。
  - `abc -liberty ...`：映射组合逻辑门到工艺库。
  - `write_verilog -noattr build/NPC_Synthesis.v`：输出门级网表。注意：给仿真用应加 `_sim`，给后端用应不加 `_sim`。
- **生成文件**：`build/NPC_Synthesis.v`（门级网表），`build/syn.log`（综合日志）。

---

## 第四阶段：门级网表仿真侧指令（验证综合结果）

**指令示例**：

```bash
iverilog -g2012 -DSYNTHESIS -s tb -o build/sim_netlist \
  tb/tb.v build/NPC_Synthesis.v /path/to/PDK/verilog/ics55_LLSC_H7CL.v
```

- **在干什么**：把门级网表当成源代码来仿真。与 RTL 仿真不同，这里必须引入标准单元行为模型（PDK 的 `.v` 文件），否则仿真器不认识 `AND2X1` 等门。
- **关键参数**：
  - `-DSYNTHESIS`：告诉 `tb.v`，现在是网表仿真阶段，请隐藏/删除那些被 Yosys 优化掉的 `debug_*` 端口。
  - `/path/to/ics55_LLSC_H7CL.v`：标准单元模型，必须与综合时使用的 `.lib` 库严格匹配（都是 H7CL）。
- **生成文件**：`build/sim_netlist`（仿真可执行文件）。
- **运行**：`cd build && ./sim_netlist`（配合 `make run-netlist`）。
- **注意**：网表仿真只能看顶层总线握手（`ifu_reqValid` 等），内部信号全被优化成 `_00123_` 乱码，速度极慢。

---

## 第五阶段：后端物理设计（ECOS Studio 相关）

**输入文件**：不带 `_sim` 的干净物理网表（`NPC_Synthesis.v`）。

**核心步骤**：

- **Floorplan（布局规划）**：确定芯片面积（Die Area）和利用率（Core Utility）。
- **Placement（布局）**：摆放标准单元。
- **CTS（时钟树综合）**：平衡时钟偏差。
- **Route（布线）**：连接所有逻辑门。
- **Signoff（签核）**：DRC、LVS、STA 检查。

**关键指标**：

- **Setup TNS/WNS**：建立时间违例（负数表示超频）。
- **Hold TNS/WNS**：保持时间违例。
- **DRC Number**：设计规则检查（0 为完美）。

**修复手段**：降频（改 Frequency）、降利用率（改 Origin Core Utilization）。

**导出**：File -> Export Signoff Package，生成 `.tar.gz` 签核包。

---

## 附录：工程化避坑核心总结

- **参数传递链**：RTL 仿真用 `-G`；Yosys 综合用 `chparam`；全程地址（如 `0x80000000`）必须严格一致。
- **文件区分**：
  - `.bin` -> `od` -> `.hex`：给 `$readmemh` 读。
  - `_sim` 网表 -> 给 iverilog 仿真。
  - 无 `_sim` 网表 -> 给 ECOS Studio 做后端。
- **X 态排查**：iverilog 特有，红线追溯到源头寄存器，补复位。
- **Makefile 与 clean**：`make clean` 会删掉整个 `build/`，跑后端前务必确保网表已备份。
