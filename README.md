# base-flash-mev-framework-core
L2 MEV framework &amp; AMM simulation engine for Base. Features bit-exact math, local EVM simulation forks, and 131 tests passing.

# An Arbitrage Research Toolkit for Base: Exact Math, Checked in a Local EVM

A production-grade, ultra-low latency triangular arbitrage infrastructure framework optimized for Layer-2 FIFO execution dynamics. Fully verified on Base network structures.

📊 **PRODUCTION CODE & COMPILED MODULES AVAILABLE AT:**  
👉 [**Get the Full Production Framework Core on Whop (\$499)**](https://whop.com/evm-core-labs/arbitrage-research-toolkit-for-base-exact-math-checked-in-a-local-evm/)

*Every statement below is backed by the repository's internal test suites or live shadow-mode recordings. This is a high-performance quantitative research toolkit; no trading profits are promised.*

---

## 🔬 What it does
- **Exact AMM math.** Bit-exact Uniswap V3 / Slipstream tick math and Aerodrome V2 (volatile and stable) quotes, compared against the chain in tests.
- **Block-by-block state.** Pool state is kept current by replaying events, with reorg handling.
- **Cycle search and sizing.** Enumerates 2–3 hop cycles through your start token and sizes the loan on the exact math.
- **EVM confirmation.** Candidates are executed against a local anvil fork of Base. The model profit and the EVM profit match to the last unit.
- **Token safety filter.** Detects fee-on-transfer and untestable tokens (the recorded run drops two of 47 tokens).
- **Shadow mode.** `watch` never sends a transaction.
- **Execution contract.** `FlashArb.sol` takes a Balancer V2 flash loan (fee 0) and reverts the whole transaction if the trade does not clear the profit gate.

## 📈 Numbers you can quote (100% Passing)
- 77 Python tests and 54 Solidity tests (40 offline, 14 against live Base state): all passing in the recorded run.
- Shadow run on 826 pools / about 116,792 cycles: 3–7 s per step on a quiet RPC endpoint, covering 2–4 Base blocks of 2 s each.
- Router speed-up from the exact-math work: 117k cycles in 18 s instead of 160 s (offline benchmark).

## ⚠️ Read this before you buy
- **No profit is promised.** In the recorded shadow run the best net result per step is negative (about -\$0.006 to -\$0.007 after estimated gas and L1 data cost). Opportunities that clear costs are rare, worth cents, and usually gone within one block.
- **Not a competitive MEV bot.** A step takes longer than a block. Searchers with private order flow and dedicated infrastructure act within one block.
- **Unaudited.** `FlashArb.sol` has no external audit and has only been run in tests and on local forks. Do not put funds behind it without your own review.
- **You carry the costs.** A reverted transaction still pays gas.
- **RPC matters.** Free public endpoints rate-limit (HTTP 429); the recorded run shows 3 of 10 steps slowed to 42–56 s. Use your own node or a paid provider for steady speed.
- **Test coverage was not measured.** 131 tests is a count, not a coverage figure.
- Not financial advice.

---

## 📦 How to Purchase the Source Code
The core algorithmic routers (Python) and gas-optimized execution binaries (Solidity) are hosted securely behind our file-delivery gate. 

To download the repository instantly via automated credit card or crypto checkout:  
🚀 [**Click Here to Purchase on Whop (\$499)**](https://whop.com/evm-core-labs/arbitrage-research-toolkit-for-base-exact-math-checked-in-a-local-evm/)
