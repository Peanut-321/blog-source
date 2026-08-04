---
title: "TCP 与嵌入式网络学习笔记"
date: 2026-07-06 14:49:04
lang: zh-CN
tags:
  - TCP
  - CS144
  - 嵌入式网络
categories:
  - 学习笔记
---

## 0. 学习目标

- 理解 TCP 的基本模型与可靠传输机制
- 能通过抓包解释协议行为
- 理解 TCP 与 IP、ARP、以太网之间的关系
- 完成 CS144 ByteStream、Reassembler、TCP Receiver 和 TCP Sender

## 第一阶段：TCP 连接基础

### 1. TCP 基本模型

TCP 是一种面向连接的传输层协议。它在不可靠的 IP 网络上，为应用程序提供：

- 可靠传输
- 按顺序交付
- 全双工通信
- 字节流服务
- 差错检测

#### 1.1 全双工字节流

一条 TCP 连接包含两个独立的数据流：

```text
客户端 ───────────────> 服务器
         客户端发送方向

客户端 <─────────────── 服务器
         服务器发送方向
```

每个方向都有独立的：

- 序列号空间
- 发送缓冲区和接收缓冲区
- 关闭过程

客户端停止发送数据，不代表服务器也必须停止发送数据。

**字节流**

TCP 将应用层数据看作连续的字节，不保留消息边界。

发送端调用：

```c
send(socket, "ABC", 3, 0);
send(socket, "DEF", 3, 0);
```

接收端可能一次收到 `ABCDEF`，也可能分多次收到 `AB`、`CDE` 和 `F`。TCP 只保证最终能够收到 `ABCDEF`，但不保证一次 `send()` 对应一次 `recv()`。因此，嵌入式应用程序需要自行设计应用层消息分帧（Application-Layer Framing）：

```text
| 消息类型 | 数据长度 | 数据内容 |
```

#### 1.2 四元组

一条 TCP 连接由四元组唯一标识：

- 源 IP 地址
- 源端口
- 目的 IP 地址
- 目的端口

```text
192.168.1.10:50000 → 192.168.1.20:8000
└────── 端点 ──────┘   └────── 端点 ──────┘
```

每个 `IP 地址 + 端口` 组成一个通信端点。由于不同连接的四元组不同，因此同一个服务器端口可以同时连接多个客户端。

#### 1.3 TCP 报文段与首部

TCP 发送的基本单位称为 TCP 报文段，即 `TCP 报文段 = TCP 首部 + 有效载荷`。

常见的 TCP 标志位：

- `SYN`：同步序列号、建立连接
- `ACK`：确认号有效
- `FIN`：关闭自己的发送方向
- `RST`：立即重置连接
- `PSH`：提示尽快将数据交付应用程序
- `URG`：存在紧急数据

#### 1.4 序列号与确认号

**序列号 `seq`**

表示当前 TCP 报文段中第一个有效载荷字节在发送字节流中的编号。每个发送方向都有独立的序列号空间，因此 `seq` 描述当前发送者的数据。

**确认号 `ack`**

表示接收方下一次期待收到的字节序号，因此 `ack` 确认相反方向的数据。

### 2. 连接管理

#### 2.1 三次握手

三次握手用于建立连接，主要完成：

- 确认两个通信方向都能够工作
- 交换初始序列号（Initial Sequence Number，ISN）
- 协商 MSS、Window Scale、SACK 等 TCP 选项

假设客户端的 `ISN = 200`，服务器的 `ISN = 900`。

完整过程：

```text
客户端                                            服务器
CLOSED                                            LISTEN
   │                                                 │
   │  SYN, seq=200                                   │
   ├────────────────────────────────────────────────>│
   │                                                 │
SYN-SENT                                      SYN-RECEIVED
   │                                                 │
   │  SYN+ACK, seq=900, ack=201                      │
   │<────────────────────────────────────────────────┤
   │                                                 │
   │  ACK, seq=201, ack=901                          │
   ├────────────────────────────────────────────────>│
   │                                                 │
ESTABLISHED                                    ESTABLISHED
```

| 步骤 | 报文 | 作用 | 状态变化 |
|---|---|---|---|
| 1 | `SYN, seq=200` | 客户端公布 ISN | 客户端：`CLOSED → SYN-SENT` |
| 2 | `SYN+ACK, seq=900, ack=201` | 服务器公布 ISN，并确认客户端的 SYN | 服务器：`LISTEN → SYN-RECEIVED` |
| 3 | `ACK, seq=201, ack=901` | 客户端确认服务器的 SYN | 双方进入 `ESTABLISHED` |

