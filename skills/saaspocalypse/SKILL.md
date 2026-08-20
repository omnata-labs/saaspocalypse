---
name: saaspocalypse
description: >
  Build metadata-driven applications on Snowflake where AI agents and data pipelines are
  first-class citizens. Uses Hybrid Tables, Stored Procedures, Snowflake Tasks, DCM Projects,
  and Snowflake App Runtime (Next.js). Everything is native Snowflake SQL.
  Triggers: new app, add entity, add table, hybrid table, stored procedure, validation,
  DCM, deploy, plan, framework, build app, UI design, page, navigation, reference data,
  semantic view, jobs, tasks, saaspocalypse, app runtime.
---

# Saaspocalypse

An application framework where AI agents and data pipelines are first-class citizens.
Everything is native Snowflake. No external services, no middleware, no proxies.

## Core Principles

1. **Business logic lives in Stored Procedures.** Validation, state transitions, audit,
   and job dispatch happen in SQL procs. NEVER put business logic in the UI.

2. **Agents and pipelines use the same interface as the UI.** Every write is
   `CALL schema.create_entity(...)` — whether from a CoCo agent, a pipeline, or the UI.

3. **Declarative deployment via DCM Projects.** Tables, procs, tasks, roles, and grants
   are all DEFINE statements. Deploy with `snow dcm plan` / `snow dcm deploy`.

4. **UI via Snowflake App Runtime.** Next.js app deployed with `snow app deploy`.
   Direct SQL access to Snowflake — no API proxy layer needed.

5. **Each app produces its own CoCo skill** documenting how agents interact with it.

## Architecture

```
Snowflake (all native — no external services)
├── Hybrid Tables              # App data (CRUD via stored procs)
├── Stored Procedures          # Validated writes
├── Jobs Table (Hybrid)        # Async work queue
├── Snowflake Task             # Job runner (scheduled, polls for pending jobs)
├── Semantic View              # Analytics (Cortex Analyst)
├── Reference Tables           # Existing Snowflake tables (read-only)
└── App Runtime (Next.js)      # UI with direct Snowflake SQL access
```

## Project Structure

```
apps/{name}/
├── manifest.yml              # DCM project manifest
├── pre_deploy.sql            # Hybrid tables (CREATE HYBRID TABLE IF NOT EXISTS)
├── post_deploy.sql           # Semantic view, other non-DCM objects
├── sources/
│   └── definitions/
│       ├── procs.sql         # DEFINE PROCEDURE statements
│       ├── tasks.sql         # DEFINE TASK for job runner
│       └── roles.sql         # DEFINE ROLE + GRANT statements
├── skill/                    # App-specific CoCo skill (for agents)
│   ├── .cortex-plugin/plugin.json
│   ├── SKILL.md
│   └── references/
├── reference_data.yaml       # External Snowflake tables documentation
├── app/                      # Next.js app (Snowflake App Runtime)
│   ├── snowflake.yml         # App Runtime config
│   ├── package.json
│   ├── app/                  # Next.js App Router
│   │   ├── layout.tsx
│   │   ├── page.tsx          # Dashboard
│   │   └── {entity}/
│   │       ├── page.tsx      # List view
│   │       └── [id]/page.tsx # Detail view
│   └── lib/
│       └── snowflake.ts      # Snowflake SQL helpers
└── ui/
    └── DECISIONS.md          # UI design decisions
```

## Data Access from the UI (App Runtime)

The Next.js app runs inside Snowflake with a built-in connection. No proxy needed.

### Internal Data API (via stored procs)

Server-side in Next.js route handlers or server components:

```typescript
// app/api/tickets/route.ts
import { callProc, query } from '@/lib/snowflake';

// CREATE — calls the validated stored procedure
export async function POST(request: Request) {
  const body = await request.json();
  const result = await callProc('support.create_ticket', [
    body.title, body.priority, body.reporter_email, body.organization_id
  ]);

  if (!result.success) {
    return Response.json(result, { status: 422 });
  }
  return Response.json(result, { status: 201 });
}

// LIST — direct SQL query
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const limit = searchParams.get('limit') || '50';
  const offset = searchParams.get('offset') || '0';

  const rows = await query(`
    SELECT * FROM support.tickets
    ORDER BY created_at DESC
    LIMIT ? OFFSET ?
  `, [limit, offset]);

  const [{ count }] = await query(`SELECT COUNT(*) as count FROM support.tickets`);

  return Response.json({ data: rows, total: count });
}
```

### Reference Data API (direct SELECT)

```typescript
// app/api/ref/organizations/route.ts
import { query } from '@/lib/snowflake';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const search = searchParams.get('search') || '';

  const rows = await query(`
    SELECT organization_id, organization_name, plan_tier
    FROM shared_data.crm.organizations
    WHERE organization_name ILIKE ?
    ORDER BY organization_name
    LIMIT 50
  `, [`%${search}%`]);

  return Response.json({ data: rows });
}
```

### Snowflake Connection Helper

