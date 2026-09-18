# Cloud Compass — Agent Guide

Multi-tenant, multi-cloud (AWS + Azure + GCP) cloud operations platform: MCP + RAG + Agent + Refine (shadcn/ui).

**Product:** Cloud Compass (formerly Cloud Cost Compass).
**Stack:** Refine + shadcn/ui + Vite + TypeScript (UI), LangGraph/LangChain (agent), Minimax M2.7 (LLM), FastMCP (tools), Qdrant (RAG vectors), PostgreSQL (state), native cloud SDKs (boto3, azure-mgmt, google-cloud-*).

> Repo directory and K8s namespace still use the historical `cloud-cost-compass` name; the **product** name is **Cloud Compass** (K8s namespace: `cloud-cost-compass` for now — see Phase 1 tracker item **F1.7**). Image registry prefix is also `cloud-cost-compass/*`.

---

## 1. Phase Plan

> The execution order is now provider-sequential: **AWS → GCP → Azure**. The detailed current-state inventory, subphases, dependencies, and acceptance gates are in [`docs/ROADMAP.md`](docs/ROADMAP.md). Existing `F1.x`/`F2.x`/`F3.x` IDs remain domain-backlog references.

| Phase | Scope | Status |
|---|---|---|
| R0 | Foundation repair: contracts, identity, builds, auth, schema, Vault, Kind | Pending |
| R1 | AWS Foundation: onboarding, cost, inventory, RAG chat | Pending |
| R2 | AWS Security + FinOps | Pending |
| R3 | AWS SCA + Compliance + Alerts; AWS beta | Pending |
| R4 | GCP full-domain parity | Pending |
| R5 | Azure full-domain parity | Pending |
| R6 | Unified multi-cloud GA and production hardening | Pending |

### Legacy domain-phase exit criteria

> These describe the final domain outcomes retained by the `F*` backlog. Delivery now occurs through R0–R6 in `docs/ROADMAP.md`, reaching each outcome for AWS first, then GCP, then Azure.

**Foundation:** Log in as tenant A, see provider spend on the Costs page; see normalized inventory on the Inventory page; ask the chat a provider-specific cost question and get a grounded answer with citations.

**Security + FinOps:** Security shows aggregated CSPM findings, drill-down works, chat answers exposure and remediation questions with KB citations, and FinOps shows rightsizing plus commitment coverage.

**SCA + Compliance + Alerts:** Upload a CycloneDX SBOM and see affected CVEs with KEV highlighted; run a provider CIS assessment and generate evidence; fire a Slack alert for a deterministic cost anomaly.

---

## 2. Decisions Locked (defaults from kickoff)

