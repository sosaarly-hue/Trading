# Deployment & Testing Guide

## Local Testing (Dry Run)

### 1. Setup

```bash
cd polymarket_bot
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 2. Configure for Testing

Edit `config.yaml`:
```yaml
capital:
  initial_usdc: 100              # Start small
  per_trade_allocation: 10       # Very conservative

market_making:
  base_half_spread_cents: 3      # Wider spread = fewer fills (safer)

risk:
  kelly_fraction: 0.25           # Quarter Kelly (conservative)
  daily_loss_limit_pct: 0.02     # Halt on -2% day
```

### 3. Test Individual Modules

```bash
# Test market making engine
python3 -c "
import logging
logging.basicConfig(level=logging.INFO)
from core.mm_engine import test_mm_engine
test_mm_engine()
"

# Test Kelly sizing
python3 -c "
import logging
logging.basicConfig(level=logging.INFO)
from risk.kelly_sizing import test_kelly_sizing
test_kelly_sizing()
"

# Test market database
python3 -c "
import asyncio
import logging
logging.basicConfig(level=logging.INFO)
from core.market_database import test_db
asyncio.run(test_db())
"
```

### 4. Dry Run (No Real Trades)

The current implementation is in **dry-run mode** — quotes are generated but orders are NOT placed.

```bash
# Set test credentials (or dummy values)
export POLY_PRIVATE_KEY="0000000000000000000000000000000000000000"
export POLY_API_KEY="test_key"
export POLY_API_SECRET="test_secret"
export POLY_PASSPHRASE="test_pass"

# Run
python main.py

# Expected output:
# 2026-04-18 14:23:45 | __main__ | INFO | Starting Polymarket trading bot
# 2026-04-18 14:23:45 | __main__ | INFO | Initial capital: $10000
# 2026-04-18 14:23:46 | ... | INFO | WebSocket connected
# 2026-04-18 14:23:47 | ... | INFO | Stage 1: Infrastructure initialization
```

---

## Testnet Deployment (Polygon Mumbai)

### 1. Get Mumbai USDC

1. Go to https://faucet.polymarket.com or https://aavefaucet.com
2. Request Mumbai USDC
3. Get test funds in your wallet

### 2. Create Testnet Bot

Edit `config.yaml` for testnet:
```yaml
polymarket:
  api_url: "https://mumbai-clob-api.polymarket.com"  # Testnet endpoint
  ws_url: "wss://mumbai-ws.polymarket.com"

capital:
  initial_usdc: 50               # Very small for testnet
```

### 3. Run on Testnet

```bash
export POLY_PRIVATE_KEY="your_mumbai_wallet_private_key"
export POLY_API_KEY="your_testnet_api_key"
export POLY_API_SECRET="your_testnet_secret"
export POLY_PASSPHRASE="your_testnet_passphrase"

python main.py
```

---

## Mainnet Deployment (Production)

### ⚠️ Checklist Before Going Live

- [ ] Tested on testnet for 48+ hours without bugs
- [ ] Backtested on historical data (if available)
- [ ] Kelly sizing implemented correctly (quarter Kelly only)
- [ ] Daily loss limits in place
- [ ] Position limits enforced
- [ ] Monitoring/alerting setup
- [ ] Reviewed all order logic 3x
- [ ] Emergency stop procedure documented
- [ ] Started with <$1000 capital
- [ ] Watched bot for 3+ hours before leaving unattended

### 1. Production Config

```yaml
capital:
  initial_usdc: 1000             # Start with $1k MINIMUM

market_making:
  base_half_spread_cents: 3      # Conservative

risk:
  kelly_fraction: 0.25           # NEVER use full Kelly
  daily_loss_limit_pct: 0.01     # Halt on -1% day

logging:
  telegram_enabled: true         # Get alerts
```

### 2. Real Credentials

```bash
# Use hardware wallet or Gnosis Safe key
export POLY_PRIVATE_KEY="your_mainnet_key"
export POLY_API_KEY="your_prod_api_key"
export POLY_API_SECRET="your_prod_secret"
export POLY_PASSPHRASE="your_prod_passphrase"

