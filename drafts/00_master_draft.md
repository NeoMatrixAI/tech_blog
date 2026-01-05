# AIFAP-BT Technical Blog Master Draft

> This document serves as the comprehensive reference for generating blog posts about the AIFAP-BT infrastructure system. LLM should read this document before writing any blog content.

**Last Updated**: 2025-12-19
**Version**: v01.00.06
**Branch**: feature/server

---

## 1. Executive Summary

**AIFAP-BT** (AI Finance Automated Platform - Bitget) is a Kubernetes-native cryptocurrency automated trading system built for the Bitget exchange. It implements portfolio rebalancing strategies with real-time monitoring, multi-channel alerting, and comprehensive performance tracking.

### Key Value Propositions
- **Automated Rebalancing**: Executes portfolio rebalancing at configurable intervals
- **Real-time Monitoring**: Tracks PV, ROE, Sharpe Ratio, Max Drawdown via Grafana
- **Multi-channel Alerts**: Slack for errors, Discord for user notifications
- **Cloud-Native**: Kubernetes deployment with Istio service mesh
- **Extensible**: User-defined strategies with Zipline-compatible interface

### Tech Stack Overview
| Layer | Technology |
|-------|------------|
| Language | Python 3.8.13 |
| API Framework | FastAPI + Uvicorn + Gunicorn |
| Container | Docker, Kubernetes |
| Service Mesh | Istio (Gateway, VirtualService) |
| Database | PostgreSQL (SQLAlchemy) |
| Monitoring | Grafana + Custom Finwatcher |
| Data Analysis | Zipline (customized), Pyfolio, Empyrical |
| Exchange | Bitget API v3 |

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
                                    ┌─────────────────────────────────────┐
                                    │           External APIs             │
                                    │  ┌─────────┐ ┌─────────┐ ┌───────┐ │
                                    │  │ Bitget  │ │  Auth   │ │RQuery │ │
                                    │  │ Trader  │ │   API   │ │  API  │ │
                                    │  └────┬────┘ └────┬────┘ └───┬───┘ │
                                    └───────┼──────────┼──────────┼─────┘
                                            │          │          │