**为什么需要三次握手？**

双方都要公布并确认各自的 ISN。第二次握手将服务器的 SYN 与对客户端 SYN 的 ACK 合并；第三次握手用于确认服务器的 ISN。SYN 占用一个序列号，因此双方第一个数据字节分别从 `201` 和 `901` 开始。

#### 2.2 四次挥手

四次挥手用于终止 TCP 连接。TCP 是全双工的，因此两个发送方向需要分别关闭；FIN 只关闭发送者自己的发送方向。

假设客户端主动关闭连接：

- 客户端是主动关闭方
- 服务器是被动关闭方

```text
客户端                                         服务器
ESTABLISHED                                    ESTABLISHED
   │                                              │
   │ FIN+ACK, seq=u, ack=v                        │
   ├─────────────────────────────────────────────>│
   │                                              │
   │ ACK, seq=v, ack=u+1                          │
   │<─────────────────────────────────────────────┤
   │                                              │
   │       服务器可以继续发送剩余数据                  │
   │<─────────────────────────────────────────────┤
   │                                              │
   │ FIN+ACK, seq=w, ack=u+1                      │
   │<─────────────────────────────────────────────┤
   │                                              │
   │ ACK, seq=u+1, ack=w+1                        │
   ├─────────────────────────────────────────────>│
   │                                              │
TIME_WAIT                                       CLOSED
```

其中 `u`、`v` 是双方当前的下一个序列号，`w` 是服务器发送完剩余数据后的序列号。

| 步骤 | 报文 | 作用 |
|---|---|---|
| 1 | `客户端 FIN, seq=u` | 关闭客户端 → 服务器方向 |
| 2 | `服务器 ACK, ack=u+1` | 确认客户端的 FIN |
| 3 | `服务器 FIN, seq=w` | 关闭服务器 → 客户端方向 |
| 4 | `客户端 ACK, ack=w+1` | 确认服务器的 FIN |

FIN 不是有效载荷，但会占用一个序列号；收到 FIN 的一方仍可继续发送数据。

**为什么 ACK 和 FIN 通常不能合并？**

TCP 协议栈收到 FIN 后立即回复 ACK，但本地应用程序可能尚未完成发送，因此自己的 FIN 通常稍后发送。若没有剩余数据，ACK 和 FIN 可以合并，抓包中可能只看到三个报文段。

#### 2.3 TCP 状态机

TCP 状态机记录连接的整个生命周期，本质上是有限状态机（Finite State Machine，FSM）。每个通信端点独立维护状态：

```text
客户端建连：CLOSED → SYN-SENT → ESTABLISHED
服务器建连：CLOSED → LISTEN → SYN-RECEIVED → ESTABLISHED

Active Close： ESTABLISHED → FIN-WAIT-1 → FIN-WAIT-2
              → TIME-WAIT → CLOSED
Passive Close：ESTABLISHED → CLOSE-WAIT → LAST-ACK → CLOSED
```

| 状态 | 含义 |
|---|---|
| `SYN-SENT` | 已发送 SYN，等待 SYN+ACK |
| `SYN-RECEIVED` | 已发送 SYN+ACK，等待最终 ACK |
| `FIN-WAIT-1` | 已发送 FIN，等待确认 |
| `FIN-WAIT-2` | FIN 已确认，等待对方 FIN |
| `CLOSE-WAIT` | 已收到 FIN，等待本地应用程序关闭 |
| `LAST-ACK` | 已发送 FIN，等待最终 ACK |
| `TIME-WAIT` | 已发送最终 ACK，等待 2MSL |

#### 2.4 TIME_WAIT

`TIME-WAIT`（常见工具显示为 `TIME_WAIT`）通常属于主动关闭方，用于：

1. 在最终 ACK 丢失时，接收重复 FIN 并再次回复 ACK。
2. 等待 `2MSL`，避免旧连接的延迟报文影响具有相同四元组的新连接。

MSL 是报文段最大生存时间（Maximum Segment Lifetime）。`TIME_WAIT` 是正常状态，不代表连接错误。

#### 2.5 CLOSE_WAIT

