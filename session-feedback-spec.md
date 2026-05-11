# Session feedback form — architecture spec

## Overview

A token-gated, per-participant feedback form hosted as a static HTML page. Each participant receives a unique URL. No login required. Responses are stored in Google Sheets via a Google Apps Script Web App. Email invites with personalised links are sent automatically via a second Apps Script attached to a sender spreadsheet.

---

## Why not Google Forms / Google Drive directly

| Approach | Problem |
|---|---|
| Google Forms (shared link) | All responses in one sheet; no per-participant isolation; "anyone with link can edit" means no access control |
| Google Doc/Form per participant | Requires recipient to be logged in with a Google account that has been explicitly granted access — breaks for external participants and all non-Google mail clients |
| Google Drive MCP | Cannot create `application/vnd.google-apps.form` — Forms API is separate and not available in the MCP connector |

**Root constraint:** Google Workspace shared links either require authenticated access (breaks external users) or are fully public (no isolation). Neither fits.

---

## Solution: token-URL form → Apps Script → Google Sheets

```
Organizer runs email sender script
        ↓
Tokens generated + written to feedback Sheet
        ↓
Each participant receives personalised email with unique link
        ↓
Participant opens link → form loads (token validated via JSONP)
        ↓
Participant submits → Apps Script upserts row in Sheet
        ↓
Organizer closes poll by setting FORM_OPEN = false and re-deploying
```

### Token URL format
```
https://olga-shpyrko-dr.github.io/session-feedback/?token=<UUID>
```
- `token` — UUID v4, unique per participant, pre-written to the Sheet by the sender script
- `session` — optional URL param; falls back to `CONFIG.sessionName` in the HTML if omitted

---

## Form fields

| Field | Type | Notes |
|---|---|---|
| Feedback timestamp | Auto | `new Date().toISOString()` on submit |
| Session name | Pre-populated | From `CONFIG.sessionName` or `?session_name=` URL param |
| Session date & time | Pre-populated | From `CONFIG.sessionDateTime` or `?session_date=` URL param |
| Q1: Overall value (1–5) | Score buttons | Required |
| Q2: Relevant topics | Multi-select checkboxes + free text "Other" | Configurable via `CONFIG.topics` array |
| Q3: Improvements | Textarea | Optional |
| Q4: Follow-up topics | Textarea | Optional |
| Name / email | Text input | Optional |
| Role | Text input | Optional |

---

## Components

### 1. Static HTML form (`index.html`)
- Zero dependencies (vanilla JS + CSS), DataRobot branded
- DR logo embedded as base64 — no external image hosting needed
- Reads `?token` from URL on load
- Uses **JSONP** (not `fetch`) to call Apps Script — required to bypass Google Workspace CORS policy that blocks `fetch` requests to `script.google.com` from external origins
- On load: JSONP GET → pre-fills any prior response, checks `open` flag
- On submit: JSONP GET with `?action=submit&payload=...` → upserts row in Sheet
- Displays "form closed" state if Apps Script returns `{ open: false }`
- Displays "invalid link" state if token is not found in Sheet
- QR-code compatible (any URL works)

### 2. Google Apps Script Web App (`Code.gs`)
Deployed as: **Execute as me / Anyone can access**

All communication goes through `doGet` — POST is not used due to CORS redirect issues:

```
doGet (no action param)  → validates token, returns { open, response }
doGet (action=submit)    → upserts response row, returns { ok: true }
```

Both endpoints wrap their response in a JSONP callback when `?callback=` is present, returning `ContentService.MimeType.JAVASCRIPT` instead of JSON.

Token comparison uses `String()` cast on both sides to avoid number/text mismatch (Google Sheets auto-converts numeric-looking tokens to numbers).

**Additional features in current version:**
- **History tab** — every submission is appended to a `History` tab (auto-created if missing), preserving a full audit trail. The main session tab always shows the latest response per participant.
- **Submission notifications** — each successful submit sends a plain-text email to `NOTIFY_EMAIL` via `MailApp` (not `GmailApp` — required for web app context).
- **Auto-close by date** — `CLOSE_DATE` constant (ISO format `YYYY-MM-DD`). Once the date passes, `isFormOpen()` returns `false` automatically. The first request after close date triggers a reminder email to `NOTIFY_EMAIL` to formally set `FORM_OPEN = false`.

