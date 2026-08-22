# ADR-001: Dev-agent enablement primitives

**Date:** 2026-08-22
**Status:** Accepted
**Deciders:** cmdaltctr/coolify-mcp fork maintainers

## Context

AI agents managing Coolify development environments lacked primitives for provisioning, deploying, inspecting, repairing, validating, rolling back, and cleaning up disposable resources. The existing 44-tool surface covered individual resource CRUD and diagnostics but left gaps an agent needed to complete a full development loop:

- No way to clone an existing application, database, or service into a disposable copy.
- No way to discover or execute application rollbacks.
- No way to manage shared environment variables at team, project, environment, or server scope (only resource-specific `env_vars` existed).
- No way to manage Docker destinations through CRUD operations.
- `scheduled_tasks(action="run_once")` used a temporary cron fallback that could fire more than once before cleanup, even on Coolify versions that expose a native execute-now endpoint.

The constraint was to give agents the minimum additional primitives without introducing a `dev_environment` abstraction, a runtime orchestration layer, a generic DevOps framework, arbitrary SSH access, or a host-shell facility. The fork's baseline `main` matched `StuMason/coolify-mcp` at `0fc0326` when work began.

## Decision

Add five targeted capabilities, each as a consolidated tool or extended action set that preserves Stu Mason's consolidated-tool philosophy (one tool per resource category, action parameter selects the operation):

1. **`clone`** — one consolidated tool for `application`, `database`, and `service`. Calls Coolify's native `POST /{resource}/{uuid}/clone` endpoints. Returns a `ClonedResourceSummary` with UUID, name, resource type, destination, and project/environment context. A follow-up `get_*` call enriches the projection so the caller has identifiers for immediate follow-up operations.

2. **`deployment` rollback** — extend the existing `deployment` tool with `rollback_images` (calls `GET /applications/{uuid}/rollback-images`) and `rollback` (calls `POST /applications/{uuid}/rollback`). Rollback is application-specific, so it belongs in the deployment tool rather than a new tool.

3. **`shared_env`** — one consolidated tool supporting `team`, `project`, `environment`, and `server` scopes. Each scope maps to its native Coolify collection route (`/team/envs`, `/projects/{uuid}/envs`, `/projects/{uuid}/environments/{env}/envs`, `/servers/{uuid}/envs`). Upstream exposes no single-item GET, so `get` resolves through scoped listing by numeric environment-variable ID. Values are masked as `***` unless `reveal: true` is supplied, matching the existing `env_vars` masking contract.

4. **`destinations`** — one consolidated CRUD tool. List may narrow to one server; create is always server-scoped (`POST /servers/{uuid}/destinations`); get/update/delete operate on `/destinations/{uuid}`. This is destination placement support, not a networking subsystem.

5. **`scheduled_tasks(action="run_once")` native execution** — prefer Coolify's native `POST /applications/{uuid}/scheduled-tasks/{task_uuid}/execute` (and the service equivalent). Create the throwaway task as `enabled: false`, dispatch it immediately, poll `list_executions` for a terminal result, then delete the task. Fall back to the previous `* * * * *` cron flow only when the native execute route is absent, detected by `isRoutingCatchAll` (a 404 carrying the catch-all `docs` key). Authorisation errors (403), validation failures (422), timeouts, and ordinary server failures (500) do not trigger the fallback. The public MCP interface is unchanged.

6. **Refreshed vendored OpenAPI** — `docs/coolify-openapi.yaml` re-vendored from `coollabsio/coolify@7fd678d` and chunks regenerated via `npm run build:chunks`.

All new tools carry the `DESTRUCTIVE` MCP annotation. Destructive operations retain elicitation behaviour. Secret masking, `reveal: true` semantics, sensitive-read requirements, and untrusted-log wrapping are preserved. No orchestration abstraction, SSH facility, or host-shell access was introduced.

## Consequences

### Positive

- Agents can complete a full development loop (inspect, clone, configure, deploy, diagnose, run diagnostics, redeploy, roll back, clean up) using existing tools plus five additions.
- Native scheduled-task execution eliminates the multi-fire cron risk on current Coolify; the fallback is narrow and version-gated.
- Shared environment variables are distinct from resource-specific `env_vars`, matching Coolify's own separation.
- Clone responses carry enough identifiers (UUID, name, project_uuid, environment_id, environment_uuid, environment_name) for immediate follow-up without a second round-trip in the common case.
- The consolidated-tool philosophy is preserved: three new tools plus two extended action sets, not 15 new single-purpose tools.

### Negative

- `shared_env(action="get")` performs a scoped list and filters by ID because upstream has no single-item GET. This is one extra request compared to a native single-item route.
- `is_shown_once: true` variables are withheld by Coolify even with `reveal: true` and sensitive-read scope. The tool faithfully returns `undefined` for the value; callers must know this is Coolify's behaviour, not a bug.
- Rollback of an invalid image tag is accepted by Coolify and fails asynchronously (the deployment reaches `failed` status). The tool cannot make Coolify reject it synchronously. Callers must poll the deployment status.
- The fallback path for `scheduled_tasks(action="run_once")` retains the multi-fire cron risk on older Coolify versions. The tool description warns about this.

### Neutral

- Tool count increased from 44 to 47. Token budget remains ~6,600.
- The vendored OpenAPI refresh is a large diff (9,786 lines) but is generated, not hand-edited.

## Alternatives Considered

| Option                                                                    | Rejected Because                                                                                                                                                                  |
| ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **A `dev_environment` orchestration tool**                                | Introduces an orchestration abstraction the task explicitly prohibited. The existing tools plus five additions already cover the loop.                                            |
| **Separate `rollback` tool**                                              | Rollback is application-specific and deployment-related. A separate tool fragments the deployment surface for no benefit.                                                         |
| **Separate `clone_application`, `clone_database`, `clone_service` tools** | Three single-purpose tools instead of one consolidated tool violates the consolidated-tool philosophy. The three clone endpoints share the same request/response shape.           |
| **Broad compatibility framework for `scheduled_tasks`**                   | A broad framework risks falling back on non-routing errors (403, 500, timeout). The narrow `isRoutingCatchAll` check ensures only a genuinely absent route triggers the fallback. |
| **Generic SSH or host-shell facility**                                    | Explicitly prohibited. In-container diagnostics are already covered by `scheduled_tasks(action="run_once")`.                                                                      |
| **Do nothing / status quo**                                               | Agents cannot complete the development loop. Cloning, rollback, shared env, and destination management are all gaps a dev agent hits immediately.                                 |

## References

- PR #1: <https://github.com/cmdaltctr/coolify-mcp/pull/1>
- Implementation record: `docs/dev-agent-enablement.md`
- Vendored OpenAPI: `docs/coolify-openapi.yaml` (from `coollabsio/coolify@7fd678d`)
- Source changes: `src/types/coolify.ts`, `src/lib/coolify-client.ts`, `src/lib/mcp-server.ts`
- Upstream architectural reference: `StuMason/coolify-mcp`
- Coolify API reference: `coollabsio/coolify`
- Live validation: Coolify v4.3.10 at `coolify.praxis.my` (disposable instance, full matrix, all disposable resources cleaned up)
