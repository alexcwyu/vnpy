# Workflows

## Event-Driven Architecture Flow

All communication in VeighNa flows through the `EventEngine`. Components produce events by calling `EventEngine.put(event)`, and the engine distributes them to registered handlers.

```mermaid
sequenceDiagram
    participant GW as Gateway
    participant EE as EventEngine
    participant Q as Event Queue
    participant PT as Processing Thread
    participant OMS as OmsEngine
    participant LOG as LogEngine
    participant APP as Strategy App
    participant UI as MainWindow

    Note over EE: EventEngine.start() launches two threads

    GW->>EE: put(Event(EVENT_TICK, tick))
    EE->>Q: enqueue event

    loop Processing Thread (_run)
        PT->>Q: get(timeout=1s)
        Q-->>PT: Event(EVENT_TICK, tick)
        PT->>PT: _process(event)
        PT->>OMS: handler(event) - update ticks dict
        PT->>APP: handler(event) - strategy on_tick
        PT->>UI: handler(event) - update display
    end

    Note over EE: Timer thread generates EVENT_TIMER every 1s
    EE->>Q: put(Event(EVENT_TIMER))
    PT->>Q: get event
    PT->>APP: timer handler - periodic checks
```

### Handler Registration and Dispatch

The `EventEngine` supports two types of handlers:

1. **Type-specific handlers** -- Registered via `register(type, handler)`, called only for matching event types
2. **General handlers** -- Registered via `register_general(handler)`, called for every event

Dispatch order: type-specific handlers first, then general handlers.

**OmsEngine handler registration** (from `register_event()`):
```
EVENT_TICK     -> process_tick_event     -> updates self.ticks[vt_symbol]
EVENT_ORDER    -> process_order_event    -> updates self.orders, self.active_orders, offset_converter
EVENT_TRADE    -> process_trade_event    -> updates self.trades, offset_converter
EVENT_POSITION -> process_position_event -> updates self.positions, offset_converter
EVENT_ACCOUNT  -> process_account_event  -> updates self.accounts
EVENT_CONTRACT -> process_contract_event -> updates self.contracts, initializes offset_converter
EVENT_QUOTE    -> process_quote_event    -> updates self.quotes, self.active_quotes
```

## Order Submission Flow

```mermaid
sequenceDiagram
    participant USR as User / Strategy
    participant ME as MainEngine
    participant GW as Gateway
    participant EXC as Exchange
    participant EE as EventEngine
    participant OMS as OmsEngine

    USR->>ME: send_order(OrderRequest, gateway_name)
    ME->>ME: write_log("委托下单")
    ME->>GW: send_order(req)
    Note over GW: Gateway creates OrderData<br/>assigns unique orderid<br/>sets status = SUBMITTING
    GW->>GW: on_order(order) - push SUBMITTING event
    GW->>EE: put(Event(EVENT_ORDER, order))
    GW->>EXC: Send order to exchange
    EE->>OMS: process_order_event - store in orders dict

    EXC-->>GW: Order accepted
    GW->>GW: Update status = NOTTRADED
    GW->>EE: put(Event(EVENT_ORDER, order))
    EE->>OMS: process_order_event - update status

    EXC-->>GW: Partial fill
    GW->>GW: Update status = PARTTRADED, create TradeData
    GW->>EE: put(Event(EVENT_ORDER, order))
    GW->>EE: put(Event(EVENT_TRADE, trade))
    EE->>OMS: process_order_event + process_trade_event

    EXC-->>GW: Full fill
    GW->>GW: Update status = ALLTRADED
    GW->>EE: put(Event(EVENT_ORDER, order))
    EE->>OMS: process_order_event - remove from active_orders

    GW-->>ME: return vt_orderid
    ME-->>USR: return vt_orderid
```

## Order Cancellation Flow

```mermaid
sequenceDiagram
    participant USR as User / Strategy
    participant ME as MainEngine
    participant GW as Gateway
    participant EXC as Exchange
    participant EE as EventEngine

    USR->>ME: cancel_order(CancelRequest, gateway_name)
    ME->>ME: write_log("委托撤单")
    ME->>GW: cancel_order(req)
    GW->>EXC: Send cancel request

    EXC-->>GW: Cancel confirmed
    GW->>GW: Update status = CANCELLED
    GW->>EE: put(Event(EVENT_ORDER, order))
```