> **Note on `MailApp` vs `GmailApp`:** `MailApp` must be used for sending from `doGet` web app handlers. `GmailApp` silently fails in this context. If notifications stop working after re-deploying, run the `testEmail()` function manually to re-authorize `MailApp` permissions.

Main session tab schema (one row per token):

| token | session | name | email | role | q1_score | q2_topics | q3_improvements | q4_followup | submitted_at | updated_at |
|---|---|---|---|---|---|---|---|---|---|---|

History tab schema (one row per submission):

| submitted_at | token | session | name | email | role | q1_score | q2_topics | q3_improvements | q4_followup |
|---|---|---|---|---|---|---|---|---|---|

### 3. Google Sheet — Feedback Responses (DataRobot Drive)
- One tab per session, tab name must match `CONFIG.sessionName` exactly
- Tokens written by the sender script (with `'` prefix to force text storage)
- Organiser closes form by setting `FORM_OPEN = false` in `Code.gs` and re-deploying
- **History tab** — auto-created by the script on first submission; contains one row per submission for full audit trail

> ⚠️ **Never delete token rows from the session tab.** The script looks up the submitted token in this tab to validate and save the response. If the row is deleted, the participant's link returns "invalid" and their submission fails. To recover: re-add the token from the Email Sender sheet (column D) back into column A with a `'` prefix. To clear data between sessions, either use a new tab or clear only the data columns (B–K), leaving column A (token) intact.

### 4. Google Apps Script — Email Sender (`Sender.gs`)
Attached to a separate **Email Sender spreadsheet** in DataRobot Drive. One-click operation via a custom **📧 Feedback** menu:
1. Iterates participant rows (name + email)
2. Generates a UUID v4 token per row
3. Appends the token to the feedback Sheet
4. Sends a branded HTML email with the personalised link
5. Writes token, link, and sent timestamp back to the sender sheet
6. Skips rows already marked as sent — safe to re-run

---

## Security model

| Threat | Mitigation |
|---|---|
| Participant A reads B's response | Tokens are UUID v4 (unguessable); GET only returns the row matching the submitted token |
| Someone enumerates responses | No list endpoint; token must be known to retrieve a row |
| Replay / double submit | Apps Script upserts by token — idempotent, last write wins |
| Form stays open indefinitely | `FORM_OPEN` flag in script; organiser sets to `false` to close |
| Token leaked from email | Responses are low-sensitivity feedback; acceptable risk |

**Note:** Security-by-obscurity on the token. Appropriate for session feedback. Do not use for sensitive PII or regulated data.

---

## Deployment steps

### Step 1 — Create the feedback Google Sheet
1. In DataRobot Drive, create a new Google Sheet: `Session Feedback Responses`
2. Add a tab named exactly as your `CONFIG.sessionName` (e.g. `DataRobot GenAI and Agentic offering`)
3. Click cell A1, add headers by pressing Tab between each:
   `token | session | name | email | role | q1_score | q2_topics | q3_improvements | q4_followup | submitted_at | updated_at`
4. Bold row 1 (`Cmd+B`) to distinguish headers from data
5. The Sheet ID is in the URL: `https://docs.google.com/spreadsheets/d/SHEET_ID/edit`

### Step 2 — Deploy `Code.gs` as a Web App
1. In the Sheet: **Extensions → Apps Script**
2. Paste the full `Code.gs` (see below)
3. Set `SHEET_ID` to your Sheet ID
4. Confirm `FORM_OPEN = true`
5. **Deploy → New deployment → Web app**
   - Execute as: **Me**
   - Who has access: **Anyone**
6. Copy the deployment URL — paste into `CONFIG.appsScriptUrl` in `index.html`
7. **Re-deploy as new version after every change to `Code.gs`**

### Step 3 — Host `index.html` on GitHub Pages
1. Go to **https://github.com/organizations/datarobot/repositories/new**
   - Name: `session-feedback` (or your chosen repo name)
   - Visibility: Public (Pages on private repos requires paid GitHub plan)
