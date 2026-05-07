# Session feedback — email automation

## Overview

A Google Apps Script attached to the **Email Sender** spreadsheet. One click:
1. Generates a UUID token for each participant (skips rows that already have one)
2. Writes tokens into the feedback Sheet so the form can validate them
3. Sends each participant a personalised email with their unique link
4. Marks each row as `sent` with a timestamp so you never double-send

---

## Spreadsheet setup

**Email Sender sheet** (already created in your Drive):
[Session Feedback — Email Sender](https://docs.google.com/spreadsheets/d/1whH_ZcsgzVs3eMUsMH62nlrwceutWdnRo-y5GKhRuaA/edit)

Columns:

| A | B | C | D | E |
|---|---|---|---|---|
| name | email | token | link | sent |

- Fill in `name` and `email` for each participant
- Leave `token`, `link`, `sent` blank — the script fills these in
- You can add rows at any time and re-run; already-sent rows are skipped

---

## Apps Script — `Sender.gs`

Open the Email Sender sheet → **Extensions → Apps Script** → paste this:

```javascript
const FEEDBACK_SHEET_ID   = '1PHdafzR4gCTKoF3eXujathRsuo4DtcjccelK04h_7DE';
const FEEDBACK_TAB_NAME   = 'DataRobot GenAI and Agentic offering';
const FORM_BASE_URL       = 'https://olga-shpyrko-dr.github.io/session-feedback/';
const SESSION_DISPLAY     = 'DataRobot GenAI and Agentic Offering';
const SESSION_DATE        = 'May 7, 2026';
const SENDER_NAME         = 'DataRobot';
const CLOSE_DATE          = 'May 14, 2026';

function generateUUID() {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
    const r = Math.random() * 16 | 0;
    return (c === 'x' ? r : (r & 0x3 | 0x8)).toString(16);
  });
}

function sendFeedbackEmails() {
  const senderSheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Sheet1')
    || SpreadsheetApp.getActiveSpreadsheet().getSheets()[0];
  const feedbackSheet = SpreadsheetApp.openById(FEEDBACK_SHEET_ID)
    .getSheetByName(FEEDBACK_TAB_NAME);

  if (!feedbackSheet) {
    SpreadsheetApp.getUi().alert('Could not find the feedback sheet tab: ' + FEEDBACK_TAB_NAME);
    return;
  }

  const data = senderSheet.getDataRange().getValues();
  const headers = data[0]; // name, email, token, link, sent
  let sent = 0;
  let skipped = 0;

  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    const name  = String(row[0]).trim();
    const email = String(row[1]).trim();
    const alreadySent = String(row[4]).trim();

    if (!name || !email) continue;
    if (alreadySent) { skipped++; continue; }

    // Generate token
    const token = generateUUID();
    const link  = FORM_BASE_URL + '?token=' + token;

    // Write token to feedback sheet (new row)
    feedbackSheet.appendRow([
      "'" + token,  // force text with apostrophe
      FEEDBACK_TAB_NAME,
      '', '', '', '', '', '', '', '', ''
    ]);

    // Write token + link + sent timestamp back to sender sheet
    senderSheet.getRange(i + 1, 3).setValue("'" + token);
    senderSheet.getRange(i + 1, 4).setValue(link);
    senderSheet.getRange(i + 1, 5).setValue(new Date().toISOString());

    // Send email
    const subject = 'Your feedback — ' + SESSION_DISPLAY + ' (' + SESSION_DATE + ')';
    const body = buildEmailBody(name, link);
    GmailApp.sendEmail(email, subject, '', { htmlBody: body, name: SENDER_NAME });

    sent++;
    Utilities.sleep(200); // avoid Gmail rate limits
  }

  SpreadsheetApp.getUi().alert(
    'Done! Sent: ' + sent + '   Skipped (already sent): ' + skipped
  );
}

function buildEmailBody(name, link) {
  const firstName = name.split(' ')[0];
  return `
<div style="font-family: 'Helvetica Neue', Arial, sans-serif; max-width: 600px; margin: 0 auto;">

  <div style="background: #0B0B0B; padding: 24px 32px;">
    <span style="color: #FFFFFF; font-size: 18px; font-weight: 500;">DataRobot</span>
    <div style="height: 3px; background: #81FBA5; margin-top: 16px;"></div>
  </div>

  <div style="padding: 32px; background: #FFFFFF;">
    <p style="font-size: 15px; color: #0B0B0B; margin: 0 0 16px;">Hi ${firstName},</p>

    <p style="font-size: 15px; color: #333; line-height: 1.6; margin: 0 0 16px;">
      Thank you for joining <strong>${SESSION_DISPLAY}</strong> on ${SESSION_DATE}.
      We'd love to hear your thoughts — it takes about 2 minutes.
    </p>

    <div style="margin: 28px 0;">
      <a href="${link}"
         style="background: #81FBA5; color: #0B0B0B; text-decoration: none;
                padding: 13px 28px; border-radius: 6px; font-size: 15px;
                font-weight: 500; display: inline-block;">
        Share your feedback →
      </a>
    </div>

    <p style="font-size: 13px; color: #777; line-height: 1.6; margin: 0 0 8px;">
      This link is personal to you. You can return to it and update your response
      until <strong>${CLOSE_DATE}</strong>.
    </p>

    <p style="font-size: 13px; color: #999; margin: 0;">
      If the button doesn't work, copy and paste this link into your browser:<br>
      <a href="${link}" style="color: #5C41FF;">${link}</a>
    </p>
  </div>

  <div style="background: #F5F5F3; padding: 16px 32px;">
    <p style="font-size: 12px; color: #999; margin: 0;">
      DataRobot · This email was sent to you as a session participant.
    </p>
  </div>

</div>`;
}

function generateTokensOnly() {
  const senderSheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Sheet1')
    || SpreadsheetApp.getActiveSpreadsheet().getSheets()[0];
  const feedbackSheet = SpreadsheetApp.openById(FEEDBACK_SHEET_ID)
    .getSheetByName(FEEDBACK_TAB_NAME);

  if (!feedbackSheet) {
    SpreadsheetApp.getUi().alert('Could not find feedback sheet tab: ' + FEEDBACK_TAB_NAME);
    return;
  }

  const data = senderSheet.getDataRange().getValues();
  let count = 0;

  for (let i = 1; i < data.length; i++) {
    const name          = String(data[i][0]).trim();
    const email         = String(data[i][1]).trim();
    const existingToken = String(data[i][2]).trim();

    if (!name || !email) continue;
    if (existingToken) continue; // already has a token, skip

    const token = generateUUID();
    const link  = FORM_BASE_URL + '?token=' + token;

    // Write token to feedback sheet
    feedbackSheet.appendRow([
      "'" + token, FEEDBACK_TAB_NAME,
      '', '', '', '', '', '', '', '', ''
    ]);

    // Write token + link back to sender sheet (no sent timestamp yet)
    senderSheet.getRange(i + 1, 3).setValue("'" + token);
    senderSheet.getRange(i + 1, 4).setValue(link);
    count++;
  }

  SpreadsheetApp.getUi().alert('Done! Generated ' + count + ' new tokens.');
}

function onOpen() {
  SpreadsheetApp.getUi()
    .createMenu('📧 Feedback')
    .addItem('Generate tokens only', 'generateTokensOnly')
    .addItem('Send feedback emails', 'sendFeedbackEmails')
    .toUi();
}
```

---

## How to use

### First time setup
1. Open **Session Feedback — Email Sender** sheet
2. **Extensions → Apps Script** → paste `Sender.gs` above
3. Update the constants at the top:
   - `FEEDBACK_SHEET_ID` — already set to your sheet
   - `FEEDBACK_TAB_NAME` — already set
   - `FORM_BASE_URL` — already set
   - `SESSION_DISPLAY`, `SESSION_DATE`, `CLOSE_DATE` — update per session
4. Save (`Ctrl+S`), then run `onOpen` once manually to authorise permissions
5. Grant access when Google prompts (needs Gmail + Sheets access)

### Menu options

| Menu item | What it does |
|---|---|
| **Generate tokens only** | Creates UUID tokens, writes them to the feedback Sheet and the `token` + `link` columns — no email sent |
| **Send feedback emails** | Sends emails to all rows that have no `sent` timestamp — generates tokens for any row that doesn't have one yet |

### Recommended two-step workflow
1. Fill in participant `name` and `email` columns
2. Run **Generate tokens only** — review the `token` and `link` columns in the sheet
3. When ready to send, run **Send feedback emails** — only rows without a `sent` timestamp are emailed

### One-step workflow
Skip straight to **Send feedback emails** — it generates tokens and sends emails in one go.

### Re-running safely
- Rows with a value in the `sent` column are always skipped by the email sender
- Rows with a value in the `token` column are always skipped by the token generator
- Add new participants any time and re-run either step — only new rows are processed
- To resend to someone, clear their `sent` cell and re-run **Send feedback emails**
- To regenerate a token, clear both `token` and `sent` cells and re-run

---

## Email preview

Subject: `Your feedback — DataRobot GenAI and Agentic Offering (May 7, 2026)`

The email is HTML, DataRobot branded (black header, green accent bar, green CTA button). It includes:
- Personalised greeting with first name
- Session name and date
- Green "Share your feedback →" button
- Plain-text fallback link below the button
- Close date reminder

---

## Extra considerations

**Gmail daily send limit** — free Google Workspace accounts: 500 emails/day. Paid: 1,500/day. Adequate for any session.

**Token format** — UUID v4 (e.g. `a3f8c2d1-...`). These are always strings so no number/text mismatch in the sheet.

**Apostrophe prefix** — tokens are written with a leading `'` to force Google Sheets to store them as text, avoiding the numeric comparison bug.

**Multiple sessions** — update `SESSION_DISPLAY`, `SESSION_DATE`, `CLOSE_DATE`, and `FEEDBACK_TAB_NAME` constants before each session. Create a new tab in the feedback sheet for each session.

**Test before sending** — add yourself as row 2 with your own email and run once to verify the email looks right before adding all participants.

---

## References
- [GmailApp.sendEmail](https://developers.google.com/apps-script/reference/gmail/gmail-app#sendEmail(String,String,String))
- [SpreadsheetApp](https://developers.google.com/apps-script/reference/spreadsheet/spreadsheet-app)
- [Apps Script UI menus](https://developers.google.com/apps-script/guides/menus)
