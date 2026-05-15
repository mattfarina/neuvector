# NeuVector Code Quality Audit

**Repository:** `mattfarina/neuvector` (fork of `neuvector/neuvector`)
**Audit date:** 2026-05-15
**Branch audited:** `copilot/conduct-quality-audit-report` (HEAD `ad4b51a`)
**Auditor toolchain:** Go 1.26.2, `go vet`, `gofmt`, `ineffassign`, `gocyclo`, `gosec`, `govulncheck`, `go test ./...`

---

## 1. Executive summary

NeuVector is a large, mature, multi-language container-security platform. The Go portion is the
bulk of the code (~256 k LOC across 454 files) with a smaller, performance-critical C/C++ data
plane (~30 k LOC). Overall code health is **good**:

- The default Go toolchain (`go vet`, `gofmt`) is essentially clean.
- The full `go test ./...` suite passes (30 packages with tests, 0 failures).
- CI is well configured: `golangci-lint`, CodeQL (Go + C/C++ + Actions), `govulncheck`,
  OpenSSF Scorecard, FOSSA, Renovate, pre-commit hook, and pinned action SHAs.
- A `SECURITY.md` policy and security advisory process are in place.

The principal risks identified are concentrated in three areas:

1. **A handful of high-impact security issues** (weak/legacy crypto choices, several
   `InsecureSkipVerify: true`, a hardcoded IV in CFB encryption, a
   `http.ListenAndServe` without timeouts, a SQL string built with `fmt.Sprintf`).
2. **High cyclomatic complexity and very large source files** in the controller's REST and
   cache layers, which depresses maintainability and review quality.
3. **Test coverage gaps** — only ~31 of 59 non-vendor Go packages have any test files at all,
   and the data-plane C/C++ code has no unit tests in this repo.

None of the findings are blocking, but the prioritized recommendations in §8 would meaningfully
reduce the project's risk surface.

---

## 2. Codebase inventory

| Metric | Value |
|---|---|
| Total non-vendor size on disk | ~39 MB |
| Go source files (non-vendor) | 454 |
| Go LOC (non-vendor) | 255,837 |
| Go test files | 83 |
| Go non-vendor packages | 59 |
| Packages with tests | 31 (~53%) |
| C / C++ files | 82 (55 `.c`, 27 `.h`) |
| C / C++ LOC | 29,608 |
| Shell scripts | 99 |
| YAML files | 72 + 13 `.yml` |
| Vendored Go modules (top-level) | 12 dirs / 185 modules in `modules.txt` |
| Direct + indirect deps in `go.mod` | ~179 entries |
| Recent contributors (last 90 days) | 1 (this fork; upstream activity differs) |

### 2.1 Largest Go source files (top 10)

These files are in the top decile by size and are repeated culprits in cyclomatic-complexity hot
spots below. They are good targets for splitting.

```
173 KB  controller/api/apis.go
173 KB  controller/rest/crdsecurityrule.go
140 KB  share/enforcer_service.pb.go            (generated)
134 KB  controller/rest/federation.go
123 KB  controller/kv/helper.go
123 KB  share/clus_apis.go
113 KB  share/scan/compliance.go
113 KB  controller/rest/system.go
107 KB  agent/probe/process.go
107 KB  controller/rest/auth.go
```

---

## 3. Build, lint, and test results

### 3.1 `go vet`
Clean. No findings on the non-vendor tree (`./controller/... ./agent/... ./share/... ./db/...
./upgrader/...`).

### 3.2 `gofmt`
**1 file** is reported by `gofmt -l`: `share/system/ns/ns_unspecified.go`. Since Go 1.17,
`gofmt` rewrites legacy `// +build` lines by adding the corresponding modern `//go:build`
directive; this file still has only `// +build !linux`, so `gofmt -w` will add the missing
`//go:build !linux` line. Trivial fix.

