# State Management

## Order State Machine

Orders in VeighNa follow a well-defined state machine. The `Status` enum in [`src/vnpy/trader/constant.py`](../src/vnpy/trader/constant.py) defines the possible states, and `ACTIVE_STATUSES` in [`src/vnpy/trader/object.py`](../src/vnpy/trader/object.py) determines which orders are considered active.

```mermaid
stateDiagram-v2
    [*] --> SUBMITTING: Gateway creates OrderData
    SUBMITTING --> NOTTRADED: Exchange accepts order
    SUBMITTING --> REJECTED: Exchange rejects order

    NOTTRADED --> PARTTRADED: Partial fill received
    NOTTRADED --> ALLTRADED: Full fill received
    NOTTRADED --> CANCELLED: Cancel confirmed

    PARTTRADED --> PARTTRADED: Additional partial fill
    PARTTRADED --> ALLTRADED: Final fill received
    PARTTRADED --> CANCELLED: Cancel confirmed

    ALLTRADED --> [*]: Terminal state
    CANCELLED --> [*]: Terminal state
    REJECTED --> [*]: Terminal state

    state "Active States" as active {
        SUBMITTING
        NOTTRADED
        PARTTRADED
    }
```

**Active statuses**: `SUBMITTING`, `NOTTRADED`, `PARTTRADED` -- these are tracked in `OmsEngine.active_orders`.

**Terminal statuses**: `ALLTRADED`, `CANCELLED`, `REJECTED` -- when an order reaches these states, it is removed from `active_orders` but retained in the `orders` dictionary.

**Status enum values** (Chinese locale):
| Status | Chinese | Meaning |
|--------|---------|---------|
| `SUBMITTING` | "提交中" | Order submitted, awaiting exchange acknowledgment |
| `NOTTRADED` | "未成交" | Accepted by exchange, no fills yet |
| `PARTTRADED` | "部分成交" | Partially filled |
| `ALLTRADED` | "全部成交" | Fully filled |
| `CANCELLED` | "已撤销" | Cancelled (may have partial fills) |
| `REJECTED` | "拒单" | Rejected by exchange |

**OmsEngine order tracking logic** (from `process_order_event`):
```
if order.is_active():
    active_orders[vt_orderid] = order    # Track active order
elif vt_orderid in active_orders:
    active_orders.pop(vt_orderid)        # Remove completed/cancelled order
```

## Quote State Machine

Quotes (two-sided orders for market making) follow the same state machine as orders, using the same `Status` enum and `ACTIVE_STATUSES` set. The `QuoteData.is_active()` method checks the same conditions.

## Position Tracking

Position state is managed at two levels: the `OmsEngine` level and the `OffsetConverter` level.

### OmsEngine Position Tracking

The `OmsEngine` maintains a flat dictionary of positions keyed by `vt_positionid`:

```
vt_positionid = "{gateway_name}.{symbol}.{exchange}.{direction}"
```

Each `PositionData` contains:

| Field | Type | Description |
|-------|------|-------------|
| `symbol` | str | Instrument symbol |
| `exchange` | Exchange | Exchange enum |
| `direction` | Direction | LONG, SHORT, or NET |
| `volume` | float | Total position volume |
| `frozen` | float | Frozen (locked) volume |
| `price` | float | Average entry price |
| `pnl` | float | Unrealized profit/loss |
| `yd_volume` | float | Yesterday's position volume |

```mermaid
classDiagram
    class PositionData {
        +symbol: str
        +exchange: Exchange
        +direction: Direction
        +volume: float
        +frozen: float
        +price: float
        +pnl: float
        +yd_volume: float
        +vt_symbol: str
        +vt_positionid: str
    }
```

### OffsetConverter Position Tracking

The `OffsetConverter` ([`src/vnpy/trader/converter.py`](../src/vnpy/trader/converter.py)) maintains detailed position state via `PositionHolding` objects for exchanges that require today/yesterday distinction (SHFE, INE).

```mermaid
classDiagram
    class PositionHolding {
        +vt_symbol: str
        +exchange: Exchange
        +long_pos: float
        +long_yd: float
        +long_td: float
        +short_pos: float
        +short_yd: float
        +short_td: float
        +long_pos_frozen: float
        +long_yd_frozen: float
        +long_td_frozen: float
        +short_pos_frozen: float
        +short_yd_frozen: float
        +short_td_frozen: float
        +active_orders: dict
        +update_position(position)
        +update_order(order)
        +update_trade(trade)
        +calculate_frozen()
        +convert_order_request_shfe(req)
        +convert_order_request_lock(req)
        +convert_order_request_net(req)
    }

    class OffsetConverter {
        +holdings: dict[str, PositionHolding]
        +update_position(position)
        +update_order(order)
        +update_trade(trade)
        +convert_order_request(req, lock, net)
        +is_convert_required(vt_symbol)
    }

    OffsetConverter --> PositionHolding : manages
```

**Position update flow**:

