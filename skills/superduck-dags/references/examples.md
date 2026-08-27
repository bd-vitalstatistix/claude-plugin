# Real-World Examples

Complete examples from the ds-dags-02 repository demonstrating different patterns and use cases.

---

## Example 1: Daily Incremental with Composite Key

**Source**: `config/sca_phonehome_kb.json`  
**Use case**: Daily SCA instance telemetry data with late-arriving corrections

### Config File

```json
{
    "defaults": {
        "schedule": 1440,
        "storage_account": "dataaggbronze",
        "storage_container": "landing-zone",
        "schema_repo": "git@github.com:bd-vitalstatistix/superduck-lake-schema-tracking.git",
        "dataset_category": "product",
        "datalake_class": "datum",
        "start_year": 2018,
        "start_month": 12,
        "start_day": 20,
        "catchup": true,
        "max_active_runs": 3,
        "write_option": "merge",
        "merge_on": "[\"clientid\", \"daystr\"]",
        "column_classification": {},
        "column_description": {},
        "folder": "dc2"
    },
    "dags": [
        {
            "dataset_name": "instance_stats",
            "dataset_product": "sca_phonehome_kb",
            "data_source": "sca_phonehome_kb",
            "conn_id": "kb_analytics_blackduck_ro",
            "notes": "Daily SCA instance telemetry from feedback.blackduck_stats_new. One row per (clientid, daystr). clientid is a SHA-256 hash of the registration ID — no reverse-mapping to customer identity available in this DB. Date-windowed by daystr for incremental backfill. merge_on=(clientid,daystr) handles late-arriving corrections.",
            "dag_name": "sca_phonehome_kb_instance_stats",
            "query": "SELECT clientid, daystr, timestamp, blackduckversion, scan_count, project_count_total, version_count_total, codelocationcount, usercount, custom_project_limit, custom_field_count, attachment_count, managed_codebase_bytes_new, codebase_managed_lines_of_code, code_location_bytes_limit, latest_scan_created_at, code_location_total_count, code_location_mapped_count, code_location_unmapped_count, number_of_fs_scans, number_of_bdio_scans, number_of_binary_scans, number_of_snippet_scans, number_of_component_ignore_adjustments, number_of_snippet_ignore_adjustments, number_of_file_match_adjustments, number_of_component_purpose_adjustments, number_of_component_attribution_report_adjustments, number_of_component_match_adjustments, number_of_component_license_text_adjustments, number_of_file_usage_adjustments, number_of_snippet_match_adjustments, number_of_component_manual_adjustments, number_of_snippet_review_adjustments, number_of_component_modification_adjustments, number_of_component_review_adjustments, number_of_component_attribution_statement_adjustments, number_of_component_license_adjustments, number_of_component_usage_adjustments FROM feedback.blackduck_stats_new WHERE daystr >= '{dag_start_date}' AND daystr < '{dag_end_date}'",
            "start_year": 2018,
            "start_month": 12,
            "start_day": 20
        }
    ]
}
```

### Key Design Decisions

**Composite merge key**: `["clientid", "daystr"]`
- Unique per client per day
- Handles corrections to previous days
- JSON array string format required for generator

**Date windowing**: `WHERE daystr >= '{dag_start_date}' AND daystr < '{dag_end_date}'`
- Processes one day per DAG run
- Enables incremental backfill
- Efficient for large datasets

**Long backfill**: Start date 2018-12-20
- `catchup: true` enables backfill
- `max_active_runs: 3` limits parallelism
- Will process ~2800 days on first deployment

**Write strategy**: `overwrite`
- Only current values needed
- Allows corrections to past days
- No history tracking (saves storage)

**Security**: No PII exposure
- `clientid` is SHA-256 hash
- No reverse-mapping to customer identity
- All columns can use default "internal" classification

---

## Example 2: Multi-Table Product with Full Refresh

**Source**: `config/trims.json` (hypothetical example based on repo structure)  
**Use case**: Weekly full refresh of security reference data

### Config File

