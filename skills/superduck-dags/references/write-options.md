# Write Options Detailed Guide

Deep dive into each `write_option` value with SQL behavior, use cases, and examples.

---

## Overview

The `write_option` parameter controls how the SuperDuck pipeline merges incoming data with existing data in Unity Catalog Delta tables. This guide explains each option in detail with before/after examples.

**Quick comparison:**

| Option | Updates? | Deletes? | History? | Performance | Use Case |
|--------|----------|----------|----------|-------------|----------|
| scd | Yes | No | Full | Slower | Dimensional tables |
| overwrite | Yes | Yes | None | Medium | Daily incremental |
| preserve | Yes | Soft | Partial | Medium | Soft-delete pattern |
| full_overwrite | N/A | All | None | Fast | Weekly full refresh |
| append | No | No | None | Fastest | Immutable logs |

---

## scd (Slowly Changing Dimensions Type 2)

### What It Does

Implements SCD Type 2: maintains full history of all changes to records.

**SQL behavior:**
1. Match incoming records to existing records using `merge_on` column(s)
2. For matches:
   - Set old record's `is_current = false`, `valid_to = now()`
   - Insert new record with `is_current = true`, `valid_from = now()`
3. For new records (no match):
   - Insert with `is_current = true`, `valid_from = now()`

**Additional columns added:**
- `is_current` - Boolean flag indicating current version
- `valid_from` - Timestamp when this version became current
- `valid_to` - Timestamp when this version stopped being current (NULL for current)

### When to Use

- Dimensional data (customers, products, accounts, employees)
- Need to answer "what did this look like on date X?"
- Regulatory/audit requirements for full history
- Data warehouse fact tables referencing historical dimensions

### Requirements

- `merge_on` is **required**
- merge_on column(s) must uniquely identify a logical record
- Source query can return full current state or just changes

### Example Scenario

**Use case**: Customer dimension table tracking address changes

**Initial state (empty table):**
```
| customer_id | name | city | is_current | valid_from | valid_to |
|-------------|------|------|------------|------------|----------|
```

**Day 1 - First load:**
```sql
-- Incoming data
customer_id | name  | city
1          | Alice | NYC
2          | Bob   | LA
```

**Result after Day 1:**
```
| customer_id | name  | city | is_current | valid_from      | valid_to |
|-------------|-------|------|------------|-----------------|----------|
| 1           | Alice | NYC  | true       | 2024-01-01      | NULL     |
| 2           | Bob   | LA   | true       | 2024-01-01      | NULL     |
```

**Day 2 - Alice moves to Boston:**
```sql
-- Incoming data
customer_id | name  | city
1          | Alice | Boston
2          | Bob   | LA
```

**Result after Day 2:**
```
| customer_id | name  | city   | is_current | valid_from      | valid_to        |
|-------------|-------|--------|------------|-----------------|-----------------|
| 1           | Alice | NYC    | false      | 2024-01-01      | 2024-01-02      |
| 1           | Alice | Boston | true       | 2024-01-02      | NULL            |
| 2           | Bob   | LA     | true       | 2024-01-01      | NULL            |
```

Now you can query "where did Alice live on 2024-01-01?" → NYC

### Configuration Example

```json
{
    "dataset_name": "customers",
    "dataset_product": "crm",
    "write_option": "scd",
    "merge_on": "customer_id",
    "query": "SELECT customer_id, name, address, city, state FROM crm.customers WHERE active = true"
}
```

### Performance Considerations

- **Slowest write option** due to multiple operations per record
- Table grows over time (one row per change)
- Queries need `WHERE is_current = true` to get current state
- Consider partitioning by date for large tables
- Indexing `merge_on` columns improves performance significantly

---

## overwrite (Replace Matching Records)

### What It Does

Deletes existing records that match on `merge_on`, then inserts new records. Unmatched records remain unchanged.

**SQL behavior:**
1. Match incoming records to existing records using `merge_on`
2. Delete all matched records
3. Insert all incoming records
4. Unmatched existing records stay in the table

### When to Use

