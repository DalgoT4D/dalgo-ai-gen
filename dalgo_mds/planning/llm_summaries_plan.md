# LLM Summaries Feature Implementation Plan

## Feature Overview
Implement automatic LLM summarization of failed pipeline logs triggered by Prefect webhook notifications. The goal is to reduce summary generation time from 30-45 seconds to near-instant by pre-generating summaries when failures occur.

## Context and Requirements
- **Current State**: Manual summarization takes 30-45 seconds per request
- **Target State**: Pre-generated summaries available instantly on failure
- **Trigger**: Prefect webhook notifications on pipeline failures
- **Scope**: Both Airbyte sync failures and general pipeline failures

## Implementation Architecture

### High-Level Data Flow
```
Prefect Pipeline Failure
    ↓
Webhook Notification (/webhooks/v1/notification/)
    ↓
handle_prefect_webhook (Celery Task)
    ↓
do_handle_prefect_webhook (checks failure state)
    ↓
[Identify failed task(s)]
    ↓
[For Airbyte Syncs] → Get latest failed job from DB
    ↓
summarize_logs (Celery Task)
    ↓
LlmSession (Database Storage)
```

## Detailed Implementation Plan

### Step 1: Modify summarize_logs to Handle System Calls

The existing `summarize_logs` function assumes `orguser` exists and accesses `orguser.org`. We need to modify it to handle system-triggered calls.

**File**: `/Users/ishankoradia/Tech4dev/Dalgo/platform/DDP_backend/ddpui/celeryworkers/tasks.py`

**Changes needed**:
1. Make `orguser_id` optional
2. Extract org from flow_run when orguser is None
3. Handle LlmSession creation with nullable orguser

```python
@app.task(bind=True)
def summarize_logs(
    self,
    orguser_id: str = None,  # Make optional for system calls
    type: str = LogsSummarizationType.DEPLOYMENT,
    flow_run_id: str = None,
    task_id: str = None,
    job_id: int = None,
    connection_id: str = None,
    attempt_number: int = 0,
    regenerate: bool = False,
):
    """
    Modified to support system-triggered summarization
    """
    taskprogress = SingleTaskProgress(self.request.id, 60 * 10)
    taskprogress.add({"message": "Started", "status": "running", "result": []})
    
    # Handle both user and system triggered calls
    orguser = None
    org = None
    
    if orguser_id:
        orguser = OrgUser.objects.filter(id=orguser_id).first()
        org = orguser.org if orguser else None
    else:
        # System-triggered call - extract org from flow_run
        if flow_run_id:
            from ddpui.utils.webhook_helpers import get_org_from_flow_run
            flow_run = prefect_service.get_flow_run_poll(flow_run_id)
            org = get_org_from_flow_run(flow_run)
        elif job_id and connection_id:
            # For airbyte jobs, get org from connection
            from ddpui.models.org import OrgPrefectBlockv1
            block = OrgPrefectBlockv1.objects.filter(
                block_name=connection_id, 
                block_type=AIRBYTECONNECTION
            ).first()
            org = block.org if block else None
    
    if not org:
        taskprogress.add({
            "message": "Unable to determine organization",
            "status": TaskProgressStatus.FAILED,
            "result": None,
        })
        return
    
    # Check if LLM is enabled for the org
    if not org.preferences.llm_optin:
        taskprogress.add({
            "message": "LLM not enabled for organization",
            "status": TaskProgressStatus.FAILED,
            "result": None,
        })
        return
    
    # Rest of the function remains the same, but use org directly instead of orguser.org
    # When querying/creating LlmSession, use orguser=orguser (can be None)
    # and org=org (always populated)
```

### Step 2: Add Summarization Logic to Webhook Handler

**File**: `/Users/ishankoradia/Tech4dev/Dalgo/platform/DDP_backend/ddpui/utils/webhook_helpers.py`

Add the summarization trigger at the end of `do_handle_prefect_webhook` function (after line 436):

```python
def do_handle_prefect_webhook(flow_run_id: str, state: str):
    """
    this is the webhook handler for prefect flow runs
    """
    # ... existing code ...
    
    finally:
        # ... existing airbyte job sync code (lines 422-436) ...
        
        # NEW: Trigger automatic summarization for failures
        if state in [FLOW_RUN_FAILED_STATE_NAME, FLOW_RUN_CRASHED_STATE_NAME]:
            org = get_org_from_flow_run(flow_run)
            if org and org.preferences.llm_optin:
                trigger_log_summarization_for_failed_flow.delay(flow_run_id, flow_run)
```