### 3.3 `ineffassign`
**~15 ineffectual assignments**, mostly in `agent/dp/ctrl.go` (variables `end`, `flag`, `end1`
overwritten before use across `~9` call sites) and one each in
`controller/rest/dlp_rule.go:1296` and `agent/policy/iprulesimulate.go:187`. Low risk; clean up
or add explanatory comments.

### 3.4 `go test ./...`
All packages with tests pass: **30 OK / 0 FAIL**, with 29 packages reporting "no test files" and
3 reporting "no tests to run". Total wall time ~45 s (dominated by `controller/rest` at ~36 s).

| Bucket | Count |
|---|---|
| `ok` | 30 |
| `FAIL` | 0 |
| `[no test files]` | 29 |
| `[no tests to run]` | 3 |

So roughly **~47% of non-vendor packages have no tests at all**. This is the largest gap in
quality assurance. Notable untested packages include `share/cluster`, `share/cluster/api`,
`share/cluster/watch`, `share/global`, `share/healthz`, `share/httpclient`, `share/httptrace`,
`share/k8sutils`, `share/migration`, `agent/dp`, `agent/nvbench`, `agent/pipe`, `agent/resource`,
and several `controller/nvk8sapi/...` subpackages.

### 3.5 `golangci-lint`
The repository's own configuration is intentionally narrow: only `errcheck`, `govet`,
`ineffassign`, `staticcheck`, `unused`, `nolintlint`, plus `gofmt` formatter. Many useful
linters are not enabled — see §8.4. CI pins `golangci-lint v2.12.2`; local execution against
the configured rule set requires a build of `golangci-lint` against Go 1.26 (it could not be
verified locally because the prebuilt binary was compiled with Go 1.25).

### 3.6 `staticcheck`
Could not run locally for the same Go-toolchain reason (binary built with 1.25, project needs
1.26.2). It is, however, exercised through `golangci-lint` in CI.

### 3.7 `govulncheck`
The CI workflow `.github/workflows/govulncheck.yml` runs daily and on `go.mod`/`go.sum` changes.
Local execution from this sandbox could not reach `vuln.go.dev` (network egress blocked), so
the vulnerability database lookup is **not independently verified in this report**. Trust the
scheduled CI run.

---

## 4. Cyclomatic complexity (gocyclo)

| Threshold | Count of functions exceeding |
|---|---|
| > 15 (default `gocyclo` warn) | **554** |
| > 30 (gocyclo "high") | **41** |
| > 50 (very high) | **39** |
| Average across all functions | **5.32** (healthy) |

The average is fine, but the long tail is worrying. The worst offenders:

| Cyclo | Function | Location |
|---:|---|---|
| 188 | `cache.startWorkerThread` | `controller/cache/cache.go:1674` |
| 179 | `rest.configSystemConfig` | `controller/rest/system.go:1268` |
| 177 | `main` | `controller/controller.go:214` |
| 100 | `rest.replacePolicyRule` | `controller/rest/policy.go:960` |
| 100 | `rest.(*WebhookServer).validate` | `controller/rest/admwebhook.go:959` |
| 85  | `cache.(CacheMethod).GetRiskScoreMetrics` | `controller/cache/compliance.go:206` |
| 85  | `probe.(*Probe).IsAllowedShieldProcess` | `agent/probe/process.go:3140` |
| 83  | `cache.isAdmissionRuleMet` | `controller/cache/admission.go:1345` |
| 82  | `rest.handlerAuthLogin` | `controller/rest/auth.go:2302` |
| 79  | `rest.handlerRegistryConfig` | `controller/rest/registry.go:579` |

These functions are also typically the longest (200–987 lines). Examples of "monster" functions
detected by line count:

- `controller/controller.go:main` — **983 lines**
- `controller/rest/system.go:configSystemConfig` — **652 lines**
- `agent/agent.go:main` — **587 lines**
- `controller/access/access.go:CompileUriPermitsMapping` — **503 lines**
- `controller/rest/policy.go:replacePolicyRule` — **423 lines**
- `controller/rest/admwebhook.go:WebhookServer.validate` — **384 lines**

