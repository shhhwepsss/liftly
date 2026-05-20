# Observability

**Recommendation: structured JSON logging (Pino) + Sentry for error tracking. Nothing else at MVP.**

- **Pino** is NestJS's fastest logger, outputs structured JSON that any log aggregator (Fly.io's built-in, Papertrail, Logtail) can ingest without configuration.
- **Sentry** free tier: captures unhandled exceptions with stack traces and request context. One `npm install @sentry/node` and one DSN env var.
- No metrics/tracing (Prometheus, Datadog, OpenTelemetry) at MVP. The user base is small enough that error reports from friends are the primary signal.
- Log these events explicitly: flush received (with Set count), reconciliation overrides (warn level), FCM dispatch failures (error level), auth failures (warn level).

When the app grows past "friends-only," the next step is a hosted log aggregator (Logtail or Axiom, both cheap) and a Grafana Cloud free-tier dashboard on Pino JSON output — no architecture change required.