```typescript
// lib/snowflake.ts
import { getConnection } from '@snowflake/app-runtime';

export async function query(sql: string, params: any[] = []) {
  const conn = await getConnection();
  const result = await conn.execute(sql, params);
  return result.rows;
}

export async function callProc(name: string, params: any[]) {
  const conn = await getConnection();
  const placeholders = params.map(() => '?').join(', ');
  const [result] = await conn.execute(
    `CALL ${name}(${placeholders})`,
    params
  );
  return result;  // VARIANT returned as JSON
}
```

## DCM Deployment

### 1. Create hybrid tables (first time, or when schema changes)

```bash
snow sql -f apps/{name}/pre_deploy.sql -c <connection>
```

### 2. Plan (preview procedure/task/role changes)

```bash
snow dcm plan <DB.SCHEMA.PROJECT> -c <connection>
```

### 3. Deploy (apply)

```bash
snow dcm deploy <DB.SCHEMA.PROJECT> -c <connection>
```

### 4. Post-deploy (semantic view, etc.)

```bash
snow sql -f apps/{name}/post_deploy.sql -c <connection>
```

### 5. Deploy the UI

```bash
cd apps/{name}/app/
snow app deploy
```

## DEFINE Statement Examples

### Hybrid Tables

```sql
DEFINE TABLE support.tickets
  WITH table_type = 'HYBRID'
AS (
    ticket_id VARCHAR(36) NOT NULL DEFAULT UUID_STRING(),
    short_id VARCHAR(20) NOT NULL,
    title VARCHAR(500) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'open',
    priority VARCHAR(20) NOT NULL DEFAULT 'medium',
    reporter_email VARCHAR(255) NOT NULL,
    organization_id VARCHAR(100) NOT NULL,
    created_at TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP(),
    updated_at TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP(),
    PRIMARY KEY (ticket_id),
    UNIQUE (short_id)
);
```

### Stored Procedures

```sql
DEFINE PROCEDURE support.create_ticket(
    p_title VARCHAR, p_priority VARCHAR,
    p_reporter_email VARCHAR, p_organization_id VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS $$
DECLARE
    v_ticket_id VARCHAR DEFAULT UUID_STRING();
    v_short_id VARCHAR;
    v_job_id VARCHAR DEFAULT UUID_STRING();
BEGIN
    -- Validate
    IF (:p_title IS NULL OR LENGTH(TRIM(:p_title)) = 0) THEN
        RETURN OBJECT_CONSTRUCT('success', FALSE, 'errors', ARRAY_CONSTRUCT(
            OBJECT_CONSTRUCT('type', 'validation', 'field', 'title',
                             'code', 'required', 'message', 'Title is required')));
    END IF;

    -- Generate short ID
    UPDATE support.sequences SET value = value + 1 WHERE name = 'ticket_seq';
    SELECT 'OMS-' || value::VARCHAR INTO :v_short_id
        FROM support.sequences WHERE name = 'ticket_seq';

    -- Insert
    INSERT INTO support.tickets (ticket_id, short_id, title, priority, reporter_email, organization_id)
    VALUES (:v_ticket_id, :v_short_id, :p_title, :p_priority, :p_reporter_email, :p_organization_id);

    -- Dispatch job
    INSERT INTO support.jobs (job_id, job_type, entity_name, record_id, payload)
    SELECT :v_job_id, 'triage_ticket', 'tickets', :v_ticket_id,
            OBJECT_CONSTRUCT('title', :p_title, 'priority', :p_priority);
    EXECUTE TASK support.job_runner;;

    RETURN OBJECT_CONSTRUCT('success', TRUE, 'ticket_id', :v_ticket_id, 'short_id', :v_short_id);
END;
$$;
```

### Tasks

```sql
DEFINE TASK support.job_runner
    WAREHOUSE = 'SUPPORT_WH'
    SCHEDULE = '60 MINUTE'
    STARTED
AS
    CALL support.process_pending_jobs();
```

## Error Response Format

All procs return VARIANT:
```json
{"success": true, "ticket_id": "abc-123", "short_id": "OMS-1001"}
{"success": false, "errors": [{"type": "validation", "field": "title", "code": "required", "message": "..."}]}
```

Error types: `validation` (field-level), `constraint` (multi-field), `state` (invalid transition), `permission` (access denied).

## Workflow

### Phase 1: Data Model
1. Interview user about entities and validation rules
2. Write DEFINE statements in `definitions/`
3. `snow dcm plan` to preview, `snow dcm deploy` to apply
4. Update the app skill

### Phase 2: UI Design Interview
- Pages/screens needed?
- Navigation style?
- List/detail/form layouts?
- Where does reference data appear?
- Document in `ui/DECISIONS.md`

### Phase 3: UI Implementation
- Write Next.js pages (App Router)
- Writes: call procs via `callProc('schema.proc', params)`
- Reads: direct SQL via `query('SELECT ...')`
- Reference data: direct SQL or cached lookups
- Handle proc error responses (field-level display)

### Phase 4: Deploy
- `snow dcm deploy` for Snowflake objects
- `snow app deploy` for the Next.js UI

### Phase 5: Maintain the App Skill