`CLOSE-WAIT`（常见工具显示为 `CLOSE_WAIT`）表示已收到对方 FIN，但本地应用程序尚未调用 `close()`。套接字中通常表现为：

```c
recv(socket, buffer, size, 0) == 0; // 对方的发送方向已经结束
```

应用程序随后调用 `close()`，发送 FIN 并进入 `LAST-ACK`。长期停留通常说明程序未处理 `recv() == 0`、忘记关闭套接字，或线程阻塞。

| 状态 | 通常属于 | 正在等待什么 |
|---|---|---|
| `TIME-WAIT` | 主动关闭方 | 等待 2MSL |
| `CLOSE-WAIT` | 被动关闭方 | 等待本地应用程序调用 `close()` |

#### 2.6 RST

RST 用于立即拒绝或中止整个连接。常见场景：

- SYN 到达未监听端口：客户端的 `connect()` 返回 `Connection refused`。
- 报文不属于任何现有连接。
- 应用程序主动异常中止。

收到 RST 后不再进行四次挥手，未完成的数据可能丢失，应用程序通常收到 `ECONNRESET`。

| 项目 | FIN | RST |
|---|---|---|
| 含义 | 正常结束发送 | 立即中止连接 |
| 影响范围 | 关闭一个发送方向 | 终止整个连接 |
| 剩余数据 | 尽量完成传输 | 不保证完成传输 |
| 应用程序常见结果 | `recv() == 0` | `ECONNRESET` |

#### 2.7 握手与挥手丢包

SYN 和 FIN 占用序列号，因此可被确认和重传。纯 ACK 不占用序列号，不会因自身定时器到期而单独重传。

| 丢失的报文 | 主要处理方式 |
|---|---|
| SYN | 客户端重传 SYN |
| SYN+ACK | 服务器重传 SYN+ACK |
| 第三次握手的 ACK | 服务器重传 SYN+ACK，客户端再次回复 ACK |
| FIN | FIN 的发送方重传 FIN |
| FIN 的 ACK | FIN 的发送方重传 FIN，对方再次回复 ACK |
| 最终 ACK | 被动关闭方重传 FIN，`TIME-WAIT` 方再次回复 ACK |

第三次握手的 ACK 丢失时，客户端已进入 `ESTABLISHED`，服务器仍处于 `SYN-RECEIVED`；服务器重传 SYN+ACK 后，客户端再次回复 ACK。若重传超过限制，TCP 协议栈会报告超时或连接错误。

### 实验 0：抓包观察连接建立与关闭

抓包时优先看三件事：方向、Flags、`seq/ack`。例如本机 loopback 上的三次握手：

```text
23:40:29.113682 IP 127.0.0.1.49559 > 127.0.0.1.8080: Flags [S], seq 3348290356, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 4199470301 ecr 0,sackOK,eol], length 0
23:40:29.113772 IP 127.0.0.1.8080 > 127.0.0.1.49559: Flags [S.], seq 3683137045, ack 3348290357, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 2175431545 ecr 4199470301,sackOK,eol], length 0
23:40:29.113793 IP 127.0.0.1.49559 > 127.0.0.1.8080: Flags [.], ack 1, win 6380, options [nop,nop,TS val 4199470301 ecr 2175431545], length 0
```

`>` 左边是发送方，右边是接收方；`length 0` 表示没有应用层数据。

| 报文 | 方向 | Flags | 关键字段 | 含义 |
|---|---|---|---|---|
| 第一次握手 | Client → Server | `SYN` | `seq=3348290356` | 客户端请求建立连接，并公布自己的 ISN |
| 第二次握手 | Server → Client | `SYN+ACK` | `seq=3683137045, ack=3348290357` | 服务器确认客户端的 SYN，同时公布自己的 ISN |
| 第三次握手 | Client → Server | `ACK` | `ack=1` | 客户端确认服务器的 SYN，连接建立完成 |

`[S.]` 表示 `SYN + ACK`。tcpdump 默认可能显示相对序列号，例如第三次握手里的 `ack 1` 逻辑上表示 `Server ISN + 1`。如果想显示绝对序列号，可以加 `-S`。

#### 抓包观察四次挥手

同一条连接关闭时，可以观察到四次挥手：

