# Meeting to actions, on n8n

A meeting transcript goes in. Decisions and action items come out, the actions land in a sheet, and the recap goes to the team only after a human has approved it.

I run a version of this workflow in production for a leadership team, built on Power Automate and SharePoint. This is a clean rebuild on n8n with synthetic data, so the logic is visible and portable: one workflow file, your own credentials, runs on a laptop.

```
Form (transcript) -> Gemini (structured JSON) -> Parse -> Split actions -> Google Sheet
                                                   \-> Gmail: approve? -> yes: recap to team
                                                                          no: note to owner
```

## What it does

1. **Intake.** An n8n form takes the meeting title, date and the pasted transcript.
2. **Extraction.** Gemini returns strict JSON: a three-sentence summary, the decisions, and action items with owner, task and due date. No owner, no action.
3. **Parsing.** A Code node validates the JSON and fails loudly if the model drifts, instead of writing garbage downstream.
4. **Logging.** One row per action item appended to a Google Sheet, status `open`.
5. **Human in the loop.** The recap is emailed to the meeting owner with Approve and Decline buttons. Nothing reaches the team until someone clicks Approve.
6. **Recap.** On approval the HTML recap goes to the team list. On decline the owner gets a note and the sheet keeps the actions for editing.

## Run it

**Locally, free.** Needs Node 18 or later.

```
npx n8n
```

Open http://localhost:5678, create the local account, then Workflows, Import from file, `workflows/meeting-to-actions.json`.

**On n8n Cloud.** Same import. The Google credentials are one click there, which is why the cloud trial is the quicker first run.

## Credentials

| Node | Credential | Where to get it | Cost |
|---|---|---|---|
| Extract decisions and actions | Header Auth, name `x-goog-api-key`, value your key | Google AI Studio, "Get API key" | Free tier |
| Log actions to sheet | Google Sheets OAuth2 | Sign in with Google inside n8n | Free |
| Human approval, Send recap, Tell the owner | Gmail OAuth2 | Sign in with Google inside n8n | Free |

Then open each node and replace the placeholders: `YOUR_GOOGLE_SHEET_ID`, `YOUR_EMAIL@example.com`, `TEAM_LIST@example.com`.

The sheet needs a tab called `Actions` with the header row: `meeting, date, owner, task, due, status`.

## Try it

`fixtures/sample-transcript.md` is a made-up leadership meeting with four action items. Paste it into the form, approve the email, and check the sheet and the recap.

## Design notes

[docs/case-study.md](docs/case-study.md) covers the production version this is based on: what broke in the first year, what the fixes were, and why the rebuild is on n8n.

## What I would add next

- A monthly job that creates the next eight meeting rows so the series never runs out.
- Carry-over: open actions from the last meeting pre-filled into the next agenda.
- An error workflow that writes failures to the sheet and emails the owner, so a failed run is visible instead of silent.
