# Plan: Share Report via Email (PDF + Link)

## Context

Users can already share reports via a public link toggle in the Share Report modal (`ReportShareModal.tsx`). However, there's no way to **proactively send** a report to someone via email. This feature adds the ability to enter email addresses in the share modal and send each recipient:

1. A branded HTML email with a "View Report" link
2. The report PDF as an attachment

PDF generation already exists via Playwright (`PdfExportService.generate_pdf()`) and takes 5-10 seconds, so the email send must happen asynchronously via a Celery task.

---

## Files to Create

| # | File | Purpose |
|---|------|---------|
| 1 | `ddpui/celeryworkers/report_tasks.py` | Celery task: generate PDF + send emails to all recipients |
| 2 | `ddpui/tests/core/reports/test_share_email.py` | Tests for the share-via-email flow |

## Files to Modify

| # | File | Change |
|---|------|--------|
| 1 | `ddpui/utils/awsses.py` | Add `send_email_with_attachment()` using MIME + `send_raw_email` |
| 2 | `ddpui/utils/email_templates.py` | Add `render_share_report_email()` template |
| 3 | `ddpui/schemas/report_schema.py` | Add `ShareViaEmailRequest` and `ShareViaEmailResponse` schemas |
| 4 | `ddpui/api/report_api.py` | Add `POST /{snapshot_id}/share/email/` endpoint |
| 5 | `webapp_v2/hooks/api/useReports.ts` | Add `shareReportViaEmail()` async function |
| 6 | `webapp_v2/components/reports/ReportShareModal.tsx` | Add "Share via Email" section with email input + send button |

---

## Implementation Details

### Step 1: `ddpui/utils/awsses.py` — Add `send_email_with_attachment`

SES's `send_email()` doesn't support attachments. We need `send_raw_email()` with a MIME multipart message.

**How MIME multipart works for our case:**

An email with HTML body + PDF attachment is structured as nested MIME parts:

```
multipart/mixed                          ← outer envelope
├── multipart/alternative                ← body (email clients pick one)
│   ├── text/plain                       ← plain-text fallback
│   └── text/html                        ← HTML body (preferred)
└── application/pdf (Content-Disposition: attachment)  ← the PDF file
```

Python's standard library (`email.mime`) builds this structure — no new dependencies needed.

```python
import email.mime.multipart
import email.mime.text
import email.mime.application

def send_email_with_attachment(to_email, subject, text_body, html_body, attachment_bytes, attachment_filename):
    """Send HTML email with a PDF attachment via SES send_raw_email."""
    ses = _get_ses_client()
    sender = os.getenv("SES_SENDER_EMAIL")

    msg = email.mime.multipart.MIMEMultipart("mixed")
    msg["Subject"] = subject
    msg["From"] = sender
    msg["To"] = to_email

    # HTML + plain-text body (alternative part)
    body_part = email.mime.multipart.MIMEMultipart("alternative")
    body_part.attach(email.mime.text.MIMEText(text_body, "plain", "utf-8"))
    body_part.attach(email.mime.text.MIMEText(html_body, "html", "utf-8"))
    msg.attach(body_part)

    # PDF attachment
    attachment = email.mime.application.MIMEApplication(attachment_bytes, "pdf")
    attachment.add_header("Content-Disposition", "attachment", filename=attachment_filename)
    msg.attach(attachment)

    return ses.send_raw_email(
        Source=sender,
        Destinations=[to_email],
        RawMessage={"Data": msg.as_string()},
    )
```

### Step 2: `ddpui/utils/email_templates.py` — Add `render_share_report_email`

New function following the same pattern as `render_mention_email()`. Returns `(plain_text, html)`.

**Template design** (matches the reference screenshot shared by user):
- Dalgo `#00897B` header bar with logo text
- Headline: `{sender_name} has shared "{report_title}" with you`
- Optional message from the sender (if they typed one)
- "View Report" CTA button linking to the public report URL
- "A PDF copy is also attached to this email." text below the CTA
- Footer explaining why they received it

```python
def render_share_report_email(
    sender_name: str,
    report_title: str,
    report_url: str,
    message: Optional[str] = None,
) -> tuple:
    """Render HTML + plain-text email for sharing a report."""
```

### Step 3: `ddpui/schemas/report_schema.py` — Add request/response schemas

```python
class ShareViaEmailRequest(Schema):
    """Schema for sharing a report via email"""
    recipient_emails: list[str] = Field(..., min_length=1, max_length=20)
    message: Optional[str] = Field(None, max_length=1000)

class ShareViaEmailResponse(Schema):
    """Schema for share-via-email response"""
    recipients_count: int
    message: str
```

### Step 4: `ddpui/api/report_api.py` — Add endpoint

```python
@report_router.post("/{snapshot_id}/share/email/", response=ApiResponse[ShareViaEmailResponse])
@has_permission(["can_share_dashboards"])
def share_report_via_email(request, snapshot_id: int, payload: ShareViaEmailRequest):
```

