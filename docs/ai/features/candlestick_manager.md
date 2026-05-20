# Candlestick Manager

## Contract

1. Return contiguous minute-series needed for indicator calculations.
2. Prefer cached data, then fetch missing ranges.
3. Synthesize zero-candles for verified no-trade gaps where required.

## Non-Obvious Details

1. Runtime synthetic candles are not always persisted to disk shards.
2. Real candles replacing synthetic candles must trigger EMA cache invalidation.
3. Gap semantics differ within-page vs between-page boundaries.
4. Full-day archive/TradFi misses are tracked separately from persistent CCXT
   gaps. This suppresses repeated archive retries without falsely declaring the
   exchange OHLCV path permanently unavailable.

## Failure Modes To Watch

1. Cache path mismatch by exchange naming.
2. Pagination edge behavior causing boundary gaps.
3. Persistent lock or stale data artifacts.

## Test Focus

1. Gap fill behavior and continuity.
2. Replacement/invalidation behavior when real data arrives.
3. Pagination boundary correctness per exchange.
4. Archive-unavailable days are retried once, then skipped on later prefetches.

## Key Code

- `src/candlestick_manager.py`
- `src/tools/verify_hlcvs_data.py`
- `docs/ai/exchange_api_quirks.md`
