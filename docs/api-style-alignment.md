# Cradle (beatbox) API — AIP-style alignment with the Tempera stack

> Status: design README. Proposes that `beatboxd`'s REST + MCP surface adopt the
> **unified Google-AIP contract language** used across the Tempera stack,
> canonically defined in
> [`tempera-dev/data-engine`'s `API_STYLE.md`](https://github.com/tempera-dev/data-engine/blob/main/API_STYLE.md).
> No code here; this is the target contract + migration mapping. Cradle stays
> standalone (protocol-boundary integration, no sibling dependency) — alignment is
> about contract *shape*.
>
> Companion to the same alignment proposed for `tempo` and `palette`. One
> language, four services (`data-engine` reference + `cradle` + `tempo` +
> `palette`).

---

## 1. Why align

`data-engine` calls cradle over HTTP to execute untrusted code (verifiers, agent
programs, self-play) — see `data-engine/CRADLE.md`. If cradle speaks the *same*
AIP dialect as data-engine, one client SDK + one MCP client works across both,
the `Operation`/error/pagination shapes match, and data-engine's verifier-farm
client is a thin wrapper instead of a parallel implementation. `data-engine` is
the reference; `API_STYLE.md` is the rulebook.

Governing rule on both sides: **contract first (OpenAPI / MCP), native optional.**

---

## 2. Cradle today (current shape — partially AIP)

`beatboxd` currently exposes (`crates/beatbox-server/src/lib.rs`, router ~line 262):

```
GET    /health                                   (unauth liveness)
GET    /openapi.json                             (utoipa-generated)
GET    /v1/capabilities
GET    /v1/integration
GET    /v1/browser/profiles
POST   /v1/browser/admit
GET    /v1/browser/adapter/contract
POST   /v1/browser/adapter/capability | register | launch/plan | launch/claim | validate | completion/validate
POST   /v1/execute                               (sync; ExecuteRequest → ExecutionResult)
POST   /v1/jobs                                  (async enqueue)
GET    /v1/jobs/{id}                             (lookup)
DELETE /v1/jobs/{id}                             (cancel)
POST   /mcp                                       (run_wasm, get_capabilities, get_integration_contract, …)
```

Wire types (`crates/beatbox-core/src/lib.rs`): `ExecuteRequest` (carries `lane`,
`source`, `input`, `policy`, `idempotency_key`), `ExecutionResult` (carries
`status`, `value`, `metrics`, `deterministic`, `inputs_digest`, `egress`), `Policy`,
`JobRecord`, `ErrorBody { code, message }`.

**Already conformant:** bearer auth (`Authorization: Bearer <token>`); an
`idempotency_key` field on `ExecuteRequest` (just needs to become the standard
`Idempotency-Key` header); an OpenAPI doc is served; an MCP surface exists with a
fixture-pinned catalog + drift test.

**Gaps vs the unified language:**
- No `projects/{project}/` parent scoping; resources are flat under `/v1`.
- `ErrorBody` is a 2-field `{code, message}` — not the shared envelope
  (`error.code/status/message/details/request_id/retryable`, `API_STYLE.md` §6).
- `/v1/jobs` is close to the `Operation` model but isn't the canonical shape; sync
  `/v1/execute` returns `ExecutionResult` directly with no `Operation` option.
- No cursor pagination on any list (capabilities/jobs would want it).
- `/openapi.json` is runtime-generated, not committed + drift-gated (the MCP
  catalog *is* fixture-drift-tested; the OpenAPI doc isn't).
- operationIds aren't dotted `<collection>.<verb>`.

---

## 3. Target mapping (current → AIP)

Cradle is multi-tenant-light, so it uses a single default project parent
(`projects/default/`) to keep the shape uniform.

| Current | AIP target | operationId |
|---|---|---|
| `POST /v1/execute` | `POST /v1/{parent=projects/*}/executions:run` | `executions.run` |
| `POST /v1/jobs` | `POST /v1/{parent=projects/*}/jobs` → returns `Operation` | `jobs.create` |
| `GET /v1/jobs/{id}` | `GET /v1/{name=projects/*/jobs/*}` | `jobs.get` |
| `DELETE /v1/jobs/{id}` | `POST /v1/{name=projects/*/jobs/*}:cancel` | `jobs.cancel` |
| `GET /v1/capabilities` | `GET /v1/{parent=projects/*}/capabilities` | `capabilities.get` |
| `GET /v1/integration` | `GET /v1/{parent=projects/*}/integration` | `integration.get` |
| `GET /v1/browser/profiles` | `GET /v1/{parent=projects/*}/browserProfiles` | `browserProfiles.list` |
| `POST /v1/browser/admit` | `POST /v1/{parent=projects/*}/browserSessions:admit` | `browserSessions.admit` |
| browser adapter/* | `POST /v1/{parent=projects/*}/browserAdapters:validate` etc. | `browserAdapters.{validate,register,plan,claim}` |
| (job poll) | `GET /v1/{name=projects/*/operations/*}` | `operations.get` |

`executions:run` is a custom verb (AIP) — cradle's hallmark operation. The
`ExecuteRequest`/`ExecutionResult` wire types stay (they're the domain model); only
the wrapping (parent path, operationId, envelope, `Operation` option) changes.

---

## 4. What changes (concrete, per `API_STYLE.md`)

1. **Parent scoping**: nest under `projects/{project}/...` (default
   `projects/default/` for the single-box daemon).
2. **operationId** = `<collection>.<verb>` (`executions.run`, `jobs.create`,
   `capabilities.get`). MCP tool name = operationId, 1:1. Update the fixture
   `crates/beatbox-server/fixtures/mcp-tools.catalog.json` + the
   `mcp_catalog_drift` test in the same PR.
3. **Custom verbs**: `executions:run`, `jobs:cancel`, `browserSessions:admit`,
   `browserAdapters:validate` (replace flat `/execute`, `/admit`, adapter sub-paths).
4. **Shared error envelope** (`API_STYLE.md` §6): promote `ErrorBody` to
   `{ error: { code, status, message, details[], request_id, retryable } }`,
   canonical codes only. Keep `ExecutionResult.error` for execution-level failures
   (timeout/oom/denied) but map transport errors to the envelope.
5. **`Operation` model**: `jobs` already maps cleanly — `JobRecord.status` →
   `Operation.done`, `JobRecord.result/error` → `Operation.response`/`error`. Add
   an `Operation`-shaped poll endpoint `projects/*/operations/*`. Optionally let
   `executions:run` return an `Operation` for long verifiers.
6. **`Idempotency-Key`**: move `ExecuteRequest.idempotency_key` to the standard
   `Idempotency-Key` header (keep the field as a fallback/deprecated). Apply to
   `jobs.create` and all mutating browser ops.
7. **Cursor pagination** on any list (`capabilities` is singular today, but
   `jobs`/`browserProfiles` lists want `page_size`/`page_token`/`next_page_token`).
8. **Committed OpenAPI + drift gate**: check the generated `openapi.json` into
   the repo and add a CI gate `GET /openapi.json` == committed doc, mirroring the
   existing `mcp_catalog_drift` pattern. Regen the `sdks/*` clients in any
   contract PR.
9. **Headers**: keep `Authorization: Bearer <token>` (and `x-beatbox-api-key` as
   the documented compat alias); add `x-request-id` echoed on responses.
10. **MCP**: keep the fixture-pinned catalog + drift test (good), but align tool
    names to the new dotted operationIds and ensure `structuredContent` +
    `isError:true` everywhere.

---

## 5. What stays Cradle-specific

The **language** is shared; the **domain** is cradle's own: `Policy`, `Lane`
(wasm/python/js/native_exec), `ExecuteRequest`/`ExecutionResult`, `inputs_digest`
determinism, `NetPolicy`, browser fail-closed contracts, the `egress[]` log. AIP
doesn't touch any of that — it wraps the same payloads in the uniform envelope,
parent scoping, and operation discipline. `data-engine`'s verifier-farm client
(`data-engine/CRADLE.md`) keeps relying on `deterministic`, `inputs_digest`, and
empty `egress[]` exactly as today.

---

## 6. Migration notes

- Pre-1.0; breaking contract changes are allowed now (uniformity over compat
  shims). Land routes + operationIds + OpenAPI regen + `sdks/*` clients + MCP
  fixture + drift gate in one slice.
- The `idempotency_key` → `Idempotency-Key` header move is the one compat-sensitive
  spot; accept both for one release, then drop the field.
- Coordinate the `Operation` + error envelope shapes with `tempo` and `palette` so
  the four services ship identical shapes in the same window.

---

## 7. Reference

- Canonical language: [`tempera-dev/data-engine` → `API_STYLE.md`](https://github.com/tempera-dev/data-engine/blob/main/API_STYLE.md)
  (data-engine is the reference implementation; its `api/openapi.yaml` is the
  byte-for-byte oracle).
- How data-engine calls cradle: `data-engine/CRADLE.md` (the wire shapes this PR
  keeps stable: `ExecuteRequest`, `ExecutionResult`, `inputs_digest`, `egress`).
- Transfer checklist: `API_STYLE.md` §16.
- Sibling alignment PRs: `tempo` `docs/api-style-alignment.md`; `palette` (AIP
  migration target in `API_STYLE.md` §15).
