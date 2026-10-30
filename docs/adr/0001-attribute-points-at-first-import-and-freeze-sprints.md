---
status: accepted
---

# Attribute points to the assignee at first import and freeze closed sprints

Jira CSV exports show only a ticket's current assignee, not who held it when it closed, so per-person velocity cannot be derived exactly. We credit a closed ticket's full points to its assignee at the sprint's first import, and then freeze that attribution; a later import only adds new tickets or updates open ones, and re-attributing a sprint needs an explicit "re-import this sprint" action. This trades some accuracy (a ticket reassigned just before closing is credited to the wrong person) for stable history that later reassignments cannot silently rewrite.

## Considered Options

- **Re-derive attribution on every import**: rejected because a reassignment months later would change past sprints' per-person velocity and the forecasts built on it.
- **Split points between everyone who touched a ticket**: rejected because a CSV carries no assignee history, so the split cannot be computed.
- **Pull assignee history from the Jira API**: deferred, not rejected. The API pull is a later nice-to-have and could replace this rule if it proves worthwhile.

## Consequences

Importing at or soon after a sprint's end gives the best attribution, so the import step should show which sprints are about to be frozen.
