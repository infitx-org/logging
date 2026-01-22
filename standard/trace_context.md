# Trace Context Propagation

This document describes how distributed tracing context is propagated through logs to enable correlation across microservices.

## Automatic Context Injection (Recommended)

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

## Manual Context Usage (NOT Recommended)

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

## Benefits of Trace Context in Logs

1. **Cross-service correlation**: Find all logs related to a single request across multiple services
2. **Trace-to-log navigation**: Jump from trace spans to related logs in observability tools
3. **Root cause analysis**: See exact sequence of events leading to errors
4. **Performance debugging**: Correlate slow traces with detailed logs