```mermaid
flowchart TD
    A[Trade Event] --> B{Direction?}
    B -->|LONG| C{Offset?}
    B -->|SHORT| D{Offset?}

    C -->|OPEN| E[long_td += volume]
    C -->|CLOSETODAY| F[short_td -= volume]
    C -->|CLOSEYESTERDAY| G[short_yd -= volume]
    C -->|CLOSE| H{SHFE/INE?}
    H -->|Yes| I[short_yd -= volume]
    H -->|No| J[short_td -= volume<br/>overflow to short_yd]

    D -->|OPEN| K[short_td += volume]
    D -->|CLOSETODAY| L[long_td -= volume]
    D -->|CLOSEYESTERDAY| M[long_yd -= volume]
    D -->|CLOSE| N{SHFE/INE?}
    N -->|Yes| O[long_yd -= volume]
    N -->|No| P[long_td -= volume<br/>overflow to long_yd]

    E & F & G & I & J & K & L & M & O & P --> Q[Recalculate totals<br/>long_pos = long_td + long_yd<br/>short_pos = short_td + short_yd]
    Q --> R[Ensure frozen <= total]
```

**Frozen volume calculation**: Active orders that close positions contribute to frozen volume. The system tracks `{side}_{period}_frozen` (e.g., `long_td_frozen`) to prevent over-closing.

## Gateway Connection State

Gateway connection state is implicit -- determined by whether the gateway has successfully connected to the exchange. The `BaseGateway` does not have a formal connection state enum; instead:

- **Connected**: The gateway has established its connection and begins pushing `on_contract`, `on_account`, `on_position`, and market data events
- **Disconnected**: The gateway should attempt automatic reconnection
- **Error**: Communicated via `write_log()` which pushes `EVENT_LOG` events

```mermaid
stateDiagram-v2
    [*] --> Idle: Gateway instantiated
    Idle --> Connecting: connect(setting) called
    Connecting --> Connected: Connection established
    Connecting --> Error: Connection failed
    Connected --> Querying: Query contracts/accounts/positions
    Querying --> Active: All queries complete
    Active --> Disconnected: Connection lost
    Disconnected --> Connecting: Auto-reconnect
    Active --> Closing: close() called
    Closing --> [*]
    Error --> Connecting: Retry
```

**On successful connection**, gateways must:
1. Query and push `ContractData` via `on_contract()`
2. Query and push `AccountData` via `on_account()`
3. Query and push `PositionData` via `on_position()`
4. Query and push `OrderData` via `on_order()` (for existing open orders)
5. Query and push `TradeData` via `on_trade()` (for recent trades)

## Strategy State (Alpha Strategy)

Alpha strategies have a simpler lifecycle managed by the `BacktestingEngine`:

```mermaid
stateDiagram-v2
    [*] --> Created: Strategy class instantiated
    Created --> Initialized: on_init() called
    Initialized --> Running: Engine replays bars

    state Running {
        [*] --> ProcessBars
        ProcessBars --> SetTargets: on_bars() sets target positions
        SetTargets --> ExecuteTrading: execute_trading() rebalances
        ExecuteTrading --> ProcessBars: Next bar slice
    }

    Running --> Complete: All bars processed
    Complete --> [*]: Statistics generated
```

**Strategy position state** is tracked in two dictionaries:
- `pos_data: dict[str, float]` -- Actual current positions (updated on `update_trade`)
- `target_data: dict[str, float]` -- Desired target positions (set by strategy logic)

The `execute_trading()` method computes the diff between target and actual, then sends buy/sell/short/cover orders accordingly.

## Account/Balance Management

Account state is straightforward:

```mermaid
classDiagram
    class AccountData {
        +accountid: str
        +balance: float
        +frozen: float
        +available: float
        +vt_accountid: str
    }
    note for AccountData "available = balance - frozen\nvt_accountid = gateway_name.accountid"
```

- **balance** -- Total account equity
- **frozen** -- Margin/collateral locked by open positions and pending orders
- **available** -- Computed as `balance - frozen`, representing funds available for new orders

The `OmsEngine` caches the latest `AccountData` per `vt_accountid`. Gateways push updated account data periodically or on change.

## Event-Driven State Consistency

All state updates flow through the `EventEngine`, ensuring a consistent update order:

```mermaid
flowchart LR
    GW[Gateway] -->|1. on_contract| EE[EventEngine]
    GW -->|2. on_account| EE
    GW -->|3. on_position| EE
    GW -->|4. on_order| EE
    GW -->|5. on_trade| EE
    GW -->|6. on_tick| EE

    EE -->|All events| OMS[OmsEngine]
    OMS -->|Stores| STATE[(In-Memory State)]
```

The single-threaded event processing loop in `EventEngine._run()` guarantees that state updates are processed sequentially, preventing race conditions between order updates, trade fills, and position changes.

## Global Settings State

Platform configuration is managed through the `SETTINGS` dictionary in [`src/vnpy/trader/setting.py`](../src/vnpy/trader/setting.py):

| Setting | Default | Description |
|---------|---------|-------------|
| `font.family` | "微软雅黑" | UI font family |
| `font.size` | 12 | UI font size |
| `log.active` | True | Enable logging |
| `log.level` | INFO | Log level |
| `log.console` | True | Log to console |
| `log.file` | True | Log to file |
| `email.*` | (various) | SMTP email configuration |
| `datafeed.name` | "" | Datafeed module name |
| `database.timezone` | (local) | Database timezone |
| `database.name` | "sqlite" | Database module name |
| `database.*` | (various) | Database connection parameters |

Settings are loaded from `vt_setting.json` in the `.vntrader` directory (under CWD or home directory). Note: `.vntrader/` is a runtime directory created by VeighNa outside the repository; it is not checked into source control.

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [Workflow](workflow.md) — Event flows and processing pipelines
- [Development](development.md) — Development guide and best practices
