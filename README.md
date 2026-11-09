# Meeting to actions, on n8n

A meeting transcript goes in. Decisions and action items come out, a person reviews them, and only approved actions are saved.

I run a version of this workflow in production for a leadership team, built on Power Automate and SharePoint. This is a clean rebuild on n8n with synthetic data, so the logic is visible and portable: one workflow file, one free API key, runs on a laptop.

```
Form (transcript) -> Gemini (structured JSON) -> Parse -> Review page: approve?
                                                            yes: create table if missing -> one row per action -> recap page
                                                            no:  "nothing was saved" page
```

## What it does

1. **Intake.** An n8n form takes the meeting title, date and the pasted transcript.
2. **Extraction.** Gemini returns strict JSON: a three-sentence summary, the decisions, and action items with owner, task and due date. No owner, no action.
3. **Parsing.** A Code node validates the JSON and fails loudly if the model drifts, instead of writing garbage downstream. Model output is escaped before it is shown on a page.
4. **Human in the loop.** The next form page shows the summary, decisions and action items, and asks the person to approve or decline. Nothing is saved until someone approves.
5. **Logging.** On approval, one row per action item goes into the n8n data table `meeting_actions`, status `open`. The workflow creates the table on its first run.
6. **Recap.** The last page shows the recap and how many actions were saved. On decline, nothing is saved.

## Run it

**Locally, free.** Needs Node 20 or 22. Tested on n8n 2.35.7.

```
npx n8n
```

Open http://localhost:5678, create the local account, then Workflows, Import from file, `workflows/meeting-to-actions.json`.

**On n8n Cloud.** Same import.

## Credentials

One credential, no Google sign-in needed.

| Node | Credential | Where to get it | Cost |
|---|---|---|---|
| Extract decisions and actions (Gemini) | Header Auth, name `x-goog-api-key`, value your key | Google AI Studio, "Get API key" | Free tier |

Open the Gemini node, create the Header Auth credential, and save the workflow.

## Try it

1. Click **Execute workflow**. The form opens in a new tab.
2. Paste `fixtures/sample-transcript.md`, a made-up leadership meeting with four action items.
3. Approve on the review page.
4. Check the rows under **Overview, Data tables, meeting_actions**.

To share a fixed form link instead, activate the workflow. The form is then at `/form/meeting-to-actions`.

## Design notes

[docs/case-study.md](docs/case-study.md) covers the production version this is based on: what broke in the first year, what the fixes were, and why the rebuild is on n8n.

## What I would add next

- A monthly job that creates the next eight meeting rows so the series never runs out.
- Carry-over: open actions from the last meeting pre-filled into the next agenda.
- An error workflow that records failures in a data table and notifies the owner, so a failed run is visible instead of silent.
- Email delivery of the approved recap, as an optional branch for teams that connect Gmail.
