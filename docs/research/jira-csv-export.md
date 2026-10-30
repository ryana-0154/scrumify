# Research: Jira CSV export format

Issue: ryana-0154/scrumify#1. Researched 2026-10-30 against Atlassian's official docs, knowledge-base (KB) articles and public bug tracker (jira.atlassian.com). Every fact has a source link. Where only Atlassian Community posts exist, the doc says so and names who answered. Anything not confirmed by a primary source is in [Uncertainties](#uncertainties).

Scope: only the fields scrumify keeps (ticket key, status, story points, assignee, Resolved date, sprint names). Vocabulary follows `GLOSSARY.md`; Jira's own word "issue" / "work item" means our **Ticket**.

## Summary

- **Cloud and Data Center (DC) both export CSV with a header row; a multi-valued field becomes several columns with the same header.** Expect two or more `Sprint` columns for a Carryover Ticket.
- **Sprint columns hold sprint names only, never sprint IDs or dates.**
- **Story points header depends on project type:** company-managed uses the "Story Points" field, team-managed uses "Story point estimate". They are different fields, so an export across both can carry both columns.
- **Resolved is formatted by a site-wide date/time pattern (not ISO 8601) in the exporting user's timezone.** In Cloud, reopening a ticket does not clear its resolution by default.
- **Cloud splits user fields into `Assignee` (display name) and `Assignee Id` (Atlassian account ID).** Only the current assignee is exported.

## 1. Column set: all fields vs current fields

