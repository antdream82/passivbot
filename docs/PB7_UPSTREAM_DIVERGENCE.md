# PB7 Upstream Divergence

## Purpose

This is the single reference for PB7 behavior that must be preserved on top of
`upstream/master`.

Use it when rebuilding PB7 from a fresh upstream checkout. The goal is not to
mirror old history, but to preserve the production behavior that still matters.
When upstream already has a cleaner contract, keep upstream and layer only the
local behavior that remains necessary.

## Rebuild Strategy

Start from `upstream/master`, then restore the following divergence packages:

1. Hyperliquid live/runtime behavior
2. Upstream scoring and metric semantics plus required local metric extensions
3. Optimize and limit semantics used in production
4. Runtime environment and downloader compatibility
5. Regression coverage

## Package 1: Hyperliquid Live And Runtime Behavior

### Files

- [src/exchanges/hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)
- [src/passivbot.py](/app/pb7/src/passivbot.py)
- [src/utils.py](/app/pb7/src/utils.py)

### Required behavior

1. Internal Hyperliquid asset ids must not leak into CCXT-facing symbol paths.
2. Reverse mapping from `baseId` and `@baseId` must recover valid CCXT symbols.
3. Pseudo-symbol fallback must not create invalid external symbols.
4. DEX-scoped balance, position, and equity reads must preserve current live
   semantics.
5. Open-order and state-fetch flows must normalize symbols consistently.
6. `cancel-gone` handling must remove ghost orders from local state when the
   exchange reports the order is already canceled or gone.
7. Recent suppress / tombstone handling must prevent stale order resurrection
   loops when Hyperliquid returns lagging open-order snapshots.
8. Snapped-balance behavior used for sizing and orchestration must be
   preserved.
9. Market metadata and margin-mode handling must preserve current local
   Hyperliquid behavior.
10. Spot namespace collision handling for stock perps / HIP-3 symbols must
    remain intact.

### Notes

- Immediate local removal on `cancel-gone` is the important part.
- Tombstone persistence across restart is operationally useful and should stay
  unless live evidence proves it unnecessary.

## Package 2: Metric Semantics And Analysis Output

### Files

- [src/config/metrics.py](/app/pb7/src/config/metrics.py)
- [src/config/scoring.py](/app/pb7/src/config/scoring.py)
- [src/config/limits.py](/app/pb7/src/config/limits.py)
- [src/backtest.py](/app/pb7/src/backtest.py)
- [src/optimize.py](/app/pb7/src/optimize.py)
- [src/pareto_store.py](/app/pb7/src/pareto_store.py)
- [src/tools/iterative_backtester.py](/app/pb7/src/tools/iterative_backtester.py)
- [passivbot-rust/src/analysis.rs](/app/pb7/passivbot-rust/src/analysis.rs)
- [passivbot-rust/src/types.rs](/app/pb7/passivbot-rust/src/types.rs)

### Required behavior

1. Optimizer scoring must follow upstream structured `metric + goal`
   semantics.
2. Engine-space objective direction handling must preserve upstream semantics.
3. Canonical metric naming must follow upstream rules.
4. Upstream metric families must remain available:
   - `*_pnl`
   - `paper_loss_*`
   - `exposure_*`
   - `win_rate*`
   - `trade_loss_*`
5. The local ulcer/UI family must remain available:
   - `ulcer_index`
   - `adg_over_ui`
   - `gain_over_ui`
6. PB7 analysis must emit realized side-exposure metrics:
   - `wallet_exposure_mean_long`
   - `wallet_exposure_median_long`
   - `wallet_exposure_max_long`
   - `wallet_exposure_mean_short`
   - `wallet_exposure_median_short`
   - `wallet_exposure_max_short`
7. PB7 analysis must emit realized-exposure-normalized metrics:
   - `gain_per_actual_exposure`
   - `adg_per_actual_exposure`
   - `adg_w_per_actual_exposure`
   - `mdg_per_actual_exposure`
   - `mdg_w_per_actual_exposure`
   - `gain_per_actual_exposure_long`
   - `gain_per_actual_exposure_short`
   - `adg_per_actual_exposure_long`
   - `adg_per_actual_exposure_short`
   - `adg_w_per_actual_exposure_long`
   - `adg_w_per_actual_exposure_short`
   - `mdg_per_actual_exposure_long`
   - `mdg_per_actual_exposure_short`
   - `mdg_w_per_actual_exposure_long`
   - `mdg_w_per_actual_exposure_short`
8. Short realized exposure must use `abs(twe_short)` because raw short TWE is
   signed negative.
9. Metric allow-lists and scoring/weight plumbing must include the local
   actual-exposure family.

### Notes

- Upstream structured scoring stays authoritative.
- Local metrics are added on top of that structure, not instead of it.

## Package 3: Optimize, Limits, And Seed Compatibility

### Files

