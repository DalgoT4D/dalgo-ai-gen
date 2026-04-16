# Scheduled Reports Feature Plan

## Context

Users can already share reports via one-off email (PDF attachment + link). This feature adds **recurring scheduled reports**: the system auto-generates a new ReportSnapshot from a dashboard on a cron schedule, generates a PDF via Playwright, and emails it to a list of recipients. Uses `django-celery-beat` for scheduling persistence.

**Key decisions:**
- Auto-generate new snapshots each run (not re-send same snapshot)
- All three frequency presets: Daily, Weekly (pick day), Monthly (pick date)
- Previous period date ranges: Daily=yesterday, Weekly=prev Mon-Sun, Monthly=prev calendar month
- `django-celery-beat` for scheduling
- UI lives inside the report share menu (third option alongside "Share via link" and "Embed in email")
- Run status tracking in MVP (last_run_at, last_run_status, last_error_message)

---

## Implementation Order

### Step 1: Install django-celery-beat

**Files to modify:**

| File | Change |
|------|--------|
| `DDP_backend/pyproject.toml` | Add `"django-celery-beat>=2.7.0"` to dependencies |
| `ddpui/settings.py` | Add `"django_celery_beat"` to `INSTALLED_APPS` |
| `ddpui/celery.py` | Add `app.conf.beat_scheduler = "django_celery_beat.schedulers:DatabaseScheduler"` |

Then run: `uv sync && python manage.py migrate django_celery_beat`

Note: The Docker entrypoint's `--schedule=/data/celerybeat-schedule` flag is harmlessly ignored when using `DatabaseScheduler`. Can be removed for clarity.

---

### Step 2: Add ReportSchedule model

**File:** `ddpui/models/report.py` (append after `ReportSnapshot`)

```python
from django_celery_beat.models import PeriodicTask

class ReportSchedule(models.Model):
    class Frequency(models.TextChoices):
        DAILY = "daily", "Daily"
        WEEKLY = "weekly", "Weekly"
        MONTHLY = "monthly", "Monthly"

    id = models.BigAutoField(primary_key=True)
    dashboard_id = models.IntegerField()          # NOT a FK (survives dashboard deletion)
    title_template = models.CharField(max_length=255)  # Supports {date} placeholder
    date_column = models.JSONField(default=dict)  # {schema_name, table_name, column_name}
    frequency = models.CharField(max_length=10, choices=Frequency.choices)
    day_of_week = models.IntegerField(null=True, blank=True)   # 0=Mon..6=Sun (weekly)
    day_of_month = models.IntegerField(null=True, blank=True)  # 1-28 (monthly)
    time_of_day = models.TimeField()
    timezone = models.CharField(max_length=63, default="UTC")
    recipient_emails = models.JSONField(default=list)
    subject_template = models.CharField(max_length=255, blank=True, default="")
    is_active = models.BooleanField(default=True)
    periodic_task = models.OneToOneField(
        PeriodicTask, on_delete=models.SET_NULL, null=True, blank=True
    )
    org = models.ForeignKey(Org, on_delete=models.CASCADE)
    created_by = models.ForeignKey(OrgUser, on_delete=models.SET_NULL, null=True)
    # Run status tracking
    last_run_at = models.DateTimeField(null=True, blank=True)
    last_run_status = models.CharField(
        max_length=20, null=True, blank=True,
        choices=[("success", "Success"), ("partial_failure", "Partial Failure"), ("failed", "Failed")]
    )
    last_error_message = models.TextField(blank=True, default="")
    last_snapshot = models.ForeignKey(
        "ReportSnapshot", on_delete=models.SET_NULL, null=True, blank=True,
        related_name="source_schedule"
    )

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "report_schedule"
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["org", "is_active"]),
            models.Index(fields=["dashboard_id"]),
        ]
```

**Key design decisions:**
- `dashboard_id` is IntegerField (not FK) -- mirrors how `ReportSnapshot.frozen_dashboard["dashboard_id"]` stores it. If dashboard is deleted, schedule fails gracefully with a notification instead of cascade-deleting.
- `day_of_month` capped at 28 in validation to avoid end-of-month issues.
- `periodic_task` is OneToOneField to `django_celery_beat.PeriodicTask` with `SET_NULL` -- we manage its lifecycle explicitly in the service layer.
- `last_run_at/status/error_message/snapshot` -- essential for users to see if their schedule is working. Updated by the Celery task after each execution.

**Migration:** `ddpui/migrations/0157_reportschedule.py` (auto-generated via `python manage.py makemigrations ddpui`)

---

### Step 3: Add schedule exceptions

**File:** `ddpui/core/reports/exceptions.py` (append after existing CommentPermissionError)

