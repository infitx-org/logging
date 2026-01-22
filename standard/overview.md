# Mojaloop Logging Standard

## Purpose

This document defines the logging standard for Mojaloop projects to ensure:
- **Consistent log levels** across all services
- **OpenTelemetry compatibility** for unified observability
- **Distributed tracing integration** via trace context propagation
- **Actionable logs** in production environments
- **Efficient debugging** without log noise
- **Performance optimization** through appropriate log levels
- **Security** by preventing sensitive data exposure

## OpenTelemetry Alignment

This standard aligns with [OpenTelemetry Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/) to ensure:
- Logs can be correlated with traces and metrics
- Compatibility with OpenTelemetry collectors and backends
- Standardized severity levels and structured format
- Automatic context propagation in distributed systems

### Standard Field Mapping

| Mojaloop Concept | OTel Field Name | Description |
|------------------|-----------------|-------------|
| Log Level        | `SeverityNumber`| Numerical value of the severity (1-24) |
| Level Name       | `SeverityText`  | The text string representation (e.g., "INFO") |
| Message          | `Body`          | The human-readable message content |
| Context          | `Attributes`    | Structured key-value pairs (event specific) |
| Service Info     | `Resource`      | Source description (Service Name, Version, Env) |
| Scope            | `InstrumentationScope` | The library/module emitting the log |

### Log Levels

### OpenTelemetry Severity Mapping

Mojaloop log levels map to OpenTelemetry SeverityNumber ranges for compatibility:

| Mojaloop Level | OTel SeverityNumber | OTel Range | Numeric Value |
|----------------|---------------------|------------|---------------|
| TRACE          | TRACE               | 1-4        | 1             |
| DEBUG          | DEBUG               | 5-8        | 5             |
| INFO           | INFO                | 9-12       | 9             |
| WARN           | WARN                | 13-16      | 13            |
| ERROR          | ERROR               | 17-20      | 17            |
| FATAL          | FATAL               | 21-24      | 21            |

When emitting logs via OpenTelemetry SDK, use the corresponding SeverityNumber. Most logging libraries will handle this mapping automatically.

### FATAL - Uncoverable System Failures

**When to use:**
- System cannot continue to function
- Critical dependency failure preventing startup
- Security breach detected requiring shutdown
- Unrecoverable data corruption

**What to include:**
- Cause of fatal error
- Shut down status
- Immediate action required

**Examples:**
```javascript
logger.fatal(`Central Ledger service shutting down: Database unreachable at startup after ${maxRetries} attempts`, {
  eventName: 'ServiceShutdown',
  reason: 'DatabaseUnreachable',
  'db.host': dbHost,
  'error.message': error.message
});
```

### ERROR - Critical Issues Requiring Immediate Attention

**When to use:**
- Unhandled exceptions or errors
- Database connection failures
- External service failures (payment processor down)
- Data corruption or integrity issues
- Authentication/authorization failures
- Critical business rule violations
- Any condition that prevents normal operation

**What to include:**
- Error message and stack trace
- Operation context (function name, operation type)
- Request/transaction IDs for tracing
- User/account IDs (if applicable)
- Timestamp

**Examples:**
```javascript
// Trace context is AUTOMATIC - don't add manually!
logger.error(`Transfer ${transfer.id} processing failed in processTransfer operation: ${error.message}`, {
  // Attributes only - trace context added automatically by OTel
  operation: 'processTransfer',
  eventName: 'TransferFailed',
  transferId: transfer.id,
  'error.type': error.name,
  'error.message': error.message,
  'error.stack': error.stack
});

logger.error(`Database connection to ${dbConfig.host}:${dbConfig.port} failed after ${retryCount} retry attempts: ${error.message}`, {
  operation: 'connectDatabase',
  eventName: 'DatabaseConnectionFailed',
  'db.host': dbConfig.host,
  'db.port': dbConfig.port,
  'db.name': dbConfig.database,
  retryCount: retryCount,
  'error.type': error.name,
  'error.message': error.message
});
```

**Do NOT use for:**
- Expected validation failures (use WARN)
- Retry-able operations that succeed on retry
- User input errors (use WARN)

### WARN - Recoverable Issues or Degraded Operation

