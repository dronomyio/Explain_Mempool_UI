# Explain_Mempool_UI

# 1. what does 
### 1. buffered Txs 
```
txCount = txs.length

What it is: Count of transactions currently in the in-memory buffer (MAX_TX_BUFFER = 200). Every alert that arrives via SSE above
the min_cliff threshold gets prepended to the buffer. When it exceeds 200, the oldest are dropped.
What it's NOT: Total processed by the worker (that's 58M+). This is only what the browser has received since the page loaded — or
since the prefetch seeded it.
Why it's often low: The buffer fills in real-time from the SSE stream. On first load it's now seeded with 50 from the prefetch.
It grows as new alerts arrive.

'''


## 1. Buffered TXs

```typescript
txCount = txs.length
```

**What it is:** Count of transactions currently in the in-memory buffer (`MAX_TX_BUFFER = 200`). Every alert that arrives via SSE above the `min_cliff` threshold gets prepended to the buffer. When it exceeds 200, the oldest are dropped.

**What it's NOT:** Total processed by the worker (that's 58M+). This is only what the browser has received since the page loaded — or since the prefetch seeded it.

**Why it's often low:** The buffer fills in real-time from the SSE stream. On first load it's now seeded with 50 from the prefetch. It grows as new alerts arrive.

---

## 2. Total Value

```typescript
totalValue = txs.reduce((s, t) => s + t.value, 0)
```

**What it is:** Sum of `value` field across all buffered transactions, displayed in millions (`$6.1M`).

**What `value` is:** This is `tx.value` from the alert payload — which is the ETH value of the transaction converted to USD. Currently shows `$0` on most alerts because `value: 0.0` — the worker sets `size` (USD value) but the page reads `value` which is the raw ETH transfer amount (0 for ERC20 swaps that don't transfer ETH directly).

**The fix needed:** The page should read `size` (the USD-denominated value) not `value`. Currently `size` is always 0 because the value decode isn't wired up for ERC20 token swaps — only native ETH transfers have non-zero value.

---

## 3. DEX Swaps

```typescript
swapCount = txs.filter(t => t.type === 'swap').length
```

**What it is:** Count of transactions where `type === 'swap'`. The worker sets `type` based on the selector match from the 16-selector DEX map (Uniswap V2/V3, 1inch etc).

**Why it matches Buffered TXs:** Almost everything in the buffer is a swap because the cliff score threshold filters to DEX-relevant transactions. Non-swap transactions (pure ETH transfers, ERC20 approvals) score too low to get through unless they have strong nonce signals.

---

## 4. Liquidations

```typescript
liquidationCount = txs.filter(t => t.type === 'liquidation').length
```

**What it is:** Count where `type === 'liquidation'`. The worker detects liquidations via specific function selectors for Aave, Compound, and similar protocols.

**Why it's almost always 0:** Liquidations are rare events — Ethereum mainnet has maybe 10-50 liquidations per day during normal market conditions. The buffer holds 200 transactions so statistically only 1-2 might be liquidations at any given time. During high volatility (crypto crash) this would jump significantly.

---

## 5. Avg Gas

```typescript
avgGasPrice = txs.reduce((s, t) => s + t.gasPrice, 0) / txs.length
```

**What it is:** Simple arithmetic mean of `gasPrice` across all buffered transactions, displayed in gwei.

**Note:** This is the `gasPrice` field from the alert payload which is `effective_gas_price_gwei` — the actual gas price for EIP-1559 transactions. The Gas Tracker panel shows a smoother version using EMA (exponential moving average with `α=0.3`) derived from the streaming `baseFee` and `priorityFee` separately.

**Why it shows low values (0.1-3.3 gwei):** Most alerts in the buffer are low-priority transactions. The highest-gas sandwich bots use much higher gas but they also get confirmed quickly and leave the mempool fast. What lingers in the mempool (and gets captured as pending) tends to be lower-gas transactions.

---

## 6. High Risk

```typescript
highRiskCount = txs.filter(
  t => t.mevRisk === 'high' || t.mevRisk === 'critical'
).length
```

**What it is:** Count where `mevRisk` is `high` or `critical`. The worker assigns `mevRisk` based on `cliff_score` thresholds:

```python
# In mempool_alert_worker.py
if cliff_score >= 0.8:   mev_risk = 'critical'
elif cliff_score >= 0.5: mev_risk = 'high'
elif cliff_score >= 0.3: mev_risk = 'medium'
else:                    mev_risk = 'low'
```

**Why it's almost always equal to Buffered TXs:** Because the SSE filter is `min_cliff=0.3` and `mevRisk='high'` starts at `cliff_score=0.5`. But the nonce gap signal alone gives `cliff_score=0.325` → `mevRisk='high'`. So nearly everything that passes the 0.3 threshold also gets tagged `high` from the nonce component alone, even if the transaction isn't actually dangerous.

**The real fix:** `mevRisk='high'` should only fire when multiple components of cliff_score are elevated — not just the nonce signal. Currently:

```
nonce_signal = 0.5 (nonce gap detected)
cliff_score  = 0.10 * 0.5 = 0.05 ... wait that's wrong
```

Actually the issue is the nonce component alone contributes `0.10 × 0.5 = 0.05` to cliff_score, but `cliff_score=0.325` means the LOB cliff component (`0.25 × 1.0 = 0.25`) is also firing — meaning the transaction IS targeting a registered pool with a real LOB cliff. So `mevRisk='high'` is technically correct — these are swaps hitting pools with liquidity cliffs. The visual confusion is that 197/200 transactions being "high risk" looks like noise, but it reflects genuine bot activity on monitored pools.

---

## Summary table

| Stat | Source field | Computed from | Current accuracy |
|---|---|---|---|
| Buffered TXs | `txs.length` | SSE buffer | Correct — grows with alerts |
| Total Value | `tx.value` | Raw ETH transfer | **Wrong** — should use `tx.size` |
| DEX Swaps | `tx.type === 'swap'` | Worker selector match | Correct |
| Liquidations | `tx.type === 'liquidation'` | Worker selector match | Correct — rare by nature |
| Avg Gas | mean of `tx.gasPrice` | Alert payload gasPrice | Correct but biased low |
| High Risk | `mevRisk === 'high'/'critical'` | cliff_score threshold | Technically correct but noisy |

The most impactful fix is **Total Value** — change `t.value` to `t.size` in the `useMempoolStats` function. Want me to patch that?


-------------------------------------------------------------------------------------------

# these are swaps hitting pools with liquidity cliffs. The visual confusion is that 197/200 transactions being "high risk" looks like noise, but it reflects genuine bot activity on monitored pools. what is the monitored pool?

Good question. "Monitored pools" means pools registered in your `pool_registry` table in Vertica — the ones the `lob_materializer` cron fetches LOB (liquidity order book) tick state for.

Check what's actually registered right now:

```bash
# Check pool registry in Vertica
vsql -h 192.168.1.140 -U dbadmin -d testdb -c \
  "SELECT pool_address, token0_symbol, token1_symbol, fee_tier 
   FROM uniswap.pool_registry 
   ORDER BY token0_symbol
   LIMIT 20"
