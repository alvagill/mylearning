# E7 SoC 接入

## 一、总线与存储层次

### SimpleBus 协议
- **两套握手**：`reqValid/respValid` 用于数据传输确认；`THRE` 轮询用于外设内部状态确认——两者是不同层次的握手（物理层 vs 设备层）。
- **读写的不对称性**：写是"单向推送"（数据推出即落盘），读是"双向往返"（发地址 → 等响应）。
  - 读需要等一拍甚至多拍。
  - 写只需要一拍（不需要 `rdata`）。
- **从机必须"自持"**：从机一旦收到请求，必须独立倒计时到 N 拍拉高 `respValid`，**不能依赖主机继续拉 `reqValid`**。
  - 错误写法：`respValid <= reqValid`——只在 1 拍延迟下恰好工作，一旦延迟 >1 拍或主机进入等待拍就死锁。
  - 正确写法：`pending + 倒计时 + respValid 保持到被消费`。
- **背靠背处理**：接收新请求的条件是 `reqValid && (!pending || respValid)`。
  - `!pending`：从机空闲。
  - `respValid`：事务已完成（响应待消费）。
  - 用 `||` 不是 `&&`——"事务已完成"由这两个充分条件之一表达。

### 地址空间与设备
- **地址译码器**：SoC 用地址高位判断"访问哪个设备"（如 `0x1000_0000` → UART，`0x8000_0000` → PSRAM），然后把请求路由到该设备。
- **寄存器偏移 vs 字节索引**：
  - 内存（PSRAM）：地址低位是**字节索引**——相邻地址是相邻字节。
  - 外设（UART/GPIO）：地址低位是**寄存器选择码**——相邻地址是不同寄存器。
- **UART16550 的 DLAB**：LCR bit 7 控制偏移 0/1 的含义切换。
  - DLAB=0：偏移 0 = THR，偏移 1 = IER。
  - DLAB=1：偏移 0 = DLL，偏移 1 = DLM。
- **MMIO 字节通道**：CPU 按字节读外设时，硬件读整个 32 位字后按 `addr[1:0]` 选字节。因此 **MMIO 返回值必须按 `addr[1:0]` 把值放到正确的字节通道**，否则 CPU 选到的就是 0。
  ```c
  uint32_t word = val << ((addr & 3) * 8);
  ```
- **LSU 地址必须透传字节地址**：不能字对齐，否则访问 LSR(0x...05) 会变成访问 MCR(0x...04)。

## 二、CPU 多周期设计

### 状态机扩展规律
- 状态数 = 需要等待的访存次数 + 1。
- 只有取指：2 状态（`S_IF1`、`S_IF2`）。
- 取指 + load：3 状态（`S_IF1`、`S_IF2`、`S_MEM`）。
- 每多一次需要等待的访存，就多一个状态。
- `store` 不需要进 `S_MEM`（写不需要等响应）；`load` 必须进 `S_MEM`。

### 关键信号语义
- **`valid`（或 `ifu_valid`）**：标记"当前这一拍是否在执行真指令"——用来屏蔽 `S_IDLE` 拍译码出的垃圾信号。
- **`in_mem`**：标记"当前处于 `S_MEM` 状态"。
- **`wb_valid`**：寄存器堆写使能门控。
  ```verilog
  wire wb_valid = (ifu_valid && !mem_to_reg) || (in_mem && mem_to_reg);
  ```
  - 非 load：`S_WAIT` 拍写回。
  - load：`S_MEM` 拍写回。
- 写回数据源也要切换：load 在 `S_MEM` 拍必须用 `lsu_rdata_ext`，不是 `alu_result`。

### debug_inst_retire（指令退休脉冲）
- **含义**：这一拍有一条指令真正退休。
- **必须是脉冲**——每个退休事件只拉高 1 拍，不能是持续电平。
- **打一拍对齐**：需要在下一拍暴露给 C++ 侧，与 PC 更新同步。
  ```verilog
  if (if_execute) debug_inst_retire_r <= !mem_op && !halt;
  else            debug_inst_retire_r <= mem_done;
  ```
- **用途**：IPC 统计。每拍检查一次，为 1 则 `g_insts++`。

### Difftest 的硬约束
- **sCPU 改，sEMU 必改**——两边行为必须一致，否则 difftest 报 mismatch。
- sEMU 的 MMIO 读必须"抄 sCPU 观测到的世界"——通过快照钩子，不能让 sEMU 自己产生值。
- sEMU 相位差补偿：RTL 时序逻辑与行为模型有固定相位差（如 `mcycle` 差 2 拍），用 `- 2` 之类补偿。这不是 bug，是模型对齐。

## 三、CSR 与 mcycle

### CSR 基础
- CSR 是 CPU 内部寄存器——用专用指令（`csrrs`/`csrrw`/`csrrc`）访问，普通 load/store 不能访问。
- 12 位地址空间（4096 个 CSR）——编码在 I 型指令的 `instr[31:20]`。
- `csrrs` 是"读 + 置位"——原子读改写。
- `csrr rd, csr` 是伪指令——汇编器展开成 `csrrs rd, csr, x0`。
- `rs1 = x0` 时不修改 CSR——所以 `csrr` 等价于只读。
- 只读 CSR 可以只实现"读"路径——不需要状态存储，`case` 硬编码返回值即可。

