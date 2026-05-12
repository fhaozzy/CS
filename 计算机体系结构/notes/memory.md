# 学习 Memory

更新时间：2026-05-08

这个文件记录稳定背景、复习策略和长期要遵守的学习偏好。后续每次完成一个知识块或发现新的考试线索，都应同步更新。

## 课程与资料背景

- 课程：计算机体系结构，HNU 课程资料 workspace。
- 当前学习目标：先学老师已经发的 PPT；Cache / AMAT / 存储层次等内容等老师后续发新 PPT 后再系统学习。
- 已有 PPT：
  - `ppt/第一章-量化计算与分析基础.pptx`
  - `ppt/附录A.pptx`
  - `ppt/L11-Pipelined-Datapath-And.ppt`
  - `ppt/附录C-流水线+记分牌和Tomasulo介绍.pptx`
- 已整理导览：
  - `notes/PPT学习导览.md`
- 参考课件：
  - `ppt/周学海课件/周学海课件/`
  - `notes/周学海参考课件索引.md`
  - 使用原则：只作为补充解释来源，不替代本校老师 PPT 和往年题导向。
- 往年资料索引：
  - `考试/往年资料/README.md`
  - `考试/往年资料/2025春季期末A卷_题型索引.md`
  - `考试/往年资料/2024期末考试回忆_题型索引.md`
  - `考试/往年资料/2023试题补充作业_题型索引.md`
  - `考试/往年资料/HNU期末复习汇总_资料索引.md`

## 学习偏好

- 讲解要以中文为主，尽量贴近期末题问法。
- 每个知识点最好配一个小例子或考试答题模板。
- 重点区分“概念理解”和“会做题”的差别。
- 不需要泛泛扩展太多未发 PPT 的内容，除非它直接帮助理解已学内容。
- 遇到高频考点时，要主动提醒它对应哪些往年题型。

## 备考策略

- 第一轮：按 PPT 建立知识框架。
- 第二轮：按往年题型补练例题。
- 第三轮：手推大题，尤其是流水线、数据冒险、Scoreboard、Tomasulo。
- 学习顺序：
  1. 第一章：量化分析基础。
  2. 附录 A：ISA 与 RISC-V。
  3. 流水线基础：IF、ID、EX、MEM、WB。
  4. 流水线冒险：结构冒险、数据冒险、控制冒险、转发、停顿、flush。
  5. 动态调度：Scoreboard 与 Tomasulo。
  6. Cache / AMAT：暂缓，等新 PPT。

## 参考课件使用规则

- 周学海课件可用于补充概念、例子和题型解释。
- 当前附录 A 阶段优先参考 `chapter02-02.pdf` 和 `chapter02-03.pdf`。
- 流水线阶段优先参考 `chapter03-02.pdf` 和 `chapter04-01.pdf`。
- Scoreboard/Tomasulo 阶段优先参考 `chapter05-03.pdf`、`chapter05-04.pdf`、`chapter05-05.pdf`。
- Cache 阶段暂缓，但后续可参考 `chapter04-02.pdf`、`chapter04-03.pdf`、`chapter04-04&05.pdf`、`chapter05-01.pdf`。
- 与本校 PPT 或老师课堂说法不一致时，以本校资料优先。

## 高频考点

- CPU Time 公式及分量分析：
  - `CPU Time = IC * CPI * Clock Cycle Time`
  - `CPU Time = IC * CPI / Clock Rate`
- Amdahl 定律：
  - `Speedup = 1 / ((1 - F) + F / S)`
- 平均 CPI：
  - `CPI = Σ(指令类型占比 * 该类 CPI)`
- RM vs RR / Load-Store。
- RISC-V 寄存器用途：`zero`、`ra`、`sp`、`a0-a7`、`t0-t6`、`s0-s11`。
- caller-saved vs callee-saved。
- RISC-V 指令格式：R、I、S、B、U、J。
- 五级流水线和流水线寄存器。
- RAW / WAR / WAW，尤其 RAW 和 load-use。
- forwarding、stall、flush。
- 分支预测：静态、动态、2 位预测。
- Scoreboard 四阶段与三张状态表。
- Tomasulo 三阶段、保留站、寄存器重命名、CDB。

## 暂缓内容

以下内容在 2025 A 卷中重要，但当前先不系统学，等老师发新 PPT 后补：

- Cache 块大小、miss rate、AMAT。
- miss per 1000 instructions。
- 统一 Cache vs 分离 I/D Cache。
- 包含式 / 互斥式缓存。
- L0/L1 Cache、现代 CPU/GPU 参数、大小核设计。

## 已形成的解释口径

- RM：运算指令可直接访问内存，指令可能更少，但硬件和流水线更复杂。
- RR / Load-Store：普通运算只在寄存器间进行，访存只能用 load/store；RISC-V 采用它是为了指令规整、译码简单、流水线友好。
- 流水线提高吞吐率，不是缩短单条指令延迟。
- 判断 CPU Time 分量口诀：
  - 少执行几条指令 -> IC
  - 少停顿/少等待 -> CPI
  - 时钟更快/周期更短 -> Clock Rate
