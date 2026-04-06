# Development Guide

## Development Setup

### Prerequisites

- **Python 3.13** (required -- see `requires-python = "==3.13.*"` in `pyproject.toml`)
- **uv** (recommended package manager)
- **TA-Lib** C library (must be installed system-wide before pip install)

### Installation

```bash
# Clone the repository
git clone https://github.com/vnpy/vnpy.git
cd vnpy

# Switch to development branch
git checkout feature/ayu_develop

# Install with uv (recommended)
uv sync --active --all-groups --all-extras

# Or install with pip in editable mode
pip install -e ".[alpha]"

# Install dev dependencies
pip install -e ".[dev-legacy]"
```

### Configuration

VeighNa stores configuration and data in a `.vntrader` directory:
- If `.vntrader/` exists in the current working directory, it uses that
- Otherwise, it creates/uses `~/.vntrader/`

Key configuration file: `.vntrader/vt_setting.json` (overrides defaults from `SETTINGS` dict).

### Running Tests

```bash
# Run all tests
pytest

# Run with specific options (configured in pyproject.toml)
pytest -ra -q --strict-markers

# Run specific test file
pytest tests/test_specific.py

# Run with coverage
pytest --cov=vnpy
```

### Code Quality

```bash
# Format code
black src/vnpy/

# Lint
ruff check src/vnpy/

# Type checking
mypy src/vnpy/
```

## Project Structure

```
vnpy/
├── pyproject.toml                    # Project metadata, dependencies, tool config
├── src/
│   └── vnpy/
│       ├── __init__.py               # Version: __version__ = "4.3.0"
│       ├── py.typed                  # PEP 561 type marker
│       │
│       ├── event/                    # Event-driven framework
│       │   ├── __init__.py           # Exports: Event, EventEngine, EVENT_TIMER
│       │   └── engine.py            # EventEngine, Event, HandlerType
│       │
│       ├── trader/                   # Core trading module
│       │   ├── __init__.py
│       │   ├── engine.py            # MainEngine, BaseEngine, OmsEngine, LogEngine, EmailEngine
│       │   ├── gateway.py           # BaseGateway abstract class
│       │   ├── app.py               # BaseApp abstract class
│       │   ├── object.py            # Data classes: TickData, BarData, OrderData, etc.
│       │   ├── constant.py          # Enums: Direction, Status, Exchange, Product, etc.
│       │   ├── event.py             # Event type constants: EVENT_TICK, EVENT_ORDER, etc.
│       │   ├── database.py          # BaseDatabase, BarOverview, TickOverview
│       │   ├── datafeed.py          # BaseDatafeed
│       │   ├── converter.py         # OffsetConverter, PositionHolding
│       │   ├── optimize.py          # OptimizationSetting, brute force & GA optimization
│       │   ├── setting.py           # SETTINGS dict, loads vt_setting.json
│       │   ├── utility.py           # Helpers: load_json, save_json, BarGenerator, ArrayManager
│       │   ├── logger.py            # Loguru logger configuration
│       │   ├── locale/              # i18n (Babel-based, Chinese/English)
│       │   │   ├── __init__.py      # _() translation function
│       │   │   ├── build_hook.py    # Hatch build hook for .mo compilation
│       │   │   └── en/LC_MESSAGES/  # English translations
│       │   └── ui/                  # Desktop GUI (PySide6)
│       │       ├── __init__.py
│       │       ├── qt.py            # Qt imports and app creation
│       │       ├── mainwindow.py    # MainWindow class
│       │       └── widget.py        # Monitor widgets, dialogs
│       │
│       ├── chart/                    # K-line charting (pyqtgraph)
│       │   ├── __init__.py
│       │   ├── widget.py            # ChartWidget, ChartCursor
│       │   ├── item.py              # ChartItem (candles, volume, etc.)
│       │   ├── manager.py           # BarManager (data management)
│       │   ├── base.py              # Colors, fonts, constants
│       │   └── axis.py              # DatetimeAxis
│       │
│       ├── rpc/                      # ZeroMQ RPC framework
│       │   ├── __init__.py
│       │   ├── server.py            # RpcServer (REP + PUB)
│       │   ├── client.py            # RpcClient (REQ + SUB)
│       │   └── common.py            # Heartbeat constants
│       │
│       └── alpha/                    # Alpha research module
│           ├── __init__.py          # Exports: AlphaLab, AlphaDataset, AlphaModel, etc.
│           ├── lab.py               # AlphaLab (research workspace)
│           ├── logger.py            # Alpha-specific logger
│           ├── dataset/             # Factor datasets
│           │   ├── __init__.py
│           │   ├── template.py      # AlphaDataset class
│           │   ├── utility.py       # Segment enum, expression helpers
│           │   ├── processor.py     # Data processors
│           │   ├── math_function.py # Math factor functions
│           │   ├── ts_function.py   # Time-series factor functions
│           │   ├── cs_function.py   # Cross-section factor functions
│           │   ├── ta_function.py   # Technical analysis factor functions
│           │   └── datasets/        # Pre-built datasets (Alpha101, Alpha158)
│           ├── model/               # ML models
│           │   ├── __init__.py
│           │   ├── template.py      # AlphaModel abstract class
│           │   └── models/          # Implementations
│           │       ├── lgb_model.py  # LightGBM model
│           │       ├── mlp_model.py  # MLP (PyTorch) model
│           │       └── lasso_model.py # Lasso regression model
│           └── strategy/            # Portfolio strategies
│               ├── __init__.py
│               ├── template.py      # AlphaStrategy abstract class
│               ├── backtesting.py   # BacktestingEngine
│               └── strategies/      # Strategy implementations
│                   └── equity_demo_strategy.py
│
├── tests/                            # Test suite
├── docs/                             # Documentation (this directory)
└── scripts/                          # Utility scripts
```

