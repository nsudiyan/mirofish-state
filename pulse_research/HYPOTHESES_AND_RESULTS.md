# Pulse: hypothesis register and results

Snapshot date: 2026-07-13. This is a research record, not investment advice.

## Decision table

| Hypothesis | Question | Evidence | Status |
|---|---|---|---|
| Volume/radar long | Does a large volume shock with quiet price predict a profitable long? | Six-year PIT run, ~6.7k trades; net about -597% at 0.31%/leg, negative train/validation/test; the live diary separates MFE from RET24 and remains negative after costs. | **KILLED / NO-TRADE** |
| BTC-hedged radar | Does the selected coin beat its BTC beta? | Gross alpha about -0.168% per trade; full and untouched test negative. | **KILLED / NO-TRADE** |
| Timeframe/holding variants | Does the candle detector work on another timeframe or horizon? | Pre-registered sweep: no positive cell survived the gates. | **KILLED / NO-TRADE** |
| OI × funding crowding | Does rising OI plus one-sided funding identify a forced unwind? | As-of rerun: net about -611%; gross about +0.044%/position. Long crowded-shorts leg about +9.7 bp/day before costs; crowded-longs leg negative. | **KILLED as a portfolio rule; mechanism remains a parent observation** |
| Cash-and-carry | Does funding pay for a delta-neutral carry trade? | Funding collected roughly 5–6% annualized gross on BTC/ETH/SOL, but turnover fees and breaches dominate; net about -129%. | **KILLED / NO-TRADE** |
| Cross-venue executable arbitrage | Can public three-venue quotes prove a tradable 14–71 bp cross? | The first economic prereg was voided by receipt/content-time and coverage defects. OBS-02 is timing-only until observability is proven. | **MEASUREMENT GATE, not an edge** |
| H-FLOW forced short-cover | Do liquidations + taker buy imbalance + ask depletion create a continuation that can survive execution costs? | Live causal collector and feature-only replay are in place. Outcomes stay closed until the firewall and prereg gate are complete. | **OPEN / DATA COLLECTION** |

## What the dashboard did and did not prove

`MFE` is the maximum excursion during the next 24 hours. It is a ceiling, not an executable exit. `RET24` is the close-to-close result available to a mechanical 24-hour hold. The dashboard now reports both, plus the fixed-cost grid and UTC-day clustering.

The good-looking `MFE` cards and “surprise up” labels therefore cannot be read as winning trades. The simple radar-alt class currently has no confirmed directional edge after this correction.

## Why the next test is structural

Raw volume does not identify who was forced to trade. The only remaining high-value public-data path is to identify a causal sequence:

1. forced short liquidations;
2. aggressive buy-side taker flow;
3. ask-side depth depletion;
4. an execution-aware entry and exit.

The test must be frozen before outcome data are opened. It must report costs at 0.14%, 0.31%, and 0.71% per round/leg convention, use UTC-day clustering and block bootstrap, and print a mechanical verdict. If it fails, the volume-based directional family is closed rather than endlessly retuned.