| Fact | Platform | Source |
|---|---|---|
| Export menu offers "Export Excel CSV (all fields)", "Export Excel CSV (current fields)" and, from a filter, "Export Excel CSV (filter fields)". | Cloud | [Export search results](https://support.atlassian.com/jira-software-cloud/docs/export-search-results/) |
| "Current fields" = the fields visible in the List view; "filter fields" = the fields configured for the filter; "all fields" = every field. | Cloud | [Export search results](https://support.atlassian.com/jira-software-cloud/docs/export-search-results/) |
| All-fields exports are slow and produce large files; Atlassian advises selecting only needed fields. | Cloud | [Export over 10,000 work items](https://support.atlassian.com/jira/kb/export-over-10-000-work-items-in-jira-cloud/) |
| DC: Export > CSV (All fields) includes every issue field, comment and attachment; CSV (Current fields) only displayed fields. Header may contain repeated "Comment"/"Attachment" columns. | DC 10.0 | [Working with search results](https://confluence.atlassian.com/jirasoftwareserver100/working-with-search-results-1442844191.html) |
| A Story Points column can be missing from some export paths (Timeline export); workaround is exporting from the All issues list. | Cloud | [JRACLOUD-93445](https://jira.atlassian.com/browse/JRACLOUD-93445) |

Consequence: "current fields" is the export to ask users for, after they add Key, Status, Assignee, Resolved, Sprint and Story Points to the list view. Column order is therefore user-controlled; match by header name, not position.

## 2. Sprint column(s)

| Fact | Platform | Source |
|---|---|---|
| Multi-valued fields are exported as repeated columns with the same heading (shown for issue links; closed Won't Fix, so this is the intended layout). | Server/DC | [JRASERVER-62217](https://jira.atlassian.com/browse/JRASERVER-62217) |
| A ticket that moved through several sprints exports several columns all headed "Sprint". Atlassian Support (Petter Gonçalves) answered: the Sprint field returns all the sprints the ticket belonged to until its resolution. | Cloud (community thread, Atlassian Support answer) | [Exporting Data from Jira Cloud: sprint fields](https://community.atlassian.com/forums/Jira-questions/Exporting-Data-from-Jira-Cloud-What-are-all-the-sprint-fields/qaq-p/1174285) |
| The CSV export gives the Sprint by **name**; a request to add a Sprint ID column is unresolved ("Gathering Interest"). | Server (clone for Cloud: JRACLOUD-71757) | [JRASERVER-65781](https://jira.atlassian.com/browse/JRASERVER-65781) |
| Sprint names are not required to be unique; sprint IDs are. | Cloud | [Add a work item to a future sprint without losing closed sprints](https://support.atlassian.com/jira/kb/add-a-jira-work-item-to-a-future-sprint-without-losing-closed-sprints/) |
| Closed sprints are kept in the Sprint field when a ticket moves to a later sprint (the KB exists because bulk-editing would otherwise drop them). | Cloud | same KB as above |
| The **Excel** export puts several sprints in one cell, separated by `;`; order observed as inconsistent (possibly alphabetical). | Cloud (community only, no Atlassian answer) | [Confusion in exported data for Sprint values](https://community.atlassian.com/forums/Jira-questions/Confusion-in-exported-data-for-Sprint-values/qaq-p/3238225) |
| Sprint start/end/complete dates are **not** in the CSV; they are available from `GET /rest/agile/1.0/sprint/{sprintId}` (`startDate`, `endDate`, `completeDate`, `state`). | Cloud API | [Jira Software REST: Sprint](https://developer.atlassian.com/cloud/jira/software/rest/api-group-sprint/) |

Belief in the issue ("repeated `Sprint` columns, one per sprint") is **confirmed for CSV**, with the caveat that the Excel/XLSX path differs.

## 3. Story points field

| Fact | Source |
|---|---|
| Team-managed spaces use the **Story point estimate** field (once Estimation is enabled). | [Enable estimation](https://support.atlassian.com/jira-software-cloud/docs/enable-estimation/) |
| Company-managed configuration (incl. custom boards) offers only **Story Points**; Atlassian calls it "functionally incompatible" with team-managed **Story point estimate**. | [Story points not showing in custom board for a team-managed project](https://support.atlassian.com/jira/kb/story-points-not-showing-in-custom-board-for-a-team-managed-jira-project/) |
| Exporting "current fields" creates **duplicate columns** when a custom field with the same name exists in both company-managed and team-managed projects; one column holds values, the other is blank. | [JRACLOUD-77919](https://jira.atlassian.com/browse/JRACLOUD-77919) |
| Custom fields appear with a `Custom field (<name>)` header, e.g. `Custom field (Story Points)`. | Community reports only: [Story point not be exported](https://community.atlassian.com/t5/Jira-Cloud-Admins-questions/Story-point-not-be-exported/qaq-p/1753074) |
| In the DC data pipeline (a different export), story points are a plain number and nulls are empty values (`,,`). | [Data pipeline export schema](https://confluence.atlassian.com/adminjiraserver0901/data-pipeline-export-schema-1155492052.html) |

Exact header and empty-value rendering for the UI CSV are in [Uncertainties](#uncertainties).

## 4. Resolved date

| Fact | Platform | Source |
|---|---|---|
| Exported date/time values follow the site's **Complete Date/Time Format** (Settings > System > Look and Feel). Example pattern `dd/MMM/yy h:mm a` → `23/May/07 3:55 AM`. | Cloud | [CSV exported issues don't show AM/PM](https://support.atlassian.com/jira/kb/csv-exported-issues-dont-show-am-pm-in-date-time-field/) |
| A wrong pattern in Date/time formats corrupts exported dates. | Cloud | [Incorrect date/time format in exported issues](https://support.atlassian.com/jira/kb/incorrect-date-time-format-in-exported-issues/) |
| Patterns are Java `SimpleDateFormat`. | DC | [Configuring the look and feel (DC 11.2)](https://confluence.atlassian.com/adminjiraserver112/configuring-the-look-and-feel-of-your-jira-applications-1688896614.html) |
| Native CSV export renders times in the **exporting user's profile timezone**, not the site default; two users can export different values. | Cloud | [Discrepancy in date and time with CSV export](https://support.atlassian.com/jira/kb/jira-csv-export-discrepancy-in-the-date-and-time-when-exporting-issues-with-csv-export/) |
| Due Date and date-picker fields were shifted to UTC in CSV exports from DC 8.16 (fixed 8.20.4 / 8.22.0); date-time fields (like Resolved) unaffected. Shows exporters' Jira version can matter. | DC | [JRASERVER-72382](https://jira.atlassian.com/browse/JRASERVER-72382) |
| Any ticket with the Resolution field set is treated as resolved, and setting Resolution stamps the resolution date. | Server/DC | [Resolution date set on unresolved issues after CSV import](https://confluence.atlassian.com/jirakb/resolution-date-set-on-unresolved-issues-after-a-csv-import-604209816.html) |
| **Reopening a ticket in Cloud does not clear its resolution** unless the workflow has a "clear field" post function; such tickets still count as resolved. | Cloud | [Clear the resolution field when an issue is reopened](https://support.atlassian.com/jira/kb/clear-the-resolution-field-when-an-issue-is-reopened-in-jira-cloud/) |
| DC data pipeline gives `resolution_date` as ISO 8601 UTC (`2020-11-05T14:44:57Z`), unlike the UI CSV. | DC | [Data pipeline export schema](https://confluence.atlassian.com/adminjiraserver0901/data-pipeline-export-schema-1155492052.html) |

The CSV carries one Resolved value per ticket; no resolution history.

## 5. Assignee

| Fact | Platform | Source |
|---|---|---|
| Since 19 Oct (2020), each user field selected for export becomes two columns: the name and the ID (`Assignee`, `Assignee Id`). Answer by Atlassian Support (Daniel Eads). | Cloud | [Assignee & Reporter ID added as new column](https://community.atlassian.com/t5/Jira-Software-questions/How-why-has-my-download-csv-had-Assignee-amp-Reporter-ID-added/qaq-p/1521343) |
| The ID column is the Atlassian account ID (AAID); the importer cannot resolve display names, so it must be mapped instead of the name. | Cloud | [CSV import error: user doesn't exist](https://confluence.atlassian.com/jirakb/jira-cloud-csv-import-error-user-doesn-t-exist-1255444110.html) |
| Usernames/user keys were removed from Cloud APIs for privacy; account ID is the stable identifier. | Cloud | [User privacy API migration guide](https://developer.atlassian.com/cloud/jira/platform/deprecation-notice-user-privacy-api-migration-guide/) |
| Server/DC: a request to show full names instead of usernames was closed Won't Do; a later request to restore usernames is open, implying the name representation changed over versions. | Server/DC | [JRASERVER-62316](https://jira.atlassian.com/browse/JRASERVER-62316), [JRASERVER-69584](https://jira.atlassian.com/browse/JRASERVER-69584) |
| CSV has no assignee history; only the current assignee. (Also the premise of ADR 0001.) | both | implied by single `Assignee` column; history needs the changelog API (see §7) |

Matching across imports: in Cloud, key on `Assignee Id`; display names can change. In DC, there is no ID column documented; match on the exported name/username.

## 6. Limits and encoding

| Fact | Platform | Source |
|---|---|---|
| Asynchronous Export CSV from the issue navigator supports up to **10,000** work items; you get a notification with a download link. The old `pager/start` URL workaround (1,000 per batch) is deprecated since March 2026. | Cloud | [Export over 10,000 work items](https://support.atlassian.com/jira/kb/export-over-10-000-work-items-in-jira-cloud/) |
| Default export cap **1,000**, set by `jira.search.views.default.max` (Advanced Settings, no restart). `tempMax` URL parameter changes the count. | Server/DC | [Filter export only contains 1000 issues](https://support.atlassian.com/jira/kb/filter-export-only-contains-1000-issues-in-jira-server/), [Working with search results](https://confluence.atlassian.com/jirasoftwareserver100/working-with-search-results-1442844191.html) |
| Encoding is **UTF-8**; old Excel may misread it as ANSI. | Cloud | [Export issues from Jira Cloud in CSV format](https://support.atlassian.com/jira/kb/how-to-export-issues-from-jira-cloud-in-csv-format/) |
| Delimiter is chosen at export: comma, semicolon, vertical bar or caret (since 7.9). If an admin disables the dialog, comma is used. | Server/DC | [JRASERVER-62414](https://jira.atlassian.com/browse/JRASERVER-62414), [Working with search results](https://confluence.atlassian.com/jirasoftwareserver100/working-with-search-results-1442844191.html) |
| A comma in text fields "breaking" CSV was closed Not a Bug (blamed on Excel regional settings). | Server | [JRASERVER-62517](https://jira.atlassian.com/browse/JRASERVER-62517) |
| Atlassian's CSV convention (importer, data pipeline): quote fields with commas, quotes or newlines; double embedded quotes (RFC 4180). | DC/Cloud | [Data pipeline export schema](https://confluence.atlassian.com/adminjiraserver0901/data-pipeline-export-schema-1155492052.html), [Create work items using the CSV importer](https://support.atlassian.com/jira-software-cloud/docs/create-issues-using-the-csv-importer/) |
| Atlassian warns that opening exported CSV in spreadsheet apps can execute macros. | Cloud | [Export search results](https://support.atlassian.com/jira-software-cloud/docs/export-search-results/) |

## 7. API alternative (brief)

- **Sprints with dates:** `GET /rest/agile/1.0/sprint/{sprintId}` returns `name`, `state`, `startDate`, `endDate`, `completeDate`, `originBoardId`; OAuth scope `read:sprint:jira-software`. [Sprint API](https://developer.atlassian.com/cloud/jira/software/rest/api-group-sprint/)
- **Ticket search:** `/rest/api/3/search` is deprecated and being removed; use `/rest/api/3/search/jql`. [Issue search API](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-search/), [Platform changelog](https://developer.atlassian.com/cloud/jira/platform/changelog/)
- **Resolution date and assignee-at-close:** `resolutiondate` is a standard issue field; assignee history needs the issue changelog (Issues API group). [Issues API](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/) (endpoint path not verified, see Uncertainties).
- **Rate limits (Cloud):** points-based hourly quota (default global pool 65,000 points/hour), per-endpoint burst limits (GET 100/s default), HTTP 429 with `Retry-After` and `RateLimit-Reason`. [Rate limiting](https://developer.atlassian.com/cloud/jira/platform/rate-limiting/)
- **DC alternative:** the data pipeline export gives ISO UTC dates, sprint objects with start/end dates as JSON, and numeric assignee IDs. [Data pipeline export schema](https://confluence.atlassian.com/adminjiraserver0901/data-pipeline-export-schema-1155492052.html)

## Sample file

`samples/jira-export.csv` is synthetic (fake names, fake account IDs, project key `FAKE`). It models a Cloud "current fields" export with comma delimiter and the documented layout above; the story points header and date pattern are the assumed defaults from Uncertainties 1 and 5.

| Key | Case it covers |
|---|---|
| FAKE-103, FAKE-106 | Carryover Ticket: two `Sprint` columns |
| FAKE-104 | Unestimated Ticket: closed, empty story points |
| FAKE-105, FAKE-109 | Unassigned: empty `Assignee` and `Assignee Id` |
| FAKE-106 | Reopened then reclosed: only the latest Resolved shows; the CSV cannot show the reopen. Summary has an embedded newline. |
| FAKE-107 | Currently reopened (status `Reopened`) but Resolution/Resolved still set, as Cloud does by default |
| FAKE-102 | Quoted field with comma and doubled quotes |
| FAKE-110 | Resolved with no sprint (candidate Unplaced Ticket) |

## Uncertainties

Not confirmed by an Atlassian primary source; verify against one real export before the importer is final.

1. **Exact story points header.** `Custom field (Story Points)` / `Custom field (Story point estimate)` comes from community reports only. Sites can rename fields, so the header is site-specific.
2. **Empty estimate rendering in UI CSV.** Assumed empty cell (as in the data pipeline). Not documented for the UI export; `0` and empty must be treated as different.
3. **Order of repeated `Sprint` columns.** Undocumented. One community post saw inconsistent order in the Excel cell. Do not infer "latest sprint" from position.
4. **Which sprints stay on a ticket.** Closed sprints are kept; whether a sprint is dropped when a ticket is removed from a still-active sprint is not documented.
5. **Default Complete Date/Time Format value and locale.** Only an example pattern (`dd/MMM/yy h:mm a`) is documented; month abbreviations may follow the user's language. No timezone offset is written in the value.
6. **Resolved after reopen-then-reclose.** Documented that Resolution is set/kept; not documented in so many words that the Resolved date is overwritten at each re-resolve (expected, as the field is single-valued).
7. **Status Category column.** No source found confirming it can be added to a CSV export.
8. **DC assignee format.** Whether DC exports display name, username or both is not stated in current docs.
9. **Cloud delimiter and quoting.** Cloud docs do not state the delimiter choice or quoting rules for the UI export; comma + RFC 4180 is assumed.
10. **Changelog endpoint path** (`GET /rest/api/3/issue/{issueIdOrKey}/changelog`) not verified in this session.

## Implications for the importer

1. **Match columns by header name, collect all repeats.** Read every column named `Sprint` into a list per ticket; ignore empty ones. Same pattern for any duplicated story-points column (take the non-empty one).
2. **Configurable field mapping with sensible defaults.** Default story points headers: `Custom field (Story Points)`, `Custom field (Story point estimate)`, `Story Points`, `Story point estimate`. Fail loudly if none is found rather than treating all tickets as Unestimated.
3. **Dates: configurable pattern + timezone at import.** Default `dd/MMM/yy h:mm a`; ask for (or store per team) the exporter's timezone. Reject the import if any Resolved value fails to parse; never guess.
4. **Do not trust Resolved alone for "closed".** Require Resolved set *and* status in a configured done-status list; a reopened Cloud ticket can keep its old Resolved date (flag it).
5. **Key people on `Assignee Id` when present** (Cloud), fall back to the `Assignee` name (DC); map to Team Members by ID. Empty assignee → team-only velocity, per GLOSSARY.
6. **Sprint names are not unique.** Scope sprint matching per team/board; sprint dates must come from scrumify's own Sprint records (or the API later), never the CSV.
7. **Parse as RFC 4180 UTF-8 (strip BOM), allow delimiter choice** (`,` `;` `|` `^`), and cap at 10,000 rows per file (Cloud limit) with a clear message to split by JQL.
