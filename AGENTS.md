# Repository Guidelines

Use this guide to align contributions with etcd maintainers' expectations and build a study plan for the codebase.

## Project Structure & Module Organization
Runtime logic resides in `server/` (notably `server/etcdserver/` for Raft orchestration and `server/storage/` for MVCC). Public Go clients live in `client/`, while CLIs (`etcdctl/`, `etcdutl/`) share helpers under `pkg/`. gRPC and protobuf definitions sit in `api/` alongside generated artifacts from the `hack/` scripts. Long-form docs and design notes are under `Documentation/`, and scenario testing is in `tests/` (with robustness assets in `tests/robustness/`).

## Architecture & Study Path
Follow `server/main.go` → `server/embed/` to see how processes start, then inspect `server/etcdserver/api/` for gRPC -> internal adapter layers. Study the Raft state machine in `server/etcdserver/raft` and the MVCC store in `server/storage/mvcc`. Configuration flows land in `server/config` and propagate via dependency injection; observing this pattern clarifies how features toggle. For best-practice Go patterns, review `pkg/traceutil` (context propagation) and `pkg/wait` (concurrency primitives).

## Build, Test, and Development Commands
- `make build` — compiles etcd binaries using `scripts/build.sh` (honours `GO_BUILD_FLAGS`).
- `make test` / `make test-unit` / `make test-e2e` — run whole or targeted suites via `scripts/test.sh`.
- `make verify` — drives gofmt, golangci-lint, license, YAML, and proto checks; pair with `make fix` when automation can resolve issues.
- `make fuzz` and `make test-coverage` — explore fuzz targets or produce `covdir/` for Codecov uploads.

## Coding Style & Naming Conventions
The repository targets Go `1.24` (`go.mod`). Format with `gofmt`/`goimports` (tabs for indentation). Exported identifiers use PascalCase, unexported ones lowerCamelCase, and filenames use snake_case. Keep generated code checked in but regenerate with `hack/update-*.sh` rather than editing manually. Run `make verify-gofmt` and `make verify-lint` before opening a pull request.

## Testing Guidelines
Place `_test.go` files beside production code and prefer table-driven cases. Name tests `Test<Component><Scenario>`, add regression cases for bug fixes, and lean on `GO_TEST_FLAGS="-run <Pattern>" make test-unit` for quick iteration. Call `make test-coverage` ahead of larger merges, and stress suspect cases with `go test -count=100` or the workflow in `CONTRIBUTING.md`.

## Commit & Pull Request Guidelines
Prefix commit subjects with the affected package (`etcdserver:` or `clientv3:`), keep the first line under 72 characters, and include `Signed-off-by:` via `git commit --signoff`. PRs should summarise behaviour, list validation commands, and link related issues. Update `Documentation/` or sample manifests when user-facing behaviour shifts, and confirm CI health before requesting review.

## Security & Configuration Notes
Avoid committing secrets or cluster credentials; reference material belongs in `Documentation/security/`. Follow the TLS/auth guidance in `security/README.md` when touching defaults, and flag breaking configuration changes in PR descriptions so release notes stay accurate.
