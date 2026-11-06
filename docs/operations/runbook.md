# Operations Runbook

This runbook lists the commands to set up, update, repair, and operate the service. For the first
setup, read the [Quick start](../../README.md#quick-start) in the main README.

## Server setup and deploy

Set up a server:

```bash
scripts/setup_server.sh
```

Run the full idempotent deploy flow:

```bash
scripts/deploy_server.sh
```

Initialize a fresh server with the last five years of NSE F&O history:

```bash
scripts/bootstrap_history.sh
```

Export a validated local database for an RDS restore:

```bash
scripts/export_postgres_dump.sh
```

For EC2, RDS, S3 dumps, and GitHub Actions, read [aws-deployment.md](aws-deployment.md).

## Daily EOD update

Run one daily update after NSE publishes the bhavcopy:

```bash
python scripts/daily_update.py --date YYYY-MM-DD
```

Install the daily ETL cron job:

```bash
scripts/install_daily_etl_cron.sh
```

`scripts/daily_update.py` refreshes the NSE and Yahoo result events on each run. To skip the
event refresh, pass `--skip-events`.

## Recompute derived analytics

After a formula change, recompute the derived analytics:

```bash
python scripts/recompute_option_volume.py
python scripts/recompute_symbol_aggregates.py
python scripts/recompute_rsi_metrics.py
python scripts/recompute_straddle_pnl.py
```

`recompute_option_volume.py` refreshes `avg_option_volume` and `option_volume_60d` from the raw
CE and PE option rows.

## Corporate actions and RV/VRP remediation

Sync NSE corporate actions, then preview or execute the all-symbol RV/VRP remediation:

```bash
python scripts/sync_corporate_actions.py --start YYYY-MM-DD --end YYYY-MM-DD
python scripts/backfill_corporate_action_metrics.py
python scripts/backfill_corporate_action_metrics.py --execute
```

The preview is read-only. Read
[corporate-action-rv-remediation.md](corporate-action-rv-remediation.md) before you run the
production backfill.

## Forward Factor backfill

After you deploy the additive schema, preview or execute the one-time historical Call/Put Forward
Factor backfill:

```bash
python scripts/backfill_forward_factors.py
python scripts/backfill_forward_factors.py --execute
```

The preview is read-only. The execute run fills the separate call and put constant-maturity IVs
and Forward Factors for existing `symbol_daily_metrics` rows. It does not change the existing
average Forward Factor. The run is idempotent and audited. Read
[forward-factor-remediation.md](forward-factor-remediation.md) for the full sequence.

## Result events and upcoming earnings

The service stores completed and filed result events from NSE corporate-event data in `events`,
with `event_type='RESULT'` and `source='nse:event-calendar'`.

The service stores upcoming earnings dates from the Yahoo Finance earnings calendar, with
`source='yahoo:earnings-calendar'`. It uses only the Yahoo `Earnings Date` field. Dividend dates
and other corporate actions do not go into `events`.

The service stores price-adjusting events from NSE in `corporate_actions`. The pipeline uses them
to adjust the in-memory OHLC window for Yang-Zhang RV and RSI. The raw `equity_historical` prices
do not change. `GET /api/symbol/{symbol}/corporate-actions` shows the factor and the verification
status of each action.

To refresh only the result events, for example on a weekend or a market holiday, run:

```bash
python scripts/update_result_events.py
```

Options:

```bash
python scripts/update_result_events.py --skip-nse
python scripts/update_result_events.py --symbols RELIANCE,TCS,INFY --skip-nse
```

On a production server:

```bash
docker compose -p pov-db -f docker-compose.prod.yml run --rm api python scripts/update_result_events.py
docker compose -p pov-db -f docker-compose.prod.yml exec -T redis redis-cli FLUSHDB
```

Each row from `GET /api/all-dashboard` contains these result fields:

- `result_date`: the next upcoming result date.
- `result_event`: `true` when a future result event exists.
- `upcoming_events`: the upcoming result events, sorted by `event_date`. Each event has
  `event_date`, `event_type`, `description`, and `source`.

## Live data

`GET /api/live` returns all active symbols in one response for frontend polling.

The production live path uses Kite. Set these values in `.env`:

```text
LIVE_QUOTE_PROVIDER=kite
LIVE_OPTION_SUMMARY_PROVIDER=kite
LIVE_OPTION_CHAIN_PROVIDER=kite
```

Kite supplies live quote snapshots but not IV. The service quotes the nearest ATM CE and PE for the
30, 60, and 90 day expiry targets. It calculates the call and put IV from the live option prices.
Then it uses the same live IV30, IV60, slope, and forward-factor pipeline. Dhan is an alternate
provider. NSE is the option-chain fallback.

The service caches live quote payloads in Redis for `LIVE_CACHE_TTL_SECONDS` seconds (default
`300`) and upserts them into `live_symbol_metrics`. API reads use Redis first and then the latest
PostgreSQL live row. After market close, the dashboard and detail APIs show the last collected live
snapshot, not the EOD metrics from the day before.

The live payload contains these values:

- The current price.
- The live all-strike CE+PE option volume.
- The ATM strike, IV, and leg volume.
- The recomputed `iv_30/60/90`, `call_iv_30/60/90`, `put_iv_30/60/90`, `fwdv_3060`,
  `fwdfct_3060`, `call_fwdfct_3060`, `put_fwdfct_3060`, `max_fwdfct_3060`, `fev_30`, and
  `iv_slope_3060`.

When live values overlay the EOD values, the EOD values stay under `eod_*` keys.

These routes overlay the latest live row at read time:

- `/api/all-dashboard`
- `/api/symbol/{symbol}`
- `/api/symbol/{symbol}/history`
- `/api/symbol/{symbol}/term-structure`
- `/api/symbol/{symbol}/volatility-cone`

Fallback rules:

- If a Kite or Dhan option-summary call fails, the service logs the failure and uses NSE
  `option-chain-v3`.
- If a Kite or Dhan quote call fails, the service logs the failure and uses Yahoo for underlying
  prices.
- If a Kite or Dhan call fails, `GET /api/live/{symbol}/option-chain` uses NSE. It does not return
  the provider failure to clients.

`LIVE_SYMBOLS=all` (or blank) makes background and manual live refreshes use every active symbol in
`symbol_universe`. To poll a smaller watchlist, set a comma-separated list.

## Daily Kite login

You cannot make a Kite access token from the API key and secret alone. Do these steps each day:

1. Open `GET /api/admin/kite/login-url`.
2. Complete the Kite login.
3. Send the redirect `request_token` in the JSON body of `POST /api/admin/kite/session`.

The service stores the access token in Redis and in `broker_access_tokens`. The worker has a
06:05 IST refresh hook, but the hook can only exchange a fresh `KITE_REQUEST_TOKEN`. If no fresh
token is available, the worker logs `kite_token_refresh` with the login URL.

## Admin error logs

`GET /api/admin/error-logs` returns paginated rows from `error_log` for operational screens.

Filters: `limit`, `offset`, `search`, comma-separated `task_name`, `symbol`, `source`,
`error_type`, `resolved`, `trade_date_min/max`, and `created_at_min/max`.

Sort with `sort_by` and `sort_dir`. The valid sort fields are `created_at`, `trade_date`, `id`,
`task_name`, `symbol`, `source`, `error_type`, and `resolved`.

Protect the `/api/admin/*` routes. The app does not authenticate them. Without a guard, any
visitor can start pipeline runs or replace the Kite session.

## Tests

```bash
python3 -m venv .venv
.venv/bin/pip install -e ".[dev]"
.venv/bin/pytest -q
```

## Formula policy

The service stores all volatilities as decimals: `0.284` means `28.4%`. API callers format
percentages at the presentation layer. The formula audit is in
[formula-audit.md](../calculations/formula-audit.md).
