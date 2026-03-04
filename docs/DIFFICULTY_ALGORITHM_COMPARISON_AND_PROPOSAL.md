# Difficulty Algorithm: Comparison and Proposal

## 1. Executive summary

BitMonero uses the same difficulty algorithm as Monero: the classic Cryptonote window-based formula with a **720-block window** and **no weighting**. When a large share of network hashrate suddenly leaves (e.g. pool shutdown, miner exodus), difficulty adjusts **very slowly** (on the order of many hours to a day). This document compares that behaviour with **Arqma’s LWMA-based** approach, then **proposes a change** (new branch `diff`) to allow faster adjustment while keeping consensus and security in mind.

---

## 2. Current behaviour: BitMonero and Monero

**Sources:** `src/cryptonote_basic/difficulty.cpp`, `src/cryptonote_config.h` (same constants as Monero).

**Algorithm:**

- Uses the last **720** blocks (with lag: 735 entries).
- Sorts by timestamp and **cuts 60** from each end to reduce outlier impact.
- Computes `time_span` and `total_work` over the **middle ~600 blocks**.
- Formula: `next_difficulty = (total_work * TARGET) / time_span` — **equal weighting** for all blocks in the window.

**Parameters:**

| Constant              | Value   | Meaning                          |
|-----------------------|---------|----------------------------------|
| DIFFICULTY_WINDOW     | 720     | blocks in window                 |
| DIFFICULTY_CUT       | 60      | trimmed from each end after sort |
| Effective window      | ~600 bl | ~20 h at TARGET = 120 s          |
| DIFFICULTY_TARGET_V2  | 120 s   | target block time                |

**Response to a sudden large drop in hashrate (e.g. −50%):**

- Block time increases (e.g. from 120 s to ~240 s).
- Difficulty is an average over **~600 blocks** with no extra weight on recent blocks.
- For difficulty to fall by ~50%, the window must be mostly filled with “slow” blocks — i.e. **hundreds** of new blocks at the new pace.
- In practice: **many hours (up to ~24 h)** before difficulty approaches the new level; blocks continue to appear at ~240 s or more instead of 120 s.

**Summary:** Very slow reaction; high stability against timestamp manipulation and short-lived hashrate spikes.

---

## 3. Arqma: LWMA and variants (fast reaction)

**Sources:** Arqma repo `src/cryptonote_basic/difficulty.cpp` (classic `next_difficulty` plus `next_difficulty_lwma`, LWMA-3, LWMA-4, **next_difficulty_v16**), `src/cryptonote_config.h`.

**Algorithms in Arqma:**

1. **next_difficulty()** — same classic Cryptonote as Monero/BitMonero (720 blocks, cut 60), used on older forks.
2. **next_difficulty_lwma()** — Zawy-style LWMA; fork-dependent N (e.g. 17 or 30) and T (120 or 240 s).
3. **next_difficulty_lwma_3()** — N = 90, T = 120 s.
4. **next_difficulty_lwma_4()** — N = 90, T = 120 s, with tempering of long solvetimes.
5. **next_difficulty_v16()** — current fork (v16): **N = 90**, **T = 120 s**, with a 3×T cap on drop, 10% jump rule, and rounding.

**LWMA idea:** Recent blocks have **higher linear weight** (block index `i` has weight `i`). Long solvetimes in the last blocks quickly increase LWMA(solvetimes), so **next_difficulty** drops soon after hashrate drops.

**Parameters (Arqma v16):**

| Constant           | Value | Meaning                    |
|--------------------|-------|----------------------------|
| DIFFICULTY_WINDOW_V16 | 90  | N blocks in LWMA           |
| DIFFICULTY_TARGET_V16 | 120 s | target block time        |
| FTL (future time limit) | 360 s | limits timestamp abuse |

**Response to a sudden large drop in hashrate (e.g. −50%):**

- Block time increases (e.g. to ~240 s).
- Because of weighting, the **first dozen to a few tens** of slow blocks already lower the computed difficulty.
- **Meaningful correction in ~1–3 h**, full adjustment in **~4–6 h** (order of N × new block time).
- Caps (e.g. 3×T, 10% jump) moderate extreme drops but reaction remains **much faster** than Monero/BitMonero.

**Summary:** Fast reaction; with N = 90 and T = 120 s the network returns to normal block pace in hours instead of a day.

---

## 4. Side-by-side comparison

| Aspect                    | BitMonero / Monero           | Arqma (LWMA v16)              |
|---------------------------|-----------------------------|-------------------------------|
| Algorithm                 | Cryptonote (sort + cut, no weights) | LWMA (linear weights)   |
| Window                    | 720 bl (~600 effective)     | 90 bl                        |
| Target block time         | 120 s                       | 120 s                        |
| Response to large hashrate drop | **Slow** — many hours to ~1 day | **Fast** — 1–6 h to clear adjustment |
| Stability vs timestamp abuse | High (long window, cut)   | Requires tight FTL (e.g. 360 s) |

---

## 5. Proposal (branch `diff`)

**Goal:** Improve difficulty reaction when a large amount of hashrate suddenly leaves the network, while preserving consensus and security.

**Options (choose one or combine):**

### Option A — Reduce window (minimal change)

- Keep the current formula; only change constants in `cryptonote_config.h`:
  - e.g. **DIFFICULTY_WINDOW** from 720 to **120** (or 90),
  - **DIFFICULTY_CUT** from 60 to **20** (or 15), so the effective window stays reasonable.
- **Pros:** Single-file change, no new code paths, faster reaction (e.g. ~2–4 h instead of ~24 h).
- **Cons:** More sensitive to timestamp manipulation and short hashrate spikes; requires a **hardfork** (new consensus rules).

### Option B — Add LWMA behind a hardfork

- Introduce an LWMA implementation (e.g. inspired by Arqma’s `next_difficulty_v16` or Zawy’s reference) in `src/cryptonote_basic/difficulty.cpp`.
- Add constants (e.g. `DIFFICULTY_WINDOW_LWMA = 90`, `DIFFICULTY_TARGET_V2` unchanged, and a reduced **CRYPTONOTE_BLOCK_FUTURE_TIME_LIMIT** for the LWMA fork, e.g. 300–360 s).
- In `blockchain.cpp`, call the new LWMA function for blocks at or above a chosen **hardfork height**; keep current algorithm below that height.
- **Pros:** Fast, smooth adjustment; proven in other coins (Arqma, others).
- **Cons:** More code, needs thorough testing and a coordinated hardfork; FTL must be tightened to avoid timestamp abuse.

### Option C — Hybrid (recommended direction)

- **Phase 1 (Option A):** Reduce window (e.g. to 120 blocks) and cut (e.g. 20) at a hardfork, with optional tightening of FTL.
- **Phase 2 (optional):** In a later hardfork, switch to LWMA (Option B) with N = 90 and the same target, for even faster and smoother reaction.

**Suggested next steps on branch `diff`:**

1. **Documentation (this file):** Keep this comparison and proposal in the repo (done).
2. **Implement Option A** in a follow-up commit: change `DIFFICULTY_WINDOW` and `DIFFICULTY_CUT` in `src/cryptonote_config.h` and document the new values and hardfork plan; or
3. **Implement Option B** (or a simplified LWMA) in `difficulty.cpp` + `blockchain.cpp` with a new constant set and hardfork gate.

**References:**

- Monero: `github.com/monero-project/monero` (same config and difficulty as BitMonero).
- Arqma: `github.com/arqma/arqma` (`difficulty.cpp`, `cryptonote_config.h` — LWMA, v16, FTL).
- Zawy LWMA: `github.com/zawy12/difficulty-algorithms`.