```text
14:23:35.098314 127.0.0.1.55409 > 127.0.0.1.8080: Flags [F.], seq 7, ack 1
14:23:35.098379 127.0.0.1.8080 > 127.0.0.1.55409: Flags [.], ack 8
14:23:35.098425 127.0.0.1.8080 > 127.0.0.1.55409: Flags [F.], seq 1, ack 8
14:23:35.098507 127.0.0.1.55409 > 127.0.0.1.8080: Flags [.], ack 2
```

| 报文 | 方向 | Flags | 关键字段 | 含义 |
|---|---|---|---|---|
| 第一次挥手 | Client → Server | `FIN+ACK` | `seq=7, ack=1` | 客户端关闭自己的发送方向 |
| 第二次挥手 | Server → Client | `ACK` | `ack=8` | 服务器确认客户端的 FIN |
| 第三次挥手 | Server → Client | `FIN+ACK` | `seq=1, ack=8` | 服务器关闭自己的发送方向 |
| 第四次挥手 | Client → Server | `ACK` | `ack=2` | 客户端确认服务器的 FIN |

四次挥手的核心规律是：谁发送 `FIN`，谁就关闭自己的发送方向；FIN 不携带应用层数据，但占用一个序列号，因此对方用 `ack = FIN 的 seq + 1` 确认。

## 第二阶段：字节流与乱序重组

### 3. 字节流与重组

前面已经说明：TCP 传输的是连续字节流，而不是一条条消息。本阶段关注接收端如何在有限缓冲区中保存字节，并把乱序、重复、重叠的片段重新整理成连续数据。

#### 3.1 ByteStream 抽象

ByteStream 可以理解为一个 FIFO 字节管道：

```text
写入端 write/push                  读取端 read/pop
        │                              ▲
        ▼                              │
       [ h ][ e ][ l ][ l ][ o ][ ... ]
```
ByteStream 的核心特征：

1. 字节按顺序进入/读出
2. 不保留消息边界
3. 容量有限
4. 可以结束输入，即到达 EOF

#### 3.2 发送缓冲区与接收缓冲区

TCP 不会直接把应用程序的数据放到网络中，而是通过发送缓冲区和接收缓冲区中转：

```text
应用程序                   网络
  │                        │
  │ send()                 │   收到 TCP segment
  ▼                        ▼
TCP 发送缓冲区          TCP 接收缓冲区
  │                        │
  │ 拆成 TCP segment        │  recv()
  ▼                        ▼
网络                    应用程序
```

应用程序将数据交给 TCP 发送缓冲区，然后 TCP 协议栈会决定：

- 什么时候发
- 分成几个 segment 发送
- 是否需要重传
- 什么时候可以从发送缓冲区删除

接收缓冲区同理。由于 TCP 是全双工的，两个方向的发送和接收互不影响。

1. `send()` 只是把数据交给 TCP 发送缓冲区，不代表对方已经收到。
2. `recv()` 是从 TCP 接收缓冲区读数据，不代表一次能读到完整消息。
3. TCP 通过缓冲区和接收窗口控制发送速度，避免接收方被撑爆。

#### 3.3 容量、反压（Backpressure）与 EOF

**容量 Capacity**

TCP 不能无限缓存数据。无论是操作系统中的 TCP，还是嵌入式中常见的 lwIP，发送端和接收端都只有有限的缓冲区。

在 ByteStream 中，需要区分两个概念：`capacity` 是缓冲区的最大容量，`remaining_capacity` 是当前还能继续写入的字节数。

例如缓冲区的最大容量是 8 字节，当前已经存入 `ABCDEF`：

```text
capacity = 8
buffer   = [ A B C D E F _ _ ]
remaining_capacity = 2
```

因此，容量不是一条 TCP 连接能够传输的数据总量，而是某一时刻缓冲区能够保存的待处理字节数。

**反压 Backpressure**

Backpressure 指的是：接收方处理不过来时，会反过来限制发送方继续发送。

例如客户端不断发送数据，而服务器应用程序没有及时调用 `recv()`，服务器的接收缓冲区就会逐渐变满。此时服务器会通过 TCP 接收窗口通知客户端：当前可用空间正在减少。

```text
Server 应用程序不读取
        ↓
Server receive buffer 变满
        ↓
Server advertised window 变小
        ↓
Client 可发送的数据量减少
        ↓
如果窗口变为 0，Client 暂停发送新的数据
```

这就是 TCP 的流量控制。它的目的不是提高网络速度，而是防止发送方耗尽接收方的缓冲区。对于嵌入式系统来说，这一点尤其重要，因为 MCU 的内存通常很小，缓冲区大小需要谨慎配置。

