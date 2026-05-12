# 实验1：RISC-V 指令理解

## 一、实验任务概述

本实验围绕 RV32I 五级流水线处理器展开。实验材料给出了一个 RV32I Core 的数据通路图，要求我们从一段 C 程序出发，得到对应的 RISC-V 汇编代码，并进一步分析指定指令在 IF、ID、EX、MEM、WB 五个阶段中的数据流和控制信号。

本实验关注的重点不是单纯“读懂汇编”，而是理解汇编指令进入处理器后如何被硬件执行。换句话说，需要把下面三件事联系起来：

```text
C 程序语句
-> RISC-V 汇编指令
-> 流水线数据通路与控制信号
```

本报告主要完成以下内容：

1. 搭建并验证 RISC-V GCC 编译环境。
2. 编译实验 C 程序，得到 RV32I 汇编与反汇编结果。
3. 找出 `A[i] = A[i-1] + 1000;` 对应的循环汇编。
4. 分析 `add`、`bge`、`lw`、`sw` 四类典型指令的数据通路。
5. 说明 `BranchE` 信号、NPC Generator 的目标选择优先级以及 Hazard Unit 的停顿/冲刷策略。

## 二、实验环境和生成结果

### 2.1 使用的工具链

本实验在 Windows 上完成，使用课程提供的 SiFive RISC-V 交叉编译工具链。工具链路径为：

```text
D:\riscv-gcc\riscv64-unknown-elf-gcc-8.3.0-2020.04.1-x86_64-w64-mingw32
```

版本检查：

```text
riscv64-unknown-elf-gcc.exe (SiFive GCC 8.3.0-2020.04.1) 8.3.0
```

生成汇编和反汇编时使用如下命令：

```powershell
riscv64-unknown-elf-gcc -S -march=rv32i -mabi=ilp32 -O0 main.c -o main.s
riscv64-unknown-elf-gcc -c -march=rv32i -mabi=ilp32 -O0 main.c -o main.o
riscv64-unknown-elf-objdump -d -M no-aliases,numeric main.o > main.dump
```

其中 `-march=rv32i` 指定只使用 RV32I 基础整数指令集，因此本报告中的反汇编结果没有压缩指令。这样做的好处是每条指令都以标准 RV32I 形式展示，便于和五级流水线设计图对应。

### 2.2 实验 C 程序

实验程序如下：

```c
void main()
{
    int A[100];
    int i;

    for (i = 0; i < 100; i++)
        A[i] = i;

    for (i = 1; i < 100; i++)
        A[i] = A[i - 1] + 1000;
}
```

程序中有两个循环。第一个循环初始化数组，第二个循环利用前一个数组元素计算当前数组元素。本实验要求重点找出第二个循环对应的汇编代码，并分析其中出现的典型指令。

## 三、从 C 程序到汇编代码

### 3.1 寄存器名称说明

反汇编使用 `x` 编号寄存器。为了读代码方便，先列出本程序中常见寄存器的含义：

| ABI 名称 | 编号 | 在本程序中的作用 |
| --- | --- | --- |
| `sp` | `x2` | 栈指针 |
| `s0` | `x8` | 帧指针，用于访问局部变量 |
| `a3` | `x13` | 临时保存地址中间量 |
| `a4` | `x14` | 临时保存数据或地址中间量 |
| `a5` | `x15` | 临时保存循环变量、数组偏移或比较常数 |

局部变量 `i` 存放在栈帧中的 `-20(x8)` 位置。数组 `A` 也位于当前栈帧中，因此访问数组元素时，需要先利用 `x8` 和下标偏移计算地址。

### 3.2 main 函数反汇编

使用 `objdump` 得到的 `main` 函数如下：

