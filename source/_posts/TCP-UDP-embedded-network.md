---
title: "TCP、UDP 与嵌入式网络学习笔记"
date: 2026-07-06 14:49:04
lang: zh-CN
tags:
  - TCP
  - UDP
  - 嵌入式网络
categories:
  - 学习笔记
---

## 0. 学习目标

- 理解 TCP 和 UDP 的工作原理
- 能通过抓包解释协议行为
- 能编写 TCP/UDP 客户端和服务器
- 理解 lwIP 在嵌入式系统中的作用
- 完成 CS144 ByteStream、Reassembler、TCP Receiver 和 TCP Sender
- 实现一个面向嵌入式场景的 TCP/UDP 通信项目

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

接收端可能一次收到：

```text
ABCDEF
```
也可能多次收到：

```text
AB
CDE
F
```

TCP 只保证最终能够收到 `ABCDEF`，但不保证一次 `send()` 对应一次 `recv()`。因此，嵌入式应用程序需要自行设计应用层消息分帧（Application-Layer Framing）：

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

TCP 发送的基本单位称为 TCP 报文段：

```text
TCP 报文段 = TCP 首部 + 有效载荷
```

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

假设：

```text
客户端 ISN = 200
服务器 ISN = 900
```

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

前面已经说明：TCP 传输的是连续 byte stream，而不是一条条 message。本阶段关注接收端如何在有限 buffer 中保存字节，并把乱序、重复、重叠的片段重新整理成连续数据。

#### 3.1 ByteStream 抽象
ByteStream可以理解成一个FIFO的字节管道：
```
写入端 write/push                  读取端 read/pop
        │                              ▲
        ▼                              │
       [ h ][ e ][ l ][ l ][ o ][ ... ]
```
因此ByteStream的核心特征：
1. 字节按顺序进入/读出
2. 不保留消息边界
3. 容量有限
4. 可以结束输入--EOF

#### 3.2 发送缓冲区与接收缓冲区
TCP不是直接将数据扔到网线上，其中间有缓冲区：
```
应用程序                   网络
  │                        │                 
  │ send()                 │   收到 TCP segment
  ▼                        ▼
TCP 发送缓冲区          TCP 接收缓冲区
  │                        │  
  │ 拆成 TCP segment        │  recv（）
  ▼                        ▼
网络                    应用程序 
```

应用程序将数据交给 TCP 发送缓冲区，然后 TCP 协议栈会决定：

- 什么时候发
- 分几个segment发
- 是否需要重传
- 什么时候可以从发送缓冲区删除

接收缓冲区同理。由于 TCP 是全双工的，两个方向的发送和接收互不影响。

1. send() 只是把数据交给 TCP 发送缓冲区，不代表对方已经收到。

2. recv() 是从 TCP 接收缓冲区读数据，不代表一次能读到完整消息。

3. TCP 通过缓冲区和接收窗口控制发送速度，避免接收方被撑爆。

#### 3.3 容量、反压（Backpressure）与 EOF

**容量 Capacity**

TCP 不能无限缓存数据。无论是操作系统里的 TCP，还是嵌入式中常见的 lwIP，发送端和接收端都有有限的 buffer。

在 ByteStream 中，需要区分两个概念：

```text
capacity            = buffer 的最大容量
remaining_capacity  = 当前还能继续写入多少字节
```

例如 buffer 的最大容量是 8 字节，当前已经存了 `ABCDEF`：

```text
capacity = 8
buffer   = [ A B C D E F _ _ ]
remaining_capacity = 2
```

因此，容量不是指一条 TCP 连接总共最多能传多少数据，而是指某个时刻 buffer 能缓存多少还没被处理的字节。

**反压 Backpressure**

Backpressure 指的是：接收方处理不过来时，会反过来限制发送方继续发送。

例如 Client 不断发送数据，而 Server 应用程序没有及时调用 `recv()`，那么 Server 的接收缓冲区会逐渐变满。此时 Server 会通过 TCP 接收窗口告诉 Client：我现在可接收的空间变少了。

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

这就是 TCP 的流量控制。它的目的不是让网络更快，而是防止发送方把接收方的 buffer 撑爆。对于嵌入式系统来说，这一点尤其重要，因为 MCU 的内存通常很小，buffer 大小需要谨慎配置。

**EOF（End of File）**

EOF 表示这个方向的字节流已经结束，在 TCP 中通常对应收到对方的 FIN。

```c
int n = recv(sock, buf, sizeof(buf), 0);
```

