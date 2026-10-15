# Technical Specification: Load Test

- **Document:** SPEC-LOAD-TEST
- **Status:** Draft (design agreed; personal feature for the fork owner)
- **Target Components (planned):**
  - `apps/desktop/src-tauri/src/loadtest/` (engine, metrics, no Tauri types in the core)
  - `apps/desktop/src-tauri/src/commands/loadtest.rs`
  - `apps/desktop/src-tauri/src/storage/history.rs` (new tables)
  - `apps/desktop/src/stores/loadtest-store.ts`
  - `apps/desktop/src/components/loadtest/`
- **Vocabulary:** see `GLOSSARY.md` (Load Test, Virtual User, Scenario, Load Profile, Iteration, Failure Criteria, Think Time, Threshold, Request Timeout, Run Snapshot).

---

## 1. Goal

Let a developer generate concurrent traffic against an API from the desktop app, watch live metrics, keep past runs, and export a shareable report (PDF, JSON, CSV). Modeled on Postman Performance Testing, which runs locally by default.

Primary user: a developer checking their own API locally. PDF is a secondary artifact for stakeholders.

## 2. Non-goals (v1)

- Upstreaming, multi-user support, cross-team sharing.
- Cloud / distributed load generation.
- Open-loop arrival-rate model (X req/s regardless of latency).
- Pre-request / test scripts during a run.
- Weighted request mixes.
- Spike and Peak profiles.
- Body or header based Failure Criteria.
- CLI / CI mode.
- Run-to-run comparison page, per-request charts, logo upload.

## 3. Design decisions

| # | Decision |
|---|----------|
| Name | Feature is **Load Test**. Stress and spike are profiles inside it. |
| Scenario | Single request, folder, or collection; requests run in order, looped per VU. |
| Model | Closed loop. A VU sends the next request after the previous response, plus optional Think Time (default 0). The UI must state this is not a fixed arrival rate. |
| Profiles | **Fixed** and **Ramp Up**. The VU scheduler is designed for ramps from the first slice. |
| Scripts | None in v1. Variables resolved once before the run. Cookies isolated per VU or off. |
| Metrics | Per request, plus Iteration rollup (iterations/sec, Iteration duration). |
| Failure Criteria | Whole-test setting. Editable success status ranges (default `200-399`), toggle for failed declarative `assert`, Request Timeout. A request's own `assert.status` overrides. |
| Request Timeout | Default 30 s. Own "timeout" bucket. VU then continues. |
| Thresholds | Optional: p95, p99, error rate, min RPS. Overall PASS/FAIL. |
| Safety | Soft warnings, not hard blocks. VU cap (default 500) and max duration (default 30 min) are editable settings. One-time confirm for any non-local target (not `localhost`, `127.0.0.1`, or private range). |
| Persistence | SQLite. Config, Run Snapshot, per-second series, summary. Keep last 50 runs; user can delete. |
| Snapshot | Config + request list with variables **unresolved** + environment name + OS, CPU count, app version. Never resolved values, tokens, or headers. |
| UI | Dedicated main-area tab; history list in side panel; started from "Run Load Test…" on a folder/collection context menu. |
| Live view | Full charts live. Same 1 s tick data as the saved series. One chart component for live view and PDF. |
| Charts | uPlot (subject to bundle-size and webview check). Fallback: Recharts. |
| Exports | PDF, JSON (full run), CSV (per-second series). |
| Scope | Desktop only, personal use. The engine may use `AppHandle` directly; no extractable-crate constraint. |

## 4. Engine

### 4.1 Execution
- One shared `reqwest::Client` with connection pooling.
- N tokio worker tasks, one per VU. Each loops over the Scenario until the deadline or cancel.
- Cancel via `tokio_util::sync::CancellationToken` (new dependency) or a `oneshot`.
- VU scheduler computes target VU count over time, so Fixed is a constant and Ramp Up is linear to a target.
- Request construction reuses existing building blocks (interpolation, auth, request builder). `run_single_request` in the Collection Runner is private and mutates shared state; the engine needs a lean, script-free path rather than calling it directly.

### 4.2 Metrics
- `hdrhistogram` (new dependency) for latency percentiles. No per-sample storage.
- Per-second buckets: count, errors by category, avg, p95, active VUs.
- Latency is measured per request from send to full response body.
- **Coordinated omission:** record intended vs actual start where applicable, and document what the closed-loop model can and cannot show.