```asm
00000000 <main>:
   0: e5010113           addi    x2,x2,-432
   4: 1a812623           sw      x8,428(x2)
   8: 1b010413           addi    x8,x2,432
   c: fe042623           sw      x0,-20(x8)
  10: 0280006f           jal     x0,38 <.L2>

00000014 <.L3>:
  14: fec42783           lw      x15,-20(x8)
  18: 00279793           slli    x15,x15,0x2
  1c: ff040713           addi    x14,x8,-16
  20: 00f707b3           add     x15,x14,x15
  24: fec42703           lw      x14,-20(x8)
  28: e6e7a623           sw      x14,-404(x15)
  2c: fec42783           lw      x15,-20(x8)
  30: 00178793           addi    x15,x15,1
  34: fef42623           sw      x15,-20(x8)

00000038 <.L2>:
  38: fec42703           lw      x14,-20(x8)
  3c: 06300793           addi    x15,x0,99
  40: fce7dae3           bge     x15,x14,14 <.L3>
  44: 00100793           addi    x15,x0,1
  48: fef42623           sw      x15,-20(x8)
  4c: 0400006f           jal     x0,8c <.L4>

00000050 <.L5>:
  50: fec42783           lw      x15,-20(x8)
  54: fff78793           addi    x15,x15,-1
  58: 00279793           slli    x15,x15,0x2
  5c: ff040713           addi    x14,x8,-16
  60: 00f707b3           add     x15,x14,x15
  64: e6c7a783           lw      x15,-404(x15)
  68: 3e878713           addi    x14,x15,1000
  6c: fec42783           lw      x15,-20(x8)
  70: 00279793           slli    x15,x15,0x2
  74: ff040693           addi    x13,x8,-16
  78: 00f687b3           add     x15,x13,x15
  7c: e6e7a623           sw      x14,-404(x15)
  80: fec42783           lw      x15,-20(x8)
  84: 00178793           addi    x15,x15,1
  88: fef42623           sw      x15,-20(x8)

0000008c <.L4>:
  8c: fec42703           lw      x14,-20(x8)
  90: 06300793           addi    x15,x0,99
  94: fae7dee3           bge     x15,x14,50 <.L5>
  98: 00000013           addi    x0,x0,0
  9c: 1ac12403           lw      x8,428(x2)
  a0: 1b010113           addi    x2,x2,432
  a4: 00008067           jalr    x0,0(x1)
```

### 3.3 第二个循环的汇编对应关系

第二个循环的 C 代码为：

```c
for (i = 1; i < 100; i++)
    A[i] = A[i - 1] + 1000;
```

对应汇编可分为四部分。

第一部分是循环变量初始化：

```asm
44: 00100793           addi    x15,x0,1
48: fef42623           sw      x15,-20(x8)
4c: 0400006f           jal     x0,8c <.L4>
```

含义是先将 `i` 置为 1，然后跳到 `.L4` 做循环条件判断。

第二部分计算并读取 `A[i-1]`：

```asm
50: fec42783           lw      x15,-20(x8)
54: fff78793           addi    x15,x15,-1
58: 00279793           slli    x15,x15,0x2
5c: ff040713           addi    x14,x8,-16
60: 00f707b3           add     x15,x14,x15
64: e6c7a783           lw      x15,-404(x15)
```

逐条解释：

```text
lw   x15,-20(x8)      取出 i
addi x15,x15,-1       得到 i-1
slli x15,x15,2        将下标乘 4，得到字节偏移
addi x14,x8,-16       生成数组寻址所需的基准地址中间值
add  x15,x14,x15      得到 A[i-1] 的地址中间值
lw   x15,-404(x15)    读取 A[i-1]
```

第三部分计算 `A[i-1]+1000` 并写入 `A[i]`：

```asm
68: 3e878713           addi    x14,x15,1000
6c: fec42783           lw      x15,-20(x8)
70: 00279793           slli    x15,x15,0x2
74: ff040693           addi    x13,x8,-16
78: 00f687b3           add     x15,x13,x15
7c: e6e7a623           sw      x14,-404(x15)
```

这里 `x14` 保存计算结果，`x15` 重新用于计算 `A[i]` 的地址。

第四部分完成 `i++` 并判断循环条件：

```asm
80: fec42783           lw      x15,-20(x8)
84: 00178793           addi    x15,x15,1
88: fef42623           sw      x15,-20(x8)

8c: fec42703           lw      x14,-20(x8)
90: 06300793           addi    x15,x0,99
94: fae7dee3           bge     x15,x14,50 <.L5>
```

最后一条 `bge` 的逻辑是：

```text
若 99 >= i，则跳回 .L5 继续执行循环体。
```

这与 C 语言中的 `i < 100` 等价。

实验指导中写的是 `bge x15,x14,-56`。本报告中反汇编显示为跳到 `50 <.L5>`，二者都表示向回跳转到循环体。偏移数值会受到指令长度和编译布局影响，不影响这条指令的语义和数据通路分析。

## 四、分析数据通路时采用的口径

为了避免只罗列信号而缺少理解，本报告先统一说明分析方法。对每条指令，可以按照以下顺序判断：

