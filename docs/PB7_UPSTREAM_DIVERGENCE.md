# PB7 Upstream Divergence

This is the rebase checklist for rebuilding local PB7 behavior on top of
`upstream/master`. Prefer upstream contracts unless a local patch below is
explicitly required for production behavior.

## Rebase Rule

Start from upstream, then reapply only the packages below. After any Rust source
change, rebuild the Python extension with the stamp-aware `rust_utils`
workflow or `maturin develop --release` plus stamp validation.

## Optimizer Metrics And Seeds

Local optimizer behavior extends upstream structured `metric + goal` scoring.
Keep upstream metric/goal semantics authoritative, then add local metrics:

- `ulcer_index`
- `adg_over_ui`
- `gain_over_ui`
- realized side exposure metrics such as `wallet_exposure_mean_long` and
  `wallet_exposure_mean_short`
- realized-exposure-normalized metrics such as
  `gain_per_actual_exposure_long` and `gain_per_actual_exposure_short`

Seed compatibility is production-critical because historical PBGUI optimize
results are reused as starting configs. Old seed configs must:

- ignore obsolete filter volatility drop bounds
- map legacy filter volume aliases to current forager volume bounds
- seed bot/live values without allowing stale seed `optimize.bounds` to replace
  the active run's search space
- accept legacy Rust fill payloads without the `liquidity` column
- keep `metrics_only` analysis export safe when fills are omitted

## Optimize Limits

Local optimize limits support suite scenario-specific checks. A limit may target
a suite `scenario`; when it does, ambiguous `stat` usage must be rejected.
Runtime optimize and pareto-store reevaluation must resolve these limits the
same way.

The original direction semantics are preserved:

- `penalize_if_greater_than` checks the maximum value across the selected scope.
- `penalize_if_lower_than` checks the minimum value across the selected scope.

## Unstuck Allowance PnL History

Upstream rolling/effective PnL lookback remains useful for recent-risk gates
such as HSL and realized-loss checks. Local production behavior keeps unstuck
allowance tied to full realized PnL history instead.

Required split:

- HSL and realized-loss gates use the configured rolling/effective PnL window.
- Backtest unstuck allowance uses full `pnl_cumsum_max` and
  `pnl_cumsum_running`.
- Live unstuck allowance uses all events from `FillEventsManager`, not
  `_get_effective_pnl_events()`.
- Orchestrator `realized_pnl_cumsum_*` fields may still report effective PnL
  stats for recent-risk gates; only the unstuck allowance calculation is
  full-history.

This patch is intentionally applied to both Rust backtest and Python live paths
so live/backtest position-management semantics stay aligned.

## Hyperliquid Runtime

Keep local Hyperliquid operational hardening unless upstream proves an equal or
better contract:

- external CCXT symbols must not be replaced by internal Hyperliquid asset IDs
- `cancel-gone` orders are removed locally and tombstoned to prevent stale
  resurrection loops
- unified-account payloads may need split core/HIP-3 position fetches
- HIP-3/non-standard perps must fail loudly when required cross-mode state is
  unavailable
- snapped balance is used for sizing/orchestration stability while raw balance
  remains authoritative for peak-sensitive risk checks

## Side-Specific Universes

Side-specific `live.approved_coins` and `live.ignored_coins` must remain
side-specific through suite, optimizer, backtest, and live entry eligibility.
Dataset preparation may use a union of coins, but candidate evaluation must not
widen one side's tradable universe with the other side's coins.

## Verification

For each rebase package:

- run targeted Python tests first
- rebuild Rust immediately after Rust edits
- run at least one optimizer/backtest smoke before pushing
- update this document when preserving, dropping, or replacing a local patch