### Step 3: Create New Task to Handle Summarization Logic

**File**: `/Users/ishankoradia/Tech4dev/Dalgo/platform/DDP_backend/ddpui/celeryworkers/tasks.py`

Add a new task to handle the summarization logic:

```python
@app.task(bind=True, max_retries=3, default_retry_delay=60)
def trigger_log_summarization_for_failed_flow(self, flow_run_id: str, flow_run: dict = None):
    """
    Triggers automatic summarization for failed pipeline runs.
    Handles both regular pipelines and airbyte syncs.
    """
    try:
        # Get flow run if not provided
        if not flow_run:
            flow_run = prefect_service.get_flow_run_poll(flow_run_id)
        
        # Check if summary already exists to prevent duplicates
        existing_summary = LlmSession.objects.filter(
            flow_run_id=flow_run_id,
            session_type=LlmAssistantType.LOG_SUMMARIZATION
        ).exists()
        
        if existing_summary:
            logger.info(f"Summary already exists for flow_run {flow_run_id}")
            return
        
        # Get failed task information
        from ddpui.ddpprefect import prefect_service
        task_runs = prefect_service.get_flow_run_graphs(flow_run_id)
        
        # Find the failed task
        failed_task = None
        for task in task_runs:
            if task.get("state_type") == "FAILED" or task.get("state_name") == "DBT_TEST_FAILED":
                failed_task = task
                break
        
        if not failed_task:
            logger.warning(f"No failed task found in flow_run {flow_run_id}")
            return
        
        # Check if this is an airbyte sync task
        is_airbyte_task = False
        connection_id = None
        
        for task in flow_run.get("parameters", {}).get("config", {}).get("tasks", []):
            if task.get("slug", "") in [TASK_AIRBYTESYNC, TASK_AIRBYTERESET, TASK_AIRBYTECLEAR]:
                is_airbyte_task = True
                connection_id = task.get("connection_id", None)
                break
        
        # Trigger appropriate summarization
        if is_airbyte_task and connection_id:
            # For airbyte syncs, get the latest failed job from database
            from ddpui.models.org import AirbyteJob
            
            # Get the latest failed job for this connection
            # Assume job is already synced by the time we reach here
            airbyte_job = AirbyteJob.objects.filter(
                config_id=connection_id,
                status="failed"
            ).order_by('-created_at').first()
            
            if not airbyte_job:
                logger.error(f"No failed airbyte job found for connection {connection_id}")
                return
            
            # Trigger airbyte sync summarization
            summarize_logs.delay(
                orguser_id=None,  # System call
                type=LogsSummarizationType.AIRBYTE_SYNC,
                flow_run_id=flow_run_id,
                task_id=failed_task.get("id"),
                job_id=airbyte_job.job_id,
                connection_id=connection_id
            )
            
            logger.info(f"Triggered airbyte summarization for job_id {airbyte_job.job_id}")
        else:
            # Trigger deployment summarization
            summarize_logs.delay(
                orguser_id=None,  # System call
                type=LogsSummarizationType.DEPLOYMENT,
                flow_run_id=flow_run_id,
                task_id=failed_task.get("id")
            )
            
            logger.info(f"Triggered deployment summarization for task {failed_task.get('id')}")
        
        logger.info(f"Successfully triggered summarization for flow_run {flow_run_id}")
        
    except Exception as e:
        logger.exception(f"Error triggering summarization for {flow_run_id}: {str(e)}")
        raise self.retry(exc=e)
```

### Step 4: Import Required Dependencies

Add necessary imports at the top of the files:

**In webhook_helpers.py**:
```python
from ddpui.celeryworkers.tasks import trigger_log_summarization_for_failed_flow
```

**In tasks.py** (for the new task):
```python
import time
from ddpui.models.llm import LlmSession, LlmAssistantType
from ddpui.ddpprefect.schema import (
    TASK_AIRBYTESYNC,
    TASK_AIRBYTERESET,
    TASK_AIRBYTECLEAR,
    FLOW_RUN_FAILED_STATE_NAME,
    FLOW_RUN_CRASHED_STATE_NAME,
)
from ddpui.utils.constants import LogsSummarizationType
```

