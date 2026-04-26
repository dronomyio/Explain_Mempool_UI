# Explain_Mempool_UI

# 1. what does 
### 1. buffered Txs 
```
What it is: Count of transactions currently in the in-memory buffer (MAX_TX_BUFFER = 200). Every alert that arrives via SSE above the min_cliff threshold gets prepended to the buffer. When it exceeds 200, the oldest are dropped.
What it's NOT: Total processed by the worker (that's 58M+). This is only what the browser has received since the page loaded — or since the prefetch seeded it.
Why it's often low: The buffer fills in real-time from the SSE stream. On first load it's now seeded with 50 from the prefetch. It grows as new alerts arrive.

'''
### 2. Total Value 
### 3. DEX swaps 
### 4. liqudations 
### 5. avg gas 
### 6. high risk how it's calculated?