- Daily/hourly incremental updates
- Only current state matters (no history needed)
- Date-windowed data where each window is complete
- Corrections/updates to recent data

### Requirements

- `merge_on` is **required**
- merge_on should be unique in incoming data (or last record wins)
- Be careful with overlapping keys across runs

### Example Scenario

**Use case**: Daily telemetry data, may receive corrections for past days

**Initial state:**
```
| client_id | date       | scan_count |
|-----------|------------|------------|
| A         | 2024-01-01 | 10         |
| A         | 2024-01-02 | 15         |
| B         | 2024-01-01 | 5          |
```

**Day 3 - New data plus correction:**
```sql
-- Incoming data
client_id | date       | scan_count
A         | 2024-01-02 | 16          -- Correction: was 15, now 16
A         | 2024-01-03 | 20          -- New day
B         | 2024-01-03 | 8           -- New day
```

**Config:**
```json
"merge_on": "[\"client_id\", \"date\"]"
```

**Result after Day 3:**
```
| client_id | date       | scan_count |
|-----------|------------|------------|
| A         | 2024-01-01 | 10         |  ← Unchanged (no match)
| A         | 2024-01-02 | 16         |  ← Updated (deleted old, inserted new)
| A         | 2024-01-03 | 20         |  ← New
| B         | 2024-01-01 | 5          |  ← Unchanged (no match)
| B         | 2024-01-03 | 8          |  ← New
```

Note: Client A's 2024-01-01 record remains because it wasn't in the incoming data.

### Configuration Example

```json
{
    "dataset_name": "instance_stats",
    "dataset_product": "sca_phonehome_kb",
    "write_option": "overwrite",
    "merge_on": "[\"clientid\", \"daystr\"]",
    "query": "SELECT clientid, daystr, scan_count, project_count FROM stats WHERE daystr >= '{dag_start_date}' AND daystr < '{dag_end_date}'"
}
```

### Performance Considerations

- **Medium performance** - delete + insert per matched record
- Table size stays roughly constant (unless data volume grows)
- Good for incremental patterns with date windowing
- Index `merge_on` columns for faster matching

### Common Patterns

**Daily incremental with 7-day lookback window:**
```sql
-- Load last 7 days each run to catch late corrections
WHERE date >= CURRENT_DATE - INTERVAL '7 days'
  AND date < CURRENT_DATE
```

**Hourly incremental:**
```sql
WHERE timestamp >= '{dag_start_date}'
  AND timestamp < '{dag_end_date}'
```

---

## preserve (Update and Archive Unmatched)

### What It Does

Similar to overwrite, but marks unmatched existing records as not current instead of leaving them alone.

**SQL behavior:**
1. Match incoming records to existing records using `merge_on`
2. For matched records: update to new values, set `is_current = true`
3. For new records: insert with `is_current = true`
4. For existing records with no match: set `is_current = false`

**Additional columns added:**
- `is_current` - Boolean flag indicating if record is current

### When to Use

- Soft-delete pattern (want to know when records disappeared)
- Active/inactive entity lists
- Need to track "deleted" records without removing them
- Source provides full current snapshot each run

### Requirements

- `merge_on` is **required**
- Source query should return complete current state (not incremental)

### Example Scenario

**Use case**: Active user list from external system

**Initial state:**
```
| user_id | name  | status | is_current |
|---------|-------|--------|------------|
| 1       | Alice | active | true       |
| 2       | Bob   | active | true       |
| 3       | Carol | active | true       |
```

**Day 2 - Bob deactivated, Dave added:**
```sql
-- Incoming data (only active users)
user_id | name  | status
1       | Alice | active
3       | Carol | active
4       | Dave  | active
```

**Result after Day 2:**
```
| user_id | name  | status | is_current |
|---------|-------|--------|------------|
| 1       | Alice | active | true       |  ← Matched, updated
| 2       | Bob   | active | false      |  ← No match, archived
| 3       | Carol | active | true       |  ← Matched, updated
| 4       | Dave  | active | true       |  ← New, inserted
```