**EOF（End of File）**

EOF 表示这个方向的字节流已经结束，在 TCP 中通常对应收到对方的 FIN。

```c
int n = recv(sock, buf, sizeof(buf), 0); // n > 0：读到数据；n == 0：对方正常关闭发送方向，本端收到 FIN
```

在 ByteStream 抽象中，真正结束需要同时满足 Writer 已关闭且缓冲区已读空，即 `is_finished = EOF received && buffer empty`。

#### 3.4 乱序、重复和重叠数据

TCP 交给应用层的必须是连续、有序且无重复的字节流，但网络中的报文段可能乱序（out-of-order）、重复（duplicate）或重叠（overlap）。

例如原始数据是 `helloworld`，接收端却先收到 `[5,10) = world`。由于 `[0,5)` 尚未到达，`world` 只能暂存在 `pending_` 中；收到 `[0,5) = hello` 后，才能输出 `helloworld`。

重复数据应直接丢弃或裁剪。例如连续两次收到 `[0,5) = hello` 时，第二段不会产生新字节，不能重复输出。

重叠数据需要合并：

```text
[3,8)  = lowor
[5,10) = world
----------------
[3,10) = loworld
```

所以 Reassembler 的工作可以总结为：已经输出过的数据直接丢弃；暂时无法输出的乱序数据先缓存；重复或重叠的数据进行合并，避免重复保存。

#### 3.5 区间重组

实现时，每段 substring 可以抽象成半开区间 `[first_index, first_index + data.size())`。例如 `first_index = 3, data = "lowor"` 对应 `[3,8)`。

`pending_` 用来保存已经到达但还不能输出的片段：

```cpp
std::map<uint64_t, std::string> pending_; // key 是 first_index
```

例如，`pending_[5] = "world"` 对应区间 `[5,10)`。

处理重叠时，关键是先算出合并后的范围：

```cpp
const uint64_t merged_start = min( new_start, old_start );
const uint64_t merged_end = max( new_end, old_end );
string merged( merged_end - merged_start, '\0' );
```

然后用 offset 把原始 stream index 转换成 `merged` 里的下标：

```cpp
merged.replace( new_start - merged_start, new_data.size(), new_data );
merged.replace( old_start - merged_start, old_data.size(), old_data );
```

以前面这个例子为例：

```text
new: [3,8)  = lowor
old: [5,10) = world

merged_start = 3
"lowor" 从 merged[0] 开始放
"world" 从 merged[2] 开始放
```

最终得到 `loworld`。

`insert()` 的主流程如下：

```cpp
const uint64_t current_index = next_index();
const uint64_t end_index = first_index + data.size();

if ( end_index <= current_index ) {
  try_close();
  return;
}

if ( first_index < current_index ) {
  const uint64_t trim_len = current_index - first_index;
  data = data.substr( trim_len );
  first_index = current_index;
}

if ( first_index == next_index() ) {
  output_.writer().push( data );
  flush_pending();
  try_close();
  return;
}
```

这几行依次处理三种情况：整段已经输出则丢弃；前缀已经输出则裁剪 prefix；片段正好接上 `next_index` 则写入 ByteStream。

如果不能马上输出，就和 `pending_` 里的片段尝试合并：

```cpp
for ( auto it = pending_.begin(); it != pending_.end(); ) {
  const uint64_t old_start = it->first;
  const string old_data = it->second;
  const uint64_t old_end = old_start + old_data.size();

  const bool separated = new_end < old_start || old_end < new_start;
  if ( separated ) {
    ++it;
    continue;
  }

  const uint64_t merged_start = min( new_start, old_start );
  const uint64_t merged_end = max( new_end, old_end );
  string merged( merged_end - merged_start, '\0' );

  merged.replace( new_start - merged_start, new_data.size(), new_data );
  merged.replace( old_start - merged_start, old_data.size(), old_data );

  new_start = merged_start;
  new_end = merged_end;
  new_data = merged;

  it = pending_.erase( it );
}

pending_[new_start] = new_data;
```

需要注意的是：如果遍历过程中要删除 `map` 元素，应使用 iterator，而不是 range-for。`it = pending_.erase(it)` 会删除当前元素，并返回下一个有效位置。

## 第三阶段：TCP 接收端

### 4. 接收端可靠性

