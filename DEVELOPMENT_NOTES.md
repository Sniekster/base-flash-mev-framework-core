# Development notes — FlashArb toolkit for Base

Last update: 2026-10-11

## Overview
- **Execution contract** (`src/FlashArb.sol`): Balancer V2 flash loan (fee 0) → 2–3 hops over Uniswap V3 / Aerodrome Slipstream /
  Aerodrome V2 → profit gate. 40 offline tests + 14 tests against live Base state.
  **The contract has not been externally audited.** It has only been run in tests and on local forks.
- **arbsim (Python)**: event-driven pool-state tracker (Swap/Mint/Burn/Sync), graph router, bit-exact AMM math, local EVM simulation
  on an anvil fork, behavioural token check, EVM confirmation of hits, and a shadow-mode watcher. It only reads and simulates:
  there is no signing code and nothing is ever broadcast. 77 pytest tests (live Base + offline).
- Usage: `python -m arbsim scan --start WETH,USDC --verify`, `python -m arbsim watch ...` (shadow), `python -m arbsim soak --blocks 90`.
- Not included, by design: mainnet deployment, wallet or key handling, real transactions.

## Development principles
- Every reported issue is first reproduced with a test, or rejected with a stated reason. Only demonstrated problems lead to code changes.
- A component counts as finished after two consecutive review rounds without a new demonstrable finding of medium severity or higher, with all tests passing.
- No profit claims. Competition from specialised MEV bots, gas and priority fees, and the thin Balancer liquidity on Base (~30 WETH, ~65k USDC)
  determine whether this can ever earn anything.

## Event tracker — results
- Pool events are replayed in (block, logIndex) order (V3/Slipstream: Swap, Mint, Burn with a tick book of net and gross liquidity;
  Aerodrome V2: Sync). Verified live: replayed state == on-chain state (reserves, slot0, liquidity, every tick).
- **Soak run on Base**: 90 blocks, 438 pool updates, 0 reorgs, 0 window reloads → `tracked state == chain state`
  (`python -m arbsim soak --blocks 90`, exit code 1 on any difference).
- Issues found and fixed (each with a regression test):
  1. Stable pools: on-chain Solidity 0.8 reverts (panic 0x11) on uint256 overflow in Aerodrome's `_get_y`; the Python model used bigints
     and returned a quote where the chain would revert (pool 0x3548…5f16, amount 228110859285604224). The model now refuses in the same
     places (checked multiplications); offline regression test + live parity.
  2. A crash halfway through `advance()` could apply Mint/Burn **twice** → a 'dirty' flag forces a full reload in the next round.
  3. The logs node can lag behind the state node (two providers) → the tracker only advances to the lowest common block.
  4. The Aerodrome V2 factory fee can change without a pool event → it is re-read every round (alongside the dynamic Slipstream fee).
  5. Logs from older blocks in a range were only checked against the head block hash → every log is now checked against its canonical block hash.
- Claims checked and rejected during review: tick-book "over-delete/duplicates" (the index holds one entry per tick; disproven with a random
  Mint/Burn property test, 5×1500 steps against a brute-force model), string sorting of log keys (the code casts to int), and that a block
  hash does not cover the full history (a block hash binds the whole chain).
- Limitations: dynamic Slipstream fees are read at block `head` while the transaction lands in the next block (`minProfit` with a margin);
  tick window ±6 words (outside it the pool is reloaded or the cycle skipped); public RPCs are rate-limited
  (use your own provider via `BASE_RPC_URL` / `BASE_LOGS_RPC_URL`).

## Flash-loan source depth — measured
- `python -m arbsim.studies depth|sensitivity`, borrowable at the time of measurement: Balancer 29.8 WETH / 66k USDC (fee 0), Morpho Blue
  82,359 WETH / 233.8M USDC (fee 0), Aave V3 17,091 WETH (fee 0.05%, address taken from the official address book and verified on-chain),
  deepest Uniswap V3 pool 36.6k WETH (fee = pool tier, 0.3%).
- The optimal trade size was ~0.0001 WETH / ~0.33 USDC, five orders of magnitude below even the Balancer cap. A 100× deeper source gives
  exactly the same best gross profit (5.9e-7 WETH); a fee (5/30 bps) only lowers it (5.4e-7 / 3.2e-7). Net > 0: **0 cycles in every setup**.
  The limit is the fixed cost per transaction (~2.5e-6 ETH) against a gross edge below 1e-6 ETH. A deeper source only matters for rare,
  large dislocations; if wanted, Morpho Blue (fee 0) is preferable to Uniswap flash.
- Shadow run `watch`, 90 blocks with pool changes (~11 min of chain time): 2,828 observations with gross > 0, **0 with net > 0**;
  best gross/cost 0.28 (WETH) and 0.41 (USDC). Best single opportunity: ~0.0025 USDC gross against ~0.006 USDC cost.
- **Bug found by this measurement** (fixed, with test): the screening probe scaled with the loan cap (hi/1e6). With a 100× deeper
  source the first probe skipped the profitable region and the router found nothing. Now a fixed probe ladder (×100 steps from a
  cap-independent lower bound).
