# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **cross-exchange delta neutral hedging system** for cryptocurrency perpetual futures. It opens simultaneous long and short positions across two exchanges (EdgeX and Lighter) to capture funding rate arbitrage while maintaining market-neutral exposure.

## Core Architecture

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

**Cross-ticks parameter** (`--cross-ticks N`): Controls how aggressively orders cross the spread. Higher values = more aggressive fills but worse pricing. Default is 10 ticks for fast execution.

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

**Lighter (lighter-python SDK)**:
- Market identification by symbol
- Leverage setting via `update_leverage()` with margin mode
- Position closing via dual reduce-only orders (buy + sell)
- Capital retrieval via WebSocket `user_stats/{account_index}` channel
- Funding rates from candlestick API

### Order Execution Strategy

Both exchanges use **aggressive limit orders** that cross the spread:
- Buy orders: priced at `best_ask + (cross_ticks * tick_size)`
- Sell orders: priced at `best_bid - (cross_ticks * tick_size)`
- Default `cross_ticks`: 10 (for fast execution)
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

### Automated Rotation Bot: `auto_rotation_bot.py`

**Primary production bot** that fully automates the funding arbitrage cycle. Runs continuously 24/7 to maximize profits.

```bash
# Run the bot (uses rotation_bot_config.json)
python auto_rotation_bot.py

# Run with custom config
python auto_rotation_bot.py --config custom_config.json --state-file custom_state.json

# Docker service (recommended for 24/7 operation)
docker-compose up auto_rotation_bot
```

**Configuration file: `rotation_bot_config.json`**
```json
{
  "symbols_to_monitor": ["BTC", "ETH", "SOL", "HYPE", ...],
  "leverage": 3,
  "notional_per_position": 320.0,
  "hold_duration_hours": 8.0,
  "wait_between_cycles_minutes": 5.0,
  "check_interval_seconds": 300,
  "min_net_apr_threshold": 5.0,
  "stop_loss_percent": 25.0,
  "enable_stop_loss": true
}
```

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
- Funding rates comparison table (top 5, highlights current and best)

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
- Cycle counter increments when position opens (line 1106)

### Liquidation Monitor: `liquidation_monitor.py`

Continuous monitoring service for delta-neutral position health (complementary to auto_rotation_bot):

```bash
# Run with defaults (60s interval, 20% margin threshold)
python liquidation_monitor.py

# Custom parameters
python liquidation_monitor.py --interval 30 --margin-threshold 15.0

# Docker service (runs continuously with restart)
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

## Example Bots

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

### `examples/edgex_data_collector.py`
Collects market data from EdgeX for analysis

### `examples/gather_lighter_data.py`
Collects market data from Lighter for analysis

## Dependencies

```bash
pip install -r requirements.txt
```

Key packages:
- `edgex-python-sdk`: EdgeX REST/WebSocket SDK
- `lighter-python`: Lighter SDK (installed from GitHub)
- `python-dotenv`: Environment variable management
- `websockets`: WebSocket client for Lighter capital queries

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

1. **Dual-client pattern**: Each operation initializes both exchange clients, performs actions, then closes connections
2. **Tick-aware rounding**: All sizes/prices rounded to exchange-specific tick sizes using `_round_to_tick()`, `_ceil_to_tick()`, `_floor_to_tick()` with Decimal arithmetic to avoid floating-point precision errors
3. **Fallback pricing**: If best bid/ask unavailable, uses last price with synthetic spread (EdgeX) or falls back to available side
4. **Concurrent execution**: Position opening/closing uses `asyncio.gather()` for simultaneous exchange actions
5. **Environment flexibility**: Supports multiple naming conventions for environment variables (LIGHTER_* vs original names)

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
- Default `cross_ticks` increased from 2 to 10 for faster order fills
- Reduces timing risk between exchanges
- User can still override with `--cross-ticks N` if needed

### Configuration Simplification
- `.env.example` file provided as template
- Margin mode hardcoded to "cross" (removed from environment variables)
- Simplified setup process for new users