Splitting even the top 10 of these into smaller, individually testable units would
disproportionately improve maintainability, reviewability, and test coverage.

---

## 5. Security audit

### 5.1 `gosec` summary

**407 findings** (Confidence ≥ MEDIUM, Severity ≥ MEDIUM) across 346 files / 187,608 lines.
With 1 `nosec` annotation. Severity split:

| Severity | Count |
|---:|---:|
| HIGH | 292 |
| MEDIUM | 115 |

By rule (top issues):

| Rule | CWE | Description | Count |
|---|---|---|---:|
| G115 | CWE-190 | Integer overflow conversions (`int→uint32`, `uint32→uint16`, `uint32→uint8`, `uint64→int64`, …) | **258** |
| G304 | CWE-22  | Potential file inclusion via variable (taint-style path arg to file ops) | 65 |
| G301 | CWE-276 | Directory permissions wider than `0750` | 16 |
| G204 | CWE-78  | Subprocess launched with variable / tainted arguments | 12 |
| G402 | CWE-295 | TLS issues (`InsecureSkipVerify` / `MinVersion` too low) | 11 |
| G306 | CWE-276 | `WriteFile` permissions wider than `0600` | 8 |
| G704 | CWE-918 | SSRF via taint analysis | 7 |
| G703 | CWE-22  | Path traversal via taint analysis | 5 |
| G404 | CWE-338 | Use of weak random number generator (`math/rand`) | 5 |
| G122 | CWE-367 | Race-prone path in `filepath.Walk` callback | 5 |
| G110 | CWE-409 | Decompression-bomb risk (`io.Copy` from compressed stream) | 3 |
| G117 | CWE-499 | JSON-marshalled secret-pattern field (e.g. `password`) | 2 |
| G124 | CWE-614 | Cookie missing `Secure`/`HttpOnly`/`SameSite` | 1 |
| G201 | CWE-89  | SQL via `fmt.Sprintf` | 1 |
| G401 | CWE-328 | Weak crypto primitive (`sha1.New`) | 1 |
| G407 | CWE-1204| Hardcoded / zeroed IV in `cipher.NewCFBEncrypter` | **1** |
| G505 | CWE-327 | Blocklisted import `crypto/sha1` | 1 |
| G114 | CWE-676 | `http.ListenAndServe` without read/write timeouts | 1 |
| G705 | CWE-79  | XSS via taint analysis (`w.Write(data)`) | 1 |

#### 5.1.1 Most material findings (require human triage)

These are the issues to look at first; the full list is available by re-running `gosec`.

1. **`share/utils/utils.go:935` — Hardcoded / zeroed IV (G407, HIGH).**
   Uses `cipher.NewCFBEncrypter(block, iv)` after an `iv` produced via `make([]byte,
   aes.BlockSize)` (zeroed). Combined with a `//nolint:staticcheck // SA1019` for `CFB` itself.
   Recommendation: switch this primitive to AES-GCM (or AES-CTR with a random per-message
   nonce) and rotate any persisted ciphertexts at upgrade.

2. **`controller/rest/rest.go:1993-1997` and `:2064` — TLS MinVersion too low (G402, HIGH).**
   The REST server's `tls.Config` does not enforce `tls.VersionTLS12` (or higher). Set
   `MinVersion: tls.VersionTLS12` (preferably TLS 1.3) explicitly.

3. **`controller/remote_repository/github.go:5,82` — `crypto/sha1` (G505, G401).**
   This is GitHub's blob-hash format which *must* be SHA-1 (it's how GitHub addresses blobs).
   This is a *required* protocol use; document it with a `//nolint:gosec // G505,G401: GitHub
   blob hash format requires SHA-1` so reviewers don't keep re-flagging it.