## Error Handling and Edge Cases

### 1. Circular Import Prevention
If circular imports occur, move the imports inside the functions:
```python
def trigger_log_summarization_for_failed_flow(...):
    from ddpui.utils.webhook_helpers import get_org_from_flow_run
    # ... rest of the function
```

### 2. Retry Strategy
- **Max Retries**: 3 attempts
- **Retry Delay**: 60 seconds (exponential backoff)
- **Specific retry scenarios**:
  - Prefect API temporary failures
  - LLM service unavailable

### 3. Duplicate Prevention
- Check for existing LlmSession before creating new summary
- Use flow_run_id + task_id as unique identifier

### 4. Org LLM Preference Check
- Skip summarization if org hasn't opted into LLM features
- Log but don't error when LLM is disabled

### 5. Airbyte Job Identification
- Query for the latest failed job with matching connection_id
- Assume job is already synced by the time webhook processing reaches this point
- Log error and return if no failed job found

## Testing Strategy

### Unit Tests

1. **Test summarize_logs with system user**:
```python
@patch('ddpui.celeryworkers.tasks.prefect_service.get_flow_run_poll')
@patch('ddpui.celeryworkers.tasks.get_org_from_flow_run')
def test_summarize_logs_system_user(mock_get_org, mock_get_flow_run):
    # Mock flow run and org
    mock_flow_run = {"id": "test-flow-run"}
    mock_org = Mock(preferences=Mock(llm_optin=True))
    mock_get_flow_run.return_value = mock_flow_run
    mock_get_org.return_value = mock_org
    
    # Call without orguser_id
    result = summarize_logs(
        orguser_id=None,
        type=LogsSummarizationType.DEPLOYMENT,
        flow_run_id="test-flow-run"
    )
    
    # Verify org was extracted from flow run
    mock_get_org.assert_called_once_with(mock_flow_run)
```

2. **Test trigger_log_summarization_for_failed_flow with Airbyte**:
```python
@patch('ddpui.celeryworkers.tasks.summarize_logs.delay')
@patch('ddpui.celeryworkers.tasks.AirbyteJob.objects.filter')
@patch('ddpui.celeryworkers.tasks.prefect_service.get_flow_run_graphs')
def test_trigger_summarization_airbyte_failure(mock_get_graphs, mock_airbyte_filter, mock_summarize):
    # Mock failed task
    mock_get_graphs.return_value = [
        {"id": "task-1", "state_type": "FAILED"}
    ]
    
    # Mock airbyte job query
    mock_job = Mock(job_id=12345)
    mock_airbyte_filter.return_value.order_by.return_value.first.return_value = mock_job
    
    # Trigger summarization for airbyte sync
    flow_run = {
        "parameters": {
            "config": {
                "tasks": [{
                    "slug": "airbyte-sync",
                    "connection_id": "conn-123"
                }]
            }
        }
    }
    
    trigger_log_summarization_for_failed_flow("flow-run-id", flow_run)
    
    # Verify AirbyteJob was queried correctly
    mock_airbyte_filter.assert_called_with(
        config_id="conn-123",
        status="failed"
    )
    
    # Verify summarize_logs was called with job_id from database
    mock_summarize.assert_called_once_with(
        orguser_id=None,
        type=LogsSummarizationType.AIRBYTE_SYNC,
        flow_run_id="flow-run-id",
        task_id="task-1",
        job_id=12345,
        connection_id="conn-123"
    )
```

3. **Test webhook integration**:
```python
@patch('ddpui.utils.webhook_helpers.trigger_log_summarization_for_failed_flow.delay')
def test_webhook_triggers_summarization_on_failure(mock_trigger):
    # Create org with LLM enabled
    org = Org.objects.create(name="Test Org")
    OrgPreferences.objects.create(org=org, llm_optin=True)
    
    # Call webhook handler with failure state
    do_handle_prefect_webhook("flow-run-id", FLOW_RUN_FAILED_STATE_NAME)
    
    # Verify summarization was triggered
    mock_trigger.assert_called_once()
```

### Integration Tests