- [src/backtest.py](/app/pb7/src/backtest.py)
- [src/config_utils.py](/app/pb7/src/config_utils.py)
- [src/limit_utils.py](/app/pb7/src/limit_utils.py)
- [src/optimize.py](/app/pb7/src/optimize.py)
- [src/pareto_store.py](/app/pb7/src/pareto_store.py)
- [src/tools/iterative_backtester.py](/app/pb7/src/tools/iterative_backtester.py)
- [src/optimization/config_adapter.py](/app/pb7/src/optimization/config_adapter.py)

### Required behavior

1. Optimize `limits` must support optional suite `scenario`.
2. When `scenario` is present, `stat` must be rejected.
3. Scenario-specific limit resolution must work both:
   - during optimize runtime
   - during pareto-store reevaluation
4. Zero-fill backtest resampling must remain safe.
5. No-trade backtests must not be misclassified as liquidation purely because
   Rust analysis defaults an analysis gap field.
6. Backtest analysis must accept both legacy 18-column fills and current
   19-column fills with `liquidity`.
7. Old starting-config seeds must remain usable:
   - ignore obsolete bounds that no longer map to bot params
   - map legacy alias bounds such as old `filter_*` keys where still sensible
   - never reuse stale seed `optimize.bounds` in a way that causes
     values/bounds length mismatches

### Notes

- Preserve upstream zero-fill safety while layering local metrics.
- Starting-config compatibility matters because PBGUI and historical optimize
  outputs are used as seeds in production.

### Critical regression: unstuck allowance must stay cumulative

This is one of the highest-risk behavior differences in the current rebase.

#### What changed upstream

Upstream commit `fa6947a7` introduced rolling / effective realized-PnL handling
for HSL and risk logic. That part is reasonable for recent-risk gating, but the
same effective/rolling PnL path was also applied to auto-unstuck allowance in:

- [passivbot-rust/src/backtest.rs](/app/pb7/passivbot-rust/src/backtest.rs)
- [src/passivbot.py](/app/pb7/src/passivbot.py)

The affected paths were:

- Rust backtest unstuck allowance used by direct backtest order logic
- Rust orchestrator input `unstuck_allowance_long/short`
- Live Python `_calc_unstuck_allowances()`
- Live Python `_calc_unstuck_allowance_for_logging()`

#### Why that is a regression

`unstuck` is position-management logic, not recent-risk gating.

Using a rolling lookback window for unstuck allowance makes the bot "forget"
older realized profits that historically acted as buffer for controlled
de-risking. In practice this reduces `close_unstuck_*` size, changes the next
entry path, and can turn a historically profitable config into an early
liquidation path.

This was reproduced on a historical one-coin config, so the issue is not
explained by forager selection or multi-coin ranking.

Reference config:

- [config.json](/app/pb7/backtests/pbgui/xyz_trailing_new8/suite_runs/2026-04-04T12_54_21/base/hyperliquid/2026-04-04T12_54_26/config.json)

Stored historical `base` result:

- `adg_usd = 0.0018088215516034456`
- `gain_usd = 149.29282756104152`
- `drawdown_worst_usd = 0.4244738513871256`
- `liquidated = None`

Current rebased engine before the fix produced:

- `adg_usd ~= -3.368e-05`
- `gain_usd ~= 0.9545`
- `drawdown_worst_usd ~= 0.9978`
- `liquidated = True`

The first concrete path divergence appeared very early in fills:

- old path: `2018-12-26 17:02:00 close_unstuck_long -0.9516`
- bad rebased path: `2018-12-26 17:02:00 close_unstuck_long -0.4112`

That smaller unstuck close immediately changed subsequent entry sizing and led
to a completely different position path.

#### Required behavior

Keep the semantics split:

- `HSL` / recent realized-loss gating:
  use rolling / lookback-aware realized PnL
- `unstuck allowance`:
  use full cumulative realized PnL history

In other words:

- keep `effective_pnl_cumsum()` for HSL-oriented logic
- do **not** use it for unstuck allowance
- add a separate cumulative helper for unstuck allowance if needed

#### Implementation rule

If upstream is rebased again, ensure the following remain true:

1. In Rust backtest, unstuck allowance uses cumulative
   `pnl_cumsum_max / pnl_cumsum_running`, not `effective_pnl_cumsum()`.
2. In Rust orchestrator input construction, `global.unstuck_allowance_*` also
   uses cumulative realized PnL.
3. In live Python, `_calc_unstuck_allowances()` uses full
   `self._pnls_manager.get_events()` history, not `_get_effective_pnl_events()`.
4. In live Python, `_calc_unstuck_allowance_for_logging()` uses full event
   history too, so the logged allowance matches the live unstuck semantics.

#### Validation

Minimum validation for this regression:

1. Rebuild Rust:
   `cd /app/pb7/passivbot-rust && VIRTUAL_ENV=/venv_pb7 PATH=/venv_pb7/bin:$PATH maturin develop --release`