```text
n > 0  : 读到了数据
n == 0 : 对方正常关闭发送方向，本端收到 FIN
```

在 ByteStream 抽象中，真正 finished 需要同时满足：writer 已关闭，且 buffer 已读空。

```c
is_finished = EOF received && buffer empty;
```

#### 3.4 乱序、重复和重叠数据

TCP 交给应用层的必须是连续、有序、无重复的字节流。但网络里来的 segment 不一定这么干净，接收端会遇到三种情况：

```text
out-of-order：后面的数据先到
duplicate：同一段数据重复到达
overlap：两段数据有部分重叠
```

例如原始数据是 `helloworld`，但是先收到：

```text
[5,10) = world
```

这时不能把 `world` 交给应用层，因为前面的 `[0,5)` 还没到。它只能先放进 pending buffer。等后面收到：

```text
[0,5) = hello
```

才能输出：

```text
helloworld
```

重复数据则应该直接丢弃或裁剪：

```text
[0,5) = hello
[0,5) = hello
```

第二段不会产生新的字节，不能重复输出。

重叠数据需要合并：

```text
[3,8)  = lowor
[5,10) = world
----------------
[3,10) = loworld
```

所以 Reassembler 的工作可以总结为：

```text
1. 已经输出过的数据直接丢弃
2. 乱序但还没法输出的数据先缓存
3. 重复或重叠的数据要合并，避免重复保存
```

#### 3.5 区间重组

实现时，每段 substring 可以抽象成一个 half-open interval：

```text
[first_index, first_index + data.size())
```

例如：

```text
first_index = 3, data = "lowor"  =>  [3,8)
```

`pending_` 用来保存已经到达但还不能输出的片段：

```cpp
std::map<uint64_t, std::string> pending_; // key 是 first_index
```

例如：

```text
pending_[5] = "world"  =>  [5,10)
```

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

这几行对应三个判断：

```text
1. 整段已经输出过：丢弃
2. 前面一部分已经输出过：裁剪 prefix
3. 正好接上 next_index：写入 ByteStream
```

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

#### 3.6 CS144 Checkpoint 0：Networking Warmup 与 ByteStream

Checkpoint 0 实现的是 `ByteStream`，它可以看成一个有容量限制的 FIFO 字节管道：

```text
Writer -> buffer -> Reader
```

实现中主要维护以下状态变量：

```text
capacity_  : buffer 最大容量
buffer_    : 当前还没被读取的数据
pushed_    : 历史累计写入字节数
popped_    : 历史累计读取字节数
closed_    : writer 是否关闭
```

写入时不能超过剩余容量：

```cpp
void Writer::push( string data )
{
  if ( closed_ ) {
    return;
  }

  uint64_t writable = min( static_cast<uint64_t>( data.size() ), available_capacity() );
  buffer_.append( data.substr( 0, writable ) );
  pushed_ += writable;
}
```

读取时也不能超过当前 buffer 中已有的数据：

```cpp
void Reader::pop( uint64_t len )
{
  uint64_t popable = min( len, static_cast<uint64_t>( buffer_.size() ) );
  buffer_.erase( 0, popable );
  popped_ += popable;
}
```

这里需要区分 `close()` 和 `is_finished()`：

```cpp
bool Reader::is_finished() const
{
  return closed_ && buffer_.empty();
}
```

`close()` 只表示 writer 不再写入；只有 buffer 也被读空，整个 stream 才算 finished。这个逻辑和 TCP 里的 FIN 很像：收到 FIN 以后，之前已经收到的数据仍然要交给应用层读取。

#### 3.7 CS144 Checkpoint 1：Reassembler

Checkpoint 1 实现的是 `Reassembler`。它的接口是：

```cpp
void Reassembler::insert( uint64_t first_index, string data, bool is_last_substring )
```

含义是：

```text
把 data 插入到 stream 的 first_index 位置
is_last_substring 表示这段是否是整个 stream 的最后一段
```

例如：

```cpp
insert(5, "world", true);
insert(0, "hello", false);
```

虽然 `world` 先到，但因为 stream index 0 开始的数据还没到，所以它必须先被缓存。等 `hello` 到达后，才能输出：

```text
helloworld
```

`insert()` 中定义了几个小工具函数，用来简化主流程。

第一个是 `next_index()`，表示下一个想要写入 ByteStream 的 index：

```cpp
const auto next_index = [&] { return output_.writer().bytes_pushed(); };
```