## Market Data Subscription Flow

```mermaid
sequenceDiagram
    participant USR as User / Strategy
    participant ME as MainEngine
    participant GW as Gateway
    participant EXC as Exchange
    participant EE as EventEngine
    participant OMS as OmsEngine
    participant UI as TickMonitor

    USR->>ME: subscribe(SubscribeRequest, gateway_name)
    ME->>ME: write_log("订阅行情")
    ME->>GW: subscribe(req)
    GW->>EXC: Subscribe to market data stream

    loop Market Data Stream
        EXC-->>GW: Raw tick data
        GW->>GW: Parse into TickData
        GW->>EE: on_tick(tick)
        Note over GW: Pushes EVENT_TICK and<br/>EVENT_TICK + vt_symbol
        EE->>OMS: process_tick_event - update ticks cache
        EE->>UI: tick handler - update display
    end
```

## CTA Strategy Lifecycle

A CTA (Commodity Trading Advisor) strategy operates on a single instrument with the following lifecycle:

```mermaid
stateDiagram-v2
    [*] --> Created: add_strategy()
    Created --> Inited: init_strategy()
    Inited --> Trading: start_strategy()
    Trading --> Stopped: stop_strategy()
    Stopped --> Inited: init_strategy()
    Stopped --> Removed: remove_strategy()
    Removed --> [*]

    state Trading {
        [*] --> WaitingForData
        WaitingForData --> ProcessingBar: on_bar() / on_tick()
        ProcessingBar --> CalculatingSignal: Compute indicators
        CalculatingSignal --> CheckPosition: Generate signal
        CheckPosition --> SendOrder: Position adjustment needed
        SendOrder --> WaitingForData: Order submitted
        CheckPosition --> WaitingForData: No action
    }
```

**CTA strategy signal-to-order flow**:
1. Strategy receives tick/bar data via callback
2. Strategy computes technical indicators (using TA-Lib, NumPy)
3. Strategy generates trading signal (buy/sell/short/cover)
4. Strategy calls `buy()`, `sell()`, `short()`, or `cover()` on the strategy engine
5. Strategy engine handles offset conversion (open/close/close_today/close_yesterday)
6. Order is sent through `MainEngine.send_order()` to the gateway
7. Gateway pushes order/trade events back through `EventEngine`
8. Strategy receives `on_order()` and `on_trade()` callbacks

## Alpha Strategy Workflow

The alpha strategy framework operates on multi-asset portfolios:

```mermaid
flowchart TD
    A[Load Bar Data<br/>AlphaLab.load_bar_df] --> B[Create AlphaDataset<br/>Define train/valid/test periods]
    B --> C[Add Features<br/>Expression-based or Polars-based]
    C --> D[Set Label<br/>Forward return expression]
    D --> E[Prepare Data<br/>Multiprocessing calculation]
    E --> F[Process Data<br/>Apply processors]
    F --> G[Train AlphaModel<br/>LGB / MLP / Lasso]
    G --> H[Predict Signals<br/>model.predict on test segment]
    H --> I[Evaluate Performance<br/>Alphalens tear sheet]
    H --> J[Run Backtest<br/>BacktestingEngine]

    subgraph "BacktestingEngine"
        J --> K[Load History Data]
        K --> L[Initialize Strategy]
        L --> M[Replay Bars Chronologically]
        M --> N[Strategy.on_bars<br/>Set targets, execute trading]
        N --> O[Cross-match Orders<br/>Generate trades]
        O --> P[Calculate Daily PnL]
        P --> Q[Generate Statistics<br/>Sharpe, drawdown, etc.]
        Q --> R[Show Charts<br/>Plotly visualization]
    end
```

## Spread Trading Workflow

Spread trading involves constructing synthetic instruments from multiple legs:

```mermaid
flowchart TD
    A[Define Spread<br/>Legs + price formula] --> B[Subscribe Leg Ticks]
    B --> C[Receive Tick Data]
    C --> D[Calculate Spread Price<br/>Using leg prices + multipliers]
    D --> E[Generate Spread Tick<br/>Synthetic tick data]
    E --> F[Strategy on_spread_tick]
    F --> G{Signal?}
    G -->|Buy Spread| H[Send Leg Orders<br/>Buy leg A, Sell leg B]
    G -->|Sell Spread| I[Send Leg Orders<br/>Sell leg A, Buy leg B]
    G -->|No Signal| C
    H --> J[Track Leg Fills]
    I --> J
    J --> K[Update Spread Position]
    K --> C
```