```json
{
    "defaults": {
        "schedule": 10080,
        "storage_account": "dataaggbronze",
        "storage_container": "landing-zone",
        "schema_repo": "git@github.com:bd-vitalstatistix/superduck-lake-schema-tracking.git",
        "dataset_category": "product",
        "datalake_class": "datum",
        "dataset_product": "trims",
        "data_source": "trims",
        "conn_id": "trims_readonly",
        "start_year": 2024,
        "start_month": 1,
        "start_day": 1,
        "catchup": false,
        "max_active_runs": 1,
        "write_option": "full_overwrite",
        "column_classification": {},
        "column_description": {},
        "folder": "dc2"
    },
    "dags": [
        {
            "dataset_name": "capec",
            "dag_name": "trims_capec",
            "notes": "CAPEC (Common Attack Pattern Enumeration and Classification) data. Weekly full refresh from TRIMS database. No history needed.",
            "query": "SELECT id, name, description, status, abstraction, typical_severity FROM capec.patterns WHERE status = 'Current'"
        },
        {
            "dataset_name": "capec_version",
            "dag_name": "trims_capec_version",
            "notes": "CAPEC version metadata. Tracks which version of CAPEC taxonomy is loaded.",
            "query": "SELECT version, release_date, description FROM capec.version_info ORDER BY version DESC LIMIT 1"
        },
        {
            "dataset_name": "cwe",
            "dag_name": "trims_cwe",
            "notes": "CWE (Common Weakness Enumeration) data. Weekly full refresh.",
            "query": "SELECT id, name, description, status, weakness_abstraction FROM cwe.weaknesses WHERE status = 'Current'"
        }
    ]
}
```

### Key Design Decisions

**Multi-DAG product**: All TRIMS tables in one config
- Related data from same source
- Same schedule and write strategy
- Easier to maintain together

**Full overwrite**: No merge_on needed
- Complete snapshot each week
- Old data completely replaced
- Simpler logic, adequate for reference data

**No backfill**: `catchup: false`
- Reference data doesn't need historical backfill
- Start from first deployment date forward
- Saves processing time

**Weekly schedule**: 10080 minutes (7 days)
- Reference data updates infrequently
- Weekly refresh adequate
- Reduces processing load

**Single active run**: `max_active_runs: 1`
- Reference data should be processed sequentially
- Prevents conflicts during refresh
- Order doesn't matter much, but safer

---

## Example 3: SCD with Composite Key

**Source**: `mlgpu/dags/cd_scan_response_aggregator_US.py` pattern  
**Use case**: Scan response data with full history tracking

### Config File

```json
{
    "defaults": {
        "schedule": 1440,
        "storage_account": "dataaggbronze",
        "storage_container": "landing-zone",
        "schema_repo": "git@github.com:bd-vitalstatistix/superduck-lake-schema-tracking.git",
        "dataset_category": "product",
        "datalake_class": "datum",
        "start_year": 2024,
        "start_month": 1,
        "start_day": 1,
        "catchup": true,
        "max_active_runs": 3,
        "column_classification": {},
        "column_description": {},
        "folder": "mlgpu"
    },
    "dags": [
        {
            "dataset_name": "scan_response_aggregator_US",
            "dataset_product": "continuous_dynamic",
            "data_source": "sentinel_results_read-only_replica_SC",
            "conn_id": "sentinel_results_SC",
            "write_option": "scd",
            "merge_on": "[\"lid\", \"slot_id\", \"instance_id\"]",
            "dag_name": "cd_scan_response_aggregator_US",
            "notes": "Multi-database scan response aggregator. Tracks changes to scan results over time. Composite key (lid, slot_id, instance_id) uniquely identifies a scan response. SCD maintains full history for audit and analysis.",
            "query": "SELECT lid, slot_id, instance_id, scan_data, status, timestamp FROM scan_responses WHERE timestamp >= '{dag_start_date}' AND timestamp < '{dag_end_date}'",
            "column_classification": {
                "lid": "internal",
                "slot_id": "internal",
                "scan_data": "confidential"
            },
            "column_description": {
                "lid": "Location identifier",
                "slot_id": "Slot identifier extracted from table name",
                "instance_id": "Instance identifier for scan",
                "scan_data": "Detailed scan results (may contain sensitive findings)",
                "status": "Current status of scan",
                "timestamp": "Scan completion timestamp"
            }
        }
    ]
}
```

### Key Design Decisions

**Three-column merge key**: `["lid", "slot_id", "instance_id"]`
- Natural composite key for scan responses
- Uniquely identifies each scan
- Must use JSON array string format

**SCD Type 2**: Full history tracking
- Audit requirements for scan results
- Need to know "what were the findings on date X?"
- Compliance and forensics use cases

**Column classification**: Mixed security levels
- `scan_data` marked confidential (contains vulnerability details)
- Location/identifiers marked internal
- Drives downstream access controls

**Column descriptions**: Comprehensive documentation
- Helps analysts understand data
- Populates data catalog
- Especially important for aggregated/derived fields

**Daily incremental**: Date windowing by timestamp
- Processes yesterday's scans each day
- Efficient for large scan volumes
- Backfill enabled for historical data

---

## Example 4: Append-Only Event Logs

**Use case**: Application event logs, immutable records

### Config File

