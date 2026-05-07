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

## Closing the form

In `Code.gs` (Apps Script attached to the feedback Sheet):
1. Change `const FORM_OPEN = true` → `false`
2. Save
3. **Deploy → Manage deployments → New version → Deploy**

## Backend

- **Apps Script:** handles token validation and response storage via JSONP (required to bypass Google Workspace CORS policy)
- **Google Sheet:** `Session Feedback Responses` in DataRobot Drive — one tab per session
- **Email Sender:** `Session Feedback — Email Sender` in DataRobot Drive

Full architecture and setup details: see `session-feedback-spec.md` in DataRobot Drive.