4. **`InsecureSkipVerify: true` — 27 occurrences across the tree.** Many are
   defensible (e.g., `upgrader/postsync.go` uses it to fetch a remote cert anonymously and
   already carries a `#nosec G402` annotation), but several have no such justification:
   - `controller/resource/kubernetes_resource.go:1788`
   - `controller/resource/kubernetes_auth.go:69, 110, 139, 217`
   - `controller/rest/ibmsa.go:607, 664, 777`
   - `share/orchestration/kubernetes.go:150`

   Each of these should be reviewed: either add an explicit comment + `nosec`/`nolint` with the
   reason, or wire in a configurable CA bundle. The `controller/cache/store.go:34` and
   `controller/cache/config.go:576` sites already do the right thing (configurable).

5. **`controller/rest/auth.go:490` — Session cookie missing security flags (G124).**
   `R_SESS` cookie does not set `HttpOnly`, `Secure`, or `SameSite`. For an authentication
   cookie this is a clear hardening gap; set `HttpOnly: true`, `Secure: true` (when serving
   over TLS — this is a security product so always TLS), and `SameSite: http.SameSiteLaxMode`
   (or Strict).

6. **`share/healthz/healthz.go:46` — `http.ListenAndServe` without timeouts (G114).**
   Healthz is low-traffic, but this is exactly the kind of endpoint that gets abused by
   slow-loris probes. Switch to an `http.Server{ReadHeaderTimeout, ReadTimeout, WriteTimeout,
   IdleTimeout}` instance and call `srv.ListenAndServe()`.

7. **`db/vulassets.go:871` — SQL `fmt.Sprintf` insert (G201).**
   `tableName` and `columns` come from caller-controlled data. Even if all current callers pass
   constants, prefer an allow-list check or keep this strictly internal with a comment so a
   future change cannot introduce SQL injection.

8. **`share/utils/extract.go:340`, `share/scan/scan_utils.go:630`, `share/scan/apps.go:516` —
   Decompression bomb (G110).** Container scanners frequently process untrusted archives. Wrap
   `io.Copy` with `io.LimitReader` (or `io.CopyN`) to cap output size and surface a typed error
   when exceeded.

9. **G115 — 258 integer-overflow conversions.** Most are defensive against unrealistically
   large inputs (e.g. `uint32 → uint16` for port-like fields whose source is already bounded).
   They are mostly low-risk individually, but the volume drowns out genuine issues. Add
   bounded-conversion helpers (`safeUint16(uint32) (uint16, error)`) and convert hot sites; the
   remaining sites can be `nosec`'d with a short reason. This is the single highest-ROI cleanup
   for the gosec output.

10. **Use of `math/rand`.** 6 imports across the tree (gosec G404 fires for ~5). Confirm none
    are used for security-sensitive randomness (token IDs, salts). Switch to
    `crypto/rand` for any such case.

### 5.2 `panic` calls in production code
Only **2** non-test panics, both in `share/cluster/api/event.go:97,101` (initialization-time
sanity checks). Acceptable.

### 5.3 C / C++ data-plane (`agent/`, `dp/`)
- 4 uses of `sprintf` (in `dp/dpi/dpi_debug.c` and `dp/dpi/sig/dpi_sig.c`); no `strcpy`,
  `strcat`, or `gets` were found. Replacing the `sprintf` calls with `snprintf` is mechanical
  and worth doing.
- The repository ships `.clang-tidy`, and CodeQL CI covers C++ — good.
- `cppcheck` is not preinstalled in this sandbox so additional static analysis was not run.
  Recommend adding a `cppcheck`/`scan-build` job to CI for the data plane.

### 5.4 Logging hygiene
8 `fmt.Println` calls remain in non-test production code (e.g. `controller/rest/group.go:2220`,
`controller/cache/cache.go:2089-2090`, `share/utils/utils.go:1380,1385`,
`db/vulassets.go:282`). These should be switched to the structured logger (`logrus`) used
elsewhere or removed.

### 5.5 TODO / FIXME / HACK count
**82** real TODO/FIXME/HACK comments across non-test Go (excluding the proto-generated
`XXX_unrecognized` field markers). Triage and either resolve, file as issues, or delete.

