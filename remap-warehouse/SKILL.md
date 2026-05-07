---
name: remap-warehouse
description: Rebuilds the data warehouse reference document by scanning live schema metadata and sampling data via the Metabase API. Use when the user says "remap the warehouse", "update the warehouse reference", "rescan the schema", "refresh the schema", "rebuild the warehouse doc", "sync the warehouse reference", "update the schema docs", "regenerate the warehouse reference", "re-scan the warehouse", or similar.
---

# Remap Warehouse

Scans the live data warehouse via the Metabase API and rewrites `.claude/WAREHOUSE_REFERENCE.md` in the current project with up-to-date schema, table, column, and sample data information.

**Runs fully autonomously — no user interaction required after invocation.**

> **Note:** The core logic lives in `dagster/scripts/remap_warehouse.py` — a standalone Python script that is also invoked by the `warehouse_reference` Dagster asset. When Claude Code invokes this skill, it should call that script directly rather than reimplementing the logic:
>
> ```bash
> METABASE_PASSWORD=<password> python dagster/scripts/remap_warehouse.py
> ```
>
> Only fall back to the manual steps below if the script is unavailable or needs extending.

---

## Target File

Always write output to:
```
<project-root>/.claude/WAREHOUSE_REFERENCE.md
```

Where `<project-root>` is the current working directory (i.e. the project Claude Code was opened in).

---

## Step 1 — Authenticate with Metabase

Use the Metabase credentials from `CLAUDE.md`:
- **URL:** `https://metabase.noholabs.com`
- **Credentials:** stored in CLAUDE.md under "Metabase Access"

```bash
curl -s -X POST https://metabase.noholabs.com/api/session \
  -H "Content-Type: application/json" \
  -d '{"username": "<email>", "password": "<password>"}'
# Returns: {"id": "<session-token>"}
```

Store the session token for all subsequent requests. If authentication fails, stop and report the error.

---

## Step 2 — Discover All Tables

Fetch the full database metadata for the warehouse (DB ID 3):

```bash
curl -s "https://metabase.noholabs.com/api/database/3/metadata" \
  -H "X-Metabase-Session: <token>"
```

This returns all schemas, tables, and fields. Parse the response to build a complete inventory:
- Schema names
- Table names and IDs per schema
- Field names, types, and descriptions per table

Print a summary of what was discovered:
```
[remap] discovered N schemas, N tables
[remap]   schema_1: N tables
[remap]   schema_2: N tables
  ...
```

---

## Step 2b — Source Attribution (Trace Fields to Code)

For each warehouse schema, scan the relevant local codebase to identify where each column originates. This produces a `source` annotation for each column in the reference doc.

### Schema → Source mapping

| Warehouse schema | Source system | Where to look |
|---|---|---|
| `bronze_app_db_postgres_public` | patient-app NestJS backend | `/Users/johnhofrichter/noho/projects/patient-app/apps/backend/src/**/*.entity.ts` |
| `bronze_canvas` | Canvas EMR (external SaaS) | No local code — annotate as "Canvas EMR API" |
| `bronze_stripe_*` | Stripe (via Fivetran) | No local code — annotate as "Stripe / Fivetran" |
| `bronze_ramp` | Ramp (via Fivetran) | No local code — annotate as "Ramp / Fivetran" |
| `bronze_braze` | Braze (via Fivetran) | No local code — annotate as "Braze / Fivetran" |
| `silver_*`, `transform_references` | data-pipeline SQL views | Defined in warehouse DB directly; source from `pg_get_viewdef()` |

### For `bronze_app_db_postgres_public` — TypeORM entity scan

TypeORM maps class names and camelCase properties to snake_case table/column names. Use this to match warehouse columns back to source code:

**Find all entity files:**
```bash
find /Users/johnhofrichter/noho/projects/patient-app/apps/backend/src -name "*.entity.ts"
```

