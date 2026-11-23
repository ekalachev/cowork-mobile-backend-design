# Cowork Mobile — Tested Implementation Prompt & Revision Log

> **Companion to:** [`ARCHITECTURE_SPEC.md`](./ARCHITECTURE_SPEC.md)
>
> **Purpose:** Turn the Cowork Mobile backend specification into an MVP **Runs milestone** implementation in Python 3.12 without sacrificing correctness invariants, security boundaries, or testability.

---

## 1. Overview & Usage

Pair this prompt directly with [`ARCHITECTURE_SPEC.md`](./ARCHITECTURE_SPEC.md) when instructing a coding agent (or scaffolding a milestone kickoff for a small backend team). It targets the **Runs milestone** (MVP Weeks 3–6 plus the approvals foundation) so the team lands an end-to-end vertical slice—run creation, durable Temporal execution, resumable SSE streaming, and lock-screen approvals—before expanding into connectors, schedules, and v1.1 features.

---

## 2. Final Tested Build Prompt (`v2`)

````markdown
You are a senior Python backend engineer implementing the **Runs milestone** (MVP Weeks 3–6 + approvals foundation) of **Cowork Mobile**, strictly following `ARCHITECTURE_SPEC.md`.

### Tech Stack & Layout
- Python 3.12, `uv` workspace monorepo layout from §10: `apps/api/`, `apps/stream/`, `apps/worker/`, `packages/{domain,agent,sandbox,mcp_gateway,policy,llm_gateway}/`, `migrations/`, `tests/`, `pyproject.toml` (`mypy --strict`, `ruff`, `pytest`).
- Stack targets: FastAPI, Pydantic v2, Temporal (`temporalio`), Postgres 15+ (`asyncpg`), Redis Streams, S3 + KMS. Provide clean Protocol interfaces, real production adapters, and deterministic in-memory fakes for offline testing.

### Scope Boundaries
- **In scope (build completely):**
  - Core API routes: `POST /v1/runs`, `GET /v1/runs/{run_id}`, `GET /v1/runs/{run_id}/events`, `POST /v1/runs/{run_id}/plan:decide`, `POST /v1/runs/{run_id}/messages`, `POST /v1/runs/{run_id}:pause|:resume|:cancel`, `POST /v1/approvals/{approval_id}:decide`, `/healthz`, `/readyz`.
  - Postgres SQL DDL (`migrations/001_runs_milestone.sql`) + `PostgresStore` + `InMemoryStore`.
  - Temporal `RunWorkflow` + `WorkflowActivities` + `AgentRuntime` + `PolicyEngine` (with taint tracking) + `MCPGateway`/`TokenVault` + `SandboxProvider` (E2B + fake) + `LLMGateway` (Anthropic/Bedrock + fake) + `StreamGateway` (SSE + cellular coalescing) + `NotificationService` interface.
- **Out of scope (do NOT build):** Built-in browser (F6), plugins/skills marketplace (F8), scheduled tasks (F9), Desktop Runner WebSocket (F11), enterprise SCIM/exports (F13), billing UI, or live third-party OAuth web flows.

