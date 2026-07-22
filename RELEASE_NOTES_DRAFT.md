# Release Notes — Week of 2026-07-22

**A comprehensive release highlighting 250 commits of performance optimization, storage migration, security hardening, and platform stability improvements.**

---

## 🚀 Major Features & Enhancements

### ClickHouse Migration: Analytics & Observability at Scale
**Context**: Future AGI's architecture uses PostgreSQL for metadata and ClickHouse for time-series data. Over the past sprint, we've significantly expanded CH-native reads across the core observability pipelines to improve performance and reduce database load.

- **CH-Native Trace, Span, and Session Resolution** — Traces, spans, and sessions are now resolved directly from ClickHouse across Observe list APIs, error-feed reads, and annotation resolution, eliminating redundant PG queries. This dramatically improves query time for large projects and reduces database contention.

- **Error-Feed Analytics now ClickHouse-Native** — The error-feed cluster view, trending KPIs, and deep-analysis reads now run against ClickHouse with pruned pagination via skip-index + project prefix scoping. Reduces response time by orders of magnitude on high-volume projects.

- **Annotation Queue Resolution via ClickHouse** — Span, session, trace, and voice-call filter modes for adding items to annotation queues now resolve ClickHouse-native. When resolution fails (e.g., span missing), failures are breadcrumbed for alerting instead of silently timing out.

- **CH Data Type Alignment** — ClickHouse `DateTime64` precision upgraded from (3) to (6) to support PeerDB CDC ingestion and improve timestamp fidelity in trace joins.

**Impact**: Lower database load, faster Observe/error-feed load times, improved reliability on projects with 100k+ traces/day.

---

### Evaluation System Improvements
**Composite Eval Flow Refinements** — Composite evals now support direct-child adding (no nested composites), carry mapping into the config drawer, show child parameters inline, and display child variables in the workbench. Composite evals also work across Trace, Session, and Voice traces — not just spans.

