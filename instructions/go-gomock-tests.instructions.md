---
description: "Use when writing or updating Go unit tests, gomock expectations, generated mocks, testify suites, handler tests, usecase tests, or refactoring tests to remove gomock.Any. Enforce strict explicit gomock values and suite.Suite-based test structure."
name: "Go Gomock Test Rules"
applyTo: "**/*_test.go"
---

# Go Gomock Test Rules

- Use `go.uber.org/mock/gomock` for mocks in Go unit tests.
- Use mocks generated from interfaces with `mockgen`.
- For every created or updated `interfaces.go` file, ensure this line exists above the interface definitions: `//go:generate mockgen -source=interfaces.go -destination=mock/mock.go -package=mock`.
- If an existing `interfaces.go` file is missing that line, add it.
- Do not write or keep `gomock.Any()` in test expectations.
- Pass concrete expected values into `EXPECT()` calls: exact request structs, IDs, contexts, errors, primitives, and return values.
- Always include `.Times(n)` on every `EXPECT()` chain so the expected call count is explicit.
- If exact matching is hard because the code uses dynamic values such as time, UUID, or request IDs, make the production code deterministic by injecting those dependencies.
- Prefer a `suite.Suite` test harness for packages with multiple scenarios or shared setup.
- Keep `_test.go` files in the same directory as the production code, but name the package `package <name>_test`.
- The suite should include a `gomock.Controller`, generated mocks, and the subject under test.
- Use `SetupTest()` to construct a fresh controller, initialize repeating variables, create fresh mocks, and build the subject under test for each test.
- Use `TearDownTest()` to call `ctrl.Finish()`.
- Add a `TestXxxSuite(t *testing.T)` function so the full suite runs through `suite.Run`.
- Keep each test focused on one success path, validation path, or error path.
- Assert the full behavior the test claims to cover: returned data, error value, HTTP status, payload, or side effect.
- If the requested test cannot avoid loose matchers without a production code change, refactor the seam instead of weakening the assertion.
