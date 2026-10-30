# Scrumify

Helps a Scrum Master estimate team capacity, calculate velocity, and keep track of work across sprints.

## Language

**Sprint**:
A fixed, configurable period of working days in which a team commits to and delivers work. Length is set per team (this team: 10 working days, i.e. two weeks).
_Avoid_: Iteration, cycle

**Ticket**:
A unit of work tracked in Jira, carrying a story point estimate and belonging to one or more sprints.
_Avoid_: Issue, task, story

**Velocity**:
The story points of tickets closed within a sprint, measured for the team and for each person. A closed ticket's points go to its assignee at import, and to team velocity only if unassigned.
_Avoid_: Throughput, burn

**Done Status**:
A Jira status the team counts as finished, such as Done or Closed. A ticket is closed only when its status is a done status and its Resolved date falls within a sprint, because Jira keeps the Resolved date on reopened tickets.
_Avoid_: Resolved status, closed state

**Unplaced Ticket**:
A ticket resolved on a date that falls in no known sprint, so it counts toward no velocity until a matching sprint exists.
_Avoid_: Orphan, stray

**Unestimated Ticket**:
A closed ticket with no story points. It adds zero to velocity and is flagged for review.
_Avoid_: Unpointed

**Capacity**:
The person-days a team has available in a sprint, after leave, public holidays and recurring overhead are removed.
_Avoid_: Bandwidth, availability

**Team Member**:
A person on the team's roster, to whom tickets are attributed, with an allocation percentage and optional start and end dates. Jira assignee names are mapped to team members.
_Avoid_: User, resource, developer

**Focus Factor**:
The team-wide share of working time actually available for ticket work after ceremonies and similar overhead.
_Avoid_: Utilisation, efficiency

**Reserved Days**:
Days a team member sets aside in a sprint for work outside the sprint's tickets, such as on-call or a support rotation. They reduce that person's capacity.
_Avoid_: Buffer, overhead days

**Points per Person-Day**:
Story points closed per available person-day, for a team member or the whole team, averaged over recent sprints. It converts capacity into a forecast of velocity.
_Avoid_: Rate, productivity

**Commitment**:
The story points of tickets in a sprint at its first import, excluding any tickets marked as scope change.
_Avoid_: Plan, baseline

**Scope Change**:
Tickets added to or removed from a sprint after its commitment was set. Detected from later imports, or marked on a ticket by the Scrum Master.
_Avoid_: Scope creep, churn

**Carryover Ticket**:
A ticket committed to a sprint that is not closed when the sprint ends and moves to the next one. It is not scope change, and its points count in the sprint where it closes.
_Avoid_: Rollover, spillover

**Forecast**:
The velocity predicted for a sprint before it starts, from its capacity and the team members' points per person-day. It is kept and later compared with actual velocity.
_Avoid_: Estimate, prediction

**Story Point**:
The unit of relative size the team estimates work in, recorded on a ticket. Not a measure of time.
_Avoid_: Hours, days, effort