```python
class ScheduleError(Exception):
    def __init__(self, message, error_code="SCHEDULE_ERROR"):
        self.message = message
        self.error_code = error_code
        super().__init__(self.message)

class ScheduleNotFoundError(ScheduleError): ...
class ScheduleValidationError(ScheduleError): ...
class SchedulePermissionError(ScheduleError): ...
```

Follows the exact same pattern as `SnapshotNotFoundError`, `CommentNotFoundError`, etc.

---

### Step 4: Build ScheduleService

**File:** `ddpui/core/reports/schedule_service.py` (new)

**Class:** `ScheduleService` with these static methods:

#### Date range calculation
```python
@staticmethod
def calculate_date_range(frequency: str, reference_date: date) -> tuple[date, date]:
    # daily:   (yesterday, yesterday)
    # weekly:  (prev Monday, prev Sunday)
    # monthly: (1st of prev month, last of prev month)
```

#### Celery-Beat integration
```python
@staticmethod
def _build_crontab(schedule: ReportSchedule) -> CrontabSchedule:
    # daily   -> minute=M, hour=H, day_of_week=*, day_of_month=*
    # weekly  -> minute=M, hour=H, day_of_week=N
    # monthly -> minute=M, hour=H, day_of_month=N
    # Uses CrontabSchedule.objects.get_or_create() with timezone

@staticmethod
def _sync_periodic_task(schedule: ReportSchedule) -> PeriodicTask:
    # Creates or updates PeriodicTask linked to this schedule
    # task = "ddpui.celeryworkers.report_tasks.execute_scheduled_report"
    # kwargs = {"schedule_id": schedule.id}

@staticmethod
def _delete_periodic_task(schedule: ReportSchedule) -> None:
    # Deletes PeriodicTask, nulls the FK on schedule
```

#### CRUD
```python
@staticmethod
def create_schedule(dashboard_id, title_template, date_column, frequency,
                    time_of_day, recipient_emails, orguser, timezone="UTC",
                    day_of_week=None, day_of_month=None, subject_template="") -> ReportSchedule:
    # Validates: dashboard exists, frequency-specific fields, timezone, recipients (max 20)
    # Creates ReportSchedule + syncs PeriodicTask in a transaction

@staticmethod
def list_schedules(org, dashboard_id=None) -> List[ReportSchedule]:

@staticmethod
def get_schedule(schedule_id, org) -> ReportSchedule:

@staticmethod
def update_schedule(schedule_id, org, orguser, **kwargs) -> ReportSchedule:
    # Only creator can modify. Re-syncs PeriodicTask if schedule fields changed.

@staticmethod
def delete_schedule(schedule_id, org, orguser) -> bool:
    # Only creator can delete. Deletes PeriodicTask in transaction.

@staticmethod
def toggle_active(schedule_id, org, orguser, is_active) -> ReportSchedule:
```

**Reuses:**
- `Dashboard.objects.filter()` from `ddpui/models/dashboard.py` for validation
- `CrontabSchedule`, `PeriodicTask` from `django_celery_beat.models`
- `pytz.timezone()` for timezone validation

---

### Step 5: Add Celery task

**File:** `ddpui/celeryworkers/report_tasks.py` (append after existing `send_report_email_task`)

```python
@app.task(bind=True, max_retries=2, default_retry_delay=300)
def execute_scheduled_report(self, schedule_id):
    """Execute a scheduled report: create snapshot, generate PDF, email recipients."""
```

**Task flow:**
1. Fetch `ReportSchedule` from DB (skip if not found or inactive)
2. Calculate date range via `ScheduleService.calculate_date_range(frequency, today)`
3. Build title from template: `title_template.replace("{date}", period_end.isoformat())`
4. Create snapshot via `ReportService.create_snapshot(title, dashboard_id, date_column, period_end, orguser, period_start)`
5. Generate PDF via `PdfExportService.generate_pdf(snapshot.id, share_token)`
6. Render email via `render_share_report_email(sender_name="Dalgo (Scheduled)", ...)`
7. Send to each recipient via `send_email_with_attachment()`
8. **Update schedule status**: `last_run_at=now, last_run_status="success"/"partial_failure"/"failed", last_snapshot=snapshot, last_error_message=...`
9. On partial failure: notify schedule creator via `create_notification()`
10. On complete failure: notify + `self.retry(exc=e)` (max 2 retries, 5min delay)

**Resource safeguard:** Route this task to a dedicated Celery queue `report_schedules` with `worker_concurrency=3` to prevent Playwright from exhausting server memory. Add to `celery.py`:
```python
app.conf.task_routes["ddpui.celeryworkers.report_tasks.execute_scheduled_report"] = {"queue": "report_schedules"}
```