第二个是 `flush_pending()`，每次写入一段连续数据后，都检查 pending 中有没有刚好能接上的片段：

```cpp
const auto flush_pending = [&] {
  while ( true ) {
    auto it = pending_.find( next_index() );
    if ( it == pending_.end() ) {
      break;
    }

    output_.writer().push( it->second );
    pending_.erase( it );
  }
};
```

所以 Reassembler 的核心状态是：

```text
next_index : 下一个可以写入 ByteStream 的 index
pending_   : 已经收到但暂时不能输出的乱序片段
```

最后是 EOF 处理：

```cpp
if ( is_last_substring ) {
  eof_seen_ = true;
  eof_index_ = first_index + data.size();
}
```

注意：最后一段先到，不代表可以马上关闭。只有连续输出到 `eof_index_`，才可以 close：

```cpp
const auto try_close = [&] {
  if ( eof_seen_ && eof_index_ == next_index() ) {
    output_.writer().close();
  }
};
```

Checkpoint 1 完成后，可以更清楚地看到：TCP 接收端不是“收到什么就交给应用层什么”，而是必须先把网络中可能乱序、重复、重叠的数据整理成干净的连续字节流。

```text
network segments -> Reassembler -> ByteStream -> application
```

## 第三阶段：TCP 接收端

### 4. 接收端可靠性

Checkpoint 2 开始进入 TCP Receiver。前面 Reassembler 接收的是：

```text
first_index + data -> Reassembler
```

而 TCP Receiver 接收的是 TCP segment：

```text
seqno + SYN + payload + FIN -> TCPReceiver -> Reassembler
```

因此这一阶段的核心问题是：

```text
TCP 报文中的 seqno，如何转换成 Reassembler 需要的 stream index？
```

#### 4.1 序列号

TCP 的 sequence number 不是给 packet 编号，而是给字节编号。

例如发送 `hello`，TCP 看到的是 5 个连续字节：

```text
seq:  1001 1002 1003 1004 1005
data: h    e    l    l    o
```

因此 sequence number 记录的是字节在 TCP 字节流中的位置，而不是 `send()` 被调用了几次。这也和前面的 byte stream 模型一致：TCP 不保留应用层消息边界。

#### 4.2 ISN 与 32 位序列号回绕

TCP 的序列号不是固定从 0 开始，而是从一个 Initial Sequence Number 开始。

```text
ISN = Initial Sequence Number
```

例如第一次握手时：

```text
Client SYN: seq = 1000
```

SYN 本身会占用一个 sequence number，所以第一个真正的数据字节不是 `1000`，而是 `1001`。

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

因此，收到 SYN 以后，payload 的位置需要从 TCP sequence space 转换到 stream index space。

```text
stream index = absolute seqno - 1
```

这里的 `-1` 来自 SYN 占用了一个 sequence number。

真实 TCP 报文里的 sequence number 是 32-bit：

```text
0 ~ 2^32 - 1
```

超过最大值后会 wrap around：

```text
4294967295 -> 0 -> 1 -> 2
```

因此 CS144 中使用 `Wrap32` 处理两种编号之间的转换：

```cpp
Wrap32::wrap();    // 64-bit absolute seqno -> 32-bit TCP seqno
Wrap32::unwrap();  // 32-bit TCP seqno -> 64-bit absolute seqno
```

其中 32-bit TCP seqno 会回绕，而 64-bit absolute seqno 更适合程序内部计算。

#### 4.3 累计确认

ACK 表示的是：接收端下一个想要的 sequence number。

它不是在说“我收到了哪个字节”，而是在说：

```text
我已经连续收到 ackno 之前的所有字节；下一个请从 ackno 开始发。
```

例如：

```text
ISN = 1000

seq 1000: SYN
seq 1001: a
seq 1002: b
seq 1003: c
```

如果接收端已经连续收到了 `SYN + abc`，那么回复：

```text
ack = 1004
```

含义是：

```text
下一个想要 seq 1004
```

这就是 cumulative ACK。

如果发生乱序：

```text
先收到 seq 1004 的 d
但是 seq 1001 的 a 还没收到
```

ACK 不能跳到 `1005`，仍然只能保持：

```text
ack = 1001
```

因为 TCP 的 ACK 只能确认已经连续收到的前缀。

#### 4.4 接收窗口

接收窗口表示 receiver 当前还能接收多少数据。

常见说法有：

```text
receive window
advertised window
rwnd
```

