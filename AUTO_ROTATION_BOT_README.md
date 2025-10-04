# Automated Delta-Neutral Rotation Bot

An intelligent bot that continuously rotates delta-neutral positions across EdgeX and Lighter exchanges to maximize funding rate arbitrage profits and volume.

## 🎯 Overview

This bot automates the entire funding arbitrage cycle:

1. **Analyzes** funding rates across multiple crypto assets
2. **Selects** the best opportunity (highest net APR)
3. **Opens** a delta-neutral position (one long, one short)
4. **Holds** for a configured duration (default: 8 hours) collecting funding payments
5. **Closes** both positions
6. **Waits** briefly for market conditions to refresh
7. **Repeats** the cycle indefinitely

### Key Features

- ✅ **Fully Automated** - Runs 24/7 without manual intervention
- ✅ **Intelligent Selection** - Always picks the best funding opportunity
- ✅ **Comprehensive PnL Tracking** - Tracks every dollar earned/lost
- ✅ **Persistent State** - Survives crashes and restarts
- ✅ **Automatic Recovery** - Detects and reconciles actual vs. expected positions
- ✅ **Health Monitoring** - Periodic checks during hold period
- ✅ **Graceful Shutdown** - Clean exit on CTRL+C
- ✅ **Dry-Run Mode** - Test without risking capital
- ✅ **Detailed Logging** - Full audit trail in `logs/auto_rotation_bot.log`
- ✅ **Color-Coded Console** - Easy-to-read status updates

## 📋 Prerequisites

1. **Python 3.8+**
2. **Active accounts** on both EdgeX and Lighter
3. **API credentials** configured in `.env` file
4. **Sufficient capital** on both exchanges
5. **Dependencies installed**: `pip install -r requirements.txt`

## 🚀 Quick Start

### 1. Configure the Bot

Edit `rotation_bot_config.json`:

```json
{
  "symbols_to_monitor": ["BTC", "ETH", "SOL", "PAXG", "HYPE", "XPL"],
  "quote": "USD",
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

### 2. Test in Dry-Run Mode

```bash
python auto_rotation_bot.py --dry-run
```

This will simulate the bot's behavior without opening any real positions.

### 3. Run Live

```bash
python auto_rotation_bot.py
```

The bot will:
- Load previous state from `bot_state.json` (if exists)
- Verify positions match expected state
- Start the automated rotation cycle

### 4. Monitor Progress

Watch the console for colored status updates:
- 🟢 **GREEN** = Profitable PnL
- 🔴 **RED** = Loss or error
- 🟡 **YELLOW** = Warnings
- 🔵 **CYAN** = Status information

**HOLDING State Display:**
```
──────────────────────────────────────────────────────────────────────
HOLDING HYPE - Delta Neutral Position (Cycle #1)
──────────────────────────────────────────────────────────────────────
📅  Opened:        2025-10-03 18:08:52 UTC
🎯  Target Close:  2025-10-04 02:08:52 UTC
⏱  Time Elapsed:  1.14h / 8.00h (14.2%)
⏰  Time Remaining: 6.86h until close
💼  Position Size: $308.75
📈  EdgeX PnL:     $-4.64
📉  Lighter PnL:   $+4.66
💰  Total PnL:     $+0.02
💵  Available:     EdgeX $0.00, Lighter $11.29
🏦  EdgeX Total:   $102.58
📊  Max new position:  $0.00 (limited by EdgeX)
──────────────────────────────────────────────────────────────────────

📊 Funding Rates Overview
──────────────────────────────────────────────────────────────────────
Symbol      EdgeX APR  Lighter APR    Net APR Status
----------------------------------------------------------------------
ASTER          74.18%      245.28%    171.10% ★ BEST
HYPE          223.12%       60.44%    162.68% ◀ CURRENT
XPL            10.95%      155.93%    144.98%
LINK           10.95%       66.58%     55.63%
ETH           -22.25%       10.51%     32.76%
----------------------------------------------------------------------
Note: ASTER currently has +8.42% better APR
──────────────────────────────────────────────────────────────────────
```

**WAITING State Display:**
```
──────────────────────────────────────────────────────────────────────
WAITING - Cooldown Between Cycles
──────────────────────────────────────────────────────────────────────
✅  Completed Cycles: 1
🔄  Next Cycle:       #2
⏳  Waiting:          5.0 minutes before next analysis
──────────────────────────────────────────────────────────────────────

⏳  4.5 minutes until next cycle...
⏳  4.0 minutes until next cycle...
...
```

Check detailed logs: `tail -f logs/auto_rotation_bot.log`

### 5. Graceful Shutdown

Press **CTRL+C** to stop the bot. It will:
- Finish the current operation
- Save state to `logs/bot_state.json`
- Display final statistics
- Exit cleanly

## 📊 Configuration Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `symbols_to_monitor` | list | `["BTC", "ETH", ...]` | Symbols to analyze for funding opportunities |
| `quote` | string | `"USD"` | Quote currency |
| `leverage` | int | `3` | Leverage to use on both exchanges |
| `notional_per_position` | float | `320.0` | Position size in USD (will adjust to actual available capital) |
| `hold_duration_hours` | float | `8.0` | How long to hold each position (hours) |
| `wait_between_cycles_minutes` | float | `5.0` | Cooldown between cycles (minutes) |
| `check_interval_seconds` | int | `300` | How often to check position health (seconds, default: 5 minutes) |
| `min_net_apr_threshold` | float | `5.0` | Minimum net APR to open position (%) |
| `stop_loss_percent` | float | `25.0` | Stop-loss threshold as % of position notional |
| `enable_stop_loss` | bool | `true` | Enable automatic stop-loss protection |

### Configuration Tips

- **symbols_to_monitor**: Include 5-15 symbols for best selection
- **leverage**: Higher = more profit but more risk (3-5x recommended)
- **notional_per_position**: Maximum position size (bot adjusts to actual available capital). Start small ($50-100) then scale up
- **hold_duration_hours**: 8h = 2 EdgeX funding periods, 8 Lighter periods
- **check_interval_seconds**: 300s (5 minutes) is a good balance between monitoring and API rate limits
- **min_net_apr_threshold**: Only take trades above this APR (prevents low-profit cycles)
- **stop_loss_percent**: Protects against extreme price movements. Adjust according to leverage with significant margin (e.g., 25% for 3x leverage)
- **enable_stop_loss**: Keep enabled for safety; disable only if you monitor positions 24/7

## 📈 PnL Tracking

The bot tracks comprehensive PnL metrics:

### Real-Time Tracking (During Hold)
- **Unrealized PnL**: Current mark-to-market profit/loss
- **EdgeX PnL**: Position PnL on EdgeX
- **Lighter PnL**: Position PnL on Lighter
- **Total PnL**: Sum of both exchanges

### Cycle Completion Metrics
- **Realized PnL**: Actual profit/loss after closing
- **Balance Changes**: Direct measurement from exchange balances
- **EdgeX Change**: `balance_after - balance_before` on EdgeX
- **Lighter Change**: `balance_after - balance_before` on Lighter
- **Duration**: Actual hold time (may differ from target)

### Cumulative Statistics
- **Total Cycles**: Number of completed rotation cycles
- **Success Rate**: Percentage of successful cycles
- **Total Realized PnL**: Cumulative profit/loss across all cycles
- **Best Cycle**: Highest single-cycle profit
- **Worst Cycle**: Largest single-cycle loss
- **Average PnL/Cycle**: Mean profit per cycle
- **Total Volume**: Sum of all position sizes traded
- **Total Hold Time**: Cumulative hours in positions

### Per-Symbol Statistics
Tracked separately for each symbol:
- Number of cycles
- Total PnL for that symbol
- Average PnL per cycle

### PnL Components

The bot tracks where P&L comes from:

1. **Trading PnL** (usually near-zero for delta-neutral)
   - Price movement impact between entry and exit
   - Should be minimal due to hedge

2. **Funding PnL** (primary profit source)
   - EdgeX: Funding payments every 4 hours
   - Lighter: Funding payments every hour
   - Net = `funding_received_short - funding_paid_long`

3. **Fees** (cost of doing business)
   - Entry fees (2 trades: open long + open short)
   - Exit fees (2 trades: close long + close short)
   - Typically 0.05-0.1% per trade per exchange

**Real PnL = Balance After - Balance Before**

This is the ground truth measurement, capturing:
- All funding payments received/paid
- All trading fees
- Price slippage on entry/exit
- Any other exchange-specific adjustments

## 📁 State Files

### `bot_state.json`
Persistent state file (saved to `logs/bot_state.json`) containing:
- Current bot state (IDLE, HOLDING, etc.)
- **Current cycle number** (persists across restarts)
- Active position details (if any)
- Capital status on both exchanges
- Completed cycles history (last 100)
- Cumulative statistics
- Error tracking

**Example structure:**
```json
{
  "version": "1.0",
  "state": "HOLDING",
  "current_cycle": 5,
  "current_position": {
    "symbol": "PAXG",
    "opened_at": "2025-10-03T12:00:00Z",
    "target_close_at": "2025-10-03T20:00:00Z",
    "entry": {
      "edgex_entry_price": 2650.50,
      "lighter_entry_price": 2651.00,
      "size_base": 0.05,
      "edgex_balance_before": 1000.00,
      "lighter_balance_before": 1000.00
    },
    "current_pnl": {
      "edgex_unrealized_pnl": -0.15,
      "lighter_unrealized_pnl": 0.20,
      "total_unrealized_pnl": 0.05
    }
  },
  "capital_status": {
    "edgex_total": 1050.30,
    "edgex_available": 50.30,
    "lighter_total": 1048.75,
    "lighter_available": 48.75,
    "total_capital": 2099.05,
    "total_available": 99.05,
    "max_position_notional": 297.15,
    "limiting_exchange": "Lighter"
  },
  "completed_cycles": [
    {
      "cycle_number": 1,
      "symbol": "BTC",
      "pnl_breakdown": {
        "total_realized_pnl": 0.45
      }
    }
  ],
  "cumulative_stats": {
    "total_cycles": 4,
    "successful_cycles": 4,
    "total_realized_pnl": 5.60,
    "best_cycle_pnl": 0.85,
    "worst_cycle_pnl": -0.15
  }
}
```

**Important Notes:**
- `current_cycle` increments when a position is opened and persists across restarts
- `capital_status.edgex_total` uses `totalEquity` from EdgeX API (includes position value)
- `capital_status` updated every `check_interval_seconds` (default: 300s)
- State file is saved after every operation for crash recovery

### `rotation_bot_config.json`
Configuration file (edit this to change bot behavior)

## 🛡️ Stop-Loss Protection

The bot includes intelligent stop-loss protection to safeguard capital during unexpected price movements:

### How It Works

1. **Monitors Both Legs**: Checks EdgeX and Lighter PnL separately every `check_interval_seconds`
2. **Identifies Worst Leg**: Finds whichever exchange has the most negative unrealized PnL
3. **Triggers on Threshold**: If the worst leg exceeds `stop_loss_percent` of notional, closes immediately

### Example

With a $100 position and `stop_loss_percent: 25.0`:
- **EdgeX PnL**: -$26.00 ❌ (triggers stop-loss)
- **Lighter PnL**: +$5.00 ✅
- **Total PnL**: -$21.00

Even though total PnL is -$21, the bot closes because EdgeX leg hit -26% (exceeds -25% threshold).

### Why This Matters

In a truly delta-neutral position, both legs should have near-zero PnL. If ONE leg is losing significantly:
- The hedge may be broken (position size mismatch)
- Extreme volatility on one exchange
- Oracle/price feed issues
- One exchange moving faster than the other

The stop-loss protects you from these scenarios by exiting early.

### Configuration

```json
{
  "stop_loss_percent": 25.0,     // Trigger at 25% loss on either leg
  "enable_stop_loss": true       // Set to false to disable
}
```

**Recommended values (adjust according to leverage):**
- For 3x leverage: 15-25% (default: 25%)
- For 5x leverage: 10-15%
- For 10x leverage: 5-10%

**Important:** Set stop-loss with significant margin above your leverage to avoid false triggers from normal market volatility.

### Stop-Loss in Action

```
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
STOP-LOSS TRIGGERED!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Reason: EdgeX leg hit stop-loss: $-26.00 (-26.0%) exceeds threshold of $-25.00 (-25.0%)
EdgeX PnL: $-26.00
Lighter PnL: $+5.00
Total PnL: $-21.00
Closing position immediately...

Position closed due to stop-loss. Waiting before next cycle...
```

## 💰 Capital Tracking

The bot continuously monitors available capital on both exchanges:

### Capital Status Display

Every check interval, the bot displays:
- **Total capital** on each exchange
- **Available capital** (not locked in positions)
- **Maximum openable position** (accounting for leverage)
- **Limiting exchange** (which one restricts position size)

### Example Output

```
Capital Status:
  EdgeX:   $1000.00 total, $950.00 available
  Lighter: $1200.00 total, $1180.00 available
  Total:   $2200.00 total, $2130.00 available
  Max Position: $2850.00 (limited by EdgeX)
```

With 3x leverage:
- EdgeX can support: $950 × 3 = $2,850
- Lighter can support: $1,180 × 3 = $3,540
- **Max position = $2,850** (minimum of the two)

### In State File

Capital status is saved to `bot_state.json`:

```json
{
  "capital_status": {
    "edgex_total": 1000.00,
    "edgex_available": 950.00,
    "lighter_total": 1200.00,
    "lighter_available": 1180.00,
    "total_capital": 2200.00,
    "total_available": 2130.00,
    "max_position_notional": 2850.00,
    "limiting_exchange": "EdgeX",
    "last_updated": "2025-10-03T14:30:00Z"
  }
}
```

This helps you:
- Monitor capital growth over time
- Identify which exchange needs more funding
- Plan position sizes for future cycles
- Detect unexpected capital changes (fees, liquidations, etc.)

## 🔄 State Machine

The bot operates through these states:

```
IDLE → ANALYZING → OPENING → HOLDING → CLOSING → WAITING → (back to IDLE)
  ↓        ↓          ↓          ↓         ↓          ↓
  └────────┴──────────┴──────────┴─────────┴──────────┴─→ ERROR
```

- **IDLE**: No position, ready to start new cycle
- **ANALYZING**: Fetching funding rates, selecting best opportunity
- **OPENING**: Placing orders to open delta-neutral position
- **HOLDING**: Position open, waiting for target close time (or stop-loss)
- **CLOSING**: Closing both positions
- **WAITING**: Cooldown period between cycles
- **ERROR**: Something went wrong, manual intervention needed

## 🛡️ State Recovery

On startup, the bot performs automatic recovery:

### Scenario 1: Clean Restart (State = IDLE)
- Verifies no positions exist
- Starts fresh cycle

### Scenario 2: Crashed During Hold (State = HOLDING)
- Checks actual positions on both exchanges
- Verifies hedge is still valid (sizes match)
- Resumes countdown to close time
- Continues tracking PnL

### Scenario 3: Position Mismatch
- Detects if position sizes don't match (broken hedge)
- Enters ERROR state
- Requires manual intervention

### Scenario 4: State Corruption (State = OPENING/CLOSING/ERROR)
- Cannot automatically recover
- Displays error message
- User must manually check exchanges and fix state

### Manual Recovery

If the bot enters ERROR state:

1. Check positions on both exchanges manually
2. Option A: Close positions manually, delete `bot_state.json`, restart bot
3. Option B: Fix `bot_state.json` to match actual positions, restart bot

## 📊 Example Output

```
═══════════════════════════════════════════════════════════════════
AUTOMATED DELTA-NEUTRAL ROTATION BOT
═══════════════════════════════════════════════════════════════════
Loaded configuration from rotation_bot_config.json
Monitoring symbols: BTC, ETH, SOL, PAXG
Position size: $100.0
Hold duration: 8.0 hours
Leverage: 3x

Performing state recovery...
Recovery complete: IDLE state verified

═══════════════════════════════════════════════════════════════════
BOT STATUS
═══════════════════════════════════════════════════════════════════
State: IDLE

Cumulative Stats:
  Total Cycles: 0

═══════════════════════════════════════════════════════════════════

Analyzing funding rates for 10 symbols...

═══════════════════════════════════════════════════════════════════
BEST OPPORTUNITY: PAXG
═══════════════════════════════════════════════════════════════════
Net APR: 15.60%
Setup: Long Lighter, Short EdgeX
EdgeX APR: 18.25%, Lighter APR: -12.00%

Capturing balance snapshots...
EdgeX balance: $1000.00 (available: $950.00)
Lighter balance: $1000.00 (available: $920.00)

Opening delta-neutral position...
Position opened successfully!
EdgeX size: -0.05
Lighter size: 0.05
Now holding until 2025-10-03 20:00:00 UTC

Holding PAXG: 7.98h remaining, PnL: $+0.12
Holding PAXG: 6.98h remaining, PnL: $+0.25
...

═══════════════════════════════════════════════════════════════════
CLOSING POSITION: PAXG
═══════════════════════════════════════════════════════════════════
Unrealized PnL before close:
  EdgeX: $0.02
  Lighter: $0.03
  Total: $0.05

Position closed successfully!

Balance changes:
  EdgeX: $+0.45
  Lighter: $+0.22
  Total: $+0.67

═══════════════════════════════════════════════════════════════════
CYCLE #1 COMPLETE - PAXG
═══════════════════════════════════════════════════════════════════
Duration: 8.02 hours
Realized PnL: $+0.67 (+0.034%)
  EdgeX: $+0.45
  Lighter: $+0.22

Cumulative Stats:
  Total Cycles: 1
  Success Rate: 100.0%
  Total PnL: $+0.67
  Best Cycle: $+0.67
  Worst Cycle: $+0.67
  Avg PnL/Cycle: $+0.67
═══════════════════════════════════════════════════════════════════

Waiting 5.0 minutes before next cycle...
```

## 🐛 Troubleshooting

### Bot enters ERROR state
**Cause**: Position mismatch or unexpected error

**Solution**:
1. Check `logs/auto_rotation_bot.log` for details
2. Manually verify positions on both exchanges
3. Close any open positions manually
4. Delete `bot_state.json`
5. Restart bot

### "No symbols available on both exchanges"
**Cause**: Symbols in config not listed on one or both exchanges

**Solution**:
- Update `symbols_to_monitor` in config
- Check symbol names match exchange format
- Verify symbols are active on both EdgeX and Lighter

### Bot keeps entering WAITING state
**Cause**: Best opportunities below `min_net_apr_threshold`

**Solution**:
- Lower `min_net_apr_threshold` in config (e.g., from 5.0% to 3.0%)
- Wait for better funding rate conditions
- Add more symbols to `symbols_to_monitor`

### High fees eating into profits
**Cause**: Trading too frequently with small positions

**Solution**:
- Increase `notional_per_position` (larger positions = lower fee %)
- Increase `hold_duration_hours` (fewer cycles = fewer fees)
- Increase `min_net_apr_threshold` (only take best opportunities)

### Positions not delta-neutral after opening
**Cause**: Tick size rounding differences between exchanges

**Solution**:
- Bot automatically handles this via coarser tick size rounding
- Check `hedge_cli.log` for size calculation details
- If severe mismatch, bot will detect and enter ERROR state

## ⚙️ Advanced Usage

### Custom Config File
```bash
python auto_rotation_bot.py --config my_custom_config.json
```

### Custom State File
```bash
python auto_rotation_bot.py --state-file my_state.json
```

### Both Custom Files
```bash
python auto_rotation_bot.py --config my_config.json --state-file my_state.json
```

### Multiple Bots (Different Strategies)
```bash
# Conservative bot: higher threshold, longer holds
python auto_rotation_bot.py --config conservative_config.json --state-file conservative_state.json

# Aggressive bot: lower threshold, shorter holds
python auto_rotation_bot.py --config aggressive_config.json --state-file aggressive_state.json
```

## 🐳 Docker Support

The bot is pre-configured in `docker-compose.yml`:

```yaml
services:
  auto_rotation_bot:
    build: .
    command: python -u auto_rotation_bot.py
    volumes:
      - ./rotation_bot_config.json:/app/rotation_bot_config.json
      - ./hedge_cli.py:/app/hedge_cli.py
      - ./auto_rotation_bot.py:/app/auto_rotation_bot.py
      - ./logs:/app/logs
    env_file:
      - .env
    restart: unless-stopped
    environment:
      - BOT_STATE_FILE=/app/logs/bot_state.json
      - PYTHONUNBUFFERED=1
```

Run with:
```bash
# Start in background
docker-compose up -d auto_rotation_bot

# View logs (live)
docker-compose logs -f auto_rotation_bot

# Stop the bot
docker-compose stop auto_rotation_bot
```

## 📊 Performance Optimization

### Maximizing Profit

1. **Increase position size** (if you have capital)
   - Fees are percentage-based, so larger positions = better ROI
   - Example: $1000 position with 15% APR = $150/year vs $100 position = $15/year

2. **Optimize hold duration**
   - Too short = more fees from frequent cycling
   - Too long = miss better opportunities
   - Sweet spot: 6-12 hours (captures multiple funding periods)

3. **Smart symbol selection**
   - Include high-volatility assets (BTC, ETH) for better funding spreads
   - Include stablecoins (if available) for consistent small profits
   - Monitor 10-15 symbols for best selection

4. **Set appropriate APR threshold**
   - Too high = bot waits forever
   - Too low = takes unprofitable trades
   - Recommended: 5-10% net APR minimum

5. **Monitor and adjust**
   - Review `completed_cycles` in `bot_state.json`
   - Identify which symbols are most profitable
   - Adjust `symbols_to_monitor` accordingly

### Risk Management

1. **Start small**
   - Test with $50-100 positions first
   - Gradually increase as you gain confidence

2. **Monitor regularly**
   - Check bot status daily
   - Review cumulative PnL weekly
   - Investigate any ERROR states immediately

3. **Diversify**
   - Don't rely solely on funding arbitrage
   - Use this as one strategy among many

4. **Capital buffers**
   - Keep >20% available margin on both exchanges
   - Prevents liquidation during volatile periods
   - Consider running `liquidation_monitor.py` alongside

## 📝 Logging

- **Console**: INFO level, color-coded status updates
- **File**: `logs/auto_rotation_bot.log` - DEBUG level, full detail
- **Rotation**: Logs can grow large, consider logrotate setup

## 🔒 Security

- Never commit `.env` file to version control
- Keep API keys secure
- Use read-only API keys if available (bot needs trading permissions though)
- Monitor bot activity regularly
- Set up alerts for ERROR states

## 🤝 Integration with Other Tools

Works seamlessly with:
- **liquidation_monitor.py**: Monitors positions for liquidation risk
- **hedge_cli.py**: Manual position management
- **market_maker_v2.py**: Can run alongside (different markets)

## 📧 Support

For issues:
1. Check `logs/auto_rotation_bot.log` for detailed errors
2. Review `bot_state.json` for state information
3. Consult `CLAUDE.md` for technical architecture
4. Check exchange APIs for platform-specific issues

## ⚠️ Disclaimer

This bot is for educational and research purposes. Cryptocurrency trading involves significant risk. Past performance does not guarantee future results. Always test in dry-run mode first and start with small position sizes. The authors are not responsible for any financial losses.

---

**Happy arbitraging! 🚀**