**When to use:**
- Expected validation failures
- Retry attempts (before final failure)
- Deprecated API usage
- Configuration issues with fallback values
- Business rule violations that can be handled
- Rate limiting triggered
- Temporary service unavailability with fallback
- Performance degradation warnings

**What to include:**
- Warning description
- Context of what triggered the warning
- Action taken (fallback, retry, skip)
- Request/transaction IDs

**Examples:**
```javascript
logger.warn(`Transfer ${transfer.id} validation failed during balance check. Account balance is ${account.balance} ${account.currency} but transfer amount is ${transfer.amount} ${transfer.currency}`, {
  operation: 'validateTransfer',
  eventName: 'TransferRejected',
  transferId: transfer.id,
  accountId: account.id,
  'account.balance': account.balance,
  'account.currency': account.currency,
  'transfer.amount': transfer.amount,
  'transfer.currency': transfer.currency,
  validationErrors: errors,
  reason: 'InsufficientFunds'
});

logger.warn(`API rate limit of ${limit} requests per ${period} reached for endpoint ${endpoint}. Throttling subsequent requests`, {
  operation: 'rateLimit',
  eventName: 'RateLimitTriggered',
  endpoint: endpoint,
  limit: limit,
  period: period,
  currentCount: requestCount
});
```

**Do NOT use for:**
- Normal business operations
- Successful validation
- Debug information

### INFO - Significant Business Events

**When to use:**
- Service startup/shutdown
- Successful completion of major operations
- State transitions (transfer approved, settlement completed)
- Configuration loaded
- Connection established/closed
- Scheduled job execution
- API request/response (at entry points only)
- Authentication success

**What to include:**
- Event description
- Key identifiers (IDs, names)
- Relevant business data (amounts, status changes)
- Duration for completed operations

**Examples:**
```javascript
logger.info(`Transfer ${transfer.id} completed successfully from ${transfer.payerFsp} to ${transfer.payeeFsp} for ${transfer.amount.amount} ${transfer.amount.currency} in ${duration}ms`, {
  operation: 'processTransfer',
  eventName: 'TransferCompleted',
  transferId: transfer.id,
  'payer.fspId': transfer.payerFsp,
  'payee.fspId': transfer.payeeFsp,
  'transfer.amount': transfer.amount.amount,
  'transfer.currency': transfer.amount.currency,
  'duration.ms': duration
});

logger.info(`Service ${serviceName} v${version} started successfully on port ${port} in ${env} environment`, {
  eventName: 'ServiceStarted',
  'service.name': serviceName,
  'service.version': version,
  'deployment.environment': env,
  'server.port': port
});
```

**Do NOT use for:**
- Internal function calls
- Loop iterations
- Temporary variable values
- Detailed operation steps (use DEBUG)

### DEBUG - Detailed Operational Information

**When to use (in non-production or when debugging):**
- Function entry/exit with parameters
- Intermediate calculation results
- State changes during operation
- Conditional branch taken
- Loop iterations (sparingly)
- Cache hits/misses
- Validation steps

**What to include:**
- Detailed context
- Variable values
- Flow indicators
- Internal state

**Examples:**
```javascript
logger.debug(`Processing transfer ${transfer.id} validation at step ${step}. Account balance is ${balance} and transfer amount is ${amount}`, {
  operation: 'validateTransfer',
  transferId: transfer.id,
  step: 'checkBalance',
  'account.balance': balance,
  'transfer.amount': amount,
  accountId: account.id
});

logger.debug(`Cache miss for key ${cacheKey}. Fetching participant ${participantId} from database`, {
  operation: 'getParticipant',
  cacheKey: cacheKey,
  participantId: participantId,
  entity: 'participant'
});
```

**Do NOT use for:**
- Sensitive data (passwords, tokens, full card numbers)
- Excessive logging in tight loops
- Information available elsewhere

### TRACE - Very Detailed Diagnostic Information

**When to use (only when explicitly needed for troubleshooting):**
- Every function entry/exit
- All variable mutations
- Detailed execution flow
- Performance profiling
- Protocol-level details

**What to include:**
- Maximum detail for diagnosis
- Full context including all parameters
- Execution timestamps

**Examples:**
```javascript
logger.trace(`Entering validateTransfer function with transfer ${transfer.id}, applying ${rules.length} validation rules in context ${context.requestId}`, {
  operation: 'validateTransfer',
  transferId: transfer.id,
  'validation.rulesCount': rules.length,
  requestId: context.requestId,
  args: { transfer, rules, context },
  timestamp: Date.now()
});
```