**Reuses** all existing infrastructure:
- `ReportService.create_snapshot()` from `ddpui/core/reports/report_service.py`
- `ReportService.ensure_share_token()` from same
- `PdfExportService.generate_pdf()` from `ddpui/core/reports/pdf_export_service.py`
- `send_email_with_attachment()` from `ddpui/utils/awsses.py`
- `render_share_report_email()` from `ddpui/utils/email_templates.py`
- `create_notification()` from `ddpui/core/notifications/notifications_functions.py`

---

### Step 6: Add schemas

**File:** `ddpui/schemas/report_schema.py` (append)

| Schema | Purpose |
|--------|---------|
| `ScheduleCreate` | Create request: dashboard_id, title_template, date_column, frequency, day_of_week, day_of_month, time_of_day, timezone, recipient_emails, subject_template |
| `ScheduleUpdate` | Partial update: all fields optional |
| `ScheduleToggle` | Enable/disable: `is_active: bool` |
| `ScheduleResponse` | Response with `from_model()` classmethod. Includes `dashboard_title`, `last_run_at`, `last_run_status`, `last_error_message`, `last_snapshot_id` |

---

### Step 7: Add API endpoints

**File:** `ddpui/api/report_api.py` (append)

All on existing `report_router`, permission `can_share_dashboards`:

| Method | Path | Handler |
|--------|------|---------|
| `GET` | `/dashboards/{dashboard_id}/schedules/` | List schedules for a dashboard |
| `POST` | `/schedules/` | Create schedule |
| `GET` | `/schedules/{schedule_id}/` | Get schedule |
| `PUT` | `/schedules/{schedule_id}/` | Update schedule |
| `DELETE` | `/schedules/{schedule_id}/` | Delete schedule |
| `PUT` | `/schedules/{schedule_id}/toggle/` | Enable/disable |

Follows same pattern as existing endpoints: thin API layer delegates to `ScheduleService`, converts exceptions to `HttpError`.

---

### Step 8: Backend tests

**File:** `ddpui/tests/core/reports/test_schedule_service.py` (new)

**Test cases:**
- `test_calculate_date_range_daily` -- reference 2026-04-12 -> (2026-04-11, 2026-04-11)
- `test_calculate_date_range_weekly` -- reference 2026-04-12 (Sat) -> prev Mon-Sun
- `test_calculate_date_range_monthly` -- reference 2026-04-15 -> (2026-03-01, 2026-03-31)
- `test_calculate_date_range_monthly_january` -- reference 2026-01-05 -> (2025-12-01, 2025-12-31)
- `test_create_schedule_creates_periodic_task` -- verify PeriodicTask + CrontabSchedule
- `test_update_schedule_syncs_crontab` -- change frequency, verify crontab updates
- `test_toggle_disables_periodic_task` -- toggle off, verify enabled=False
- `test_delete_schedule_cleans_up` -- verify PeriodicTask deleted
- `test_create_schedule_invalid_dashboard` -- 400 error
- `test_create_schedule_invalid_day_of_week` -- validation error
- `test_execute_scheduled_report_task` -- mock PDF + email, verify calls

---

### Step 9: Frontend types & hooks

**File:** `webapp_v2/types/reports.ts` (append)

```typescript
export type ScheduleFrequency = 'daily' | 'weekly' | 'monthly';
export type ScheduleRunStatus = 'success' | 'partial_failure' | 'failed';

export interface ReportSchedule {
  id: number;
  dashboard_id: number;
  dashboard_title?: string;
  title_template: string;
  date_column?: DateColumn;
  frequency: ScheduleFrequency;
  day_of_week?: number;
  day_of_month?: number;
  time_of_day: string;  // "HH:MM"
  timezone: string;
  recipient_emails: string[];
  subject_template?: string;
  is_active: boolean;
  // Run status
  last_run_at?: string;
  last_run_status?: ScheduleRunStatus;
  last_error_message?: string;
  last_snapshot_id?: number;
  // Metadata
  created_by?: string;
  created_at: string;
  updated_at: string;
}
```

**File:** `webapp_v2/hooks/api/useReports.ts` (append)

- `useDashboardSchedules(dashboardId)` -- SWR hook for listing schedules
- `createSchedule(data)` -- POST mutation
- `updateSchedule(id, data)` -- PUT mutation
- `deleteSchedule(id)` -- DELETE mutation
- `toggleSchedule(id, isActive)` -- PUT toggle mutation

---

### Step 10: Frontend UI components

**File:** `webapp_v2/components/reports/report-share-menu.tsx` (modify)

Add third `DropdownMenuItem` with `<Clock />` icon: "Schedule recurring". Opens `ScheduleDialog`. Needs new prop `dashboardId` (extracted from `frozen_dashboard.dashboard_id` in the report view data) and `dateColumn`.