```text
1. 这条指令需要读哪些寄存器？
2. 是否需要生成立即数？立即数属于哪种格式？
3. ALU 在这条指令中承担什么任务？
4. 是否访问 Data Memory？
5. 是否在 WB 阶段写回寄存器？
6. 是否会改变下一条 PC？
```

四条指令的总体差异如下：

| 指令 | 类型 | 主要任务 | 最终结果去向 |
| --- | --- | --- | --- |
| `add x15,x14,x15` | R-type | 两个寄存器相加 | 写回 `x15` |
| `bge x15,x14,-56` | B-type | 比较两个寄存器并决定是否跳转 | 改变 PC 或顺序执行 |
| `lw x15,-20(x8)` | I-type load | 计算地址并读内存 | 写回 `x15` |
| `sw x15,-20(x8)` | S-type store | 计算地址并写内存 | 写入 Data Memory |

后续分析中的控制信号使用语义值表示。例如 `AluSrc2=Imm` 表示 ALU 第二个输入来自立即数，而不是给出具体编码。

## 五、典型指令的数据通路和控制信号

### 5.1 `add x15, x14, x15`

#### 5.1.1 指令作用

```asm
add x15, x14, x15
```

其语义为：

```text
x15 = x14 + x15
```

在第二个循环中，这条指令位于：

```asm
58: slli    x15,x15,0x2
5c: addi    x14,x8,-16
60: add     x15,x14,x15
```

其中 `x15` 保存数组下标偏移，`x14` 保存基准地址中间值，因此这条 `add` 实际上是在形成数组元素地址。

#### 5.1.2 分阶段分析

| 阶段 | 数据流 | 关键控制信号 |
| --- | --- | --- |
| IF | `PCF` 送入 Instruction Memory，取出 `add` 指令 | `StallF=0`，`FlushF=0`，NPC 选择 `PC+4` |
| ID | 译码得到 `rs1=x14`、`rs2=x15`、`rd=x15`；Register File 读出两个源操作数 | `RegWriteD=1`，`MemWriteD=0`，`MemToRegD=0`，`BranchTypeD=NoBranch`，`AluControlD=ADD`，`AluSrc1D=Reg`，`AluSrc2D=Reg` |
| EX | 两个源操作数进入 ALU，执行加法 | `AluControlE=ADD`，`AluSrc1E=Reg/Forward`，`AluSrc2E=Reg/Forward` |
| MEM | 不访问数据存储器，ALU 结果继续向后传递 | `MemWriteM=0`，`RegWriteM=1`，`MemToRegM=0` |
| WB | 将 ALU 结果作为 `ResultW` 写入 `x15` | `RegWriteW=1`，`MemToRegW=0`，`A3=x15` |

这条指令的主数据通路为：

```text
Instruction Memory
-> Register File 读 x14 和 x15
-> ALU 做加法
-> 写回 Register File 的 x15
```

需要注意的是，本程序中该 `add` 紧跟在 `slli` 和 `addi` 后面，两个源寄存器的值可能还没有真正写回寄存器堆。因此处理器需要通过 forwarding 将较新的结果送到 ALU 输入端。此时 `Forward1E`、`Forward2E` 的选择会影响 `add` 是否拿到正确操作数。

### 5.2 `bge x15, x14, -56`

#### 5.2.1 指令作用

```asm
bge x15, x14, -56
```

语义为：

```text
if ((signed)x15 >= (signed)x14)
    PC = PC + offset;
else
    PC = PC + 4;
```

在循环判断处，`x15` 中保存 99，`x14` 中保存 `i`。因此：

```text
bge x15,x14,target
```

等价于：

```text
if (99 >= i) goto loop_body;
```

也就是继续执行 `i < 100` 的循环体。

#### 5.2.2 分阶段分析

| 阶段 | 数据流 | 关键控制信号 |
| --- | --- | --- |
| IF | 取出 branch 指令；若采用默认不跳转预测，取指阶段先按 `PC+4` 继续 | `StallF=0`，`FlushF=0`，NPC 暂选 `PC+4` |
| ID | 译码得到 `rs1=x15`、`rs2=x14`；Immediate Unit 生成 B-type 偏移量 | `RegWriteD=0`，`MemWriteD=0`，`BranchTypeD=BGE`，`ImmTypeD=B`，`RegReadD=rs1,rs2` |
| EX | Branch Decision 比较 `x15 >= x14`，并生成 `BrE`；同时得到 branch target | `BranchTypeE=BGE`，`BrE=比较结果`，`RegWriteE=0`，`MemWriteE=0` |
| MEM | 不读写 Data Memory | `MemWriteM=0`，`RegWriteM=0` |
| WB | 无寄存器写回 | `RegWriteW=0` |