**For each entity file, extract:**
1. The `@Entity()` class — converts to snake_case table name (e.g. `ShopOrder` → `shop_order`)
2. Each `@Column()` / `@PrimaryGeneratedColumn()` property — camelCase to snake_case (e.g. `auth0Id` → `auth0_id`, `membershipStripeCustomerId` → `membership_stripe_customer_id`)
3. The `@ApiProperty({ description: '...' })` on each column — use as human-readable description
4. The TypeScript type (e.g. `USStateCode | null`) — note nullable status

**camelCase → snake_case conversion rule:**
Insert `_` before each uppercase letter and lowercase: `membershipType` → `membership_type`, `RUOConsentSignedAt` → `r_u_o_consent_signed_at` (or check actual DB column if uncertain).

**Example output for `user.entity.ts`:**

| Column | Type | Source | Description |
|---|---|---|---|
| `id` | int4 | `patient-app/src/users/user.entity.ts:12` `@PrimaryGeneratedColumn()` | User ID |
| `auth0_id` | varchar | `patient-app/src/users/user.entity.ts:16` `@Column()` | Auth0 user ID |
| `state` | varchar | `patient-app/src/users/user.entity.ts:54` `@Column()` | User's US state code (e.g. CA) — set during intake |

**Table name matching:** The warehouse table is named `<entity_snake_case>` under `bronze_app_db_postgres_public`. Match by converting the `@Entity()` class name to snake_case.

### For `silver_*` — View definition scan

For silver views, fetch the SQL view definition from the warehouse and include it as the "source":

```sql
SELECT pg_get_viewdef('silver_membership.membership_subscriptions'::regclass, true);
```

Annotate each column's source as `silver_<schema>/<view>.sql — derived from <bronze_table>.<column>` based on what appears in the SELECT.

### Attribution format

In the WAREHOUSE_REFERENCE.md column tables, add a **Source** column:

```markdown
| Column | Type | Source | Notes |
|---|---|---|---|
| `state` | varchar | [`user.entity.ts:54`](patient-app/apps/backend/src/users/user.entity.ts) — `@Column() state: USStateCode \| null` | US state code set during intake |
| `auth0_id` | varchar | [`user.entity.ts:16`](patient-app/apps/backend/src/users/user.entity.ts) — `@Column()` | Auth0 user ID |
| `_fivetran_synced` | timestamptz | Fivetran internal | Sync timestamp, not a source field |
```

For external schemas (Canvas, Stripe, Ramp, Braze), use the Source column to note the external system rather than leaving it blank.

Print progress:
```
[remap] source attribution: scanning patient-app entities (31 files)...
[remap]   matched 12/15 columns in bronze_app_db_postgres_public.user
[remap]   matched 8/8 columns in bronze_app_db_postgres_public.membership
[remap] source attribution: bronze_canvas → Canvas EMR (no local code)
[remap] source attribution: bronze_stripe_* → Stripe / Fivetran
```

---

## Step 2c — Fetch Silver Layer Comments

Open the Aptible tunnel (reuse if already running) and query `pg_description` to capture `COMMENT ON` documentation for all silver-layer objects:

```bash
export PATH="/opt/homebrew/opt/libpq/bin:$PATH"
nc -z 127.0.0.1 15441 || (
  tmux kill-session -t aptible-tunnel 2>/dev/null
  tmux new-session -d -s aptible-tunnel \
    "aptible db:tunnel data-warehouse --environment noho-production --type postgresql --port 15441"
  sleep 12
)
```

```sql
-- Schema comments
SELECT nspname AS schema_name,
       obj_description(oid, 'pg_namespace') AS comment
FROM pg_namespace
WHERE nspname IN ('silver_membership', 'silver_meds', 'silver_clinical', 'transform_references')
ORDER BY nspname;

-- View/table comments
SELECT n.nspname AS schema_name,
       c.relname AS object_name,
       CASE c.relkind WHEN 'v' THEN 'view' WHEN 'r' THEN 'table' WHEN 'm' THEN 'matview' END AS kind,
       obj_description(c.oid, 'pg_class') AS comment
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname IN ('silver_membership', 'silver_meds', 'silver_clinical', 'transform_references')
  AND c.relkind IN ('v', 'r', 'm')
ORDER BY n.nspname, c.relname;

-- Column comments for silver views
SELECT n.nspname AS schema_name,
       c.relname AS table_name,
       a.attname AS column_name,
       col_description(c.oid, a.attnum) AS comment
FROM pg_attribute a
JOIN pg_class c ON c.oid = a.attrelid
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname IN ('silver_membership', 'silver_meds', 'silver_clinical', 'transform_references')
  AND a.attnum > 0
  AND col_description(c.oid, a.attnum) IS NOT NULL
ORDER BY n.nspname, c.relname, a.attnum;
```

