# Migrate — Database & Schema Migration Workflow

Design and ship a migration (schema change, data backfill, or both) that is reversible, tested on production-like data, and safe to deploy without downtime.

The rule: **no migration ships until `up` + `down` run cleanly on a copy of production data, and the app code works both before AND after the migration.**

## Triggers
Use when: "migration", "db migration", "schema change", "add column", "drop column", "rename table", "backfill", "migrate data", "alter table"
Proactively suggest when: the user asks to change a database schema, add/remove/rename a field, or backfill data.

---

## Step 0: Detect Migration Framework

```bash
# Node / TypeScript
ls prisma/schema.prisma 2>/dev/null && echo "prisma"
ls drizzle.config.* 2>/dev/null && echo "drizzle"
ls knexfile.* 2>/dev/null && echo "knex"
ls -d migrations/ src/migrations/ 2>/dev/null

# Python
ls alembic.ini 2>/dev/null && echo "alembic"
ls manage.py 2>/dev/null && grep -q django && echo "django-migrations"

# Go
grep -rE "goose|golang-migrate|atlas" go.mod 2>/dev/null

# Ruby
ls db/migrate 2>/dev/null && echo "rails-active-record"

# Raw SQL in a migrations dir
ls sql/migrations/ migrations/*.sql 2>/dev/null
```

Answer:
- **Framework** — what's generating/running migrations?
- **DB engine** — Postgres? MySQL? SQLite? Each has different DDL semantics.
- **Online DDL tools?** — pt-online-schema-change, gh-ost, pg_repack? Needed for large tables.
- **Replication?** — changes that lock tables can break replicas.
- **Downtime tolerance** — zero-downtime, or maintenance window?

---

## Step 1: Classify the Migration

### Additive (safe)
- Add a nullable column
- Add a new table
- Add a new index (with `CONCURRENTLY` on Postgres)
- Add a new enum value

### Subtractive (dangerous)
- Drop a column
- Drop a table
- Drop an index
- Remove an enum value (not directly supported in Postgres)

### Mutative (most dangerous)
- Rename a column or table
- Change a column type (int → bigint, varchar → text)
- Add a NOT NULL constraint to an existing column
- Change a foreign key

### Data backfill
- Populating a new column from existing data
- Splitting/combining columns
- Normalizing / denormalizing

**Rule:** Subtractive and mutative migrations require a **multi-step deploy** (expand → migrate → contract). Never do them in one release.

---

## Step 2: Design the Migration

### Zero-downtime pattern: Expand → Migrate → Contract

For any non-additive change:

1. **Expand** — add the new shape (new column / table / index) alongside the old one. App code writes to BOTH.
2. **Backfill** — populate the new shape from the old data (in small batches, not a single big `UPDATE`).
3. **Migrate reads** — flip the app to read from the new shape.
4. **Remove writes to old** — app writes only to new shape.
5. **Contract** — drop the old shape after all nodes have fully rolled out + stabilized.

Each step is a separate deploy. Never combine steps 1 and 5.

### Write BOTH directions

```sql
-- up.sql
ALTER TABLE orders ADD COLUMN status_v2 VARCHAR(32);
CREATE INDEX CONCURRENTLY idx_orders_status_v2 ON orders(status_v2);

-- down.sql
DROP INDEX CONCURRENTLY IF EXISTS idx_orders_status_v2;
ALTER TABLE orders DROP COLUMN IF EXISTS status_v2;
```

If you can't write a `down` that works, the migration isn't rollback-safe. Redesign.

---

## Step 3: Test on Production-Like Data

Empty-schema tests don't catch:
- Locking behavior on a 50M-row table
- Backfill query taking 4 hours
- Index creation blocking reads
- Foreign key violation during a rename

Minimum test matrix:
- [ ] `up` on empty DB
- [ ] `down` on empty DB (must work)
- [ ] `up` on a dump of production (at least a recent snapshot)
- [ ] `down` on the post-`up` state (data preserved as much as possible)
- [ ] Backfill query timed — must finish in maintenance window OR be chunked