Checkpoint 2 开始进入 TCP Receiver。Reassembler 接收 `first_index + data`，而 TCP Receiver 接收包含 `seqno + SYN + payload + FIN` 的 TCP segment，再将 payload 交给 Reassembler。因此，这一阶段的核心问题是：如何把 TCP 报文中的 seqno 转换成 Reassembler 需要的 stream index？

#### 4.1 序列号

TCP 的序列号不是给报文编号，而是给字节编号。

例如发送 `hello`，TCP 看到的是 5 个连续字节：

```text
seq:  1001 1002 1003 1004 1005
data: h    e    l    l    o
```

因此，序列号记录的是字节在 TCP 字节流中的位置，而不是 `send()` 被调用的次数。这也与前面的字节流模型一致：TCP 不保留应用层消息边界。

#### 4.2 ISN 与 32 位序列号回绕

TCP 的序列号不是固定从 0 开始，而是从初始序列号 ISN 开始。例如第一次握手时，客户端可能发送 `SYN, seq = 1000`。

SYN 本身会占用一个序列号，所以第一个真正的数据字节不是 `1000`，而是 `1001`。

如果发送 `abc`：

```text
TCP seqno:
1000   1001   1002   1003
SYN    a      b      c
```

但是在 ByteStream/Reassembler 中，payload 的 stream index 从 0 开始：

```text
stream index:
       0      1      2
       a      b      c
```

因此，收到 SYN 后，需要将 payload 的位置从 TCP 序列号空间转换到 ByteStream 索引空间：`stream index = absolute seqno - 1`。这里的 `-1` 来自 SYN 占用的一个序列号。

真实 TCP 报文中的序列号是 32 位无符号数，范围为 `0 ~ 2^32 - 1`；超过最大值后会回绕，例如 `4294967295 -> 0 -> 1 -> 2`。

因此 CS144 中使用 `Wrap32` 处理两种编号之间的转换：

```cpp
Wrap32::wrap();    // 64-bit absolute seqno -> 32-bit TCP seqno
Wrap32::unwrap();  // 32-bit TCP seqno -> 64-bit absolute seqno
```

其中，32 位 TCP seqno 会回绕，而 64 位 absolute seqno 更适合程序内部计算。

#### 4.3 累计确认

ACK 表示接收端下一个期待的序列号。它不是在说“我收到了哪个字节”，而是在说：“我已经连续收到 ackno 之前的所有字节，下一个请从 ackno 开始发送。”

例如：

```text
ISN = 1000

seq 1000: SYN
seq 1001: a
seq 1002: b
seq 1003: c
```

如果接收端已经连续收到 `SYN + abc`，就会回复 `ack = 1004`，表示下一个期待的序列号是 1004。这就是累计确认（cumulative ACK）。

如果先收到 `seq = 1004` 的 `d`，但 `seq = 1001` 的 `a` 尚未到达，ACK 不能跳到 1005，只能保持 `ack = 1001`，因为 TCP 的 ACK 只能确认已经连续收到的前缀。

#### 4.4 接收窗口

接收窗口表示接收端当前还能接收多少数据，也称为 receive window、advertised window 或 `rwnd`。

例如接收缓冲区容量为 1000 字节，当前已经缓存 300 字节，则 `available_capacity = 700`。接收端可以返回 `window_size = 700`，表示从 ackno 开始，发送端最多还能发送 700 字节。

这就是 TCP 的流量控制，目的是防止发送端耗尽接收缓冲区，而不是提高传输速度。在嵌入式系统中，接收窗口尤为重要，因为 RAM 通常有限，TCP 缓冲区不可能无限增长。

#### 4.5 SYN、FIN 与序列号空间

Checkpoint 2 最容易出错的地方，是区分 TCP 序列号空间和 ByteStream 索引空间。

TCP 序列号的计算规则是：`SYN` 占 1 个序列号，payload 的每个字节占 1 个序列号，`FIN` 也占 1 个序列号；ByteStream 的 stream index 只计算 payload 字节。

例如：

```text
seq:    1000   1001   1002   1003   1004
thing:  SYN    a      b      c      FIN

stream index:
              0      1      2
              a      b      c
```

因此，`SYN/FIN` 占用 TCP seqno，但不占用 stream index；payload 则同时占用 TCP seqno 和 stream index。

可以用下面这张图总结：