## Gateway Development Guide

To create a new gateway for connecting to an exchange:

### 1. Create the gateway package

Gateway implementations live in separate packages named `vnpy_{gateway_name}` (e.g., `vnpy_ctp`, `vnpy_ib`).

### 2. Implement BaseGateway

```python
from vnpy.event import EventEngine
from vnpy.trader.gateway import BaseGateway
from vnpy.trader.object import (
    TickData, OrderData, TradeData, PositionData,
    AccountData, ContractData, LogData,
    SubscribeRequest, OrderRequest, CancelRequest,
    HistoryRequest, BarData, Exchange
)
from vnpy.trader.constant import Direction, OrderType, Offset, Status


class MyExchangeGateway(BaseGateway):
    """Gateway for MyExchange."""

    default_name: str = "MYEXCHANGE"

    default_setting: dict = {
        "api_key": "",
        "api_secret": "",
        "server": "REAL",      # REAL or TEST
    }

    exchanges: list[Exchange] = [Exchange.SMART]

    def __init__(self, event_engine: EventEngine, gateway_name: str) -> None:
        super().__init__(event_engine, gateway_name)
        # Initialize exchange API client, websocket connections, etc.

    def connect(self, setting: dict) -> None:
        """Connect to the exchange.

        Must:
        1. Establish connection using credentials from setting
        2. Log connection status via write_log()
        3. Query and push contracts via on_contract()
        4. Query and push account info via on_account()
        5. Query and push positions via on_position()
        6. Query and push open orders via on_order()
        """
        api_key = setting["api_key"]
        api_secret = setting["api_secret"]
        # ... establish connection ...
        self.write_log("Exchange connected")

    def close(self) -> None:
        """Close the gateway connection."""
        pass

    def subscribe(self, req: SubscribeRequest) -> None:
        """Subscribe to market data for a symbol."""
        # Subscribe to exchange websocket for the symbol
        pass

    def send_order(self, req: OrderRequest) -> str:
        """Send a new order. Returns vt_orderid.

        Must:
        1. Create OrderData from req using req.create_order_data()
        2. Assign a unique orderid
        3. Send to exchange
        4. Set status to SUBMITTING (success) or REJECTED (failure)
        5. Call on_order() to push the order event
        6. Return vt_orderid
        """
        # Generate unique order ID
        self.order_count += 1
        orderid = str(self.order_count)

        order = req.create_order_data(orderid, self.gateway_name)
        order.status = Status.SUBMITTING
        self.on_order(order)

        # Send to exchange API
        # ...

        return order.vt_orderid

    def cancel_order(self, req: CancelRequest) -> None:
        """Cancel an existing order."""
        # Send cancel to exchange API
        pass

    def query_account(self) -> None:
        """Query account balance."""
        # Fetch from exchange, create AccountData, call on_account()
        pass

    def query_position(self) -> None:
        """Query holding positions."""
        # Fetch from exchange, create PositionData, call on_position()
        pass

    def query_history(self, req: HistoryRequest) -> list[BarData]:
        """Query historical bar data (optional)."""
        # Fetch from exchange API
        return []
```