例如接收端 buffer 容量是 1000 字节，当前已经缓存了 300 字节：

```text
available capacity = 700
```

那么 receiver 返回的 window size 可以是：

```text
window_size = 700
```

含义是：

```text
从 ackno 开始，sender 最多还能发送 700 字节。
```

这就是 TCP 的 flow control。它的目的不是提高速度，而是防止发送端把接收端 buffer 撑爆。

在嵌入式系统中，receive window 更重要，因为 RAM 通常有限，TCP buffer 不可能无限增长。

#### 4.5 SYN、FIN 与序列号空间

Checkpoint 2 最容易出错的地方，是区分 TCP sequence space 和 ByteStream stream index space。

TCP sequence number 的计算规则是：

```text
SYN 占 1 个 sequence number
payload 的每个 byte 占 1 个 sequence number
FIN 占 1 个 sequence number
```

但是 ByteStream 的 stream index 只计算 payload。

例如：

```text
seq:    1000   1001   1002   1003   1004
thing:  SYN    a      b      c      FIN

stream index:
              0      1      2
              a      b      c
```

因此：

```text
SYN/FIN 占 TCP seqno，但不占 stream index
payload 既占 TCP seqno，也占 stream index
```

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

#### 4.6 CS144 Checkpoint 2：TCP Receiver

## 第四阶段：TCP 发送端

### 5. 发送端可靠性

#### 5.1 发送缓冲区与未确认数据

#### 5.2 滑动窗口

#### 5.3 超时与重传

#### 5.4 RTT 与 RTO

#### 5.5 指数退避（Exponential Backoff）

#### 5.6 重复 ACK 与快速重传

#### 5.7 CS144 Checkpoint 3：TCP Sender

## 第五阶段：TCP 流量与拥塞控制

### 6. TCP 性能

#### 6.1 流量控制与 rwnd

#### 6.2 拥塞控制与 cwnd

#### 6.3 慢启动

#### 6.4 拥塞避免

#### 6.5 快速重传与快速恢复

#### 6.6 带宽时延积（Bandwidth-Delay Product）与吞吐率

#### 6.7 CS144 Checkpoint 4：Measuring the Real World

## 第六阶段：网络接口与 IP 路由器

### 7. 链路层与网络层

#### 7.1 以太网帧与 MAC 地址

#### 7.2 ARP

#### 7.3 IPv4 数据报

#### 7.4 网络接口

#### 7.5 IP 路由与最长前缀匹配

#### 7.6 CS144 Checkpoint 5：Network Interface

#### 7.7 CS144 Checkpoint 6：IP 路由器

#### 7.8 CS144 Checkpoint 7：端到端互联网

## 第七阶段：UDP 与协议选择

### 8. UDP

#### 8.1 数据报模型与消息边界

#### 8.2 UDP 首部与校验和

#### 8.3 `sendto()`、`recvfrom()` 与 UDP 回显

#### 8.4 广播与多播

#### 8.5 应用层序列号、超时与重传

#### 8.6 TCP 与 UDP 对比

#### 8.7 典型嵌入式使用场景

## 第八阶段：嵌入式网络与 lwIP

### 9. lwIP 架构

#### 9.1 协议栈分层与数据流

#### 9.2 netif

#### 9.3 pbuf

#### 9.4 tcp_pcb 与 udp_pcb

#### 9.5 Raw API

#### 9.6 Netconn API

#### 9.7 Socket API

### 10. 嵌入式约束

#### 10.1 固定大小缓冲区与内存池

#### 10.2 环形缓冲区

#### 10.3 定时器与超时

#### 10.4 RTOS 任务、队列与线程安全

#### 10.5 网络字节序与二进制协议

#### 10.6 心跳、看门狗与断线重连

#### 10.7 链路状态、DHCP 与 DNS

## 第九阶段：最终项目

### 11. 虚拟嵌入式遥测设备

#### 11.1 项目需求与系统架构

#### 11.2 自定义二进制帧格式

#### 11.3 TCP 传输模式

#### 11.4 UDP 传输模式

#### 11.5 固定内存与缓冲区管理

#### 11.6 心跳、超时与断线重连

#### 11.7 抓包、丢包和断线测试

### 12. 项目记录

#### 12.1 状态机与数据结构

#### 12.2 内存使用情况

#### 12.3 测试结果

#### 12.4 抓包分析

#### 12.5 README 与求职展示

## 第十阶段：问题与总结

### 13. 尚未理解的问题

### 14. 学习总结
