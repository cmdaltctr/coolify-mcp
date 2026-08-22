# Dev-agent enablement baseline

This file records the source snapshots and API verification used for the targeted dev-agent additions. It is an implementation record, not a new orchestration contract.

## Baseline

- Fork branch before work: `main`
- Fork HEAD: `0fc0326da41312a42a17a9e2c38e11b1c0a9d0bb`
- `StuMason/coolify-mcp` `main` HEAD: `0fc0326da41312a42a17a9e2c38e11b1c0a9d0bb`
- Previous vendored API state: the v4.2-era re-vendor in `84d9d44` (2026-07-29), with 14 generated chunks
- Coolify source and OpenAPI snapshot used here: `coollabsio/coolify@7fd678d9073428dc315267ce0c1ab6d8b9582340`
- Implementation branch: `feat/dev-agent-enablement`

The baseline build, 672 unit tests, client/spec drift check, and chunk drift check passed. Baseline lint had 52 warnings and no errors. The first dependency install attempt failed because npm tried to use a non-writable cache under `/root`; a clean install with a cache under `/tmp` passed. These were recorded rather than treated as product failures.

## API verification

The target endpoints were checked in this order: `openapi.yaml`, `routes/api.php`, the relevant API controller, and focused Coolify tests where present.

| Capability               | Verified native API                                                                                                                          |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Clone                    | `POST /applications/{uuid}/clone`, `/databases/{uuid}/clone`, `/services/{uuid}/clone`                                                       |
| Rollback                 | `GET /applications/{uuid}/rollback-images`, `POST /applications/{uuid}/rollback`                                                             |
| Shared environment       | CRUD collections under `/team/envs`, `/projects/{uuid}/envs`, `/projects/{uuid}/environments/{environment}/envs`, and `/servers/{uuid}/envs` |
| Destinations             | `GET /destinations`, CRUD on `/destinations/{uuid}`, and list/create under `/servers/{server_uuid}/destinations`                             |
| Scheduled-task execution | `POST /applications/{uuid}/scheduled-tasks/{task_uuid}/execute` and the service equivalent                                                   |

No route/controller disagreement was found for the operations implemented. Two specification limits matter to the MCP design:

- Shared environment variables have collection-list and create/update/delete routes but no native single-item GET. `shared_env(action="get")` therefore lists the selected scope and selects the numeric environment-variable id.
- The upstream OpenAPI omits request-body field schemas for project, environment, and server shared-variable writes even though their controllers validate the same key/value/flags/comment fields as team variables. The MCP input is based on those controller validations.

The refreshed `docs/coolify-openapi.yaml` was copied unchanged from that Coolify snapshot. `docs/openapi-chunks/` was regenerated with `node scripts/split-openapi-chunks.mjs`; generated chunks were not hand-edited.

## Scope decisions

- Cloning remains one `clone` resource tool.
- Rollback is part of the existing `deployment` tool.
- Shared inherited configuration stays separate from resource-specific `env_vars`.
- Docker destination management is one consolidated CRUD tool and does not introduce a networking layer.
- `scheduled_tasks(action="run_once")` keeps its public interface. Current Coolify uses execute-now; only a routing 404 (the route is absent) selects the previous cron fallback. A controller-level 404 is returned as an error rather than being misread as an old server.
- No dev-environment, runtime, SSH, host-shell, workflow-engine, or other orchestration abstraction was added.

## Complete-loop sufficiency review

The end-to-end development loop does not need another MCP concept. Each step maps to an existing or targeted consolidated primitive:

| Agent step                                           | MCP primitive                                                                |
| ---------------------------------------------------- | ---------------------------------------------------------------------------- |
| Inspect existing infrastructure                      | `get_infrastructure_overview`, resource list/get tools                       |
| Clone or create resources                            | `clone`, `application`, `database`, `service`                                |
| Choose or manage placement                           | `destinations`                                                               |
| Apply inherited and resource configuration           | `shared_env`, then `env_vars`                                                |
| Deploy and wait                                      | `deploy(wait=true)`                                                          |
| Inspect builds and runtime                           | `deployment`, `logs`, `diagnose_app`, `diagnose_server`                      |
| Run tests, migrations, or health checks in-container | `scheduled_tasks(action="run_once")`                                         |
| Redeploy and verify a source fix                     | `deploy`, status/log tools                                                   |
| Recover a bad application deployment                 | `deployment(action="rollback_images")`, then `deployment(action="rollback")` |
| Remove disposable resources                          | the existing application/database/service delete actions                     |

Source changes remain the responsibility of a GitHub MCP or other source-control client. The Coolify MCP supplies the deployment-side primitives and deliberately does not automate that cross-system workflow.

## Validation

- TypeScript build: passed
- Unit tests: 690 passed across 8 suites
- Lint: no errors; the same 52 pre-existing warnings as baseline
- Client/OpenAPI drift: all 131 client routes match the vendored spec
- Generated chunk drift: all 14 chunks match the vendored spec
- Formatting: passed
- Deterministic eval contract: 47-tool roster, schemas, annotations, safety classification, and token budget passed
- Eval package typecheck, lint, and formatting: passed

No live Coolify credentials were available in the validation environment, so the native endpoints were not invoked against a user instance. Endpoint behaviour is covered by client and MCP-handler tests, including API failures, parameter validation, execute-now, routing-404 fallback, and the distinction between a missing route and a resource-level 404.
