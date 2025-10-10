# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **cross-exchange delta neutral hedging system** for cryptocurrency perpetual futures. It opens simultaneous long and short positions across two exchanges (EdgeX and Lighter) to capture funding rate arbitrage while maintaining market-neutral exposure.

## Core Architecture

The system consists of four main Python modules:
- **`hedge_cli.py`**: Manual trading CLI tool with all core exchange functions
- **`lighter_edgex_hedge.py`**: Automated 24/7 rotation bot (imports hedge_cli functions)
- **`lighter_client.py`**: Lighter exchange helper functions (balance, positions, orders, closing)
- **`emergency_close.py`**: Emergency position closer (uses lighter_client functions)

### Main CLI Tool: `hedge_cli.py`

The primary interface for managing hedged positions. Commands:

```bash
# Analysis Commands
python hedge_cli.py capacity                      # Check available capital
python hedge_cli.py funding                       # Show funding rates (auto-updates config)
python hedge_cli.py funding_all                   # Compare multiple markets
python hedge_cli.py check_leverage                # Show leverage info for multiple markets
python hedge_cli.py status                        # Check current position status

# Trading Commands
python hedge_cli.py open                          # Open position using config notional
python hedge_cli.py open --size-base 0.05         # Open position (base units)
python hedge_cli.py open --size-quote 100         # Open position (quote units)
python hedge_cli.py close                         # Close both positions
python hedge_cli.py close --cross-ticks 5         # Close with aggressive fills

# Testing Commands
python hedge_cli.py test_leverage                 # Test leverage setup (no trading)
python hedge_cli.py test --notional 20            # Test open+close cycle ($20)
python hedge_cli.py test_auto --notional 20       # Test with auto-close after 5s

# Note: All commands use hedge_config.json by default
# Use --config BEFORE the command if using a different file:
python hedge_cli.py --config my_config.json open --size-quote 100
```

**Cross-ticks parameter** (`--cross-ticks N`): Controls how aggressively orders cross the spread. Higher values = more aggressive fills but worse pricing. Default is 100 ticks for near-instant execution.

### Configuration System

**hedge_config.json** defines the hedging strategy:
- `symbol`: Base asset (e.g., "PAXG")
- `quote`: Quote currency (default "USD")
- `long_exchange`: Which exchange takes the long position ("edgex" or "lighter")
- `short_exchange`: Which exchange takes the short position ("edgex" or "lighter")
- `leverage`: Leverage to apply on both exchanges
- `notional`: Default notional size in quote currency for `open` command (default: 40.0 USD)

**.env file** contains all exchange credentials (use `.env.example` as a template):

