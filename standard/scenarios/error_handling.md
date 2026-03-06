# Error Handling and Propagation Standard

## Overview
This standard defines how to log and propagate errors to ensure that the root cause is preserved and that logs remain actionable without being redundant.

## Logging Exceptions

When an exception occurs that cannot be handled immediately (or is being handled by a final error handler), it must be logged with specific context.

### Required Fields for Error Logs

| Field Name             | Type | Description | Example |
|------------------------|------|-------------|---------|
| `exception.type`       | string | The error class or slug | "ValidationError", "participant.notFound" |
| `exception.message`    | string | The technical error message | "Invalid account ID: 123" |
| `exception.stacktrace` | string | The full stack trace (incl. causes) | "Error: ... at verify (file.js:10)..." |
| `error.type`           | string | Error classification (class name, error code, or slug) | "ValidationError", "ECONNREFUSED" |
| `error.user_message`   | string | The user-facing notification | "Operation failed, contact provider" |

### Log Level Guidelines
For detailed definitions of log levels, refer to the [Log Levels Standard](../log_levels.md).

*   **FATAL**: The process will likely exit immediately.
*   **ERROR**: The request failed, but the process continues.
*   **WARN**: The error was handled/recovered, or is a validation issue.

## Propagating Errors

Do **NOT** simply log and re-throw the same error without context.
Do **NOT** swallow the stack trace when wrapping errors.

### Correct Pattern
Wrap the error, preserving the original cause.
```javascript
try {
  await db.query();
} catch (originalError) {
  // Wrap and throw, do NOT log here if a higher level handler will log it.
  throw new DatabaseError("Failed to query user", { cause: originalError });
}
```

### Top-Level Error Handler
Only the top-level handler (e.g., API middleware, worker loop) should log the final error with the full stack trace.

```javascript
// Global error handler
logger.error("Request failed due to unhandled exception", {
  "exception.type": err.slug || err.name,
  "exception.message": err.message,
  "exception.stacktrace": err.stack,
  "error.type": err.code || err.name,
  "error.user_message": err.notice // If available
});
```

## Review Checklist
*   Are **FATAL** errors causing a process exit?
*   Are **stack traces** strictly excluded from API responses in Production?
*   Are **wrapped errors** preserving the `cause` chain?

