# Logging Patterns

Use this reference for Roze structured logging, file maintenance, audit records, event schemas, and generated tracing lifecycle. Local output is configured by `ServiceConfig.logging`; distributed span export remains under `ServiceConfig.telemetry`.

## Configuration And Sinks

When `logging` is omitted, Roze uses enabled text output to stdout at `info`. For production, prefer JSON without ANSI and configure stdout, ordinary file, and audit sinks explicitly. Filter precedence is `logging.env_filter`, then `RUST_LOG`, then `logging.level`.

Stdout and ordinary file sinks use separate bounded asynchronous writers. `lossy: true` permits ordinary sinks to drop lines rather than block service work. Keep `TracingGuard` alive until orderly shutdown; inspect `TracingGuard::dropped_lines`, `roze_log_lines_dropped_total{sink}`, and `log.lines.dropped`. Sink labels are bounded to `stdout`, `file`, and `audit`.

File sinks support hourly, daily, or never rotation; optional size segments; gzip compression levels 0-9; retention; and periodic maintenance. Hourly/daily `file_name_pattern` values require exactly one `{date}` token, while `rotation: never` forbids it. A zero `max_file_size_bytes` disables size rotation. Maintenance touches only inactive files matching the configured pattern, never the active or unrelated files. Sink preparation and subscriber installation fail startup when invalid.

The old `file_name` and `compress_rotated` fields are removed, not aliases. Migrate atomically to `file_name_pattern` and `compression`; `LogFileConfig` rejects unknown fields in every profile.

## Audit

Configure `logging.audit` as an independent JSONL sink and emit through `roze_log::audit_info!`, `audit_warn!`, or `audit_error!`, which use the reserved `roze.audit` target. Without a dedicated audit sink, these events remain in ordinary output rather than disappearing. The audit queue is always non-lossy, so producers wait when full; audit sink preparation is fail-closed and accepted records flush during orderly shutdown.

Audit records need a stable `event`, bounded actor/subject identity, resource type/id, operation, and outcome, plus tenant/request/trace IDs when available. Give audit files separate paths, longer retention where required, and restricted filesystem permissions. Never include credentials, tokens, bodies, before/after snapshots, or raw dependency errors.

## Event And Data Rules

Use stable dotted names from `roze_log::events`. Boundary fields stay bounded: service/protocol, operation or route/method/topic, request/trace IDs, elapsed time, outcome, status, numeric code, and error kind. Maud completion uses `response_kind: html` and `content_type: text/html; charset=utf-8` without the rendered body. Normal milestones are `INFO`, client rejection/cancellation is `WARN`, and failures are `ERROR`; do not emit completion for a failed operation.

Never log request/message bodies, rendered HTML, authorization/cookie values, credentials, tokens, SQL arguments, dependency response bodies, or fallback payloads. Prefer omission; if representation is necessary, wrap the value with `roze_log::Sensitive` so both `Display` and `Debug` render `[REDACTED]`. Configuration redaction protects only `Debug`, not direct display, serialization, or arbitrary error strings.

Generated REST/RPC logic emits started and exactly one completed/failed event with elapsed time and request correlation. Maud emits `html.render.completed`; Stream includes bounded topic/partition/offset/attempt context and omits payloads. Update generator templates and tests when changing these event contracts rather than editing generated services.