**Do NOT use:**
- In production by default (too verbose)
- For sensitive data

## Log Level Decision Tree

```
Is the application unable to continue? 
  YES → ERROR

Can the application continue with degraded functionality?
  YES → WARN

Is this a significant business event users/operators care about?
  YES → INFO

Is this needed for troubleshooting but not in production?
  YES → DEBUG

Is this only needed for deep diagnostic analysis?
  YES → TRACE
```

## Verbosity and Sampling Rules

### Per-Level Configuration
The complexity and structure of logs depend on their level.

*   **WARN / ERROR / FATAL**: Always logged, regardless of configuration.
    *   **Must** include full stack traces for exceptions.
    *   **Must** include relevant context (IDs) to make the error actionable.
*   **INFO**: Standard operational level. Logged by default in Production.
    *   Should describe *what* happened (business events).
    *   Should *not* describe *how* (internal implementation details).
*   **DEBUG**: Disabled by default in Production.
    *   Contains detailed state changes, payload summaries, and logic flow.
    *   Intended for developers debugging non-production environments.
*   **TRACE**: Disabled by default.
    *   Contains full payload dumps (body content), loop iterations, and variable values.
    *   Should only be enabled explicitly for deep diagnostics.

### Dynamic Tracing Override (Force-Logging)
To support debugging specific transactions in production without increasing the global log verbosity:

*   If an incoming request indicates tracing is enabled (e.g., via `X-Trace-Enabled` header or sampled flag in OTel context):
    *   **WARN / ERROR / FATAL / TRACE**: Automatically logged for that request context.
    *   **INFO / DEBUG**: Automatically promoted to be logged even if the service default is set to `WARN` or `INFO`.
    *   *Goal:* Allow end-to-end tracing of a specific request through the entire system at high fidelity while keeping the rest of the system quiet.

## Production Logging Guidelines

### Performance Considerations
- **Avoid expensive operations at DEBUG/TRACE** - These should be cheap since they may be enabled temporarily
- **Use lazy evaluation** - Don't compute log data if level won't be logged
- **Structured logging** - Log objects, not concatenated strings
- **Avoid JSON.stringify** - Let the logging library handle serialization

### Examples of Performance-Aware Logging

**Bad:**
```javascript
// Always executes stringify even if DEBUG is disabled
logger.debug(`Processing: ${JSON.stringify(largeObject)}`);
```

**Good:**
```javascript
// Object only serialized if DEBUG enabled
logger.debug('Processing transfer', { transfer });

// Or with conditional logging for very expensive operations
if (logger.isDebugEnabled()) {
  logger.debug('Detailed state', { computedState: computeExpensiveState() });
}
```

## Anti-Patterns to Avoid

### 1. Wrong Log Level
```javascript
// ❌ BAD - Using console.log for errors, generic message
console.log('Database connection failed', error);

// ✅ GOOD - Proper level with descriptive message
logger.error(`Database connection to ${dbHost}:${dbPort} failed: ${error.message}`, {
  operation: 'connectDatabase',
  'db.host': dbHost,
  'db.port': dbPort,
  'error.type': error.name,
  'error.message': error.message,
  'error.stack': error.stack
});
```

### 2. Logging Everything at INFO
```javascript
// ❌ BAD - Internal details at INFO, generic messages
logger.info('Entering validateTransfer function');
logger.info('Retrieved account from database');
logger.info('Balance check passed');

// ✅ GOOD - Only significant events at INFO with context
logger.info(`Transfer ${transferId} validated successfully for ${amount} ${currency}`, {
  operation: 'validateTransfer',
  eventName: 'TransferValidated',
  transferId: transferId,
  'transfer.amount': amount,
  'transfer.currency': currency
});
```

### 3. Missing Context in Message
```javascript
// ❌ BAD - Message doesn't explain what happened
logger.error('Validation failed');

// ✅ GOOD - Descriptive message with inline context plus structured attributes
// Trace context is automatic, don't add manually!
logger.error(`Transfer ${transfer.id} validation failed at step '${step}': ${validationErrors.join(', ')}`, {
  operation: 'validateTransfer',
  eventName: 'ValidationFailed',
  transferId: transfer.id,
  validationStep: step,
  validationErrors: validationErrors
});
```