---

## 6. Project / repository hygiene

| Item | Status | Notes |
|---|---|---|
| `LICENSE` | Apache-2.0 | OK |
| `README.md` | Present | Minimal but functional; OpenSSF Scorecard badge is great. |
| `SECURITY.md` | Present | Clear vulnerability-disclosure process via SUSE Rancher. |
| `CONTRIBUTING.md` | Present | Short — could expand on local build, lint, test workflow. |
| `CODE_OF_CONDUCT.md` | Present | OK |
| `CODEOWNERS` | **Single line** (`* @neuvector/backend`) | Consider per-area owners (e.g., `dp/` to dataplane team, `controller/cache/` etc.) for better review routing. |
| `.pre-commit-config.yaml` | Present (golangci-lint hook) | Good. |
| `.golangci.yml` | Narrow ruleset | See §8.4. |
| `.clang-tidy` | Present | Good. |
| `renovate` / dependency PRs | Active | Recent commit: `chore(deps): update module github.com/glebarez/go-sqlite to v1.22.0`. |
| Pinned action SHAs in workflows | Yes | Every action call has SHA + version comment. |
| Release workflow | Present | `release.yml`. |

### 6.1 GitHub Actions workflows
Present and well-organized:

- `unitest.yaml` — `go test ./...` plus `redocly lint` of OpenAPI specs.
- `golangci-lint.yml` — pinned `v2.12.2`, 30-minute timeout.
- `codeql.yml` — Go, C++, **and Actions** language matrix (excellent).
- `govulncheck.yml` — daily + on dep changes; pinned to `v1.1.4` SHA.
- `scorecard.yml` — OpenSSF scorecard.
- `fossa.yml` — license/dependency scan.
- `renovate-vault.yml`, `updatecli.yaml` — dependency automation.

**Permissions:** workflows declare `permissions: {}` at the top level and explicit narrow scopes
on jobs — best practice.

---

## 7. Dependency posture

- `go.mod`: `go 1.26.2` (very current) with **~179 direct + indirect entries** and a
  `replace` block. Most direct deps are at recent versions
  (e.g., `golang.org/x/net v0.53.0`, `k8s.io/api v0.32.3`, `grpc v1.81.0`,
  `golang-jwt/jwt/v5 v5.3.1`, `aws-sdk-go v1.55.7`).
- A few indirects look old (e.g., `imdario/mergo v0.3.10`, `streadway/simpleuuid` from 2013,
  `codegangsta/inject` from 2015) — pinned via transitive constraints; worth verifying they
  remain reachable through the call graph or can be dropped.
- `aws-sdk-go v1` is in maintenance mode upstream; planning a migration to `aws-sdk-go-v2`
  would future-proof the AWS integration code.

---

## 8. Recommendations (prioritized)

### 8.1 P0 — Security fixes that should land soon
1. Set `MinVersion: tls.VersionTLS12` on the REST server's `tls.Config` and any other
   `tls.Config` constructed without one.
2. Replace the zeroed-IV CFB encryption in `share/utils/utils.go` with AES-GCM
   (random per-message nonce; authenticated ciphertext).
3. Set `HttpOnly`, `Secure`, and `SameSite` on the `R_SESS` cookie in `controller/rest/auth.go`.
4. Replace `http.ListenAndServe` in `share/healthz/healthz.go` with an `http.Server` that has
   read / write / idle timeouts.
5. Audit each `InsecureSkipVerify: true` site listed in §5.1 #4 — either justify and annotate or
   wire in a CA bundle.
6. Cap decompression with `io.LimitReader`/`io.CopyN` in the three G110 sites.
7. Convert the `db/vulassets.go:871` insert to use parameterized statements or an
   identifier-allow-list.

