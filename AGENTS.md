# Repository Guidelines

## Project Structure & Module Organization
Core server code lives in `server/`, while clients and API definitions reside in `client/` and `api/` (protocol buffers plus generated stubs). Shared utilities are under `pkg/`. Command-line tools (`etcdctl/`, `etcdutl/`) build through `scripts/build.sh`. Long-form docs stay in `Documentation/`, and E2E or robustness suites live in `tests/`. Contributor automation and generators are in `hack/` and `scripts/`.

## Build, Test, and Development Commands
- `make build` — compiles etcd binaries with repo defaults (`scripts/build.sh`).
- `make tools` — builds local helper binaries from `tools/`.
- `make test-unit` / `make test-integration` / `make test-e2e` — runs targeted suites via `scripts/test.sh`.
- `make verify` — executes static analysis (gofmt, golangci-lint, license, YAML, etc.).
- `GO_BUILD_FLAGS="-race" make build` — example of passing extra Go flags when needed.

## Coding Style & Naming Conventions
Follow Go community guidance (`go fmt`, `goimports`, Go Code Review Comments). Run `make verify-gofmt` and `make verify-lint` before sending a PR; `make fix` applies most auto-formatting. Exported symbols use PascalCase, unexported identifiers use lowerCamelCase, and file names stay snake_case. Generated artifacts belong under their existing `api/` or `pkg/` subtree; never edit generated files manually.

## Testing Guidelines
Place Go tests alongside sources with a `_test.go` suffix and `Test...`/`Benchmark...` naming. Use table-driven tests for API surface changes and include regression cases for bugs. Run `make test-unit` for quick checks and `make test-e2e` before large features; `make test-coverage` writes results to `covdir/` and feeds the Codecov uploader. For flaky or timing-sensitive changes, stress locally with `go test -run <Name> -count=100 ./path/...`.

## Commit & Pull Request Guidelines
Start commit subjects with the touched package (`etcdserver: guard lease deletion`) and keep them under 72 characters. Add a `Signed-off-by: ...` trailer (`git commit --signoff`). Each PR should describe intent, list validation commands, and link any GitHub issues. Include configuration or API changes in `Documentation/` when applicable, and attach logs or screenshots when debugging operational topics.
