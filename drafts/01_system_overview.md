# AIFAP: A Kubernetes-Native Crypto Trading Platform Architecture

> Building a production-grade automated trading system from scratch

**Category**: System Update
**Reading Time**: ~10 minutes
**Tags**: #kubernetes #python #trading #fintech #devops #cryptocurrency

---

## Introduction

When I started building an automated cryptocurrency trading platform, I had a simple goal: create a system that could execute portfolio rebalancing strategies reliably, 24/7, without manual intervention. But as the project evolved, a bigger vision emerged—building a **Multi-Manager System** where multiple managers can each oversee multiple traders, all coordinated under a single supervisor.

In this post, I'll walk you through the architecture of **AIFAP** (AI Finance Agent Platform), a Kubernetes-native trading system that I've been developing and running in production. Currently integrated with Bitget, with plans to support Binance, Bybit, and other major exchanges. This first post covers the high-level architecture and design decisions. Future posts will dive deep into specific components.

### The Multi-Manager Vision

The ultimate goal of AIFAP extends beyond a single trading bot:

- **Supervisor (Admin)**: Top-level administrator who manages the entire system
- **Managers**: Hired by the supervisor, each manager oversees their own team of traders
- **Traders**: Execute individual strategies under a manager's supervision

Think of it like a hedge fund structure: one fund (supervisor) employs multiple portfolio managers, and each manager runs several trading strategies. AIFAP aims to automate this entire hierarchy.

---

## What Does AIFAP Do?

At its core, AIFAP is a **portfolio rebalancing system** for cryptocurrency futures and spot trading. Currently running on Bitget, with support for additional exchanges coming soon. Here's what that means in practice:

1. **User-Defined Strategies**: Users write Python functions that analyze market data and output target portfolio weights
2. **Automated Rebalancing**: The system periodically compares current positions against target weights and executes trades to match
3. **Real-Time Monitoring**: Every minute, performance metrics are calculated and stored for visualization
4. **Discord Alerts**: Errors and important events trigger notifications via Discord

### A Simple Example

A user might define a momentum strategy like this:

```python
def strategy(context, config_dict):
    """Simple momentum strategy"""
    assets = ['BTCUSDT', 'ETHUSDT', 'SOLUSDT']

    # Get 20-bar returns
    prices = context.get_history(assets, window=100, frequency='1m', fields='close')
    returns = prices.pct_change(20)

    # Equal weight to positive momentum assets
    weights = {}
    positive = [a for a in assets if returns[a].iloc[-1] > 0]

    if positive:
        for asset in positive:
            weights[asset] = {
                'weight': 1.0 / len(positive),
                'presetStopSurplusPrice': prices[asset].iloc[-1] * 1.05,
                'presetStopLossPrice': prices[asset].iloc[-1] * 0.97
            }

    return weights
```

The system handles everything else: fetching market data, calculating position differences, executing orders, setting stop-losses, and tracking performance.

---

## Architecture Overview

Let me show you how this all fits together.