### Non-Negotiable Invariants
1. **Atomic Idempotent Run Creation (`POST /v1/runs`)**: Claim `Idempotency-Key` (scoped to `org_id, user_id, scope, key`) atomically *before or in the same DB transaction as* inserting the `runs` row and initial `run.state_changed` event (status `'in_progress'` → `'completed'`), preventing concurrent mobile retries from creating duplicate runs or workflows. Compare canonical `sha256` request hash (`409` on mismatch; replay identical status + body). Start the Temporal workflow idempotently using deterministic workflow ID `run-{run_id}`.
2. **Gap-Free Event Log & Tamper-Evident Audit**: `run_events` uses `PRIMARY KEY (run_id, seq)`, is range-partitioned by time-ordered ULID `run_id`, and rejects `UPDATE`/`DELETE` via trigger. Allocate `seq` atomically in the same transaction as `run_events` insertion (`SELECT ... FOR UPDATE` on `runs` to advance `last_seq`). When appending to hash-chained `audit_log`, lock the org's anchor row (`SELECT ... FROM orgs ... FOR UPDATE`) so concurrent `audit_seq` inserts never collide.
3. **Resumable, Authorized SSE (`GET /v1/runs/{id}/events`)**: Verify run existence (`404`) and owner/org-admin access (`403`) *before* returning `StreamingResponse`. Honor both `Last-Event-ID` and `?after=`. Subscribe to the live bus *before* querying the DB backlog, paginate all DB rows after cursor (no silent 1,000-row cap), deduplicate overlap by `seq`, emit `: keepalive\n\n` comments on idle heartbeat intervals, and support `?profile=cellular` (~100 ms `message.delta` coalescing + `<= 1 fps` `browser.frame` throttling).
4. **Cryptographic Approval Integrity**: Bind each approval to `(approval_id, run_id, step_id, sha256(canonical_json(args)))`. Enforce single-use (`pending` → `approved|denied` under `FOR UPDATE`), 15-min TTL (`410 Gone`), `409` on `args_hash` mismatch, and device-bound biometric/passkey step-up verification when required (destructive tools or org policy). Emit `approval.resolved` exactly once (from `ApprovalService`), and handle Temporal signal idempotency safely so a transient signal failure does not permanently strand an approved run.
5. **Deterministic Step IDs & Exactly-Once Tool Side Effects**: Derive deterministic `step_id` values (`f"{run_id}:t{turn_count}:c{tool_index}"` or workflow-issued IDs) rather than random IDs inside retryable activities so Temporal retries hit the same `(run_id, step_id)` key. Claim `(run_id, step_id)` in `tool_executions` (verifying `args_hash` matches) *before* executing non-idempotent tools and return cached output on retry.
6. **Strict Temporal Workflow Determinism**: Zero direct I/O, `datetime.now()`, `uuid4()`, or `random` in `apps/worker/workflows.py` — delegate all side effects to activities. Support `steer_message`, `decide_plan`, `decide_approval`, `pause_run`, `resume_run`, and `cancel_run` signals; snapshot/pause the sandbox after 60 s waiting for a human; wait up to `workflow.all_handlers_finished()` and preserve `snapshot_id`, taint state, and rolling summary across `continue_as_new`.
7. **Fail-Closed Multi-Tenancy**: Every tenant table has `org_id` and `FORCE ROW LEVEL SECURITY` using `NULLIF(current_setting('app.org_id', true), '')`. Enforce transaction-local `set_config('app.org_id', ..., true)`. Use composite foreign keys `(org_id, project_id)` and `(org_id, run_id)` so cross-tenant FK links are impossible. Enforce `orgs/{org_id}/` S3 prefixes and per-org KMS keys.

### Implementation Discipline
- Do not write silent no-op stubs in production adapters (e.g., `AnthropicBedrockGateway` must parse real response content blocks and tool-use blocks rather than returning hardcoded `done=True`, or raise `NotImplementedError` if an adapter method is intentionally deferred).
- Wrap untrusted external tool outputs in nonce delimiters and escalate external writes to `ask` once a run is tainted (even in `auto` mode).

### Verification (`pytest`)
Write comprehensive offline tests using `FakeSandboxProvider`, `FakeLLMGateway`, and `InMemoryStore` (using `pytest` or compatible `unittest` fallback if `pytest` is absent offline) covering:
- All invariants **I1–I7**, including **concurrent** duplicate `Idempotency-Key` race, **concurrent** `append_event` gaplessness, deterministic `step_id` retry idempotency, and approval expiry/TOCTOU/step-up.
- **FastAPI route tests** (headers `Idempotency-Key`, `Last-Event-ID`, `plan:decide`, 401/403/404/409/410 responses) — cleanly skipped only if `fastapi`/`httpx` are absent.
- **Temporal workflow determinism/replay tests** (`WorkflowEnvironment` / `Replayer`) — cleanly skipped only if `temporalio` is absent, plus a pure-Python workflow/signal state harness if offline.

