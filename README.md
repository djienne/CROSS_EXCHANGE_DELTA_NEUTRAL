# Cross-Exchange Delta-Neutral Hedging System

**Automated 24/7 funding rate arbitrage bot** for EdgeX and Lighter cryptocurrency perpetual futures exchanges.

This system continuously monitors multiple markets, executes delta-neutral positions to capture funding rate differences, and automatically rotates them to maximize profit while maintaining market-neutral exposure.

Referral link to support this work: https://pro.edgex.exchange/referral/FREQTRADE

## 🎯 Core Features

- 🤖 **Fully Automated 24/7 Trading**: The `auto_rotation_bot.py` runs continuously, requiring no manual intervention.
- 📈 **Intelligent Market Selection**: Analyzes a user-defined list of markets and always opens a position in the one with the highest net funding APR.
- 🔄 **Automatic Position Rotation**: Opens a delta-neutral position, holds it for a configurable duration (e.g., 8 hours) to collect funding, then closes and rotates to the next best opportunity.
- 🛡️ **Stop-Loss Protection**: Automatically closes positions if a leg's loss exceeds a defined percentage of the notional value.
- 💥 **Crash Recovery & State Persistence**: Saves bot state, including cycle history and PnL. Can recover from restarts and reconcile existing positions.
- 🖥️ **Real-time Monitoring**: A clean terminal dashboard shows the current position, PnL, available capital, and top funding opportunities.

## ⚠️ Important: Manual Fund Rebalancing

This bot **cannot** automatically rebalance funds between your Lighter and EdgeX accounts. Due to the nature of hedging, one account will accumulate profits while the other incurs losses.

You must **manually rebalance** your capital between the two exchanges periodically. This should be done **when the bot is stopped** and involves withdrawing funds from one exchange and depositing to the other, a process that requires manual blockchain transactions.

## 🚀 Quick Start (Automated Bot)

### 1. Installation

```bash
# Navigate to the project directory
cd /path/to/CROSS_EXCHANGE_DELTA_NEUTRAL

# Install dependencies
pip install -r requirements.txt
```

### 2. Configure API Credentials

Copy the example environment file and add your API keys.

```bash
cp .env.example .env
# Edit .env with your actual credentials
```

### 3. Configure the Rotation Bot

Edit `rotation_bot_config.json` to define your strategy.

```json
{
  "symbols_to_monitor": ["BTC", "ETH", "SOL", "PAXG", "HYPE", "XPL"],
  "quote": "USD",
  "leverage": 3,
  "notional_per_position": 320.0,
  "hold_duration_hours": 8.0,
  "min_net_apr_threshold": 5.0,
  "stop_loss_percent": 25.0
}
```
- `symbols_to_monitor`: More symbols provide more opportunities.
- `notional_per_position`: Max position size. The bot uses the lesser of this value or your available capital.
- `leverage`: Recommended: 3-5x.
- `stop_loss_percent`: Safety threshold. Recommended: 25% for 3x leverage.

### 4. Run the Bot

```bash
# Start the bot directly
python auto_rotation_bot.py

# Or run with Docker for 24/7 operation (recommended)
docker-compose up -d auto_rotation_bot

# View live logs
docker-compose logs -f auto_rotation_bot
```
The bot will start, reconcile any existing state, and begin its analysis-trade-rotate cycle.

### 5. Monitor the Bot

<img src="rotation_bot.png" alt="Rotation Bot Terminal Output" width="800">

The dashboard displays the current cycle, PnL, capital, top funding opportunities, and time until the next rotation.

---

## 🔧 Advanced Usage & Details

<details>
<summary><b>⚙️ Full Configuration Details</b></summary>

### `rotation_bot_config.json`

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `symbols_to_monitor` | array | `["BTC", "ETH", ...]` | List of symbols to analyze for funding opportunities |
| `quote` | string | `"USD"` | Quote currency for all markets |
| `leverage` | number | `3` | Leverage to use on both exchanges for all positions |
| `notional_per_position` | number | `320.0` | Maximum position size in USD (bot adjusts to actual available capital) |
| `hold_duration_hours` | number | `8.0` | How long to hold each position before closing (hours) |
| `wait_between_cycles_minutes` | number | `5.0` | Cooldown period between closing one position and opening the next (minutes) |
| `check_interval_seconds` | number | `300` | How often to check position health while holding (seconds, default: 5 minutes) |
| `min_net_apr_threshold` | number | `5.0` | Minimum net APR required to open a position (%) |
| `stop_loss_percent` | number | `25.0` | Stop-loss threshold as % of position notional (triggers on either leg) |
| `enable_stop_loss` | boolean | `true` | Enable automatic stop-loss protection |