```json
{
    "defaults": {
        "schedule": 60,
        "storage_account": "dataaggbronze",
        "storage_container": "landing-zone",
        "schema_repo": "git@github.com:bd-vitalstatistix/superduck-lake-schema-tracking.git",
        "dataset_category": "product",
        "datalake_class": "datum",
        "start_year": 2024,
        "start_month": 6,
        "start_day": 1,
        "catchup": false,
        "max_active_runs": 5,
        "write_option": "append",
        "column_classification": {},
        "column_description": {},
        "folder": "mlgpu"
    },
    "dags": [
        {
            "dataset_name": "application_events",
            "dataset_product": "logging",
            "data_source": "app_db_readonly",
            "conn_id": "app_db_ro",
            "dag_name": "logging_application_events",
            "notes": "Application event logs. Immutable append-only data. Each run adds new events from the last hour. No updates or deletions ever needed. Deduplication handled downstream if needed.",
            "query": "SELECT event_id, timestamp, user_id, session_id, event_type, event_data, ip_address, user_agent FROM events.application_log WHERE timestamp >= '{dag_start_date}' AND timestamp < '{dag_end_date}' ORDER BY timestamp",
            "column_classification": {
                "ip_address": "confidential",
                "event_data": "internal"
            },
            "column_description": {
                "event_id": "Unique event identifier (UUID)",
                "timestamp": "Event occurrence time (UTC)",
                "user_id": "Authenticated user ID (NULL for anonymous)",
                "session_id": "Session identifier",
                "event_type": "Event classification (login, click, error, etc)",
                "event_data": "JSON payload with event details",
                "ip_address": "Client IP address",
                "user_agent": "Client browser/app user agent string"
            }
        }
    ]
}
```

### Key Design Decisions

**Append-only**: No merge_on
- Events never change after creation
- Each DAG run adds new records
- Simplest and fastest write option

**Hourly schedule**: High frequency
- Near real-time event ingestion
- Reduces lag for monitoring dashboards
- Small batches process quickly

**No backfill**: `catchup: false`
- Event logs don't need backfill
- Start fresh from deployment date
- Historical events may be in different system

**Higher parallelism**: `max_active_runs: 5`
- Hourly runs can overlap during catch-up
- Source database can handle parallel queries
- Faster recovery if DAG is paused

**IP address confidential**: Privacy considerations
- PII classification for GDPR/privacy compliance
- Downstream masking/anonymization required
- Access controls enforced

**Ordered query**: `ORDER BY timestamp`
- Ensures consistent ordering in output
- Helpful for time-series analysis
- Small performance cost acceptable

---

## Example 5: Polaris Multi-Region

**Source**: `config/polaris.json` pattern  
**Use case**: Same data from multiple regional databases

### Config File

```json
{
    "defaults": {
        "schedule": 1440,
        "storage_account": "dataaggbronze",
        "storage_container": "landing-zone",
        "schema_repo": "git@github.com:bd-vitalstatistix/superduck-lake-schema-tracking.git",
        "dataset_category": "product",
        "datalake_class": "datum",
        "dataset_product": "polaris",
        "start_year": 2024,
        "start_month": 1,
        "start_day": 1,
        "catchup": true,
        "max_active_runs": 3,
        "write_option": "overwrite",
        "column_classification": {},
        "column_description": {},
        "folder": "mlgpu"
    },
    "dags": [
        {
            "dataset_name": "issue_category_KSA",
            "data_source": "gcp-sig-datainsight-shared-prd",
            "conn_id": "gcp-sig-datainsight-shared-prd",
            "merge_on": "[\"weakness_id\", \"category_id\", \"version\"]",
            "dag_name": "Polaris_issue_category_KSA",
            "notes": "Polaris issue category data from KSA (Kingdom of Saudi Arabia) region. Maps weaknesses to categories with versioning.",
            "query": "SELECT weakness_id, category_id, version, category_name, severity FROM polaris_ksa.issue_categories WHERE updated_at >= '{dag_start_date}' AND updated_at < '{dag_end_date}'"
        },
        {
            "dataset_name": "issue_category_PRD",
            "data_source": "gcp-sig-datainsight-shared-prd",
            "conn_id": "gcp-sig-datainsight-shared-prd",
            "merge_on": "[\"weakness_id\", \"category_id\", \"version\"]",
            "dag_name": "Polaris_issue_category_PRD",
            "notes": "Polaris issue category data from PRD (Production US) region. Same structure as KSA but different regional data.",
            "query": "SELECT weakness_id, category_id, version, category_name, severity FROM polaris_prd.issue_categories WHERE updated_at >= '{dag_start_date}' AND updated_at < '{dag_end_date}'"
        },
        {
            "dataset_name": "issue_category_PEU",
            "data_source": "gcp-sig-datainsight-shared-prd",
            "conn_id": "gcp-sig-datainsight-shared-prd",
            "merge_on": "[\"weakness_id\", \"category_id\", \"version\"]",
            "dag_name": "Polaris_issue_category_PEU",
            "notes": "Polaris issue category data from PEU (Production Europe) region. Same structure as other regions.",
            "query": "SELECT weakness_id, category_id, version, category_name, severity FROM polaris_peu.issue_categories WHERE updated_at >= '{dag_start_date}' AND updated_at < '{dag_end_date}'"
        }
    ]
}
```

