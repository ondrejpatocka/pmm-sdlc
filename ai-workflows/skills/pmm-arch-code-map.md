---
name: pmm-arch-code-map
description: PMM architecture and code-path map — core data flows, Node→Service→Agent model, and a component→code-paths table. Use to decide which PMM component/code is impacted or owns a symptom.
---

# Skill: pmm-arch-code-map

Architecture & code-path context for any PMM workflow that needs to go from a symptom to
the owning code. Referenced (not copied) by workflows such as
`pmm-triage-bug-ticket/pmm-triage-bug-workflow.md`. Takes no argument — it is reference
context.

Paths are relative to the `percona/pmm` monorepo root; cite any you land on as GitHub
links per the calling workflow's code-reference convention. Treat the **checked-out code
as truth** — if a path has moved, trust the checkout and record the divergence. (Each
component also carries its own `AGENTS.md` in-repo if you need deeper architectural
intent than this map provides.)

**Core data flows** — use these to decide which component owns a symptom:

- **Metrics:** exporters (`node`, `mysqld`, `mongodb`, `postgres`, `proxysql`, `valkey`, `rds`, `azure`) → VMAgent scrapes → VictoriaMetrics (storage) → Grafana (viz); alerting via VMAlert → Alertmanager.
- **QAN:** QAN agents inside pmm-agent (perfschema, slowlog, `pg_stat_statements`, `pg_stat_monitor`, Mongo profiler) → pmm-managed (gRPC receiver) → qan-api2 (collector) → ClickHouse → UI / Grafana.
- **Agent communication:** pmm-agent ↔ pmm-managed over a single bidirectional gRPC stream (server sends `SetStateRequest`/`StartAction`/`StartJob`/`Ping`; agent sends `StateChanged`/`QanCollect`/`ActionResult`/`JobResult`/`Pong`).
- **Backups:** pmm-managed orchestrates → pmm-agent jobs (PBM for MongoDB; `xtrabackup`/`mysqldump` for MySQL) → S3 / MinIO / local.
- **Inventory domain model:** `Node → Service → Agent` (a Node has many Services; an Agent `runs_on_node_id` and optionally monitors a `service_id`; child agents belong to a parent `pmm_agent_id`).

**Component → purpose → key code paths:**

