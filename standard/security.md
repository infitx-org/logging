# Security and Sensitive Information

This document defines the requirements for handling sensitive data in logs to insure compliance and security.

## Sensitive Information (PII)

**Requirement:** Strictly avoid logging sensitive information.

**Never Log:**
- Passwords / Secrets / Keys
- Full Credit Card / Bank Account Numbers (Mask: `****1234`)
- Personally Identifiable Information (PII) like Names, Phone Numbers, Addresses (unless authorized and necessary for debugging in secure envs)
- Authentication Tokens (Bearer tokens)
