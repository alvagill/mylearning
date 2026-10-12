Verilog 硬件描述语言 核心学习笔记

*Verilog语法原理与数字电路设计实战总结，依托实操案例梳理核心知识点、时序坑点、设计心得，重点记录波形时序、锁存器、状态机设计核心学习要点*

# Verilog 数字电路设计 核心学习笔记

## 一、Verilog 基础语法核心规范

### 1. 赋值运算符区分（硬件本质规则）

#### **阻塞赋值 `=`**

- 使用场景：组合逻辑 `always @(*)`
- 行为：顺序执行，立即更新变量值

#### **非阻塞赋值 `<=`**

- 使用场景：时序逻辑 `always @(posedge clk)`
- 行为：同一时钟周期内并行更新，时钟沿结束统一生效，匹配寄存器硬件行为

✅ **黄金铁律**：

组合逻辑用 `=`，时序逻辑用 `<=`；**禁止同一信号混用两种赋值方式**

### 2. wire / reg 信号规范

- **wire**：物理连线，只能由 `assign` 连续赋值、模块输出驱动；不能在 `always` 块内赋值
- **reg**：寄存器变量，**仅允许在 always 块内赋值**；综合结果可能是触发器（时序逻辑）或连线（组合逻辑）

### 3. 常量定义：parameter VS localparam

- **parameter**：模块参数，支持外部实例化时重写
- **localparam**：局部常量，外部无法修改

👉 **状态机编码优先使用 localparam**，封装性更好

### 4. 向量可变索引（位切片语法）

❌ 非法写法：`data[start : start+3]` 左右边界均为变量，Verilog语法不支持

✅ 标准可变切片：

```verilog
data[base +: WIDTH]  // 从base位向高位，取WIDTH位
data[base -: WIDTH]  // 从base位向低位，取WIDTH位
```

## 二、组合逻辑 & 锁存器（Latch）避坑

### 1. 锁存器产生根源

`always @(*)` 组合逻辑内，`case / if-else` 存在未覆盖分支；部分条件通路信号没有赋值，综合器生成锁存器。

### 2. 零锁存标准写法（工程通用防御手段）

```verilog
always @(*) begin
    // 第一步：给所有输出预先设置默认值
    next_state = A;
    out = 1'b0;
    flag = 1'b0;
    // 后续case分支按需覆盖默认值
    case(state)
        // 状态分支
    endcase
end
```

### 3. Moore / Mealy 状态机输出时序差异

- **Moore输出**：仅由当前状态 `state` 决定
- **Mealy输出**：由当前状态 `state` + 当前输入共同决定

⚠️ **时序波形学习核心要点**：

输出使用 `assign` 组合逻辑直接绑定state，不要放到时序 `always @(posedge clk)` 内部；

时序块内寄存输出会**天然滞后1个时钟周期**，直接导致波形错位。

## 三、时序逻辑 & 复位设计规范

### 1. 同步复位 VS 异步复位

#### 同步复位（仅时钟沿生效，常规设计最常用）

```verilog
always @(posedge clk) begin
    if(reset)
        state <= A;
    else
        state <= next_state;
end
```

#### 异步复位（复位信号立刻生效，无需等待时钟）

```verilog
always @(posedge clk, posedge areset) begin
    if(areset)
        state <= A;
    else
        state <= next_state;
end
```

### 2. 低电平有效复位（命名规范后缀 _n）

```verilog
always @(posedge clk, negedge resetn) begin
    if(!resetn) // 低电平触发复位，必须取反
        state <= A;
    else
        state <= next_state;
end
```

### 3. 复位输出时序陷阱

❌ 错误思路：在时序块 `if(reset)` 手动把 `z <= 0`

会生成触发器，复位后z需要等待下一个时钟沿才拉低。

✅ 正确方案：

```
assign z = (state == E) || (state == F);
```

组合逻辑持续驱动；复位强制state跳转到初始状态，z会瞬间跟随变0，无需额外清零。