Store these comments — they will be included verbatim in the Silver Layer section of the reference document.

---

## Step 3 — Sample Key Tables

For each table, run a row count and a sample query via the Metabase dataset API:

```bash
# Row count
curl -s -X POST "https://metabase.noholabs.com/api/dataset" \
  -H "X-Metabase-Session: <token>" \
  -H "Content-Type: application/json" \
  -d '{"database": 3, "type": "native", "native": {"query": "SELECT COUNT(*) FROM <schema>.<table>"}}'

# Sample rows
curl -s -X POST "https://metabase.noholabs.com/api/dataset" \
  -H "X-Metabase-Session: <token>" \
  -H "Content-Type: application/json" \
  -d '{"database": 3, "type": "native", "native": {"query": "SELECT * FROM <schema>.<table> LIMIT 3"}}'
```

**Prioritize tables with rows.** Skip tables with 0 rows for sampling — note them as empty stubs.

For each non-empty table, collect:
- Row count
- Column names and types (from metadata)
- 1–3 sample rows to understand actual data shape and values

Print progress per schema:
```
[remap] scanning bronze_stripe_memberships (N tables)...
[remap]   subscription_history: 69 rows ✓
[remap]   invoice: 266 rows ✓
[remap]   apple_pay_domain: 6 rows ✓
[remap]   transfer: 0 rows — stub, skipping sample
  ...
```

---

## Step 4 — Check Data Freshness

Run a freshness check across all schemas that have a `_fivetran_synced` column:

```sql
SELECT '<schema>' AS source, MAX(_fivetran_synced) AS last_synced
FROM <schema>.<key_table>
UNION ALL
...
ORDER BY last_synced ASC
```

Record the last sync timestamp per schema to include in the reference doc.

---

## Step 5 — Write WAREHOUSE_REFERENCE.md

Using all collected data, write a comprehensive reference document to `<project-root>/.claude/WAREHOUSE_REFERENCE.md`.

The document must follow this structure:

```markdown
# Data Warehouse — Reference

> Last remapped: <ISO timestamp of this run>

**Instance:** ...
**Database:** ...
**Sync:** Fivetran. Every table has `_fivetran_synced`. Filter `_fivetran_deleted IS NOT TRUE` for active records.
**No FK constraints** — all joins by convention on ID columns.

---

## Data Freshness

| Source | Last Synced |
|---|---|
| bronze_stripe_memberships | <timestamp> |
| ... | ... |

---

## Schemas Overview

| Schema | Source | Tables | Key Tables |
|---|---|---|---|
| `schema_name` | Source system | N | table_a, table_b |
...

---

## Schema: `<schema_name>`

**Purpose:** <inferred from table names and sample data>

### Tables

#### `<schema>.<table>` — N rows
<One-sentence description inferred from column names and sample data>

| Column | Type | Source | Notes |
|---|---|---|---|
| col_name | type | source file:line or "Fivetran" or "Canvas EMR" | notes from sample data |
...

<Repeat for each non-empty table>

**Empty stubs (0 rows):** table_a, table_b, table_c

---
<Repeat for each schema>

---

## Silver Layer

> These are hand-crafted views that clean and filter bronze data. Descriptions are pulled from native PostgreSQL `COMMENT ON` metadata — update them with `COMMENT ON VIEW <view> IS '...'`.

### Schema: `silver`
<schema comment from pg_description>

| View | Description | Source Table |
|---|---|---|
<one row per view, using the comment from pg_description as the description>

### Schema: `silver_clinical`
<schema comment from pg_description>

| Table | Description |
|---|---|
<one row per table>

### Schema: `transform_references`
<schema comment from pg_description>

| Table | Description |
|---|---|
<one row per table>

---

## Cross-Schema Relationships

<Infer from column names — e.g. customer_id columns, email joins, external_id patterns>

| From | Column | To | Notes |
|---|---|---|---|
...

---

## Query Patterns

<Include 4–6 common query patterns inferred from the schema — MRR calculation, active subscriptions, revenue by product, spend by category, etc.>
```