### 4. Missing Error Stack
**Requirement:** "Verify that Errors are logged with Error Code, Error Stack defined".

```javascript
// ❌ BAD - Losing the stack trace
logger.error(`Failed: ${error.message}`);

// ✅ GOOD - Passing the error object ensures stack is captured
logger.error(`Transfer failed: ${error.message}`, {
  eventName: 'TransferFailed',
  error: error // Logger serializer should handle 'error.stack' and 'error.code'
});
```

### 5. Sensitive Data Exposure
```javascript
// ❌ BAD
logger.debug('User authentication', { password: user.password });

// ✅ GOOD
logger.debug('User authentication', { 
  userId: user.id,
  method: 'password'
});
```

### 5. Over-Logging
```javascript
// ❌ BAD - Logging inside tight loops with repeated generic messages
for (const transfer of transfers) {
  logger.info('Processing transfer', transfer); // Could be thousands
}

// ✅ GOOD - Aggregate with descriptive summary
logger.info(`Processing batch of ${transfers.length} transfers for batch ${batch.id}`, { 
  operation: 'processBatch',
  eventName: 'BatchProcessingStarted',
  'batch.id': batch.id,
  'batch.transferCount': transfers.length,
  'batch.totalAmount': transfers.reduce((sum, t) => sum + t.amount, 0)
});
```

## Structured Logging Format

### Log Message Best Practices

**The message (Body) should be human-readable and self-explanatory:**
- Include specific values inline (IDs, amounts, states)
- Describe what happened, not just the operation name
- Be detailed enough to understand the issue without looking at attributes
- Use complete sentences with context

**Examples of good messages:**
- ✅ `"Transfer abc-123 validation failed during balance check. Account balance is 100.00 USD but transfer amount is 150.00 USD"`
- ✅ `"Payment xyz-789 processing completed successfully from PayerFSP to PayeeFSP in 250ms"`
- ✅ `"Database connection to mysql://prod-db:3306 failed after 3 retry attempts: Connection timeout"`

**Examples of poor messages:**
- ❌ `"Validation failed"` (too generic, no context)
- ❌ `"Processing transfer"` (not clear what happened)
- ❌ `"Error occurred"` (provides no information)

**The attributes provide structured data for querying:**
- Use attributes for filtering, grouping, and aggregation
- Attributes enable queries like "all transfers > $1000" or "errors from service X"
- Attributes support correlation (traceId, spanId, transferId)

### OpenTelemetry Log Record Structure

Following OpenTelemetry conventions, logs should include:

**Core Fields (automatic):**
- **Timestamp**: Time when event occurred (ISO 8601 format)
- **SeverityText**: Human-readable level (ERROR, WARN, INFO, DEBUG, TRACE)
- **SeverityNumber**: Numeric severity (17=ERROR, 13=WARN, 9=INFO, 5=DEBUG, 1=TRACE)
- **Body**: Human-readable message describing the event

**Trace Context Fields (for distributed tracing):**
- **TraceId**: W3C Trace Context trace ID (**automatic** when using OTel-instrumented logging)
- **SpanId**: Current span ID (**automatic** when using OTel-instrumented logging)
- **TraceFlags**: W3C trace flags (**automatic**, typically set by tracing SDK)

**Important:** You should **NOT** manually add traceId/spanId to log calls. These are automatically injected by the logging library when properly configured with OpenTelemetry instrumentation.

**Resource Fields (describes the source):**
- `service.name`: Service name (e.g., 'central-ledger')
- `service.version`: Service version (e.g., '1.2.3')
- `deployment.environment`: Environment (e.g., 'production', 'staging')
- `host.name`: Hostname or container ID
- `process.pid`: Process ID

**Attributes (business context):**
- `operation`: Function or operation name
- `duration.ms`: For completed operations (milliseconds)
- `transferId`, `userId`, `accountId`: Business entity IDs
- `error.type`: Error class name (for errors)
- `error.message`: Error message (for errors)
- `error.stack`: Stack trace (for errors)
- `eventName`: For business events (e.g., 'TransferCompleted', 'PaymentFailed')

### Distinction: Resource vs Attributes