### 8.2 P1 — Quality fixes
1. Run `gofmt -w share/system/ns/ns_unspecified.go` to add the modern `//go:build` tag.
2. Clean up the 15 `ineffassign` findings (most are clustered in `agent/dp/ctrl.go`).
3. Replace 8 `fmt.Println` calls with `logrus` calls (or delete debug leftovers).
4. Replace the 4 `sprintf` calls in the C data plane with `snprintf`.
5. Triage the 82 TODO/FIXME/HACK comments — convert real work into GitHub issues.
6. For each `nolint`/`nosec` annotation, ensure a one-line rationale follows it.

### 8.3 P1 — Maintainability
1. Decompose the top-10 monster functions (§4) into smaller helpers; aim for cyclo ≤ 30 and
   ≤ 200 lines per function for these specifically.
2. Split the largest hand-written files (`controller/rest/crdsecurityrule.go`,
   `controller/rest/federation.go`, `controller/rest/system.go`, `controller/rest/auth.go`,
   `controller/kv/helper.go`, `agent/probe/process.go`) into per-feature subfiles within the
   same package.
3. Expand `CODEOWNERS` so reviews route to the team most familiar with each area
   (`dp/`, `controller/cache/`, `controller/rest/auth*`, `share/auth/`, etc.).

### 8.4 P2 — Tooling enhancements
1. Add tests for the highest-risk untested packages first: `share/cluster*`, `share/auth/oidc`,
   `share/httpclient`, `share/migration`, `controller/nvk8sapi/*`. Goal: cover ≥ 75% of
   non-vendor packages.
2. Expand `.golangci.yml` to enable (at minimum):
   - `gosec` (project-wide, not just CodeQL),
   - `bodyclose`, `errorlint`, `gocritic`, `nilnil`, `nilerr`,
   - `misspell`, `revive` (replaces `golint`),
   - `gosimple` (already in via `staticcheck` in v2 default groups),
   - `prealloc`, `unconvert`,
   - `forbidigo` to ban `fmt.Println` in production code.
3. Add a `cppcheck` (or `clang-analyzer`) job for `dp/` and `agent/` C sources.
4. Enable `go test -race -coverprofile=coverage.out` in CI and publish coverage on PRs (e.g.,
   to Codecov). Even with low absolute coverage, the trend is what matters.
5. Migrate from `aws-sdk-go` v1 to `aws-sdk-go-v2` to avoid future EOL pressure.
6. Add a `make audit` target that runs `go vet`, `gofmt -l`, `staticcheck`, `gosec`,
   `govulncheck`, and `gocyclo -over 30` so contributors can reproduce CI locally.

### 8.5 P3 — Documentation
1. Expand `CONTRIBUTING.md` with a "How to build & test locally" section
   (`apt install libpcre3-dev`, then `./unitest.sh`, then `make`).
2. Add a `docs/architecture.md` (or link to one) — the codebase has many subsystems
   (`controller`, `agent`, `dp`, `upgrader`, `monitor`, `share/cluster`) whose relationships
   are not obvious from the file tree.

---

## 9. Reproducing this audit

From a checkout (Go 1.26.2 toolchain available):

```sh
sudo apt-get install -y libpcre3-dev
go vet ./controller/... ./agent/... ./share/... ./db/... ./upgrader/...
gofmt -l agent controller share db upgrader dp
go install github.com/gordonklaus/ineffassign@latest
go install github.com/fzipp/gocyclo/cmd/gocyclo@latest
go install github.com/securego/gosec/v2/cmd/gosec@latest
GOTOOLCHAIN=go1.26.2 go install golang.org/x/vuln/cmd/govulncheck@latest

ineffassign ./controller/... ./agent/... ./share/... ./db/... ./upgrader/... ./dp/...
gocyclo -over 30 -avg agent controller share db upgrader dp
gosec -quiet -severity=medium -confidence=medium ./controller/... ./agent/... ./share/... ./db/... ./upgrader/... ./dp/...
govulncheck ./...
go test ./...
```

`golangci-lint` and `staticcheck` should be built against Go 1.26+ to match the project's
toolchain (the prebuilt `v2.6.x` releases used Go 1.25 at audit time and refuse to run).
