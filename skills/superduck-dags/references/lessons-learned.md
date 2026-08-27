# SuperDuck DAGs: Lessons Learned

This document captures specific insights and solutions discovered while working with SuperDuck DAGs. These are historical lessons that provide context for current practices documented in the main SKILL.md file.

---

## 2026-06-05: Composite merge_on Formatting

**Problem**: DAG generation succeeded but merge operations failed silently when using comma-separated strings for composite keys.

**Root cause**: The `dag_generator.py` script has special handling for `merge_on` that expects either:
- A simple string for single keys: `"id"`
- A JSON array string for composite keys: `'["col1", "col2"]'`

Using a comma-separated string like `"col1,col2"` bypasses the generator's formatting logic and produces incorrect merge statements in Unity Catalog.

**Solution**: Always use JSON array string format for composite keys:
```json
"merge_on": "[\"clientid\", \"daystr\"]"
```

**Files affected**:
- `dc2/dags/generated/sca_phonehome_kb/sca_phonehome_kb_instance_stats.py` (corrected)
- All similar DAGs should follow this pattern

**Prevention**: Add validation to generator script to catch this pattern and error early.

**References**:
- Generator code: `utilities/dag_generator.py` lines 28-36
- Examples: `mlgpu/dags/cd_scan_response_aggregator_US.py` line 48

---

## 2026-04-14 through 2026-05-28: PostgreSQL Identifier Quoting Journey

**Problem**: Multiple SQL errors when querying PostgreSQL with mixed-case table/column names, causing "relation does not exist" errors, syntax errors, and DAG import failures.

**Root cause**: PostgreSQL requires double quotes for case-sensitive identifiers and reserved words. The team went through several incorrect attempts:

1. **Backticks (MySQL syntax)**: `` `TRIMSserver_capec` `` - Wrong database dialect
2. **Single quotes**: `'TRIMSserver_capec'` - String literal, not identifier
3. **Triple-quote with double-quoted identifiers**: Created `""""` syntax error in Python

**Solution**: Use PostgreSQL double-quoted identifiers with proper Python string quoting:

```json
// ✅ CORRECT - Double quotes for SQL identifiers
"query": "SELECT * FROM \"TRIMSserver_capec\""

// ✅ CORRECT - Quote reserved words
"query": "SELECT \"references\" FROM table"

// ✅ CORRECT - Quote mixed-case columns
"query": "SELECT \"base_CWE_confidence\" FROM table"
```

**When to quote SQL identifiers:**
- **Mixed-case table names**: `TRIMSserver_capec` → `"TRIMSserver_capec"`
- **Mixed-case column names**: `base_CWE_confidence` → `"base_CWE_confidence"`
- **Reserved SQL keywords**: `references`, `user`, `order` → `"references"`, `"user"`, `"order"`
- **Names with special characters**: Spaces, hyphens, etc.

**Python string quoting patterns:**

```json
// Single-quoted Python string with escaped double quotes (preferred)
"query": "SELECT \"Column\" FROM \"Table\""

// Or use single quotes in f-string (if template uses triple quotes differently)
"query": 'SELECT "Column" FROM "Table"'
```

**Triple-quote syntax error:**

When using Python triple-quoted f-strings with queries ending in double-quoted identifiers:

```python
# ❌ WRONG - Creates """" which Python parses as closed string + stray quote
query = f"""SELECT * FROM "TRIMSserver_capec""""

# ✅ FIX 1 - Use single quotes instead
query = f'SELECT * FROM "TRIMSserver_capec"'

# ✅ FIX 2 - Put closing triple-quote on new line
query = f"""SELECT * FROM "TRIMSserver_capec"
"""
```

**Files affected**:
- All TRIMS DAGs (8 files) - fixed in commits 2a66ad5, 7a0d11e, 866ae51, 858ad99, fd73789
- `config/trims.json` - updated all queries with proper quoting
- Template may need adjustment to handle this pattern better

**Prevention**:
- Always test queries directly against PostgreSQL before adding to config
- Use `\d` in psql to check actual case of table/column names
- Quote identifiers in config JSON: `\"IdentifierName\"`
- For MySQL sources, use backticks; for PostgreSQL, use double quotes
- Be aware of database-specific reserved words

**References**:
- Commit 2a66ad5: Initial PostgreSQL quoted identifier fix
- Commit 858ad99: Triple-quote syntax error fix with detailed explanation
- Commit cef6c2b: Reserved word `references` fix
- Commit 3742267: Mixed-case column `base_CWE_confidence` fix
- [PostgreSQL Identifiers Documentation](https://www.postgresql.org/docs/current/sql-syntax-lexical.html#SQL-SYNTAX-IDENTIFIERS)

---

## [Future Learnings]

Additional lessons will be added here as they are discovered. Suggested format:

```markdown
### YYYY-MM-DD: Brief Title

**Problem**: What went wrong or what was unclear

**Root cause**: Why it happened

**Solution**: How to fix or prevent it

**Files affected**: Which DAGs or configs

**Prevention**: How to avoid in the future
```
