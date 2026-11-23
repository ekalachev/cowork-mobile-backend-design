# Cowork Mobile — SaaS Backend Architecture & Tested Build Prompt

Architecture specification and empirically tested Python 3.12 implementation prompt for a **mobile-app-centric SaaS backend** for Claude Cowork.

## Deliverables

| Artifact | Purpose |
|---|---|
| [`ARCHITECTURE_SPEC.md`](./ARCHITECTURE_SPEC.md) | Full system design (§0 TL;DR through §13 Risks & Open Questions): maps all 14 Claude Cowork features (`F1`–`F14`) to backend capabilities, includes 4 native Mermaid diagrams, mobile-first adaptations, API/SSE contracts, multi-tenant Postgres schema, taint-aware security model, SLOs & unit economics, ADRs 1–7, and a 12-week MVP plan. |
| [`IMPLEMENTATION_PROMPT.md`](./IMPLEMENTATION_PROMPT.md) | Final tested `v2` Python 3.12 build prompt scoped to the MVP **Runs milestone**, plus prompt design rationale, verbatim `v1` baseline prompt, and an honest 20-point rubric verification log (`15/20` → `20/20`). |

## Architecture Highlights

- **Phone as a remote control:** Execution moves from a local desktop VM to **per-run cloud microVM sandboxes** (E2B/Firecracker behind a swappable `SandboxProvider` interface), with an optional paired **Desktop Runner** over mTLS WebSocket (`v1.1`).
- **Durable agent runs:** Every run executes as a **Temporal workflow**, surviving mobile app backgrounding, dropped cellular connections, worker restarts, `continue-as-new` history compaction, and hours-long human approvals (sandboxes pause and snapshot after 60 s idle).
- **Resumable mobile streaming & lock-screen control:** Progress writes to a gap-free append-only Postgres `run_events` log (`PRIMARY KEY (run_id, seq)`, ULID range-partitioned) and tails live over SSE (`Last-Event-ID`, cellular delta coalescing, thumbnail frames). Destructive or tainted external writes trigger actionable push notifications / Live Activities with single-use biometric/passkey approvals bound to `(run_id, step_id, sha256(args))`.
- **Fail-closed tenancy & secrets isolation:** OAuth tokens remain server-side inside the **MCP Gateway** token vault; untrusted connector/browser outputs taint the run; every tenant table enforces Postgres `FORCE ROW LEVEL SECURITY` via transaction-local `SET LOCAL app.org_id` with composite tenant foreign keys and per-org KMS keys.

## Target Stack

- **Control & Stream Plane:** Python 3.12 · FastAPI · Pydantic v2 · SSE (`Redis Streams` + `Postgres`) · mTLS WebSocket (Dispatch runner)
- **Execution Plane:** Temporal (`temporalio`) · Anthropic API + Amazon Bedrock failover · Server-side MCP Gateway · E2B microVM Sandboxes
- **Data & Security:** Postgres 15+ (`asyncpg`, RLS, ULID partitions, hash-chained `audit_log`) · S3 · AWS KMS · OIDC + WebAuthn Passkeys