- **Resource**: Describes WHERE the log came from (service, host, environment) - static per service instance
- **Attributes**: Describes WHAT happened (operation, IDs, business data) - varies per log entry

**Example with OpenTelemetry Structure:**
```javascript
// Resource (set once at service startup)
const resource = {
  'service.name': 'central-ledger',
  'service.version': '1.2.3',
  'deployment.environment': 'production',
  'host.name': 'pod-xyz-123'
};

// Log entry with descriptive message, trace context, and attributes
logger.error(
  `Transfer ${transfer.id} processing failed during ${operation} operation from ${transfer.payerFsp} to ${transfer.payeeFsp}: ${error.message}`,
  {
    // Trace context (automatic if using OTel SDK)
    traceId: '4bf92f3577b34da6a3ce929d0e0e4736',
    spanId: '00f067aa0ba902b7',
    
    // Attributes (business context)
    operation: 'processTransfer',
    eventName: 'TransferFailed',
    transferId: transfer.id,
    'payer.fspId': transfer.payerFsp,
    'payee.fspId': transfer.payeeFsp,
    'transfer.amount': transfer.amount.amount,
    'transfer.currency': transfer.amount.currency,
    'error.type': 'ValidationError',
    'error.message': error.message,
    'error.stack': error.stack,
    'duration.ms': 250
  }
);
```

## Trace Context Propagation

### Automatic Context Injection (Recommended)

**When using OpenTelemetry-instrumented logging libraries, trace context is automatically injected:**

```javascript
// Setup at application startup (ONCE)
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { PinoInstrumentation } = require('@opentelemetry/instrumentation-pino');

const sdk = new NodeSDK({
  instrumentations: [
    new PinoInstrumentation()
  ]
});

sdk.start();

// Now ALL log calls automatically include trace context - no manual work needed!
logger.info('Transfer completed', { transferId: '123' });
// Logs will automatically include: { traceId: '...', spanId: '...', transferId: '123' }
```

### Manual Context Usage (NOT Recommended)

⚠️ **Only use this if your logging library doesn't support OpenTelemetry instrumentation:**

```javascript
const { trace } = require('@opentelemetry/api');

// Manual approach - avoid if possible
const span = trace.getActiveSpan();
if (span) {
  const spanContext = span.spanContext();
  
  logger.info('Processing transfer', {
    traceId: spanContext.traceId,  // Manual - not ideal
    spanId: spanContext.spanId,     // Manual - not ideal
    transferId: transfer.id
  });
}
```

**Recommendation:** Use auto-instrumentation instead of manual trace context injection.

### Benefits of Trace Context in Logs

1. **Cross-service correlation**: Find all logs related to a single request across multiple services
2. **Trace-to-log navigation**: Jump from trace spans to related logs in observability tools
3. **Root cause analysis**: See exact sequence of events leading to errors
4. **Performance debugging**: Correlate slow traces with detailed logs

## Where to Log (Avoiding Duplication)

**Goal:** Avoid "Errors captured in more than one place (often three times)".

### The "Catch and Log" Anti-Pattern
Do **NOT** catch an error just to log it and throw it again, unless you are adding significant context that cannot be added anywhere else.

```javascript
// ❌ BAD - Duplicates logs up the stack
try {
  await performAction();
} catch (error) {
  logger.error(error); // Log 1
  throw error;
}

// ... caller ...
try {
  await service.action();
} catch (error) {
  logger.error(error); // Log 2 (Duplicate)
  throw error;
}
```

### The "Catch, Context, and Bubble" Pattern
If you catch an error to add context, wrap it or attach properties, but do NOT log it until the "Edge" of the application.

```javascript
// ✅ GOOD - Add context, don't log yet
try {
  await db.query();
} catch (error) {
  throw new DatabaseError('Failed to query', { cause: error });
}

// ... Global Error Handler (The Edge) ...
// Log ONLY here
logger.error(finalError); 
```

### Exceptions
- **Background Jobs**: Log errors inside the job as they have no caller to bubble to.
- **Async Event Handlers**: Log errors inside consumers/handlers if they don't have a standardized global error handler.

## Logging HTTP Request Errors

Standardize how HTTP failures are recorded to ensure consistent dashboards.

