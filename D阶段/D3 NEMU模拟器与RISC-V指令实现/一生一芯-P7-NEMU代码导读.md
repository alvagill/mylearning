---
title: 一生一芯 P7 · NEMU 代码导读 知识点讲义
series: 一生一芯 P7·P9·P10·P11 知识点讲义
lecture: P7（NEMU 代码导读，约 107 分钟）
tags:
  - 一生一芯
  - NEMU
  - Makefile
  - kconfig
  - 计算机系统
source: https://ysyx.oscc.cc/slides/2306/
created: 2026-11-04
---

# 一生一芯 P7 · NEMU 代码导读 知识点讲义

> 本册只含 P7 一讲：用一个 3500 行的真实项目，讲“怎么用 C 语言实现一台机器”。

**本册包含**：P7《NEMU 代码导读》全讲 + 本讲自测答案 + 术语表。

## 导读：这份讲义怎么用

| 标记 | 含义 | 你应当做到 |
| :---: | :--- | :--- |
| ★★★ | 核心必掌握 | 能不看讲义独立复述、推导或写出代码 |
| ★★ | 理解 | 能解释“为什么这样设计”，能读懂相关汇编/源码 |
| ★ | 了解 | 知道有这回事、需要时会查 |
| 重点（红） | 本节最关键的结论 | 先看结论，再看推导 |
| 易错（橙） | 反直觉、常考、常写错 | 复习时优先回看，建议抄进错题本 |
| 实操（蓝） | 命令、代码、可动手验证 | 边看边敲，否则看完就忘 |
| 考点（绿） | 自测题 / 面试高频问法 | 合上讲义作答，答案在附录 B |

**怎么读**：1.2 与 1.5 是重点。建议先 `make -nB` 看构建，再用 GDB 跟一遍 `init_monitor()`，最后对着 `decode_exec()` 抄一遍手册。

## 目录

- 第 1 讲 NEMU 代码导读（P7）
  - 1.1 本讲定位：NEMU 是“加强版 YEMU”
  - 1.2 make 与项目构建
  - 1.3 先跑起来：从运行输出认识 NEMU
  - 1.4 NEMU 的初始化：`init_monitor()` 里发生了什么
  - 1.5 指令执行：从 `cpu_exec()` 到 `decode_exec()`
  - 1.6 menuconfig 与 kconfig 配置系统
  - 1.7 本讲方法论：读代码 ≠ “读”代码
  - 1.8 自测（答案见附录 B）
- 附录 B 自测题答案要点
  - B.1 第 1 讲
- 附录 C 术语表
- 附录 D 参考资料

---

# 第 1 讲 NEMU 代码导读（P7）

> 视频时长 107 分钟。本章对应视频 P7，课件：`ysyx.oscc.cc/slides/2306/07.html`。

**视频索引**（点击时间可直接跳到对应片段）：