| # | Question | Decision | Alternative considered |
|---|---|---|---|
| D1 | MCP server topology | **Single FastMCP server**, namespaced tools | Per-domain or per-cloud servers (more YAML, harder ops) |
| D2 | AuthZ enforcement | **Roles checked in BOTH UI and MCP tool wrappers** (defense in depth) | UI-only, or MCP-only |
| D3 | Cloud credentials (v1) | **Static keys/secrets in Vault** (AWS access key, Azure SP secret, GCP SA JSON key) | Cross-account role assumption, Workload Identity Federation (deferred to v2) |
| D4 | MCP write-actions (v1) | **Read-only tools**, remediation as runbook text | Direct remediation writes (safety risk; deferred) |
| D5 | Compliance frameworks (v1) | **CIS AWS / Azure / GCP + SOC2 CC subset** | HIPAA, PCI, ISO 27001 (out of scope for v1) |
| D6 | Cost snapshotting | **Daily persisted** `cost_history` (cron) **+ real-time on demand** from MCP | Real-time only (no trends) or persisted only (slow UX) |
| D7 | Multi-account per tenant | **Many AWS accounts, one Azure tenant, many GCP projects** | Strictly 1:1:1 (too restrictive) |
| D8 | Per-cloud region default | **Single default per provider per tenant, overrideable per request** | Per-resource region only (cluttered UX) |
| D9 | K8s namespace | **`cloud-cost-compass` for now** (F1.7 may rename to `cloud-compass`) | Rename immediately (disruptive; deferred) |
| D10 | Repo name | **Unchanged** — user will handle the rename | Rename to `cloud-compass` (we don't touch git remotes) |
| D11 | UI framework | **Refine + shadcn/ui** (Vite + TypeScript) | Streamlit (weak tables/streaming/RBAC), Gradio (notebook feel), Next.js (heaviest), Appsmith/Tooljet (low-code, less flexible) |
| D12 | Provider delivery order | **AWS first, then GCP, then Azure**; finish each provider release gate before production work on the next | All-cloud horizontal delivery (delays usable vertical releases) |

> All "Alternatives considered" entries are recorded here so future maintainers (and the agent) can revisit them. If a tradeoff is overturned, update this table AND the matching tracker item.

---

## 3. Domain → MCP Tool Map

| Domain | MCP tools |
|---|---|
| Cost | `cost.get_costs`, `cost.get_forecast` |
| FinOps | `finops.get_rightsizing`, `finops.get_reservation_coverage`, `finops.get_reservation_utilization`, `finops.get_idle_resources` |
| Inventory | `inventory.list_resources`, `inventory.get_tag_coverage`, `inventory.get_unused_resources` |
| Security | `security.list_findings`, `security.get_iam_anomalies`, `security.get_public_assets`, `security.get_encryption_status` |
| SCA | `sca.list_vulnerabilities`, `sca.get_sbom`, `sca.ingest_sbom`, `sca.sync_cve_feed` |
| Compliance | `compliance.list_frameworks`, `compliance.get_control_status`, `compliance.generate_evidence` |
| Alerts | `alerts.list_rules`, `alerts.create_rule`, `alerts.delete_rule`, `alerts.list_events`, `alerts.test_channel` |
| Auth | `auth.whoami` |

---

## 4. Tracker — `F<N>.<M>` item format

> **Convention:** `F1.x` = Phase 1 item, `F2.x` = Phase 2, `F3.x` = Phase 3, `FX.x` = cross-cutting.
> **Status legend:** `[ ]` pending · `[~]` in progress · `[x]` done · `[!]` blocked · `[-]` cancelled / superseded.

### Phase 1 — Foundation

| ID | Task | Status | Owner | Notes |
|---|---|---|---|---|
| F1.1 | Rename product `Cloud Cost Compass` → `Cloud Compass` in `README.md` | `[x]` | agent | done in this commit |
| F1.2 | Rewrite `AGENTS.md` with phased plan + tracker (this file) | `[x]` | agent | done in this commit |
| F1.3 | Phase 1 SQL: `001_tenants_and_creds.sql`, `003_cost.sql`, `002_inventory.sql` | `[ ]` | agent | supersedes existing `001_initial_schema.sql`, `002_seed_tenants.sql` |
| F1.4 | Provider abstraction: `mcp-server/providers/{base,aws,gcp,azure,factory}.py` | `[ ]` | agent | `CloudProvider` protocol; AWS port from current `server.py`; implement per D12 |
| F1.5 | MCP tools `cost.*`, `inventory.*` with provider parity | `[ ]` | agent | namespaced; tenant + role injected from JWT; deliver AWS → GCP → Azure per D12 |
| F1.6a | Refine + shadcn/ui scaffold: Vite/TS/Tailwind, routing, providers, layout | `[~]` | agent | replaces Streamlit; multi-stage Docker build, nginx serve |
| F1.6b | OIDC + role-based route guards: Keycloak code flow, `viewer`/`operator`/`admin` | `[ ]` | agent | refine-auth provider; mirrors D2 (defense in depth) |
| F1.6c | Pages: Overview, Costs, Inventory, Chat, Settings (Phase 1 surface) | `[ ]` | agent | TanStack Table for inventory, Vercel AI SDK `useChat` for chat |
| F1.7 | Decide on K8s namespace rename `cloud-cost-compass` → `cloud-compass` | `[ ]` | human | D9 deferred; revisit after first multi-tenant deploy |
| F1.8 | Vault paths updated to `secret/tenants/{tid}/providers/{aws,azure,gcp}.json` | `[ ]` | agent | keep old `aws.json` path aliased during cutover |
| F1.9 | K8s manifests: namespace, Vault, Keycloak, Postgres, MCP, app, gateway, migrations, RAG, Qdrant | `[ ]` | agent | reconcile raw manifests and Helm; refresh image refs |
| F1.10 | `scripts/{setup-kind,deploy-eks}.sh` adapted for new namespace + images | `[ ]` | agent | |
| F1.11 | LangGraph agent skeleton: `classify_intent → plan → retrieve_context → execute_tools → synthesize → reflect` | `[ ]` | agent | tool registry mirrors MCP surface |
| F1.12 | End-to-end smoke: log in tenant A, see provider spend + inventory | `[ ]` | agent | deliver AWS gate first, then extend the same suite to GCP and Azure |

### Phase 2 — Security + FinOps

| ID | Task | Status | Owner | Notes |
|---|---|---|---|---|
| F2.1 | Migrations `004_security.sql` (findings, iam_principals), `007_recommendations.sql` (partial — finops only) | `[ ]` | agent | |
| F2.2 | MCP tools `finops.*` with provider parity | `[ ]` | agent | deliver AWS → GCP → Azure per D12 |
| F2.3 | MCP tools `security.*` with provider parity | `[ ]` | agent | Security Hub → SCC → Defender for Cloud per D12 |
| F2.4 | RAG collection `kb-{tid}-security` + router `/security_kb` | `[ ]` | agent | seed with CIS Benchmarks + provider hardening |
| F2.5 | Refine pages: Security, FinOps | `[ ]` | agent | severity donut, top control IDs, drill-down (TanStack Table + Recharts) |
| F2.6 | Daily `inventory-snapshot` CronJob (K8s `09-cronjobs.yaml` shape) | `[ ]` | agent | populates `resource_inventory` |
| F2.7 | Exit criterion: Security page + FinOps page work; chat cites KB | `[ ]` | agent | |

### Phase 3 — SCA + Compliance + Alerts

| ID | Task | Status | Owner | Notes |
|---|---|---|---|---|
| F3.1 | Migrations `005_sca.sql`, `006_compliance.sql`, `007_recommendations.sql` (rest), `008_alerts.sql`, `009_seed.sql` | `[ ]` | agent | |
| F3.2 | MCP tools `sca.*` | `[ ]` | agent | ECR / ACR / GAR image scan ingestion, SBOM normalize to PURL |
| F3.3 | MCP tools `compliance.*` | `[ ]` | agent | framework packs from `compliance/*.yaml` |
| F3.4 | MCP tools `alerts.*` | `[ ]` | agent | rules CRUD, test channel, list events |
| F3.5 | `alerts-service` (FastAPI :8002) + Slack channel | `[ ]` | agent | webhook rendered by Vault Agent |
| F3.6 | CronJobs: `cve-sync` (NVD/EPSS/KEV, daily), `anomaly-eval` (hourly), `compliance-evidence` (weekly) | `[ ]` | agent | |
| F3.7 | RAG endpoints `/cve`, `/compliance_kb`; collections `cve-{tid}`, `kb-{tid}-compliance` | `[ ]` | agent | |
| F3.8 | Compliance framework packs in `compliance/`: CIS AWS / Azure / GCP, SOC2 CC | `[ ]` | agent | YAML, registered at startup |
| F3.9 | Refine pages: SCA, Compliance, Alerts | `[ ]` | agent | SBOM upload (React Dropzone), KEV badge, control matrix |
| F3.10 | Exit criterion: SBOM upload → CVE list with KEV; CIS scan → evidence bundle; anomaly → Slack | `[ ]` | agent | |

### UI Stack (locked)

- **Framework**: [Refine](https://refine.dev) (React, headless on K8s)
- **Component library**: [shadcn/ui](https://ui.shadcn.com) (Radix + Tailwind)
- **Build**: Vite + TypeScript
- **Data**: Refine data providers wrapping MCP / RAG / Alerts REST endpoints
- **Chat**: Vercel AI SDK `useChat` over SSE to the LangGraph agent
- **Auth**: Refine auth provider against Keycloak OIDC (code flow, PKCE)
- **Tables**: TanStack Table (via Refine `useTable`)
- **Charts**: Recharts
- **Serve**: nginx (multi-stage Docker build, SPA + `/api/*` reverse proxy)

> Alternatives considered: Streamlit (weak tables/streaming/RBAC), Gradio (notebook feel), Next.js (heaviest, full custom), Appsmith/Tooljet (low-code, less flexible). Refine wins on the balance of structure + flexibility for a multi-domain ops console with a serious chat surface.

### Cross-cutting

| ID | Task | Status | Owner | Notes |
|---|---|---|---|---|
| FX.1 | OpenTelemetry SDK in every service (OTLP exporter to future collector) | `[ ]` | agent | structured JSON logs with `tenant_id` field |
| FX.2 | Envelope encryption for `tenant_credentials.encrypted_blob` (DEK per row, KEK = `ENCRYPTION_KEY`) | `[ ]` | agent | upgrade from current single-key blob |
| FX.3 | Automated quality gates and `pytest` per service with mocked `CloudProvider` | `[ ]` | agent | same commands locally and in CI; no live cloud credentials required |
| FX.4 | `docs/ARCHITECTURE.md`, `docs/SECURITY.md`, `docs/RUNBOOKS.md` | `[ ]` | agent | replaces `docs/ARCHITECTURE_PLAN.md` |
| FX.5 | `bootstrap-tenant.sh`, `seed-vault.sh` scripts | `[ ]` | agent | seed minimal tenant + creds for dev |
| FX.6 | Current-state inventory + provider-sequential completion roadmap | `[x]` | agent | `docs/ROADMAP.md`; execution order AWS → GCP → Azure |

---

## 5. Agent Invocation Tracker

> This is the running log of agent sessions working on Cloud Compass. Each entry is added by the agent when it starts and updates an item.
> **Format:** `YYYY-MM-DD HH:MM | item | status change | summary`.

| When | Item | Δ | Summary |
|---|---|---|---|
| 2026-06-09 00:00 | F1.1 | `[ ] → [x]` | Renamed product to Cloud Compass in `README.md` |
| 2026-06-09 00:00 | F1.2 | `[ ] → [x]` | Rewrote `AGENTS.md` with phased plan, locked decisions, and tracker |
| 2026-06-09 00:00 | FX.4 | `[ ] → [~]` | Started architecture doc rewrite; will replace `docs/ARCHITECTURE_PLAN.md` |
| 2026-06-09 12:00 | D11 (new) | `Streamlit → Refine + shadcn/ui` | Locked UI: Refine (React, Vite, TS) + shadcn/ui (Radix + Tailwind). Reason: better tables (TanStack Table), streaming chat (Vercel AI SDK), OIDC/RBAC out of the box. Alternatives considered: Streamlit, Gradio, Next.js, Appsmith, Tooljet. |
| 2026-06-09 12:00 | F1.6 | `[ ] → split` | Split into F1.6a (scaffold), F1.6b (OIDC + guards), F1.6c (Phase 1 pages) |
| 2026-06-09 12:00 | F1.6a | `[ ] → [~]` | Scaffolding Refine + Vite + TS + Tailwind + shadcn/ui in `app/` |
| 2026-09-18 22:56 | FX.6 | `[ ] → [~]` | Started repository-backed current-state inventory and completion roadmap |
| 2026-09-18 22:56 | D12 (new) | `all-cloud horizontal → AWS → GCP → Azure` | Locked provider-sequential delivery order |
| 2026-09-18 22:56 | FX.6 | `[~] → [x]` | Added complete R0–R6 roadmap with subphases, dependencies, and exit gates |

> When you (the agent) start a new task, **append a row** here with the timestamp, the `F<n>.<m>` item, the new status, and a one-line summary. When the task completes, append a second row flipping the status to `[x]`.

---

## 6. Multi-Tenancy Rules (DO NOT VIOLATE)

- `tenant_id` MUST come from verified OIDC identity context — never from request body, query string, or unverified headers. R0.1 must decide a dedicated tenant claim versus server-side membership lookup; `sub` remains the user identifier.
- Every Postgres query in `app/`, `mcp-server/`, `rag-service/`, `alerts-service/` MUST include `WHERE tenant_id = %s`.
- Every Qdrant call MUST target a tenant-prefixed collection (`rag-{tid}`, `kb-{tid}-*`, `cve-{tid}`).
- Every Vault read for cloud creds MUST be scoped to `secret/tenants/{tenant_id}/providers/...`.
- The MCP tool server MUST inject `tenant_id` and `role` from the verified token; it MUST NOT trust `tenant_id` from the request payload.
- Role checks MUST happen in both the UI (page guard) and the MCP tool wrapper (server-side enforcement).

---

## 7. Secret Hygiene

- **No Kubernetes `Secret` objects** for application secrets (cloud creds, API keys, Slack webhooks). All go through Vault Agent sidecars rendering to `emptyDir`.
- `Secret` objects ARE allowed for ephemeral bootstrap only (e.g., `00-secrets-bootstrap.yaml` feeding the Vault init job) and for TLS cert material issued by cert-manager.
- Never commit real credentials. The `placeholder` values in manifests are explicit and meant to be replaced by the Vault seed job.
- `ENCRYPTION_KEY` is a single key today; envelope encryption (FX.2) upgrades this.

---

## 8. Non-Obvious Commands

```bash
# Local Kind setup
./scripts/setup-kind.sh

# EKS deployment
./scripts/deploy-eks.sh

# Build images
docker build -t cloud-cost-compass/app:latest -f app/Dockerfile app/
docker build -t cloud-cost-compass/mcp-server:latest -f mcp-server/Dockerfile mcp-server/
docker build -t cloud-cost-compass/rag-service:latest -f rag-service/Dockerfile rag-service/

# Kind image load
kind load docker-image cloud-cost-compass/app:latest --name cloud-cost-compass
kind load docker-image cloud-cost-compass/mcp-server:latest --name cloud-cost-compass
kind load docker-image cloud-cost-compass/rag-service:latest --name cloud-cost-compass
kind load docker-image qdrant/qdrant:v1.7.4 --name cloud-cost-compass
kind load docker-image hashicorp/vault:1.16 --name cloud-cost-compass
```

## 9. Architecture (high level)

```
Browser → Envoy Gateway → Refine + shadcn/ui (8080) + LangGraph agent (SSE)
                              │
              ┌───────────────┼───────────────┐
              │               │               │
        MCP Server (8000) RAG Service (8001) Alerts Service (8002, P3)
              │               │               │
   AWS / Azure / GCP SDKs  Qdrant + Postgres  Slack webhook
                              │
                  PostgreSQL (history, inventory, findings, SBOM, compliance, alerts)
                  Vault Agent sidecar in every pod → emptyDir → /etc/secrets
```

## 10. K8s Service Inventory

| Service | Port | Phase | Notes |
|---|---|---|---|
| vault | 8200 | P1 | Dev mode, no persistence |
| keycloak | 8080 | P1 | Dev mode; realm `cloud-compass` |
| postgres | 5432 | P1 | No persistence (dev) |
| mcp-server | 8000 | P1 | FastMCP, all tools |
| streamlit | — | — | **Removed** — replaced by Refine + shadcn/ui (D11) |
| app (Refine) | 8080 | P1 | Multi-page dashboard; nginx serves static + proxies `/api/*` to backends |
| rag-service | 8001 | P1 | FastAPI |
| qdrant | 6333/6334 | P1 | gRPC/HTTP, persistent |
| alerts-service | 8002 | P3 | FastAPI; rules + channels |
| cronjobs | — | P3 | snapshot, cve-sync, anomaly-eval, compliance-evidence |

## 11. RAG

- `tenant_id` scoped chunking and retrieval.
- Sources: cloud billing docs (scraped) + tenant-uploaded cost/runbook/SBOM reports + provider hardening guides + CVE corpus.
- Embedding: Minimax `embo` (384 dim, DOT similarity).
- Chunking: 512-char fixed, 50-char overlap.
- Collections: `rag-{tid}`, `kb-{tid}-security`, `kb-{tid}-compliance`, `cve-{tid}`.

## 12. Quality baseline

The UI defines npm build and TypeScript-check scripts, but dependencies are not locked and no automated test suite exists yet. Python services have no tests, lint, or type-check configuration. FX.3 and roadmap R0.2 establish reproducible local and CI quality gates.
