# Crypto Trading System

## 第一步的实现，一个python的crypto交易系统 ---- MVP (Minimum Viable Product)

### 架构图

```mermaid
flowchart LR
    subgraph GW[Python Gateway]
        WS[Binance WS] --> MD[MarketDataGateway]
        EX[REST Orders] --> OG[OrderGateway]
    end

    subgraph Core[Core Engine]
        BUS[Event Bus]
        STR[Strategy]
        RISK[Risk]
        OMS[OMS State]
        MET[Latency Metrics]
    end

    subgraph Side[Sidecar]
        REC[Recorder/Replay]
    end

    MD --> BUS
    OG --> BUS
    BUS --> STR --> RISK --> OMS --> OG
    BUS --> MET
    BUS --> REC
```

### 基本组件

1. MarketEvent
2. MarketDataGateway (Binance WS)
3. OrderGateway (REST 下单 + user data stream 回报)
4. EventBus (先用python queue实现)
5. OMS (订单状态机 + intent_id 映射)
6. Risk (max_pos/max_qty/rate_limit)
7. Recorder + Replay (append-only log)
8. Latencty metrics (p50/p99: tick->decision->send->fill)

### MarketEvent

为什么交易系统必须是事件驱动？

1. 交易系统的输入是异步的（行情、回报、风控信号不会同时到）。
2. 事件驱动能够把系统拆成对外部变化的响应，而不是函数调用顺序。
3. 事件是可记录的、可回放的，这是可复现性的基础。
4. 事件模型允许系统在断线、重启后恢复状态。
5. 非事件驱动的系统很难解释“为什么下了这笔单”。

### 骨架 + Recorder

### MarketDataGateway

market data websocket 总流程 flow chart (连接 -->  消费 --> 断线 --> 重连):

```mermaid
flowchart TD
A["run_forever()"] --> B["tracker.new_session()"]
B --> C[build ws url]
C --> D{connect WS}
D --> |success| E[connected]
D --> |exception| X[except: disconnect]

E --> F["_consume(ws)"]
F --> |returns normally| G[reset backoff]
G --> B

X --> H[compute backoff + jitter]
H --> I["sleep(backoff)"]
I --> B
```

_consume 细节

```mermaid
flowchart TD

A["_consume(ws)"] --> B[last_msg_ns = now]
B --> C{loop}
C --> D["await wait_for(ws.recv(), timeout=recv_timeout_s)"]

D --> |got raw msg| E[ts_recv = monotic_ns]
E --> F[last_msg_ns = ts_recv]
F --> G[parse raw -> MarketEvent?]
G --> |me != None| H["on_event(me)"]
G --> |None| C
H --> C

D --> |TimeoutError| I[now = monotic_ns]
I --> J{now - last_msg_ns > stale_timeout?}
J --> |no| C
J --> |yes| K[raise RuntimeError stale websocket]
```

断网事故演练

```mermaid
sequenceDiagram
    participant OS as Network/OS
    participant GW as Gateway(run_forever)
    participant WS as WebSocket
    participant REC as Recorder

    GW->>WS: connect()
    WS-->>GW: connected
    loop msgs
        WS-->>GW: raw bookTicker
        GW->>GW: parse -> MarketEvent
        GW->>REC: append(ndjson)
    end

    OS-->>WS: network down (no packets)
    note over WS,GW: socket may not close immediately
    GW->>WS: recv() (wait_for timeout)
    GW->>GW: stale check
    GW-->>GW: raise RuntimeError (stale)
    GW->>GW: except -> log disconnect
    GW->>GW: backoff sleep
    OS-->>WS: network up
    GW->>WS: reconnect()
    WS-->>GW: connected

```

行情不能假设连续、有序，因为接收到的只是交易所通过网络推送的观察结果。
网络会断、会抖，WebSocket 不保证补发，也不保证顺序。
因此在MarketData Gateway 里，断网重连会导致接收时间戳出现断层，而这并不对应真实市场的停顿。
因此，交易系统只能以事件（event）为基本单位，通过状态机来容忍缺失、乱序和重复，而不能依赖连续、有序的数据假设。

### Recorder & Replay

> Replay 比回测更重要，因为 replay 复现的是“系统行为”, 而回测只验证“策略假设”。在真实交易中，亏钱往往来自系统行为错误，而不是策略公式错误。

Replay 是可解释性（Explainability）的技术基础。