```bash
# Typical pattern
./scripts/restore-prod-snapshot.sh stage-db
npm run migrate:up --env=stage
./scripts/run-smoke-tests.sh stage
npm run migrate:down --env=stage
```

---

## Step 4: Code Compatibility

The migration ships in a deploy. Before/during/after that deploy, the app must keep working.

Checklist:
- [ ] Code version N works with schema version N (obvious)
- [ ] Code version N works with schema version N-1 (during rolling deploy, old pods see old schema)
- [ ] Code version N-1 works with schema version N (during rollout of schema before code, or if code rolls back)

If any of those fails, you have a deploy ordering problem. Usually solved by:
- Making the new column nullable first (code can handle NULL)
- Double-writing old + new fields from code for one release

---

## Step 5: Backfill Safely

Big `UPDATE` statements hold locks forever and block replication. Use batches.

```sql
-- Bad: locks table, blocks replicas for hours
UPDATE orders SET status_v2 = LOWER(status) WHERE status_v2 IS NULL;

-- Good: chunked, throttled
DO $$
DECLARE batch_size INT := 1000;
DECLARE rows_updated INT;
BEGIN
  LOOP
    UPDATE orders
    SET status_v2 = LOWER(status)
    WHERE id IN (
      SELECT id FROM orders WHERE status_v2 IS NULL LIMIT batch_size
    );
    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    EXIT WHEN rows_updated = 0;
    PERFORM pg_sleep(0.1);  -- throttle
  END LOOP;
END $$;
```

Or move the backfill to a background job outside the migration. Migrations should be fast; backfills can take hours.

---

## Step 6: Pre-Flight Checklist

- [ ] Migration framework's `validate` / `check` command passes
- [ ] `up` + `down` tested on prod-snapshot
- [ ] Backfill tested on prod-snapshot, timing recorded
- [ ] No `DROP COLUMN` / `DROP TABLE` in same release as code that stopped using it (wait one release)
- [ ] Indexes created with `CONCURRENTLY` (Postgres) or equivalent
- [ ] Foreign keys added NOT VALID first, then VALIDATE in a separate step
- [ ] Enum additions only (no removals/renames — use new enum values)
- [ ] Staging migration run + smoke tested BEFORE merging to main

---

## Step 7: Deploy

1. Merge the migration + code change together.
2. CI deploys to staging — migration runs first, then code, then smoke tests.
3. Staging healthy for 30+ minutes → promote to production.
4. In production, deploy order:
   - Run migration FIRST (if additive)
   - OR deploy dual-write code FIRST, then schema (for rename/subtract)
5. Watch slow query log for 24 hours — new column access patterns may need a new index.

---

## Step 8: Rollback Plan

For every migration, the `/rollback` plan must be written BEFORE deploy:

```
IF <metric> regresses:
  - Code rollback: revert commit <sha>, redeploy
  - Schema rollback: NOT NEEDED (additive only)  
        OR: run `migrate:down` (safe, tested)
        OR: run compensating migration `<sha>.sql`
```

If the schema rollback isn't safe (data loss), document it: "This migration is forward-only after backfill starts. Revert by writing a new migration."

---

## Hard rules

- **Never drop a column in the same release as the code that stopped using it.** Wait one release.
- **Never `ALTER TABLE` on a big table without online DDL or a maintenance window.**
- **Never `UPDATE` the whole table in a migration.** Batch it, or move it to a background job.
- **Never skip the `down` migration.** Even if forward-only in practice, the `down` documents intent.
- **Never merge a migration that hasn't run against a prod-like DB.**

---

## Completion

- **DONE — SHIPPED** — migration applied, app healthy, backfill complete
- **DONE — STAGED** — migration merged, awaiting production deploy window
- **BLOCKED** — prod-snapshot test failed, redesign needed


---

## CRITICAL LESSON LEARNED — Build Verification

Migrations that pass on empty schemas crash on production data. ALWAYS test against a production snapshot before merging.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

"Migration runs" ≠ "migration is correct." Write a test that queries the migrated data and proves the new shape is right. Check row counts before + after.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

Migrations must land on the same branch as the code that uses them. Never ship a schema change on develop that matches code on a feature branch — or vice versa.
