---
name: go-unit-test-gomock
description: 'Detailed workflow for substantial Go unit test work. Use when adding new *_test.go files, building mock-based tests, introducing sqlmock-backed repository tests, converting tests to suite.Suite, or tightening weak gomock expectations. For small routine test edits, rely on the always-on unit test instruction instead of loading this skill.'
argument-hint: 'Describe the Go package, target function or method, and the behavior or cases to test.'
user-invocable: true
---

# Go Unit Tests With Gomock

## What This Skill Produces

This skill creates or updates Go unit tests that:

- use `go.uber.org/mock/gomock`
- use `github.com/DATA-DOG/go-sqlmock` when testing SQL-backed behavior
- use generated mocks instead of handwritten mock data
- avoid `gomock.Any()` and pass concrete expected values
- group related tests under a `suite.Suite` test harness so the package can run the full suite together

## When to Use

Use this skill when you need to:

- add a new `*_test.go` file in a Go package
- make a substantial change to test structure or shared setup
- test a handler, usecase, repository wrapper, service, or validator
- add or refactor `sqlmock`-backed SQL tests
- mock dependencies behind interfaces
- convert ad hoc tests into a suite-based structure
- tighten weak gomock expectations so tests assert the exact inputs being passed

Do not load this skill just to make a small local edit that already fits the always-on instruction rules.

## Rules

1. Use mocks generated from interfaces with `mockgen`.
2. For every `interfaces.go` file that is created, add this line directly above the interface definitions:

```go
//go:generate mockgen -source=interfaces.go -destination=mock/mock.go -package=mock
```

3. If an existing `interfaces.go` file is missing that line, add it.
4. Do not use `gomock.Any()`.
5. Use explicit values in `EXPECT()` calls. Pass the concrete request, context, ID, struct, error, or primitive that the code should send.
6. Always include `.Times(n)` on every `EXPECT()` chain so the expected call count is explicit.
7. Use `Do()` to inspect, assign, or verify arguments when pointer inputs, out parameters, or mutation make direct equality matching unreliable.
8. If the code currently makes exact matching difficult, make the test deterministic first. Inject time, UUIDs, or other dynamic dependencies so the expectation can still use exact values.
9. Use a suite struct that embeds `suite.Suite` and stores shared fixtures such as `ctrl`, mocks, and the system under test.
10. Use `SetupTest()` to initialize repeating variables, shared dependencies, and the subject under test for every test.
11. Use `TearDownTest()` to call `ctrl.Finish()`.
12. Add a `TestXxxSuite` entrypoint so `go test` runs the full suite.
13. Keep `_test.go` files in the same directory as the production code, but use the external test package form: `package <name>_test`.
14. Keep expectations narrow. Each test should assert one behavior or failure path.
15. When a test exercises SQL queries, transactions, or repository behavior against `*sql.DB` or `*gorm.DB`, use `sqlmock` for the database expectation layer instead of gomock or ad hoc in-memory databases.
16. Call `time.Now()` or an injected `Now()` method at most once per test path unless the behavior under test genuinely depends on multiple distinct timestamps.
17. When multiple related timestamps are needed in one test, capture a single base time and derive the others from it with `Add` or equivalent operations.

## Procedure

1. Inspect the target package.
Find the production code, the interfaces it depends on, and any existing test helpers or mock packages.

If the code under test issues SQL through `database/sql`, GORM, or a repository implementation backed by a real DB handle, plan to use `sqlmock` for the DB interaction and keep gomock for the remaining interface dependencies.

2. Verify mock generation.
Check whether the package already has a `mock/` folder and the required directive in `interfaces.go`.

```go
//go:generate mockgen -source=interfaces.go -destination=mock/mock.go -package=mock
```

If the directive is missing from `interfaces.go`, add it before working on the mocks.

3. Choose the test package.
Keep the test file in the same directory as the production code and use the external test package form: `package foo_test`.