```text
TCP seqno:      ISN   ISN+1   ISN+2   ISN+3   ISN+4
                SYN   data0   data1   data2   FIN

absolute seq:   0     1       2       3       4

stream index:         0       1       2
```

因此在 `TCPReceiver` 中：

```text
收到 SYN：记录 ISN
收到 payload：把 TCP seqno 转换成 stream index，交给 Reassembler
收到 FIN：标记 stream 结束
发送 ACK：ackno = 下一个想要的 TCP sequence number
发送 window：window_size = ByteStream 剩余容量
```

## 第四阶段：TCP 发送端

### 5. 发送端可靠性

发送端主要负责四件事：

1. 从应用层拿数据
2. 根据窗口大小决定能不能发
3. 记录已经发出但尚未确认的数据
4. 超时仍未收到 ACK 时重传

#### 5.1 发送缓冲区与未确认数据

发送端内部有两类数据：

- 还没发送的数据
- 已经发送但尚未确认的数据，即 outstanding data，保存在 outstanding segments 中

一个 outstanding segment 可能包含 `SYN`、`payload` 和 `FIN`。其中 `SYN`、`FIN` 各占用一个序列号，`payload` 占用 `payload.size()` 个序列号。

因此，发送端必须记录：

- `next_seqno`：下一个要发送的序列号
- `bytes_in_flight`：已经发送但尚未确认的序列号数量

虽然名称中带有 `bytes`，但在 CS144 中，`SYN` 和 `FIN` 也会使 `bytes_in_flight` 增加 1。

#### 5.2 滑动窗口

接收方通过滑动窗口告诉发送方当前还能接收多少数据。窗口大小来自 `TCPReceiverMessage.window_size`，发送方的可用窗口为 `available_window = window_size - bytes_in_flight`。

如果 ACK 已经确认 `[0, 101)` 且 `window_size = 5`，那么允许发送的范围是 `[101, 106)`。假设发送方已经发送 `[101, 104)`，则 `bytes_in_flight = 3`、`available_window = 2`，此时最多还能发送两个字节。

**ACK 到达后窗口如何滑动**

```text
发送方已经发送：
seq = 101, payload = "ABC"
seq = 104, payload = "DE"

接收方返回：
ack = 104
```

`ack = 104` 表示序列号小于 104 的内容均已收到，因此 `"ABC"` 可以从未确认队列中删除，`bytes_in_flight` 从 5 降为 2。释放出的窗口可以继续发送新数据，这就是窗口向右滑动。

**零窗口**

如果接收方返回 `window_size = 0`，表示接收缓冲区已经没有空间。但发送方不能永久停止，否则窗口重新打开后，发送方可能无法及时得知。

在 CS144 中，可以将零窗口暂时视为大小为 1 的有效窗口，从而发送一个探测报文：

```cpp
effective_window = window_size == 0 ? 1 : window_size;
```

#### 5.3 超时与重传

发送方发出报文后，会将其放入 outstanding queue，并启动重传计时器等待 ACK。

**1. ACK 按时到达**

如果 ACK 确认了新的数据：

- 删除已经完全确认的报文
- 减少 `bytes_in_flight`
- 将 RTO 恢复为初始值
- 如果还有未确认报文，重新启动计时器
- 如果没有未确认报文，停止计时器

**2. 等待 ACK 超时**

如果计时器达到 RTO，但最早的未确认报文仍未得到确认：

- 重传最早的未确认报文
- 不创建新的报文
- 不为它分配新的序列号
- 原报文继续留在未确认队列中

也就是说，哪个报文尚未确认，就使用原序列号重新发送该报文。

#### 5.4 RTT 与 RTO

- RTT：报文从发送方到达接收方，再由 ACK 返回发送方所经历的时间，即 `收到对应 ACK 的时间 - 发送报文的时间`
- RTO：发送方在认为报文可能丢失之前等待的时间

RTO 不能简单等于某一次 RTT：设置过短，网络暂时延迟也会触发不必要的重传；设置过长，报文真正丢失后又需要等待很久才能重传。

CS144 Checkpoint 3 会直接提供初始 RTO，当前重点是正确维护计时器，而不是实现完整的 RTT 估计算法。

#### 5.5 指数退避（Exponential Backoff）

为了避免网络已经拥塞时仍然频繁重传，每次超时重传后，发送方会将 RTO 加倍：`初始 RTO -> 2 × RTO -> 4 × RTO -> 8 × RTO`。

