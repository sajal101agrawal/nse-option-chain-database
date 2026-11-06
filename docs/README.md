# Documentation

This folder documents the NSE Option Chain Database: the data sources, the schema, the formulas,
and the operations. For the project overview and the quick start, read the
[main README](../README.md).

## Architecture

| Document | Topic |
|---|---|
| [Data sources](architecture/data-sources.md) | Bhavcopy fallback order, risk-free rate, F&O universe, trading calendar, events, and corporate actions. |
| [Database dictionary](architecture/database-dictionary.md) | Every table and column. |
| [Live option chain](architecture/live-option-chain.md) | Live provider design (Kite, Dhan, NSE, Yahoo), unit policy, and fallbacks. |

## Calculations

| Document | Topic |
|---|---|
| [Formula audit](calculations/formula-audit.md) | Unit convention, IV and Greeks, ATM IV, Yang-Zhang RV, VRP, forward factor, skew, RSI, percentiles, and backtests. |

## Operations

| Document | Topic |
|---|---|
| [Operations runbook](operations/runbook.md) | Setup, daily update, recompute, backfill, events, live data, Kite login, and error logs. |
| [Retries, failures, and validation](operations/retries-errors-validation.md) | Retry policy, source fallback, error logging, and validation scripts. |
| [AWS deployment](operations/aws-deployment.md) | EC2, RDS, S3 dumps, and GitHub Actions deploy. |
| [Corporate-action RV remediation](operations/corporate-action-rv-remediation.md) | Factor policy and production rollout for RV/VRP repair. |
| [Forward Factor remediation](operations/forward-factor-remediation.md) | One-time Call/Put Forward Factor backfill. |
| [Validation report, 2026-05-20](operations/current-validation-2026-05-20.md) | Row counts and validation results for the 5-year load. |