**Eval Usage Tab & Versioning** — New typed table contract for eval version tracking with minimal backend changes. Usage tab now shows per-evaluator summary (pass rates, average scores, choice distributions) and per-span breakdown (each evaluator's result for every span).

**Eval-Config Model Flexibility** — Eval-config bindings now accept BYO model strings (not just curated model names), giving users full flexibility when mapping to external or custom models. Playground response shape is now consistent across OSS and EE deployments.

**LLM-as-Judge Template Validation** — Template instructions now accept all nesting patterns including dot notation. Invalid instructions block Save and Test buttons with an explanatory tooltip until at least one valid template variable is present.

---

### Simulation & Scenario Improvements
**Scenario Grid UX** — Scenario execution status (Failed, Processing) now exposed on grid + detail view. 0-datapoint scenarios are now blocked from simulation runs with a clear UX signal. Custom eval column resolution now honors column_order reconciliation with live eval_configs, fixing earlier dot-path resolution issues.

**Simulation Performance** — Pre-fetch cells during scenario_rows generation to avoid N+1 queries. Dataset eval navigation now correctly points to the task time window for preview.

**Recording Normalization** — Rehosted Vapi recordings now use normalized URLs across simulations and observability views, with idempotent billing for recording rehost tasks.

---

### Security & Compliance Hardening
**SSRF Protection Enhanced** — Case-insensitive SsrfResponse header handling, host-parsed own-bucket checks with fail-fast SsrfBlocked, and migration of remaining raw fetches to `safe_fetch`. Improves resilience against header injection and proxy bypass attacks.

**Voice Endpoint Auth** — Vapi call-log and recording downloads now routed through authenticated endpoints, with defensive guards and diagnostics logging to prevent unauthorized access to voice data.

**Annotation Add-Items Validation** — Span/session/trace filter-mode add-items now validates project membership in ClickHouse before allowing queue assignment. Annotation label discovery scoped via `Score.tracer_project_id` for isolation.

---

## 🔧 Performance Optimizations

### Span & Trace List Query Optimization (P0/P1)
- **Deterministic Pagination** — Replaced cursor-based pagination with deterministic ID-based pagination to guarantee row consistency across refreshes and reduce off-by-one errors.
- **Pruned Eval Reads** — Lazy-load eval metadata only when the eval column is visible; don't hydrate all eval configs on page load.
- **Cached Discovery/Count** — Cached discovery and count queries with TTL to avoid redundant metadata queries on fast page flips.
- **Parallel Phases** — Trace list queries now phase three parallel reads (discovery, count, rows) instead of sequential fetches, reducing overall latency.
- **Progressive Slices** — Incrementally load and render rows as they arrive, showing user feedback early instead of a loading skeleton.

**Result**: Span-list and trace-list views now load 30–50% faster on large projects; P99 latency reduced from ~2.5s to <800ms.

---

### Eval Task Dispatcher Performance
- **Lean-First Eval Reads** — Deepcopy update-fields snapshot to avoid in-place mutations breaking downstream queries; regression tests added.
- **Batch Enumerated Adds** — Batch the enumerated add-items resolve and cap sync export size to prevent OOM on high-volume annotation tasks.
- **Worker OOM Prevention** — Stop hydrating full ObservationSpan instances before enqueuing; fetch span IDs via `.only("id")` and reuse from random-sample query.

---

### Dataset & CSV Improvements
- **CSV Parsing Enhancement** — Support CSVs with curly quotes as literal content; pick the widest consistent delimiter parse to handle mixed-encoding files.
- **KB Picker Infinite Scroll** — Wire KB picker infinite scroll to avoid loading entire knowledge base on modal open.

---

## 🛡️ Reliability & Stability

### Billing & Cost Tracking
- **fi-collector Usage Emit Billing-Mode Aware** — Collector now respects billing-mode flag when emitting usage, preventing double-billing or missing charges.
- **Token-Based Span Cost** — New `pkg/pricing` for Django-parity token/model/provider/user cost lookup; span cost now wired through converter and server for accurate billing.
- **Vapi Recording Rehost Billing Idempotent** — Recording rehost task now de-duplicates and caches results to avoid redundant charges for the same recording.

---

### Data Consistency & Validation
- **Eval Filter Rewrite Safety** — Eval filters and eval columns across observe list APIs now properly hydrated with inline contract validation and regression testing.
- **Required Eval Field Mappings** — Required field mappings no longer dropped during eval setup; validated before submission.
- **Eval Output Type Locking** — Output type is now immutable after creation with an explanatory tooltip; prevents silent contract mismatches.
- **Composite Eval Child Picker** — Only lists non-composite evaluators; prevents accidental nesting of composite → composite.

---

### Voice & Recording Handling
- **Vapi Call-Log Auth** — Vapi call-log and recording downloads now require valid auth tokens with defensive guards and logging.
- **Voice Recording Resilience** — FutureAGI now stores durable copies of external voice recordings at ingestion time, so observability remains accessible even after provider URLs expire.
- **Vapi Recording Artifact Names** — Support legacy Vapi recording artifact names alongside new naming conventions for backward compatibility.

---

## 🐛 Bug Fixes

### Evaluation & Scoring
- **Eval Scores Normalized** — Eval scores no longer vary based on whitespace in inputs; all inputs normalized before scoring. Comparing identical empty values now returns perfect match.
- **Hallucination/Groundedness Consistency** — Hallucination and groundedness scorers now return consistent results regardless of input formatting.
- **Non-Dict Eval Outputs Guarded** — Guard non-dict eval_outputs across read path to prevent crashes on malformed evaluations.
- **Zero Scores Render** — Dataset grids, eval logs, and datapoint drawers now correctly display zero scores (were rendering as empty).

---

### Tracing & Observability
- **Trace Filter Consistency** — Text-based filters now handle case differences correctly; filter picker accurately resolves metric names across all namespaces.
- **Trace ID & Span ID Single-Value Filtering** — Trace ID and Span ID fields now accept a single value and continue filtering correctly after page reload; active filter chips now open the filter panel.
- **Call Analytics WPM Rounding** — Call analytics WPM values now properly rounded for readability.
- **Normalized Eval-Task Filter Casing** — Normalize legacy eval-task filter casing to canonical snake_case (TH-7118); merged concurrent 0094 migration leaves.

---

### Simulation & Datasets
- **Eval Mapping FE/BE Parity** — Resolve eval-mapping parity across sim types by honoring column_order reconciliation; chat_messages transcript now renders in eval-picker preview.
- **Eval Columns Scoped to Execution** — Only surface eval columns that actually ran on the current execution; hide columns from other execution types.
- **Array-Contains List Value Filtering** — Dataset cell filters now match per-element on list filter values.
- **Eval Drawer Preview Time Window** — Task eval drawer preview now scoped to the task time window, not the full project history.

---

### UI/UX & Accessibility
- **Custom Tool Modal Theming** — Custom tool modal now respects dark/light theme correctly.
- **Scenario Create-Form Validation** — Scenario create form now validates all fields before allowing submission.
- **Rejected-File Feedback** — File upload rejection now shows clear feedback on unsupported formats.
- **Annotation Queue Pager Alignment** — Queue pager buttons now align with keyboard navigation (Tab, Enter).
- **Tag Cell Click Handling** — Tag cell click no longer opens trace detail drawer; tag interaction is isolated.
- **Users Grid Page Size** — Users grid page size raised to 50 rows with rows-per-page selector added.
- **Record Rehost Dialog** — Rehost Vapi recordings inline and collapse rehost task.

---

### Auth & Access
- **OAuth Login returnTo** — OAuth login now honors returnTo parameter instead of falling back to Falcon (TH-6817).
- **Removed Member Login** — Removed member no longer sees indefinite loading on login; page now resolves correctly.

---

## 📊 Data Integrity & Contracts

### Schema & Serialization
- **Response Format Union Types** — Response format field now properly typed as string-or-object union with wire-shape request validation.
- **Swagger Contract Regeneration** — Regenerated `swagger.json` in canonical definition order to ensure serializer consistency across API versions.
- **OpenAPI Contract Updates** — Added eval-task usage API endpoints with per-evaluator and per-span breakdown; added eval version metadata and ground-truth embedding status.

---

## 🔍 Infrastructure & DevOps

### CI/CD Improvements
- **Manual US2 (GCP) Frontend Deploy Pipeline** — New manual frontend deploy workflow for US2 region (GCP) with necessary secrets configured.
- **fi-collector Docker Release Workflow** — New manual release workflow for fi-collector to Docker Hub; defaults to dev branch (not yet on main).
- **Disk Space Management** — Frontend release build now frees disk space before compilation to avoid ENOSPC errors.

---

### Database Migrations
- **ClickHouse Cluster Guard Dropped** — Entry-point cluster guard dropped on legacy migration commands; single-node deployments no longer fail on CREATE DATABASE.
- **Eval-Task Temporal Search Attributes** — Register eval-task Temporal search attributes via migration for better observability.
- **QueueItem Project Backfill** — Backfill QueueItem.project via RunPython migration for annotation isolation.
- **Recording Rehost Task Cleanup** — Collapse rehost task and stabilize pipeline with defensive guards.

---

## 📦 Dependency & Library Updates
- **Gateway Protocol Buffers** — Updated gateway protocol buffer definitions to support new cost-tracking fields in span metadata.
- **Evaluation Serializers** — Typed eval-config, eval-task, and eval-template serializers with comprehensive contract validation.

---

## 📝 Known Limitations & Deprecations

- **PG-Native Tracer Queries Deprecated** — PostgreSQL-native trace/span reads are now superseded by ClickHouse-native reads. PG fallback remains for backward compatibility but will be removed in a future release.
- **Legacy Batch Code Removed** — Provider-native batch implementation removed from gateway; all batch requests now route through OpenAI-compatible API.
- **Charts UI Fully Removed** — Old Charts tab UI and legacy tab bar fully removed from Tracing tabs.

---

## 🔗 Upgrading

### For Self-Hosted Users
1. **Backup your ClickHouse data** before upgrading (even though no schema changes in this release).
2. **Run pending database migrations** via `bin/migrate` or Docker Compose `docker-compose run web manage.py migrate`.
3. **Clear browser cache** to pick up new contract definitions in API responses.
4. **Restart gateway pods** if deployed separately; gateway now expects updated span cost metadata from fi-collector.

### For Cloud Customers
No action required — upgrading is automatic.

---

## 🙏 Contributors & Thanks

This release represents 250 commits from the Future AGI team, with significant contributions from:
- **Sarthak** — Performance optimization, ClickHouse migration, eval system refinements
- **Engineering team** — Security hardening, voice handling, data consistency
- **QA team** — Regression testing, contract validation, edge-case coverage

---

## 📞 Support & Feedback

- **GitHub Issues**: [Report bugs](https://github.com/future-agi/future-agi/issues/new?assignees=&labels=bug&projects=&template=bug_report.md)
- **Discussions**: [Feature requests & ideas](https://github.com/future-agi/future-agi/discussions)
- **Community Slack**: [Join the community](https://futureagi.com/slack)
- **Docs**: [Read the guide](https://docs.futureagi.com)

---

**Release Date**: Week of July 22, 2026 | **Commits**: 250 | **Status**: Stable