这条指令的数据通路可以概括为：

```text
Instruction Memory
-> Register File 读 x15 和 x14
-> Immediate Unit 生成 B-type offset
-> Branch Decision 判断条件
-> 条件成立时通知 NPC Generator 选择 branch target
```

`bge` 和 `add` 的区别在于：`add` 的计算结果要写回寄存器，而 `bge` 的“结果”是一个控制结果，即下一条 PC 到底走顺序地址还是跳转地址。

#### 5.2.3 控制冒险说明

如果采用静态不跳转预测，则 branch 在 IF 阶段之后，前端会先取 `PC+4` 的指令。当 `bge` 到 EX 阶段后：

- 若 `BrE=0`，说明不跳转，预测正确。
- 若 `BrE=1`，说明实际要跳转，之前按顺序取入的指令属于错误路径，需要 flush。

因此 `BrE` 不只是比较结果，它还会触发 NPC 重定向和流水线冲刷。

### 5.3 `lw x15, -20(x8)`

#### 5.3.1 指令作用

```asm
lw x15, -20(x8)
```

语义为：

```text
x15 = Mem[x8 - 20]
```

本程序中 `-20(x8)` 是变量 `i` 的存放位置，因此该指令常用于把 `i` 从栈帧中读入寄存器。

#### 5.3.2 分阶段分析

| 阶段 | 数据流 | 关键控制信号 |
| --- | --- | --- |
| IF | 取出 `lw` 指令 | `StallF=0`，`FlushF=0`，NPC 选择 `PC+4` |
| ID | 译码得到 `rs1=x8`、`rd=x15`；读出基址寄存器 `x8`；生成 I-type 立即数 `-20` | `RegWriteD=1`，`MemToRegD=1`，`MemWriteD=0`，`ImmTypeD=I`，`AluControlD=ADD`，`AluSrc1D=Reg`，`AluSrc2D=Imm` |
| EX | ALU 计算有效地址 `x8 + (-20)` | `AluControlE=ADD`，`AluSrc1E=Reg`，`AluSrc2E=Imm` |
| MEM | Data Memory 按 ALU 输出地址读取一个 word | `MemWriteM=0`，`MemToRegM=1`，`RegWriteM=1` |
| WB | 选择内存读出的数据写回 `x15` | `RegWriteW=1`，`MemToRegW=1`，`A3=x15` |

这条指令的核心路径为：

```text
Register File 读 x8
-> Immediate Unit 生成 -20
-> ALU 计算内存地址
-> Data Memory 读数据
-> WB 写回 x15
```

对于 `lw`，ALU 的输出不是最终要写回的值，而是访存地址。最终写回寄存器的数据来自 Data Memory，因此 `MemToReg=1`。

### 5.4 `sw x15, -20(x8)`

#### 5.4.1 指令作用

```asm
sw x15, -20(x8)
```

语义为：

```text
Mem[x8 - 20] = x15
```

本程序中，该指令用于把更新后的循环变量 `i` 保存回栈帧。

#### 5.4.2 分阶段分析

| 阶段 | 数据流 | 关键控制信号 |
| --- | --- | --- |
| IF | 取出 `sw` 指令 | `StallF=0`，`FlushF=0`，NPC 选择 `PC+4` |
| ID | 译码得到 `rs1=x8`、`rs2=x15`；读出基址和待写数据；生成 S-type 立即数 | `RegWriteD=0`，`MemWriteD=1`，`ImmTypeD=S`，`AluControlD=ADD`，`AluSrc1D=Reg`，`AluSrc2D=Imm` |
| EX | ALU 计算有效地址 `x8 + (-20)`；待写数据随流水线传到 MEM | `AluControlE=ADD`，`AluSrc1E=Reg`，`AluSrc2E=Imm` |
| MEM | Data Memory 将 `StoreDataM` 写入 `AluOutM` 指定地址 | `MemWriteM=1`，`RegWriteM=0` |
| WB | store 指令没有写回 | `RegWriteW=0` |

这条指令的核心路径为：