1. **End-to-End Pipeline Failure**:
```python
def test_e2e_pipeline_failure_creates_summary():
    # Setup org with LLM enabled
    org = create_test_org_with_llm_enabled()
    
    # Create and trigger failing pipeline
    deployment = create_test_deployment(org)
    flow_run_id = trigger_pipeline_failure(deployment)
    
    # Wait for async processing
    time.sleep(5)
    
    # Verify summary was created
    summary = LlmSession.objects.filter(
        flow_run_id=flow_run_id,
        orguser__isnull=True  # System generated
    ).first()
    
    assert summary is not None
    assert summary.response is not None
```

2. **End-to-End Airbyte Failure**:
```python
def test_e2e_airbyte_failure_creates_summary():
    # Setup org and connection
    org = create_test_org_with_llm_enabled()
    connection = create_test_airbyte_connection(org)
    
    # Create failed airbyte job in database
    failed_job = AirbyteJob.objects.create(
        job_id=99999,
        config_id=connection.connection_id,
        status="failed",
        created_at=timezone.now()
    )
    
    # Trigger webhook for failed airbyte sync
    flow_run = create_airbyte_flow_run(connection.connection_id)
    do_handle_prefect_webhook(flow_run["id"], FLOW_RUN_FAILED_STATE_NAME)
    
    # Wait and verify
    time.sleep(5)
    summary = LlmSession.objects.filter(
        airbyte_job_id=failed_job.job_id
    ).first()
    
    assert summary is not None
```

### Manual Testing Steps

1. **Enable LLM for test org**:
   ```sql
   UPDATE ddpui_orgpreferences SET llm_optin = true WHERE org_id = <test_org_id>;
   ```

2. **Test Regular Pipeline Failure**:
   - Create a dbt model with syntax error
   - Trigger pipeline execution
   - Verify webhook is received
   - Check LlmSession for summary

3. **Test Airbyte Sync Failure**:
   - Configure airbyte connection
   - Force sync failure (invalid credentials)
   - Verify job is synced to AirbyteJob table
   - Check that latest failed job is used for summary
   - Verify LlmSession contains correct job_id

4. **Test Duplicate Prevention**:
   - Trigger same failure twice
   - Verify only one summary is created

## Monitoring and Observability

### Logging
Add comprehensive logging:
```python
logger.info(f"Triggering automatic summarization for flow_run {flow_run_id}")
logger.info(f"Organization {org.id} has LLM enabled: {org.preferences.llm_optin}")
logger.info(f"Found failed task {task_id} in flow_run {flow_run_id}")
logger.info(f"Found airbyte job {job_id} for connection {connection_id}")
logger.info(f"Summary created for flow_run {flow_run_id}, session_id: {session.id}")
```

### Metrics to Track
- Count of automatic summarizations triggered
- Success/failure rate
- Average processing time
- LLM API latency

## Implementation Checklist

- [ ] Modify `summarize_logs` to handle system calls (no orguser)
- [ ] Add summarization trigger to `do_handle_prefect_webhook`
- [ ] Create `trigger_log_summarization_for_failed_flow` task
- [ ] Implement logic to query latest failed AirbyteJob from database
- [ ] Handle circular imports if they occur
- [ ] Write unit tests for all modified functions
- [ ] Write integration tests
- [ ] Test with real Prefect webhooks
- [ ] Test with airbyte sync failures
- [ ] Verify duplicate prevention works
- [ ] Verify correct job_id is retrieved for airbyte failures
- [ ] Add comprehensive logging
- [ ] Review code for production readiness
- [ ] Update any relevant documentation

## External Documentation References

### Celery Best Practices
- **Task Resilience**: https://blog.gitguardian.com/celery-tasks-retries-errors/
- **Retry Patterns**: https://testdriven.io/blog/retrying-failed-celery-tasks/
- **Async LLM Processing**: https://medium.com/algomart/async-llm-tasks-with-celery-and-celery-beat-31c824837f35

### Webhook Handling
- **Queue-Based Architecture**: https://webhookantipatterns.com/
- **Idempotency**: https://reintech.io/blog/error-handling-retry-policies-celery-tasks

## Confidence Score: 9.5/10

This plan provides comprehensive context and clear implementation steps. The key simplification from the previous version:
- Removed the retry loop for waiting for Airbyte job sync
- Assumes job is already synced when webhook processing reaches this point
- Simpler error handling - just log and return if job not found

The implementation leverages existing infrastructure with minimal new code, making it straightforward to implement and maintain.