### 4.3 Failure classification
Order of evaluation per response:
1. Transport error, then category `transport`.
2. Exceeds Request Timeout, then category `timeout`.
3. Status outside accepted ranges (unless request `assert.status` expects it), then category `status`.
4. Failed declarative `assert` (if toggle on), then category `assertion`.

### 4.4 Events (to frontend)
- `loadtest:tick`, once per second: active VUs, requests, RPS, errors, current p50/p95/p99.
- `loadtest:complete`: final summary and verdict.
- Core engine emits through a trait/channel; the Tauri layer adapts it to events.

## 5. Data model (SQLite, in `history.db`)

Sketch only; final schema is decided in M3.

- `load_test_runs`: id, started_at, duration_s, profile, vus, status, verdict, title, notes, config_json, snapshot_json, summary_json.
- `load_test_series`: run_id, second, vus, requests, errors_by_category, avg_ms, p50, p95, p99.
- `load_test_request_stats`: run_id, request_key, count, errors, avg, p95, max.

Retention: prune to the last 50 runs on insert.

Trade-off: only per-second aggregates are stored, so new percentiles cannot be recomputed from raw samples later.

## 6. Report

### 6.1 PDF contents (in order)
1. Title, run date, duration, app version, PASS/FAIL (if Thresholds set).
2. Configuration: profile, VUs, duration, think time, timeout, Failure Criteria, Thresholds with actual vs target.
3. Summary: total requests, RPS, error rate, avg / p50 / p90 / p95 / p99.
4. Charts: latency, RPS, VUs, errors over time.
5. Per-request table: count, error %, avg, p95, max.
6. Error breakdown: by status code and by category.
7. Environment note: OS, CPU, and a disclaimer that results reflect the machine and network used.

### 6.2 Approach
All PDF generation sits behind a single `exportPdf(run)` function so the implementation can be swapped without touching callers.

- **Decided:** webview print-to-PDF (WebView2). The owner develops and tests on Windows only, so this is final for now. It is the cheapest route.
- Not verified on macOS or Linux. If a second OS is ever used, switch to frontend `jsPDF` with native text/tables and charts as PNG, since webview printing is unreliable on Linux WebKitGTK and prompts on macOS.
- `html2canvas` rejected (raster text). Rust PDF crates rejected (more work, same result).
- The release workflow builds all platforms, but this does not constrain the feature.

### 6.3 Redaction and branding
- Editable report title and notes, stored with the run.
- Redaction on by default: query strings stripped, host shown. Export-time toggle shows full URLs.
- Never include resolved variables, tokens, or headers.

## 7. Milestones

| M | Scope | Verified by |
|---|-------|-------------|
| M1 Engine | Fixed profile, closed loop, Failure Criteria, timeouts, cancel, histogram + per-second metrics | Rust tests only |
| M2 Run UI | Config form, live dashboard, Ramp Up, safety limits, confirm dialog | Manual + store tests |
| M3 Persistence | SQLite history, snapshots, delete, 50-run cap, JSON/CSV export | Rust + store tests |
| M4 Thresholds + PDF | Thresholds, verdict, PDF report | Spike, then manual |

Each milestone is shippable alone. Order may change freely since there are no upstream PRs.

## 8. Testing

1. **Unit:** percentile math against known inputs; per-second bucketing; failure classifier.
2. **Integration:** local `axum` server (already a dependency) with fixed latency, set error rate, hang (timeouts), status mix. Assert measured p95 and error rate within tolerance.
3. **Accuracy:** server stalls ~2 s mid-run; reported max latency and error count must reflect the stall.
4. Timing-sensitive tests use loose tolerances and are marked so CI noise does not flake the pipeline.

## 9. Branching

- Personal feature for the fork owner only. It will not be upstreamed.
- Code, spec, glossary and ADRs all live on `feat/load-test`, branched from `main` (not `upstream-sync`), then merged into `main`.
- The `AGENTS.md` rule that `feat/*` branches start from upstream and exclude internal docs applies only to upstream PRs, so it does not apply here.

## 10. ADRs to write

- Persisted per-second aggregates instead of raw samples (hard to reverse; limits later recomputation).
- No ADR for the PDF approach: it is isolated behind `exportPdf` and cheap to change.

## 11. Open items

- Target platform is Windows only (owner's test machine). macOS and Linux behavior of the PDF export is untested and unsupported.
- uPlot bundle size / webview check.
- Exact private-range definition for the confirm dialog.
- Whether cookies are isolated per VU or off by default.
- Final SQLite schema.