### 3. Key gateway requirements

- **Thread-safety**: All methods must be thread-safe with no mutable shared state
- **Non-blocking**: Methods should not block the calling thread
- **Auto-reconnect**: Implement automatic reconnection on connection loss
- **Immutable callbacks**: Data passed to `on_*` callbacks must not be modified afterward; use `copy.copy()` if caching data internally
- **Dual event push**: The `on_tick`, `on_trade`, `on_order`, `on_position`, `on_account`, and `on_quote` methods push both a general event and a symbol/id-specific event

### 4. Register the gateway

In the application startup code:

```python
from vnpy.trader.engine import MainEngine
from vnpy.event import EventEngine
from my_gateway import MyExchangeGateway

event_engine = EventEngine()
main_engine = MainEngine(event_engine)
main_engine.add_gateway(MyExchangeGateway)
```

## Strategy Development (CTA Template)

CTA strategies operate on single instruments. While the CTA engine itself is an external app (`vnpy_ctastrategy`), strategy development follows a standard template pattern.

### Basic CTA Strategy Pattern

```python
from vnpy.trader.object import BarData, TickData, TradeData, OrderData
from vnpy.trader.constant import Direction, Offset


class MyCtaStrategy:
    """Example CTA strategy structure."""

    # Strategy parameters (can be optimized)
    fast_window: int = 10
    slow_window: int = 30

    # Strategy variables (runtime state)
    fast_ma: float = 0
    slow_ma: float = 0
    pos: int = 0

    def on_init(self) -> None:
        """Called when strategy is initialized.

        Load historical data for indicator warm-up.
        """
        self.write_log("Strategy initialized")
        self.load_bar(10)  # Load 10 days of history

    def on_start(self) -> None:
        """Called when strategy starts trading."""
        self.write_log("Strategy started")

    def on_stop(self) -> None:
        """Called when strategy stops."""
        self.write_log("Strategy stopped")

    def on_bar(self, bar: BarData) -> None:
        """Called on each new bar.

        Core strategy logic goes here.
        """
        # Update indicators
        am = self.array_manager
        am.update_bar(bar)
        if not am.inited:
            return

        self.fast_ma = am.sma(self.fast_window)
        self.slow_ma = am.sma(self.slow_window)

        # Generate signals
        if self.fast_ma > self.slow_ma and self.pos == 0:
            self.buy(bar.close_price, 1)
        elif self.fast_ma < self.slow_ma and self.pos > 0:
            self.sell(bar.close_price, 1)

    def on_order(self, order: OrderData) -> None:
        """Called on order update."""
        pass

    def on_trade(self, trade: TradeData) -> None:
        """Called on trade fill."""
        pass
```

## Alpha Strategy Development

Alpha strategies operate on multi-asset portfolios using the built-in alpha framework.

### Alpha Strategy Pattern