2. Run:
   `cd /app/pb7 && /venv_pb7/bin/pytest tests/test_passivbot_balance_split.py -q`
3. Re-run the historical suite config above and confirm the `base` result
   returns to the historical behavior band:
   - `adg_usd` positive
   - `gain_usd` near `149.29`
   - `drawdown_worst_usd` near `0.424`
   - `liquidated = False`

## Package 4: Runtime Environment And Downloader Compatibility

### Files

- [requirements-live.txt](/app/pb7/requirements-live.txt)
- [src/procedures.py](/app/pb7/src/procedures.py)
- [src/downloader.py](/app/pb7/src/downloader.py)

### Required behavior

1. The rebase environment is standardized around current `ccxt`.
2. `requirements-live.txt` and the actual `/venv_pb7` must match.
3. `assert_correct_ccxt_version()` must pass in normal runtime.
4. `downloader.py` must not crash on Hyperliquid because of the old
   `ohlcvs`-path assumption.
5. Downloader compatibility is kept for correctness, even though Hyperliquid
   history limits still cap its usefulness for long-span stock-perp warmups.

### Notes

- Current rebase baseline uses `ccxt==4.5.47`.
- Any change under [passivbot-rust/src](/app/pb7/passivbot-rust/src) requires a Rust
  extension rebuild before validating Python-side behavior or backtest/optimize
  outputs.
- Rebuild command:
  `cd /app/pb7/passivbot-rust && VIRTUAL_ENV=/venv_pb7 PATH=/venv_pb7/bin:$PATH maturin develop --release`
- If this rebuild is skipped, Python may keep using an older installed
  `passivbot_rust` extension and newly added metrics can appear missing from
  backtest or optimize results even when the Rust source already contains them.

## Package 5: Regression Coverage

### Files

- [tests/test_utils_maps.py](/app/pb7/tests/test_utils_maps.py)
- [tests/test_stock_perps.py](/app/pb7/tests/test_stock_perps.py)
- [tests/test_hyperliquid_balance_cache.py](/app/pb7/tests/test_hyperliquid_balance_cache.py)
- [tests/test_passivbot_balance_split.py](/app/pb7/tests/test_passivbot_balance_split.py)
- [tests/test_order_orchestration.py](/app/pb7/tests/test_order_orchestration.py)
- [tests/test_backtest_analysis.py](/app/pb7/tests/test_backtest_analysis.py)
- [tests/test_optimizer_limits_integration.py](/app/pb7/tests/test_optimizer_limits_integration.py)
- [tests/test_pareto_limits.py](/app/pb7/tests/test_pareto_limits.py)
- [tests/test_config_utils_helpers.py](/app/pb7/tests/test_config_utils_helpers.py)
- [tests/optimization/test_config_adapter.py](/app/pb7/tests/optimization/test_config_adapter.py)
- [tests/optimization/test_optimize.py](/app/pb7/tests/optimization/test_optimize.py)
- [tests/test_ohlcvs_downloader.py](/app/pb7/tests/test_ohlcvs_downloader.py)

### Required behavior

1. Tests must be aligned to rebased APIs without losing local intent.
2. Hyperliquid symbol mapping and state-fetch behavior must stay covered.
3. Actual-exposure metrics and scenario-aware limits must stay covered.
4. Backtest zero-fill, no-trade, and legacy-fill edge cases must stay covered.
5. Starting-config compatibility for legacy seeds must stay covered.
6. Downloader compatibility for Hyperliquid must stay covered.

## Intentional Non-Reapplications

1. Old local structure that only mirrored pre-upstream refactors.
2. String-only scoring assumptions replaced by upstream structured scoring.
3. Workarounds whose only purpose was to compensate for older upstream bugs now
   fixed in current upstream.

## Validation Checklist

1. `python -m py_compile` passes for touched source and test files.
2. Rust extension is rebuilt against the rebased sources:
   `cd /app/pb7/passivbot-rust && VIRTUAL_ENV=/venv_pb7 PATH=/venv_pb7/bin:$PATH maturin develop --release`
3. Hyperliquid/runtime tests pass:
   - `test_utils_maps.py`
   - `test_stock_perps.py`
   - `test_hyperliquid_balance_cache.py`
   - `test_passivbot_balance_split.py`
   - `test_order_orchestration.py`
4. Optimize/backtest compatibility tests pass:
   - `test_backtest_analysis.py`
   - `test_optimizer_limits_integration.py`
   - `test_pareto_limits.py`
   - `test_config_utils_helpers.py`
   - `tests/optimization/test_config_adapter.py`
   - `tests/optimization/test_optimize.py`
   - `test_ohlcvs_downloader.py`
5. Actual-exposure metrics appear in real backtest analysis output.
6. Scenario-aware limits trigger in real optimize execution.
7. Historical seed configs load without obsolete-bound failures.
8. Hyperliquid live startup completes without symbol-mapping regressions.

## Status

This document is the current upstream-based reference for PB7 divergence.