| 章节 | 起止时间 | 回看 |
| :--- | :---: | :--- |
| make 项目构建 | 00:00 – 44:20 | [00:00](https://www.bilibili.com/video/BV1up4y1j7Ji?t=0) |
| NEMU 的初始化 | 44:20 – 1:07:36 | [44:20](https://www.bilibili.com/video/BV1up4y1j7Ji?t=2660) |
| NEMU 指令执行选讲 | 1:07:36 – 1:29:20 | [1:07:36](https://www.bilibili.com/video/BV1up4y1j7Ji?t=4056) |
| menuconfig 配置系统 | 1:29:20 – 1:45:12 | [1:29:20](https://www.bilibili.com/video/BV1up4y1j7Ji?t=5360) |
| 总结 | 1:45:12 – 1:47:10 | [1:45:12](https://www.bilibili.com/video/BV1up4y1j7Ji?t=6312) |

## 1.1 本讲定位：NEMU 是“加强版 YEMU”

### 1.1.1 从 YEMU 到 NEMU ★★

前两讲已经建立了两个基本事实：

- C 程序在模拟器上如何执行：**YEMU 用 C 语言实现 ISA 状态机** S = <R, M>。
- C 程序在电路上如何执行：**YPC 用数字电路实现同一个状态机**；Verilator 用 C 语言实现数字电路的状态机。

这三者其实是同一个东西的三种实现。区别只在“指令实现得多不多”：**多实现一条指令，模拟器/处理器就完整一分**。NEMU 就是“指令实现得很全的 YEMU”。

所以本讲要解决的不是“怎么再写一个 YEMU”，而是工程问题：一个 3500 行、50 多个源文件的项目，**怎么构建、怎么初始化、怎么执行一条指令、怎么配置**。

### 1.1.2 本讲的四个板块 ★

| 板块 | 核心问题 |
| :--- | :--- |
| make 项目构建 | 一堆 .c 文件怎么变成可执行文件？我该怎样读懂别人的 Makefile？ |
| NEMU 的初始化 | `main()` 之后到“可以执行指令”之前，发生了哪些事？ |
| 指令执行选讲 | `decode_exec()` 里那套“抄手册宏”到底是什么？ |
| menuconfig 配置系统 | 上百个 `CONFIG_xxx` 宏是谁在维护？ |

> [!important] 重点
> 整讲的方法论只有一句：**读代码不等于“读”代码，先看踪迹（trace），再看源码（source）。**

## 1.2 make 与项目构建

### 1.2.1 make 的三要素与“时间戳” ★★★

`make` 的全部工作可以用一句话概括（摘自 `man make`）：

> make 根据 `makefile` 中描述的**依赖关系**，以及文件的**最后修改时间**，决定哪些文件需要重新生成，然后执行相应命令。

一条规则由三部分组成：

```makefile
目标: 依赖
<TAB>命令        # 行首必须是 TAB，不是空格
```

- **目标（target）**：要生成的文件（或伪目标）。
- **依赖（prerequisite）**：生成它所需要的文件。
- **命令（recipe）**：怎么生成。

判断逻辑：只要“依赖比目标新”，就重新执行命令；否则输出 `'a.out' is up to date.`。

> [!warning] 易错
> 命令行的缩进必须是 **TAB**。这是初学者第一个、也是最经典的报错（`missing separator`）。

> [!tip] 实操
> 想知道 make 到底做了什么，不要猜，开日志：

```bash
make -d            # make 的完整调试日志
make --debug=v     # 只输出“为什么重建”的决策信息
make -n            # 只打印命令，不执行（最常用）
make -nB           # -B 强制全部重建，配合 -n 可看到完整编译命令
strace -f -o strace.log make --debug=v   # 看 make 如何与操作系统交互
```

### 1.2.2 一个能看懂的最小 Makefile ★★★

课件用 YEMU v2.0 的 Makefile 作为样板，它几乎包含了后面 NEMU 用到的所有语法：

```makefile
BIN       = yemu
USER_BIN  = hello.bin
RV_CFLAGS = -march=rv32g -mabi=ilp32 -ffreestanding -nostdlib -static \
            -Wl,-Ttext=0x0 -O2

SRCS = $(shell ls *.c)     # 把 shell 命令的输出赋给变量
OBJS = $(SRCS:.c=.o)       # 后缀替换：把 .c 换成 .o
CC   = gcc

prog/hello.elf: prog/hello.c
	riscv64-linux-gnu-gcc $(RV_CFLAGS) -o $@ $^    # $@ 目标，$^ 全部依赖

$(USER_BIN): prog/hello.elf
	riscv64-linux-gnu-objcopy -j .text -O binary $^ $@

$(BIN): $(OBJS)            # 没写命令？用隐含规则
run-yemu: $(BIN) $(USER_BIN)   # 伪目标：不生成同名文件
	./$(BIN) $(USER_BIN)

.PHONY: run-yemu clean
clean:
	-@rm $(OBJS) $(BIN) prog/hello.elf $(USER_BIN) 2> /dev/null
```

四个必须掌握的语法点：

| 语法 | 作用 | 记忆点 |
| :--- | :--- | :--- |
| `$(VAR)`、`$(SRCS:.c=.o)` | 变量引用与后缀替换 | 变量 ≈ C 的宏，本质是字符串替换 |
| `$@`、`$^`、`$<` | 自动变量：目标、全部依赖、第一个依赖 | 只在规则内有意义 |
| `%.o: %.c` | 隐含规则 | 没写规则的 .o 会自动找同名 .c 编译 |
| `.PHONY` | 伪目标 | 不检查依赖、每次必执行（`clean`、`run` 必须声明） |

> [!important] 重点
> `-ffreestanding -nostdlib -static -Wl,-Ttext=0x0` 这一串不是凑数的：它告诉编译器“**没有操作系统、没有标准库、程序从地址 0 开始**”。这正是裸机程序的编译方式，第 4 讲会再遇到。

### 1.2.3 怎样读一个“看不懂”的 Makefile ★★★

NEMU 的构建系统比 YEMU 复杂得多：好几个 `.mk` 互相包含、大量变量和编译选项、`wildcard/patsubst/filter-out`、`ifeq ... endif` 条件编译、还有隐含规则。**直接从头读会崩**。

正确姿势是**先看踪迹，再回看源码**：

1. `make -nB | vim -` 导出全部编译命令；
2. 在 vim 里做文本处理，把噪音删掉（课件给出的四步）：

```vim
:%!grep "^\(gcc\|g++\)"                       " 只留编译命令
:%!sed -e "s+$NEMU_HOME+\$NEMU_HOME+g"        " 变量替换成符号
:%s+\$NEMU_HOME/build/obj-riscv32-nemu-interpreter+$OBJ_DIR+g
:%s/-O2.*=riscv32/$CFLAGS/g                   " 把编译选项折叠成 $CFLAGS
```

3. 于是你看到的就是“**NEMU 其实就是把一堆 .c 用这些 flag 编译、再链接**”；
4. 这时再回去看 `nemu/Makefile`，就会发现只剩下三类东西：

| 文件 | 内容 |
| :--- | :--- |
| `nemu/Makefile` | `SRCS`（要编译的源文件）、`CFLAGS`、`include $(NEMU_HOME)/scripts/native.mk` |
| `nemu/scripts/native.mk` | 与“运行/清除”相关的伪目标 |
| `nemu/scripts/build.mk` | 真正的编译规则；用 gcc 的 `-MMD` 生成 `.d` 依赖文件，再由 `fixdep` 处理 |

> [!important] 重点
> `-MMD` 让 gcc 在编译时顺手输出“这个 .o 依赖哪些头文件”。没有它，改一个头文件就不会触发重编译，你会调试到怀疑人生。

### 1.2.4 RTFM 的正确姿势 ★★

有同学反馈 `man make` 太长，“一打开就不想读”。课件的回答很值得记：

- **误区**：读手册 = 从头读完。x86 手册几千页，没人这么干。
- **真相**：RTFM 是**在手册中快速定位**，本质是信息检索能力。
- **技巧**：`man make` 有 30 个选项，**只读每个选项的第一句话**，5 秒一个，2.5 分钟就能筛完；再配合 `/` 搜索关键字，通常 1 分钟就能找到 `-n` 和 `--trace`。

## 1.3 先跑起来：从运行输出认识 NEMU

### 1.3.1 第一次运行 ★★

```bash
$ make run
[src/utils/log.c:28 init_log] Log is written to .../nemu-log.txt
[src/memory/paddr.c:56 init_mem] physical memory area [0x80000000, 0x87ffffff]
[src/monitor/monitor.c:51 load_img] No image is given. Use the default build-in image.
[src/monitor/monitor.c:28 welcome] Trace: ON
Welcome to riscv32-NEMU!
For help, type "help"
(nemu) c
[src/cpu/cpu-exec.c:120 cpu_exec] nemu: HIT GOOD TRAP at pc = 0x8000000c
[src/cpu/cpu-exec.c:88 statistic] host time spent = 95 us
[src/cpu/cpu-exec.c:89 statistic] total guest instructions = 4
[src/cpu/cpu-exec.c:90 statistic] simulation frequency = 42,105 inst/s
```

这段输出信息量极大，值得逐行读：

- `[src/utils/log.c:28 ...]`：**日志自带“文件:行号 函数名”**，这就是 `Log()` 宏比 `printf` 多做的事（见 1.4.3）。
- `physical memory area [0x80000000, 0x87ffffff]`：物理内存 128 MB，起始地址 `0x80000000`（RISC-V 习惯）。
- `No image is given`：没给镜像，用的是**内置的 4 条指令**。
- `HIT GOOD TRAP at pc = 0x8000000c`：执行到 `ebreak`，且是“好的”停机。
- `total guest instructions = 4`：**guest**（被模拟的程序）执行了 4 条指令；与之相对的是 host（NEMU 自己）。

“内置程序”只有 4 条指令，正好是 NEMU 的最小自检：

```asm
0x80000000: 80 00 02 b7   lui  t0, 524288      # t0 = 0x80000000
0x80000004: 00 02 a0 23   sw   zero, 0(t0)     # *(0x80000000) = 0
0x80000008: 00 02 a5 03   lw   a0, 0(t0)       # a0 = *(0x80000000)
0x8000000c: 00 10 00 73   ebreak               # 停机
```

> [!important] 重点
> 能跑通这 4 条指令，说明**取指、译码、执行、访存、PC 更新**这条链路是通的。后面所有指令都是往这个骨架上“加肉”。

### 1.3.2 读代码的三条入口 ★★

找 `main()` 在哪，课件给了四种方法，按实用性排序：

1. **GDB（最推荐）**：源码是静态的，运行是动态的，两者结合最好理解。
2. IDE / `vim + ctags` 的跳转。
3. `grep -nr "\bmain\b" nemu/src`。
4. 直接 `ls` 猜文件名（`nemu/src/nemu-main.c`）。

```bash
make menuconfig     # Build Options -> Enable debug information
make clean && make gdb
(gdb) b main
(gdb) layout src    # TUI：左边源码、右边命令行
```

常用 GDB 命令（建议抄一张便利贴）：

| 命令 | 作用 |
| :--- | :--- |
| `r` | 重新运行 |
| `s` / `n` | 单步进入 / 单步跳过（跳过库函数） |
| `finish` | 执行到当前函数返回 |
| `p expr` | 打印变量/寄存器 |
| `x /16xb addr` | 扫描内存 |
| `bt` | 查看调用栈 |
| `b` / `watch` | 断点 / 监视点 |
| `help xxx` | 查看命令帮助 |

## 1.4 NEMU 的初始化：`init_monitor()` 里发生了什么

### 1.4.1 总览 ★★★

`main()` 的逻辑非常短：解析参数 → `init_monitor()` → `engine_start()`。其中 `init_monitor()` 是重头戏，顺序大致如下：

| 函数 | 做什么 | 值得注意的点 |
| :--- | :--- | :--- |
| `parse_args()` | 解析命令行 | 用 `getopt_long()`；`--log` 就是在这里被识别的 |
| `init_rand()` | 初始化随机数 | 后面 `init_mem` 要用 |
| `init_log()` | 打开日志文件 | `Assert()` / `Log()` 宏 |
| `init_mem()` | 初始化内存 M | **把内存填成随机数** |
| `init_isa()` | 设置初始状态 | 拷贝内置程序、设 PC、`gpr[0]=0` |
| `load_img()` | 加载镜像 | 命令行给了镜像才执行 |
| `init_difftest()` | DiffTest | 本阶段未开启 |
| `init_sdb()` | 简化版 GDB | PA1 要你实现 |
| `init_disasm()` | 反汇编库 | 使用 LLVM 的能力 |
| `welcome()` | 欢迎信息 | 顺带打印 trace 状态、编译时间 |

> [!important] 重点
> 把这张表背下来，你就有了 NEMU 的“启动地图”：任何“为什么程序还没开始跑就崩了”的问题，答案都在这 10 个函数里。

### 1.4.2 为什么要把内存填成随机数 ★★★

```c
// init_mem()：把物理内存初始化为随机值
```

这不是恶作剧，而是**故意的**：

- 如果内存初始为 0，那么“访问未初始化内存”的 bug 会安静地得到 0，程序看起来正常；
- 填成随机数后，同样的 bug 会立刻表现为“结果莫名其妙”，**把 UB 暴露出来**。

> [!important] 重点
> 这条设计思想会反复出现：**让错误尽早、尽量响亮地暴露**。后面 P9 讲的 sanitizer 是同一思想在编译器的体现。

### 1.4.3 `Log()` 宏：比 printf 多的三件事 ★★

```c
#define Log(...) ...
```

| 增强 | 实现手段 | 有什么用 |
| :--- | :--- | :--- |
| 输出位置 | `__FILE__` / `__LINE__` / `__func__` | 一眼定位是哪一行打的日志 |
| 输出颜色 | ANSI escape code（`man console_codes`） | 错误红色、警告黄色，快速扫读 |
| 自动换行/刷新 | 末尾 `\n` + 行缓冲 | 不换行时输出可能滞留缓冲区 |

`__func__` 是个**预定义变量**（不是宏），这是 C99 的细节，容易被忽略。

> [!tip] 实操
> 验证缓冲区问题：

```c
#include <stdio.h>
int main() { printf("Hello, RISC-V"); while (1); return 0; }
```

不换行、不刷新，你可能什么都看不到。`man setbuf` 能找到解释。

### 1.4.4 `init_isa()`：状态机的初始状态 ★★★

```c
memcpy(guest_to_host(RESET_VECTOR), img, sizeof(img));  // 内置程序放进 M
cpu.pc = RESET_VECTOR;                                  // 0x80000000
cpu.gpr[0] = 0;                                         // x0 恒为 0
```

对应状态机 S = <R, M> 的两半：

- M：把程序从“数组”拷贝到“物理内存”；
- R：PC 指向入口；`gpr[0]` 强制为 0（RISC-V 规定 `zero` 不可写）。

> [!warning] 易错
> `RESET_VECTOR` 默认 `0x80000000`，可以在 menuconfig 里改。如果你在 PA 里把镜像加载到别处却忘了改这里，现象就是“PC 指向一片随机数”。

## 1.5 指令执行：从 `cpu_exec()` 到 `decode_exec()`

### 1.5.1 调用链 ★★★

```
sdb_mainloop()            // 读命令，输入 c 后……
  └─ cpu_exec()           // 执行 n 条指令，处理 trap
       └─ execute()
            └─ exec_once()          // 取指、译码、执行、更新 PC —— 一个指令周期
                 └─ isa_exec_once() // ISA 相关部分
                      ├─ inst_fetch(&s->snpc, 4)   // 取 4 字节
                      └─ decode_exec(s)            // 译码 + 执行
```

核心只有两行：

```c
int isa_exec_once(Decode *s) {
  s->isa.inst.val = inst_fetch(&s->snpc, 4);  // 取指，同时算出 snpc = pc + 4
  return decode_exec(s);                      // 译码并执行
}
```

- `snpc`（static next PC）：**顺序执行时**的下一条指令地址，即 `pc + 4`；
- 真正的下一条 PC 还要看指令是否跳转（`dnpc`），这部分在 `decode_exec` 里由具体指令改写。

> [!important] 重点
> 这和 YEMU 的 `inst_cycle()` 是同一个循环：**取指 → 译码 → 执行 → 更新 PC**。NEMU 只是把它拆成了更清晰的分层。

### 1.5.2 “抄手册宏”：`INSTPAT` ★★★

NEMU 实现一条指令长这样：

```c
INSTPAT("??????? ????? ????? ??? ????? 01101 11", lui, U, R(rd) = src1);
```

- 第一个参数是**指令的二进制模式**：`?` 表示“这一位不关心”，其余是定值。把它和 RISC-V 手册里的指令格式对照，会发现**几乎一模一样**；
- 第二个参数是指令名（用来生成函数名/字符串）；
- 第三个参数是操作数类型（`U`/`I`/`S`/`B`/`J`/`R`），决定立即数怎么解析；
- 第四个参数是**执行体**，`R(rd) = src1` 就是“把源操作数写进 rd”。

课件的评价非常到位：

- **不言自明**：虽然不知道背后的“黑科技”，但也能猜到它在干什么；
- **不言自证**：无脑抄手册，只要没抄错，基本就是对的。

> [!note] 考点
> 为什么说“越接近手册，不言自明和不言自证就做得越好”？因为**代码与规格同构**时，审查代码就等于审查实现是否符合手册，几乎不需要额外的“翻译”步骤。

### 1.5.3 宏到底展开成了什么 ★★

`INSTPAT` 背后是一套宏：把模式串在**编译期**解析成 `key`（定值位）、`mask`（哪些位关心）、`shift`，再生成一连串 `if` 判断。

```c
// 简化后的逻辑
static int decode_exec(Decode *s) {
  ...
  INSTPAT_START();
  INSTPAT("...01101 11", lui, U, R(rd) = src1);
  ...
  INSTPAT_END();
}
```

三个实现细节值得注意：

1. **`pattern_decode()` 用循环展开而不是 `for` 循环**：输入全是常量，开优化后整个计算被编译器“算掉”，运行时零开销；
2. 调用时传 `STRLEN(pattern)` 而不是 `strlen(pattern)`：长度在编译期就知道，不需要运行时数一遍；
3. **`STRLEN` 依赖“模式串是字符串字面量”**，这是一个隐含约束。

怎么看展开结果？两种办法：

- 土办法：**拿笔在纸上人肉展开**。人脑短期记忆只有 7±2 个单元，盯着看必溢出；写下来 3 分钟就懂。
- 现代办法：让编译器展开。在 `build.mk` 的编译规则里加一行，然后格式化：

```makefile
@$(CC) $(CFLAGS) -E -MF /dev/null $< | clang-format > $@.i
```

> [!important] 重点
> “瞪眼半小时，不如动笔 3 分钟”——这句话对后面读 AM 的链接脚本同样有效。

### 1.5.4 `nemu/include/macro.h` 里的四个小工具 ★★

| 宏 | 作用 | 类比 |
| :--- | :--- | :--- |
| `IFDEF` / `IFNDEF` | 在表达式里做条件选择，替代 `#ifdef` | 让代码更紧凑 |
| `MAP(c, f)` | 对参数列表逐个应用 `f` | X-macro |
| `BITS(x, hi, lo)` | 取位段 | Verilog 的 `x[hi:lo]` |
| `SEXT(x, len)` | 符号扩展 | 借助 `struct` 位域实现 |

> [!warning] 易错
> `SEXT` 用位域实现，依赖编译器对位域的具体布局（实现定义行为）。它在“同一编译器、同一 ABI”下没问题，但**不是可移植代码**——这正是“用编译器的便利换可移植性”的典型取舍。

### 1.5.5 指令实现方案的十年演进 ★

| 年份 | 方案 | 不言自明 | 不言自证 |
| :---: | :--- | :---: | :---: |
| 2014 | Copy-Paste（反面教材） | ✗ | ✗ |
| 2015–2016 | 函数定义宏 + include | ✗ | ✗ |
| 2017–2019 | 填表宏 + IR | ✓ | ✓ |
| 2020 | 填 switch 宏 + IR | ✓ | ✓ |
| 2021 | 抄手册宏 + IR | ✓✓ | ✓✓ |
| 2022–2023 | 抄手册宏 | ✓✓✓ | ✓✓✓ |

**准则：越接近手册，不言自明和不言自证就做得越好。**

> [!note] 考点
> “为什么非要用这么复杂的宏？”——本质是**编码开销是一次性的，维护开销是长期的**。Copy-Paste 最好写，但改一条指令要改几十处；对有一定规模的项目，**可维护 > 好懂**，好懂可以靠注释补。

## 1.6 menuconfig 与 kconfig 配置系统

### 1.6.1 要解决的问题 ★★★

宏一多，问题就来了：

- 宏之间有依赖：定义了 `A`，`B` 必须是某值，`C` 不能取某范围；
- 靠人维护必出错：**忘了改关联的宏 = bug = 一下午的调试**。

于是把这件事交给工具：

1. 开发者用 **kconfig 语言**写配置描述文件（`nemu/Kconfig`）：选项的类型、默认值、依赖关系、层次关系；
2. 开发者只需“提供一部分满足依赖的选项”，其余取默认值；
3. kconfig 编译器根据描述文件 + 选择结果，**生成宏定义代码**。

### 1.6.2 三个产物 ★★★

| 产物 | 给谁用 | 内容 |
| :--- | :--- | :--- |
| `nemu/include/generated/autoconf.h` | C 代码 | 一堆 `CONFIG_xxx` 宏 |
| `nemu/include/config/auto.conf` | Makefile | 变量定义 |
| `nemu/include/config/auto.conf.cmd` | Makefile | 与配置描述文件相关的依赖规则 |

另外还有一个用**时间戳**维护的目录树 `nemu/include/config/`，配合 `fixdep` 使用。

> [!important] 重点
> `autoconf.h` 被 `common.h` 无条件包含，于是**每个源文件都间接依赖它**。这就引出了下一个问题。

### 1.6.3 `fixdep`：改一个宏，别重编整个世界 ★★★

问题：只要在 menuconfig 里改任意一个选项，`autoconf.h` 的时间戳就变了，于是**所有文件全部重编译**。3500 行还好，QEMU 那种规模就是灾难。

`fixdep` 的思路很聪明：

- 分析每个 `.c` 文件里**实际用到了哪些 `CONFIG_xxx`**；
- 为每个 `CONFIG_xxx` 建一个**空文件**，让 `.o` 依赖这些空文件，而不是依赖 `autoconf.h`；
- 改配置时，kconfig 只更新**受影响的空文件**的时间戳。

于是只有真正用到该宏的文件才重编译。

### 1.6.4 例：Makefile 里的 filelist ★★

需求：某些配置项决定某些源文件是否参与编译。NEMU 的做法是用**集合运算**：

| 变量 | 含义 |
| :--- | :--- |
| `SRCS-y` | 参与编译的源文件候选集 |
| `SRCS-BLACKLIST-y` | 不参与编译的黑名单 |
| `DIRS-y` | 参与编译的目录（目录下所有文件进 `SRCS-y`） |
| `DIRS-BLACKLIST-y` | 不参与编译的目录 |

**实际编译集合 = 候选集 − 黑名单**。例如：

```makefile
# nemu/src/filelist.mk
DIRS-BLACKLIST-$(CONFIG_TARGET_AM) += src/monitor/sdb
```

含义：当目标是 AM（NEMU 作为 AM 的后端）时，SDB 调试器不需要，从编译集合里去掉。

> [!note] 考点
> 这种“`-y` 后缀 + 黑名单”的写法不是 NEMU 独创，Linux 内核的 Kbuild 就是这么干的。看懂它，你就看懂了内核 Makefile 的一半。

## 1.7 本讲方法论：读代码 ≠ “读”代码

### 1.7.1 source 与 trace ★★★

| 方式 | 优点 | 缺点 |
| :--- | :--- | :--- |
| 看源码（source） | 能看到每一处静态细节 | 繁琐，容易迷路 |
| 看踪迹（trace） | 快速了解运行动态行为 | 不全面 |

**正确顺序：先从容易的方式下手（trace），再结合动态行为理解静态代码。**

本讲给出的三层 trace，正好对应三种粒度：

| 层次 | 工具 | 看什么 |
| :--- | :--- | :--- |
| 系统调用 | `strace` | 程序/工具如何与操作系统交互 |
| 工具决策 | `make -d` / `make --debug=v` | make 为什么重建这个目标 |
| 面向用户 | `make -n` / `make --trace` | 实际执行了哪些命令 |

同一套方法用在 NEMU 上就是：**先 `make run` 看输出 → 再用 GDB 断在关键函数 → 最后读源码**。

### 1.7.2 本讲小结 ★★★

- NEMU = 加强版 YEMU：同一个 ISA 状态机，只是指令更全、工程更复杂。
- 构建：`makefile` 描述依赖 + 时间戳决定重建；读大项目先看 `make -nB`，再看 `.mk`。
- 初始化：`init_monitor()` 的十个步骤，把 R 和 M 摆到“可以开始执行”的状态。
- 执行：`cpu_exec → execute → exec_once → isa_exec_once → decode_exec`，本质仍是取指-译码-执行。
- 译码：`INSTPAT` 让代码与手册同构；宏展开可以人肉推，也可以让编译器 `-E` 出来。
- 配置：kconfig 管依赖，menuconfig 管选择，`fixdep` 管增量编译。
- 方法：**先 trace 后 source；善用工具；RTFM 是检索不是通读。**

## 1.8 自测（答案见附录 B）

1. 用一句话解释：为什么 `make` 第二次执行会输出 `up to date`？
2. `$@` 和 `$^` 分别是什么？在 `$(OBJ_DIR)/%.o: %.c` 这条规则里，它们展开成什么？
3. NEMU 的 `init_mem()` 为什么把内存填随机数而不是清零？
4. `INSTPAT` 的第一个参数里，`?` 代表什么？为什么这种写法能“不言自证”？
5. 如果 `autoconf.h` 被所有文件包含，为什么改一个配置项不会导致全量重编译？
6. `SRCS-y` 和 `SRCS-BLACKLIST-y` 的关系是什么？举一个 NEMU 中的实际例子。

---

# 附录 B 自测题答案要点

## B.1 第 1 讲

1. **why up to date**：`a.out` 的修改时间比 `a.c` 新，make 判定无需重建。
2. **`$@` / `$^`**：目标、全部依赖；在 `$(OBJ_DIR)/%.o: %.c` 中分别是 `build/xxx.o` 和 `xxx.c`。
3. **随机填充**：让“访问未初始化内存”这种 UB 立刻暴露，而不是安静地读到 0。
4. **`?`**：该位不关心（don't care）；模式串与手册同构，所以“抄”就是“实现”，审查代码等于审查规格。
5. **不会全量重编**：`fixdep` 把对 `autoconf.h` 的依赖拆成对若干空文件的依赖，只更新时间戳变化的那些。
6. **集合关系**：实际编译集合 = `SRCS-y` − `SRCS-BLACKLIST-y`；例：`CONFIG_TARGET_AM` 时排除 `src/monitor/sdb`。

---

# 附录 C 术语表

| 术语 | 含义 |
| :--- | :--- |
| ISA | 指令集体系结构，软件与硬件的接口 |
| ABI | 应用二进制接口，规定调用约定、数据布局等 |
| AM | Abstract Machine，按功能模块化的裸机运行时环境 |
| TRM / IOE / CTE / VME / MPE | AM 的五个模块 |
| NEMU | NJU EMUlator，全系统模拟器 |
| YEMU | 教学用的极简模拟器（NEMU 的前身） |
| DiffTest | 差分测试：用参考实现对照被测实现 |
| UB | 未定义行为，编译器可假设其不发生 |
| prologue / epilogue | 函数准备/结束阶段 |
| caller-saved / callee-saved | 调用者保存 / 被调用者保存 |
| padding | 为满足对齐要求插入的空洞 |
| klib | AM 提供的机器无关 C 库 |
| kconfig / menuconfig | 配置描述语言 / 其菜单式前端 |
| fixdep | 根据宏使用情况拆分头文件依赖的工具 |

---

# 附录 D 参考资料

- 课程主页：`https://ysyx.oscc.cc/docs/`
- 本讲义对应的四份课件：
  - P7 `https://ysyx.oscc.cc/slides/2306/07.html`
  - P9 `https://ysyx.oscc.cc/slides/2306/09.html`
  - P10 `https://ysyx.oscc.cc/slides/2306/10.html`
  - P11 `https://ysyx.oscc.cc/slides/2306/11.html`
- RISC-V 调用约定：`https://github.com/riscv-non-isa/riscv-elf-psabi-doc`
- RISC-V 汇编手册：`https://github.com/riscv-non-isa/riscv-asm-manual`
- GNU Make 手册：`https://www.gnu.org/software/make/manual/make.pdf`
- GDB TUI：`https://sourceware.org/gdb/current/onlinedocs/gdb/TUI.html`

> 视频地址（B 站，合集“第六期‘一生一芯’”）：P7 `BV1up4y1j7Ji`、P9 `BV1ow411275B`、P10 `BV19H4y1d7Yi`、P11 `BV1Vu4y1s73Y`。