Bob's record remains but marked `is_current = false`, so we know he was active but no longer is.

### Configuration Example

```json
{
    "dataset_name": "active_users",
    "dataset_product": "iam",
    "write_option": "preserve",
    "merge_on": "user_id",
    "query": "SELECT user_id, name, email, department FROM iam.users WHERE status = 'active'"
}
```

### Performance Considerations

- **Medium performance** - similar to overwrite
- Table grows over time as records are archived
- Queries typically need `WHERE is_current = true`
- Consider purging old archived records periodically

### Difference from SCD

- **SCD**: Keeps full history of every change
- **Preserve**: Keeps latest version + marks deletions

If a record changes value:
- SCD: Creates new row for each change
- Preserve: Updates the existing row

---

## full_overwrite (Complete Table Replacement)

### What It Does

Deletes the entire table and loads fresh data. Complete replacement.

**SQL behavior:**
1. Truncate/delete all records in target table
2. Insert all incoming records

### When to Use

- Weekly/monthly full refresh from external system
- Source provides complete snapshot
- Historical comparison not needed
- Small to medium tables where full reload is acceptable
- Simplest pattern when merge logic not needed

### Requirements

- `merge_on` is **NOT used**
- Source query should return complete data set

### Example Scenario

**Use case**: Weekly vendor data dump

**Existing data (from last week):**
```
| product_id | name      | price |
|------------|-----------|-------|
| 1          | Widget A  | 10.00 |
| 2          | Widget B  | 15.00 |
| 3          | Widget C  | 20.00 |
```

**New week - vendor sends updated file:**
```sql
-- Incoming data (complete catalog)
product_id | name      | price
1          | Widget A  | 11.00  -- Price changed
2          | Widget B  | 15.00  -- Unchanged
4          | Widget D  | 25.00  -- New product
-- Note: Widget C discontinued (not in file)
```

**Result after load:**
```
| product_id | name      | price |
|------------|-----------|-------|
| 1          | Widget A  | 11.00 |
| 2          | Widget B  | 15.00 |
| 4          | Widget D  | 25.00 |
```

Widget C is gone. Previous data is completely replaced.

### Configuration Example

```json
{
    "dataset_name": "vendor_catalog",
    "dataset_product": "external_feeds",
    "write_option": "full_overwrite",
    "schedule": 10080,
    "catchup": false,
    "query": "SELECT product_id, name, description, price, category FROM vendor.products"
}
```

### Performance Considerations

- **Fast for small/medium tables** (single delete + bulk insert)
- **Slow for large tables** (minutes to hours for billions of rows)
- No merge logic overhead
- Table size matches source data exactly
- Consider partitioning for very large tables

### When NOT to Use

- Large tables (>100GB) where incremental is feasible
- Data needs to be available 24/7 (table is empty during load)
- Need to track changes over time
- Incremental updates are available from source

---

## append (Insert Only, Never Update)

### What It Does

Simply inserts all incoming records. Never updates or deletes anything.

**SQL behavior:**
1. Insert all incoming records
2. That's it

### When to Use

- Immutable event logs
- Append-only data (clickstream, application logs, transactions)
- Each DAG run produces new unique records
- No updates ever needed
- Deduplication happens downstream (or not at all)

### Requirements

- `merge_on` is **NOT used**
- Source should not produce duplicate records (or duplicates are acceptable)

### Example Scenario

**Use case**: Application event logs

**Existing data:**
```
| event_id | timestamp           | user_id | action |
|----------|---------------------|---------|--------|
| 1        | 2024-01-01 10:00:00 | 100     | login  |
| 2        | 2024-01-01 10:05:00 | 100     | click  |
| 3        | 2024-01-01 10:10:00 | 200     | login  |
```

**New hour - more events:**
```sql
-- Incoming data
event_id | timestamp           | user_id | action
4        | 2024-01-01 11:00:00 | 100     | click
5        | 2024-01-01 11:05:00 | 300     | login
6        | 2024-01-01 11:10:00 | 100     | logout
```

