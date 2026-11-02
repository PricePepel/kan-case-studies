# Polymarket Tracker

**Year:** 2026
**Status:** Personal research · Paper-traded only · Scanning paused since October 28, 2026
**Role:** Solo
**Repo:** [github.com/PricePepel/polymarket-tracker](https://github.com/PricePepel/polymarket-tracker)

---

## What it is

A scanner plus a paper-trading signal layer for Polymarket's BTC up-or-down 5-minute markets. The scanner logs what the top 50 CRYPTO accounts trade. The signal layer reads their late net flow and issues a paper ticket shortly before each market closes. It never placed a real order.

## The question

Do the wallets at the top of the CRYPTO leaderboard have predictive signal on short-term BTC direction, and can it be traded at the prices actually on offer?

The result: the signal was real enough to be right about 71% of the time, and the prices already reflected it. The answer to the second half was no.

## Results

| Measure | Result |
|---|---|
| Settled paper trades | 1,493 (1,053 wins, 440 losses), all in May 2026 |
| Hit rate | 70.5% (95% CI about 68% to 73%) |
| Break-even hit rate at the prices paid | about 70.4%, size-weighted |
| Win probability the strategy assumed | 88.2% |
| Paper bankroll | $20.00 start, $202.84 peak on May 5, $0.10 final (-99.51%) |

These come from the public [`signal/PAPER-TRADER-LOG.md`](https://github.com/PricePepel/polymarket-tracker/blob/HEAD/signal/PAPER-TRADER-LOG.md) and `signal/state.json`. The log's counters show 1,493 trades, and the ticket list in `state.json` holds 1,495. The hit-rate interval and break-even figure are computed from the ticket list.

## What happened

**The early sample flattered it.** Around the first 51 trades the hit rate was about 81%, with a 95% confidence interval of 63% to 90%. The full sample landed at 70.5%, inside that interval and well below the 88.2% the strategy had locked in.

**There was no edge at those prices.** A share bought at 70 cents has to win 70% of the time to break even. The size-weighted hit rate was almost exactly that. Wins of $880 and losses of $900 on $3,023 of paper volume come to a loss of 0.66%.

**Sizing turned no edge into ruin.** Position size came from quarter Kelly on the assumed 88.2%, capped at 10% of bankroll. Betting a fixed share of the bankroll with no edge loses money through compounding even when wins and losses nearly cancel out. A lucky start took $20 to $202.84 in four days. The bankroll fell below $10 by May 13 and below $1 by May 20.

**Then it stopped itself.** Size rounds down to the nearest cent, so at $0.10 a 10% ticket rounds to zero. The last ticket was on May 31. Scanning continued until October 28 without placing another.

## What the numbers point to

- A win probability fixed from a small sample needs a rule that compares it with live results. Locking parameters prevents overfitting, but it also meant nothing corrected a wrong assumption.
- Hit rate alone says little in a market. What matters is hit rate against the break-even implied by the entry price.
- Position sizing should assume the edge may be zero until a large live sample says otherwise.

## Architecture

```
cron-job.org ──► workflow_dispatch ─┐
                                    ├──► scan: track.mjs ──► data/btc-5m/YYYY-MM-DD.md, state.json
GitHub Actions schedule ────────────┘         │
                                              ▼
                                     signal/ (features, sizing, live, resolve)
                                              │
                                              ▼
                              signal/state.json ──► signal/PAPER-TRADER-LOG.md
```

`track.mjs` has no dependencies and uses native `fetch`. It can also run locally and write to Obsidian through the Local REST API.

The signal layer computes features for each market, including net USD flow in the late window and how concentrated the flow is across wallets. It decides 90 seconds before close. Tickets settle against resolved outcomes, and the log regenerates entirely from `state.json`.

## Stack

- **Scanner:** Node 20, native `fetch`, no dependencies
- **Schedule:** GitHub Actions cron every 5 minutes (15 at first), plus cron-job.org as a second trigger
- **State:** `state.json` and `signal/state.json`, committed to the repo
- **Validation:** walk-forward and backtest scripts in `signal/bin/`

## Notable decisions

1. **Two triggers, not one.** GitHub's schedule can run late, so an external cron also starts the workflow. A 10-minute lookback over recently closed markets recovers any that drift skipped.

2. **State as a committed file, not a database.** There's no database to keep alive, and `git log` gives a full audit trail of what was seen and when.

3. **Parameters locked after walk-forward validation.** This avoided re-tuning on results, and it's also why the optimistic win probability survived. See the results above.

4. **Paper only.** No real capital was ever at risk, which is what made it safe to run this to the end and publish the outcome.
