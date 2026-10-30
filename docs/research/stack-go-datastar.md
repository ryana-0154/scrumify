# Research: Go backend + Datastar UI for scrumify

Follow-up to ryana-0154/scrumify#2 and [stack.md](stack.md). The owner has ruled out Django and Ruby and asks whether **Go + Datastar** is a sound choice. Researched 2026-10-30 against primary sources only (data-star.dev, the `starfederation/datastar` and `starfederation/datastar-go` repos and their releases and issues, go.dev, pkg.go.dev, and each library's own repo or docs). Every fact has a link. Inferences are marked "(inference)". Anything that could not be confirmed is under [Uncertainties](#uncertainties).

Requirements are the ones in [stack.md](stack.md#requirements-used): R1 one team now, multi-tenant later; R2 Jira CSV import; R3 calculation-heavy and unit-testable; R4 local single command, login mandatory before hosting; R5 solo maintainer; R6 charts. ADR-0001 adds transactional, idempotent per-Sprint imports.

## Verdict

**Sound with caveats.** Go covers R1 to R5 well (see candidate D in stack.md), and Datastar 1.0 can do everything this app's UI needs (forms, CSV upload, tables that update, loading states, server-side validation, Chart.js) through a small, MIT-licensed script and an official Go SDK. The caveats are Datastar's very small maintainer base, its lack of a published stability promise (behaviour changed in a patch release after 1.0), and the fact that the Go SDK leaves request-size limits and multipart uploads to you. None of these block the app, but they make Go + htmx or Go + plain pages the safer option if long-term stability matters more than Datastar's SSE model.

## Findings

### 1. Datastar maturity

**Version and cadence**
- Current stable: **v1.0.4, 2026-09-21**. 1.0.0 shipped 2026-04-16, followed by 1.0.1 (04-20), 1.0.2 (06-02) and 1.0.3 (08-27). That is roughly one patch release every 1 to 3 months ([releases](https://github.com/starfederation/datastar/releases)).
- The road to 1.0 was long: 0.21.x in December 2024, 11 betas from January to March 2025, then 8 release candidates from July 2025 to March 2026 ([releases](https://github.com/starfederation/datastar/releases)).

**Stability promise and breaking-change history**
- No semver or "no breaking changes" statement was found on data-star.dev or in the repo (see Uncertainties).
- Breaking changes landed throughout the pre-1.0 line:
  - RC.1 changed SSE event handling and all SDKs, renamed `data-star-ignore` to `data-ignore`, and removed `data-on-signal-change` ([RC.1](https://github.com/starfederation/datastar/releases/tag/v1.0.0-RC.1)).
  - RC.6 changed the attribute delimiter from `-` to `:` (`data-on-click` became `data-on:click`), renamed `data-on-load` to `data-init`, and removed file-upload progress monitoring ([RC.6](https://github.com/starfederation/datastar/releases/tag/v1.0.0-RC.6)).
- **1.0.0 itself renamed an option**: `retryMaxWaitMs` became `retryMaxWait` ([v1.0.0](https://github.com/starfederation/datastar/releases/tag/v1.0.0)).
- **1.0.2, a patch release, changed default behaviour.** In-flight requests are now cancelled by method + URL regardless of the element that started them, and checkbox/radio `data-bind` now syncs on `input` instead of `change` ([v1.0.2](https://github.com/starfederation/datastar/releases/tag/v1.0.2)).

**Licence and Pro tier**
- Core: **MIT** ([GitHub repo metadata](https://github.com/starfederation/datastar)).
- A paid **Datastar Pro** tier exists: Solo $349, Team $1,299, Enterprise custom, one-time lifetime licence, no redistribution in public repos ([data-star.dev/pro](https://data-star.dev/pro)).
- Pro-only attributes are `data-animate`, `data-custom-validity`, `data-match-media`, `data-on-raf`, `data-on-resize`, `data-persist`, `data-query-string`, `data-replace-url`, `data-scroll-into-view` and `data-view-transition` ([attributes](https://data-star.dev/reference/attributes)).
- Pro-only actions are `@clipboard`, `@fit` and `@intl`. All backend actions (`@get`, `@post` and so on) are free ([actions](https://data-star.dev/reference/actions)).
- Nothing this app needs is Pro-only.

**Maintainers (bus-factor risk)**
- All-time commits to `starfederation/datastar`: bencroker 675, delaneyj 371, khalwat 69, and every other contributor 44 or fewer ([contributors API](https://api.github.com/repos/starfederation/datastar/contributors)).
- **Since 2026-01-01, 55 of 59 commits are by one person (bencroker)** ([commits API](https://api.github.com/repos/starfederation/datastar/commits?since=2026-01-01T00:00:00Z)).
- The repo has about 5.2k stars and 26 open issues ([repo](https://github.com/starfederation/datastar)).
- The effective bus factor is 1 to 2.

**How it covers what this app needs**

| Need | How Datastar does it | Source |
|---|---|---|
| CSV file upload | Option 1: `data-bind` on a file input base64-encodes the files into a `{name, contents, mime}[]` signal and sends it as JSON. Option 2: a `<form enctype="multipart/form-data">` submitted with `@post(url, {contentType: 'form'})`, which the docs recommend for server uploads. **Use option 2** (inference): base64-in-JSON inflates the payload, and the Go SDK's `ReadSignals` reads the whole body with no size limit (see Q2). | [attributes](https://data-star.dev/reference/attributes), [actions](https://data-star.dev/reference/actions), [file_upload example](https://data-star.dev/examples/file_upload) |
| Forms | `contentType: 'form'` finds the closest form, validates its elements and sends them as form data. The default sends all signals as JSON. | [actions](https://data-star.dev/reference/actions), [form_data example](https://data-star.dev/examples/form_data) |
| Tables that update | The server sends `datastar-patch-elements` events that morph HTML by id or selector (modes: outer, inner, append, prepend, remove and others). Examples cover edit row, bulk update, delete row, active search and infinite scroll. | [Go SDK elements-sugar.go](https://github.com/starfederation/datastar-go/blob/main/datastar/elements-sugar.go), [examples](https://data-star.dev/examples) |
| Charts (Chart.js) | No Chart.js example exists. The official chart example uses **Rocket** (custom elements) with ECharts, and Rocket is "currently in beta and its API is subject to change". 11 of the 22 open issues are Rocket bugs. **Plain approach** (inference): put the `<canvas>` under `data-ignore-morph` so patches do not wipe it, then either re-patch the whole chart element with fresh JSON or patch a signal and redraw from `data-effect`. | [rocket_echarts](https://data-star.dev/examples/rocket_echarts), [rocket reference](https://data-star.dev/reference/rocket), [attributes](https://data-star.dev/reference/attributes), [open issues](https://github.com/starfederation/datastar/issues) |
| Loading / progress | `data-indicator` sets a signal to true "while a fetch request is in flight". One SSE response can stream several patches, so a long CSV import can push progress as it goes ([progress_bar example](https://data-star.dev/examples/progress_bar)). Upload-byte progress was **removed** in RC.6, and the docs recommend a file-upload library instead. | [attributes](https://data-star.dev/reference/attributes), [RC.6](https://github.com/starfederation/datastar/releases/tag/v1.0.0-RC.6) |
| Server-side validation errors | The server answers with patched elements (error messages) or patched signals. The inline-validation example debounces input and calls `@query`. The docs say to "always implement backend validation". Default `retry: 'auto'` retries on network errors only; `'error'` also retries 4xx/5xx. The docs do not say how a 4xx body is handled, so return 200 + SSE patches for validation errors (inference). | [inline_validation](https://data-star.dev/examples/inline_validation), [security](https://data-star.dev/reference/security), [actions](https://data-star.dev/reference/actions) |

**Other facts that matter**
- Backend actions accept `text/event-stream`, `text/html`, `application/json` (signals) and `text/javascript` responses ([actions: response handling](https://data-star.dev/reference/actions)).
- Every request carries a `Datastar-Request: true` header ([actions](https://data-star.dev/reference/actions)).
- Expressions run through the `Function()` constructor, so the default CSP needs `unsafe-eval`. A nonce-based CSP mode exists. Never interpolate user data into expressions; pass it through signals instead ([security](https://data-star.dev/reference/security)).

### 2. Datastar Go SDK

- **Official.** It lives in the core `starfederation` org, and data-star.dev lists it with Delaney Gillilan as maintainer. data-star.dev does not label any SDK "official" in words ([SDK list](https://data-star.dev/reference/sdks), [repo](https://github.com/starfederation/datastar-go)). MIT licence, requires Go 1.24+ ([README](https://github.com/starfederation/datastar-go/blob/main/README.md)).
- **Version: v1.2.2, 2026-06-02** ([proxy.golang.org](https://proxy.golang.org/github.com/starfederation/datastar-go/@latest)). Tags: v1.0.0 2025-07-14, v1.1.0 2025-12-15, v1.2.0 2026-04-16, the same day as Datastar 1.0.0 ([tags](https://github.com/starfederation/datastar-go/tags)).
- **Small project.** About 209 stars, 7 open issues, last push 2026-06-02. Top contributors: delaneyj 9 commits, bencroker 7 ([repo](https://github.com/starfederation/datastar-go), [contributors](https://api.github.com/repos/starfederation/datastar-go/contributors)).
- **API surface** ([pkg.go.dev](https://pkg.go.dev/github.com/starfederation/datastar-go/datastar), [source](https://github.com/starfederation/datastar-go/tree/main/datastar)):
  - `ReadSignals(r, &v)`: reads JSON from the `datastar` query parameter on GET and DELETE, otherwise from the body.
  - `NewSSE(w, r, opts...)`: sets SSE headers and flushes. Options include `WithContext` and compression.
  - `PatchElements(html, opts...)` with `WithSelector`, `WithSelectorID`, `WithModeInner/Outer/Append/Prepend/Remove/...`, `WithViewTransitions`; `PatchElementf`, `PatchElementTempl` (templ), `PatchElementGostar`, `RemoveElement*`.
  - `PatchSignals(bytes)`, `MarshalAndPatchSignals(v)`, `WithOnlyIfMissing`.
  - `ExecuteScript`, `Redirect`, `DispatchCustomEvent` and similar, plus attribute helpers `GetSSE`/`PostSSE`/`PutSSE`/`PatchSSE`/`DeleteSSE`.
- **Testing story.** `NewSSE` writes to a plain `http.ResponseWriter`, and the SDK's own tests use `httptest.NewRecorder()` ([sse_test.go](https://github.com/starfederation/datastar-go/blob/main/datastar/sse_test.go)). Handler tests can therefore assert on the recorded SSE text. There are no dedicated assertion helpers.
- **Gaps and sharp edges:**
  - `ReadSignals` reads the full body with no size limit. A `ReadSignalsWithLimit` was proposed in issue [#11](https://github.com/starfederation/datastar-go/issues/11) (open since 2025-12-12). Wrap bodies in [`http.MaxBytesReader`](https://pkg.go.dev/net/http#MaxBytesReader) yourself.
  - No multipart helper. Use stdlib `r.ParseMultipartForm` for CSV uploads (inference).
  - `NewSSE` **panics** if the `ResponseWriter` cannot flush ([sse.go](https://github.com/starfederation/datastar-go/blob/main/datastar/sse.go)). Middleware that wraps the writer without exposing `Flush`/`Unwrap` will crash handlers (inference from the code).
  - Compression middleware can garble SSE ([#21](https://github.com/starfederation/datastar-go/issues/21), open).
  - Several `DispatchCustomEvent` calls in a row cause JS errors ([#19](https://github.com/starfederation/datastar-go/issues/19), open).
  - The SDK pulls in `CAFxX/httpcompression`, `klauspost/compress`, `valyala/bytebufferpool` and `santhosh-tekuri/jsonschema` as direct requirements ([go.mod](https://github.com/starfederation/datastar-go/blob/main/go.mod)).
  - A feature-by-feature comparison with the TypeScript SDK was not done (see Uncertainties).

### 3. Gaps in Go's standard library

Baseline: Go 1.27.3 is current (2026-10-19), and "each major Go release is supported until there are two newer major releases" ([release history](https://go.dev/doc/devel/release)). Stack.md already covers `ServeMux` routing, `encoding/csv` and `time/tzdata`.

| Gap | Recommendation | Why |
|---|---|---|
| **HTML templating** | **`html/template`** (stdlib). Render a fragment with `ExecuteTemplate` into a buffer, then pass it to `PatchElements`. Switch to **templ** (v0.3.1100, 2026-10-26, MIT) only if untyped template data causes real bugs. | `html/template` needs no dependency, no codegen and falls under Go 1 compatibility. It escapes by context: it strips the `data-` prefix, so `data-on:*` values get JS escaping, while `data-text`/`data-show` get only attribute escaping ([attr.go](https://github.com/golang/go/blob/master/src/html/template/attr.go)). So keep Datastar expressions static, whichever template engine you use. templ compiles components to Go ([templ.guide](https://templ.guide/)) and the SDK has `PatchElementTempl`, but templ is still pre-1.0 ([releases](https://github.com/a-h/templ/releases)) and adds a generate step. |
| **SQLite driver** | **`modernc.org/sqlite` v1.61.0** (pure Go, cgo-free) ([proxy](https://proxy.golang.org/modernc.org/sqlite/@latest)) | Keeps `CGO_ENABLED=0` single-binary builds. goose itself depends on it ([goose go.mod](https://github.com/pressly/goose/blob/main/go.mod)). It is 1.3x to 2x slower than C SQLite on CPU-bound benchmarks ([pkg.go.dev](https://pkg.go.dev/modernc.org/sqlite)), which is irrelevant at one team's load. |
| **PostgreSQL path** | **`jackc/pgx` v5.11.0** through its `database/sql` adapter `pgx/v5/stdlib` ([releases](https://github.com/jackc/pgx/releases), [stdlib pkg](https://pkg.go.dev/github.com/jackc/pgx/v5/stdlib)) | The same `*sql.DB` interface as SQLite, so the app code above the query layer does not change. |
| **SQL access** | **sqlc v1.31.1** (MIT), generating typed Go from hand-written SQL ([releases](https://github.com/sqlc-dev/sqlc/releases)) | Plain functions over `*sql.DB`/`*sql.Tx`, no ORM. The generated `WithTx` covers ADR-0001's transactional imports ([transactions](https://docs.sqlc.dev/en/latest/howto/transactions.html)). **Caveat**: Go + SQLite support is **Beta**, Go + PostgreSQL is Stable ([language support](https://docs.sqlc.dev/en/latest/reference/language-support.html)). `engine` is set per `sql` block, so supporting both databases means two query sets ([config](https://docs.sqlc.dev/en/latest/reference/config.html)). The fallback is plain `database/sql`. Not an ORM: an ORM would add the largest dependency for the least gain in a calculation app. |
| **Migrations** | **pressly/goose v3.29.0** (MIT), with migrations embedded via `embed` and run at startup ([releases](https://github.com/pressly/goose/releases), [README](https://github.com/pressly/goose)) | Supports SQLite and Postgres, and sqlc reads goose files as schema, skipping down migrations ([sqlc ddl](https://docs.sqlc.dev/en/latest/howto/ddl.html)). |
| **Sessions** | **alexedwards/scs v2.9.0** (MIT) with `sqlite3store`, later `postgresstore`/`pgxstore` ([repo](https://github.com/alexedwards/scs)) | `sqlite3store` imports only `database/sql`, so it works with modernc (inference: its go.mod still names mattn) ([source](https://github.com/alexedwards/scs/blob/master/sqlite3store/sqlite3store.go)). Documents `RenewToken` on login/logout against session fixation, plus `HttpOnly`, `Secure` and `SameSite` cookies ([README](https://github.com/alexedwards/scs#preventing-session-fixation)). Last release 2025-07-08, so it is slow-moving. |
| **Password login** | **`golang.org/x/crypto/argon2`** `IDKey` (Argon2id) ([pkg](https://pkg.go.dev/golang.org/x/crypto/argon2)) | OWASP recommends Argon2id with at least 19 MiB memory, 2 iterations and parallelism 1. bcrypt is for legacy use only, with work factor 10+ and a 72-byte input limit ([OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)). |
| **CSRF with SSE/Datastar** | **`net/http.CrossOriginProtection`** (stdlib, Go 1.25+) wrapping the mux ([pkg](https://pkg.go.dev/net/http#CrossOriginProtection), [Go 1.25 notes](https://go.dev/doc/go1.25)) | Datastar actions are same-origin `fetch` calls ([actions](https://data-star.dev/reference/actions)), so browsers send `Sec-Fetch-Site`/`Origin` and no token plumbing is needed. It rejects cross-origin non-safe methods and always allows GET/HEAD/OPTIONS. Datastar's docs give no CSRF guidance beyond a `headers` option example ([actions](https://data-star.dev/reference/actions), [security](https://data-star.dev/reference/security)). **Rule: `@get` handlers must never change state.** |
| **Config** | stdlib `flag` + `os.Getenv`, no library | Only a few settings (listen address, DB URL, session key, auth on/off). A library is not worth the dependency (inference). |
| **Testing helpers** | stdlib `testing`, `net/http/httptest`, `testing/synctest` (GA in Go 1.25) ([Go 1.25](https://go.dev/doc/go1.25)) | Calculations are pure functions with table tests. Handlers are tested with `httptest.NewRecorder` (as the SDK does). The repository layer runs against a temp-file SQLite per test (inference). |

**Minimum safe auth before hosting** (built from the rows above):
1. Argon2id hashes stored with salt and parameters.
2. scs sessions with `RenewToken` on login and logout, and `Secure` + `HttpOnly` + `SameSite=Lax` cookies.
3. `CrossOriginProtection` on the mux.
4. Login rate limiting, for example [`golang.org/x/time/rate`](https://pkg.go.dev/golang.org/x/time/rate) per IP and account.
5. Accounts created by an admin or by invite only (no open sign-up), plus HTTPS (inference).

### 4. Multi-tenancy without a rewrite

Approach (inference, built on the tools above):
1. **Schema.** Every table gets `team_id NOT NULL` from the first migration, and unique keys include it (e.g. `UNIQUE(team_id, jira_key)`). While there is one team, a seed row holds that team.
2. **Queries.** Every sqlc query takes `team_id` as a parameter, so the generated Go functions cannot be called without one. A grep or test in CI fails if a query on a tenant table lacks `team_id`.
3. **Request scope.** Middleware resolves the team from the session (or the single default team when auth is off) and puts a `TeamID` type in `context`. Handlers pass it down, and the calculation package never sees tenancy.
4. **Defence in depth on Postgres.** Add row-level security policies keyed on a per-transaction setting. Table owners bypass RLS unless `FORCE ROW LEVEL SECURITY` is set ([Postgres RLS](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)).
5. **Tests.** One isolation test per repository function: seed two teams and assert no cross-team rows come back.

Moving from SQLite to Postgres costs one extra sqlc engine block and goose migrations ported to Postgres syntax. Handlers and calculations stay as they are.

## Recommended minimal library set

| Concern | Choice | Version (2026-10-30) |
|---|---|---|
| Language / router / templates / CSV / CSRF / tests | Go stdlib (`net/http`, `html/template`, `encoding/csv`, `CrossOriginProtection`, `testing`) | Go 1.27.x |
| UI | Datastar JS (vendored, embedded in the binary) + `github.com/starfederation/datastar-go` | 1.0.4 / v1.2.2 |
| Charts | Chart.js, vendored, under `data-ignore-morph` | see stack.md |
| DB drivers | `modernc.org/sqlite`; later `github.com/jackc/pgx/v5/stdlib` | v1.61.0 / v5.11.0 |
| SQL + migrations | sqlc (dev tool only) + `github.com/pressly/goose/v3` | v1.31.1 / v3.29.0 |
| Sessions + passwords | `github.com/alexedwards/scs/v2` + `golang.org/x/crypto/argon2` | v2.9.0 / x/crypto v0.58.0 |

## Risks

1. **Bus factor and pace (biggest risk).** One person wrote 55 of 59 core commits in 2026, and the Go SDK has two main contributors and no release since June ([commits](https://api.github.com/repos/starfederation/datastar/commits?since=2026-01-01T00:00:00Z), [SDK tags](https://github.com/starfederation/datastar-go/tags)). *Mitigation*: vendor and pin `datastar.js` inside the binary, and use only a small core (`data-on`, `data-bind`, `data-indicator`, `data-ignore-morph`, `@get`/`@post`, `PatchElements`/`PatchSignals`). The SSE wire format is simple enough to emit by hand if the SDK stalls (inference).
2. **No stability promise.** 1.0.2 changed defaults in a patch release ([v1.0.2](https://github.com/starfederation/datastar/releases/tag/v1.0.2)). *Mitigation*: upgrade deliberately, read the release notes, and keep a handful of end-to-end checks.
3. **Charts are off the beaten path.** The only official chart example uses beta Rocket, which has 11 open bugs. Chart.js wiring is your own code (see Q1).
4. **Upload hardening is on you.** No body limit in `ReadSignals`, no multipart helper, and no upload-byte progress (Q1, Q2).
5. **Operational sharp edges.** `NewSSE` panics on writers that cannot flush, compression middleware can garble SSE ([#21](https://github.com/starfederation/datastar-go/issues/21)), and the default CSP needs `unsafe-eval` ([security](https://data-star.dev/reference/security)).

**When Go + htmx or Go + plain pages would be the better choice:**
- **Go + plain server-rendered pages + Chart.js.** The core of this app is read-mostly: import a CSV, view Sprint history, view charts. Full-page POST/redirect/GET covers that with zero UI dependencies and the strongest stability. Choose this if interactivity stays at "submit a form, see a page".
- **Go + htmx.** htmx is far more widely used (about 49.6k stars vs 5.2k) ([htmx repo](https://github.com/bigskysoftware/htmx), [datastar repo](https://github.com/starfederation/datastar)). Its 2.x line still gets patches (2.0.10, 2026-09-06) even after 4.0.0 shipped on 2026-08-28 ([htmx releases](https://github.com/bigskysoftware/htmx/releases)). Responses are plain HTML, so no SDK is needed. Choose htmx if you value ecosystem and examples over Datastar's signals and SSE streaming.
- **Datastar earns its place only if** you want server-streamed progress (long imports), live what-if sliders (Capacity, Focus Factor) driven by signals, or many partial updates from one request. If none of these appear in the first milestones, Datastar is more framework than the app needs.

## Uncertainties

1. No formal Datastar stability or semver policy was found. Its absence is inferred from searching the site and repo, not from a statement.
2. The Go SDK was not compared feature by feature with the TypeScript SDK. "No major gaps beyond those listed" is unverified.
3. How Datastar handles a 4xx/5xx response body is not documented on the pages read.
4. That `alexedwards/scs/sqlite3store` works with `modernc.org/sqlite` is inferred from its imports. Not tested.
5. sqlc's named parameters (`sqlc.arg`) behaving the same on SQLite and Postgres was not verified. Expect some query duplication between engines.
6. How templ escapes `data-*` attribute values was not checked. The `html/template` behaviour comes from the Go source.
7. The Chart.js + `data-ignore-morph` + `data-effect` wiring is a design inference, not a documented Datastar pattern. Prototype it before committing.
8. Whether Rocket is free or Pro is not stated on the Rocket or Pro pages.
9. Star counts and commit counts are snapshots from the GitHub API on 2026-10-30.
