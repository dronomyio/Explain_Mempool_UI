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
### 2. Total Value
```
totalValue = txs.reduce((s, t) => s + t.value, 0)

What it is: Sum of value field across all buffered transactions, displayed in millions ($6.1M).
What value is: This is tx.value from the alert payload — which is the ETH value of the transaction converted to USD. 
Currently shows $0 on most alerts because value: 0.0 — the worker sets size (USD value) but the page reads value which 
is the raw ETH transfer amount (0 for ERC20 swaps that don't transfer ETH directly).
The fix needed: The page should read size (the USD-denominated value) not value. Currently size is always 0 because the 
value decode isn't wired up for ERC20 token swaps — only native ETH transfers have non-zero value.
```
### 3. DEX swaps
```
swapCount = txs.filter(t => t.type === 'swap').length

What it is: Count of transactions where type === 'swap'. The worker sets type based on the selector match 
from the 16-selector DEX map (Uniswap V2/V3, 1inch etc).
Why it matches Buffered TXs: Almost everything in the buffer is a swap because the cliff score threshold filters 
to DEX-relevant transactions. Non-swap transactions (pure ETH transfers, ERC20 approvals) score too low to get 
through unless they have strong nonce signals.
```
### 4. liqudations
```
liquidationCount = txs.filter(t => t.type === 'liquidation').length

What it is: Count where type === 'liquidation'. The worker detects liquidations via specific function selectors for Aave,
Compound, and similar protocols.
Why it's almost always 0: Liquidations are rare events — Ethereum mainnet has maybe 10-50 liquidations per day during 
normal market conditions. The buffer holds 200 transactions so statistically only 1-2 might be liquidations at any given time. 
During high volatility (crypto crash) this would jump significantly.
```
### 5. avg gas
```
avgGasPrice = txs.reduce((s, t) => s + t.gasPrice, 0) / txs.length

What it is: Simple arithmetic mean of gasPrice across all buffered transactions, displayed in gwei.
Note: This is the gasPrice field from the alert payload which is effective_gas_price_gwei — the actual gas price for EIP-1559 transactions. 
The Gas Tracker panel shows a smoother version using EMA (exponential moving average with α=0.3) derived from the streaming 
baseFee and priorityFee separately.
Why it shows low values (0.1-3.3 gwei): Most alerts in the buffer are low-priority transactions. The highest-gas sandwich bots 
use much higher gas but they also get confirmed quickly and leave the mempool fast. What lingers in the mempool
(and gets captured as pending) tends to be lower-gas transactions.
```
### 6. high risk how it's calculated?
```
What it is: Count where mevRisk is high or critical. The worker assigns mevRisk based on cliff_score thresholds:
#### In mempool_alert_worker.py
if cliff_score >= 0.8:   mev_risk = 'critical'
elif cliff_score >= 0.5: mev_risk = 'high'
elif cliff_score >= 0.3: mev_risk = 'medium'
else:                    mev_risk = 'low'

The real fix: mevRisk='high' should only fire when multiple components of cliff_score are elevated — not just the nonce signal. Currently:

nonce_signal = 0.5 (nonce gap detected)
cliff_score  = 0.10 * 0.5 = 0.05 ... wait that's wrong

Actually the issue is the nonce component alone contributes 0.10 × 0.5 = 0.05 to cliff_score, but cliff_score=0.325 means 
the LOB cliff component (0.25 × 1.0 = 0.25) is also firing — meaning the transaction IS targeting a registered pool with a 
real LOB cliff. So mevRisk='high' is technically correct — these are swaps hitting pools with liquidity cliffs. The visual 
confusion is that 197/200 transactions being "high risk" looks like noise, but it reflects genuine bot activity on monitored pools.


```
Let me read the exact calculation for each stat card from the live page.tsx:Now I have the exact calculations. Here's a precise breakdown of all six:

---

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