```

Or check what pools the worker has cached:

```bash
docker compose logs mempool-worker | grep "pool_cache\|Pool token cache" | tail -5
```

From the worker stats you've been seeing: `pool_cache=50 pools` — so 50 pools are registered. These are the pools where the LOB cliff component of cliff_score activates.

**What the LOB cliff component does:**

When a transaction targets a registered pool, the worker fetches the current tick state from Vertica (`vw_lob_cliffs`) and checks how close the current price is to the nearest liquidity cliff — a price level where liquidity drops suddenly. Transactions near a cliff are more vulnerable to sandwich attacks because the price impact per dollar is higher there.

```
lob_cliff = 0.0   → transaction not targeting a registered pool
                    OR pool is registered but far from any cliff
lob_cliff = 1.0   → transaction is very close to a liquidity cliff
                    (high sandwich incentive for bots)
```

**The cliff score formula:**
```
cliff_score = 0.25 × lob_cliff      ← only fires for registered pools
            + 0.20 × vpin_imbalance
            + 0.15 × gas_z_score
            + 0.10 × nonce_signal
            + 0.15 × hawkes_score
            + 0.15 × kyle_score
```

So if `lob_cliff = 1.0` and `nonce_signal = 0.5`, cliff_score = `0.25 + 0.05 = 0.30` — right at the threshold. That's exactly what you're seeing: `cliffScore: 0.325` on most alerts.

**To see which specific pools are monitored:**

```bash
# From the worker pool cache
docker exec mempool-worker python3 -c "
import json, redis
r = redis.from_url('redis://192.168.1.140:6379')
keys = r.keys('pool_token:*')
print(f'Registered pools: {len(keys)}')
for k in keys[:10]:
    print(r.get(k))
" 2>/dev/null
```

Or from Vertica:

```bash
vsql -h 192.168.1.140 -U dbadmin -d testdb -c \
  "SELECT pool_address, token0_symbol, token1_symbol 
   FROM uniswap.pool_registry LIMIT 10"
```

The 197/200 being high risk means almost every alert is hitting one of these 50 registered pools near a liquidity cliff — which makes sense because your pool registry likely contains the highest-volume Uniswap V3 pools (USDC/WETH, WETH/USDT etc) which are exactly the pools bots target most aggressively.

----------------------------------------------------------------------------

# explain : 1. GAS TRACKEREIP-1559 · live from mempool
```
1. Avg Gas
37.2
gwei

1. Priority
0.35
gwei

1. Est. Next
39.1
↑ 5.0%

1. Est. Block Pressure74% what are these and how do you get them?
```
Let me read the exact gas tracker calculations:Now I have the full picture. Here's exactly what each number is, where it comes from, and how accurate it is:

---

## 1. Avg Gas (37.2 gwei) — `baseFee`

**What it is:** An exponential moving average (EMA) of `gasPrice` from incoming mempool alerts.

**How computed:**
```typescript
α = 0.3   // smoothing factor

