---
status: accepted
---

# Build scrumify in Go with a Datastar UI

We build scrumify as a Go backend with server-rendered HTML and Datastar for interactivity, using SQLite locally and PostgreSQL once hosted. We chose it over Django (the top pick of the stack research in `docs/research/stack.md`) because the owner ruled out Django and Ruby; Go was that research's runner-up, and `docs/research/stack-go-datastar.md` found it sound with caveats.

## Considered Options

- **Django, templates and htmx**: recommended by the research for its built-in login and migrations, but excluded by the owner.
- **Rails 8, SvelteKit**: scored lower in the research; Rails is also excluded by the owner.
- **Go with htmx or plain server-rendered pages**: the fallback if Datastar's risk materialises. The backend and library set stay the same, so only the template layer changes.

## Consequences

Datastar has a small maintainer base (one author wrote most 2026 commits) and no stated stability policy, so `datastar.js` is vendored and pinned, and only a small core set of its features is used. Login, sessions and migrations are assembled from small libraries instead of a framework, and the first prototype is the Chart.js wiring under Datastar. The library choices live in `docs/research/stack-go-datastar.md`, not here, because they are easy to swap.