**File:** `webapp_v2/components/reports/schedule-dialog.tsx` (new)

Dialog containing:
- List of existing schedules for this dashboard (fetched via `useDashboardSchedules`)
- Each schedule as a card showing:
  - Title template + frequency description ("Weekly on Monday at 09:00 UTC")
  - Last run status badge (green=success, yellow=partial, red=failed) + timestamp
  - Recipients count
  - Active/paused toggle
  - Edit / Delete actions
- "Create schedule" button opens the form
- Empty state: "Automate your reporting. Create a schedule to auto-generate and email reports."
- Uses existing `Dialog` component pattern from `share-via-email-dialog.tsx`

**File:** `webapp_v2/components/reports/schedule-form.tsx` (new)

Form (React Hook Form) with:
- Frequency selector (Select: Daily/Weekly/Monthly)
- Day-of-week picker (shown for weekly, Select: Mon-Sun)
- Day-of-month picker (shown for monthly, Select: 1-28)
- Time picker (Input: HH:MM)
- Timezone selector (Select: common IANA timezones)
- Title template (Input with `{date}` hint)
- Recipient emails (reuse email chip pattern from `share-via-email-dialog.tsx`)
- Subject template (Input, optional)

**File:** `webapp_v2/components/reports/utils.ts` (append)

- `FREQUENCY_OPTIONS`, `DAY_OF_WEEK_OPTIONS`, `TIMEZONE_OPTIONS` constants
- `formatScheduleFrequency()` -- "Weekly on Monday at 09:00 UTC"

---

## Resource Safeguards

| Concern | Mitigation |
|---------|------------|
| Playwright memory (each Chromium ~200-400MB) | Dedicated Celery queue `report_schedules` with `worker_concurrency=3` |
| Peak load (many schedules at 9am) | Celery queue serializes execution; 3 concurrent max |
| Unbounded snapshot growth | Future: add `max_snapshots` field, auto-cleanup old ones in task |
| Email rate limits (SES) | Individual send per recipient with error handling; partial failures don't block others |

---

## Edge Cases

| Scenario | Handling |
|----------|----------|
| Dashboard deleted | Task fails on `ReportService.create_snapshot()`, sets `last_run_status="failed"`, notifies creator |
| Creator removed from org | `created_by` is `SET_NULL`, task skips with log |
| Duplicate execution | Idempotent -- creates extra snapshot, acceptable |
| DST transitions | `django-celery-beat` CrontabSchedule handles natively |
| Monthly on 29-31 | Prevented by validation (capped at 28) |
| 3 consecutive failures | Notifies creator each time; manual intervention to fix/disable |

---

## Verification

1. **Unit tests:** `uv run pytest ddpui/tests/core/reports/test_schedule_service.py -v`
2. **Migration:** `python manage.py makemigrations ddpui && python manage.py migrate`
3. **Manual E2E:**
   - Open a report -> Share -> "Schedule recurring"
   - Create a daily schedule with 1 recipient
   - Verify in Django admin: `PeriodicTask` + `CrontabSchedule` records exist
   - Force-run the task: `from ddpui.celeryworkers.report_tasks import execute_scheduled_report; execute_scheduled_report(schedule_id)`
   - Verify: new ReportSnapshot created, PDF generated, email received
   - Toggle schedule off -> verify `PeriodicTask.enabled=False`
   - Delete schedule -> verify `PeriodicTask` deleted

---

## Critical Files Reference

| Existing file | What we reuse |
|---------------|---------------|
| `ddpui/celery.py` | Celery app (add beat_scheduler config) |
| `ddpui/models/report.py` | Add ReportSchedule model next to ReportSnapshot |
| `ddpui/core/reports/report_service.py` | `create_snapshot()`, `ensure_share_token()`, `_build_private_url()` |
| `ddpui/core/reports/pdf_export_service.py` | `PdfExportService.generate_pdf()` |
| `ddpui/celeryworkers/report_tasks.py` | Add `execute_scheduled_report` task next to `send_report_email_task` |
| `ddpui/utils/awsses.py` | `send_email_with_attachment()` |
| `ddpui/utils/email_templates.py` | `render_share_report_email()` |
| `ddpui/core/notifications/notifications_functions.py` | `create_notification()` for error alerts |
| `ddpui/schemas/report_schema.py` | Add schedule schemas |
| `ddpui/api/report_api.py` | Add schedule endpoints to existing `report_router` |
| `ddpui/core/reports/exceptions.py` | Add schedule exception classes |
| `webapp_v2/components/reports/report-share-menu.tsx` | Add "Schedule recurring" menu item |
| `webapp_v2/hooks/api/useReports.ts` | Add schedule hooks |
| `webapp_v2/components/reports/share-via-email-dialog.tsx` | Pattern reference for dialog + email chips |