- The contract was not changed: the data gives no reason to switch flash source, and adding a second flash provider is a
  security-sensitive change that needs its own review.

## Long-tail tokens — measured
- `python -m arbsim scan|watch|soak --voter-tokens 40` reads the Aerodrome **Voter** (live gauges) and adds the 40 most-used gauged
  tokens: 47 tokens, 850 pools, 118,038 cycles of ≤3 hops.
- **fBOMB "profit" was a model artefact**: 42 of 52 net-positive observations ran through fBOMB, which **burns 1% on every transfer**
  (measured in the local fork: the receiver gets 0.99×). The EVM simulation correctly rejected those cycles (revert selector
  `0xa932492f` = `K()`, the Aerodrome V2 invariant check). This is why the EVM is the judge and the model only a filter.
- **Behavioural token check** (`arbsim/tokencheck.py`): in the local fork the token's largest pool is impersonated and pool→wallet,
  wallet→wallet and wallet→pool transfers are made; a token is clean only if sender and receiver move by exactly the amount sent.
  Result: fBOMB out (99%), NVDAc out (cannot run in the fork: `OpcodeNotFound`), 45 clean, 826 pools kept. Verdicts are cached for 12 hours.
  Tests cover fake tokens with a tax on each path, reverts, insufficient balance, cache expiry, and live tokens; removing the
  wallet→pool step makes 2 tests fail.
- **EVM confirmation of hits** (`arbsim/confirm.py`, on by default in `watch`, off with `--no-verify-hits`): every net-positive hit is
  executed in a local fork with the real `FlashArb`; it only counts if the EVM succeeds and the profit equals the model to the wei
  (Slipstream routes with dynamic fee: within 2%). `watch --jsonl` records full pool addresses and the EVM verdict, so
  `python -m arbsim.replay` can repeat any observation later. A failed confirmation does not stop the loop; the hit is marked UNVERIFIED.
- **The remaining 10 observations** were all in one block (52422252) via AERO/USDC and reproduced in the EVM to the wei
  (e.g. `USDC>AERO>USDC`: borrow 15.4 USDC, gross 13,262 units = $0.013, net ~6,700 units = **~$0.007**). In the next block the gross
  margin fell to 826 (3-hop variants to 0), already below the ~6,555 units of cost, and ~0 after 2–3 blocks.
  Reproduce with `python -m arbsim.replay --jsonl review/longtail/hit_52422252.jsonl --block 52422252`.
- Tracker drift ruled out: `soak --voter-tokens 40` over 850 pools: 116 blocks, 615 pool updates, 0 reorgs → tracked state == chain state.
- Latest shadow run on the filtered universe (826 pools, 8 states over 183 blocks, blocks 52423404–52423587): 14,900 observations with
  gross > 0, **0 net-positive**; best gross/cost 0.50.
- Fresh scans (blocks 52422425 and 52423055): **0 cycles net > 0**; 16 cycles gross > 0, all simulate OK, 15 of 16 equal to the model
  to the wei (the 16th differs by 0.6%: dynamic Slipstream fee).
- Robustness: `soak` and `watch` share `with_backoff` (HTTP 429 and timeouts are retried with exponential back-off; safe because
  `Tracker.advance()` is crash-safe).
- Assessment: the long tail contains real mispricings, but those found are worth less than a cent and disappear within seconds.
  It is a race for ordering within the block; each attempt costs ~$0.006 whether or not it wins.

## Router speed-up
- Profiling showed the time per `watch` step (~1 minute over 850 pools) was in the router, not the tracker or RPC (~1–2 s):
  ~35,000 cycles per step, each needing hundreds of exact integer quotes.
- Changes (core math unchanged): (1) the V3 swap derives ticks lazily and memoises `sqrt_ratio_at_tick`; (2) a float pre-screen on the
  product of marginal rates (the profit curve is concave with f(0)=0, so product ≤ 1 + flash fee means never profitable), with a rigorous
  rounding-noise bound and cost awareness (`min_gross`); undecidable pools are always computed in full; (3) golden-section search instead of
  integer ternary search (~70 instead of ~270 quotes per cycle) plus early exit in the screen.
- A float-price-per-pool design (fixed loan size, no ticks, sizing or costs) would be faster still, but brings back the false positives the
  exact math removes (e.g. fee-on-transfer tokens), so it is deliberately not used.
- **Result**: offline on a frozen snapshot of 117,000 cycles **160 s → 18.4 s**; live `watch --voter-tokens 40` (826 pools, 116,792 cycles):
  **3.6–7.2 s per step** (tracker ~0.9 s, router 2.8–5.5 s over 17–35k cycles), covering **2–4 blocks per step instead of 20–35**.