### `.env` Environment Variables

- **EdgeX**: `EDGEX_BASE_URL`, `EDGEX_WS_URL`, `EDGEX_ACCOUNT_ID`, `EDGEX_STARK_PRIVATE_KEY`
- **Lighter**: `LIGHTER_BASE_URL`, `LIGHTER_WS_URL`, `API_KEY_PRIVATE_KEY`, `ACCOUNT_INDEX`, `API_KEY_INDEX`

**Note:** Margin mode is hardcoded to "cross" for delta-neutral hedging.

</details>

<details>
<summary><b>🛠️ Manual Trading CLI (`hedge_cli.py`)</b></summary>

For manual analysis and trading, use `hedge_cli.py`. This tool is useful for testing or when you need direct control. It uses `hedge_config.json` for its parameters.

**Key Commands:**

| Command | Description |
|---------|-------------|
| `funding_all` | Compare funding rates across all markets |
| `funding` | Check funding rates for the configured symbol |
| `capacity` | Calculate max position size from available capital |
| `status` | Check current position status on both exchanges |
| `open` | Open a delta-neutral position |
| `close` | Close both positions |
| `test` | Run a small, self-closing test trade |

**Example:**
```bash
# Check funding for PAXG and auto-configure long/short exchanges
python hedge_cli.py funding --config hedge_config.json

# Open a $100 position in the configured market
python hedge_cli.py open --size-quote 100 --config hedge_config.json
```

</details>

<details>
<summary><b>🐳 Docker Details</b></summary>

The `docker-compose.yml` is the easiest way to run the bot 24/7.

**Primary Service:**
```bash
# Start the automated bot in the background
docker-compose up -d auto_rotation_bot

# View live logs
docker-compose logs -f auto_rotation_bot

# Stop the bot
docker-compose stop auto_rotation_bot
```

Other services for manual commands (`open`, `close`, `funding`, etc.) and the `liquidation_monitor` are included but commented out in `docker-compose.yml`. Uncomment them to use them via `docker-compose run <service_name>`.

</details>

<details>
<summary><b>🎓 How It Works (Technical Summary)</b></summary>

- **Funding Rate Arbitrage**: The bot shorts the exchange with a higher funding rate and longs the one with a lower rate, profiting from the difference while remaining price-neutral.
- **Position Sizing**: It automatically calculates the largest possible identical position size that respects the tick size rules of both exchanges, preventing mismatches.
- **Order Execution**: It uses aggressive limit orders that cross the spread to ensure immediate execution, simulating market orders but with price protection. Orders are placed concurrently to minimize timing risk.

</details>

<details>
<summary><b>🛡️ Optional Liquidation Monitor</b></summary>

An optional, standalone service (`liquidation_monitor.py`) can run alongside the main bot to provide an extra layer of safety.

- Monitors margin ratios on both exchanges.
- Automatically closes positions if the margin ratio exceeds a safety threshold (default: 80%).
- Detects and flags unhedged (one-sided) positions.

**Run via Python:**
```bash
python liquidation_monitor.py --interval 60 --margin-threshold 80.0
```

**Run via Docker:**
```bash
# First, uncomment the 'liquidation_monitor' service in docker-compose.yml
docker-compose up -d liquidation_monitor
```

</details>

## ⚠️ Risk Management

- ⚠️ **Start small.** Test the system with a small amount of capital ($50-100) that you are willing to lose.
- ⚠️ **Monitor actively.** Especially during the first few trading cycles.
- ⚠️ **Leverage is risky.** It amplifies both gains and losses.
- ⚠️ **Network failures can happen.** The bot is designed to detect if one leg of a trade fails, but you should be prepared to intervene manually.
- ⚠️ **Maintain a margin buffer.** Keep extra capital in your accounts (>20%) to avoid liquidation during normal price fluctuations.

<details>
<summary><b>🔍 Troubleshooting</b></summary>

- **Bot enters ERROR state**: Check `logs/auto_rotation_bot.log`. If state is corrupt, you may need to delete `logs/bot_state.json` and restart.
- **"Computed size rounds to zero"**: Your `notional_per_position` is too small for the instrument's minimum size, or you have insufficient capital.
- **"Leverage setting failed"**: Verify API keys and exchange permissions.
- **Bot is waiting, not trading**: The `min_net_apr_threshold` may be too high for current market conditions, or you may need to add more symbols to `symbols_to_monitor`.

</details>
