---
title: "A RISC-V Based RoCC Convolution Accelerator"
date: 2026-05-09 22:30:00
lang: en
tags:
  - RISC-V
  - Chisel
  - RoCC
categories:
  - MyConvAccel
---

- Repository: [MyConvAccel](https://github.com/Peanut-321/MyConvAccel)

## Phase 0: Project Planning & Protocol Design

Before writing any code, we need to settle a few fundamental questions.

### Protocol Design: RoCC Instruction Encoding

The RoCC protocol is straightforward: the CPU passes register values to the accelerator via custom instructions, and the accelerator responds with a result.

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

Our accelerator uses a three-step flow: **set addresses → trigger → poll**. First, the CPU tells the accelerator where the input matrix, kernel, and output reside in memory. Then it fires a start signal. Finally, it polls a status register to know when the computation is done.

| funct7 | Instruction | rs1 | Description |
|--------|------|-----|------|
| 0 | SET_ADDR_IN | base addr | Base address of input matrix |
| 1 | SET_ADDR_KER | base addr | Base address of kernel |
| 2 | SET_ADDR_OUT | base addr | Base address of output matrix |
| 3 | START_ACCEL | — | Non-blocking start |
| 4 | POLL_STATUS | — | Read status register to rd |

The status register packs four bits for software polling:

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | busy | Accelerator is computing |
| 1 | done | Computation complete |
| 2 | overflow | Accumulator overflow |
| 3 | addr_err | Address check failed |

### Design Rationale

1. **Why split addresses into three separate instructions?**

    A single RoCC instruction only carries two source operands (`rs1` and `rs2`) — not enough to pass three base addresses at once. The alternative is passing a struct pointer, but that forces the accelerator to issue its own DMA read just to fetch the configuration, adding complexity and latency. Three independent instructions pass one address each, keeping the hardware interface as simple as possible.

2. **Why non-blocking START + polling instead of blocking for the result?**

    Non-blocking lets the CPU issue `START_ACCEL` and immediately switch to other work, so cycles aren't wasted. Polling is preferred over interrupts because a 5×5 convolution completes in a short, predictable window — the overhead of an interrupt controller and context switch would outweigh the benefit.

3. **Why continuous funct7 encoding instead of sparse assignment?**

    A single `funct7 <= 4` check covers all valid instructions, keeping the hardware decoder minimal. Sparse encoding would add combinatorial logic with no upside — there is no security or compatibility requirement that demands non-contiguous opcodes.

### Key Design Decisions

There are several questions that need clear answers before starting this project.

1. **Is the convolution kernel size fixed or configurable?**

    Supporting variable kernel sizes increases control logic complexity significantly. In the end, we chose a fixed 5×5 hardware datapath. Kernels smaller than 5×5 are pre-zero-padded by software.

2. **How to handle boundary pixels?**

    Zero-padding is the simplest approach — no special edge-case logic is needed in the sliding-window FSM.

3. **Fixed-point precision trade-off.**

    - **Why not floating-point?** IEEE 754 multipliers are expensive — they consume significant area and add pipeline stages. A Q8.8 fixed-point multiplier is just a 16-bit × 16-bit integer multiplier, producing a result in a single cycle with no extra hardware.
    
    - **Why not plain integer?** Convolution weights are naturally fractional (e.g. 0.125, -0.5). Plain integers can't represent them. Q8.8 splits 16 bits evenly: 8 bits for the integer part (±128 range), 8 bits for the fractional part (1/256 precision).
    
    - **Why a 32-bit accumulator?** A 5×5 convolution sums 25 Q8.8 products. Each 16-bit × 16-bit multiply produces up to 32 bits. A 32-bit accumulator holds the full sum without overflow, and no wider.

4. **Why alignment constraints?**

    TileLink DMA works best when addresses are word-aligned. An unaligned address forces the bus to split a single transfer into multiple chunks, and the hardware has to shift and stitch bytes back together. We skip all of that — the accelerator just checks the address on every `SET_ADDR_*` and rejects unaligned ones. Software's problem, not ours.
    

| Parameter | Value |
|------|-----|
| Input Matrix | 32×32 |
| Max Kernel | 5×5 |
| Data Format | Q8.8 Fixed Point（16-bit） |
| Accumulator | 32-bit |
| Output | Same size, zero-padding |
| Performance Target | <2500 cycles, ≥40× speedup |

---

## Phase 1: Architecture Overview

### The Problem: Running Convolution on a CPU

A 5×5 convolution over a 32×32 matrix means sliding a 5×5 window across 1024 output positions. Each output pixel requires 25 multiplies and 24 additions — 25,000 multiply-accumulates in total. On a general-purpose CPU, the arithmetic itself is not the bottleneck. The real bottleneck is the sliding. Every slide requires computing address offsets, updating loop counters, loading pixels and weights from memory, and storing results back. Most instructions are spent on loop control and address bookkeeping, not math.

A rough estimate: on a simple in-order RISC-V core, each output pixel costs ~100–150 cycles (two loads, one store, ~50 arithmetic ops, plus loop branches). Across 1024 pixels, that adds up to **100,000–150,000 cycles**. Our accelerator does the same job in **2428 cycles** — roughly a 50× speedup.

### The Fix: Offload to an Accelerator

The CPU is not built for sliding-window number crunching — so we hand that job to a RoCC accelerator. The accelerator does three things: fetch data from SRAM, run the convolution, and write results back. The CPU's role shrinks to three steps: set addresses, issue START, and poll.

What does the accelerator look like inside?

The naive approach is a serial pipeline: DMA load → MAC compute → DMA store. Simple, but wasteful. The DMA waits for the MAC. The MAC waits for the DMA. Only one piece of hardware is active at any moment.

What we actually built overlaps all three. An inputQueue preloads the entire matrix, freeing the DMA to stream results back while the ConvEngine is still crunching pixels. The ConvEngine runs a 6-stage pipeline, producing one result per cycle once full. A storeQueue absorbs results and applies backpressure upstream when it fills. The result: load, compute, and store all run concurrently — 2428 cycles end to end.

### Module Map

| Module | Role | Phase |
|--------|------|-------|
| ConvControl | RoCC decode + 5-state master FSM + status register | 2 |
| ConvDMA | 7-state DMA engine over TileLink | 3 |
| LineBuffer + ConvEngine | Sliding window + MAC pipeline, 6-stage compute datapath | 4 |
| InputQueue + StoreQueue | Elastic buffers with backpressure | 5 |

The following phases walk through each module in detail.

---

## Phase 2: ConvControl — Instruction Decode & FSM

Phase 1 drew the architecture diagram. The top line — RoCC control — is what Phase 2 is about. The CPU issues a custom instruction carrying `funct7`. ConvControl decodes it, executes it, and responds via `rd`.

### Interface

ConvControl talks to the CPU through five signals and a valid-ready handshake: `instrCmd.valid` / `funct7` / `rs1` / `rd` arrive from the CPU, `instrReady` goes back. When both `valid` and `ready` are high on the same cycle, the instruction fires.

### Instruction Decode

Decoding is five comparators — `funct7 === 0.U` through `funct7 === 4.U`. No priority encoder, no lookup table. The continuous encoding from Phase 0 pays off here: `funct7` is literally the instruction number.

Not every instruction is welcome at every moment. The rules:

- **SET (0–2):** blocked in sBusy. Changing addresses mid-computation would corrupt the run.
- **START (3):** only accepted in sIdle or sDone. You cannot start an already-running accelerator.
- **POLL (4):** always accepted. It is a pure read — it cannot interfere with anything.

### SET_ADDR: Storing Three Base Addresses 

Depending on `funct7`, `rs1` is written into `addrIn`, `addrKer`, or `addrOut`. If the accelerator is in `sError`, any SET clears it back to `sIdle`.

### START_ACCEL: Validate and Fire

1. **Address non-zero check**

    If any base address (`addrIn`, `addrKer`, `addrOut`) is zero, the state goes to sError.

2. **Address alignment check**

    Input and output matrices require 8-byte alignment (`addrIn(2,0) === 0.U`). The kernel requires 2-byte alignment (`addrKer(0) === 0.U`). The DMA bus is 64 bits wide — each transfer moves 4 pixels. The kernel has only 25 coefficients, so 2-byte alignment is sufficient. If data is misaligned, the hardware must perform extra byte shifting and splicing.
    
If all checks pass → sIdle → sBusy, and actual computation begins. If any check fails → sError, and the `addr_err` bit in the status register goes high.

### FSM: Four States

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

- **sIdle:** reset default. Waits for START.
- **sBusy:** only accepts POLL. Counts down to zero, then raises `done` and moves to sDone.
- **sDone:** holds `done = 1`. CPU can re-START (→ sBusy) or SET to reconfigure (→ sIdle).
- **sError:** holds `addrErr = 1`. Only SET pulls it back to sIdle — no direct path to sBusy.

---

## Phase 3: ConvDMA — TileLink DMA Engine

The convolution engine constantly reads pixels and weights from SRAM, then writes results back. A dedicated DMA engine manages these transfers. Phase 3 starts with a strictly serial DMA — one request in flight at a time. This keeps the FSM simple and makes the initial test cycle-accurate. Once verified, we will add pipelining so multiple requests can be issued without waiting for the prior response.

### DMA Interface

Three pure Bundles connect the DMA to the L1 data cache. No logic, just signals — valid/ready handshakes come from `Decoupled`.

- **SimpleMemReq** — DMA's request to L1. Carries 64-bit address, 64-bit write data, byte mask, a read/write flag, and a 4-bit tag. Tag is 0 in the serial DMA; reserved so a pipelined version can match out-of-order responses.
- **SimpleMemResp** — L1's reply. Returns 64-bit read data and echoes the request's tag.
- **SimpleMemIO** — bundles `req` (DMA→L1) and `resp` (L1→DMA, `Flipped`) into one port.

```
      ConvDMA                  SimpleMemIO            L1 data cache
    ┌─────────┐              ┌─────────────┐          ┌──────────┐
    │         │── req ──────►│ req (output) │────────►│          │
    │  FSM    │              │             │          │  L1 D$   │
    │         │◄─ resp ──────│ resp (input) │◄────────│          │
    └─────────┘              └─────────────┘          └──────────┘
```

### Serial FSM

ConvDMA moves data between SRAM and the compute unit through two paths:

- **Load path** (sIdle → sIssue → sWaitResp → sUnpack → loop): reads 64-bit words from memory, unpacks each into four 16-bit elements, pushes them into `elemQueue`.
- **Store path** (sIdle → sGather → sIssue → loop): collects four 16-bit elements from the compute unit, packs them into a 64-bit word, writes it to memory.

Both paths share `sIssue`, branching on `opReg`. At most one request is in flight at any time — issue one, wait for the response, finish unpacking/packing, then issue the next.

```
Load path:   sIdle → sIssue → sWaitResp → sUnpack(×4) ─┐
              ▲                                        │
              └────────────────────────────────────────┘

Store path:  sIdle → sGather(×4) → sIssue ─┐
             ▲                             │
             └─────────────────────────────┘

Error path:  sIdle ──► sError  (SET addr ⇒ sIdle)
```

| State | Responsibility | Path |
|-------|----------------|------|
| sIdle | Accept command, latch baseAddr and length, check alignment | common |
| sIssue | Fire mem.req (read or write, depending on opReg) | shared |
| sWaitResp | Wait for mem.resp.valid, latch response data | load |
| sUnpack | 4 cycles: slice 64-bit word into 4 × 16-bit elements, push into elemQueue | load |
| sGather | 4 cycles: collect 4 × 16-bit elements from elemQueue, pack into 64-bit word | store |
| sDone | Transfer complete, waiting for upper layer to consume result | common |
| sError | Address misaligned, waiting for a valid address to be written | common |

### Bottleneck: 6 cycles/word

Take the load path. One word goes through sIssue → sWaitResp → sUnpack×4 → back to sIssue — exactly 6 cycles. Memory sits idle for 4 of those 6 cycles during unpack. The bottleneck is not memory latency, it's the FSM refusing to overlap issue with unpack.

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

256 words × 6 + 1 cycle overhead = **1537 cycles**. The next section fixes this.

### Pipelined DMA

This stage fixes the bottleneck of the serial FSM. The core idea is to decouple issue and unpack into two concurrent hardware processes that overlap in time.

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

A response FIFO absorbs the rate mismatch: the issue engine fires at 1 word/cycle (up to the inflight limit), responses drop into the FIFO automatically, and the unpack engine drains the FIFO at 4 cycles/word. Neither blocks the other. 1537 cycles → ~1033 cycles, a 33% improvement. Details of the step-by-step implementation are covered in a follow-up post.

#### Changes from Serial to Pipelined

**1. Register → Queue**

The serial DMA uses a single `respWord` register to latch one response at a time — sufficient when only one request is in flight. The pipelined version replaces it with a response FIFO that buffers multiple responses. Data drops into the FIFO automatically; the FSM only reads from the FIFO when it needs data. This decouples response collection from the FSM.

**2. sWaitResp disappears**

`sWaitResp` no longer waits for `mem.resp.valid` or latches response data. Its only remaining job is to wait until the FIFO has data to pop.

**3. inflightCount — credit-based flow control**

The issue engine fires faster (1 word/cycle) than the unpack engine consumes (1 element/cycle → 0.25 word/cycle). Without a cap, the FIFO overflows. `inflightCount` acts as a credit account: issue spends a credit per request, unpack returns a credit per word consumed. `inflightMax = 4` caps the window at 4 in-flight words, matching the unpack engine's digestion rate.

**4. sLoadActive — concurrent FSM (core change)**

The three serial states `sIssue`, `sWaitResp`, and `sUnpack` merge into a single `sLoadActive` state. Inside, two independent `when` blocks run concurrently — the issue engine fires requests up to `inflightMax`, and the unpack engine drains the FIFO.

A naive ping-pong FSM (sIssue → sUnpack → sIssue → ...) cannot achieve overlap: issue and unpack never share a cycle, yielding 5 cycles/word. True concurrency requires both engines active in the same state.

In `sLoadActive`, the issue engine checks two conditions each cycle — are there bytes left to send, and is the inflight window still open? If both hold, it fires a 64-bit read request and advances the address. Meanwhile, the unpack engine checks whether the respFIFO has data, slices the 64-bit word into 4 × 16-bit elements, and pushes them into elemQueue one by one. Neither engine waits for the other — both are active in the same cycle.

Below is a trace of the first nine cycles, showing the burst start, steady state, and the moment issue and unpack fire together:

```
 cyc |      issue engine       |      unpack engine      | inflight | what happened
   0 | fire read → addr 0x1000 |                         |    1     | burst starts
   1 | fire read → addr 0x1008 |                         |    2     |
   2 | fire read → addr 0x1010 | deq elem[0] from word 0 |    3     | unpack begins, both engines active
   3 | fire read → addr 0x1018 | deq elem[1]             |    4     | inflight window full
   4 |           —              | deq elem[2]             |    4     | issue blocked, unpack continues alone
   5 | fire read → addr 0x1020 | deq elem[3] (last)      |    4     | both fire: +1 from issue, −1 from unpack, net zero
   6 |           —              | deq elem[4] from word 1 |    4     | inflight full again, issue waits
   7 |           —              | deq elem[5]             |    4     |
   8 |           —              | deq elem[6]             |    4     |
   9 | fire read → addr 0x1028 | deq elem[7] (last)      |    4     | word 1 done, credit returned, issue fires
```

Key observations from the trace:

- **Cycles 0–3:** The issue engine fires four read requests back-to-back before unpack even starts. By cycle 3, inflight hits the cap of 4.
- **Cycles 5 and 9:** These are the moments where both engines advance in the same cycle. Issue fires a new request (+1) while unpack simultaneously finishes the last element of a word (−1). The net change to `inflightCount` is zero — a credit is spent and repaid in the same cycle.
- **Cycles 4 and 6–8:** The issue engine is blocked because `inflightCount == inflightMax`. It waits for unpack to finish a word and return a credit.

The only conflict is `inflightCount`. Both engines update it — issue +1, unpack −1. Splitting them into two `when` blocks would break under Chisel's last-connect-wins rule: when both fire in the same cycle, only the later assignment takes effect, silently dropping the other update. The fix is a single expression: `inflightCount + issueFired − unpackWordDone`. Both boolean events feed one arithmetic operation — +1 and −1 cancel cleanly when they coincide, and neither update is lost.

## Phase 4: Compute Datapath — LineBuffer & ConvEngine

> TODO

## Phase 5: Top-Level Integration & Master FSM

> TODO

---

## Phase 6: Chipyard Integration & Verilator Build

> TODO

---

## Phase 7: Bare-Metal C Test Program

> TODO

---

## Phase 8: Performance Report & Summary

> TODO
