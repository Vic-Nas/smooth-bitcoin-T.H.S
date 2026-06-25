# Fair Bitcoin B.T (FBTC)

## Parameters (user set at fork)

| Parameter | Description | Default |
| --- | --- | --- |
| `S` | Total supply in millions | `1000` |
| `T` | Tail emission in basis points of initial block reward | `0` |

## Derived at Startup (cached, never recomputed)

```toml
blocks_per_halving = 210,000
initial_reward     = (S × 10⁶) / 2 / blocks_per_halving
                   ≈ S × 1.488 coins/block
tail_reward        = initial_reward × (T / 10,000)
fee_rate           = initial_reward / max_block_bytes
```

> At S = 21, T = 0: initial reward ~50 FBTC/block, hard cap, no tail.
> At S = 21, T = 50: tail kicks in at 0.25 FBTC/block (0.5% of initial reward).
> At S = 100, T = 10: initial reward ~148.8 FBTC/block, tail at 0.149 FBTC/block.
> The halving shape is identical to Bitcoin.

## Emission Formula

```toml
reward(block_height) = max(
  initial_reward × (1/2)^floor(block_height / 210,000),
  tail_reward
)

fee_rate(block_height) = reward(block_height) / max_block_bytes
```

Same halving rhythm as Bitcoin. Tail floor kicks in when halving reward drops below `tail_reward`. Fee rate is always pegged to current block reward — as reward stabilizes at tail, so does the fee rate.

---

## Implementation Steps

- [ ] **Step 1 — Identity**

  ```yaml
  File:    chainparams.cpp
  Change:  Address prefix bc1 → fbc1
  Touches: bech32 prefix string
  Difficulty: trivial
  ```

- [ ] **Step 2 — Supply & Emission**

  ```yaml
  Change:  MAX_MONEY from 21,000,000 to S × 10⁶
           Scale initial block reward to S × 1.488 coins/block
           Add tail floor: reward never drops below tail_reward
           Peg fee_rate to current block reward / max_block_bytes
           Cache initial_reward, tail_reward, fee_rate at startup
  Difficulty: trivial
  ```

- [ ] **Step 3 — Fixed Fee Rate**

  ```yaml
  Change:  Remove fee market and mempool fee estimation
           Enforce fixed rate per byte
           100% of fees go to miner (lock it)
  Difficulty: low
  ```

- [ ] **Step 4 — Round-Robin Block Construction**

  ```yaml
  Change:  Remove fee priority ordering
           Miners must fill blocks round-robin by sender address:
             - at most 1 transaction per sender address per block
             - before any address gets a second transaction included
             - FIFO preserved within each address across blocks
  Validation: nodes scan block and reject if any sender address
              appears twice before all waiting addresses appear once
              O(n) local check, no global queue state required
  Tradeoff:  single-address high-volume senders wait ~1 block (~10 min)
             per transaction; multiple sending addresses unaffected
  Difficulty: low-medium
  ```

- [ ] **Step 5 — Genesis Block**

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
Steps 1-4: independent, any order
Step 5:    must be last
```

---

## What This Changes vs Bitcoin

```yaml
Address prefix:     bc1  → fbc1
Total supply:       hardcoded 21M → parameter S (millions)
Tail emission:      none → parameter T (basis points of initial reward, default 0)
Fee market:         removed, fee rate pegged to current block reward / max_block_bytes
Mempool ordering:   fee priority → round-robin by sender address
Miner enforcement:  blocks rejected if any sender address appears twice
                    before all waiting addresses appear once (O(n) check)
```

## What Stays Identical to Bitcoin Core

```yaml
SHA-256 proof of work
Halving schedule (every 210,000 blocks)
Emission shape (geometric halving series, floor at tail_reward if T > 0)
UTXO model
Block time (10 minutes)
Block size (1 MB)
Difficulty adjustment formula
P2P network protocol
Block structure
Wallet structure
Multisig
Lightning compatibility (HTLC attack mitigated by fixed fee economics)
```

---

## Design Thesis

> You can't buy your way to the front of the line. You can't be silenced either.

Bitcoin's fee market means the highest bidder gets priority — a structural advantage for the wealthy and for bots. A miner can also quietly exclude any address they choose, invisibly, with no rule broken.

FBTC replaces this with round-robin block construction: each address gets at most one transaction per block before anyone gets a second. Miners cannot exclude an address without leaving block space empty — which is economically self-punishing. Validation is a simple local check; no global queue state required.

The fee rate is not a market — it is pegged to the current block reward, falling predictably with each halving. If a tail emission (`T > 0`) is set, the fee rate stabilizes at a permanent floor when the halving reward drops below it, giving miners a sustainable baseline income forever.

The unit size (`S`) is a conscious choice rather than a historical accident, so users can hold and think in whole coins. Everything else — the proof of work, the halving rhythm, the UTXO model — stays exactly as Bitcoin users know it.