- Caveats: a step is still longer than the 2 s block time (all events are replayed, so no state is lost, but it is evaluated at the end of
  each window). On cycles where coarse token units make the profit curve noisy (cbBTC: 1 raw unit ≈ 3e11 wei), old and new search land on
  different noise peaks; difference at most 1.9e11 wei (~$0.0006 against $0.006 cost), and **neither version finds a net-positive cycle**.
  Speed improves the completeness of the measurement, not the profit.
- On free public endpoints, 3 of 10 steps in one run took 42–56 s due to HTTP 429 on `eth_getLogs` plus back-off; router time stayed at
  1.9–6.0 s. Use your own node or a paid provider (`--rpc`, `--logs-rpc`) for steady speed.
- 17 tests added: lazy tick == original loop (8 seeds), marginal rate is an upper bound (V2/V3/stable), the pre-screen never discards a
  cycle that can reach `min_gross`, golden-section == ternary on smooth cycles, quote count, and the CLI step-time output.

## Contract review rounds (newest first)
### Round 3 (v3) — no demonstrable findings
- Claimed: `_hash` does not update the free-memory pointer → rejected (the assembly is `memory-safe` and the memory is not used after the hash).
- Claimed: missing bounds checks on calldata helpers → rejected: `execute` requires `p.length == 53 + n*41` exactly, and
  `receiveFlashLoan` only accepts the identical payload (hash binding), so every read position is within bounds.
- Hardening (not a finding): every hop must sell one of the two tokens of that pool (`BadPath`). 2 new tests; removing the check makes both fail.
### Round 2 (v2) — no demonstrable findings
- Claimed: Aerodrome V2 lacks `getAmountOut` → incorrect, disproven with a live fork test (volatile + stable pool, both directions,
  `getAmountOut` == actual swap output).
- Claimed: issues with `_transfer` scratch space, `data[:20]` as payment token, `_hash` free pointer, dirty bits → all rejected
  (scratch space 0x00–0x3f is reserved for assembly; `data[:20]` is built by the contract and equals the validated `held` token;
  `shr` leaves no dirty bits).
### Round 1 (v1) — 1 critical, fixed
- A compromised operator key could, via self-made (factory-genuine) pools, spend the **idle balances of other tokens**: dust loan of
  token A, then repay the raw amount in token B. Reproduced on a live Base fork (test balance 5000 USDC → 0 on v1).
- Fix: the registry stores `token0/token1`; `_route` enforces path continuity start → … → start. Removing any of the path checks makes
  at least one test fail.

## Tests
- Python (`python -m pytest arbsim/tests -q`): 77 passing (live Base + offline).
- Solidity: 40 offline (mocks) passing; 14 live Base fork tests passing
  (`BASE_RPC_URL=https://mainnet.base.org forge test --match-path test/FlashArb.fork.t.sol`).
- Test coverage has not been measured.
- Gas for `execute()`: roughly 258k–392k for 2–3 hops on the majors; long tail with tick crossings up to ~613k (excluding L1 data costs).

## Simulation and routing — results
- The math models are **bit-exact** against real pools on Base: Aerodrome V2 volatile + stable (`getAmountOut`), Uniswap V3 and
  Slipstream (real `swap` via a revert-in-callback quoter). Slipstream has a **dynamic fee** that differs per block context; it is read
  in the same context as the simulation.
- End-to-end on a local fork: an artificial price gap is found and sized by the router, and `FlashArb.execute` returns exactly the
  locally computed profit (to the wei); the reverse direction reverts atomically; `minProfit` is enforced.
- Fixed during review: (a) screening at too small a dust size could discard a real cycle due to rounding noise → minimum dust and
  multiple probes; (b) the L1 data fee was priced on the raw payload instead of the actual `execute(bytes)` calldata → corrected, with test.
- Rejected during review: ternary search not being unimodal (the output is concave; rounding noise is wei-scale), anvil not pinned (it is),
  tick-crossing gas in the optimizer (±$0.0004 on Base).

## Profitability — honest summary
- On the unmodified Base market, neither `scan` nor the shadow watcher finds **any** cycle on the major pools that is net profitable
  after gas and L1 data fee. Gross profits are on the order of 1e-7 ETH against costs on the order of 1e-6 ETH. Specialised bots close
  these gaps within the same block, often with private order flow and priority fees that are not modelled here.
- Long tail: larger gross edges (gross/cost up to 0.5), but the apparent fBOMB profit was a 1% burn, and the one real, EVM-reproduced
  net-positive opportunity (AERO/USDC, ~$0.007) was below cost one block later. Sample: 1 opportunity in 30 observed states,
  with a failed attempt costing ~$0.006.
- What the code is: a tested foundation (contract, exact pool math, event tracker, token check, EVM confirmation) to recognise
  opportunities and execute them atomically without principal risk (a revert costs only gas). What it is not: a guarantee of profit.
- Not modelled or built: evaluation fast enough for every block (real block coverage needs a different language or process model),
  a dedicated node or low-latency sequencer access, a priority-fee strategy, and the fact that competing bots see the same opportunities.
- Before using real funds: an external audit of `FlashArb.sol`, a long shadow run on your own RPC provider, and your own decisions on
  deployment, key management and amounts.