// On every incoming alert:
ema.baseFee = α × tx.gasPrice + (1 - α) × ema.baseFee
// = 30% new value + 70% historical average
```

**What `tx.gasPrice` actually is:** The `effective_gas_price_gwei` from the alert payload — the total gas price the transaction is willing to pay, combining base fee + priority fee.

**Accuracy issue:** This is NOT the actual Ethereum base fee from the block header. It's inferred from pending transaction gas prices in the mempool. It's a reasonable approximation but will lag behind the true base fee and be biased by which transactions happen to score above your cliff threshold. Low-value transactions with low gas dominate the buffer, pulling the average down.

**True source would be:** `eth_getBlockByNumber("latest")` → `baseFeePerGas` field. That's the authoritative number.

---

## 2. Priority (0.35 gwei) — `priorityFee`

**What it is:** EMA of `maxPriorityFee` (the miner tip) from incoming alerts.

**How computed:**
```typescript
ema.priority = α × tx.maxPriorityFee + (1 - α) × ema.priority
```

**What `maxPriorityFee` is:** EIP-1559 introduced a two-part gas price — `baseFeePerGas` (burned, set by the network) + `maxPriorityFeePerGas` (paid to the validator as a tip). The priority fee is what validators actually receive on top of the base fee.

**Normal range:** 0.01–2 gwei for regular transactions. During congestion or NFT mints: 10–100+ gwei. Sandwich bots during active attacks: 50–500 gwei (they need to guarantee block inclusion ahead of the victim).

**Why yours shows 0.35 gwei:** Your buffer is dominated by regular low-priority swaps. The high-gas sandwich bot transactions confirm quickly and leave the mempool before the next poll cycle captures them.

---

## 3. Est. Next (39.1 ↑ 5.0%) — `nextBaseFee`

**What it is:** An estimate of what the next block's base fee will be.

**How computed:**
```typescript
nextBaseFee = ema.baseFee × 1.05
// Always +5% of current base fee
```

**The real EIP-1559 formula:** The protocol adjusts base fee by up to ±12.5% per block based on how full the previous block was:
```
if block_gas_used > target (50% full):
    nextBaseFee = baseFee × (1 + 0.125 × (gas_used/target - 1))
if block_gas_used < target:
    nextBaseFee = baseFee × (1 - 0.125 × (1 - gas_used/target))
```

**The arrow direction:** `↑` if `nextBaseFee > baseFee`, `↓` if lower. The percentage shown is `|(nextBaseFee - baseFee) / baseFee| × 100`.

**Accuracy:** The `×1.05` constant is a rough guess — it assumes the block is about 90% full (which would give roughly +5% adjustment). The real number depends on actual block utilisation which you don't currently fetch from the chain.

---

## 4. Est. Block Pressure 74% — `gasUsedRatio`

**What it is:** An estimate of how full the current Ethereum block is (0% = empty, 100% = completely full at 30M gas).

**How computed:**
```typescript
gasUsedRatio = Math.min(1, ema.baseFee / 50)
// Divides current gas price by 50 gwei as a "full block" proxy
```

**The logic:** Historically, gas prices around 50 gwei correspond to heavily congested blocks (~100% full). So `gasPrice / 50` gives a rough utilisation estimate. At 37.2 gwei → `37.2/50 = 74%`.

**Accuracy:** This is a proxy, not a real measurement. The true value comes from `eth_getBlockByNumber("latest")` → `gasUsed / gasLimit`. At 15M gas used out of 30M limit = 50% pressure. Your 74% from the formula is plausible but not precise.

**The bar color:** Green below 50%, orange 50–80%, red above 80%.

---

## Summary — what's real vs estimated

| Display | Source | Method | Accuracy |
|---|---|---|---|
| Avg Gas 37.2 | `tx.gasPrice` from alerts | EMA α=0.3 | ~Approximate — biased by alert filter |
| Priority 0.35 | `tx.maxPriorityFee` | EMA α=0.3 | ~Approximate — low end biased |
| Est. Next 39.1 ↑5% | `baseFee × 1.05` | Fixed multiplier | Rough estimate |
| Block Pressure 74% | `baseFee / 50` | Linear proxy | Rough estimate |

---

## How to make these accurate

The proper fix is one additional data source — a background poll of `eth_getBlockByNumber("latest")` every 12 seconds (one block):

```typescript
// In the api-server or page.tsx
async function fetchBlockGasStats() {
  const res = await fetch("https://ethereum.publicnode.com", {
    method: "POST",
    body: JSON.stringify({
      jsonrpc: "2.0", method: "eth_getBlockByNumber",
      params: ["latest", false], id: 1
    })
  });
  const { result } = await res.json();
  return {
    baseFee:      parseInt(result.baseFeePerGas, 16) / 1e9,   // true base fee gwei
    gasUsed:      parseInt(result.gasUsed, 16),
    gasLimit:     parseInt(result.gasLimit, 16),
    utilisation:  parseInt(result.gasUsed, 16) / parseInt(result.gasLimit, 16),
    // nextBaseFee: compute from EIP-1559 formula
  };
}
```

This would give you exact numbers from the Ethereum protocol itself rather than approximations from pending transaction prices. 

