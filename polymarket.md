# Polymarket Tracker

**Year:** 2025
**Status:** Personal · Production cron, hands-off
**Role:** Solo
**Repo:** [github.com/PricePepel/polymarket-tracker](https://github.com/PricePepel/polymarket-tracker)

---

## What it is

A cron-driven scraper plus paper-trader signal layer that watches the top 50 Polymarket CRYPTO accounts trading BTC up-or-down 5-minute markets. The scraper logs new positions; the signal layer derives a "consensus direction" from the wallet activity and runs a paper-trading harness that records hypothetical entries.

## Problem

I wanted to know whether the wallets that consistently top the Polymarket CRYPTO leaderboards have predictive signal on BTC short-term direction. The Polymarket public API exposes positions per wallet, but there's no aggregation across wallets, no historical state, and no signal layer. So I built it.

## Architecture

Single zero-dependency Node 20 script (`track.mjs`, native fetch only) deployed two ways for redundancy.

```
                    ┌─► GitHub Actions cron (every 15 min) ──► commit state.json + log
Polymarket API ───► track.mjs                                        │
                    └─► Windows Task Scheduler (local) ──► Obsidian Local REST API
                                                                     │
                                                                     ▼
                                                        signal-layer.mjs (PAPER mode)
                                                                     │
                                                                     ▼
                                                              paper-trade ledger
```

`state.json` is the persisted state file: last-seen timestamps per wallet plus recent transaction hashes for dedup. It's committed to the repo on every run, which means the GitHub Actions workflow naturally serves as both deploy and as the audit log of what was seen when. Anyone (including me, weeks later) can `git log state.json` to see the wallet activity timeline.

The signal layer is a separate `signal-layer.mjs` that consumes the same scraper output but doesn't touch Polymarket itself. It derives a directional consensus from the top-50 wallets' positions (weighted by wallet ROI) and runs a paper-trading harness in `PAPER` mode: every "trade" is recorded but no real order is placed. The mode is gated by an environment variable; flipping to `LIVE` would require additional execution wiring that I deliberately haven't built.

## Stack

- **Runtime:** Node 20 (native `fetch`, no dependencies)
- **Cron 1:** GitHub Actions, scheduled workflow with `workflow_dispatch` triggered by cron-job.org webhook every 15 minutes
- **Cron 2:** Windows Task Scheduler running the same script locally, writing to Obsidian via the Local REST API
- **State:** Plain `state.json` committed to the repo
- **Signal layer:** Same Node 20 runtime, same zero-dep policy, separate file

## Verification

- **Workflow stays green** across 6+ months of runs.
- Observed cron response: HTTP 204 (success, no body — the workflow_dispatch ack).
- **Security-audited end to end** before going live: no secrets in committed state, no sensitive data in logs, dedup logic prevents replay attacks if a malicious wallet replays old txs.
- Backtested the signal layer against committed historical state. Documented findings (paper PnL, hit rate) in the repo's `notes/` directory.

## Notable decisions

1. **Zero dependencies.** This was non-negotiable for me. Polymarket's API exposes a clean enough JSON shape that adding `axios` or `polymarket-sdk` would have been pure dependency creep. Native `fetch` is enough; the script is 200-ish lines and there's no `npm audit` surface.

2. **Two crons, not one.** GitHub Actions is reliable but not 100% reliable, and the runners can be slow on cold starts. Windows Task Scheduler running locally on my dev box is a cheap belt-and-suspenders setup. Both write to the same dedup-aware state, so duplicates are rejected at the state layer.

3. **State as a committed file, not a database.** This is unconventional but the right call for a solo personal tool: no DB to keep alive, no migration to manage, full audit trail comes for free via `git log`, and forking the repo gives you the data alongside the code.

4. **Paper mode locked behind an env var.** I built the signal layer because the question "do these wallets have predictive signal?" is interesting, but I have no intention of going live without rigorous validation. The `PAPER` default is the safe rail; flipping to `LIVE` requires conscious work, not a misconfigured deploy.