2. Rename `feedback.html` → `index.html` before uploading
3. **Add file → Upload files** → drag `index.html` → commit to `main`
4. **Settings → Pages** → Source: `main` branch, `/ (root)` → Save
5. Live at: `https://<org>.github.io/<repo>/` within ~60 seconds

To update the form: re-upload `index.html` via **Add file → Upload files** (overwrites existing file). Hard refresh (`Cmd+Shift+R`) after deploy to bypass cache.

### Step 4 — Set up the email sender
1. Open **Session Feedback — Email Sender** sheet in DataRobot Drive
2. **Extensions → Apps Script** → paste `Sender.gs`
3. Update constants at the top: `SESSION_DISPLAY`, `SESSION_DATE`, `CLOSE_DATE`
4. Run `onOpen` once manually to register the menu and grant permissions
5. Fill in participant `name` and `email` columns
6. Use **📧 Feedback → Send feedback emails** to send

### Step 5 — QR code for slides
Generate at https://qrcode.show using the base form URL (without token — the slide QR is a visual prompt; the actual submission link comes via email). Use error-correction level Q or H for slides.

### Step 6 — Close the form
In `Code.gs`: change `FORM_OPEN = true` → `false`, save, re-deploy as new version.

---

## `Code.gs` — full script

The current version of `Code.gs` is maintained as a separate file (`Code.gs`) downloadable from this conversation. It includes History tab, submission notifications, and auto-close features.

Key constants at the top:

```javascript
const SHEET_ID        = 'YOUR_SHEET_ID_HERE';
const FORM_OPEN       = true;
const CLOSE_DATE      = '2026-05-14';          // ISO YYYY-MM-DD — auto-closes after this date
const NOTIFY_EMAIL    = 'olga.shpyrko@datarobot.com';
const SESSION_DISPLAY = 'DataRobot GenAI and Agentic offering';
const HISTORY_TAB     = 'History';

// Legacy minimal version (no notifications, no history):
const SHEET_ID  = 'YOUR_SHEET_ID_HERE';
const FORM_OPEN = true;

function doGet(e) {
  const callback = e.parameter.callback;

  function respond(obj) {
    const json = JSON.stringify(obj);
    if (callback) {
      return ContentService
        .createTextOutput(callback + '(' + json + ')')
        .setMimeType(ContentService.MimeType.JAVASCRIPT);
    }
    return ContentService
      .createTextOutput(json)
      .setMimeType(ContentService.MimeType.JSON);
  }

  // Submit action
  if (e.parameter.action === 'submit') {
    if (!FORM_OPEN) return respond({ error: 'form closed' });
    const data = JSON.parse(e.parameter.payload);
    const { token, session } = data;
    if (!token || !session) return respond({ error: 'missing params' });
    const sheet = SpreadsheetApp.openById(SHEET_ID).getSheetByName(session);
    if (!sheet) return respond({ error: 'session not found' });
    const rows = sheet.getDataRange().getValues();
    const headers = rows[0];
    const tokenCol = headers.indexOf('token');
    const now = new Date().toISOString();
    for (let i = 1; i < rows.length; i++) {
      if (String(rows[i][tokenCol]) === String(token)) {
        headers.forEach((h, idx) => {
          if (data[h] !== undefined) sheet.getRange(i + 1, idx + 1).setValue(data[h]);
        });
        sheet.getRange(i + 1, headers.indexOf('updated_at') + 1).setValue(now);
        return respond({ ok: true });
      }
    }
    return respond({ error: 'token not found' });
  }

  // Load existing response
  const token = e.parameter.token;
  const session = e.parameter.session;
  if (!token || !session) return respond({ error: 'missing params' });
  const sheet = SpreadsheetApp.openById(SHEET_ID).getSheetByName(session);
  if (!sheet) return respond({ error: 'session not found' });
  const rows = sheet.getDataRange().getValues();
  const headers = rows[0];
  const tokenCol = headers.indexOf('token');
  for (let i = 1; i < rows.length; i++) {
    if (String(rows[i][tokenCol]) === String(token)) {
      const response = {};
      headers.forEach((h, idx) => { response[h] = rows[i][idx]; });
      return respond({ open: FORM_OPEN, response });
    }
  }
  return respond({ open: FORM_OPEN, response: null });
}
```