```text
Register File 读 x8 和 x15
-> Immediate Unit 生成 -20
-> ALU 计算内存地址
-> Data Memory 写入 x15 的值
```

`sw` 和 `lw` 都用 ALU 计算地址，但二者的数据方向相反。`lw` 是从内存到寄存器，`sw` 是从寄存器到内存。因此 `sw` 的 `MemWrite=1`，而 `RegWrite=0`。

## 六、四条指令控制信号对比

将四条指令放在一起，可以更清楚地看到它们的差别：

| 控制信号 | `add` | `bge` | `lw` | `sw` |
| --- | --- | --- | --- | --- |
| 指令类型 | R-type | B-type | I-type | S-type |
| `RegWrite` | 1 | 0 | 1 | 0 |
| `MemWrite` | 0 | 0 | 0 | 1 |
| `MemToReg` | 0 | X | 1 | X |
| `LoadNpc` | 0 | 0 | 0 | 0 |
| `RegRead` | `rs1,rs2` | `rs1,rs2` | `rs1` | `rs1,rs2` |
| `BranchType` | 无 | BGE | 无 | 无 |
| `ImmType` | 无 | B | I | S |
| `AluControl` | ADD | 比较相关/无写回 | ADD | ADD |
| `AluSrc1` | Reg | Reg | Reg | Reg |
| `AluSrc2` | Reg | Reg | Imm | Imm |
| Data Memory | 不访问 | 不访问 | 读 | 写 |
| WB 写回 | ALU 结果 | 无 | 内存读出值 | 无 |

这张表也体现出一个规律：控制信号的本质是根据指令类型来选择数据通路。指令要写寄存器，就打开 `RegWrite`；指令要写内存，就打开 `MemWrite`；指令写回数据来自内存，就打开 `MemToReg`；指令需要立即数参与 ALU 运算，就让 `AluSrc2` 选择立即数。

## 七、BranchE 信号分析

`BranchE` 是 EX 阶段分支判断结果信号，也可以写作 `BrE`。它由 Branch Decision 模块产生。

以 `bge x15,x14,-56` 为例，Branch Decision 根据 `BranchTypeE=BGE` 执行有符号比较：

```text
BranchE = (x15 >= x14)
```

当 `BranchE=1` 时，说明分支条件成立，下一条 PC 应该改为 branch target；当 `BranchE=0` 时，说明分支不成立，继续顺序执行。

因此，`BranchE` 有两个直接作用：

1. 作为 NPC Generator 的选择依据。如果 `BranchE=1`，NPC Generator 应选择 `BrT`，而不是默认的 `PC+4`。
2. 作为 Hazard Unit 判断是否需要冲刷的依据。如果采用默认不跳转预测，而 `BranchE=1`，说明先前取入的顺序指令是错误路径，需要清除。

所以 `BranchE` 不是普通的数据结果，而是会影响取指方向的控制信号。

## 八、NPC Generator 的 target 选择优先级

NPC Generator 面临的候选下一 PC 主要有四类：

```text
PC + 4      顺序执行
JalT        jal 的跳转目标
JalrT       jalr 的跳转目标
BrT         branch 成立时的跳转目标
```

这些 target 之间需要有优先级。原因是流水线中不同阶段的指令年龄不同。EX 阶段的指令比 ID 阶段的指令更早进入流水线，因此当 EX 阶段指令确认需要改变 PC 时，应当优先处理 EX 阶段的重定向。

一种合理的优先级可以表示为：

```text
JalrE / BrE 产生的 EX 阶段重定向
> JalD 产生的 ID 阶段重定向
> PC + 4
```

若把 EX 阶段的两类情况展开，可写成：

```text
JalrT
> BrT
> JalT
> PC + 4
```

不过正常情况下，EX 阶段同一条指令不可能同时既是 `jalr` 又是 branch，因此 `JalrT` 和 `BrT` 的相对先后主要是硬件实现细节。真正需要强调的是：EX 阶段的跳转/分支修正要优先于 ID 阶段的 `jal`。

举例说明：假设某个周期 EX 阶段的 `bge` 判断成立，同时 ID 阶段译码出一条 `jal`。这时 `jal` 是更年轻的指令，而且可能已经位于错误路径上。NPC Generator 应该选择 EX 阶段 branch 的目标 `BrT`，并冲刷后面的错误路径指令，而不能让 ID 阶段的 `JalT` 覆盖掉更早指令的控制流结果。

