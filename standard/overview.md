# Mojaloop Logging Standard

**Version:** 1.0.0

## Purpose

This document defines the logging standard for Mojaloop projects to ensure:
- **Consistent log levels** across all services
- **OpenTelemetry compatibility** for unified observability
- **Distributed tracing integration** via trace context propagation
- **Actionable logs** in production environments
- **Efficient debugging** without log noise
- **Performance optimization** through appropriate log levels
- **Security** by preventing sensitive data exposure

## Standard Components

The standard is broken down into the following sections:

*   [Log Levels](./log_levels.md) - Definitions, OTel mapping, and decision trees.
*   [Data Model](./data_model.md) - JSON structure, required fields, and naming conventions.
*   [Trace Context](./trace_context.md) - Propagation and correlation rules.
*   [Security](./security.md) - PII and sensitive data handling.
*   [Best Practices](./best_practices.md) - Performance, anti-patterns, and implementation guidelines.

## Quick Reference Summary

| Level | OTel Severity | Production | Use Case | Example |
|-------|---------------|-----------|----------|---------|
| ERROR | 17-20 | ✅ Always | System/operation failures | Database connection lost |
| WARN  | 13-16 | ✅ Always | Recoverable issues | Validation failure, retry attempt |
| INFO  | 9-12  | ✅ Always | Business events | Transfer completed, service started |
| DEBUG | 5-8   | ⚠️ Temporarily | Troubleshooting | Function calls, state changes |
| TRACE | 1-4   | ❌ Rarely | Deep diagnosis | All variable mutations, protocol details |

**Default production setting: INFO level (SeverityNumber >= 9)**
- Captures all important business events
- Minimizes performance impact
- Reduces log noise
- Can be changed to DEBUG temporarily for troubleshooting
- Compatible with OpenTelemetry severity filtering

## Extended Scenarios

Specific standards for common scenarios are available in the specific standards documents:

*   [HTTP Requests](./scenarios/http_requests.md) - Standard for incoming and outgoing HTTP logging.
*   [Error Handling](./scenarios/error_handling.md) - Rules for logging exceptions and propagating errors.
*   [Database Queries](./scenarios/sql_queries.md) - Guidelines for logging SQL and DB interactions.
