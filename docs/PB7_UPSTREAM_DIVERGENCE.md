# PB7 Upstream Divergence

## Purpose

This is the single source of truth for PB7 behavior that must be preserved on
top of `upstream/master`.

Use this document when rebuilding PB7 from a fresh upstream checkout.
It records only divergence that still matters in production. If upstream now
has a better contract for a behavior, keep upstream and do not force the old
local implementation back in.

## Base Strategy

Rebuild from `upstream/master`, then restore the required local behavior in
small packages:

1. Hyperliquid live/runtime behavior
2. Upstream scoring/metrics semantics
3. Local optimize extensions
4. Regression tests aligned to the rebased codebase

The rebased PB7 should prefer upstream semantics where they are cleaner, then
layer local-only behavior on top.

## Package 1: Hyperliquid Live / Runtime Behavior

### Files

- [src/exchanges/hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)
- [src/passivbot.py](/app/pb7/src/passivbot.py)
- [src/utils.py](/app/pb7/src/utils.py)

### Required behavior

1. Internal Hyperliquid asset ids must not leak into CCXT-facing symbol paths.
2. Reverse mapping from `baseId` or `@baseId` must restore valid CCXT symbols.
3. Pseudo-symbol fallback must not create invalid external symbols.
4. DEX-scoped balance, position, and equity reads must preserve current live
   semantics.
5. Open-order and state fetch flows must normalize symbols consistently.
6. `cancel-gone` handling must remove ghost orders from local state when the
   exchange reports the order is already canceled or gone.
7. Recent suppress / tombstone handling must prevent stale order resurrection
   loops when Hyperliquid returns lagging open-order snapshots.
8. Snapped-balance behavior used for sizing/orchestration must be preserved.
9. Market metadata and margin-mode handling must preserve current local
   Hyperliquid behavior.
10. Spot namespace collision handling for stock perps / HIP-3 symbols must
    remain intact.

### Notes

- `cancel-gone` immediate local removal is the important part.
- tombstone persistence across restart is operationally useful and should be
  kept unless live evidence proves it unnecessary.

## Package 2: Upstream Scoring And Metrics Semantics

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

1. Optimizer scoring follows upstream `metric + goal` semantics.
2. Engine-space objective handling preserves upstream direction semantics.
3. Canonical metric naming follows upstream rules.
4. Upstream metric families remain available:
   - `*_pnl`
   - `paper_loss_*`
   - `exposure_*`
   - `win_rate*`
   - `trade_loss_*`
5. `ulcer_index`, `adg_over_ui`, and `gain_over_ui` remain available.

### Notes

- Keep upstream structured scoring support.
- Do not regress to old string-only scoring assumptions.

## Package 3: Local Optimize Extensions

### Files

- [src/backtest.py](/app/pb7/src/backtest.py)
- [src/config_utils.py](/app/pb7/src/config_utils.py)
- [src/limit_utils.py](/app/pb7/src/limit_utils.py)
- [src/optimize.py](/app/pb7/src/optimize.py)
- [src/pareto_store.py](/app/pb7/src/pareto_store.py)
- [src/tools/iterative_backtester.py](/app/pb7/src/tools/iterative_backtester.py)
- [passivbot-rust/src/analysis.rs](/app/pb7/passivbot-rust/src/analysis.rs)
- [passivbot-rust/src/types.rs](/app/pb7/passivbot-rust/src/types.rs)

### Required behavior

1. PB7 analysis must emit realized side exposure metrics:
   - `wallet_exposure_mean_long`
   - `wallet_exposure_median_long`
   - `wallet_exposure_max_long`
   - `wallet_exposure_mean_short`
   - `wallet_exposure_median_short`
   - `wallet_exposure_max_short`
2. PB7 analysis must emit realized-exposure-normalized metrics:
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
3. Short realized exposure must use `abs(twe_short)` because raw short TWE is
   signed negative.
4. Metric allow-lists and objective weight maps must include these local actual
   exposure metrics.
5. Optimize `limits` must support optional `scenario` for suite runs.
6. When `scenario` is present, `stat` must be rejected.
7. Scenario-specific limit resolution must work:
   - during optimize runtime
   - during pareto-store reevaluation
8. Zero-fill backtest resampling must remain safe.
9. No-trade backtests must not be misclassified as liquidation purely because
   Rust analysis defaulted `equity_balance_diff_neg_max` to `1.0`.

### Notes

- Zero-fill safety now exists upstream; preserve it while adding local metrics.
- Local optimize semantics should be reintroduced without breaking upstream
  scoring structure.

## Package 4: Runtime Tooling / Environment

### Files

- [requirements-live.txt](/app/pb7/requirements-live.txt)
- [src/procedures.py](/app/pb7/src/procedures.py)

### Required behavior

1. The rebase environment is standardized around current `ccxt`.
2. `requirements-live.txt` and the actual `/venv_pb7` must match.
3. `assert_correct_ccxt_version()` must pass in normal runtime.

### Notes

- Current rebase baseline uses `ccxt==4.5.47`.

## Package 5: Tests

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

### Required behavior

1. Tests must be aligned to rebased APIs without losing local intent.
2. Hyperliquid symbol mapping/state fetch behavior must stay covered.
3. Actual exposure metrics and scenario-aware limits must stay covered.
4. Backtest zero-fill and no-trade edge cases must stay covered.

## Known Divergence That Is Intentionally Not Reapplied

1. Old local structure that only mirrored pre-upstream refactors.
2. String-only scoring assumptions replaced by upstream structured scoring.
3. Any workaround whose only purpose was to compensate for older upstream bugs
   now fixed in current upstream.

## Validation Checklist

After rebuilding PB7 from upstream, verify all of the following:

1. `python -m py_compile` passes for touched source and test files.
2. Rust extension is rebuilt against the rebased sources.
3. Hyperliquid/runtime tests pass:
   - `test_utils_maps.py`
   - `test_stock_perps.py`
   - `test_hyperliquid_balance_cache.py`
   - `test_passivbot_balance_split.py`
   - `test_order_orchestration.py`
4. Optimize/backtest tests pass:
   - `test_backtest_analysis.py`
   - `test_optimizer_limits_integration.py`
   - `test_pareto_limits.py`
   - `test_config_utils_helpers.py`
5. Actual-exposure metrics appear in real backtest analysis output.
6. Scenario-aware limits trigger in real optimize execution.
7. Hyperliquid live startup completes without symbol-mapping regressions.

## Current Status On This Worktree

This worktree is the current upstream-based PB7 reference implementation.

Already restored here:

1. Core Hyperliquid symbol mapping and state fetch behavior
2. Current `ccxt` alignment in the rebase environment
3. Upstream scoring semantics with local optimize extensions
4. `cancel-gone` handling and ulcer-related metrics
5. Regression coverage for the tested paths

This file should be kept up to date whenever upstream-based PB7 divergence
changes.