**Required Fields:**
- `http.request.method`: GET, POST, etc.
- `http.response.status_code`: 400, 500, etc.
- `url.path`: /transfers
- `http.request.id`: Unique Request ID
- `error.code`: Application specific error code

**Example:**
```javascript
logger.error('Downstream service request failed', {
  operation: 'sendQuote',
  eventName: 'DownstreamRequestFailed',
  'http.request.method': 'POST',
  'http.response.status_code': 502,
  'url.path': '/quotes',
  'target.service': 'quoting-service',
  'error.message': 'Bad Gateway'
});
```

## Sensitive Information (PII)

**Requirement:** Strictly avoid logging sensitive information.

**Never Log:**
- Passwords / Secrets / Keys
- Full Credit Card / Bank Account Numbers (Mask: `****1234`)
- Personally Identifiable Information (PII) like Names, Phone Numbers, Addresses (unless authorized and necessary for debugging in secure envs)
- Authentication Tokens (Bearer tokens)

**Mitigation:**
- Use a "Redaction" middleware or serializer in the logger configuration.
- Validate logs in code reviews.

## Dynamic Log Level Configuration

Services should support changing log levels without restart:
- Via environment variables: `LOG_LEVEL=debug`
- Via configuration endpoint: `PUT /admin/log-level`
- Per-component configuration: `LOG_LEVEL_DATABASE=debug`

## Migration Path

For existing code:
1. Replace `console.*` with proper logger
2. **Set up OpenTelemetry SDK** with resource attributes
3. **Enable trace context propagation** (automatic with OTel instrumentation)
4. Review all ERROR logs - ensure they're actual errors
5. Review all INFO logs - move internal details to DEBUG
6. Add attributes to all logs (IDs, operation names)
7. **Use `eventName` for business events** (TransferCompleted, PaymentFailed)
8. **Separate resource attributes** (service.name, service.version) from log attributes
9. Remove sensitive data from all logs
10. Remove JSON.stringify - use structured logging

### Quick Migration Checklist

- [ ] Install OpenTelemetry SDK and instrumentation
- [ ] Configure resource attributes (service name, version, environment)
- [ ] Replace console.* with logger.* calls
- [ ] Add traceId/spanId to logs (automatic with OTel)
- [ ] Use eventName for significant business events
- [ ] Verify log level appropriateness (ERROR/WARN/INFO/DEBUG/TRACE)
- [ ] Add operation and business entity IDs to attributes
- [ ] Test trace-to-log correlation in observability backend

## Summary

| Level | OTel Severity | Production | Use Case | Example |
|-------|---------------|-----------|----------|---------|------|
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

## OpenTelemetry Integration

### Recommended Logging Libraries

**Node.js:**
- [Pino](https://github.com/pinojs/pino) with [@opentelemetry/instrumentation-pino](https://www.npmjs.com/package/@opentelemetry/instrumentation-pino)
- [Winston](https://github.com/winstonjs/winston) with OTel transport
- [@opentelemetry/api-logs](https://www.npmjs.com/package/@opentelemetry/api-logs) (direct API usage)

### Example Setup with OpenTelemetry

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPLogExporter } = require('@opentelemetry/exporter-logs-otlp-http');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'central-ledger',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.2.3',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: 'production'
  }),
  logRecordProcessor: new BatchLogRecordProcessor(
    new OTLPLogExporter({
      url: 'http://otel-collector:4318/v1/logs'
    })
  )
});

sdk.start();
```

### Exporting to OpenTelemetry Collector

Logs can be sent to OpenTelemetry Collector using:
- **OTLP/HTTP**: `http://collector:4318/v1/logs`
- **OTLP/gRPC**: `grpc://collector:4317`

The collector can then:
- Enrich logs with additional resource attributes (k8s metadata, cloud info)
- Route to multiple backends (Elasticsearch, Loki, CloudWatch)
- Correlate with traces and metrics
- Apply sampling and filtering rules

## Extended Scenarios

Specific standards for common scenarios are available in the specific standards documents:

*   [HTTP Requests](./scenarios/http_requests.md) - Standard for incoming and outgoing HTTP logging.
*   [Error Handling](./scenarios/error_handling.md) - Rules for logging exceptions and propagating errors.
*   [Database Queries](./scenarios/sql_queries.md) - Guidelines for logging SQL and DB interactions.