**Guidelines for writing the document:**
- Infer purpose and descriptions from column names, sample data values, and table names — don't leave these blank
- Note monetary unit conventions (cents vs dollars) where applicable
- Flag SCD2 tables (those with `_fivetran_start`, `_fivetran_end`, `_fivetran_active`) with the correct query pattern
- Include the "Last remapped" timestamp so readers know when it was generated
- Keep the document human-readable and agent-readable — it will be used by both

---

## Step 6 — Trigger Metabase Schema Sync

After writing the reference doc, trigger a Metabase database sync so any new views or tables discovered during remapping become queryable in the Metabase UI.

```bash
# 1. Trigger a full schema sync for DB 3
curl -s -X POST "https://metabase.noholabs.com/api/database/3/sync_schema" \
  -H "X-Metabase-Session: <token>" \
  -H "Content-Type: application/json"

# 2. Trigger a full field values rescan
curl -s -X POST "https://metabase.noholabs.com/api/database/3/rescan_values" \
  -H "X-Metabase-Session: <token>" \
  -H "Content-Type: application/json"
```

Both calls return 200 with an empty body on success. If either fails, log the HTTP status but do not abort — the reference doc is already written.

Print:
```
[remap] metabase sync triggered — new views will appear in UI within ~30s
```

---

## Step 7 — Update Notion

After writing the reference doc, update the Notion warehouse doc page with the full reference content.

**Notion credentials:**
- Integration token: `NOTION_API_KEY` env var (available in shell per global `CLAUDE.md`)
- Target page ID: `342b0007-eb73-809b-af9a-ee60037b7d8f` (warehouse doc page)

If `NOTION_API_KEY` is not set, skip this step and print:
```
[remap] ⚠ skipping Notion update — NOTION_API_KEY not set
```

Otherwise:

**1. Clear all existing blocks from the page:**
```bash
# Get existing block IDs (paginate if has_more)
curl -s "https://api.notion.com/v1/blocks/342b0007-eb73-809b-af9a-ee60037b7d8f/children?page_size=100" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2022-06-28"

# Delete each block
curl -s -X DELETE "https://api.notion.com/v1/blocks/<block_id>" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2022-06-28"
```

**2. Write the warehouse reference as Notion blocks in batches of 100:**

Convert WAREHOUSE_REFERENCE.md to Notion blocks:
- `## …` → `heading_2`, `### …` → `heading_3`
- Markdown tables → Notion `table` blocks (with `table_row` children, first row as header)
- All other non-empty lines → `paragraph`

```bash
curl -s -X PATCH "https://api.notion.com/v1/blocks/342b0007-eb73-809b-af9a-ee60037b7d8f/children" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Notion-Version: 2022-06-28" \
  -d '{"children": [<batch of up to 100 block objects>]}'
```

Repeat until all blocks are written. If any batch returns an error, raise immediately — do not silently skip.

Print:
```
[remap] writing N blocks to Notion...
[remap] Notion warehouse doc updated
```
or on error:
```
[remap] ⚠ Notion update failed: <error>
```

> **Note:** The script (`dagster/scripts/remap_warehouse.py`) implements this automatically via `_update_notion_doc()`. Call the script rather than reimplementing this step manually.

---

## Step 8 — Report

Print a completion summary:

```
[remap] ✓ WAREHOUSE_REFERENCE.md written
[remap]   schemas: N
[remap]   tables documented: N (N empty stubs noted)
[remap]   path: <absolute path to file>
[remap]   freshness: oldest sync was <schema> at <timestamp>
[remap]   metabase: schema sync + rescan triggered ✓
```
