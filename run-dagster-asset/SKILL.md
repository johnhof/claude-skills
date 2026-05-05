---
name: run-dagster-asset
description: Run a Dagster asset locally via Docker Compose with a local DuckDB warehouse, then open the results in the DuckDB UI.
---

The user wants to run a Dagster asset locally. The asset name may be provided as an argument (e.g. `/run-dagster-asset google_sheets_sync`). If no asset is specified, ask the user which asset to run.

## Steps

1. Run the asset using Docker Compose from the `dagster/` directory:
   ```bash
   cd /Users/johnhofrichter/noho/projects/data-pipeline/dagster
   ASSET=<asset_name> docker compose -f docker-compose.run-asset.yml up --build
   ```
   No tunnel needed — the compose file uses a local DuckDB file at `dagster/output/warehouse.duckdb`.

2. Stream the container logs and report:
   - Which tables/ranges were loaded
   - Any warnings or errors
   - Final success or failure

3. After a successful run, open the DuckDB UI:
   ```bash
   duckdb /Users/johnhofrichter/noho/projects/data-pipeline/dagster/output/warehouse.duckdb -ui
   ```
   This opens a browser-based SQL interface pointed at the local output file.

4. Clean up containers:
   ```bash
   docker compose -f docker-compose.run-asset.yml down
   ```

## Notes

- Output file: `dagster/output/warehouse.duckdb` (gitignored, persists between runs)
- The compose file uses `WAREHOUSE_DUCKDB_PATH=/output/warehouse.duckdb` — no Postgres or tunnel needed.
- `GOOGLE_SERVICE_ACCOUNT_JSON` is not needed — the compose file mounts `google_service_account.json` directly.
- The default asset (when `ASSET` is not set) is `google_sheets_sync`.
- The compose file is at `dagster/docker-compose.run-asset.yml`.