> 回测主要验证策略在理想化市场条件下是否赚钱，但真实交易的风险更多来自于系统行为本身。Replay 用真实行情事件作为输入，复现断线、重连、时间戳间隔和顺序变化，验证系统在真实条件下的行为一致性。如果同一份事件在 replay 和 live 下产生不同行为，那说明系统存在 bug，而不是市场变化。因此，在交易系统工程中，replay 比 backtest 更重要，是复现和可演进的基础。

> Replay is for correctness, backtest is for profitability. A system that can't replay can't be trusted.


### Event Bus
> EventBus 是单线程的事件队列，系统里的所有模块都只通过它交换事件。

如果不用EventBus，只有一条路径，(强绑定，单路)如果要拓展新的模块（比如Strategy，Metrics，Risk...）就要改MarketDataGateway（可能有多个）
Gateway 知道所有下游，加模块要改GW，GW变胖，可维护性差。

```mermaid
flowchart LR
    BinanceWS[Binance WebSocket]
    Gateway[MarketData Gateway]
    Recorder[Recorder]

    BinanceWS --> Gateway
    Gateway --> Recorder
```
用了 EventBus 之后，Gateway 只连接 Bus，Bus 决定事件流向，下游模块互不认识。
Gateway只知道 Bus，加模块只需要subscribe。
```mermaid
flowchart LR
    BinanceWS[Binance WebSocket]
    Gateway[MarketData Gateway]
    Bus[EventBus<br/>in-memory queue]
    
    Recorder[Recorder]
    Strategy[Strategy]
    Metrics[Metrics]
    Risk[Risk]

    BinanceWS --> Gateway
    Gateway --> |publish MarketEvent| Bus

    Bus --> |market| Recorder
    Bus --> |market| Strategy
    Bus --> |market| Metrics
    Bus --> |market| Risk
```
```mermaid
flowchart LR
	LiveGW[Live Gateway]
	ReplayGW[Replay Gateway]
	Bus[EventBus]
	
	Recorder[Recorder]
	Strategy[Strategy]
	
	LiveGW --> Bus
	ReplayGW --> Bus
	
	Bus --> Recorder
	Bus --> Strategy
```

一些关键的问题：
- 为什么Gateway不应该直接调用Recorder？
	- 因为Gateway的职责是接入外部市场，只负责publish event，recorder是系统内部的一个消费者，Gateway不应该知道任何内部模块的存在，谁来消费不关Gateway的事情。
	- 如果Gateway直接调用recorder，会导致耦合，一旦新增Strategy，Metrics，Risk等模块，就需要不断改Gateway
- 如果handler很慢，会导致什么？
	- 因为EventBus目前是单线程分发，任意一个handler变慢，整个事件流会被拖慢，queue会开始积压，最终导致内存压力甚至OOM
	- 因果链：handler慢 --> 单线程分发被阻塞 --> producer 继续 publish --> queue 积压 --> latency 飙升 / OOM
- queue慢了怎么办？为什么必须有策略
	- 考虑：市场可以无限快，内存是有限的，延迟堆积比丢数据更危险
	- 所以策略选项：
		- drop newest
		- drop oldest
		- block producer（危险）
		- spill to disk（复杂）
- 为什么replay不应该知道recorder的存在
	- 因为replay和live必须是可替换的event source。
	- replay只负责emit event，至于谁接，必须交给EventBus
- 如果以后加Strategy/OMS，需要改Gateway吗
	- 不需要。Strategy/OMS是事件消费者，只需要订阅EventBus，Gateway只面向event，不面向模块。拓展系统能力，不修改事件源。
总结：
> EventBus的作用，是让系统里的事件流动起来。
> Gateway只负责产生事件，不负责处理。职责仅仅是产生MarketEvent并publish到EventBus。
> Replay和live是等价的事件源。
> 单线程保证顺序和状态安全。
> queue满说明系统过载，必须有丢弃或限流策略。
> 新功能只能通过订阅事件拓展，而不能修改Gateway。

### 接 user data stream + 真回报 (可选 testnet)

### 风控 + 限速

### 总结

### Appendix A. Logging

为了解释系统行为，打log，但是记录所有发生的事情，log会过多。

开发 MD Gateway 的时候，打log记录：
- connect / connected
- disconnect / reconnect
- warn / error 异常
- session 关键状态变化