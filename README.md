# Smooth Bitcoin T.H.S. (SBTC)

## Parameters (user set at fork)

| Parameter | Description | Default |
| --- | --- | --- |
| `S` | Total supply in billions | `1` |
| `H` | % of supply emitted in first T years | `50` |
| `T` | Emission period in years | `4` |

## Derived at Startup (cached, never recomputed)

```toml
blocks_per_year = 52,596
N = T × blocks_per_year
k = -ln(1 - H/100) / N
R = k × S × 10⁹
```

## Emission Formula

```toml
reward(block_height) = R × e^(-k × block_height)
```

---

## Implementation Steps

- [X] **Step 1 — Identity**

  ```yaml
  File:    chainparams.cpp
  Change:  Address prefix bc1 → sbc1
  Touches: bech32 prefix string
  Difficulty: trivial
  ```

- [X] **Step 2 — Display Unit** (Abandoned)

  ```yaml
  Change:  Display alias: cent at 10⁻²
           satoshis internal precision unchanged at 10⁻⁸
  Difficulty: trivial
  ```

- [ ] **Step 3 — Supply**

  ```yaml
  Change:  MAX_MONEY from 21,000,000 to S × 10⁹
  Difficulty: trivial
  ```

- [ ] **Step 4 — Fixed Fee Rate**

  ```yaml
  Change:  Remove fee market and mempool fee estimation
           Enforce fixed rate per byte
           100% of fees go to miner (lock it)
  Difficulty: low
  ```

- [ ] **Step 5 — Emission Curve**

  ```yaml
  Remove:  50 × (1/2)^(block_height/210000)
  Replace: R × e^(-k × block_height)
           Compute and cache k, R at startup from S, H, T
  Difficulty: low
  ```

- [ ] **Step 6 — Mempool Fair Queuing**

  ```yaml
  Change:  Remove fee priority ordering
           Implement fair queuing by sender address:
             - track per-address queue depth
             - fewer queued = higher priority
             - FIFO preserved within each address queue
  Difficulty: medium
  ```

- [ ] **Step 7 — Strict Miner Ordering**

  ```yaml
  Change:  Enforce block transaction order matches
           fair queue order exactly
           Reject blocks where miner deviated
  Difficulty: medium
  ```

- [ ] **Step 8 — Stealth Addresses**

  ```yaml
  Change:  Receiver-side stealth addresses:
             - sender derives one-time address from
               recipient's published meta-address
             - recipient scans chain to identify own UTXOs
           Does not affect sender address visibility
           Does not affect fair queuing (sender-side)
  Difficulty: hard
  ```

- [ ] **Step 9 — Genesis Block**

  ```yaml
  Message: "2026: Same rules for every transaction,
            every address, every person."
  Action:  Set genesis timestamp
           Compute new genesis hash
           All previous steps must be complete and stable
  Difficulty: hard (finalizes the chain, irreversible)
  ```

---

## Dependency Order

```yaml
Steps 1-5: independent, any order
Step 6:    must precede step 7
Step 8:    independent, anytime before step 9
Step 9:    must be last
```

---

## What Stays Identical to Bitcoin Core

```yaml
SHA-256 proof of work
UTXO model
Block time (10 minutes)
Block size (1MB)
Difficulty adjustment formula
P2P network protocol
Block structure
Wallet structure
Multisig
Lightning compatibility (HTLC attack mitigated by fixed fee economics)
```