┌───────────────────────────────────────────┼──────────┼──────────┼─────────────────┐
│                              Kubernetes Cluster (fin-finplatform)                  │
│                                           │          │          │                  │
│  ┌────────────────────────────────────────┴──────────┴──────────┴───────────────┐ │
│  │                         Istio Gateway + VirtualService                        │ │
│  │                     Host: aifapbt.fin.cloud.ainode.ai                         │ │
│  └───────────────────────────────────┬───────────────────────────────────────────┘ │
│                                      │                                             │
│  ┌───────────────────────────────────┴───────────────────────────────────────────┐ │
│  │                         AIFAP-BT API Pod (32Gi, 2 CPU)                        │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐  │ │
│  │  │                    FastAPI Server (Port 8888)                           │  │ │
│  │  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐            │  │ │
│  │  │  │ /command  │  │ /upload   │  │ /logging  │  │  /temp    │            │  │ │
│  │  │  │  router   │  │  router   │  │  router   │  │  router   │            │  │ │
│  │  │  └─────┬─────┘  └───────────┘  └───────────┘  └───────────┘            │  │ │
│  │  │        │                                                                │  │ │
│  │  │        │ subprocess.Popen()                                             │  │ │
│  │  │        ▼                                                                │  │ │
│  │  │  ┌─────────────────────────────────────────────────────────────────┐   │  │ │
│  │  │  │              Rebalancing System (Per User Session)              │   │  │ │
│  │  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │  │ │
│  │  │  │  │   Strategy  │  │    Main     │  │      Finwatcher         │  │   │  │ │
│  │  │  │  │   Process   │◄─┤   Process   │  │       Process           │  │   │  │ │
│  │  │  │  │             │  │             │  │                         │  │   │  │ │
│  │  │  │  │execute_     │  │auto_trade() │  │StackFuturesResult       │  │   │  │ │
│  │  │  │  │strategy()   │  │run_trading()│  │StackFuturesPfResult     │  │   │  │ │
│  │  │  │  │             │  │             │  │StackFuturesIndicator    │  │   │  │ │
│  │  │  │  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘  │   │  │ │
│  │  │  │         │                │                     │                │   │  │ │
│  │  │  │         └────────────────┼─────────────────────┘                │   │  │ │
│  │  │  │                          │                                      │   │  │ │
│  │  │  │                   order_info.txt (FileLock)                     │   │  │ │
│  │  │  └─────────────────────────────────────────────────────────────────┘   │  │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                               │ │
│  │  ┌──────────────────────┐                                                     │ │
│  │  │   Dashboard Server   │  Port 8080 (http.server)                            │ │
│  │  │   /dashboard/*       │  Grafana integration HTML                           │ │
│  │  └──────────────────────┘                                                     │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                    │
│  ┌────────────────────────────────────────────────────────────────────────────┐   │
│  │                              Database Layer                                 │   │
│  │  ┌─────────────────────────┐    ┌─────────────────────────────┐            │   │
│  │  │     finplatform DB      │    │       finwatcher DB         │            │   │
│  │  │  (Auth & Config)        │    │    (Trading Results)        │            │   │
│  │  │                         │    │                             │            │   │
│  │  │  - user_details         │    │  - futures_result_1min      │            │   │
│  │  │  - permissions          │    │  - futures_portfolio_result │            │   │
│  │  │  - active_sessions      │    │  - futures_indicator_result │            │   │
│  │  └─────────────────────────┘    └─────────────────────────────┘            │   │
│  └────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                    │
│  ┌────────────────────────────────────────────────────────────────────────────┐   │
│  │                           Alert System                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │   │
│  │  │   Slack Alert   │  │  Discord Alert  │  │  CronJob Alert  │             │   │
│  │  │  (Error Alerts) │  │ (User Notifs)   │  │ (Auto-liquidation)│           │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘             │   │
│  └────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                    │
│  ┌────────────────────────────────────────────────────────────────────────────┐   │
│  │                         Grafana Monitoring                                  │   │
│  │                   Host: monitoring.fin.cloud.ainode.ai                      │   │
│  └────────────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Process Synchronization (Event-Based)

```
┌─────────────────────┐                              ┌─────────────────────┐
│    Main Process     │                              │  Strategy Process   │
│   (run_trading)     │                              │   (run_strategy)    │
├─────────────────────┤                              ├─────────────────────┤
│                     │                              │                     │
│  1. Wait for        │   rebalance_start_event      │  1. Wait for        │
│     strategy_done   │ ────────────────────────────►│     rebalance_start │
│                     │                              │                     │
│                     │                              │  2. execute_        │
│                     │                              │     strategy()      │
│                     │                              │                     │
│                     │   strategy_done_event        │  3. Write to        │
│  2. Read order_info │ ◄────────────────────────────│     order_info.txt  │
│                     │                              │                     │
│  3. auto_trade()    │                              │                     │
│     - batch orders  │                              │                     │
│     - set SL/TP     │                              │                     │
│                     │                              │                     │
│                     │   autotrade_done_event       │                     │
│  4. Signal done     │ ────────────────────────────►│  4. Wait for next   │
│                     │                              │     rebalancing     │
│                     │   rebalance_done_event       │     interval        │
│  5. Wait for next   │ ◄────────────────────────────│                     │
│     cycle           │                              │  5. Trigger         │
│                     │                              │     rebalance_start │
└─────────────────────┘                              └─────────────────────┘

┌─────────────────────┐
│  Finwatcher Process │
├─────────────────────┤
│  - Runs every minute│
│  - Independent of   │
│    trading cycle    │
│  - Saves results to │
│    finwatcher DB    │
└─────────────────────┘
```

### 2.3 Data Flow

```
User Strategy File                 Bitget Exchange
       │                                 │
       ▼                                 ▼
┌─────────────────┐            ┌─────────────────┐
│ LiveDataContext │◄───────────│  OHLCV Data     │
│                 │  fetch     │  (1m candles)   │
└────────┬────────┘            └─────────────────┘
         │
         │ get_history()
         ▼
┌─────────────────┐
│ User Strategy   │
│ strategy(ctx,   │
│   config_dict)  │
└────────┬────────┘
         │
         │ returns {symbol: {weight, SL, TP}}
         ▼
┌─────────────────┐            ┌─────────────────┐
│ Weight Calc     │            │ Current Position│
│ (process_symbol)│◄───────────│ from Bitget     │
└────────┬────────┘            └─────────────────┘
         │
         │ diff_weight = new_weight - current_weight
         ▼
┌─────────────────┐
│ Order Info      │
│ order_info.txt  │
│ (with FileLock) │
└────────┬────────┘
         │
         │ read by auto_trade()
         ▼
┌─────────────────┐            ┌─────────────────┐
│ Order Execution │───────────►│  Bitget API     │
│ (batch of 5)    │  place     │  place_order()  │
│ + SL/TP setup   │  orders    │  place_sltp()   │
└────────┬────────┘            └─────────────────┘
         │
         │ results
         ▼
┌─────────────────┐            ┌─────────────────┐
│ Finwatcher      │───────────►│  PostgreSQL     │
│ (every minute)  │   save     │  finwatcher DB  │
└─────────────────┘            └─────────────────┘
```

---

## 3. Component Details

### 3.1 Trading Engine

#### Main Entry Point (`futures_rebalancing_main.py`)

**Key Functions**:

```python
def auto_trade(api_client, user_id, ...):
    """
    Executes orders from order_info.txt
    - Reads order info with FileLock
    - Sorts: close orders first, then open orders
    - Batches orders in groups of 5 (API rate limit)
    - Sets SL/TP after each order
    - Tracks failed orders for retry
    """

def run_trading(api_client, events, ...):
    """
    Main event loop
    - Waits for strategy_done_event
    - Calls auto_trade()
    - Signals autotrade_done_event
    - Checks trading_hours for termination
    """

def run_finwatcher(user_id, ...):
    """
    Monitoring process (runs independently)
    - Executes at second 0 of each minute
    - Calls StackFuturesResult, StackFuturesPfResult, etc.
    """
```

**Process Management**:
```python
# Main process spawning
strategy_process = multiprocessing.Process(target=run_strategy, ...)
finwatcher_process = multiprocessing.Process(target=run_finwatcher, ...)

strategy_process.start()
finwatcher_process.start()

run_trading(...)  # Main loop in parent process
```

#### Strategy Execution (`futures_rebalancing_strategy.py`)

**Key Functions**:

```python
def execute_strategy(api_client, strategy_module, config, ...):
    """
    Core strategy execution
    1. Load user strategy dynamically
    2. Create LiveDataContext for market data
    3. Call user's strategy(context, config_dict)
    4. Process each symbol with ThreadPoolExecutor
    5. Calculate weight differences
    6. Generate order info
    """

def process_symbol(symbol, strategy_result, current_positions, ...):
    """
    Handles individual symbol processing
    Cases:
    - 'new': No existing position → open new
    - 'same': Same direction → increase/decrease size
    - 'opposite': Opposite direction → close then open new
    """
```

**Weight Calculation Logic**:
```python
# Current weight based on margin
current_weight = marginSize / allocated_balance

# New weight from strategy
new_weight = strategy_results[symbol]['weight']

# Difference determines action
diff_weight = new_weight - current_weight

if diff_weight > threshold:
    # Open or increase position
elif diff_weight < -threshold:
    # Reduce or close position
```

### 3.2 Data Pipeline

#### DataContext Interface (`module/data_context.py`)

```python
from abc import ABC, abstractmethod

class DataContext(ABC):
    """Abstract interface for strategy data access"""

    @property
    @abstractmethod
    def current_dt(self) -> datetime:
        """Current datetime"""
        pass

    @abstractmethod
    def get_history(self, assets: List[str], window: int,
                    frequency: str, fields: str) -> pd.DataFrame:
        """
        Get historical OHLCV data
        - assets: ['BTCUSDT', 'ETHUSDT']
        - window: number of bars
        - frequency: '1m', '5m', '1h', etc.
        - fields: 'close', 'open', 'high', 'low', 'volume'
        Returns: Zipline-compatible DataFrame
        """
        pass
```

#### Live Implementation (`module/live_data_context.py`)

```python
class LiveDataContext(DataContext):
    """Real trading data context"""

    def __init__(self, data_client, tz_str='Asia/Seoul'):
        self.data_client = data_client
        self.tz = pytz.timezone(tz_str)

    @property
    def current_dt(self):
        return datetime.now(self.tz)

    def get_history(self, assets, window, frequency, fields):
        """
        Fetches OHLCV from Bitget API via data_client
        Returns DataFrame with MultiIndex columns (symbol, field)
        """
        # Fetch from Bitget
        raw_data = self.data_client.fetch_ohlcv(assets, window, frequency)
        # Transform to Zipline format
        return self._transform_to_zipline_format(raw_data, fields)
```

### 3.3 Monitoring (Finwatcher)

#### Performance Tracking (`monitoring/futures_finwatcher.py`)

**Data Collection Classes**:

| Class | Table | Metrics |
|-------|-------|---------|
| `StackFuturesResult` | `futures_result_1min` | Per-symbol: PV, marginSize, unrealized_pnl, ROE |
| `StackFuturesPfResult` | `futures_portfolio_result_1min` | Portfolio: total PV, netprofit, cash, available |
| `StackFuturesIndicatorResult` | `futures_indicator_result_1min_1day` | Indicators: Sharpe, MaxDD, Win Rate |
| `StackFuturesHistoricalTx` | `futures_historical_transactions_details` | Trade history |
| `StackFuturesHistoricalPos` | `futures_historical_positions_info` | Position history |

**Key Metrics**:

```python
# Portfolio Value
current_pv = account_balance + unrealized_pnl

# Net Profit
netprofit = current_pv - initial_balance

# Return on Equity (improved calculation)
roe = netprofit / available  # Using actual deployed capital

# Performance Indicators (from metrics.py)
sharpe_ratio = empyrical.sharpe_ratio(returns)
max_drawdown = empyrical.max_drawdown(returns)
win_rate = winning_trades / total_trades
```

**Database Schema** (column_id mapping):
```
futures_portfolio_result_1min:
  column_id 1: current_pv
  column_id 2: netprofit
  column_id 3: roe
  column_id 4: position_value
  column_id 5: cash
  column_id 8: unrealized_pnl
  column_id 9: unrealized_roe
  column_id 10: marginSize
  column_id 11: available (NEW in v01.00.06)
```

### 3.4 API Layer

#### Command Router (`api/routers/command.py`)

**Endpoints**:

```python
@router.post("/run-system")
async def run_system(request: RunSystemRequest):
    """
    Start trading session
    Request: {
        "tradeType": "futures",      # futures | spot
        "strategy_name": "my_strat",
        "trade_mode": "rebalancing", # rebalancing | general | volatility_targeting
        "trade_env": "live"          # live | demo
    }
    Response: {
        "message": "AIFAP-BT running (session ID: 2412091530)",
        "session_id": "2412091530",
        "dashboard_url": "https://..."
    }
    """
    # Spawns subprocess running futures_rebalancing_main.py
    process = subprocess.Popen([
        'python', 'futures_rebalancing_main.py',
        '--user_key', user_key,
        '--strategy_name', strategy_name,
        '--trade_env', trade_env
    ])

@router.get("/session-check")
async def session_check():
    """List active trading sessions"""

@router.get("/terminate")
async def terminate(session_id: str):
    """
    Terminate trading session
    - Sends SIGTERM to process
    - Triggers position cleanup
    """
```

### 3.5 Alert System

#### Slack Alert Module (`module/slack_alert.py`)

```python
def send_slack_alert(user_id: str, error_type: str,
                     error_msg: str, module_name: str = "system"):
    """
    Send error alert to Slack channel
    Format:
    [Futures] {error_type} ({module_name})
    > Time: 2025-12-19 10:35 +0900
    > User: {user_id}
    > Module: {module_name}
    [Stack trace]
    """

def send_slack_start_notification(user_id: str, strategy_name: str,
                                   trade_env: str, trading_hours: float):
    """
    Send system start notification
    Format:
    [Futures] System Started
    > Time: 2025-12-19 10:30 +0900
    > User: {user_id}
    > Strategy: {strategy_name}
    > Environment: {trade_env}
    > Duration: {trading_hours}h
    """
```

**Error Handling Philosophy**:
```python
# In all major processes:
try:
    # Main logic
except Exception as e:
    send_slack_alert(user_id, "Critical Error", str(e), "module_name")
    sys.exit(1)  # All errors are critical after retry exhaustion
```

---

## 4. Infrastructure

### 4.1 Kubernetes Setup

#### Current Architecture (v1.00.x)

**Deployment** (`services/deploy/container/deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dep-fin-aifap-bt-api
  namespace: fin-finplatform
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: fin-aifap-bt-api
        image: registry.gitlab.com/01ai/eng/finplatform/aifap_bt:aifap_bt-feat-v01.00.06
        resources:
          requests:
            memory: "8Gi"
            cpu: "500m"
          limits:
            memory: "32Gi"
            cpu: "2"
        ports:
        - containerPort: 8888  # API
        - containerPort: 8080  # Dashboard
        volumeMounts:
        - name: user-storage
          mountPath: /app/user/
```

**Service**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: srv-fin-aifap-bt-api
spec:
  ports:
  - name: api
    port: 8000
    targetPort: 8888
  - name: dashboard
    port: 8080
    targetPort: 8080
```

#### Planned Architecture (v1.01.x - CronJob + Dynamic Job)

```
Memory Efficiency: 32Gi (v1.00.x) → ~4Gi dynamic (87% reduction)

┌─────────────────────────────────────────────────────────────┐
│  CronJob: Scheduler (* * * * *)                             │
│  - Checks for rebalancing targets                           │
│  - Creates Strategy-Trading Jobs 1 minute before            │
│  - Memory: 128Mi~256Mi, CPU: 100m~250m                      │
│  - Duration: 5-10 seconds                                   │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ Creates
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Job: Strategy-Trading-{user_id}-{timestamp}                │
│  - Sleeps until exact rebalancing time                      │
│  - Executes strategy + auto_trade (0 delay)                 │
│  - Self-terminates after completion                         │
│  - Memory: 512Mi~2Gi, CPU: 250m~1                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  CronJob: Finwatcher (* * * * *)                            │
│  - Runs every minute                                        │
│  - Processes all active users                               │
│  - Exits immediately if no active users                     │
│  - Memory: 512Mi~2Gi, CPU: 250m~1                           │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Database Design

**finplatform DB** (PostgreSQL):
- Host: `srv-finplatform-postgres.fin-finplatform.svc.cluster.local:5432`
- Tables: user_details, permissions, active_sessions

**finwatcher DB** (PostgreSQL):
- Host: `srv-finwatcher-postgres.fin-finwatcher.svc.cluster.local:5432`
- Tables: futures_result_1min, futures_portfolio_result_1min, etc.

### 4.3 Networking (Istio)

**Gateway**:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: gw-fin-aifap-bt-api
spec:
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: cert-fin-aifap-bt-api
    hosts:
    - aifapbt.fin.cloud.ainode.ai
```

**VirtualService**:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
spec:
  hosts:
  - aifapbt.fin.cloud.ainode.ai
  http:
  - match:
    - uri:
        prefix: /dashboard/
    route:
    - destination:
        port:
          number: 8080
  - route:
    - destination:
        port:
          number: 8000
```

### 4.4 CI/CD (GitLab Registry)

**Build Process**:
```bash
# Registry
REPO=registry.gitlab.com/01ai/eng/finplatform/aifap_bt
VERSION=v01.00.06

# Build
docker buildx build \
  -f server/services/deploy/image-build/dockerfile \
  -t ${REPO}:aifap_bt-feat-${VERSION} .

# Push
docker push ${REPO}:aifap_bt-feat-${VERSION}

# Deploy (update deployment.yaml image tag)
kubectl apply -f server/services/deploy/container/
```

---

## 5. Recent Updates

### 5.1 Alert System Overhaul (v01.00.06)

**Changes**:
1. **Modularized Slack Functions**
   - Created `module/slack_alert.py`
   - Moved `send_slack_alert()` from finwatcher
   - Added `send_slack_start_notification()`

2. **System Start Notifications**
   - Sends Slack message when trading session starts
   - Includes: user, strategy, environment, duration

3. **Error Alerts for All Processes**
   - `auto_trade()`: Order execution errors
   - `run_trading()`: Critical errors
   - `run_strategy()`: Strategy execution errors
   - `run_finwatcher()`: All stack classes

4. **Discord Alert Deployment**
   - New K8s manifests for Discord CronJob
   - User position alerts

### 5.2 Finwatcher Improvements

**Changes**:
1. **Added `available` Field**
   - Tracks actual available capital
   - Used for accurate ROE calculation
   - New column_id 11 in DB

2. **Simplified Error Handling**
   - Removed RECOVERABLE_ERRORS concept
   - All errors after retry = critical
   - System stops on any error

3. **ROE Calculation Fix**
   - Old: `roe = netprofit / current_pv`
   - New: `roe = netprofit / available`
   - More accurate reflection of actual returns

### 5.3 Code Quality Improvements

- Strategy skip logging with position details
- Snapshot function returns closed_symbols
- Better error messages with context

---

## 6. Upcoming Features

### v1.01.x Architecture Migration
- CronJob-based scheduler
- Dynamic Job creation
- 87% memory reduction
- Per-user isolation

### Enhanced Monitoring
- More granular metrics
- Better Grafana dashboards
- Historical comparison views

---

## 7. Code Snippets for Blog

### 7.1 Weight Calculation
```python
def calculate_order_info(symbol, strategy_result, current_position, allocated_balance):
    """Calculate order based on weight difference"""
    new_weight = strategy_result['weight']

    if current_position:
        current_weight = current_position['marginSize'] / allocated_balance
        diff_weight = new_weight - current_weight

        if abs(diff_weight) < THRESHOLD:
            return None  # Skip small adjustments

        if diff_weight > 0:
            # Increase position
            return {'action': 'open', 'size': diff_weight * allocated_balance}
        else:
            # Decrease position
            return {'action': 'reduce', 'size': abs(diff_weight) * allocated_balance}
    else:
        # New position
        return {'action': 'open', 'size': new_weight * allocated_balance}
```

### 7.2 Event-Based Synchronization
```python
import multiprocessing

# Shared events
rebalance_start_event = multiprocessing.Event()
strategy_done_event = multiprocessing.Event()
autotrade_done_event = multiprocessing.Event()
rebalance_done_event = multiprocessing.Event()

def run_strategy(events):
    while True:
        events['rebalance_start'].wait()
        events['rebalance_start'].clear()

        execute_strategy()

        events['strategy_done'].set()
        events['autotrade_done'].wait()
        events['autotrade_done'].clear()

        time.sleep(rebalancing_interval)
        events['rebalance_done'].set()

def run_trading(events):
    while True:
        events['strategy_done'].wait()
        events['strategy_done'].clear()

        auto_trade()

        events['autotrade_done'].set()
        events['rebalance_done'].wait()
        events['rebalance_done'].clear()
```

### 7.3 API Request Example
```python
import requests

# Start trading session
response = requests.post(
    'https://aifapbt.fin.cloud.ainode.ai/command/run-system',
    headers={'Authorization': f'Bearer {token}'},
    json={
        'tradeType': 'futures',
        'strategy_name': 'momentum_strategy',
        'trade_mode': 'rebalancing',
        'trade_env': 'live'
    }
)

result = response.json()
# {
#     "message": "AIFAP-BT running (session ID: 2412191030)",
#     "session_id": "2412191030",
#     "dashboard_url": "https://monitoring.fin.cloud.ainode.ai/..."
# }
```

### 7.4 User Strategy Template
```python
def strategy(context, config_dict):
    """
    User-defined rebalancing strategy

    Args:
        context: DataContext with get_history() method
        config_dict: Strategy configuration

    Returns:
        dict: {symbol: {'weight': float, 'presetStopSurplusPrice': float, 'presetStopLossPrice': float}}
    """
    assets = ['BTCUSDT', 'ETHUSDT', 'SOLUSDT']

    # Get 100 bars of 1-minute close prices
    prices = context.get_history(assets, window=100, frequency='1m', fields='close')

    # Calculate momentum (example)
    returns = prices.pct_change(20)  # 20-bar return

    # Equal weight to positive momentum assets
    weights = {}
    positive_assets = [a for a in assets if returns[a].iloc[-1] > 0]

    if positive_assets:
        weight_per_asset = 1.0 / len(positive_assets)
        for asset in positive_assets:
            current_price = prices[asset].iloc[-1]
            weights[asset] = {
                'weight': weight_per_asset,
                'presetStopSurplusPrice': current_price * 1.05,  # 5% TP
                'presetStopLossPrice': current_price * 0.97     # 3% SL
            }

    return weights
```

---

## 8. External Resources

### API Documentation
- **BitgetTrader**: https://bitgettrader.fin.cloud.ainode.ai
- **Auth API**: https://auth.fin.cloud.ainode.ai
- **RQuery API**: https://rquery.fin.cloud.ainode.ai

### Monitoring
- **Grafana**: https://monitoring.fin.cloud.ainode.ai

### Repository
- **GitLab Registry**: registry.gitlab.com/01ai/eng/finplatform/aifap_bt

---

## 9. Glossary

| Term | Description |
|------|-------------|
| **Rebalancing** | Adjusting portfolio weights to match target allocation |
| **Weight** | Proportion of portfolio allocated to an asset |
| **PV** | Portfolio Value - total account value |
| **ROE** | Return on Equity - netprofit / invested capital |
| **Sharpe Ratio** | Risk-adjusted return metric |
| **Max Drawdown** | Maximum peak-to-trough decline |
| **SL/TP** | Stop Loss / Take Profit orders |
| **Finwatcher** | Custom monitoring system for tracking performance |

---

*This document should be updated whenever significant changes are made to the AIFAP-BT system.*