| Component (dir) | Purpose | Key code paths (grep/open these first) |
|---|---|---|
| **pmm-managed** (`managed/`) | Server backend: inventory, gRPC/REST APIs, VM & Grafana integration, backup, alerting, HA | `managed/cmd/pmm-managed/` (bootstrap & service wiring), `managed/models/` (reform ORM models + `database.go` schema migrations), `managed/services/inventory/` (Node/Service/Agent CRUD + validation), `managed/services/agents/` (agent registry & lifecycle), `managed/services/victoriametrics/` (scrape-config generation), `managed/services/grafana/`, `managed/services/ha/`. gRPC :7771, REST :7772. |
| **pmm-agent** (`agent/`) | Client agent: runs exporters, QAN/RTA collectors, actions, backup/restore jobs | `agent/commands/run.go` (main event loop), `agent/agents/supervisor/` (reconciles desired↔actual state), `agent/agents/process/` (exporter process state machine), `agent/client/` (gRPC stream handler), `agent/runner/actions/` (explain, pt-summary), `agent/runner/jobs/` (backup/restore), `agent/agents/{mysql,postgres,mongodb}/` (built-in collectors). No DB — state comes from managed via gRPC. |
| **pmm-admin** (`admin/`) | CLI to add/remove monitored services | `admin/cmd/pmm-admin/main.go`, `admin/commands/management/` (e.g. `add_mysql.go`), `admin/commands/inventory/`, `admin/commands/base.go`. Kong CLI; talks to managed via **generated Swagger HTTP clients** (`api/*/json/client/`), not gRPC. |
| **API** (`api/`) | Protobuf contracts — source of truth for gRPC/REST + validation | `api/inventory/v1/`, `api/management/v1/`, `api/qan/v1/`, `api/backup/v1/`, `api/alerting/v1/`, `api/server/v1/`; `api/buf.gen.yaml`. Edit `.proto` only, regenerate with `make gen`. **Never edit** `*.pb.go`, `*.pb.gw.go`, `json/client/`. |
| **qan-api2** (`qan-api2/`) | QAN backend: ingest into ClickHouse, serve analytics | `qan-api2/main.go`, `qan-api2/db.go` (ClickHouse conn + migrations), `qan-api2/models/data_ingestion.go` (`MetricsBucket` batched writer), `qan-api2/models/reporter.go` (dynamic SQL report builder), `qan-api2/services/receiver/` (agent ingestion), `qan-api2/services/analytics/` (report queries), `qan-api2/migrations/sql/`. Raw SQL — no ORM. |
| **vmproxy** (`vmproxy/`) | VictoriaMetrics reverse proxy injecting LBAC label filters | `vmproxy/main.go`, `vmproxy/proxy/proxy.go` (reads `X-Proxy-Filter` header → injects `extra_filters[]`). Stateless; invalid header → 412. |
| **UI** (`ui/`) | React/TS frontend (runs inside the Grafana iframe) | `ui/apps/pmm/src/router.tsx` (routes), `ui/apps/pmm/src/Providers.tsx` (Auth/Settings/Theme contexts), `ui/apps/pmm/src/api/` (API clients), `ui/apps/pmm/src/hooks/` (React Query hooks per API domain), `ui/packages/shared/src/messenger.ts` (cross-frame Grafana comms). TanStack Query + MUI; Vite. |
| **Grafana dashboards** (`dashboards/dashboards/`) | Dashboard JSON by DB/domain | `dashboards/dashboards/{MySQL,MongoDB,PostgreSQL,OS,Valkey}/`, `.../Insight/`, `.../PMM Health/`, `.../Query Analytics/`; `dashboards/misc/cleanup-dash.py` normalizes exported JSON. |
| **QAN app** (`dashboards/pmm-app/`) | Grafana plugin bundling dashboards + the QAN panel | `dashboards/pmm-app/src/plugin.json` (manifest & dashboard includes), `dashboards/pmm-app/src/module.ts`, `dashboards/pmm-app/src/pmm-qan/panel/QueryAnalytics.tsx` (root QAN panel), `.../panel/components/` (Overview/Details/Filters/BarChart). |
| **API tests** (`api-tests/`) | Integration tests against a live server | `api-tests/init.go`, `api-tests/helpers.go`, `api-tests/{inventory,management,alerting,backup,server,user}/` (e.g. `management/mysql_test.go`). Uses the Swagger clients; needs a running server (`make env-up`). |
| **Build & packaging** (`build/`) | Docker, RPM/DEB, Ansible, AMI | `build/docker/server/Dockerfile.el9` + `entrypoint.sh`, `build/packages/{rpm,deb}/`, `build/ansible/roles/` (clickhouse, grafana, postgres, nginx, supervisord, dashboards…), `build/packer/pmm.json`. Ansible roles are the source of truth for server setup. |

**Adjacent code (outside the monorepo):** root causes often bottom out in the exporters (metrics collection), the Grafana fork (visualization), or PBM (MongoDB backups). GitHub homes: `percona/node_exporter`, `percona/mysqld_exporter`, `percona/mongodb_exporter`, `percona/postgres_exporter`, `percona/proxysql_exporter`, `percona/rds_exporter`, `percona/azure_metrics_exporter`, `percona/grafana`, `percona/percona-backup-mongodb`, `percona/pmm-qa`. A root cause can also sit in a non-Percona upstream org (`grafana/grafana`, `VictoriaMetrics/VictoriaMetrics`, `ClickHouse/ClickHouse`, `prometheus/*`, `mongodb/*`) — link where it actually lives.