---

## Configuring `index.html`

All session-specific settings are in the `CONFIG` block (~line 564):

```javascript
const CONFIG = {
  appsScriptUrl: 'https://script.google.com/macros/s/.../exec',
  sessionName:   'DataRobot GenAI and Agentic offering',  // must match Sheet tab name exactly
  sessionDateTime: 'May 7, 2026',
  topics: [
    'Agentic use cases',
    'Agent Application Templates',
    // ...
  ]
};
```

Session name and date can also be overridden per-link via URL params:
```
?session_name=My+Session&session_date=June+1+2026&token=abc123
```

---

## Known issues & fixes applied

| Issue | Root cause | Fix |
|---|---|---|
| Form shows "closed" on load | `!data.open` triggers on `undefined` when script returns an error object with no `open` field | Changed to strict `data.open === false` |
| CORS block on fetch | DataRobot Google Workspace policy blocks `fetch` to `script.google.com` from external origins | Replaced all `fetch` calls with JSONP (`<script>` tag injection) |
| Token not found despite correct value | Google Sheets auto-converts numeric-looking tokens to numbers; strict `===` fails on string vs number | Token comparison uses `String()` cast on both sides |
| Tokens must be stored as text | Same numeric conversion issue | Sender script and manual entry prefix tokens with `'` to force text storage |
| Submission notifications not received | `GmailApp` silently fails when called from `doGet` web app handler | Replaced with `MailApp`; run `testEmail()` manually to trigger re-authorization if needed |
| Deleting token rows breaks submissions | Script looks up token in session tab to validate; deleted rows return "token not found" | Never delete rows from session tab — clear data columns B–K only, leave column A (token) intact |

---

## Extra considerations

**Mail client compatibility** — plain `https://` links work universally. Form opens in browser, no auth required.

**Outlook Safelinks** — corporate Outlook may wrap URLs and mangle query params. The email sender script can be extended to use Bitly API for per-token short URLs if needed.

**QR code + token** — a single slide QR cannot carry per-participant tokens. Use the QR as a visual reminder; email is the delivery mechanism for personalised links.

**Response editing** — participants can return to their link and resubmit until the form is closed. Last submission wins.

**Apps Script limits** — 20,000 requests/day, 6 min execution/call. Gmail: 500 emails/day (free), 1,500/day (paid Workspace). Adequate for any session at realistic scale.

**New session checklist** — for each new session: (1) add a new tab to the feedback Sheet, (2) update `CONFIG` in `index.html`, (3) update `FEEDBACK_TAB_NAME`, `SESSION_DISPLAY`, `SESSION_DATE`, `CLOSE_DATE` in `Sender.gs`, (4) re-upload `index.html` to GitHub, (5) re-deploy `Code.gs` as a new version with updated `SESSION_DISPLAY` and `CLOSE_DATE`.

**Feedback Sheet management** — the History tab accumulates all submissions across sessions. The main session tab holds one row per participant (latest response only). Never delete token rows from the session tab — if tokens are accidentally deleted, recover them from the Email Sender sheet column D.

---

## Google Drive resources

- Feedback Sheet: https://docs.google.com/spreadsheets/d/1PHdafzR4gCTKoF3eXujathRsuo4DtcjccelK04h_7DE/edit
- Email Sender Sheet: https://docs.google.com/spreadsheets/d/1whH_ZcsgzVs3eMUsMH62nlrwceutWdnRo-y5GKhRuaA/edit

## References

- [Google Apps Script Web Apps](https://developers.google.com/apps-script/guides/web)
- [Apps Script Spreadsheet Service](https://developers.google.com/apps-script/reference/spreadsheet)
- [MailApp.sendEmail](https://developers.google.com/apps-script/reference/mail/mail-app) — used for notifications from web app context
- [GmailApp reference](https://developers.google.com/apps-script/reference/gmail/gmail-app) — used in Sender.gs for outbound invite emails
- [JSONP pattern](https://en.wikipedia.org/wiki/JSONP)
- [UUID v4 generator](https://www.uuidgenerator.net/)
- [QR code generator](https://qrcode.show)
- [GitHub Pages](https://pages.github.com/)