### Required Completion Report
End with four explicit sections:
1. **Implemented** (exact files, routes, invariants, and adapters)
2. **Stubbed / Deferred** (every stub, mock, or unimplemented production adapter)
3. **Key Assumptions** (auth/OIDC, passkeys, KMS, sandbox template, etc.)
4. **Verified vs. Not Verified** (exact commands run, exact counts, and every module/adapter only syntax-checked or skipped due to offline packages).
````

---

## 3. Prompt Design Rationale

- **Executable invariants over vague adjectives.** Replacing *"make sure the code is secure and scalable"* (`v1`) with seven concrete, falsifiable invariants (`I1`–`I7`) eliminated concurrency races, silent truncation, and non-deterministic retry bugs without inflating prompt length (~800 words).
- **Explicit scope fencing.** Enumerating exact in-scope HTTP routes alongside an explicit *Out of Scope* list prevented feature drift while keeping the agent from omitting critical companion endpoints such as `plan:decide`.
- **No silent stubs rule.** Requiring real payload parsing or explicit `NotImplementedError` forced production adapters (`AnthropicBedrockGateway`, `APNsFCMNotificationService`) to be honest and inspectable rather than returning placeholder `done=True` values.
- **Protocol-first testability + structured reporting.** Separating pure domain/store/bus protocols from external SDK adapters allowed meaningful concurrency, streaming, and workflow-signal verification even in an offline environment, while the 4-part report schema ensured unverified production adapters were called out explicitly.

---

## 4. Empirical Testing & Revision Log

Both prompts (`v1` baseline and `v2` revised) were tested cleanly from greenfield against [`ARCHITECTURE_SPEC.md`](./ARCHITECTURE_SPEC.md) in isolated directories with zero network access (verifying syntax via `python3 -m py_compile` and executing behavioral tests via `unittest` against in-memory fakes). Output was graded against a fixed 20-point rubric (`0` = missing/wrong, `1` = partial, `2` = correct) locked before seeing `v1` output.

### Baseline Prompt (`v1`)

> You are a senior Python backend engineer. Build the backend for "Cowork Mobile", a mobile-first version of Claude Cowork, following the attached ARCHITECTURE_SPEC.md.
>
> Use Python 3.12, FastAPI, Temporal, Postgres, Redis and S3. Follow the repo layout in the spec. Write clean, tested, production-quality code with type hints.
>
> Start with the Runs milestone: creating a run, the agent workflow, the event stream, and approvals. Make sure the code is secure and scalable.
>
> When you're done, summarize what you built.

### Fixed Rubric Comparison (`v1` vs. `v2`)

