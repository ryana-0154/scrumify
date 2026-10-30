# Research: stack for scrumify

Issue: ryana-0154/scrumify#2. Researched 2026-10-30 against official docs and support policies. Every factual claim has a source link; anything not confirmed by a primary source is in [Uncertainties](#uncertainties).

Judging rule: candidates are scored only against the requirements in the issue. No language, framework or tooling preference from the owner's other projects was used as a requirement or tie-breaker.

## Recommendation (short)

**Django (Python) with server-rendered templates + htmx + Chart.js, SQLite locally, PostgreSQL when hosted.**
Runner-up: **Go standard library single binary** (candidate D).
Main flip condition: if "small dependency surface" and "single binary, no runtime" outweigh "auth, migrations and admin built in", pick Go.

## Requirements used

From the issue, plus the extra criteria it asks for:

| # | Requirement |
|---|---|
| R1 | Standalone web app for one team, multi-tenant later without a rewrite (every record belongs to a Team) |
| R2 | Jira CSV import, read-only, keep a few fields, discard the rest |
| R3 | Calculation-heavy domain (Velocity, Capacity, Points per Person-Day over 6 Sprints, Forecast vs actual), easy to unit test |
| R4 | Local first, single command, no auth; login mandatory before hosting |
| R5 | Solo maintainer: low ops burden, small dependency surface, cheap hosting, long-term support |
| R6 | Read and chart data: Sprint history, Forecast vs actual, simple burndown |
| X1 | Ecosystem maturity for CSV, date, time-zone handling |
| X2 | Testing story |
| X3 | Hosting options and cost for a small SaaS |
| X4 | Hiring and community size |
| X5 | Migration path single-team to multi-tenant |

ADR-0001 (freeze attribution at first import) adds one implicit need: imports must be transactional and idempotent per Sprint, which any stack with a relational database and transactions satisfies.

## Shortlist

| | Language | Web framework | Storage | Front end |
|---|---|---|---|---|
| **A** | Python | Django 6.x | SQLite local, PostgreSQL hosted (one ORM) | Server-rendered templates + htmx + Chart.js |
| **B** | Ruby | Rails 8.x | SQLite local, PostgreSQL hosted (Active Record) | Server-rendered (Hotwire default) + Chart.js |
| **C** | TypeScript | SvelteKit on Node LTS (adapter-node) | SQLite (`node:sqlite` or a driver), PostgreSQL hosted | Svelte components + Chart.js |
| **D** (different in kind) | Go | Standard library only (`net/http`, `html/template`, `database/sql`) | SQLite via cgo-free driver, PostgreSQL hosted | Server-rendered templates + htmx + Chart.js, all embedded in one binary |

