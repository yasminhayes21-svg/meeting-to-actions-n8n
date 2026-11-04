# Case study: a meeting workflow that ran for a year, and what broke

The production version of this workflow runs for a leadership team on Power Automate and SharePoint. It takes a weekly transcript, produces the agenda, the recap and the carry-over of open items, and keeps a ledger of what has run. These notes are from a review I wrote after a year of running it. Names, dates and system details are removed.

## The nine things that broke

1. **The ledger was claimed before the work could fail.** One failed run marked the meeting as done and locked it forever. Fix: read the ledger first, treat a `started` older than two hours as stale, and wrap every flow in try, catch and finally so a failure writes `failed` and emails the owner.
2. **A silent rejection path.** A typo in the attendee roster rejected every submission for six days and nobody was told. Fix: the roster is data in a list, not a string in an expression, and every rejection sends a message.
3. **The page date was maintained by string replace inside canvas JSON.** Fix: a yes/no column the flow sets, and a view that filters on it.
4. **Items vanished with no audit trail.** Fix: list versioning on, members get contribute without delete.
5. **The meeting rows ended on a fixed date and nothing created more.** Fix: a monthly generator that creates eight weeks ahead.
6. **Content left the tenant through an email relay** that was a workaround for a block that no longer existed. Fix: route through the native transcript integration now that it is allowed.
7. **Updates resent every field,** so a null overwrote good data. Fix: patch only the fields that change.
8. **Run history exposed item text** because secure inputs and outputs were off. Fix: turn them on for every content action.
9. **Everything ran on one person's account.** Fix: a service account owns the connections and the site.

## The one change behind all of them

The flows existed only as clicks in a designer, with site URLs, list IDs and connection IDs baked in. Nothing was versioned, nothing was configurable, and the only record of what the flows did was my own notes.

Treat the flows as code: export the definitions into a repo, pull configuration (site, list names, offsets, approver, roster) into a config list the flows read on start, and keep one set of flow definitions that serves every environment. That is what makes a second team, or a second company, a config file instead of a rebuild.

## Why the rebuild is on n8n

Workflows are JSON files in git by nature. Credentials are separate from logic. There is an execution log, an error workflow, and a Docker image or `npx n8n` that runs on a laptop. The whole cycle can be shown in an afternoon, and the LLM step is a node, not a licensed add-on.

For a production instance where content must stay inside a company tenant, n8n needs to be self-hosted inside that estate. The design question is the same either way: is the content allowed to leave, and if not, who hosts the runner.