**Endpoint logic:**
1. Get the snapshot (404 if not found)
2. Ensure `public_share_token` exists (create one if not)
3. Enable `is_public=True` so the "View Report" link works for recipients
4. Build the public report URL
5. Dispatch Celery task with: `snapshot_id`, `share_token`, `recipient_emails`, `message`, `sender_name`, `report_title`, `report_url`
6. Return immediately with `ShareViaEmailResponse(recipients_count=len(emails), message="Emails are being sent")`

**Why auto-enable `is_public`:** Recipients clicking "View Report" need to access the report without authentication. The PDF attachment works regardless, but the link requires public access. We'll show a note in the frontend about this.

### Step 5: `ddpui/celeryworkers/report_tasks.py` — Celery task

```python
from ddpui.celery import app

@app.task(bind=True)
def send_report_email_task(self, snapshot_id, share_token, recipient_emails, sender_name, report_title, report_url, message=None):
```

**Task logic:**
1. Generate PDF once: `PdfExportService.generate_pdf(snapshot_id, share_token)` → `pdf_bytes`
2. Sanitize title for filename: `f"{safe_title}.pdf"`
3. Render email template: `render_share_report_email(sender_name, report_title, report_url, message)`
4. For each recipient email:
   - Call `send_email_with_attachment(to_email, subject, plain_text, html_body, pdf_bytes, filename)`
   - Log success/failure per recipient (failures don't stop other sends)
5. Log summary: "Sent report to X of Y recipients"

**Error handling:**
- If PDF generation fails, the entire task fails (no point sending emails without PDF)
- Individual email failures are caught and logged but don't stop other sends

### Step 6: `webapp_v2/hooks/api/useReports.ts` — Add API function

```typescript
export async function shareReportViaEmail(
  snapshotId: number,
  data: { recipient_emails: string[]; message?: string }
): Promise<{ recipients_count: number; message: string }> {
  const res = await apiPost(`/api/reports/${snapshotId}/share/email/`, data);
  return res.data;
}
```

### Step 7: `webapp_v2/components/reports/ReportShareModal.tsx` — Add email sharing UI

Add a new `Card` section between the "Public Access" card and the "Actions" row:

**UI elements:**
- Card with `Mail` icon and "Share via Email" heading
- Text input for typing email addresses
- "Add" button (or Enter key) to add email to a chip list
- Chip list showing added emails with X to remove
- Optional textarea for a personal message
- "Send" button (disabled when no emails, shows spinner while sending)
- Info text: "Recipients will receive a PDF attachment and a link to view the report online. Public access will be enabled automatically."

**State management:**
- `emailInput: string` — current text in the email input
- `recipientEmails: string[]` — list of added emails
- `personalMessage: string` — optional message
- `isSending: boolean` — loading state

**Email validation:** Basic regex check on add. Show toast error for invalid emails.

**On send:**
1. Call `shareReportViaEmail(snapshotId, { recipient_emails, message })`
2. On success: toast "Report is being sent to X recipients", clear the email list
3. On error: toast error
4. Refresh share status (since `is_public` may have been enabled)

---

## Key Design Decisions

1. **Celery task for async send**: PDF generation takes 5-10s + email sending. The API returns immediately so the user isn't blocked.

2. **Generate PDF once, send to all**: The PDF is the same for all recipients, so we generate it once and attach to each email.

3. **Auto-enable `is_public`**: The "View Report" link requires public access. We auto-enable it and inform the user in the UI. This is the same behavior as the existing PDF export endpoint which ensures a `public_share_token` exists.

4. **No new model/migration needed**: Reuses existing `ReportSnapshot` sharing fields (`public_share_token`, `is_public`).

5. **Standard library MIME**: Uses Python's `email.mime` module for multipart message construction — no new dependencies.

6. **SES `send_raw_email`**: Required for attachments. `send_email` only supports text/HTML body without attachments.

---

## Verification

1. **Run backend tests:**
   ```bash
   cd DDP_backend && uv run pytest ddpui/tests/core/reports/test_share_email.py -v
   ```

2. **Manual E2E test:**
   - Open a report → click Share → enter an email → click Send
   - Verify toast shows "Report is being sent"
   - Verify the share toggle flips to public
   - Check recipient's inbox for HTML email with PDF attachment
   - Click "View Report" in the email → should open the public report page

3. **Preview email template:**
   - Render `render_share_report_email()` and write to `/tmp/share_email_preview.html`, open in browser

---

## Implementation Order

1. `ddpui/utils/awsses.py` — add `send_email_with_attachment`
2. `ddpui/utils/email_templates.py` — add `render_share_report_email`
3. `ddpui/schemas/report_schema.py` — add request/response schemas
4. `ddpui/celeryworkers/report_tasks.py` — create Celery task
5. `ddpui/api/report_api.py` — add endpoint
6. `ddpui/tests/core/reports/test_share_email.py` — write and run tests
7. `webapp_v2/hooks/api/useReports.ts` — add API function
8. `webapp_v2/components/reports/ReportShareModal.tsx` — add email UI
