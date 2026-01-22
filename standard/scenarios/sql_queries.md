# SQL and Database Logging Standard

## Overview
This standard defines the required fields for logging database interactions. Note that full query logging is generally reserved for DEBUG level or specific audit requirements to avoid leaking sensitive data (PII).

## Query Logging

### Log Level
*   **DEBUG**: Log all queries for development/debugging.
*   **INFO/WARN**: Log slow queries (exceeding a defined threshold).
*   **ERROR**: Log failed queries.

### Required Fields

| Field Name | Type | Description | Example |
|------------|------|-------------|---------|
| `db.system` | string | The database management system | "mysql", "postgresql", "redis" |
| `db.statement` | string | The sanitized SQL statement | "SELECT * FROM users WHERE id = ?" |
| `db.query.name` | string | Logical name (e.g. from Knex comment) | "FetchParticipant" |
| `db.rows_affected` | number | Count of rows affected/returned | 42 |
| `duration.ms` | number | Execution time in milliseconds | 15.5 |

### Optional Fields (TRACE Only)
These fields are expensive or sensitive and should **only** be logged at `TRACE` level or when tracing is explicitly enabled.

*   `db.parameters`: Array of query parameter values (Must be masked for PII).
*   `db.rows_returned`: Preview of the result set (Truncated).

### Security Warning
*   **Never** logs `db.parameters` or `db.rows_returned` by default in Production (`INFO`).
*   **Never** include raw values in `db.statement`. Use placeholders (`?`, `$1`).
*   **Masking**: Even at `TRACE`, sensitive fields (passwords, PINs) must be redacted from parameters and results.

### Example
```json
{
  "level": "DEBUG",
  "message": "Executing SQL Query",
  "db.system": "mysql",
  "db.statement": "SELECT * FROM transfers WHERE id = ?",
  "db.query.name": "GetTransferById",
  "db.rows_affected": 1,
  "duration.ms": 23
}
```

