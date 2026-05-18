---
title: "基于 RISC-V RoCC 的卷积加速器"
date: 2026-05-09 22:30:00
lang: zh-CN
tags:
  - RISC-V
  - Chisel
  - RoCC
categories:
  - MyConvAccel
---

## 背景

卷积是深度学习和图像处理最基本的运算之一。当我们在 CPU 上用 5×5 kernel 对 32×32 的矩阵进行卷积运算的时候，需要大量的 cycle 去完成卷积运算，造成了严重的性能浪费。

因此我们使用基于 RISC-V 的 RoCC（Rocket Custom Coprocessor）接口设计了一个专门用于计算卷积的加速器——把算力密集的任务从 CPU 卸载到专用硬件上。CPU 只需要发出指令，RoCC 即可进行运算，其中不需要 CPU 进行任何参与。

- Repository：[MyConvAccel](https://github.com/Peanut-321/MyConvAccel)

## Phase 0：方案规划 & 协议设计

首先我们需要设计此项目的架构以及目标。

### 协议设计：RoCC 指令编码

RoCC 协议：CPU 通过 custom instruction 把操作数(`rs1`, `rs2`)发送给加速器，等待加速器计算结束后通过`resp.data` 再把结果发送回来。

```
Rocket Core (CPU)                      Accelerator
      |                                      |
      |  custom inst (rs1=addr, rs2=data)    |   ← RoCC interface: control
      | -----------------------------------> |
      |                                      |
      |  DMA read / write                    |   ← TileLink bus: data
      | <==================================> |
      |                                      |
      |  resp.data = status / result         |   ← RoCC interface: status
      | <----------------------------------- |
```

因此我们加速器使用三步走的流程：**setting address → trigger → polling**

CPU 先告诉输入矩阵、kernel、输出各在内存的什么位置，然后发一个 START，最后 polling status register 观察加速器是否完成运算。

Rocket Core 通过 RoCC 接口和加速器进行通信。每条 custom instruction 带有 `funct7`、`rs1`、`rs2`、`rd` 四个字段。

| funct7 | Instruction | rs1 | Description |
|--------|------|-----|------|
| 0 | SET_ADDR_IN | base addr | Base address of input matrix |
| 1 | SET_ADDR_KER | base addr | Base address of kernel |
| 2 | SET_ADDR_OUT | base addr | Base address of output matrix |
| 3 | START_ACCEL | — | Non-blocking start |
| 4 | POLL_STATUS | — | Read status register to rd |

设计包含四位状态寄存器用于进行polling。

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | busy | Accelerator is computing |
| 1 | done | Computation complete |
| 2 | overflow | Accumulator overflow |
| 3 | addr_err | Address check failed |

### 设计决策

1. **为什么地址分三条独立指令？**

    一条 RoCC 指令只带两个操作数（`rs1` 和 `rs2`），不够一次传三个base address。替代方案是传结构体指针，但那会让加速器为了读配置额外发起一次 DMA，导致复杂度和延迟增加。设置三条独立的指令，一次只传一个地址，硬件接口最干净。

2. **为什么non-blocking 启动 + polling，而不是阻塞等结果？**

    non-blocking 允许 CPU 发出 `START_ACCEL` 后立刻去进行其他工作。选polling不选中断，是因为 5×5 卷积的延迟很短而且可预期——中断控制器和上下文切换的开销反而更大。

3. **为什么 funct7 连续编码而不是跳跃分配？**

    一个 `funct7 <= 4` 判断就能覆盖全部合法指令，解码器最小。跳跃编码会增加combinational logic。

### 关键设计取舍

下面几个问题需要在项目开始前定下来。

1. **kernel 大小是固定还是可配置的？**

    如果支持可变kernel导致control logic复杂度显著增加。最终选取固定的 5×5 硬件datapath。对于小于 5×5 的kernel由软件预先zero-padding补齐。

2. **边界像素怎么处理？**

    zero-padding是最简单的办法——滑动窗口 FSM 不需要任何边界处理特殊逻辑。

3. **为什么选fixed-point不用floating-point？**

    - **为什么不用floating-point？** IEEE 754 multiplier面积大、延迟高。Q8.8 fixed-point multiplier就是一个 16-bit × 16-bit 整数乘法，一个周期出结果，硬件零开销。
    
    - **为什么不用纯整数？** kernel权重基本上都是小数（比如 0.125、-0.5），没有办法用纯整数表示。这里使用Q8.8定点数：8 位整数部分（范围 ±128）与8 位小数部分（精度 1/256）进行组合形成一个16bits数据。
    
    - **为什么accumulator是 32 位？** 5×5 卷积要累加 25 个 Q8.8 定点数的乘积。每个 16-bit × 16-bit 产生 32 位数据，32 位accumulator刚好能做到不溢出。

4. **为什么有地址对齐约束？**

    TileLink DMA 在地址按字对齐时进行搬运最高效。如果地址没对齐的话，一次传输会被拆分成多次传输，同时硬件需要对其进行字节移位和拼接的处理。因此只需要在加速器运行中对每条 `SET_ADDR_*` 检查地址即可。

| Parameter | Value |
|------|-----|
| Input Matrix | 32×32 |
| Max Kernel | 5×5 |
| Data Format | Q8.8 Fixed Point（16-bit） |
| Accumulator | 32-bit |
| Output | Same size, zero-padding |
| Performance Target | <2500 cycles, ≥40× speedup |

---

## Phase 1：架构总览

### 问题：为什么通过 CPU 直接进行卷积运算会这么慢？

5×5 kernel 在 32×32 的矩阵上滑动运算的时候，总共进行 1024 次输出。对于每次输出要进行 25 次乘法和 24 次加法——1024个输出一共约 25,000 次乘加。在 CPU 上，真正的限制是"滑动"本身。Kernel每次进行滑动，CPU 需要计算地址偏移、更新循环变量、从内存加载像素和权重、再写回结果，因此绝大多数指令花在了循环控制和地址搬运。

因此我们可以进行粗略估计：在一颗简单的顺序 RISC-V 核上，每个输出像素大约花 100~150 cycles（两次 load，一次 store，约 50 条算数指令，外加循环分支）。1024 个像素加起来，**10 万到 15 万 cycles**。而我们的加速器干同样的活只要 **2428 cycles**——大约 50 倍的加速。

### 把计算卸载进加速器中

当我们使用加速器去计算卷积的时候，CPU只需要发指令即可。加速器自行完成剩下其他步骤：从内存中取数据、计算卷积、写回数据。CPU只需要发出：SET, START以及POLL指令。

加速器内部看上去是什么样的？

最简单的方法是将DMA load部分，MAC计算部分与DMA store部分进行串行连接。这种连接非常简单，但是依旧会产生大量的资源浪费。当DMA进行数据传输的时候，MAC部分需要暂停运算等待数据传输结束，反之亦然。因此此链接方法在同一个周期内只有一个硬件进行工作。为了提高加速器的工作效率，我们将三个硬件工作进行重叠，使其能够流水线工作。最终达到接近 50 倍的加速。

### Module Map

| Module | Role | Phase |
|--------|------|-------|
| ConvControl | RoCC decode + 5-state master FSM + status register | 2 |
| ConvDMA | 7-state DMA engine over TileLink | 3, 4 |
| LineBuffer | 5×32×16 sliding window with zero-padding | 5 |
| InputQueue + StoreQueue | Elastic buffers with backpressure | 7 |
| ConvEngine | ShiftWindow + KernelROM + ConvUnit, 6-stage MAC pipeline | 6 |

The following phases walk through each module in detail.

---

## Phase 2：ConvControl-指令解码与控制FSM

之前我们已经设计了整个加速器的架构，由 Phase 1 可知，最顶层是 RoCC control。其中 CPU 发出 `funct7`，ConvControl 负责进行解码以及执行，最终通过 `rd` 进行回复。

### Interface

ConvControl 通过五个信号和一个有效就绪握手与 CPU 通信：当 CPU 接收到 `instrCmd.valid`、`funct7`、`rs1` 和 `rd` 信号，并返回 `instrReady` 信号。当 `valid` 和 `ready` 在同一周期内都为高电平时，即可执行指令。

### 指令解码

在解码过程中使用五个比较器——`funct7 === 0.U` 到 `funct7 === 4.U`，不需要优先级编码器，也不需要查找表。Phase 0 的连续编码在这里得到了回报：`funct7` 本身就是指令编号。

- **SET (0–2):** 在 sBusy 状态下阻塞。在计算过程中更改地址会破坏运算。
- **START (3):** 仅在 sIdle 或 sDone 状态下接受。不能启动已在运行的加速器。
- **POLL (4):** 始终接受。这是一个纯读取操作——不会对任何操作进行干扰。

### SET_ADDR：存储三个基地址

根据 `funct7` 的值，`rs1` 被写入 `addrIn`、`addrKer`、`addrOut` 中。如果加速器处于 `sError` 状态，任何 SET 都会将其清除并回到 `sIdle`。

### START_ACCEL：地址检查与触发

1. **地址非零检查**

    如果任意一个基地址（`addrIn`、`addrKer`、`addrOut`）为零，状态跳转到 sError。

2. **地址对齐检查**

    输入和输出矩阵要求 8 字节对齐（`addrIn(2,0) === 0.U`）。kernel 要求 2 字节对齐（`addrKer(0) === 0.U`）。DMA 总线宽度为 64 位，每次传输搬 4 个像素。kernel 只有 25 个系数，2 字节对齐足够。如果数据未对齐，硬件需要额外进行字节移位和拼接。

所有检查通过 → sIdle → sBusy，计算开始。任意一项失败 → sError，status register 中 `addr_err` 位置位。

### FSM：四状态

```
                   START pass                done
          ┌─── sIdle ────────────────► sBusy ─────────────► sDone
          │      ▲                       ▲                    │
          │      │                       │ START              │ SET
          │      │                       └────────────────────┘
          │      │
          │      │ SET
          │      │
          │ START fail
          │      │
          │      ▼
          └──► sError
```

- **sIdle：** 复位后的默认状态。等待 START。
- **sBusy：** 只接受 POLL。计数到零后拉高 `done`，进入 sDone。
- **sDone：** 保持 `done = 1`。CPU 可以重新 START（→ sBusy）或 SET 重配地址（→ sIdle）。
- **sError：** 保持 `addrErr = 1`。只有 SET 能拉回 sIdle——没有直达 sBusy 的路径。

---

## Phase 3：ConvDMA — TileLink DMA 引擎

卷积引擎需要频繁地从 SRAM 读取像素和权重，再将结果写回。ConvDMA 负责管理这些数据传输。Phase 3 从严格串行的 DMA 开始——同一时刻只有一个请求在飞。这样 FSM 足够简单，初始测试也能逐周期精确验证。验证通过后，我们再添加流水线，让多个请求可以不等前一个响应就发出。

### DMA 接口定义

三个纯 Bundle 连接 DMA 与 L1 data cache。仅信号声明，不含逻辑——valid/ready 握手由 `Decoupled` 提供。

- **SimpleMemReq** — DMA 发给 L1 的请求。携带 64-bit 地址、64-bit 写数据、字节掩码、读写标志和 4-bit tag。串行 DMA 中 tag 恒为 0，预留 4-bit 给流水线版本做乱序响应匹配。
- **SimpleMemResp** — L1 的回复。返回 64-bit 读数据，并回传请求的 tag。
- **SimpleMemIO** — 将 `req`（DMA→L1）和 `resp`（L1→DMA，`Flipped`）捆为一个端口。

```
      ConvDMA                  SimpleMemIO            L1 data cache
    ┌─────────┐              ┌─────────────┐          ┌──────────┐
    │         │── req ──────►│ req (output) │────────►│          │
    │  FSM    │              │             │          │  L1 D$   │
    │         │◄─ resp ──────│ resp (input) │◄────────│          │
    └─────────┘              └─────────────┘          └──────────┘
```

### 串行 FSM

ConvDMA 有两条数据通路：

- **Load path**（sIdle → sIssue → sWaitResp → sUnpack → 循环）：从内存读 64-bit word，拆解为 4 个 16-bit 元素，推入 `elemQueue` 供计算单元消费。
- **Store path**（sIdle → sGather → sIssue → 循环）：从计算单元收集 4 个 16-bit 元素，拼成 64-bit word，写回内存。

两条路径共享 `sIssue`，通过 `opReg` 分叉。严格遵守"发射一个请求 → 等响应 → 拆/拼数据 → 再发射下一个"的串行铁律——任意时刻只有 1 个请求在飞。

```
Load path:   sIdle → sIssue → sWaitResp → sUnpack(×4) ─┐
              ▲                                        │
              └────────────────────────────────────────┘

Store path:  sIdle → sGather(×4) → sIssue ─┐
             ▲                             │
             └─────────────────────────────┘

Error path:  sIdle ──► sError  (SET addr ⇒ sIdle)
```


### 瓶颈：6 cycles/word

以 load path 为例，读一个 word 走 sIssue → sWaitResp → sUnpack×4 → 回到 sIssue，恰好 6 拍。sUnpack 占 4 拍，内存在这 4 拍完全空闲。瓶颈不在内存延迟，而在 FSM 不让发射和拆包重叠。

```
 cyc |     state |     mem.req    |   mem.resp   | loadStream
   0 |    sIssue | fire(rd)       |              |
   1 | sWaitResp |                | fire         |
   2 |   sUnpack |                               |
   3 |   sUnpack |                               | deq elem[0]
   4 |   sUnpack |                               | deq elem[1]
   5 |   sUnpack |                               | deq elem[2]
   6 |    sIssue | fire(rd)       |              | deq elem[3]
```

256 word × 6 + 1 cycle overhead = **1537 cycles**。

### 流水线化 DMA

这一步解决串行 FSM 的瓶颈。核心思路：将发射（issue）和拆包（unpack）解耦为两个并发硬件进程，在时间上重叠。

```
Serial:
  Issue:    [sIssue]              [sIssue]              [sIssue]
  Response:    [sWait]               [sWait]               [sWait]
  Unpack:         [sUnpack×4]            [sUnpack×4]            [sUnpack×4]
             ↑── 6 cycles/word ──↑

Pipelined:
  Issue:    [sIssue][sIssue][sIssue][sIssue][sIssue]...
  Response:    [1 cycle]  [1 cycle]  [1 cycle]  [1 cycle]  [1 cycle]  ← enqueue FIFO
  Unpack:         [sUnpack×4][sUnpack×4][sUnpack×4]...               ← dequeue FIFO
             ↑── 4 cycles/word (unpack is the bottleneck) ──↑
```

响应 FIFO 吸收速率差：issue 引擎以 1 word/cycle 发射（直到 inflight 上限），响应自动落入 FIFO，unpack 引擎以 4 cycles/word 从 FIFO 取出拆解。两者互不阻塞。1537 cycles → ~1033 cycles，提升约 33%。逐步实现细节见后续文章。

#### 串行到流水线的四项改动

**1. Register → Queue**

串行 DMA 用单个 `respWord` 寄存器锁存一次响应——因为最多只有 1 个 in-flight 请求。流水线版本替换为响应 FIFO 缓冲多个响应。数据自动落入 FIFO，FSM 只在需要时从 FIFO 取数。这拆掉了响应收集与 FSM 状态机之间的耦合。

**2. sWaitResp 退化**

`sWaitResp` 不再等 `mem.resp.valid`，也不再锁存响应数据。唯一剩下的工作就是等 FIFO 有数据可取。

**3. inflightCount — 信用制流控**

issue 引擎发射速率（1 word/cycle）快于 unpack 引擎消费速率（1 element/cycle → 0.25 word/cycle）。不加限流 FIFO 会溢出。`inflightCount` 作为信用账户：issue 每发一个请求消耗 1 点信用，unpack 每拆完一个 word 归还 1 点信用。`inflightMax = 4` 将窗口限制为 4 个 in-flight word，恰好匹配 unpack 引擎消化速度。

**4. sLoadActive — 并发 FSM（核心改动）**

将串行的三个状态 `sIssue`、`sWaitResp`、`sUnpack` 合并为一个 `sLoadActive` 状态。在此状态内，两个独立的 `when` 块并发运行：issue 引擎发射请求直至 `inflightMax`，unpack 引擎从 FIFO 排空数据。两者可在同一拍内同时推进——issue 与 unpack 真正重叠。

---

## Phase 4：计算数据通路 — LineBuffer & ConvEngine

DMA 每个周期送入一个像素，顺序为行优先——第 0 行从左到右，然后第 1 行、第 2 行。但 5×5 卷积在输出像素 (r, c) 处需要 25 个像素，排列为以 (r+2, c+2) 为中心的 5×5 邻域。单个像素毫无用处——计算数据通路必须在 MAC 单元启动前拼出完整的 5×5 窗口。

这是一个数据重组问题，自然拆成两个维度：LineBuffer 负责垂直方向（行），ShiftWindow 负责水平方向（列）。LineBuffer 从 DMA 流中收集 5 行 × 32 列像素，然后每周期输出同一列上 5 个垂直相邻像素——从行优先到列优先，做了 90° 旋转。ShiftWindow 缓冲来自 LineBuffer 的连续 5 列，每周期右移，输出完整的 5×5 窗口给 MAC 单元。

**为什么不直接从 SRAM 读取？** 每个输出像素需要来自 5 个不同行的值。直接读取需要每周期 5 次独立读取，指向 5 个不同地址——五个内存端口。LineBuffer 用一个写端口（DMA 每周期写入一个像素）和一个 5 路读端口（5 行 × 同一列）替代了它。160 条目的寄存器堆远便宜过一个 5 端口 SRAM。

```
    sIdle ──► sPrime (加载前 5 行) ──► sActive (32 个输出行) ──► sDone
```

零填充通过两种机制覆盖四个边界：

- **上 / 下：** 由五个缓冲行存的内容决定。输出行 0 时，顶部两行为零；输出行 31 时，底部两行为零。窗口下滑过程中，真实行自然轮转进出——无需额外控制逻辑。
- **左 / 右：** 每个输出行输出 36 列（2 左填充 + 32 数据 + 2 右填充）。`colValid` 信号标记数据列。低电平时，ShiftWindow 忽略 `colOut` 的值，自行填零。

---

## Phase 5：顶层集成与主 FSM

> TODO

---

## Phase 6：Chipyard 集成与 Verilator 构建

> TODO

---

## Phase 7：裸机 C 测试程序

> TODO

---

## Phase 8：性能报告与总结
