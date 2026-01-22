# Best Practices and Guidelines

This document provides guidelines on implementation, performance optimization, and common anti-patterns to avoid.

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

### High Volume Sampling and Aggregation

In high-throughput environments (e.g., 10,000 TPS), logging every successful event can overwhelm storage and analysis systems.

*   **Metrics vs. Logs:**
    *   Do **NOT** use logs for counting volume or calculating success rates. Use **Metrics** (Counters, Histograms) for throughput, latency, and error rate tracking.
    *   Use **Logs** for high-cardinality details that cannot be captured in metrics (e.g., specific transaction IDs, error reasons).
*   **Sampling:**
    *   Operational INFO logs (e.g., "Request received", "Health check") should be subject to **probabilistic sampling**.
    *   **Recommendation:** Keep 100% of ERROR/WARN logs, but sample INFO success logs (e.g., 1% or 0.1%) in high-volume paths.
    *   OpenTelemetry offers native Sampling processors to handle this at the collection layer.

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