# Telegram alerts
export TELEGRAM_BOT_TOKEN="your_bot_token"
export TELEGRAM_CHAT_ID="your_chat_id"
```

### 3. Run on Mainnet

```bash
python main.py > logs/bot_$(date +'%Y%m%d_%H%M%S').log 2>&1 &
```

### 4. Monitoring

Monitor these metrics every 15 minutes:

```python
# Check bot health
portfolio = engine.get_portfolio_state()
if portfolio["daily_pnl"] < -capital * 0.01:
    logger.CRITICAL("Daily loss limit hit—stopping bot")
    # Emergency exit
```

---

## Emergency Procedures

### Stop All Trading

```bash
# Kill process
pkill -f "python main.py"

# Check open positions
# (Implement position checker in production)
```

### Manual Order Cancellation

```python
from core.polymarket_client import PolymarketClient, PolymarketAuth
import asyncio

async def cancel_all():
    auth = PolymarketAuth.from_env()
    async with PolymarketClient(auth) as client:
        orders = await client.get_open_orders()
        for order in orders:
            await client.cancel_order(order['id'])
            print(f"Cancelled {order['id']}")

asyncio.run(cancel_all())
```

---

## Monitoring & Logging

### Log Levels

| Level | When to Use |
|-------|------------|
| DEBUG | Every tick (orderbook update) - verbose |
| INFO | Important events (fills, quotes, P&L) |
| WARNING | Edge cases (stale data, low edge) |
| ERROR | Failed orders, API errors |
| CRITICAL | Daily loss limit hit, emergency |

### Key Metrics to Log

```python
# Every minute
"timestamp": datetime.utcnow(),
"active_markets": len(self.mm_engine.inventory),
"total_exposure": sum(abs(pos) for pos in positions),
"exposure_pct": exposure / capital,
"daily_pnl": pnl,
"daily_pnl_pct": pnl / capital,

# Every order
"order_id": order_id,
"market_id": market_id,
"side": side,
"price": price,
"size": size,
"timestamp": datetime.utcnow(),

# Every fill
"fill_id": fill_id,
"order_id": order_id,
"side": side,
"price": price,
"size": size,
"pnl": pnl,
"timestamp": datetime.utcnow(),
```

---

## Scaling Strategy

### Phase 1: Bootstrap ($100-1k)
- Only market making on geopolitics (0% fee)
- 3-5 markets max
- Wide spreads (3-4 cents)
- Monitor for 1-2 weeks

### Phase 2: Validate ($1k-10k)
- Add logical arb on sports/politics
- Expand to 10+ markets
- Tighten spreads (2-3 cents)
- Track win rate, Sharpe ratio

### Phase 3: Optimize ($10k-100k)
- Implement AI probability arb
- Full market graph traversal
- Dynamic spread optimization
- 50+ markets

### Phase 4: Scale ($100k+)
- Multi-venue (Kalshi, Manifold cross-arb)
- Advanced ML for probability estimation
- High-frequency quoting (sub-second refresh)
- Dedicated infrastructure

---

## Common Issues & Fixes

### WebSocket disconnects frequently
- Check internet connection
- Increase staleness threshold in config
- Implement exponential backoff reconnection

### Orders not filling
- Spread too tight (bid/ask crossing mid)
- Not enough liquidity in market
- Check maker rebate eligibility

### Negative P&L
- Fees too high—focus on geopolitics (0% fee)
- Estimation error in prices—widen spreads
- Adverse selection—check fill distribution

### High latency
- Ensure network is stable
- Move bot to cloud (AWS, GCP near Polygon)
- Optimize WebSocket connections (reuse, compression)

---

##  Production Checklist

- [ ] 48+ hours on testnet without crashes
- [ ] All limits enforced and tested
- [ ] Emergency stop procedure works
- [ ] Monitoring/alerts configured
- [ ] Started with $1,000 or less
- [ ] Watched bot for 3+ hours before leaving
- [ ] Daily reconciliation process in place
- [ ] API keys rotated monthly
- [ ] Backup wallet for emergency liquidity
- [ ] Insurance/hedges considered

---

**Remember: Start small, test thoroughly, scale gradually!**
