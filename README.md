# Session Feedback Form

A token-gated, per-participant feedback form for DataRobot customer sessions. Hosted on GitHub Pages. No login required for participants — works in any browser, any mail client.

## How it works

Each participant gets a unique link (e.g. from an email invite). They open it, fill in the form, and submit. Responses are saved to a Google Sheet. Participants can return to their link and update their response until the organiser closes the form.

```
Email invite → unique link → form → Google Apps Script → Google Sheet
```

## Live form

**URL:** https://olga-shpyrko-dr.github.io/session-feedback/

A token is required to access the form:
```
https://olga-shpyrko-dr.github.io/session-feedback/?token=<UUID>
```

## Files

| File | Purpose |
|---|---|
| `index.html` | The feedback form — self-contained, no dependencies |
| `README.md` | This file |

## Reusing for a new session (copying the Sheet)

When you copy the Google Sheet to reuse for a new session, the Apps Script code is copied but **the deployment is not** — you must create a fresh one each time:

1. Open the new sheet → **Extensions → Apps Script**
2. Update `SHEET_ID` to the new sheet's ID (from the URL between `/d/` and `/edit`)
3. **Deploy → New deployment → Web app**
   - Execute as: **Me** / Who has access: **Anyone**
4. Copy the new deployment URL
5. Update `CONFIG.appsScriptUrl` in `index.html` and re-upload to GitHub

---

## Configuring for a new session

Open `index.html` and edit the `CONFIG` block (~line 564):

```javascript
const CONFIG = {
  appsScriptUrl: 'https://script.google.com/macros/s/.../exec',
  sessionName:   'Session name here',   // must match the Sheet tab name exactly
  sessionDateTime: 'May 7, 2026',
  topics: [
    'Topic 1',
    'Topic 2',
    // ...
  ]
};
```

Re-upload `index.html` to this repo after editing. Changes go live within ~60 seconds.

## Sending invites

Use the **Email Sender** spreadsheet in DataRobot Drive:
1. Add participant names and emails
2. Open **📧 Feedback → Send feedback emails**
3. Tokens are generated automatically and personalised emails are sent

### Before each new session — update the Email Sender

The Email Sender spreadsheet and its `Sender.gs` script are **session-specific**. Before sending invites for a new session, update the following in `Sender.gs` (Extensions → Apps Script):

```javascript
const FEEDBACK_TAB_NAME = 'Your new session tab name';  // must match the Sheet tab exactly
const SESSION_DISPLAY   = 'Your session title';          // shown in email subject + body
const SESSION_DATE      = 'June 1, 2026';                // shown in email subject + body
const CLOSE_DATE        = 'June 8, 2026';                // deadline shown in email body
```

Then save and **clear the participant rows** (or use a fresh sheet) so tokens from the previous session are not reused. The `sent` column protects against double-sending within a session but does not reset between sessions.

## Closing the form

In `Code.gs` (Apps Script attached to the feedback Sheet):
1. Change `const FORM_OPEN = true` → `false`
2. Save
3. **Deploy → Manage deployments → New version → Deploy**

## Backend

- **Apps Script:** handles token validation and response storage via JSONP (required to bypass Google Workspace CORS policy)
- **Google Sheet:** `Session Feedback Responses` in DataRobot Drive — one tab per session; includes a `History` tab with full submission audit trail
- **Email Sender:** `Session Feedback — Email Sender` in DataRobot Drive

Full architecture and setup details: see `session-feedback-spec.md` in DataRobot Drive.

---

## ⚠️ Known issues

### Submission notification emails — not working
`Code.gs` is configured to send an email to `olga.shpyrko@datarobot.com` on each form submission. This feature is **not yet working as expected** — the notification email is not being delivered despite `MailApp` permissions being granted. Investigation is paused. The form itself, response storage, and History tab all work correctly.

To debug when ready: add `debugLog()` calls to `Code.gs` (see spec) to write diagnostic output to a `DebugLog` tab in the feedback Sheet, then re-deploy and submit a test response.