### Key Design Decisions

**Multi-region pattern**: Same table structure, different sources
- Each region gets its own DAG and Unity table
- `dataset_name` includes region suffix (KSA, PRD, PEU)
- Same connection ID (all regions in same GCP project)
- Query differs only in schema name

**Three-column merge key**: Versioned dimensional data
- `weakness_id`: Which weakness
- `category_id`: Which category
- `version`: Which version of taxonomy
- Composite key required for unique identification

**Shared defaults**: Maximizes DRY
- All regions use same schedule, write_option, dates
- Only dataset_name and query differ per region
- Easy to add new regions

**Versioning**: Tracks taxonomy changes
- `version` column in merge_on
- Different versions coexist in table
- Analytics can compare across versions

**Date windowing**: Incremental by updated_at
- Only process changed records
- Efficient for large reference tables
- Handles corrections and new versions

---

## Example 6: Preserve with Soft Deletes

**Use case**: Active user list with deletion tracking

### Config File

```json
{
    "defaults": {
        "schedule": 1440,
        "storage_account": "dataaggbronze",
        "storage_container": "landing-zone",
        "schema_repo": "git@github.com:bd-vitalstatistix/superduck-lake-schema-tracking.git",
        "dataset_category": "product",
        "datalake_class": "datum",
        "start_year": 2024,
        "start_month": 1,
        "start_day": 1,
        "catchup": true,
        "max_active_runs": 3,
        "column_classification": {},
        "column_description": {},
        "folder": "mlgpu"
    },
    "dags": [
        {
            "dataset_name": "active_users",
            "dataset_product": "iam",
            "data_source": "iam_database_ro",
            "conn_id": "iam_db_readonly",
            "write_option": "preserve",
            "merge_on": "user_id",
            "dag_name": "iam_active_users",
            "notes": "Active user list from IAM system. Uses preserve write option to track when users are deactivated. Query returns only active users; preserve marks unmatched records (deactivated users) as is_current=false.",
            "query": "SELECT user_id, username, email, department, role, last_login, created_at FROM iam.users WHERE status = 'active'",
            "column_classification": {
                "email": "confidential",
                "username": "internal"
            },
            "column_description": {
                "user_id": "Unique user identifier (UUID)",
                "username": "User login name",
                "email": "User email address",
                "department": "Department code",
                "role": "User role (admin, user, readonly)",
                "last_login": "Most recent login timestamp",
                "created_at": "Account creation timestamp"
            }
        }
    ]
}
```

### Key Design Decisions

**Preserve write option**: Soft-delete pattern
- Query returns only `status = 'active'`
- Preserve marks unmatched records as `is_current = false`
- Can see when users were deactivated

**Single-column merge**: `user_id`
- Simple unique identifier
- No composite key needed

**Why not SCD**: Simpler history model
- Don't need every change to user record
- Only care about active vs inactive
- Less storage than full SCD

**Daily schedule**: Catch deactivations promptly
- Security/compliance requirement
- Access revocation tracking
- Audit trail for user lifecycle

**Email confidential**: Privacy classification
- Downstream encryption/masking
- Access controls
- GDPR compliance

---

## Common Patterns Summary

### Pattern: Daily Incremental with Corrections
**Examples**: SCA phone-home, scan responses  
**Characteristics**: Date windowing, overwrite, composite key  
**When**: Large datasets, late-arriving corrections possible

### Pattern: Weekly Full Refresh
**Examples**: TRIMS reference data  
**Characteristics**: full_overwrite, no merge_on, weekly schedule  
**When**: Small reference tables, complete snapshot available

### Pattern: SCD Historical Tracking
**Examples**: Customer dimensions, product catalog  
**Characteristics**: scd write option, composite key optional  
**When**: Need full change history, audit/compliance requirements

### Pattern: Append-Only Logs
**Examples**: Application events, clickstream  
**Characteristics**: append, no merge_on, high frequency  
**When**: Immutable events, no updates ever

### Pattern: Multi-Region Replication
**Examples**: Polaris regional data  
**Characteristics**: Same structure, different dataset_name suffix  
**When**: Same data from multiple sources/regions

### Pattern: Soft-Delete Tracking
**Examples**: Active users, active subscriptions  
**Characteristics**: preserve, query filters to current only  
**When**: Need to know when records disappeared