## 九、Hazard Unit 和气泡插入

流水线提高了指令吞吐率，但也带来了冒险问题。本实验设计图中的 Hazard Unit 主要处理数据冒险和控制冒险。

### 9.1 可以通过 forwarding 解决的数据相关

例如：

```asm
slli x15,x15,0x2
addi x14,x8,-16
add  x15,x14,x15
```

`add` 需要的 `x15`、`x14` 都是前面刚产生的值。如果等待它们写回寄存器堆，会产生不必要的停顿。此时可以使用 forwarding，将 MEM 或 WB 阶段中已经得到的结果直接送到 EX 阶段 ALU 输入端。

这类情况一般不需要插入 NOP，只需要正确设置：

```text
Forward1E
Forward2E
```

### 9.2 需要插入气泡的三类情况

根据本实验中的五级流水线结构，需要特别关注以下三类会引入气泡或冲刷的情况。

| 情况 | 原因 | 典型控制方式 | 代价 |
| --- | --- | --- | --- |
| load-use 数据冒险 | load 的结果到 MEM 后才可用，后一条指令在 EX 就要使用 | 暂停 IF/ID，冲刷 EX 插入 NOP | 1 个周期 |
| `jal` 控制冒险 | `jal` 在 ID 阶段确定目标，但 IF 已经取了顺序下一条 | 选择 `JalT`，flush 错误取入指令 | 约 1 个气泡 |
| branch/jalr 控制冒险 | branch/jalr 到 EX 阶段才确定 PC 是否重定向 | 选择 `BrT` 或 `JalrT`，flush 错误路径 | 约 2 个气泡 |

load-use 的例子：

```asm
lw   x15,0(x8)
add  x16,x15,x17
```

`lw` 读出的数据在 MEM 阶段才得到，而紧随其后的 `add` 在 EX 阶段就要使用 `x15`。这时 forwarding 也来不及解决，只能让前端停一个周期：

```text
StallF = 1
StallD = 1
FlushE = 1
```

这样 EX 阶段插入一个气泡，等 load 数据可以被前递后，再继续执行后续指令。

### 9.3 静态不跳转预测下 branch 的 flush/stall

如果采用静态分支预测，并且默认 branch 不跳转，则处理器遇到 branch 时会先按顺序地址继续取指。

当 branch 到 EX 阶段后，分两种情况：

若 `BranchE=0`：

```text
实际不跳转，预测正确
NPC 继续选择 PC+4
FlushD = 0
FlushE = 0
StallF = 0
StallD = 0
```

若 `BranchE=1`：

```text
实际跳转，预测错误
NPC 改选 BrT
FlushD = 1
FlushE = 1
StallF = 0
StallD = 0
```

也就是说，branch 的预测错误通常不是通过 stall 解决，而是通过 flush 清空错误路径上的指令，然后从正确目标地址重新取指。

## 十、实验结论

本实验从一段 C 程序出发，观察了编译器生成的 RV32I 汇编，并重点分析了第二个循环中出现的典型指令。通过分析可以得到以下结论：

1. `add` 属于 R-type 指令，主要经过 Register File、ALU 和 WB。它不访问数据存储器，写回值来自 ALU。
2. `lw` 和 `sw` 都需要用 ALU 计算访存地址，但数据方向不同。`lw` 从内存读数据并写回寄存器，`sw` 将寄存器数据写入内存且不写回寄存器。
3. `bge` 的核心不是写回结果，而是通过 Branch Decision 得到 `BranchE`，再影响 NPC Generator 对下一条 PC 的选择。
4. 控制信号的作用可以理解为“为当前指令选择正确的数据路径”。例如 `RegWrite` 控制是否写寄存器，`MemWrite` 控制是否写内存，`MemToReg` 控制写回数据来源，`AluSrc2` 控制 ALU 第二个输入来源。
5. Hazard Unit 通过 forwarding、stall 和 flush 保证流水线正确执行。普通 ALU 相关通常可由 forwarding 解决，load-use 冒险需要 1 个周期停顿，branch 预测错误需要冲刷错误路径。

通过本实验，对 RISC-V 指令的理解从“汇编语义层面”推进到了“硬件执行层面”。一条指令是否读寄存器、是否生成立即数、是否使用 ALU、是否访问内存、是否写回寄存器、是否改变 PC，都会反映到具体的数据通路和控制信号中。这也是后续学习流水线优化、分支预测和乱序执行的基础。
