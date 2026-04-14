# Pigfest 70 MCP Server Build Plan

## Working mode

This environment can use **Pi itself as a spawned sub-agent process** via `pi -p "...prompt..."`.

Execution will be done as a disciplined workflow using handoff documents, report files, and explicit spawned-agent commands:

1. **Test agent phase** — define the core tests from the spec and write a handover for implementation.
2. **Vertical slice implementation phase** — build one coherent capability at a time.
3. **Verification phase** — run only the relevant test set for that slice and write a capability report.
4. **Fix phase** — address failures, re-run tests, update the report.
5. **Promote phase** — once the slice is solid, move to the next slice.

This keeps the work vertical, test-driven, and easy to audit.

See `docs/sub-agent-workflow.md` for the exact command patterns.

---

## Spec summary

Source of truth: `Specs/pigfest-70-mcp-spec.md`

The app must provide:

- A Dockerised Python service
- An MCP server exposed over HTTP/SSE
- Optional bearer-token authentication
- Tools for:
  - running bash commands
  - reading files under `/data`
  - writing files under `/data`
  - listing directories under `/data`
  - deleting files under `/data`
- Basic structured error handling so tool failures do not crash the server
- Minimal logging to `/logs/app.log`
- Startup/runtime support for:
  - writing rclone config from env
  - initial sync
  - periodic sync loop
- Docker/compose configuration for Portainer deployment
- GHCR publishing workflow

---

## Delivery strategy: vertical slices

### Slice 0 — Foundation and test harness
Goal: create the minimum project structure needed to test and iterate safely.

Deliverables:
- Python package structure cleaned up
- dependency set chosen
- pytest test suite scaffold
- common config/path utilities
- report templates and handoff docs

Exit criteria:
- tests can run locally
- basic import smoke tests pass

### Slice 1 — Local file tool layer
Goal: implement the safe local file operations under `/data`.

Deliverables:
- path resolution rooted at `/data`
- read file tool logic
- write file tool logic
- list directory tool logic
- delete file tool logic
- traversal protection
- structured success/error payloads

Relevant tests:
- read/write/list/delete happy paths
- nested directory creation for writes
- missing file behaviour
- directory vs file error cases
- path traversal rejection

Exit criteria:
- all file-operation tests pass

### Slice 2 — Bash execution tool
Goal: implement unrestricted bash execution with safe process handling and structured output.

Deliverables:
- bash runner function/tool
- stdout/stderr/exit-code capture
- timeout/error handling policy
- tests for success/failure

Relevant tests:
- successful command
- failing command
- stderr capture
- non-crashing exceptions

Exit criteria:
- all bash tests pass

### Slice 3 — MCP server surface
Goal: expose the tools through an actual MCP-compatible HTTP/SSE server.

Deliverables:
- FastMCP integration
- tool registration
- `/sse` endpoint exposure
- configurable host/port

Relevant tests:
- app creation smoke test
- registered tool presence
- server config from env

Exit criteria:
- MCP app boots correctly and tools are registered

### Slice 4 — Bearer token authentication
Goal: enforce auth only when configured.

Deliverables:
- optional token config
- auth enabled/disabled behaviour
- tests for both modes

Relevant tests:
- no-token mode allows access
- token mode rejects missing/incorrect token
- token mode accepts correct token

Exit criteria:
- auth tests pass

### Slice 5 — Logging
Goal: basic operational logging to `/logs/app.log`.

Deliverables:
- logger setup
- startup log events
- sync/tool error log coverage

Relevant tests:
- log file creation
- messages written to configured path

Exit criteria:
- logging tests pass

### Slice 6 — rclone config + sync orchestration
Goal: implement the container runtime pieces around Drive sync.

Deliverables:
- env parsing for service account json/folder id
- write rclone config on startup
- initial sync command
- background periodic sync loop
- error logging around sync failures

Relevant tests:
- config generation from env
- startup command composition
- sync loop scheduling behaviour
- failures logged not crashing app

Exit criteria:
- orchestration tests pass

### Slice 7 — Container and deployment integration
Goal: complete Docker/compose/runtime ergonomics.

Deliverables:
- Dockerfile installs rclone and Python deps
- startup entrypoint script
- compose file aligned with app behaviour
- docs updated

Relevant tests/checks:
- Dockerfile/static checks
- env contract validation
- compose references correct image/ports/volumes

Exit criteria:
- container build succeeds
- runtime contract matches spec

### Slice 8 — End-to-end confidence pass
Goal: prove the assembled app hangs together.

Deliverables:
- integrated test run
- final capability report
- known limitations list

Exit criteria:
- core test suite green
- docs reflect actual implementation

---

## Test-first workflow artifacts

For each slice we will maintain:

- `docs/handover/test-agent-handover.md` — spec distilled into test requirements
- `docs/handover/implementation-handover-slice-<n>.md` — what the build agent should implement next
- `docs/reports/slice-<n>-capability-report.md` — test results, current capability, defects, next steps

---

## Core test scope only

The test suite should focus on core functionality, not frivolous checks. That means:

Include:
- behaviour required by the spec
- error handling that prevents crashes
- auth and path-safety boundaries
- startup/runtime orchestration logic
- config parsing and command composition

Exclude unless needed:
- cosmetic string snapshots
- trivial getters/setters
- implementation-detail tests that hinder refactoring
- exhaustive framework internals already covered by dependencies

---

## Planned implementation order

1. Write the test handover and the core test suite scaffold
2. Implement Slice 1 and iterate until green
3. Implement Slice 2 and iterate until green
4. Implement Slice 3 and iterate until green
5. Implement Slice 4 and iterate until green
6. Implement Slice 5 and iterate until green
7. Implement Slice 6 and iterate until green
8. Finish Slice 7 integration work
9. Run end-to-end confidence pass and produce final report

---

## Immediate next step

Act as the **test-writing sub-agent** next:
- extract the spec into actionable test requirements
- create the first handover doc for implementation
- add the core tests before building production code