```python
from vnpy.alpha.strategy.template import AlphaStrategy
from vnpy.trader.object import BarData, TradeData


class MyAlphaStrategy(AlphaStrategy):
    """Multi-asset portfolio strategy."""

    # Parameters
    rebalance_days: int = 5
    price_add: float = 0.01

    # Variables
    day_count: int = 0

    def on_init(self) -> None:
        """Initialize strategy."""
        self.write_log("Alpha strategy initialized")

    def on_bars(self, bars: dict[str, BarData]) -> None:
        """Called with bar data for all symbols.

        bars is a dict: {vt_symbol: BarData}
        """
        self.day_count += 1

        if self.day_count % self.rebalance_days != 0:
            return

        # Get model signals
        signal = self.get_signal()

        # Set target positions based on signals
        for vt_symbol in self.vt_symbols:
            target = signal.get(vt_symbol, 0)
            self.set_target(vt_symbol, target)

        # Execute rebalancing
        self.execute_trading(bars, self.price_add)

    def on_trade(self, trade: TradeData) -> None:
        """Handle trade fills."""
        pass
```

### Running an Alpha Backtest

```python
from datetime import datetime
from vnpy.alpha import AlphaLab, BacktestingEngine
from vnpy.trader.constant import Interval

# Set up lab and engine
lab = AlphaLab("/path/to/lab")
engine = BacktestingEngine(lab)

# Configure backtest
engine.set_parameters(
    vt_symbols=["IF2401.CFFEX", "IC2401.CFFEX"],
    interval=Interval.DAILY,
    start=datetime(2023, 1, 1),
    end=datetime(2023, 12, 31),
    capital=1_000_000,
    risk_free=0.03,
    annual_days=240
)

# Load contract settings
engine.load_contract_settings()

# Load data and run
engine.load_data()
engine.add_strategy(MyAlphaStrategy, {})
engine.run_backtesting()

# View results
engine.calculate_result()
engine.calculate_statistics()
engine.show_chart()
```

## App Development Guide

Apps extend VeighNa with pluggable functionality. Each app is a separate package that registers with the platform.

### 1. Define the App class

```python
from pathlib import Path
from vnpy.trader.app import BaseApp
from .engine import MyEngine


class MyApp(BaseApp):
    app_name: str = "my_app"
    app_module: str = __module__
    app_path: Path = Path(__file__).parent
    display_name: str = "My Application"
    engine_class: type = MyEngine
    widget_name: str = "MyWidget"
    icon_name: str = str(Path(__file__).parent.joinpath("ui", "my_app.ico"))
```

### 2. Implement the Engine

```python
from vnpy.event import Event, EventEngine
from vnpy.trader.engine import BaseEngine, MainEngine
from vnpy.trader.event import EVENT_TICK, EVENT_ORDER, EVENT_TRADE, EVENT_TIMER


class MyEngine(BaseEngine):
    def __init__(self, main_engine: MainEngine, event_engine: EventEngine) -> None:
        super().__init__(main_engine, event_engine, "my_app")
        # Register event handlers
        self.event_engine.register(EVENT_TICK, self.process_tick)
        self.event_engine.register(EVENT_TIMER, self.process_timer)

    def process_tick(self, event: Event) -> None:
        """Handle tick events."""
        tick = event.data
        # App-specific logic

    def process_timer(self, event: Event) -> None:
        """Handle timer events (every 1 second)."""
        pass

    def close(self) -> None:
        """Clean up resources."""
        pass
```

### 3. Create the UI widget

```python
from vnpy.trader.ui import QtWidgets
from vnpy.trader.engine import MainEngine
from vnpy.event import EventEngine


class MyWidget(QtWidgets.QWidget):
    def __init__(self, main_engine: MainEngine, event_engine: EventEngine) -> None:
        super().__init__()
        self.main_engine = main_engine
        self.event_engine = event_engine
        self.init_ui()

    def init_ui(self) -> None:
        self.setWindowTitle("My Application")
        # Build PySide6 UI
```

### 4. Register the app

```python
main_engine.add_app(MyApp)
```

The `MainWindow` automatically discovers registered apps and adds them to the "功能" (Functions) menu.

## Testing

### Test structure

Tests live in the `tests/` directory and use pytest with the configuration in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
minversion = "9.0"
addopts = ["-ra", "-q", "--strict-markers", "--import-mode=importlib"]
testpaths = ["tests"]
pythonpath = ["src"]
```

### Writing tests

```python
import pytest
from vnpy.event import EventEngine, Event
from vnpy.trader.constant import Exchange, Direction, Status
from vnpy.trader.object import OrderData, TickData