Why D is different in kind: A, B and C are "batteries-included framework on an interpreted runtime". D has no framework and no runtime to install; it is assembled from the standard library and ships as one compiled file with assets embedded ([embed in std](https://pkg.go.dev/std)).

Excluded: a separate SPA + JSON API pair (two deployables, doubles the dependency surface for a read-mostly app, against R5); ASP.NET Core and Phoenix (viable, but not evaluated here to keep the shortlist to 4; see Uncertainties).

## Facts per candidate

### A. Django

- Support: each Django feature release gets about three years of security and data-loss fixes; from Django 2028 every release gets one year mainstream plus two years extended support ([Django download page](https://www.djangoproject.com/download/), [release process](https://docs.djangoproject.com/en/6.1/internals/release-process/)). 5.2 LTS extended support ends April 2028; 6.2 LTS (April 2027) to April 2030 ([download page](https://www.djangoproject.com/download/)).
- Deprecations warn across at least two feature releases before removal ([release process](https://docs.djangoproject.com/en/6.1/internals/release-process/)).
- Python versions get five years of support; 3.14 to 2030-10, 3.15 to 2031-10 ([Python devguide](https://devguide.python.org/versions/)).
- Auth built in and enabled in new projects: users, permissions, groups, password hashing, sessions, login views ([Django auth](https://docs.djangoproject.com/en/6.1/topics/auth/)). Login throttling, OAuth and object-level permissions need third-party packages (same source).
- Officially supports SQLite 3.37+ and PostgreSQL 15+ with one ORM ([Django databases](https://docs.djangoproject.com/en/6.1/ref/databases/)).
- Time zones: `USE_TZ = True` in new projects, stores UTC, uses stdlib `zoneinfo` with IANA keys ([Django time zones](https://docs.djangoproject.com/en/6.1/topics/i18n/timezones/)).
- CSV: stdlib `csv.DictReader` maps rows to dicts by header and handles quoted fields with embedded newlines when opened with `newline=''` ([Python csv](https://docs.python.org/3/library/csv.html)). Selecting a few fields and dropping the rest is a dict comprehension, no dependency.
- `zoneinfo` (stdlib since 3.9) uses system tz data and falls back to the first-party `tzdata` package ([Python zoneinfo](https://docs.python.org/3/library/zoneinfo.html)).
- Testing: `manage.py test` on `unittest`, separate test database (in-memory with SQLite), per-test transactions ([Django testing](https://docs.djangoproject.com/en/6.1/topics/testing/overview/)). Pure calculation code is plain Python and needs no Django to test.

### B. Rails 8

- Support: bug fixes for one year and security fixes for two years per minor release; a new feature release about every six months. 8.0 security support ends 2026-11-07, 8.1 ends 2027-10-10 ([Rails maintenance policy](https://rubyonrails.org/maintenance)).
- Ruby: 3.3 security maintenance to 2027-03-31 (expected); 3.2 EOL 2026-04-01 ([Ruby branches](https://www.ruby-lang.org/en/downloads/branches/)).
- Rails 8 ships an authentication generator (User, Session, password reset; no sign-up) ([Rails security guide](https://guides.rubyonrails.org/security.html)).
- Rails 8 ships Kamal 2 deploy, Thruster proxy, and database-backed Solid Queue/Cache/Cable that remove the need for Redis; Solid Queue works with SQLite ([Rails 8.0 release notes](https://guides.rubyonrails.org/8_0_release_notes.html)).

### C. SvelteKit on Node

- Node LTS lines get about 30 months of critical fixes; 24.x EOL 2028-04-30, 26.x EOL 2029-04-30 ([Node previous releases](https://nodejs.org/en/about/previous-releases), [nodejs/release schedule](https://github.com/nodejs/release)).
- `adapter-node` builds a server started with `node build` ([SvelteKit adapter-node](https://svelte.dev/docs/kit/adapter-node)).
- `node:sqlite` is built in, no flag since 22.13/23.4, status "Release candidate" (1.2) since 25.7 ([Node sqlite](https://nodejs.org/api/sqlite.html)).
- No CSV parser in the Node standard library; a third-party package is needed ([Node API index](https://nodejs.org/api/)).
- `node:test` runner is stable since 20.0 and can mock `Date` and timers ([Node test](https://nodejs.org/api/test.html)).
- Temporal (date/time API) is TC39 Stage 4 and listed as shipped in Node 26 ([Temporal proposal](https://github.com/tc39/proposal-temporal)).
- No auth, ORM or migrations in SvelteKit itself (see Uncertainties: not confirmed from a SvelteKit doc page).

### D. Go standard library

- Each major Go release is supported until two newer major releases exist; latest is 1.27.3 (2026-10-19) ([Go release history](https://go.dev/doc/devel/release)).
- Go 1 compatibility promise: programs keep compiling and running across Go 1 releases, at source level, with exceptions for security and bug fixes ([Go 1 compatibility](https://go.dev/doc/go1compat)).
- Since Go 1.22, `net/http.ServeMux` supports method and wildcard patterns such as `GET /items/{id}` ([Go 1.22 notes](https://go.dev/doc/go1.22)), so no router dependency is needed.
- Std has `html/template`, `encoding/csv`, `database/sql`, `embed`, `testing`; it has no package for sessions, login or schema migrations ([Go std](https://pkg.go.dev/std)).
- `encoding/csv` implements RFC 4180, handles quoted newlines, returns `[]string` per record (no header-to-map reader) ([encoding/csv](https://pkg.go.dev/encoding/csv)).
- `time.LoadLocation` uses system tz data, with `time/tzdata` as an embeddable fallback ([time](https://pkg.go.dev/time#LoadLocation)).
- `modernc.org/sqlite` is a cgo-free SQLite driver for `database/sql`, about 1.3x to 2x slower than C on CPU-bound benchmarks ([modernc sqlite](https://pkg.go.dev/modernc.org/sqlite)).

### Shared facts (all candidates)

- SQLite: "fewer than 100K hits/day should work fine"; one writer at a time per database; avoid network filesystems ([SQLite when to use](https://www.sqlite.org/whentouse.html)). In WAL mode readers and the writer run concurrently ([SQLite WAL](https://www.sqlite.org/wal.html)).
- PostgreSQL majors are supported five years; 18 is current, EOL 2030-11-14 ([Postgres versioning](https://www.postgresql.org/support/versioning/)).
- PostgreSQL row-level security gives default-deny per-row policies; table owners bypass unless `FORCE ROW LEVEL SECURITY` ([Postgres RLS](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)). Usable with any candidate as tenant-isolation defence in depth.
- htmx is dependency-free and needs no build system ([htmx docs](https://htmx.org/docs/)). Chart.js is MIT-licensed and loads from a script tag with no build step ([Chart.js](https://www.chartjs.org/docs/latest/), [getting started](https://www.chartjs.org/docs/latest/getting-started/)).
- Example small host: Fly.io shared-cpu-1x 256 MB at $2.19/month, volumes $0.15/GB/month, managed Postgres from $38/month ([Fly.io pricing](https://docs.fly.io/about/pricing)).
- Community, Stack Overflow 2025 survey, professional developers: Python 54.8%, TypeScript 48.8%, Go 17.4%, Ruby 6.9%; Django 11.7%, Rails 6.2%, Svelte 6.9% ([SO survey 2025](https://survey.stackoverflow.co/2025/technology)).

## Comparison table

Scores: ++ strong, + adequate, - weak. Each cell's basis is in the facts section above.

| Requirement | A Django | B Rails 8 | C SvelteKit/Node | D Go stdlib |
|---|---|---|---|---|
| R1 one team, multi-tenant later | ++ ORM + migrations; `team` FK on every model | ++ same via Active Record | + needs chosen ORM/migration tool | + hand-written SQL + migration tool of choice |
| R2 Jira CSV, keep few fields | ++ stdlib `DictReader` | + stdlib CSV (not verified here) | - third-party parser needed | + stdlib `encoding/csv`, map headers by hand |
| R3 calc-heavy, unit-testable | ++ plain functions + `unittest`/Django test DB | ++ minitest built in (not verified here) | + `node:test` stable, Date mocking | ++ `testing` in std, table tests, typed |
| R4 local single command, auth later | ++ `runserver`; auth built in, add login before hosting | ++ `bin/rails server`; auth generator | + `node build`; auth is DIY or library | ++ one binary; - auth/sessions DIY |
| R5 low ops, small deps, cheap, long support | + 3-year releases, Python 5 years; few deps | - 2-year security per minor, 6-month cadence, 8.0 EOL next week | - Node 30 months, framework + npm tree churn | ++ compatibility promise, no runtime; - only ~1 year per Go release (upgrades are routine) |
| R6 charts | ++ templates + Chart.js | ++ same | ++ Svelte + Chart.js | ++ templates + Chart.js |
| X1 CSV/date/TZ maturity | ++ csv, zoneinfo, Django `USE_TZ` | + (not verified here) | + Temporal in Node 26; CSV third-party | ++ csv, time, tzdata embeddable |
| X2 testing | ++ | ++ | + | ++ |
| X3 hosting cost | ++ any container host; SQLite on a volume ~$2-3/mo | ++ Kamal to a VPS or any container host | ++ any Node host | ++ smallest image, any host |
| X4 community/hiring | ++ Python 54.8%, Django 11.7% | - Ruby 6.9%, Rails 6.2% | ++ TS 48.8%, Svelte 6.9% | + Go 17.4% |
| X5 single to multi-tenant | ++ same ORM on SQLite and Postgres; add RLS | ++ same | + depends on ORM picked | + SQL dialect differences SQLite vs Postgres are yours to manage |

## Recommendation

**A: Django + templates + htmx + Chart.js, SQLite locally, PostgreSQL when hosted.**

Why:
1. It covers R4's later requirement (mandatory login) and R1/X5 (one ORM and migrations across SQLite and PostgreSQL) with built-in, officially supported parts, so the solo maintainer writes and maintains the least non-domain code.
2. R2 and X1 need no third-party library: `csv`, `zoneinfo`, aware datetimes are stdlib or core.
3. R3: the calculation core can be plain functions with no framework imports, tested in milliseconds; the import (ADR-0001 freeze) gets transactional tests on a throwaway database.
4. R5: three-year support windows with two-release deprecation warnings mean an upgrade roughly every 1 to 3 years, not every 6 months (contrast Rails).

Trade-offs accepted:
- Needs a Python runtime and a few packages (Django, a Postgres driver, possibly `tzdata`), so more moving parts than D's single binary.
- Django is slower than Go, which does not matter at one team's traffic (SQLite's own guidance covers far more).
- Dynamic typing: the calculation core should carry type hints and tests to catch unit mix-ups (points vs person-days).

What would change it:
- **Go (D) wins** if the owner ranks "smallest dependency surface / single file to run and ship" above "auth, sessions, migrations, admin built in", or wants to distribute scrumify as a downloadable binary to other Scrum Masters. Cost: write login, sessions, CSRF and migrations yourself, or add libraries for them.
- **SvelteKit (C) wins** if the UI becomes highly interactive (drag-and-drop planning, live what-if Capacity sliders) beyond what htmx handles comfortably.
- **Rails (B) wins** only if the owner already maintains Rails apps; on requirements alone its six-month release cadence and two-year security windows cost a solo maintainer more upgrades.

### Draft ADR: not needed

The choice is hard to reverse (language and framework), but not surprising: it is a conventional framework picked for built-in auth and ORM. Per the issue, a draft ADR is required only when both hold. The owner may still record it as an ADR once approved.

## Uncertainties

1. Ruby `CSV` and Rails' minitest were not checked against primary docs in this pass; Rails scores on R2, R3, X1 assume them.
2. SvelteKit's lack of built-in auth/ORM is from general knowledge, not a fetched SvelteKit page.
3. Node LTS EOL dates came from the `nodejs/release` repo, which says dates may change.
4. Ruby 3.4/4.0 EOL dates are listed as TBD by ruby-lang.org.
5. Hosting price is one provider (Fly.io) as an example; other hosts were not compared.
6. Stack Overflow survey percentages are usage among respondents, a proxy for hiring pool, not a measure of it.
7. ASP.NET Core (21.3% in the same survey) and Elixir Phoenix were not evaluated; .NET has an LTS policy that may score well on R5 and was left out only to keep the shortlist to four.
8. Recommending Python coincides with the owner's habitual stack. That was not an input; the deciding factors are listed under "Why" and could be re-checked by someone ignoring this note.
