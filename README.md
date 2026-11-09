# Operations workflows on n8n

Four n8n workflows for the work an operations or enablement team does every week: turning meetings into actions, triaging automation requests, scoring leads, and reporting on email campaigns. Each one runs on a laptop with synthetic data and at most one free API key.

They come from my current role: a meeting workflow that runs in production on Power Automate and SharePoint, weekly campaign reporting, a lead scoring rollout on the CRM, and intake for automation requests. These are clean rebuilds on n8n with synthetic data, so the logic is visible and portable.

| Workflow | File | What it shows | Credential |
|---|---|---|---|
| [Meeting to actions](#meeting-to-actions) | `workflows/meeting-to-actions.json` | LLM extraction, a person approves before anything is saved | Gemini key |
| [Automation request triage](#automation-request-triage) | `workflows/automation-requests.json` | Intake form, time cost estimate, LLM classification, priority queue | Gemini key |
| [Lead scoring with a backtest](#lead-scoring-with-a-backtest) | `workflows/lead-scoring.json` | Explainable rules, checked against past outcomes before going live | None |
| [Campaign results report](#campaign-results-report) | `workflows/campaign-report.json` | Numbers computed in code, LLM commentary checked against them | Gemini key |

The same patterns run through all four:

- **The model never owns a number or a decision.** Code computes, a rule or a person decides, the model drafts text.
- **Fail loudly or fall back, never silently.** Model output is validated. If Gemini is down, the request is still logged and the report still goes out, marked as such.
- **Every run leaves a record** in an n8n data table, created on the first run.

## Run it

**Locally, free.** Needs Node 20 or 22. Tested on n8n 2.35.7.

```
npx n8n
```

Open http://localhost:5678, create the local account, then Workflows, Import from file, and pick a file from `workflows/`.

**On n8n Cloud.** Same import.

**Gemini key.** Three workflows call Gemini (`gemini-flash-lite-latest`, free tier) through an HTTP Request node.

| Credential | Where to get it | Cost |
|---|---|---|
| Header Auth, name `x-goog-api-key`, value your key | Google AI Studio, "Get API key" | Free tier |

Open the Gemini node, create the Header Auth credential once, and pick it in the other workflows. Each call times out after 60 seconds and retries three times, because a busy model can hang instead of failing.

**Trying a workflow.** Click **Execute workflow** and the form opens in a new tab. To share a fixed form link instead, activate the workflow; the form path is listed under each workflow below. Saved rows are under **Overview, Data tables**.

## Meeting to actions

A meeting transcript goes in. Decisions and action items come out, a person reviews them, and only approved actions are saved.

```
Form (transcript) -> Gemini (structured JSON) -> Parse -> Review page: approve?
                                                            yes: create table if missing -> one row per action -> recap page
                                                            no:  "nothing was saved" page
```

1. **Intake.** A form takes the meeting title, date and the pasted transcript.
2. **Extraction.** Gemini returns strict JSON: a three-sentence summary, the decisions, and action items with owner, task and due date. No owner, no action.
3. **Parsing.** A Code node validates the JSON and fails loudly if the model drifts, instead of writing garbage downstream. Model output is escaped before it is shown on a page.
4. **Human in the loop.** The next page shows the summary, decisions and actions, and asks the person to approve or decline. Nothing is saved until someone approves.
5. **Logging.** On approval, one row per action goes into the `meeting_actions` data table, status `open`.

**Try it:** paste `fixtures/sample-transcript.md`, a made-up leadership meeting, and approve on the review page. Form path: `/form/meeting-to-actions`.

[docs/case-study.md](docs/case-study.md) covers the production version: what broke in the first year, what the fixes were, and why the rebuild is on n8n.

## Automation request triage

Anyone in the company describes a task they do by hand. The workflow estimates what it costs, judges how well it suits automation, and puts it in a priority queue.

```
Form -> hours per month (code) -> Gemini: category, fit, approach, risk -> score and priority (code) -> automation_requests table -> confirmation page
```

1. **Intake.** Name, team, what they do by hand, tools, how often, minutes each time, how many people.
2. **Cost.** Code turns frequency, minutes and people into hours per month.
3. **Classification.** Gemini returns the category, the automation fit (high, medium, low), a suggested approach, the main risk, and the one question that would most change the estimate.
4. **Priority.** Score = hours per month weighted by fit (high 1, medium 0.6, low 0.25). P1 from 10, P2 from 3, P3 below. The model judges fit; the priority is arithmetic anyone can check.
5. **Queue.** The request goes into `automation_requests` with status `new`, and the requester sees the ID, priority and the question back to them.

If Gemini does not answer, the request is still logged with fit `unknown` and category "needs manual triage".

**Try it:** "Every Monday I export last week's support tickets to a spreadsheet, tag each by product area, count them and paste the counts into a slide", weekly, 90 minutes, 2 people. That comes out at 13 hours a month, fit high, P1. Form path: `/form/automation-request`.

## Lead scoring with a backtest

A CSV of leads goes in. Every lead gets a score, a tier and the reasons. If the file says which leads converted, the workflow checks whether the tiers would have predicted it, before anyone trusts the score.

```
Form (CSV upload) -> read CSV -> scoring rules (edit me) -> score each lead -> backtest -> lead_scores table -> results page
```

1. **Rules in one place.** The `Scoring rules (edit me)` node holds every weight: company size bands, target industries and markets, buyer seniority, demo requests, site visits and email clicks, and the tier cut-offs.
2. **Explainable scores.** Each lead's score comes with its reasons, e.g. `200+ employees +25; Logistics +15; requested a demo +20`. A sales rep can see why a lead is tier A.
3. **Backtest.** For leads with a known outcome, the conversion rate per tier is compared with the average. The rules pass when the tiers are in order and tier A converts at least twice as often as the average lead.
4. **Shadow run.** Scores are saved to `lead_scores` with the run ID and mode `shadow`. Nothing is written to a CRM. Writing back is a separate step, taken after a shadow run has been checked against what sales actually did.

No credential needed: the scoring is rules, not a model, so it is cheap, fast and auditable.

**Try it:** upload `fixtures/leads-history.csv`, 80 made-up leads with outcomes. Tier A converts at 60% against an average of 27.5%, so the rules pass. Then set `tiers.A` to 60 in the rules node and run it again: tier A grows, its lift drops to 1.6, and the backtest says to adjust the rules before going live. Form path: `/form/lead-scoring`.

## Campaign results report

An email campaign export goes in. A report with rates, flags and a written commentary comes out, with every number in the commentary checked against the data.

```
Form (title + CSV) -> read CSV -> compute the metrics (code) -> Gemini commentary -> check the numbers -> campaign_metrics table -> report page
```

1. **Metrics in code.** Open, click, click-to-open, unsubscribe and bounce rates per campaign and in total, plus flags: bounces above 2%, unsubscribes above 0.5%, opens below 20%.
2. **Commentary.** Gemini gets only the computed numbers and writes a headline and three points: what worked, what did not, what to change.
3. **Number check.** Every number in the commentary is matched against the data. Anything that does not match is shown in red as "check before sending". If Gemini is down, the report goes out without commentary.
4. **History.** One row per campaign in `campaign_metrics`. Uploading the same export again updates the rows instead of duplicating them.

The CSV columns follow a typical email platform export: `campaign, send_date, audience, emails_sent, hard_bounces, unique_opens, unique_clicks, unsubscribes`.

**Try it:** title "November email results", upload `fixtures/campaign-export.csv`, eight made-up campaigns. Form path: `/form/campaign-report`.

## What I would add next

- An error workflow that records failures in a data table and notifies the owner, so a failed run is visible instead of silent.
- Meeting carry-over: open actions from the last meeting pre-filled into the next agenda.
- Lead scoring write-back to the CRM once a shadow run is signed off, with the previous score kept for comparison.
- A scheduled campaign report that pulls the export from the email platform's API instead of a file upload.
- Email delivery of the approved recaps and reports, as an optional branch for teams that connect Gmail.
