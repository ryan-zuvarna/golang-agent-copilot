---
description: "Use when writing or updating Go unit tests, gomock expectations, generated mocks, testify suites, handler tests, usecase tests, or refactoring tests to remove gomock.Any. Enforce strict explicit gomock values and suite.Suite-based test structure."
name: "Go Gomock Test Rules"
applyTo: "**/*_test.go"
---

# Go Gomock Test Rules

- Use `go.uber.org/mock/gomock` for mocks in Go unit tests.
- Use mocks generated from interfaces with `mockgen`.
- For every created `interfaces.go` file, add this line above the interface definitions:

```go
//go:generate mockgen -source=interfaces.go -destination=mock/mock.go -package=mock
```

- If an existing `interfaces.go` file is missing that line, add it.
- Do not write or keep `gomock.Any()` in test expectations.
- Pass concrete expected values into `EXPECT()` calls: exact request structs, IDs, contexts, errors, primitives, and return values.
- Always include `.Times(n)` on every `EXPECT()` chain so the expected call count is explicit.
- If exact matching is hard because the code uses dynamic values such as time, UUID, or request IDs, make the production code deterministic by injecting those dependencies.
- Prefer a `suite.Suite` test harness for packages with multiple scenarios or shared setup.
- The suite should include a `gomock.Controller`, generated mocks, and the subject under test.
- Use `SetupTest()` to construct a fresh controller and fresh mocks for each test.
- Use `TearDownTest()` to call `ctrl.Finish()`.
- Add a `TestXxxSuite(t *testing.T)` function so the full suite runs through `suite.Run`.
- Keep each test focused on one success path, validation path, or error path.
- Assert the full behavior the test claims to cover: returned data, error value, HTTP status, payload, or side effect.

## Preferred Pattern

```go
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
```

## Expectation Rule

Prefer this:

```go
ctx := context.Background()
req := VerifyReq{RegisterID: "1", Email: "user@example.com"}

s.uc.EXPECT().InitVerify(ctx, req).Return("reference-code", nil).Times(1)
```

Never generate this:

```go
s.uc.EXPECT().InitVerify(gomock.Any(), gomock.Any())
```

Do not omit `.Times(n)` from an `EXPECT()` chain.

## Completion Check

Before finishing a Go test change, verify that:

- no new `gomock.Any()` appears
- mocks come from generated mock types
- shared setup uses `suite.Suite` when the test shape warrants it
- every `EXPECT()` call includes `.Times(n)`
- expectations use exact values
- the targeted `go test` command is run when feasible

## Escalation

If the requested test cannot avoid loose matchers without a production code change, say so explicitly and refactor the seam instead of weakening the assertion.