只有 ACK 的确认范围真正向前移动，也就是确认了新的数据，才会将 RTO 恢复为初始值，并清零连续重传次数。

在 CS144 中，零窗口探测超时后仍然可以重传，但不进行指数退避，也不增加连续重传次数，因为此时未得到确认不一定是网络拥塞。

## 第五阶段：TCP 流量与性能

这一阶段以理解和面试为主，不实现拥塞控制算法。

### 6. TCP 性能
TCP在保证可靠的基础上，一次允许多少数据处于网络中，以及如何避免把接收端或网络压垮
```text
实际发送窗口 = min(rwnd, cwnd)
```
#### 6.1 rwnd 与 cwnd
TCP发送速度受到两个窗口限制：rwnd(接收窗口——保护接收端)，cwnd(拥塞窗口——保护网络)

- rwnd--接收窗口：
接收方告诉发送方：“我的接收缓冲区还剩多少空间”。
在checkpoint2中`message.window_size = writer().available_capacity;`，checkpoint3中的发送端收到后保存`receiver_window_size_ = msg.window_size`

所以rwnd解决的是flow control（流量控制）：防止发送方发的太快，撑爆接收方缓冲区。

- cwnd--拥塞窗口：（发送端内部状态，不是接收方提供的，不放进TCP首部）
发送端自己维护拥塞窗口，根据当前网络状况，认为最多允许多少数据同时处于网络中。

#### 6.2 慢启动、拥塞避免与快速重传概览
- Slow Start：慢启动
  发送端从较小的cwnd开始，每当ACK返回，就增大cwnd，快速探测网络容量
```
第一个RTT：cwnd = 1MSS
第二个RTT：cwnd = 2MSS
...（1 -> 2 -> 4 -> 8 -> 16）
```

- Congestion Avoidance: 拥塞避免
当cwnd达到ssthresh(慢启动阈值)后，不再指数增长，改为较慢的线性增长。

- 发生超时
如果一直没有收到ACK，最终触发RTO，发送端认为网络可能出现了严重拥塞

- 快速重传
接收端连续收到后面的报文（说明链路没断，一切正常），但是中间缺少一个报文，会返回相同的ACK（其中一个报文走丢了）
```
发送：1  2  3  4  5
接收：1  2  ×  4  5

ACK： 2  3  3  3  3
```
多个重复ACK表示：网络还在传输数据，但是中间某个包丢失了 
->TCP在收到三个Duplicate ACK后，不需要等待RTO，直接重传缺失报文。

#### 6.3 带宽时延积与吞吐率
- Bandwidth-Delay Product,(BDP)--带宽时延积
BDP = 带宽xRTT
表示要让一条网络链路始终保持繁忙，网络中需要同时存在多少未确认数据。

TCP 吞吐率 ≈ min(链路带宽, rwnd / RTT, cwnd / RTT)

## 第六阶段：网络接口与 IP

### 7. 链路层与网络层

TCP Segment 不能直接在以太网上传输。发送数据时，协议栈会逐层添加首部，最终将数据交给网卡。

**发送流程：**

```text
应用数据
   ↓
TCP Segment
   ↓
IPv4 Datagram
   ↓
确定下一跳
   ↓
ARP 查询下一跳的 MAC 地址
   ↓
Ethernet Frame
   ↓
网卡驱动 → DMA → MAC → PHY → 网络
```

**下一跳的判断：**

- 目标 IP 与本机位于同一子网：下一跳是目标设备。
- 目标 IP 与本机位于不同子网：下一跳是默认网关。

IP 地址用于确定数据的最终目的地，MAC 地址用于完成当前一跳的传输。

**嵌入式网络中的常见故障：**

- Link 状态未更新；
- ARP Cache 异常；
- IP 地址与子网掩码不匹配；
- 默认网关配置错误；
- MTU 或 MSS 设置不当；
- 网卡驱动没有把收到的数据交给协议栈。


## 阶段总结

至此，本文完成了对 TCP 基本模型、连接管理、可靠传输、流量控制与发送端重传机制的梳理，并结合 CS144 Checkpoint 0–3 实现了 ByteStream、Reassembler、TCP Receiver 和 TCP Sender。

受当前时间安排限制，网络接口、UDP 与 lwIP 暂不展开。后续在嵌入式项目中实际使用这些内容时，再继续学习和补充记录。