4. Build the suite.
Create a suite struct that embeds `suite.Suite` and includes:

- `ctrl *gomock.Controller`
- one field per generated mock
- one field for the system under test

5. Initialize shared state.
In `SetupTest()`, create the controller with `gomock.NewController(s.T())`, initialize mocks with the generated constructors, initialize repeating variables, and build the subject under test with deterministic dependencies.

Use `SetupTest()` as the single place for repeated setup like validators, request helpers, mock dependencies, and handler or usecase structs. Example:

```go
func (t *HandlerTestSuite) SetupTest() {
	t.ctrl = gomock.NewController(t.T())
	t.ext = mock.NewMockIExternal(t.ctrl)
	t.uc = mock.NewMockIUsecase(t.ctrl)
	t.now = time.Date(2026, time.March, 19, 10, 0, 0, 0, time.UTC)
	validator := validator.New()
	t.h = onboarding.NewHandler(t.ext, t.uc, validator)
}
```

6. Add the suite runner.
Expose one top-level function:

```go
func TestUsecaseSuite(t *testing.T) {
	suite.Run(t, new(UsecaseSuite))
}
```

7. Write one test method per scenario.
Name methods like `Test_GetProducts_Success` or `Test_InitVerify_ValidationError`.

8. Set strict gomock expectations.
Use exact values in `EXPECT()` calls. Example:

```go
ctx := context.Background()
req := Request{ID: "abc-123", Email: "user@example.com"}

s.repo.EXPECT().Create(ctx, req).Return(nil).Times(1)
```

Do not replace `ctx` or `req` with `gomock.Any()`. Do not omit `.Times(n)`.

When a method takes a pointer that is populated or mutated inside the call, use `Do()` to inspect or assign the pointed value instead of weakening the expectation. Example:

```go
expected := database.ProductBuffer{ProductID: "BTC-USD"}

s.repo.EXPECT().Create(ctx, s.db, gomock.AssignableToTypeOf(&[]database.ProductBuffer{})).
	Do(func(_ context.Context, _ *gorm.DB, data any, _ ...func(*gorm.DB) *gorm.DB) {
		buffers := data.(*[]database.ProductBuffer)
		s.Require().Len(*buffers, 1)
		s.Equal(expected.ProductID, (*buffers)[0].ProductID)
	}).
	Return(nil).
	Times(1)
```

If the subject under test reaches SQL, build the DB fixture with `sqlmock` and assert the exact query, args, rows, and transaction flow there as well. Do not replace SQL assertions with `gomock.Any()` or a loose fake database.

9. Assert the full outcome.
Verify returned values, errors, HTTP status codes, response payloads, and state changes. Use `s.Require()` for preconditions and `s.Equal()` or `s.Error()` for behavior assertions.

10. Run the narrowest useful test command.
Prefer the package-level command first, then widen scope only if needed:

```bash
go test ./path/to/package -run TestUsecaseSuite
go test ./path/to/package
```

11. Regenerate mocks when interfaces change.
Run:

```bash
go generate ./...
```

## Suite Template

```go
package yourpkg_test

import (
	"testing"

	"github.com/stretchr/testify/suite"
	"go.uber.org/mock/gomock"

	"your/module/yourpkg"
	"your/module/yourpkg/mock"
)

type UsecaseSuite struct {
	suite.Suite

	ctrl *gomock.Controller
	repo *mock.MockIRepository
	ext  *mock.MockIExternal

	uc *yourpkg.Usecase
}

func TestUsecaseSuite(t *testing.T) {
	suite.Run(t, new(UsecaseSuite))
}

func (s *UsecaseSuite) SetupTest() {
	s.ctrl = gomock.NewController(s.T())
	s.repo = mock.NewMockIRepository(s.ctrl)
	s.ext = mock.NewMockIExternal(s.ctrl)
	s.uc = yourpkg.NewUsecase(s.repo, s.ext)
}

func (s *UsecaseSuite) TearDownTest() {
	s.ctrl.Finish()
}

func (s *UsecaseSuite) Test_DoThing_Success() {
	input := yourpkg.Input{ID: "dealer-001"}
	expected := yourpkg.Output{Status: "ok"}

	s.repo.EXPECT().Load(input.ID).Return(expected, nil).Times(1)

	actual, err := s.uc.DoThing(input)
	s.Require().NoError(err)
	s.Equal(expected, actual)
}
```