**Result after load:**
```
| event_id | timestamp           | user_id | action |
|----------|---------------------|---------|--------|
| 1        | 2024-01-01 10:00:00 | 100     | login  |
| 2        | 2024-01-01 10:05:00 | 100     | click  |
| 3        | 2024-01-01 10:10:00 | 200     | login  |
| 4        | 2024-01-01 11:00:00 | 100     | click  |
| 5        | 2024-01-01 11:05:00 | 300     | login  |
| 6        | 2024-01-01 11:10:00 | 100     | logout |
```

All old records remain, new records appended.

### Configuration Example

```json
{
    "dataset_name": "application_events",
    "dataset_product": "logging",
    "write_option": "append",
    "schedule": 60,
    "query": "SELECT event_id, timestamp, user_id, action, details FROM events WHERE timestamp >= '{dag_start_date}' AND timestamp < '{dag_end_date}'"
}
```

### Performance Considerations

- **Fastest write option** (single bulk insert)
- Table grows indefinitely
- Must implement retention/purge strategy separately
- Partitioning by date highly recommended
- Perfect for streaming/event data pipelines

### Deduplication

If you need deduplication:
- Handle it downstream (in transformation layer)
- Or ensure source query filters duplicates
- Or accept duplicates and use `DISTINCT` in queries

**Example with dedup in source:**
```sql
SELECT DISTINCT ON (event_id)
    event_id, timestamp, user_id, action
FROM events
WHERE timestamp >= '{dag_start_date}'
ORDER BY event_id, timestamp DESC
```

---

## Choosing Between Similar Options

### SCD vs Preserve

**Use SCD if:**
- Need full history of every change
- Multiple changes per day are common
- Must answer "what was the value at timestamp X?"

**Use preserve if:**
- Only care about current + deletions
- Changes are infrequent
- Want simpler queries (just filter `is_current`)

### Overwrite vs Full_Overwrite

**Use overwrite if:**
- Incremental loads (date windowing)
- Want to keep old data outside the window
- Source provides delta/changes

**Use full_overwrite if:**
- Source provides complete snapshot
- Historical data not needed
- Simpler logic preferred
- Table size manageable for full reload

### Overwrite vs Append

**Use overwrite if:**
- Records can change after creation
- Need to correct/update past data
- Source may send duplicates

**Use append if:**
- Records are immutable
- Each run produces new unique records
- Duplicates impossible or acceptable

---

## Migration: Changing Write Options

Can you change write_option later? Sometimes, with considerations.

### Safe Migrations

**From full_overwrite to overwrite:**
- Add `merge_on` column(s)
- Adjust query to be incremental
- Safe: just changes from full to partial updates

**From append to overwrite:**
- Add `merge_on` column(s)
- Now handles duplicates/updates
- Safe: table data unchanged, behavior more sophisticated

### Risky Migrations

**From scd/preserve to overwrite/full_overwrite:**
- Loses `is_current`, `valid_from`, `valid_to` columns
- Historical data becomes current data
- **Action**: Export historical data first, or accept data loss

**From overwrite to scd:**
- Need to add tracking columns
- Can't reconstruct lost history
- **Action**: Start fresh or migrate carefully with transformation

**From any to append:**
- Will duplicate data on next run
- **Action**: Truncate table first

### Recommended Approach

1. Backup the table
2. Test new write_option in dev environment
3. Consider creating a new table with `_v2` suffix
4. Validate data quality
5. Cutover when confident

---

## Performance Summary

From fastest to slowest (for typical workloads):

1. **append** - Simple insert, no logic
2. **full_overwrite** - One delete + bulk insert (small tables)
3. **overwrite** - Delete matching + insert per record
4. **preserve** - Similar to overwrite, plus marking archived
5. **scd** - Multiple updates + inserts per changed record

**For large tables:**
- full_overwrite can be slower than incremental options
- Proper indexing on merge_on is critical
- Partitioning helps all options

**General optimization tips:**
- Index merge_on columns in target table
- Partition by date where applicable
- Limit incoming data volume (date windowing)
- Increase max_active_runs if backfilling and source can handle it