def test_event_engine_dispatch():
    """Test that EventEngine dispatches events to handlers."""
    engine = EventEngine()
    received = []

    def handler(event: Event):
        received.append(event.data)

    engine.register("test", handler)
    engine.start()
    engine.put(Event("test", "hello"))

    import time
    time.sleep(0.5)
    engine.stop()

    assert received == ["hello"]


def test_order_is_active():
    """Test OrderData.is_active() status check."""
    order = OrderData(
        symbol="IF2401",
        exchange=Exchange.CFFEX,
        orderid="001",
        gateway_name="TEST"
    )
    order.status = Status.SUBMITTING
    assert order.is_active()

    order.status = Status.ALLTRADED
    assert not order.is_active()
```

### Key testing patterns

- Use `EventEngine` in tests but remember to `start()` and `stop()` it
- Event processing is asynchronous -- add small delays or use synchronization
- Data objects are dataclasses -- construct them with required fields
- The `.vntrader` directory may be created during tests; use tmp directories when possible

## Configuration Reference

VeighNa reads settings from `.vntrader/vt_setting.json`, which overrides the defaults in `src/vnpy/trader/setting.py`. If `.vntrader/` exists in the current working directory it is used; otherwise `~/.vntrader/` is used.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `font.family` | `str` | `"微软雅黑"` | GUI font family |
| `font.size` | `int` | `12` | GUI font size |
| `log.active` | `bool` | `true` | Enable/disable logging |
| `log.level` | `int` | `20` (INFO) | Python logging level (10=DEBUG, 20=INFO, 30=WARNING) |
| `log.console` | `bool` | `true` | Log output to console |
| `log.file` | `bool` | `true` | Log output to file |
| `email.server` | `str` | `"smtp.qq.com"` | SMTP server for email alerts |
| `email.port` | `int` | `465` | SMTP port |
| `email.username` | `str` | `""` | SMTP username |
| `email.password` | `str` | `""` | SMTP password |
| `email.sender` | `str` | `""` | Sender email address |
| `email.receiver` | `str` | `""` | Receiver email address |
| `datafeed.name` | `str` | `""` | Datafeed module name (e.g., `"vnpy_rqdata"`) |
| `datafeed.username` | `str` | `""` | Datafeed account username |
| `datafeed.password` | `str` | `""` | Datafeed account password |
| `database.timezone` | `str` | *(auto-detected)* | Timezone for database timestamps (via `tzlocal`) |
| `database.name` | `str` | `"sqlite"` | Database backend (`"sqlite"`, `"mysql"`, `"postgresql"`, `"mongodb"`, `"influxdb"`) |
| `database.database` | `str` | `"database.db"` | Database name or file path |
| `database.host` | `str` | `""` | Database host (empty for SQLite) |
| `database.port` | `int` | `0` | Database port (0 for SQLite) |
| `database.user` | `str` | `""` | Database username |
| `database.password` | `str` | `""` | Database password |

Example `vt_setting.json`:

```json
{
    "log.level": 10,
    "database.name": "postgresql",
    "database.host": "localhost",
    "database.port": 5432,
    "database.database": "vnpy",
    "database.user": "trader",
    "database.password": "secret",
    "datafeed.name": "vnpy_rqdata",
    "datafeed.username": "myuser",
    "datafeed.password": "mytoken"
}
```

## Troubleshooting

### 1. `ModuleNotFoundError: No module named 'vnpy_ctp'` (or other `vnpy_*` module)

Gateway and app modules are distributed as separate packages. Install the one you need:
```bash
pip install vnpy_ctp vnpy_ctastrategy
```

### 2. TA-Lib installation fails

TA-Lib requires a system-level C library before the Python wrapper can be installed:
```bash
# Ubuntu/Debian
sudo apt-get install -y libta-lib-dev

# macOS
brew install ta-lib

# Then install the Python wrapper
pip install ta-lib
```

### 3. PySide6 / Qt platform plugin errors

If you see `qt.qpa.plugin: Could not find the Qt platform plugin`:
```bash
# Linux: install platform dependencies
sudo apt-get install -y libxcb-xinerama0 libegl1

