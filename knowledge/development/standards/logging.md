# Logging Guidelines

## Overview

This document defines logging standards using the **standard Mule `<logger>` component**. Logs are simple, level-tagged, plain-text messages — no structured-content payloads, no custom logger module, no tracepoint discipline.

For XML examples, see [Logging Usage Examples](../usage/logging.md).

## Logger Component

All APIs MUST use Mule's built-in `<logger>` component.

```xml
<logger level="INFO" message="Order retrieval started" doc:name="Log start"/>
```

- **`level`** — one of `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`
- **`message`** — plain text. May reference variables inline (e.g., `"Order #[vars.orderId] retrieved"`) but should NOT embed multi-line DataWeave structures
- **`doc:name`** — short human-readable description of what is being logged

Do NOT use `<json-logger:logger>`, `<json-logger:config>`, or any custom logging module. Do NOT introduce a `loggerContext` variable or any equivalent structured-context payload.

## Log Levels

| Level | Usage |
|-------|-------|
| `ERROR` | Errors that require immediate attention (ALL error handling) |
| `WARN`  | Potential issues or degraded operations |
| `INFO`  | Key operational events (flow start/end, important business steps) |
| `DEBUG` | Detailed information for troubleshooting (variable values, decision points) |
| `TRACE` | Very detailed debugging (rarely used) |

## General Rules

- **Level = ERROR** for ALL error-handler logging
- **INFO** for flow start, flow end, and significant business milestones
- **DEBUG** for fine-grained troubleshooting (request/response context, variable state)
- Keep messages **short and plain**. Reference variables inline with `#[…]` only when the value is genuinely useful in the log
- Never log sensitive data — see [Field Masking](#field-masking) below
- Log levels per environment are controlled by `log4j2.xml`, not by the logger component itself

## Correlation ID Tracking

Even though logs are now plain text, request tracing still depends on `correlationId`:

| Rule | Requirement |
|------|-------------|
| HTTP Requests | MUST always send `correlationId` |
| Connectors | MUST send `correlationId` if supported |
| Format | Use `vars.correlationId` (preferred over plain `correlationId`) |
| Header Name | `x-correlation-id` |

The correlation ID is typically captured into `vars.correlationId` at the start of the main flow from the inbound `x-correlation-id` header (falling back to Mule's built-in `correlationId` keyword), then forwarded on every outbound HTTP request. See [usage/logging.md](../usage/logging.md) for the XML.

## Anti-Patterns

| ❌ Anti-Pattern | ✅ Correct |
|---|---|
| Using `<json-logger:logger>` | Use `<logger level="…" message="…"/>` |
| Embedding multi-line DataWeave inside a logger message | Set the value to a variable first, then reference it from the message |
| Logging full request/response payloads at INFO | Use DEBUG, and only when troubleshooting |
| Logging passwords, tokens, or PII at any level | Mask or omit — see below |
| ERROR level for expected non-error flows | Use INFO or DEBUG |

## Field Masking

Standard Mule logger has no built-in masking. To avoid logging sensitive data:

- Do not include secrets, tokens, passwords, credit-card numbers, SSNs, or other PII in `message` or in any variable that gets logged
- If you need to log a payload during troubleshooting, transform it first to remove sensitive fields, then log the redacted version
- Common fields to keep out of logs: `password`, `token`, `secret`, `authorization`, `apiKey`, `clientSecret`, `creditCard`, `ssn`, plus project-specific fields (e.g., `taxId`, `bankAccount`)

## Log Level Configuration

Configure log levels per environment in `log4j2.xml`:

| Environment | Level |
|---|---|
| `prod` | INFO and above |
| `dev` / `qa` | DEBUG for troubleshooting |
| Library namespaces (`org.mule.runtime`, `org.apache.http`) | WARN to reduce noise |

## Best Practices

### Do's

- Use the standard Mule `<logger>` everywhere
- Use INFO for flow start/end and key business milestones
- Use DEBUG for detailed troubleshooting
- Use ERROR inside error-handler scopes
- Keep messages short and plain
- Forward `correlationId` on every outbound HTTP request

### Don'ts

- Don't use `<json-logger:logger>` or any custom logger module
- Don't embed multi-line DataWeave inside a logger message
- Don't log secrets, tokens, or PII
- Don't log full payloads at INFO level
- Don't log inside tight loops (performance impact)
- Don't use ERROR for expected scenarios

## Logging Checklist

- [ ] All loggers use Mule's standard `<logger>` component
- [ ] Every logger has a `level` and a `doc:name`
- [ ] Messages are plain text (variable references with `#[…]` are OK; multi-line DataWeave is not)
- [ ] INFO at flow start and end
- [ ] DEBUG for troubleshooting details
- [ ] ERROR inside error-handler scopes
- [ ] Outbound HTTP requests propagate `correlationId` via `x-correlation-id`
- [ ] No payloads at INFO level
- [ ] No secrets, tokens, or PII in any log

## References

- [Logging Usage Examples](../usage/logging.md) — full XML implementations
- [Naming Conventions](./naming-conventions.md) — variable & flow naming
- [Error Handling](./error-handling.md)
- [Mule Logger Component Documentation](https://docs.mulesoft.com/mule-runtime/latest/logger-component-reference)

---
Last updated: 2026-05-16
Owner: Integration Team