![System_Architecture](../images/01_system_overview/01_system_architecture.png)
<!-- [IMAGE NEEDED]
- Filename: 01_system_architecture
- Type: System Architecture Diagram
- Purpose: Show the complete AIFAP system structure with Kubernetes namespaces, cross-namespace communication, and data flow
- Content:
  - Layer 1 (Top): "External APIs" region
    - "Bitget Exchange API" (orange, #FF6B35) - external exchange, centered at top
  - Layer 2 (Main): "Kubernetes Cluster" large boundary box (dashed border)
    - **Namespace 1: finplatform** (blue boundary, #E3F2FD) - LEFT side
      - "Istio Gateway" box (purple, #9B59B6)
      - "Auth API Pod" box (blue, #4A90D9)
      - "AIFAP Pod" main box (largest component)
        - Inside Pod: "FastAPI Server" label at top
        - Inside Pod: "Trading Processes x3" label below
      - "ConfigMap" small box (gray)
    - **Namespace 2: fin-crypto** (orange boundary, #FFF3E0) - CENTER
      - "CronJob: OHLCV Collector" box (yellow, #FFC107) - small, shows scheduled task
        - Label: "Fetches real-time data from Bitget"
      - "CoinAPI Pod" box (orange, #FF6B35)
        - Label: "OHLCV Data Service (Serves data from DB)"
    - **Namespace 3: fin-finwatcher** (green boundary, #E8F5E9)
      - "RqueryAPI Pod" box (green, #2ECC71)
      - "PyfolioAPI Pod" box (teal, #26A69A)
      - Label: "Monitoring & Analysis Services"
    - **Namespace 4: fin-finwrapper** (red boundary, #FFEBEE) - RIGHT side, PROMINENT position
      - "BitgetTrader Pod" box (red, #E53935) - LARGER, emphasized as critical component
      - Label: "Exchange Gateway"
      - Sub-label: "Orders, Positions, Account Data"
  - Layer 3 (Bottom): "Data Layer" region
    - "PostgreSQL (finplatform)" cylinder icon (blue)
    - "PostgreSQL (fin-crypto)" cylinder icon (orange) - stores OHLCV data
    - "PostgreSQL (finwatcher)" cylinder icon (green)
    - "Grafana" dashboard icon (orange)
- Layout: Vertical 3-tier architecture, namespaces shown as grouped regions
- Connections:
  **Data Collection Flow (OHLCV) - numbered sequence:**
  - Bitget Exchange API → CronJob: one-way arrow DOWN (labeled "1. Fetch OHLCV")
  - CronJob → PostgreSQL (fin-crypto): one-way arrow DOWN (labeled "2. Store")
  - PostgreSQL (fin-crypto) → CoinAPI Pod: one-way arrow UP (labeled "3. Read") - DB serves data TO CoinAPI
  - AIFAP Pod → CoinAPI Pod: cross-namespace arrow LEFT-to-CENTER (labeled "4. Query OHLCV")
  **Exchange Communication Flow - CRITICAL PATH (use thicker/bolder arrows):**
  - AIFAP Pod ↔ BitgetTrader Pod: cross-namespace BIDIRECTIONAL arrow, THICK
    - To BitgetTrader: "Send Orders"
    - From BitgetTrader: "Positions, Balance, Trade Results"
  - BitgetTrader Pod ↔ Bitget Exchange API: BIDIRECTIONAL arrow, THICK
    - UP: "Execute Orders"
    - DOWN: "Positions, Account, Trade Data"
  - Note: BitgetTrader is the ONLY gateway to exchange - ALL exchange communication goes through it (orders, positions, balance, trade history)
  **Internal Communication:**
  - Auth API Pod ↔ AIFAP Pod: bidirectional arrow (authentication)
  - Istio Gateway → Auth API, AIFAP: routing arrows
  - AIFAP Pod ↔ PostgreSQL (finplatform): read/write (config, user data)
  - AIFAP Pod → RqueryAPI Pod: cross-namespace arrow (result query)
  - AIFAP Pod → PyfolioAPI Pod: cross-namespace arrow (performance analysis)
  - RqueryAPI/PyfolioAPI ↔ PostgreSQL (finwatcher): read/write
  - PostgreSQL (finwatcher) → Grafana: data query
- Style:
  - Background: light gray (#F5F5F5) or white
  - Namespace boundaries: colored dashed lines matching namespace theme
  - Kubernetes boundary: dark dashed line, subtle shadow
  - Cross-namespace arrows: thicker, labeled with "cross-ns"
  - Order execution arrows: EXTRA THICK (red or bold) to emphasize critical path
  - Data flow arrows: numbered (1, 2, 3, 4) to show sequence
  - CronJob: dashed border to indicate scheduled task
  - BitgetTrader: slightly larger box, red glow or emphasis to show importance
  - Arrows: solid lines with triangle heads
  - Font: Sans-serif (Inter, Roboto, or similar)
  - Component shadows: subtle drop shadow for depth
- Reference: AWS/GCP architecture diagram style, Kubernetes namespace visualization -->

### The Key Components

**Namespace: finplatform** (Core Trading Platform)
1. **Auth API**: User authentication and authorization
2. **AIFAP API**: Main trading service with FastAPI server
3. **Trading Processes**: Each user session spawns three coordinated processes

**Namespace: fin-crypto** (Market Data)
4. **CronJob (OHLCV Collector)**: Scheduled task that fetches real-time OHLCV data from Bitget and stores it in PostgreSQL
5. **CoinAPI**: OHLCV data service (Go-based) that reads from the database and serves data to AIFAP

**Namespace: fin-finwatcher** (Monitoring & Analytics)
6. **RqueryAPI**: Query trading results from Finwatcher database
7. **PyfolioAPI**: Performance analysis using Pyfolio library

**Namespace: fin-finwrapper** (Exchange Gateway)
8. **BitgetTrader API**: Gateway to Bitget exchange - handles order execution, real-time position queries, account balance, and trade history

**Infrastructure**
9. **PostgreSQL Databases**: Separate DBs for auth/config, OHLCV market data, and trading results
10. **Grafana**: Visualizes performance metrics in real-time
11. **Istio**: Handles TLS, routing, and cross-namespace traffic management

---

## The Three-Process Architecture

When a user starts a trading session, the system spawns three processes that work together:

![Threee_Process_Architecture](../images/01_system_overview/02_three_process_architecture.png)
<!-- [IMAGE NEEDED]
- Filename: 02_three_process_architecture
- Type: Process Interaction Diagram
- Purpose: Explain how the 3 processes communicate and their execution sequence
- Content:
  - Box 1 (Top-Left): "Strategy Process"
    - Background color: blue (#4A90D9)
    - Icon: brain or calculator symbol (top-right corner)
    - Internal steps (vertical list, numbered in circles):
      1. "Execute strategy.py"
      2. "Calculate target weights"
      3. "Write order_info.txt"
    - Bottom label: "Runs on schedule (e.g., every 5 min)"
  - Box 2 (Top-Right): "Main Process"
    - Background color: green (#2ECC71)
    - Icon: play/execute symbol (top-right corner)
    - Internal steps (vertical list, numbered in circles):
      1. "Wait for strategy_done_event"
      2. "Read order_info.txt"
      3. "Execute orders (batch of 5)"
      4. "Set SL/TP prices"
    - Bottom label: "Waits for signals"
  - Box 3 (Bottom-Center): "Finwatcher Process"
    - Background color: orange (#FF6B35)
    - Icon: chart/monitor symbol (top-right corner)
    - Internal content: "Every 60 seconds:"
      - "• Query account balance"
      - "• Calculate PV, ROE, metrics"
      - "• Save to PostgreSQL"
    - Bottom label: "Independent schedule"
- Layout:
  - Strategy and Main: top row, side by side with gap
  - Finwatcher: bottom row, centered, smaller width
  - Overall: inverted triangle arrangement
- Connections:
  - Strategy → Main: dashed arrow pointing right
    - Label above: "strategy_done_event"
  - Main → Strategy: dashed arrow pointing left
    - Label below: "autotrade_done_event"
  - Between arrows: label "multiprocessing.Event"
  - Finwatcher: no connection lines
    - Label nearby: "Independent (no sync needed)"
- Style:
  - Boxes: rounded corners (8px radius), subtle shadow
  - Step numbers: white circle with number inside
  - Arrows: dashed lines, medium thickness
  - Font: Sans-serif, step text 12-14px
  - Overall background: white or very light gray -->

### Why Three Processes?

1. **Separation of Concerns**: Strategy logic, order execution, and monitoring are isolated
2. **Fault Tolerance**: A crash in one process doesn't immediately kill others
3. **Timing Independence**: Finwatcher runs on its own schedule (every minute)

### Event-Based Synchronization

The Strategy and Main processes communicate through `multiprocessing.Event` objects:

```python
# Shared events between processes
rebalance_start_event = multiprocessing.Event()
strategy_done_event = multiprocessing.Event()
autotrade_done_event = multiprocessing.Event()
```

This pattern ensures strict sequencing: strategy calculates → main executes → strategy waits for next interval.

---

## The Tech Stack

Here's what powers AIFAP:

![Tech_Stack_Table](../images/01_system_overview/03_tech_stack_table.png)
<!-- [IMAGE NEEDED]
- Filename: 03_tech_stack_table
- Type: Tech Stack Table
- Purpose: Show all technologies used in AIFAP and why each was chosen
- Content (3 columns: Category | Technology | Why We Use It):
  | Category       | Technology            | Why We Use It                          |
  |----------------|----------------------|----------------------------------------|
  | Language       | Python 3.8           | Rich financial libraries (pandas, numpy) |
  | API Framework  | FastAPI + Uvicorn    | Async support, auto OpenAPI docs, fast |
  | Container      | Docker + Kubernetes  | Cloud-native, orchestration, scaling   |
  | Service Mesh   | Istio                | TLS termination, traffic routing       |
  | Database       | PostgreSQL           | Reliable, excellent for time-series    |
  | Monitoring     | Grafana              | Beautiful dashboards, alerting rules   |
  | Analysis       | Zipline (modified)   | Backtesting compatibility, context API |
- Layout:
  - Header row: dark background (navy #1A237E or dark gray #333)
  - Data rows: alternating white (#FFFFFF) and light gray (#F5F5F5)
  - First column: slightly darker background for emphasis
  - Total width: fill container, proportional columns (20% / 25% / 55%)
- Style:
  - Border: thin lines (#E0E0E0), rounded corners on outer edge
  - Header font: bold, white text
  - Data font: regular weight, dark gray (#333) text
  - Row height: comfortable padding (12-16px vertical)
  - Optional: small technology logo icons next to each name (Python snake, Docker whale, etc.)
  - Shadow: subtle drop shadow under entire table -->

### Why Kubernetes?

Running a trading system has unique requirements:

- **High Availability**: Can't afford downtime during market hours
- **Resource Isolation**: Each user session needs guaranteed resources
- **Easy Updates**: Rolling deployments without interrupting active sessions
- **Observability**: Built-in logging and metrics collection

### Namespace Separation

Services are deployed across four Kubernetes namespaces for isolation and organization:

- **finplatform**: Core trading services (Auth API, AIFAP API)
- **fin-crypto**: Market data services (CoinAPI for OHLCV data)
- **fin-finwatcher**: Monitoring and analytics (RqueryAPI, PyfolioAPI)
- **fin-finwrapper**: Order execution wrapper (BitgetTrader API)

This separation provides security boundaries between services and allows independent scaling and deployment of each service group.

---

## Data Flow: From Strategy to Execution

Let me trace the path of a single rebalancing cycle:

![Rebalancing_Data_Flow](../images/01_system_overview/04_rebalancing_data_flow.png)
<!-- [IMAGE NEEDED]
- Filename: 04_rebalancing_data_flow
- Type: Vertical Data Flow Diagram
- Purpose: Trace the complete path of a single rebalancing cycle from strategy to monitoring
- Content:
  - Start Point: oval shape "Rebalancing Cycle Start" (top)
  - Stage 1 "Strategy Execution" (blue region #E3F2FD):
    - Badge: circle with "1" on left side
    - Flow boxes (horizontal):
      - "Load strategy.py" → "Fetch OHLCV data" → "Calculate weights"
    - Output box (parallelogram): "{BTC: 0.5, ETH: 0.5}"
  - Stage 2 "Weight Comparison" (green region #E8F5E9):
    - Badge: circle with "2" on left side
    - Flow boxes:
      - "Get current positions" → "Calculate current weights" → "Compute difference"
    - Output box: "{BTC: +0.2, ETH: +0.3}"
  - Stage 3 "Order Generation" (yellow region #FFF8E1):
    - Badge: circle with "3" on left side
    - Flow boxes:
      - "Convert weights to sizes" → "Add SL/TP prices" → "Write order_info.txt"
  - Stage 4 "Order Execution" (orange region #FFF3E0):
    - Badge: circle with "4" on left side
    - Flow boxes:
      - "Read order_info.txt" → "Sort orders (close first)" → "Batch execute (5)" → "Set SL/TP"
  - Stage 5 "Monitoring" (purple region #F3E5F5):
    - Badge: circle with "5" on left side
    - Flow boxes:
      - "Query balance" → "Calculate PV, ROE" → "Save to PostgreSQL"
  - End Point: oval shape "Wait for next cycle" (bottom)
- Layout:
  - Direction: top to bottom (vertical)
  - Each stage: horizontal band with light background color
  - Stage badges: positioned on left edge, overlapping stage boundary
  - Flow boxes: arranged horizontally within each stage
  - Bold down arrows between stages
- Connections:
  - Within stage: thin arrows connecting boxes left to right
  - Between stages: bold arrows pointing down
  - Output boxes: dashed arrows to next stage input
- Style:
  - Stage backgrounds: pastel colors as specified above
  - Flow boxes: white background, rounded corners, thin border
  - Output boxes: parallelogram shape, slightly darker background
  - Badge circles: solid color matching stage, white number
  - Arrows: dark gray (#666), triangle heads
  - Overall: clean, professional flowchart style -->

### The Weight Calculation

The core logic is surprisingly simple:

```python
# What we have
current_weight = position_margin / total_allocation

# What we want
target_weight = strategy_results[symbol]['weight']

# What we need to do
diff_weight = target_weight - current_weight

if diff_weight > threshold:
    # Buy more
elif diff_weight < -threshold:
    # Sell some
```

The threshold prevents excessive trading for tiny differences.

---

## Monitoring: Every Minute Counts

The Finwatcher process runs independently, collecting metrics every minute:

![Metrics_Talbe](../images/01_system_overview/05_metrics_table.png)
<!-- [IMAGE NEEDED]
- Filename: 05_metrics_table
- Type: Metrics Definition Table
- Purpose: Explain each performance metric tracked by Finwatcher
- Content (3 columns: Metric | Formula | Description):
  | Metric          | Formula                           | Description                              |
  |-----------------|-----------------------------------|------------------------------------------|
  | PV              | Balance + Unrealized PnL          | Total portfolio value at current moment  |
  | ROE             | (PV - Initial) / Initial × 100%   | Return on equity since session start     |
  | Unrealized PnL  | Σ (Current Price - Entry Price) × Size | Paper gains/losses on open positions |
  | Sharpe Ratio    | (Return - Rf) / Std(Return)       | Risk-adjusted return (annualized)        |
  | Max Drawdown    | (Peak - Trough) / Peak × 100%     | Worst decline from peak to trough        |
- Layout:
  - Header row: dark blue (#1A237E) background, white bold text
  - Data rows: alternating white and light blue (#E3F2FD)
  - Column widths: 20% / 35% / 45%
  - Metric column: bold text for emphasis
- Style:
  - Border: thin (#BDBDBD), rounded corners (8px)
  - Font: Metric names in bold, formulas in monospace
  - Padding: 10-12px per cell
  - Shadow: subtle drop shadow -->

These metrics are stored in PostgreSQL and visualized in Grafana:

![Grafana_Dashboard](../images/01_system_overview/06_grafana_dashboard.png)
<!-- [IMAGE NEEDED]
- Filename: 06_grafana_dashboard
- Type: Grafana Dashboard Mockup
- Purpose: Show what the actual monitoring interface looks like
- Content:
  - Row 1 (Top): 3 Stat Panels (equal width, horizontal)
    - Panel 1: "ROE"
      - Value: "+12.5%"
      - Color: green (#00FF88)
      - Small up arrow icon
    - Panel 2: "Sharpe Ratio"
      - Value: "1.8"
      - Color: blue (#4A90D9)
      - No trend icon
    - Panel 3: "Max Drawdown"
      - Value: "-5.2%"
      - Color: red (#FF6B6B)
      - Small down arrow icon
  - Row 2 (Middle, largest): Time Series Line Chart
    - Title: "Portfolio Value Over Time"
    - X-axis: Date/time labels (Dec 15 - Dec 22)
    - Y-axis: Dollar values ($10,000 - $12,000)
    - Line: smooth curve showing upward trend with small fluctuations
    - Line color: neon green (#00FF88)
    - Fill: gradient from line color to transparent
    - Grid: subtle gray lines
  - Row 3 (Bottom): 2 charts side by side
    - Left: Pie Chart "Position Allocation"
      - BTC: 50% (orange #FF6B35)
      - ETH: 30% (blue #4A90D9)
      - SOL: 20% (purple #9B59B6)
      - Legend: below or right side
    - Right: Horizontal Bar Chart "Position Sizes (USDT)"
      - BTC: 5,000 USDT bar
      - ETH: 3,000 USDT bar
      - SOL: 2,000 USDT bar
      - Bars colored matching pie chart
- Layout:
  - Grid: 3 rows
  - Row heights: 1/6 (stats), 1/2 (main chart), 1/3 (bottom charts)
  - Gaps: 8-12px between panels
  - Card style: each panel in a rounded container
- Style:
  - Theme: Grafana Dark Mode
  - Background: #1A1A1A (overall), #2A2A2A (panel cards)
  - Text: white (#FFFFFF)
  - Panel borders: subtle #3A3A3A or none
  - Chart line: glow effect on main line
  - Time range selector: visible in top-right corner
- Reference: Grafana official demo dashboards, cryptocurrency trading dashboards -->

---

## Alert System: Never Miss a Problem

When something goes wrong, you need to know immediately. AIFAP sends real-time alerts through Discord, keeping you informed of both critical errors and trading activities.

![Discord_Alert_System](../images/01_system_overview/07_discord_alert_system.png)
<!-- [IMAGE NEEDED]
- Filename: 07_discord_alert_system
- Type: Discord Message Mockup / Alert Flow Diagram
- Purpose: Show the different types of Discord notifications and their visual appearance
- Content:
  - Left side: "Alert Types" vertical list with icons
    - 🚀 "System Start" (blue badge)
    - 🟢 "Position Opened/Added" (green badge)
    - 📝 "Position Updated" (yellow badge)
    - 🔴 "Position Closed" (red badge)
    - 🚨 "Critical Errors" (dark red badge)
  - Right side: Discord message mockups (5 examples stacked vertically)
    - Example 1: System Start (blue left border)
      ```
      🚀 [Futures] System Started
      Time: 2025-12-30 09:07 +0900
      User: trader_01
      Strategy: momentum_v2
      Environment: live
      Trading Duration: 720h 0m
      ```
    - Example 2: Position Opened (green left border)
      ```
      🟢 [Trader A] Position Added
           Sym  Side   Size       Entry      SL      TP FundingFee
      DOGEUSDT  long    133     0.12278 0.09824 0.12894
       LTCUSDT short    0.4    78.17000   93.91   74.35
       BTCUSDT  long 0.0004 87200.00000 69781.2 91587.8
      ```
    - Example 3: Position Updated (yellow left border)
      ```
      📝 [Trader A] Position Updated
           Sym  Side   Size   Entry      SL      TP              FundingFee
       XRPUSDT  long      3 1.85000  1.4801  1.9426  0.0001 -> 0.0001394625
      AVAXUSDT short    3.1 12.3670  14.836  11.745  0.0019 -> 0.0026564241
      ```
      Note: "->" shows value change within same column (old -> new)
    - Example 4: Position Closed (red left border)
      ```
      🔴 [Trader A] Position Closed: LTCUSDT
      🔴 [Trader A] Position Closed: DOGEUSDT, BNBUSDT
      ```
    - Example 5: Error Alert (dark red left border)
      ```
      🚨 [Futures] StackFuturesIndicatorResult Partial Failure (1)
      Time: 2026-01-05 03:33 +0900
      Module: StackFuturesIndicatorResult
      An issue occurred. Admin has been notified.
      ```
- Layout:
  - Two-column layout (30% / 70%)
  - Alert types list on left with colored badges
  - Discord mockups on right with colored left borders matching type
  - Arrow connecting alert type to corresponding example
  - Monospace font for position table data (aligned columns)
- Style:
  - Discord dark theme background (#36393F)
  - Message backgrounds: slightly lighter (#40444B)
  - Left border colors: blue (#5865F2) for start, green (#57F287) for opened, yellow (#FEE75C) for updated, red (#ED4245) for closed, dark red (#A12D2F) for errors
  - Text: white (#FFFFFF) for titles, gray (#B9BBBE) for body
  - Font: Discord-like sans-serif, monospace for table data
  - Timestamp: small gray text
  - Position table: fixed-width columns, right-aligned numbers
- Reference: Discord embed message style, trading terminal table format -->

### Discord (Primary Channel)

Discord is the main notification hub for AIFAP. It handles multiple types of real-time notifications:

**System Start** - Notification when trading session begins:
```
🚀 [Futures] System Started
Time: 2025-12-30 09:07 +0900
User: trader_01
Strategy: momentum_v2
Environment: live
Trading Duration: 720h 0m
```

**Position Opened** - New positions with entry, SL/TP details:
```
🟢 [Trader A] New Positions Opened
    Sym  Side Size  Entry SL    TP FundingFee
DOTUSDT short   43  1.823    1.733
```

**Position Added** - Additional positions to existing portfolio:
```
🟢 [Trader A] Position Added
     Sym  Side   Size       Entry      SL      TP FundingFee
DOGEUSDT  long    133     0.12278 0.09824 0.12894
 LTCUSDT short    0.4    78.17000   93.91   74.35
 BTCUSDT  long 0.0004 87200.00000 69781.2 91587.8
```

**Position Updated** - Changes to existing positions (SL/TP adjustments, funding fees):
```
📝 [Trader A] Position Updated
     Sym  Side   Size   Entry      SL      TP              FundingFee
 XRPUSDT  long      3 1.85000  1.4801  1.9426  0.0001 -> 0.0001394625
AVAXUSDT short    3.1 12.3670  14.836  11.745  0.0019 -> 0.0026564241
 DOTUSDT short     43  1.8230   2.189   1.733  0.0052 -> 0.00779289
```
Note: "->" shows value change within the same column (old value -> new value)

**Position Closed** - Closed positions:
```
🔴 [Trader A] Position Closed: LTCUSDT
🔴 [Trader A] Position Closed: DOGEUSDT, BNBUSDT
```

**Error Alerts** - System issues that need attention:
```
🚨 [Futures] StackFuturesIndicatorResult Partial Failure (1)
Time: 2026-01-05 03:33 +0900
Module: StackFuturesIndicatorResult
An issue occurred. Admin has been notified.
```

### The Error Philosophy

After implementing retry logic at the API level, any error that bubbles up is genuinely critical:

```python
try:
    # API call with built-in retries
    result = retry_request(place_order, max_retries=3)
except Exception as e:
    # If we get here, something is seriously wrong
    send_discord_alert(user_id, "Critical", str(e), "auto_trade")
    sys.exit(1)  # Stop the system
```

---

## Related Topics

This post covered the big picture. Future posts will dive deep into specific components:

- **Rebalancing Engine**: Weight calculations, process synchronization
- **Data Pipeline**: How we fetch and transform market data
- **Finwatcher**: Building a real-time monitoring system
- **Kubernetes Deployment**: From single Pod to CronJobs
- **Alert System**: Multi-channel notifications
- **API Design**: FastAPI backend architecture

---

## Key Takeaways

1. **Separate Concerns**: Three processes handle strategy, execution, and monitoring independently
2. **Event-Based Sync**: `multiprocessing.Event` provides clean coordination between processes
3. **Cloud-Native Design**: Kubernetes + Istio handle the infrastructure complexity
4. **Monitor Everything**: Per-minute metrics enable real-time visibility
5. **Fail Loudly**: After retries, any error is critical - alert and stop

Building a trading system is as much about infrastructure as it is about trading logic. The architecture decisions you make early on determine how reliable, maintainable, and scalable your system can be.

---

## Resources

- **Kubernetes**: https://kubernetes.io/docs/
- **Istio**: https://istio.io/latest/docs/
- **FastAPI**: https://fastapi.tiangolo.com/
- **Bitget API**: https://www.bitget.com/api-doc/

---

*Questions? Feedback? Feel free to reach out.*

---

**About the Author**
Building automated trading systems and cloud infrastructure. Passionate about making complex systems reliable and maintainable.

---

*Subscribe to get notified when new posts are published.*
