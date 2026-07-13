# Pulse research archive

This directory is the review package for the Pulse crypto-trading project as of 2026-07-13.

## What is included

- `HYPOTHESES_AND_RESULTS.md` — the decision register: tested ideas, failed ideas, surviving mechanisms, and the current next gate.
- `STUDIES_SNAPSHOT.md` — preregistrations, results, adversarial audits, and data-manifest excerpts.
- `CODE_SNAPSHOT.md` — the reproducible research engines and independent verifiers used for the main tests.
- `DASHBOARD_SNAPSHOT.md` — the dashboard measurement contract and the distinction between MFE, RET24, costs, and clustered significance.

## Current scientific status

The simple directional rule “volume spike → buy” is killed. The six-year test and the live diary both show that MFE peaks are not executable returns; RET24 after realistic costs is the relevant quantity. The research machine is still alive, but no trading edge has been confirmed.

The active frontier is structural flow: forced liquidations, taker imbalance, order-book pressure, and execution-aware maker economics. H-FLOW is a data/measurement gate first; it is not a live trading signal.

## Reproducibility and scope

The package preserves the frozen rules and negative results. It intentionally excludes raw market dumps, runtime logs, databases, server addresses, credentials, Telegram/channel configuration, and paid-data samples. Those files are either private, large, or unsafe to publish; the manifests and code document how to reproduce or audit them.

No file in this directory authorizes live trading. Every new outcome must pass a frozen preregistration, point-in-time checks, explicit cost grid, clustered statistics, and an independent verifier.