## Decision Points

### If the dependency input is dynamic

- Do not fall back to `gomock.Any()`.
- Inject deterministic collaborators such as clocks, UUID generators, or request IDs.
- Build the exact expected struct after setting those deterministic values.

### If the code depends on current time

- Prefer an injected clock or `Now()` seam over calling `time.Now()` directly inside the code under test.
- Read time at most once per test path unless the test is specifically verifying behavior across distinct moments.
- If multiple timestamps are needed for setup or assertions, derive them from a single base value.
- Multiple time reads are justified when the scenario truly models separate events, such as two HTTP requests sent at different points in the flow.

### If a dependency receives pointers or mutated arguments

- Use `Do()` to inspect assigned values, captured arguments, or pointer contents.
- Keep `.Times(n)` on the expectation.
- Use `Do()` as a precise check, not as a replacement for the expectation itself.

### If a method takes `context.Context`

- Reuse a known context variable created in the test and pass that same variable into the subject call.
- If the production code creates derived contexts internally, prefer refactoring so the externally relevant values can still be asserted exactly.

### If the package already uses plain `testing.T`

- Keep the existing style only when the change is intentionally small.
- For new multi-scenario tests, prefer converting to `suite.Suite` so shared setup and mocks stay consistent.

### If interfaces or constructors are hard to mock

- Fix the production seam instead of writing brittle tests.
- Move side effects behind interfaces and inject them into the constructor.

### If the code talks to SQL

- Use `sqlmock` to create the backing `*sql.DB`.
- If GORM is involved, open GORM with the mocked SQL connection so the repository still runs real query-building code.
- Assert the exact SQL behavior that matters for the scenario: query text or pattern, args, returned rows, exec result, begin, commit, and rollback.
- Keep gomock for non-database collaborators only.

## Completion Checks

The test is complete only if all of the following are true:

- mocks come from `mockgen`
- every created or updated `interfaces.go` file has the required `//go:generate mockgen -source=interfaces.go -destination=mock/mock.go -package=mock` line
- `_test.go` files stay in the same directory as the production code and use `package <name>_test`
- the suite embeds `suite.Suite`
- `SetupTest()`, `TearDownTest()`, and `TestXxxSuite()` exist
- `SetupTest()` initializes repeated variables, shared dependencies, and the subject under test
- no `gomock.Any()` appears in the new or updated tests
- every `EXPECT()` call includes explicit `.Times(n)`
- `Do()` is used when pointer arguments or mutation need value inspection or assignment
- SQL-related tests use `sqlmock` for DB expectations instead of gomock or ad hoc in-memory databases
- `time.Now()` or injected `Now()` reads are limited to one per test path unless distinct timestamps are part of the behavior being tested
- expectations use exact values
- the assertions cover the behavior the test is named for
- the targeted `go test` command passes

## Example Prompts

- `/go-unit-test-gomock add tests for GetProducts success and repository failure in app/market/usecases.go`
- `/go-unit-test-gomock create a suite-based handler test for app/onboarding/handlers.go using generated mocks`
- `/go-unit-test-gomock refactor this test to remove gomock.Any and assert the exact request struct`
- `/go-unit-test-gomock add repository tests that use sqlmock for the SQL path and gomock for the external dependency`
- `/go-unit-test-gomock update these tests to inject time and keep Now() to one call unless the flow needs distinct timestamps`