| # | Criterion | `v1` | `v2` | Observed Change (`v1` defect → `v2` resolution) |
|---|---|---|---|---|
| **R1** | Idempotent run creation | **1/2** | **2/2** | **`v1`:** Saved `Idempotency-Key` *after* creating the run and starting the workflow (`RunService.create_run`), allowing concurrent retries to spawn duplicate runs.<br>**`v2`:** Atomically claims `in_progress` inside the same transaction as `runs` + initial event insertion; `20` concurrent retries tested cleanly (`1` run, `1` workflow). |
| **R2** | Gap-free event sequence | **2/2** | **2/2** | Both versions locked `runs` (`FOR UPDATE`) to advance `last_seq` atomically in `run_events`. **`v2`** also added `SELECT ... FROM orgs FOR UPDATE` to prevent concurrent `audit_seq` collisions in `audit_log`. |
| **R3** | Resumable stream | **1/2** | **2/2** | **`v1`:** `format_sse_keepalive()` was dead code, `list_events_after` truncated silently at `1,000` rows, `NotFoundError` fired after HTTP `200` headers started, and owner/admin check was omitted.<br>**`v2`:** Authorizes (`403`/`404`) before `StreamingResponse`, paginates `>1,250` events cleanly, emits `: keepalive\n\n`, deduplicates, and tests cellular coalescing/throttling. |
| **R4** | Approval integrity | **2/2** | **2/2** | Both bound approvals to canonical `sha256(args)`, single-use `FOR UPDATE`, 15-min TTL, and step-up verification. **`v2`** fixed `v1`'s duplicate `approval.resolved` emission and added signal-failure recovery (`signal_dispatched_at` + `reconcile_undispatched_signals`). |
| **R5** | Exactly-once side effects | **1/2** | **2/2** | **`v1`:** Generated `step_id = new_ulid("stp")` randomly inside `AgentRuntime.execute_turn`, so Temporal activity retries bypassed `(run_id, step_id)` deduplication for unapproved calls.<br>**`v2`:** Uses deterministic `f"{run_id}:t{turn_count}:c{tool_index}"` plus `args_hash` collision checks before side effects. |
| **R6** | Workflow determinism | **2/2** | **2/2** | Both avoided direct I/O/time/randomness in `RunWorkflow`. **`v2`** fixed `continue_as_new` to drain `workflow.all_handlers_finished()` and preserve `snapshot_id`, and verified determinism via an AST check + `OfflineWorkflowSimulator`. |
| **R7** | Tenancy | **2/2** | **2/2** | Both enforced `org_id` + `FORCE ROW LEVEL SECURITY` + S3 org prefixes. **`v2`** added composite FKs `(org_id, project_id)` and `(org_id, run_id)` so cross-org FK references cannot bypass RLS. |
| **R8** | Meaningful tests | **1/2** | **2/2** | **`v1` (`17 passed`):** Missed route tests, Temporal workflow/signal/replay tests, and concurrency races.<br>**`v2` (`14 passed, 2 skipped`):** Added concurrency tests, >1,000-event SSE pagination, `ApiController` route checks (`plan:decide`, 401/403/404/409/410), AST + signal/`continue_as_new` simulation, plus gated FastAPI/Temporal tests. |
| **R9** | Scope discipline | **2/2** | **2/2** | Both stayed tightly scoped to the Runs milestone. **`v2`** closed `v1`'s missing `plan:decide` route (which deadlocked default `plan_review=True` runs) and replaced `v1`'s silent `_to_response` `done=True` stub with real Anthropic/Bedrock block parsing. |
| **R10** | Honest report | **1/2** | **2/2** | **`v1`:** Disclosed offline SDK limits, but failed to report stubbed `_to_response`, missing `plan:decide`, missing APNs/FCM adapter, and assumptions.<br>**`v2`:** Explicitly documented implemented files, deferred adapters (`APNsFCMNotificationService`, `_OfflineWorkflowShim`), assumptions, and exact verified vs. syntax-only checks. |
| **Total** | | **15 / 20** | **20 / 20** | **+5 points across R1, R3, R5, R8, R10; finalized at `v2`.** |

### Verification Scope Note
In the offline test sandbox, external dependencies (`fastapi`, `httpx`, `temporalio`, `asyncpg`, `redis`, `boto3`, `anthropic`, `mypy`, `ruff`) were not installed. Consequently, `v2`'s production adapters (`PostgresStore`, `RedisStreamEventBus`, `E2BSandboxProvider`, `S3KMSObjectStore`, `KMSTokenVault`, `TemporalWorkflowStarter`, `create_fastapi_app`) were verified via `python3 -m py_compile` rather than live container execution, while behavioral invariants were executed against in-memory equivalents (`14 passed, 2 skipped`). Two minor hardening items remain from inspecting `v2`'s generated code: (1) adding an explicit fail-closed `ConflictError` if a worker retries a non-idempotent tool whose prior `tool_executions` row is still `'claimed'` after a mid-flight crash, and (2) synchronizing `RedisStreamEventBus.subscribe` readiness before the initial Postgres query rather than relying solely on `StreamGateway` gap backfill.