**EdgeX credentials:**
- `EDGEX_BASE_URL` (default: https://pro.edgex.exchange)
- `EDGEX_WS_URL` (default: wss://quote.edgex.exchange)
- `EDGEX_ACCOUNT_ID`
- `EDGEX_STARK_PRIVATE_KEY`

**Lighter credentials:**
- `LIGHTER_BASE_URL` or `BASE_URL` (default: https://mainnet.zklighter.elliot.ai)
- `LIGHTER_WS_URL` or `WEBSOCKET_URL` (default: wss://mainnet.zklighter.elliot.ai/stream)
- `API_KEY_PRIVATE_KEY` or `LIGHTER_PRIVATE_KEY`
- `ACCOUNT_INDEX` or `LIGHTER_ACCOUNT_INDEX` (default: 0)
- `API_KEY_INDEX` or `LIGHTER_API_KEY_INDEX` (default: 0)

**Note:** Margin mode is hardcoded to "cross" for delta-neutral hedging. A `.env.example` file is provided with placeholder values.

### Exchange Integration

**EdgeX (edgex-python-sdk)**:
- Contract identification by symbol+quote (e.g., "PAXGUSD")
- Leverage setting via internal authenticated endpoint
- Position closing via offsetting aggressive limit orders
- Capital retrieval from `get_account_asset()` endpoint
- Funding rates from quote API and historical endpoint
- **Note:** EdgeX-specific code is in `hedge_cli.py` (no separate connector module)

**Lighter (lighter-python SDK)**:
- Market identification by symbol
- Leverage setting via `update_leverage()` with margin mode
- Position closing via dual reduce-only orders (buy + sell)
- Capital retrieval via WebSocket `user_stats/{account_index}` channel
- Funding rates from candlestick API
- **Helper module:** `lighter_client.py` contains reusable functions:
  - `get_lighter_balance()`: Fetch balance via WebSocket
  - `get_lighter_market_details()`: Get market_id and tick sizes
  - `get_lighter_best_bid_ask()`: Fetch prices via WebSocket
  - `get_lighter_open_size()`: Get position size for a market
  - `get_lighter_position_details()`: Full position info with PnL
  - `get_all_lighter_positions()`: Fetch all non-zero positions
  - `lighter_close_position()`: Close position with reduce-only order
  - `lighter_place_aggressive_order()`: Place market-crossing limit order
  - `cross_price()`: Calculate aggressive price crossing the spread

### Order Execution Strategy

Both exchanges use **aggressive limit orders** that cross the spread:
- Buy orders: priced at `best_ask + (cross_ticks * tick_size)`
- Sell orders: priced at `best_bid - (cross_ticks * tick_size)`
- Default `cross_ticks`: 100 (for very fast, near-instant execution)
- This ensures immediate fills while avoiding market order unpredictability

**Position opening**: Places both legs concurrently using `asyncio.gather()`

**Position closing**:
- Lighter: Sends dual reduce-only orders (only the offsetting side executes)
- EdgeX: Detects current position size and sends offsetting order

### Capital Management

The `capacity` command calculates maximum delta-neutral position size:
1. Fetches available USD on both exchanges
2. Applies safety margin (1%) and fee buffer (0.1%)
3. Calculates per-venue capacity: `available_usd * (1 - buffers) * leverage / mid_price`
4. Max size = minimum of long and short venue capacities
5. Rounds conservatively using both exchanges' tick sizes

### Automated Rotation Bot: `lighter_edgex_hedge.py`

**Primary production bot** that fully automates the funding arbitrage cycle. Runs continuously 24/7 to maximize profits.

```bash
# Run the bot (uses bot_config.json)
python lighter_edgex_hedge.py

# Run with custom config
python lighter_edgex_hedge.py --config custom_config.json --state-file custom_state.json

# Docker service (recommended for 24/7 operation)
docker-compose up lighter_edgex_hedge
```

**Configuration file: `bot_config.json`**
```json
{
  "symbols_to_monitor": ["BTC", "ETH", "SOL", "HYPE", ...],
  "leverage": 3,
  "notional_per_position": 320.0,
  "hold_duration_hours": 8.0,
  "wait_between_cycles_minutes": 5.0,
  "check_interval_seconds": 300,
  "min_net_apr_threshold": 5.0,
  "enable_stop_loss": true,
  "enable_pnl_tracking": true,
  "enable_health_monitoring": true
}
```

**Note on stop-loss:** When `enable_stop_loss: true`, the bot auto-calculates stop-loss percentage as `(100/leverage) * 0.7` safety margin. For 3x leverage: ~23.3%, for 4x: 17.5%, for 5x: 14.0%. You can manually override by adding `"stop_loss_percent": 25.0`.

**State Machine Architecture:**
1. **IDLE** → Waiting to start analysis
2. **ANALYZING** → Fetching funding rates, selecting best opportunity
3. **OPENING** → Executing delta-neutral position entry
4. **HOLDING** → Monitoring position health, collecting funding
5. **CLOSING** → Exiting both positions
6. **WAITING** → Cooldown period before next cycle
7. **ERROR** → Manual intervention required

**Persistent State (`logs/bot_state.json`):**
- `current_cycle`: Cycle counter (persists across restarts)
- `current_position`: Active position details (symbol, sizing, entry prices, PnL)
- `capital_status`: Real-time balance tracking on both exchanges
- `completed_cycles`: Historical cycle records
- `cumulative_stats`: Aggregate performance metrics

**Key Features:**
- **Automatic Recovery**: On restart, verifies actual positions match expected state
- **Cycle Persistence**: Cycle counter stored in state file, survives restarts
- **Real-time Monitoring**: Checks position health every `check_interval_seconds` (default: 300s/5min)
- **Stop-Loss Protection**: Closes position if worst leg exceeds `stop_loss_percent` of notional
- **Funding Rates Display**: Shows top 5 opportunities with current position highlighted
- **Capital Tracking**: Monitors EdgeX total equity (includes positions) vs available balance
- **PnL Calculation**: Manual computation for EdgeX (uses quote API), Lighter provides directly
- **Graceful Shutdown**: CTRL+C completes current operation before exiting

**Display Output (HOLDING state):**
- Cycle number (e.g., "Cycle #1")
- Position opened/target close timestamps
- Time elapsed/remaining with progress percentage
- EdgeX/Lighter/Total unrealized PnL (color-coded)
- Available capital and max new position size
- Funding rates comparison table (top 10, highlights current and best)

**Display Output (STARTUP):**
- Initial funding rate scan showing top 10 opportunities
- Formatted table with Symbol, EdgeX APR, Lighter APR, Net APR, Long exchange
- Color-coded markers: ★ BEST for highest APR, ◀ CURRENT for active position

**Display Output (WAITING state):**
- Completed cycles count
- Next cycle number
- Countdown timer with 30-second updates

**Important Implementation Details:**
- Uses `hedge_cli.py` functions for exchange operations (avoid duplication)
- EdgeX capital: Must use `totalEquity` from `collateralAssetModelList` (not `amount`)
- EdgeX PnL: Calculate manually using `(current_price × size) - open_value`
- Lighter capital: WebSocket `user_stats/{account_index}` channel
- State file updates after every operation for crash recovery
- Cycle counter increments when position opens
- State file location defaults to `logs/bot_state.json` but can be customized via `--state-file` argument or `BOT_STATE_FILE` environment variable

### Emergency Close: `emergency_close.py`

**Critical safety tool** for immediately closing all open positions on both exchanges, regardless of configuration files or normal workflow.

**⚠️ WINDOWS USERS:** This script ONLY works on Linux/macOS due to Lighter SDK limitations. On Windows, you **MUST** use Docker:

```bash
# LINUX/MACOS - Direct execution
python emergency_close.py --dry-run          # Check positions
python emergency_close.py                     # Close (requires 'CLOSE')
python emergency_close.py --cross-ticks 200  # Ultra-aggressive

# WINDOWS - Must use Docker
docker-compose run emergency_close --dry-run              # Check positions
docker-compose run emergency_close                        # Interactive mode
echo CLOSE | docker-compose run emergency_close          # Auto-confirm
docker-compose run emergency_close --cross-ticks 200     # Ultra-aggressive
```

**Platform Detection:** The script automatically detects Windows and refuses to run, directing you to use Docker instead.

**Key features:**
- Independent of `hedge_config.json` - closes ANY open positions
- Requires typing 'CLOSE' to confirm (safety measure)
- Dry-run mode to inspect positions first
- Configurable aggressiveness via `--cross-ticks` (default: 100)
- Clear color-coded output showing positions and PnL
- Works even if lighter_edgex_hedge or hedge_cli are stuck
- **Uses `lighter_client.py` functions** for Lighter position fetching and closing
- **EdgeX positions closed directly** using EdgeX SDK with aggressive limit orders

**Use cases:**
- Emergency exit when normal close commands fail
- Liquidation risk mitigation
- Quick exit during extreme volatility
- Recovery from bot errors or unhedged positions

**Important:** This script places market-crossing limit orders for immediate fills. Higher `cross_ticks` = faster execution but worse prices. Default of 100 ticks provides near-instant fills.

### Liquidation Monitor: `examples/liquidation_monitor.py`

Continuous monitoring service for delta-neutral position health (complementary to lighter_edgex_hedge). Located in `examples/` directory.

```bash
# Run with defaults (60s interval, 20% margin threshold)
python examples/liquidation_monitor.py

# Custom parameters
python examples/liquidation_monitor.py --interval 30 --margin-threshold 15.0

# Docker service (uncomment in docker-compose.yml first, then run)
docker-compose up liquidation_monitor
```

**Key features:**
- Monitors positions on both exchanges every N seconds
- Checks margin ratio and unrealized PnL
- Auto-closes positions if margin falls below threshold
- Colored console output (green/yellow/red based on health)
- Logs to `logs/liquidation_monitor.log` (DEBUG level)
- Uses same configuration as `hedge_cli.py` (hedge_config.json + .env)

**Margin calculation:**
- EdgeX: Uses `marginRatio` from positions API (% of margin used)
- Lighter: Uses `user_stats` WebSocket for available margin vs. total

**Auto-close triggers:**
- Margin ratio exceeds threshold (default: 80%)
- Position becomes unhedged (only one side has position)
- Critical errors fetching position data

## Environment Variables

The system supports flexible environment variable naming for Lighter exchange:

**Primary names** (recommended):
- `LIGHTER_BASE_URL` (default: https://mainnet.zklighter.elliot.ai)
- `LIGHTER_WS_URL` (default: wss://mainnet.zklighter.elliot.ai/stream)
- `LIGHTER_PRIVATE_KEY`
- `LIGHTER_ACCOUNT_INDEX` (default: 0)
- `LIGHTER_API_KEY_INDEX` (default: 0)

**Legacy fallback names** (still supported):
- `BASE_URL` → `LIGHTER_BASE_URL`
- `WEBSOCKET_URL` → `LIGHTER_WS_URL`
- `API_KEY_PRIVATE_KEY` → `LIGHTER_PRIVATE_KEY`
- `ACCOUNT_INDEX` → `LIGHTER_ACCOUNT_INDEX`
- `API_KEY_INDEX` → `LIGHTER_API_KEY_INDEX`

**Auto-rotation bot specific**:
- `BOT_STATE_FILE` - Override default state file location (default: `logs/bot_state.json`)
- `PYTHONUNBUFFERED=1` - For real-time Docker logs

## Example Bots

All example bots are in the `examples/` directory:

### `examples/edgex_trading_bot.py`
Market maker for EdgeX with Avellaneda-Stoikov spread optimization:
- Dynamic sizing based on capital usage ratio
- Configurable leverage and spread parameters
- Balance snapshot tracking
- Optional volume limits

### `examples/market_maker_v2.py`
Advanced Lighter market maker with:
- SuperTrend-based directional bias (flip logic)
- Avellaneda parameter integration
- WebSocket order book and account state tracking
- Position management with dynamic sizing

### `examples/market_maker.py` & `examples/market_maker_with_EMA.py`
Additional market maker variations with different strategies

### `examples/edgex_data_collector.py`
Collects market data from EdgeX for analysis

### `examples/gather_lighter_data.py`
Collects market data from Lighter for analysis

### `examples/check_markets.py`
Utility to verify market availability on both exchanges

### `examples/calculate_stoploss_by_leverage.py` & `examples/calculate_stoploss_extended.py`
Utilities to calculate appropriate stop-loss percentages based on leverage

## Dependencies

```bash
pip install -r requirements.txt
```

Key packages:
- `edgex-python-sdk`: EdgeX REST/WebSocket SDK (imports as `edgex_sdk`)
- `lighter-python`: Lighter SDK (installed from GitHub)
- `python-dotenv`: Environment variable management
- `websockets`: WebSocket client for Lighter capital queries

**Note:** The EdgeX package installs as `edgex-python-sdk` but imports as `edgex_sdk`:
```python
from edgex_sdk import Client as EdgeXClient
```

## Development & Testing Workflow

### Local Development
```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Configure credentials
cp .env.example .env
# Edit .env with your API keys

# 3. Test funding rates (no trading)
python hedge_cli.py funding_all

# 4. Check available capital
python hedge_cli.py capacity

# 5. Run small test trade
python hedge_cli.py test --notional 20

# 6. Monitor existing positions
python hedge_cli.py status
```

### Testing Commands (Safe)
These commands are safe for testing and development:

**Analysis only (no execution):**
- `funding` / `funding_all` - Check funding rates
- `capacity` - Calculate max position size
- `check_leverage` - Verify leverage settings
- `status` - Check current positions

**Test with minimal capital:**
- `test --notional 20` - Full cycle with $20 position
- `test_auto --notional 20` - Auto-closing test
- `test_leverage` - Verify leverage without trading

**Emergency commands:**
- Linux/macOS: `python emergency_close.py --dry-run` - Check all positions
- Linux/macOS: `python emergency_close.py` - Emergency close all positions
- **Windows: `docker-compose run emergency_close --dry-run`** - MUST use Docker
- **Windows: `docker-compose run emergency_close`** - MUST use Docker

### Production Deployment
```bash
# 1. Edit production config
nano bot_config.json

# 2. Test in dry-run mode first
python lighter_edgex_hedge.py --dry-run

# 3. Run small position for first real cycle
# (temporarily set notional_per_position to 50-100)

# 4. Deploy with Docker for 24/7 operation
docker-compose up -d lighter_edgex_hedge

# 5. Monitor logs
docker-compose logs -f lighter_edgex_hedge

# 6. Scale up notional after confirming success
```

## Docker Support

All CLI commands are available as Docker services:

```bash
# Analysis commands
docker-compose run capacity
docker-compose run funding
docker-compose run funding_all
docker-compose run status
docker-compose run check_leverage

# Trading commands
docker-compose run open
docker-compose run close

# Testing commands
docker-compose run test_leverage
docker-compose run test
docker-compose run test_auto
```

You can override command arguments:
```bash
docker-compose run open --size-quote 100
docker-compose run test --notional 50
```

## Key Design Patterns

### Code Architecture Principles

1. **Modular architecture with function reuse**:
   - `hedge_cli.py` contains all core exchange interaction functions (EdgeX + Lighter)
   - `lighter_client.py` contains Lighter-specific helper functions (balance, positions, orders)
   - `lighter_edgex_hedge.py` imports and reuses hedge_cli functions (no duplication)
   - `emergency_close.py` imports and uses lighter_client functions for Lighter operations
   - **Dual-client pattern**: Each operation initializes both exchange clients, performs actions, then closes connections
   - Clean separation: CLI logic vs bot state machine logic vs exchange helpers

2. **Tick-aware rounding**: All sizes/prices rounded to exchange-specific tick sizes using `_round_to_tick()`, `_ceil_to_tick()`, `_floor_to_tick()` with Decimal arithmetic to avoid floating-point precision errors
   - Critical for delta-neutral hedging: sizes must be IDENTICAL
   - Solution: Use coarser tick size (max of both exchanges) and floor to ensure both round the same
   - Example: EdgeX tick=0.001, Lighter tick=0.01 → use 0.01 and floor

3. **Fallback pricing**: If best bid/ask unavailable, uses last price with synthetic spread (EdgeX) or falls back to available side
   - Prevents order placement failures due to missing orderbook data
   - Synthetic spread: `last_price ± (last_price * 0.0001)` for bid/ask

4. **Concurrent execution**: Position opening/closing uses `asyncio.gather()` for simultaneous exchange actions
   - Minimizes timing risk between exchanges
   - Both legs execute at nearly the same time for true delta-neutral entry/exit

5. **Environment flexibility**: Supports multiple naming conventions for environment variables (LIGHTER_* vs original names)
   - Backwards compatibility with legacy configs
   - Code checks `LIGHTER_*` first, falls back to original names

6. **State persistence pattern** (lighter_edgex_hedge.py):
   - Every state change saves to JSON immediately
   - On restart: verify actual positions match expected state
   - Crash recovery: can resume HOLDING state if hedge still valid

### Critical Implementation Details

**EdgeX specifics:**
- Contract name = symbol + quote (no separator): "PAXGUSD"
- Leverage set via internal REST endpoint (not public SDK method)
- Capital query: Use `totalEquity` from `collateralAssetModelList` (includes position value)
- PnL calculation: Manual computation using `(current_price × size) - open_value`
- Position close: Detect size, send offsetting aggressive limit order

**Lighter specifics:**
- Market identification: Symbol only (e.g., "PAXG")
- Leverage set via `update_leverage(leverage, margin_mode='cross')`
- Capital query: WebSocket channel `user_stats/{account_index}` (via `get_lighter_balance()`)
- PnL: Provided directly by API (no manual calculation)
- Position close: Dual reduce-only orders (buy + sell), only offsetting side executes (via `lighter_close_position()`)
- Position attributes: Use `pos.position` (unsigned size) with `pos.sign` (1=long, -1=short), NOT `pos.size`
- Entry price attribute: `pos.avg_entry_price` (NOT `pos.entry_price`)

**Rounding functions (using Decimal for precision):**
```python
from decimal import Decimal, ROUND_DOWN, ROUND_UP, ROUND_HALF_UP

def _round_to_tick(value: float, tick_size: float) -> float:
    """Round to nearest tick (banker's rounding)"""

def _floor_to_tick(value: float, tick_size: float) -> float:
    """Round down to tick boundary"""

def _ceil_to_tick(value: float, tick_size: float) -> float:
    """Round up to tick boundary"""
```

**Aggressive limit order pricing:**
```python
# Buy: cross the spread upward
buy_price = best_ask + (cross_ticks * tick_size)

# Sell: cross the spread downward
sell_price = best_bid - (cross_ticks * tick_size)
```
This ensures immediate fills without true market orders' unpredictability.

## Contract Naming

- **EdgeX**: Symbol + Quote concatenated (e.g., PAXG + USD = "PAXGUSD")
- **Lighter**: Symbol only (e.g., "PAXG")

## Important Notes

- The system uses aggressive limit orders (not true market orders) for better control
- **Leverage is set on both exchanges before opening positions** with verification via `setup_leverage_both_exchanges()`
- **Position sizing ensures IDENTICAL sizes** on both exchanges by using the coarser tick size and flooring
- Position size verification happens after opening to confirm both legs executed correctly
- WebSocket connections are ephemeral - created per-query for capital checks, not persistent
- Funding rate arbitrage is the primary profit mechanism for this delta-neutral strategy
- The `funding` command auto-updates `hedge_config.json` if suboptimal configuration detected
- **⚠️ WINDOWS LIMITATION:** The Lighter SDK only supports Linux/macOS. On Windows, ALL commands must be run via Docker (e.g., `docker-compose run close` instead of `python hedge_cli.py close`)

## Troubleshooting

### Common Issues

**"Position size mismatch" error:**
- Caused by different tick sizes between exchanges
- Solution: Bot automatically uses coarser tick size and floors the value
- Check logs for detailed size calculations

**"Leverage setup failed" warning:**
- EdgeX leverage may not be verifiable until position exists
- Warning is informational, proceed with caution
- Run `test_leverage` command to verify setup

**WebSocket connection errors (Lighter capital query):**
- Ephemeral connections created per-query, not persistent
- Retry logic built-in for transient failures
- Check network connectivity and `LIGHTER_WS_URL` setting

**"No funding rate data" errors:**
- API timeout or rate limiting
- Verify exchange API endpoints are accessible
- Check for exchange maintenance windows

**Unhedged position detected:**
- One leg failed to execute or was manually closed
- Bot enters ERROR state requiring manual intervention
- **Quick fix:** Run `python emergency_close.py` to close all positions
- Alternative: Check both exchanges manually and close remaining position
- Delete or fix `logs/bot_state.json` before restart

**Auto-rotation bot stuck in ANALYZING:**
- No symbols meet `min_net_apr_threshold`
- Lower threshold or wait for better market conditions
- Add more symbols to `symbols_to_monitor`

**Docker container exits immediately:**
- Check `.env` file exists and has valid credentials
- View logs: `docker-compose logs lighter_edgex_hedge`
- Verify configuration files are mounted correctly

**Need to exit all positions immediately:**
- Use `python emergency_close.py` for fastest exit
- Independent of config files or bot state
- Dry-run first: `python emergency_close.py --dry-run`
- Increase urgency with `--cross-ticks 30` (faster but worse prices)

### Logging & Debugging

**Console output:**
- `hedge_cli.py`: WARNING level and above (clean output)
- `lighter_edgex_hedge.py`: INFO level, color-coded status

**Log files:**
- `hedge_cli.log`: DEBUG level, all hedge_cli operations
- `logs/lighter_edgex_hedge.log`: DEBUG level, full bot activity
- `logs/liquidation_monitor.log`: DEBUG level, position monitoring

**State inspection:**
```bash
# View current bot state
cat logs/bot_state.json | python -m json.tool

# Monitor bot logs live
tail -f logs/lighter_edgex_hedge.log

# Check hedge CLI debug output
tail -f hedge_cli.log
```

## Recent Improvements (2025)

### Position Size Consistency
- Uses coarser tick size (larger of the two exchanges) to ensure identical position sizes
- Verifies both exchanges will round to the same value before execution
- Displays tick sizes and scaled units in output for transparency
- Prevents unhedged exposure from rounding mismatches

### Leverage Management
- `setup_leverage_both_exchanges()` function sets leverage on both exchanges before opening
- EdgeX leverage verification via positions API (when position exists)
- `test_leverage` command to verify leverage setup without trading
- Clear warnings if leverage setup fails, but allows proceeding with caution

### Enhanced Commands
- `status`: Check current delta-neutral position status across both exchanges
- `funding_all`: Compare funding rates across multiple markets simultaneously
- `check_leverage`: Show max/default leverage for multiple markets on both exchanges
- `test`: Complete open+close cycle with configurable notional ($20 default)
- `test_auto`: Similar to `test` with automatic close after 5 seconds
- `test_leverage`: Test leverage configuration without opening positions
- `open` command: Now uses `notional` from config if no size arguments provided
- All trading commands support `--cross-ticks N` parameter (default: 10)

### Improved Error Handling
- Position verification after opening and closing
- Clear warnings for unhedged positions if one leg fails
- Formatted output boxes for better readability
- Detailed logging to `hedge_cli.log` (DEBUG+) with clean console (WARNING+)

### Precision Fixes
- All rounding functions now use Python's `Decimal` type for exact arithmetic
- Eliminates floating-point precision errors like `0.059000000000000004`
- Ensures exchange APIs receive properly formatted numeric strings

### Execution Improvements
- Default `cross_ticks` set to 100 for near-instant order fills
- Minimizes timing risk between exchanges (critical for delta-neutral hedging)
- Prioritizes execution speed over price improvement
- User can still override with `--cross-ticks N` if needed

### Configuration Simplification
- `.env.example` file provided as template
- Margin mode hardcoded to "cross" (removed from environment variables)
- Simplified setup process for new users

### DateTime Handling (2025)
- All datetime operations use timezone-aware UTC objects (`timezone.utc`)
- Proper ISO timestamp formatting and parsing with helper functions:
  - `utc_now()`: Returns timezone-aware UTC datetime
  - `utc_now_iso()`: Returns ISO 8601 timestamp with Z suffix
  - `to_iso_z()`: Converts datetime to ISO string with Z suffix
  - `from_iso_z()`: Parses ISO timestamps, handling malformed formats gracefully
- Eliminated all timezone-naive datetime operations
- Compatible with Python 3.7+ (uses `timezone.utc` instead of `datetime.UTC`)
- No more deprecation warnings from `datetime.utcnow()`

### Funding Rate Display
- Funding rate comparison table now shown at three key points:
  1. **Startup** - Initial scan of all symbols showing market landscape
  2. **Before opening position** - Full comparison before selecting best opportunity
  3. **During holding** - Real-time updates every check interval
- Consistent table format across all displays
- Color-coded status markers (★ BEST, ◀ CURRENT)
- Shows top 10 opportunities by default