## Portfolio Management Workflow

```mermaid
flowchart TD
    A[Initialize Portfolio Strategy] --> B[Subscribe to Instruments]
    B --> C[Receive Bar Slice<br/>on_bars callback]
    C --> D[Calculate Target Positions<br/>Based on signals/model]
    D --> E[Compare Target vs Actual]
    E --> F{Adjustment Needed?}
    F -->|Yes| G[Execute Trading<br/>Cancel all + rebalance]
    F -->|No| C
    G --> H[Send Orders for Each Leg]
    H --> I[Process Fills]
    I --> J[Update Position Tracking]
    J --> C
```

## Backtesting Flow

```mermaid
flowchart TD
    A[Set Parameters<br/>symbols, interval, date range] --> B[Load Contract Settings<br/>rates, size, pricetick]
    B --> C[Load History Data<br/>From AlphaLab Parquet files]
    C --> D[Initialize Strategy<br/>strategy.on_init]
    D --> E[Run Backtesting<br/>Iterate sorted datetimes]

    E --> F[For each datetime:<br/>Collect bars for all symbols]
    F --> G[Cross-match Limit Orders<br/>Against bar OHLC]
    G --> H[Generate Trades<br/>Update positions]
    H --> I[Call strategy.on_bars<br/>Strategy logic executes]
    I --> J[New Orders Submitted?]
    J -->|Yes| F
    J -->|No| F

    E --> K[After all bars processed]
    K --> L[Calculate Daily Results<br/>PnL, commission, slippage]
    L --> M[Generate Statistics<br/>Total return, Sharpe, max drawdown]
    M --> N[Show Charts<br/>Equity curve, drawdown, daily PnL]
```

## Offset Conversion Flow

For Chinese futures exchanges (SHFE/INE), closing positions requires specifying whether to close today's or yesterday's positions. The `OffsetConverter` handles this automatically.

```mermaid
flowchart TD
    A[OrderRequest<br/>direction + offset] --> B{Exchange Type?}

    B -->|SHFE/INE| C{Offset = OPEN?}
    C -->|Yes| D[Pass through as-is]
    C -->|No| E[Check Available Positions]
    E --> F{Volume <= Today Available?}
    F -->|Yes| G[Set offset = CLOSETODAY]
    F -->|No| H[Split into CLOSETODAY + CLOSEYESTERDAY]

    B -->|Lock Mode| I{Today Volume > 0?}
    I -->|Yes| J[Lock: Set offset = OPEN<br/>Opens opposite to lock]
    I -->|No| K[Close YD first, then OPEN remainder]

    B -->|Net Mode| L[Close available first<br/>then OPEN remainder]

    B -->|Other Exchange| M[Pass through as-is]
```

## RPC Distributed Deployment

```mermaid
sequenceDiagram
    participant Client as RpcClient
    participant REQ as ZMQ REQ Socket
    participant REP as ZMQ REP Socket
    participant Server as RpcServer
    participant SUB as ZMQ SUB Socket
    participant PUB as ZMQ PUB Socket

    Note over Server: Bind REP + PUB sockets
    Note over Client: Connect REQ + SUB sockets

    Client->>REQ: send_pyobj([func_name, args, kwargs])
    REQ->>REP: ZMQ transport
    Server->>REP: recv_pyobj -> call function
    Server->>REP: send_pyobj([True, result])
    REP->>REQ: ZMQ transport
    REQ-->>Client: return result

    loop Every 10 seconds
        Server->>PUB: publish(HEARTBEAT_TOPIC, timestamp)
        PUB->>SUB: ZMQ transport
        SUB-->>Client: heartbeat received
    end

    loop Data Push
        Server->>PUB: publish(topic, data)
        PUB->>SUB: ZMQ transport
        SUB-->>Client: callback(topic, data)
    end
```

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [State Management](state-management.md) — State lifecycle and data models
- [Development](development.md) — Development guide and best practices