### 学号 CSR
- `mvendorid`（0xF11）— 存厂商 ASCII 码（"ysyx" = 0x79737978）。
- `marchid`（0xF12）— 存学号。

### mcycle / mcycleh
- 硬件是一个 64 位自增计数器——每个时钟沿 +1，不是两个独立 32 位寄存器。
- 两个 CSR 地址（0xB00 / 0xB80）是"读窗口"——分别读低 32 位和高 32 位。
- **"高低高"读法**：防读两次之间发生进位。
  ```c
  hi0 = csrr mcycleh;
  lo  = csrr mcycle;
  hi1 = csrr mcycleh;
  if (hi0 != hi1) lo = csrr mcycle;  // 进位了，重读 lo
  return ((uint64_t)hi1 << 32) | lo;
  ```

### 内联汇编
- `asm volatile("csrr %0, mcycle" : "=r"(v))` 展开链：
  1. C 编译器分配寄存器给 `%0`，替换成 `csrr a0, mcycle`。
  2. 汇编器展开伪指令 → `csrrs a0, 0xB00, x0`。
  3. 编码成机器码 `0xB0002A73`。
  4. CPU 解码执行。
- **`volatile` 不能省**——否则编译器可能优化掉或合并两次读。
- 约束符号：`=` 只写、`+` 读写、`r` 通用寄存器、`m` 内存、`i` 立即数。

## 四、GPIO 与外设

### APB 协议
- 两拍协议：`setup`（`PSEL=1, PENABLE=0`）+ `access`（`PSEL=1, PENABLE=1`）。
- 零等待模式：`pready = PSEL & PENABLE`——access 相位立即完成。
- 信号：`PSEL` / `PENABLE` / `PWRITE` / `PADDR` / `PWDATA` / `PRDATA` / `PREADY`。
- `pstrb` 字节选通：控制 `PWDATA` 哪些字节真的写入——和 CPU 的 `sb`/`sh`/`sw` 语义对应。

### GPIO 控制器寄存器
| 偏移 | 属性 | 含义 |
|------|------|------|
| 0x0  | RW   | 16 位 LED |
| 0x4  | RO   | 16 位拨码开关 |
| 0x8  | RW   | 32 位数码管（每 4 位一个十六进制数） |
| 0xc  | —    | 保留 |

### 七段数码管译码
- NVBoard 位序：`bit7..0 = A B C D E F G P`，高有效。
- 段码表：`0 = 0xFC`，`1 = 0x60` 等。
- `seg_0` 是最右侧数码管——对应 `seg_reg[3:0]`。

## 五、UART 通信

- **8N1 格式**：8 数据位、无校验、1 停止位。LCR 的 `0x03` 就是 8N1。
- **除数公式**：`D = SoC时钟 / (波特率 × 16)`——16 是 16550 内部固定的过采样倍数。
  - `25MHz / (115200 × 16) = 13.56` → 取整 13。
  - 实际波特率略高，误差容忍 ±5%。
- **16 倍过采样**：接收端用 16 倍波特率时钟采样，每位采 16 次，取中间 3 次做多数判决——抗时钟漂移和噪声。
- **除数拆两个寄存器**：`DLL`（低 8 位）+ `DLM`（高 8 位）——因为 UART16550 接口只有 8 位。
- **UART 初始化顺序**：
  1. 写 LCR 设 DLAB=1。
  2. 写 DLL / DLM（除数）。
  3. 写 LCR 设 8N1 且 DLAB=0。
- **`putch` 必须先轮询 LSR 的 THRE（bit 5）**——发送队列满时不能写，否则丢字符。

## 六、NVBoard 集成

- `nvboard_bind_all_pins(top)`：绑定 Verilog 信号到 NVBoard 元件（由 `auto_pin_bind.py` 从 `.nxdc` 生成）。
- `nvboard_init()`：打开窗口。
- `nvboard_update()`：刷新一次——每 clock 周期调用一次（多时钟域时挂在慢钟上）。
- `nvboard_quit()`：关闭窗口。
- `setenv("NVBOARD_HOME", ...)`：NVBoard 加载字体需要此环境变量——Makefile 里的 `export` 对手动运行无效，须在 C++ 里设。
- 约束文件格式：`top=<模块名>` + 每行 `Verilog信号名 (NVBoard引脚名)`。
- **TX/RX 交叉连接**：SoC 的 `uart_tx` 接 NVBoard 的 `UART_RX`。

## 七、性能评测与仿真环境

### 时钟比例（CLK_RATIO）
- 真实 SoC：CPU 由 PLL 倍频（如 400MHz），外设慢（25MHz）。
- 仿真：`cpuClock` 每步翻转，`clock` 每 `CLK_RATIO` 步翻转一次。
- **k 的上限**：`k × 25MHz ≤ NPC 综合频率`。
- 用 yosys + ABC 报的 `fmax` 算 k。
  - 2452ps 路径 → 408MHz → k=16。
