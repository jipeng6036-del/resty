# Resty transport failures and half-open recovery

Base: `78dfc5c1dff969d4d30313f7fde8b0987c681135` (upstream `v3`). Candidate: `130cd1a8819ff2e9fa9dc2cec2f417aad028b133`, tree `e9c8535ae6418ec7e9df5b054c2cc4618b46b217`.
Based on [#1262](https://github.com/go-resty/resty/pull/1262): the two original commits remain in this branch with original author metadata and `cherry-pick -x` provenance. This version adds attempt-owned half-open recovery and real HTTP concurrency/wrapper regressions.

## Reproduce

```sh
git clone https://github.com/jipeng6036-del/resty.git
cd resty
git checkout 130cd1a8819ff2e9fa9dc2cec2f417aad028b133
go test -v -race -count=1 -run 'TestCircuitBreakerIgnoredHalfOpenProbe|TestIgnoredStaleAttemptCannotReleaseCurrentProbe|TestExternalCircuitBreakerAdmission|TestEmbeddedCircuitBreakerAdmission' ./...
go run gotest.tools/gotestsum@latest -f testname -- ./... -race -count=1 -coverprofile=coverage.txt -covermode=atomic -coverpkg=./... -shuffle=on
go vet ./...
go build ./...
```

The regressions use the actual public Resty client with loopback HTTP servers, cancellation barriers and real transport timeouts; they do not call private breaker helpers or mock the transport. `PUBLIC_FIXTURES.json` records the additional external-module execution (only the test package name is changed). Final source bytes and file modes are bound by `COMMIT_BINDING.json` and `INPUT_BINDINGS.json`.

## Results and limits

- Linux configured CI starting with Go1.27.3 and Go1.23.12: each1,113 tests passed, race/shuffle/atomic coverage enabled. gotestsum1.13.0 requires Go>=1.24, so its nominal1.23 run selected Go1.26.10; a separate `GOTOOLCHAIN=local` actualGo1.23.12 full suite also passed1,113 tests.
- Formatting, vet and build passed. Changed production statement coverage:54/54,63/63,63/63 across the three profiles (see raw block mapping).
- Actual external HTTP modules:16 business cases per actualGo1.27.3/1.23.12, including closed transport failures, ignored probe errors, stale request ownership, ordinary custom breakers and embedded wrappers.
- The four wrapper admission cases pass on the original patch and final candidate and specifically fail on the intermediate v3 implementation. The intermediate implementation was never submitted upstream.
- Ignored-error probe recovery applies to concrete built-in Count/Ratio breakers registered directly. Custom/embedded wrappers retain their public `Allow` behavior; their ignored-error half-open recovery is not newly guaranteed.
- Local verification only. Upstream CI/approval, Windows/macOS behavior, Codecov upload, Plumber and any real external service are not claimed as passed.

The JSON/log paths describe the isolated runner. These are raw local validation results, not a claim of human review or hosted CI. SHA-256 allowlist is in `manifest.json`.