Every saaspocalypse app MUST have its own CoCo skill at `apps/{name}/skill/`.
Update it every time you add entities, procs, or jobs.

The app skill must document:
1. **Purpose** — what the app does, key terminology
2. **Entities** — every table, columns, meanings
3. **How to write** — proc signatures with examples (`CALL support.create_ticket(...)`)
4. **How to read** — useful SELECT queries
5. **Validation rules** — what gets rejected
6. **Jobs** — what async work exists, triggers, behavior
7. **Reference data** — external tables available
8. **Workflows** — common multi-step operations

#### When to update:
- After adding an entity → add to entities section
- After adding a proc → add signature + examples
- After adding a job type → add to jobs table
- After changing validation → update constraints

## Backend Services

Snowflake App Runtime handles the UI but authenticates all requests via Snowflake SSO.
For unauthenticated inbound traffic (webhooks from external services like Mailgun,
Stripe, GitHub, etc.), use a **Cloudflare Worker**.

The pattern:
1. External service sends webhook to your Cloudflare Worker URL
2. Worker verifies the webhook signature (HMAC, API key, etc.)
3. Worker calls a Snowflake stored proc via the Snowflake REST API or SQL API
4. Proc validates and inserts the data into hybrid tables

```
External Service → Cloudflare Worker → Snowflake SQL API → CALL proc(...)
```

Workers live in `apps/{name}/workers/` alongside the rest of the app.

## Setting Up the Git Repo as a Snowflake Stage

```bash
snow git setup APP_REPO
```

Deploy DCM from the stage:
```sql
EXECUTE DCM PROJECT support_project DEPLOY
FROM @my_db.my_schema.app_repo/branches/main/apps/support/;
```

## Stopping Points

- After DEFINE statements, ask user to review before deploying
- After UI design interview, present decisions before coding
- After `snow dcm plan`, show changeset and wait for approval
- After writing UI, confirm before `snow app deploy`

## DCM Gotchas

These constraints apply when writing DEFINE statements and stored procedures:

### Tables
- **Hybrid Tables go in `pre_deploy.sql`** — DCM cannot create hybrid tables (no supported syntax). Define them as `CREATE HYBRID TABLE IF NOT EXISTS` in `pre_deploy.sql`, which runs before `snow dcm plan`/`deploy`. This means tables are NOT managed by DCM — schema changes require manual ALTER statements or recreating tables.
- `PRIMARY KEY`, `UNIQUE`, and other constraints work in hybrid table DDL.
- Use `CREATE HYBRID TABLE IF NOT EXISTS` for idempotent reruns.
- Seed reference data with `MERGE INTO ... WHEN NOT MATCHED THEN INSERT`.
- DEFINE TABLE (regular tables) in DCM supports `PRIMARY KEY` inline — use only if hybrid is not needed.

### Stored Procedures
- **No `OBJECT_CONSTRUCT()` in VALUES clauses** — Snowflake rejects it. Always use `INSERT INTO ... SELECT ...` form:
  ```sql
  -- WRONG:
  INSERT INTO t (id, payload) VALUES (:v_id, OBJECT_CONSTRUCT('key', :val));
  -- CORRECT:
  INSERT INTO t (id, payload) SELECT :v_id, OBJECT_CONSTRUCT('key', :val);
  ```
- **`EXECUTE TASK` for immediate job dispatch** — After inserting a job, call `EXECUTE TASK` to trigger immediate processing. The scheduled task (1 hour default) is only a fallback for edge cases — normal operation relies on `EXECUTE TASK`:
  ```sql
  INSERT INTO schema.jobs (...) SELECT ...;
  EXECUTE TASK schema.job_runner;
  ```
- **Multi-line IF with OR** — DCM analyze may reject multi-line OR conditions. Keep on one line:
  ```sql
  IF ((:a IS NOT NULL AND :a != :b) OR (:c IS NOT NULL AND :c != :d)) THEN
  ```
- **Use `PARSE_JSON('{}')` not `OBJECT_CONSTRUCT()`** for empty object defaults in INSERT...SELECT.

### Tasks
- The `STARTED` keyword enables the task immediately on deploy.
- Task warehouse must exist or be defined in the same project.

## Local UI Development

For fast UI iteration, run the Next.js app locally instead of deploying to App Runtime:

```bash
cd apps/{name}/app
SNOWFLAKE_DEFAULT_CONNECTION_NAME=myconn npm run dev
```

### Setup
- Add `snowflake-sdk` and `smol-toml` to `package.json` dependencies
- Create `.env.local` with `SNOWFLAKE_DEFAULT_CONNECTION_NAME=<connection>` to select from `~/.snowflake/config.toml`
- The `lib/snowflake.ts` helper auto-detects auth: SPCS token → env vars → TOML config

### Known Issues
- Snowflake SDK returns **UPPERCASE column names** — use quoted aliases (`as "total"`) or handle both cases in code
- Next.js 15+: `searchParams` and `params` are **Promises** — must `await` them in server components
- Bind parameters (`?`) are not natively supported by the SDK in this mode — inline them via a helper that escapes strings and passes numbers raw