## 四、核心模块设计：有限状态机FSM（学习重点）

### 标准三段式/两段式模板（工程学习、设计实操最常用）

```verilog
// 1. 组合逻辑：次态逻辑
always @(*) begin
    next_state = A; // 默认值防锁存
    case(state)
        A: next_state = w ? B : A;
        B: next_state = w ? C : D;
        // ...其余状态
        default: next_state = A;
    endcase
end

// 2. 时序逻辑：状态寄存器
always @(posedge clk) begin
    if(reset)
        state <= A;
    else
        state <= next_state;
end

// 3. Moore输出（组合逻辑）
assign z = (state == E) || (state == F);
```

### 各类核心模块学习要点与设计心得

#### 1. 独热编码（One-hot）设计

核心原理：电路存在冗余非法状态，常规case匹配无法规避异常情况

设计思路：拆解为独立布尔逻辑方程，逐位赋值实现次态跳转，规避综合异常

```verilog
assign next_state[0] = state[0] & ~w | state[2] & w;
assign next_state[1] = state[0] & w | state[1] & w;
```

#### 2. 序列检测器（101、串行匹配）

区分：可重叠检测 / 不可重叠检测

不可重叠：匹配成功后，末尾bit作为下一轮起始状态，不要直接回到IDLE

#### 3. 串行编解码（PS/2、UART、HDLC）

- UART：起始位=0，使用计数器采样；停止位、奇偶校验失败直接丢弃数据
- HDLC：连续5个1后自动删除插入0；连续7个1代表报错；6个1为帧边界

#### 4. 运算电路：加法器、BCD修正

普通加法：`{cout, sum} = a + b + cin;`

BCD加法：4位和 ≥10 时，结果+6，同时产生进位，不能直接二进制加法

#### 5. 移位寄存器 & generate块

循环移位示例：`q <= {q[0], q[99:1]};`

generate循环**必须添加块标签**，否则综合报错

```verilog
genvar i;
generate
    for(i=0;i<8;i=i+1) : adder_block
    begin
        // 实例化代码
    end
endgenerate
```

## 五、仿真调试总结：常见时序问题与学习反思

### 故障现象1：输出z整体超前一拍 / 滞后一拍

- z写在时序always块内 → 滞后1clk
- z使用assign组合逻辑输出 → 和状态同步

\> 常规数字电路设计，均期望组合逻辑输出与状态同步对齐

### 故障现象2：状态机卡死、无状态跳转

1. 组合always块缺少default分支
2. case存在未覆盖分支，生成锁存器
3. next_state部分通路没有赋值

### 故障现象3：多驱动冲突报错

同一个信号，同时被 `assign` 和 `always` 赋值；或者两个always块驱动同一reg。

### 故障现象4：复位行为异常

- 异步复位敏感列表漏写复位信号
- 低电平复位误用 `posedge resetn`，应当使用 `negedge resetn`

### 故障现象5：波形完全没有输出脉冲

1. 状态转移条件写反（w=0/w=1箭头颠倒）
2. Moore/Mealy输出模型和评测预期不匹配
3. 计数器重置时机错误，采样窗口错位

## 六、整体学习总结与设计心得

1. 设计状态机时，优先梳理完整的状态跳转逻辑，严谨梳理条件分支，杜绝主观直觉判断，避免出现跳转条件颠倒的基础时序错误；
2. 时序波形错位的核心排查逻辑：区分**时序寄存输出**与**组合逻辑输出**的硬件本质差异，这是数字时序设计的核心要点；
3. 锁存器规避是组合逻辑设计的重中之重，通用设计思路为：在组合逻辑块内为所有变量预先设置默认值，杜绝不完全赋值；
4. 状态机设计必须牢记两类模型核心区别：Moore输出仅由当前状态决定，Mealy输出由当前状态+外部输入共同决定，时序特性完全不同。