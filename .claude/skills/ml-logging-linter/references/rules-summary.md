# Mojaloop Logging Rules — Quick Reference

Use this table when the full RULES.md cannot be read at runtime.
For detailed examples and traceability, see the full file at the path specified in the skill.

| Rule | Severity | What it catches |
|------|----------|-----------------|
| `generic-log-message` | Warning | Vague messages ("failed", "processing") without inline context |
| `no-manual-trace-id` | Error | Manual traceId/spanId/traceFlags in log objects (OTel handles this) |
| `catch-and-log-bubble` | Warning | Catch-log-rethrow (pure or wrapped) at `error` level — remove log or downgrade to `warn`. Exception: background jobs without global handler |
| `http-semantics` | Warning | HTTP logs missing OTel attributes per direction; duration in seconds; redact url.query; error.type resolution order |
| `losing-error-stack` | Error | Logging `error.message` instead of the full error object |
| `no-console` | Error | `console.log/error/warn` instead of the standard logger |
| `no-error-context` | Error | `logger.error(err)` without a descriptive context message |
| `no-manual-level-check` | Warning | `if (logger.level === 'debug')` instead of built-in level gating |
| `no-stringified-json` | Warning | `JSON.stringify()` inside log calls (library handles serialization) |
| `non-standard-attributes` | Warning | camelCase attributes instead of OTel dot notation (`error.message`, `duration.ms`) |
| `semantic-log-levels` | Warning | Level/keyword mismatch (e.g., "failed" at `info`, "retrying" at `error`) |
| `sensitive-data` | Error | Never-log (passwords, tokens, keys) or mask (accounts → `****1234`); redact at transport level |
| `sql-semantics` | Warning | SQL logs missing `db.query.text`/`db.system.name`; duration in seconds; never log `db.query.parameter.*` in prod |
| `unnecessary-debug-guard` | Warning | `isDebugEnabled` guarding simple log statements (only needed for expensive computation) |
| `valid-log-levels` | Error | Non-standard levels like `critical`, `warning` (use `fatal`, `warn`) |
| `deprecated-logger` | Error | Uppercase `Logger` usage instead of `logger` (ContextLogger instance) |
| `constant-log-prefix` | Warning | Log prefix not searchable — flag function calls, mutable vars, high-cardinality IDs; accept literals and `const` variables |
| `no-string-interpolation-context` | Warning | Dynamic values in message string instead of structured attributes |
| `kafka-semantics` | Warning | Kafka logs missing OTel messaging attributes (use Kafka wrappers) |
| `no-silent-catch` | Warning | Catch block neither logs nor rethrows — silently swallowed error |
| `no-loop-logging` | Warning | Logging inside tight loops — batch or log aggregates instead |
| `expected-error-level` | Warning | Expected/recoverable errors (ER_DUP_ENTRY, retries) at `error` instead of `warn` |
| `fspiop-header-handling` | Error | FSPIOP-Signature logged raw instead of hashed |
| `sql-no-raw-values` | Error | SQL query text with interpolated values instead of parameterized placeholders |
| `exception-attributes` | Warning | Manual error attrs must use OTel names; `error.type` resolution: err.code → err.name → "UnknownError" |
| `no-silent-function` | Warning | Handler/domain/service function with zero log statements — invisible to operators |
