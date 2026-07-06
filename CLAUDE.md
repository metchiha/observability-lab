# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A standalone Docker Compose observability stack (LGTM-style) for the `devops-lab-api` application, which lives in a **separate** repo/host and is not part of this codebase. This repo contains only infrastructure config — no application source code, no build system, no tests.

## Commands

```bash
# Start the full stack
docker compose up -d

# Restart a single service after editing its config
docker compose restart <service>      # e.g. prometheus, loki, tempo, otel-collector, alloy, grafana

# Validate a config file was picked up / check for errors
docker compose logs -f <service>

# Tear down (keep volumes/data)
docker compose down

# Tear down and wipe stored metrics/logs/traces
docker compose down -v
```

There is no build, lint, or test tooling — changes are verified by restarting the affected container and checking its logs, and by inspecting data in Grafana at `http://localhost:3000` (admin/devopslab).

## Architecture: the telemetry pipeline

Data flows in two parallel paths that both end up queryable from Grafana:

1. **App telemetry (traces, logs, metrics) → OTel Collector → backends**
   The `devops-lab-api` app pushes OTLP data to `otel-collector` on ports 4317 (gRPC) / 4318 (HTTP). The collector (`otel-collector-config.yaml`) fans it out:
   - traces → Tempo via `otlphttp/tempo` exporter on port **4418** (Tempo's OTLP HTTP — intentionally non-default to avoid clashing with the collector's own 4318)
   - logs → Loki via the `loki` exporter, with a `resource` processor that copies `service.name` into `loki.resource.labels` so logs are labeled by service
   - metrics → re-exposed on `:8889` in Prometheus format (the collector does not push metrics anywhere; Prometheus scrapes it)

2. **Container logs → Alloy → Loki**
   `alloy-config.alloy` runs a completely separate path: Alloy discovers all Docker containers via the mounted `docker.sock`, relabels container metadata (`container`, `stream`, `image`) into Loki labels, and ships raw container stdout/stderr logs to Loki directly. This is how you see infra-level logs (e.g. from Grafana, Prometheus itself) as opposed to app-emitted logs.

3. **Metrics scraping (Prometheus)**
   `prometheus.yaml` scrapes three targets: the app's own `/metrics` endpoint directly (both a local docker-compose target `app:8000` and a hardcoded production IP `5.161.250.79:8001` — **both entries must be kept in sync if the app's metrics port changes**), the otel-collector's self-metrics on `:8889`, and Prometheus itself.

4. **Grafana wiring** (`grafana/provisioning/datasources/datasources.yml`)
   Cross-signal correlation is configured declaratively, not clicked together in the UI:
   - Prometheus → Tempo: exemplars with `trace_id` jump to a trace
   - Loki → Tempo: a `derivedFields` regex extracts `trace_id=([a-f0-9]+)` from log lines to link to the matching trace
   - Tempo → Loki: `tracesToLogsV2` jumps from a span to correlated logs in a ±1m window
   - Tempo → Prometheus: `serviceMap` renders the service graph

## Port map (non-obvious ones)

- `4317`/`4318` — OTLP gRPC/HTTP, app → otel-collector
- `4417`/`4418` — OTLP gRPC/HTTP, otel-collector → Tempo (deliberately offset from the 4317/4318 app-facing ports)
- `8889` — otel-collector's own metrics, scraped by Prometheus (pull, not push)
- `12345` — Alloy's debug UI, useful for inspecting the log discovery/relabeling pipeline live

## When editing configs

- All five backend configs (`prometheus.yaml`, `loki-config.yaml`, `tempo-config.yaml`, `otel-collector-config.yaml`, `alloy-config.alloy`) are bind-mounted read-only into their containers — edits require a `docker compose restart <service>`, not a rebuild.
- Storage is filesystem-backed with no external object store (S3/GCS) — Loki and Tempo write to named Docker volumes (`loki-data`, `tempo-data`). Retention is capped short: Prometheus 7d (`--storage.tsdb.retention.time`), Tempo 48h (`compactor.compaction.block_retention`); Loki has no explicit retention set.
- Any new Grafana datasource or correlation link goes in `grafana/provisioning/datasources/datasources.yml`, not configured manually in the UI, so it survives `docker compose down -v`.