# Or run headless (no GUI):
export QT_QPA_PLATFORM=offscreen
```

### 4. `.vntrader` directory not found or settings not loading

VeighNa looks for `.vntrader/` in the current working directory first, then falls back to `~/.vntrader/`. Ensure you run your scripts from the project root, or create the directory manually:
```bash
mkdir -p ~/.vntrader
echo '{}' > ~/.vntrader/vt_setting.json
```

### 5. EventEngine events not being received

Events are processed asynchronously in a background thread. After calling `event_engine.put()`, allow time for dispatch:
```python
import time
event_engine.put(Event("test", data))
time.sleep(0.5)  # wait for async processing
```
Always call `event_engine.start()` before putting events and `event_engine.stop()` when done.

### 6. Gateway connection timeout or authentication failure

- Verify credentials in the gateway's `default_setting` dict
- Check network connectivity to the exchange (CTP uses specific IPs/ports)
- Ensure your IP is whitelisted on the exchange
- For CTP test environment, use `vnpy_ctptest` instead of `vnpy_ctp`

### 7. `ImportError` with `polars` or `lightgbm` when using Alpha module

The Alpha module requires optional dependencies:
```bash
pip install -e ".[alpha]"
# or
uv sync --active --all-extras
```

### 8. Database timezone mismatch

If bar timestamps appear shifted, check that `database.timezone` in `vt_setting.json` matches your data source. The default uses the system timezone detected by `tzlocal`.

### 9. Python version incompatibility

VeighNa 4.x requires exactly Python 3.13.x (`requires-python = "==3.13.*"`). Other versions will fail to install.

### 10. Encoding errors on Windows with Chinese characters

Ensure your terminal and Python environment support UTF-8:
```bash
set PYTHONIOENCODING=utf-8
chcp 65001
```

## Security Considerations

### API Key Management

- **Never commit credentials** -- Gateway API keys, secrets, and passwords should never be checked into version control. Add `.vntrader/` to `.gitignore`.
- **Use environment variables** -- For production deployments, consider loading credentials from environment variables rather than `vt_setting.json`:
  ```python
  import os
  setting = {
      "api_key": os.environ["EXCHANGE_API_KEY"],
      "api_secret": os.environ["EXCHANGE_API_SECRET"],
  }
  gateway.connect(setting)
  ```
- **CTP credentials** -- CTP broker credentials (BrokerID, UserID, Password, AuthCode) grant full trading access. Treat them like banking credentials.

### Credential Storage

- `vt_setting.json` stores credentials in **plaintext JSON**. Restrict file permissions:
  ```bash
  chmod 600 ~/.vntrader/vt_setting.json
  ```
- Gateway connection settings (API keys for each exchange) are stored in separate JSON files within `.vntrader/`. Apply the same file permission restrictions.
- The email password in `vt_setting.json` is also stored in plaintext -- use app-specific passwords where possible.

### Network Security

- **RPC framework** -- The ZeroMQ-based RPC server/client (`vnpy.rpc`) does not provide built-in encryption or authentication. Only run RPC on trusted networks or behind a firewall.
- **Exchange connections** -- CTP gateways use proprietary TCP protocols. Ensure your firewall allows outbound connections to broker-provided IPs and ports.
- **Interactive Brokers** -- IB Gateway/TWS should be configured to accept connections only from localhost (`127.0.0.1`).
- **Web interface** -- If using `vnpy_webtrader`, bind to `127.0.0.1` in production and place behind a reverse proxy with TLS.

### Database Security

- SQLite databases (default) have no built-in access control. Protect the database file with filesystem permissions.
- For PostgreSQL/MySQL/MongoDB backends, use dedicated database users with minimal required privileges and enable TLS for remote connections.
- Database passwords in `vt_setting.json` are stored in plaintext -- restrict access to the configuration file.

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [Workflow](workflow.md) — Event flows and processing pipelines
- [State Management](state-management.md) — State lifecycle and data models