- **`nvboard_update()` 必须只在 clock 边沿调用**——否则 UART 波特率会被算错。

### IPC 计算
- `IPC = 指令数 / cpuClock 周期数`。
- 有 `debug_inst_retire` 时逐拍累计指令数。
- SoC 模式下顶层是 `SimTop`，用 Verilator 层次化访问读出内部信号：
  ```cpp
  top->rootp->SimTop__DOT__asic__DOT__soc__DOT__cpu__DOT__core0__DOT__debug_inst_retire
  ```
- 路径用 `__DOT__` 转义 `.`。
- 改层次结构时路径要重新 `grep obj_dir/` 找。

### 性能瓶颈（Amdahl 定律）
- 提频几乎不缩短程序时间——瓶颈在 25MHz 外设总线。
- 无 cache 时：97% 的拍在等外设。
- Amdahl 定律：优化只占 3% 的计算部分——整体提升约 3%。
- 真正的优化方向：**加 cache > 加流水线 > 提频**。
- `timer.c` 系数是"凑数"——仿真里没有真实频率概念，需要调系数让输出间隔"接近 1 秒"。
  - 真实芯片：系数 = CPU 频率 / 1e6。
  - 仿真里：选一个让"仿真 1 秒"≈"墙钟 1 秒"的值。

### 仿真效率取舍
- `minirv-npc`：快（DPI-C 瞬时返回），用于功能调试。
- `minirv-ysyxsoc`：慢（多级总线 + SPI/PSRAM 模型），用于性能评测。
- 两者互补——不是替代。
- 用 Makefile 变量 / 编译宏在两种环境间切换。

## 八、主要注意事项

### RTL 设计
1. 从机的 `respValid` 必须自持——不能从 `reqValid` 派生，否则延迟 >1 拍死锁。
2. MMIO 返回值的字节通道要正确——按 `addr[1:0]` 左移，否则 CPU 按字节读时读到 0。
3. LSU 地址透传——不做字对齐，否则会选错外设寄存器。
4. 状态机状态数 = 需要等待的访存次数 + 1。
5. `debug_inst_retire` 必须是脉冲——不是电平。
6. 写回数据源要按指令类型切换——load 用 `rdata_q`，其他用 `alu_result`。

### Difftest
1. sCPU 改，sEMU 必改。
2. sEMU 的 MMIO 读要用快照钩子——不能自己产生值。
3. 相位差用常数补偿——不是 bug，是模型对齐。
4. MMIO 快照在时钟沿前采样——防止下一条指令的组合读覆盖。

### 构建系统
1. Makefile 的 recipe 行必须用真 Tab——网页复制会变成空格。
2. Makefile 里 `~` 不展开——用 `$(HOME)` 或绝对路径。
3. Verilator 的 `-D` 和 `+define+` 等价——但要和源码里 `ifdef` 的宏名精确一致。
4. Verilator 不允许跨模块重复导入同名 DPI 函数——要用重命名导入：
   ```verilog
   import "DPI-C" pmem_read = function int unsigned lsu_pmem_read(...);
   ```
5. `parameter integer` 赋给窄位宽变量需要显式截断——如 `LATENCY[3:0] - 4'd1`。
6. `-static` 不能和 SDL2 一起用（NVBoard 依赖动态链接的 SDL2）。

### UART / 通信
1. 不检查 THRE 就写 THR 会丢字符。
2. 地址 `0x10000005` 是 LSR——不是 `0x10000004`（MCR）。
3. 两套除数公式：
   - UART16550：除数 = 频率 / (波特率 × 16)。
   - NVBoard：`divisor = 16 × UART16550除数`（因为 NVBoard 直接用 bit 周期采样）。
4. 除数变化时两边都要改——16550 内部和 NVBoard 的 divisor。

### CSR
1. `csrr` 是伪指令——展开成 `csrrs rd, csr, x0`。
2. 只读 CSR 不需要状态存储——`case` 硬编码返回值。
3. 64 位 CSR 要用"高低高"读法——防进位。
4. 内联汇编的 `volatile` 不能省——否则被优化掉。

### 性能
1. 提频几乎无收益（访存瓶颈）——除非先加 cache。
2. k 的上限是综合频率 / 25MHz——用 yosys 或 ecc 报告算。
3. 跑分用 cpuClock 周期数——不是 clock 周期数。
4. `CLK_RATIO=1` 用于日常迭代——`CLK_RATIO=k_max` 用于正式跑分。

### 其他
1. AM 的 mainargs 是"占位符 + insert-arg 原地替换"——手动跑仿真器要先 `insert-arg`，否则参数是占位符字符串。
2. `rv32e` 只有 16 个寄存器——`trap.S` 用标准宏 `__riscv_32e` 判断。
3. `memcpy` 不处理重叠——重叠区域要用 `memmove`。
4. `strcmp` 只要求符号关系——不必精确返回 -1/0/1。
