---
description: "Always-on baseline for Go unit test edits. Keep expectations explicit, use gomock or sqlmock appropriately, and keep time handling deterministic. Put heavier workflow guidance in the unit test skill instead of here."
name: "Go Unit Test Rules"
applyTo: "**/*_test.go"
---

# Go Unit Test Rules

- Use `go.uber.org/mock/gomock` for interface mocks and `github.com/DATA-DOG/go-sqlmock` for SQL-backed tests.
- Use mocks generated from interfaces with `mockgen`.
- For every created or updated `interfaces.go` file, ensure this line exists above the interface definitions: `//go:generate mockgen -source=interfaces.go -destination=mock/mock.go -package=mock`.
- Do not write or keep `gomock.Any()` in test expectations.
- Pass concrete expected values into `EXPECT()` calls and always include `.Times(n)`.
- Use `Do()` when pointer inputs or mutation make direct equality matching unreliable.
- For SQL-backed tests, assert the relevant SQL behavior explicitly: query or exec pattern, args, rows, begin, commit, and rollback.
- Keep dynamic values deterministic by injecting time, UUID, request ID, or similar dependencies.
- `time.Now()` or an injected `Now()` method should be read at most once per test path unless the behavior genuinely depends on distinct timestamps.
- If multiple timestamps are needed for one logical moment, capture one base time and derive the rest from it.
- Keep `_test.go` files in the same directory as the production code and use `package <name>_test`.
- Prefer `suite.Suite` when the package has shared setup or multiple scenarios.
- Keep each test focused on one behavior and assert the full result it claims to cover.
- If the test cannot stay explicit without loose matchers, refactor the production seam instead of weakening the assertion.
