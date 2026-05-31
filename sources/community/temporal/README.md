# Temporal

Query self-hosted Temporal Server workflow runtime data — namespaces, workflow executions, and schedules — via the Temporal HTTP API (v1.24+).

## Requirements

- **Temporal Server v1.24 or later** with the HTTP API enabled (the HTTP API frontend is separate from the gRPC frontend and typically runs on port 7243).
- For **open self-hosted clusters** (no auth): leave `TEMPORAL_API_KEY` unset.
- For **auth-enabled clusters**: a bearer token from your cluster's authorization plugin.

To confirm the HTTP API is reachable, run:

```bash
curl http://localhost:7243/api/v1/namespaces?pageSize=1
```

A JSON response (even `{"namespaces":[]}`) confirms the HTTP API is up.

## Setup

### Self-hosted Temporal Server

Ensure the HTTP API frontend is enabled in your `temporal.yaml` config (or equivalent):

```yaml
services:
  frontend:
    rpc:
      httpPort: 7243
```

Restart the server if you changed this setting.

### Add the Source

```bash
coral source add --file sources/community/temporal/manifest.yaml
```

When prompted, provide:

- `TEMPORAL_ADDRESS`: Base URL of the Temporal HTTP API. Do not include a trailing slash.
  - Examples: `http://localhost:7243`, `http://temporal.internal:7243`, `https://temporal.mycompany.com:7243`
- `TEMPORAL_API_KEY` *(optional)*: Bearer token. Leave blank for open (unauthenticated) clusters.

### Verify Setup

```bash
coral sql "SELECT name, state FROM temporal.namespaces LIMIT 5"
```

If the source is configured correctly, the query returns registered namespaces.

## Tables

### `namespaces`

All namespaces registered on the Temporal cluster. Use the `name` column as the required `namespace` filter when querying `workflows` and `schedules`.

Useful for namespace discovery, retention policy review, cluster topology inspection, and finding namespace names.

Columns include: `name`, `state`, `description`, `owner_email`, `workflow_execution_retention_ttl`, `active_cluster_name`, `is_global_namespace`.

### `workflows`

Workflow executions in a Temporal namespace. Each row is one execution instance — running or closed. The `namespace` filter is required.

Optionally narrow results with a [Temporal Visibility query](https://docs.temporal.io/visibility) using the `query` filter. Visibility queries support filtering by `ExecutionStatus`, `WorkflowType`, `TaskQueue`, custom search attributes, and more.

Useful for workflow monitoring, failure analysis, latency inspection, task queue auditing, and finding specific executions.

Columns include: `namespace` (virtual), `workflow_id`, `run_id`, `workflow_type`, `task_queue`, `status`, `start_time`, `close_time`, `execution_time`, `history_length`, `history_size_bytes`, `parent_workflow_id`, `parent_run_id`, `search_attributes`, `memo`.

### `schedules`

Schedules in a Temporal namespace. Each row is one schedule that triggers workflow executions on a cron or interval basis. The `namespace` filter is required.

Useful for schedule inventory, cron expression review, next-trigger inspection, and identifying paused or broken schedules.

Columns include: `namespace` (virtual), `schedule_id`, `workflow_type`, `paused`, `notes`, `spec`, `recent_actions`, `future_action_times`.

## Authentication

When `TEMPORAL_API_KEY` is set, the source sends it as a bearer token:

```text
Authorization: Bearer <TEMPORAL_API_KEY>
```

When `TEMPORAL_API_KEY` is left unset (open clusters), the header is sent with an empty value. Temporal Server accepts requests without a valid token on unauthenticated clusters.

## Example Queries

### Namespace inventory

```sql
SELECT name, state, active_cluster_name, workflow_execution_retention_ttl
FROM temporal.namespaces
ORDER BY name
```

### Running workflows by type

```sql
SELECT workflow_type, COUNT(*) AS running_count
FROM temporal.workflows
WHERE namespace = 'default'
  AND query = 'ExecutionStatus="Running"'
GROUP BY workflow_type
ORDER BY running_count DESC
```

### Recently failed workflows

```sql
SELECT workflow_id, run_id, workflow_type, task_queue,
       start_time, close_time
FROM temporal.workflows
WHERE namespace = 'default'
  AND query = 'ExecutionStatus="Failed"'
ORDER BY close_time DESC
LIMIT 20
```

### Long-running workflows (by history size)

```sql
SELECT workflow_id, workflow_type, status, history_length, history_size_bytes,
       start_time
FROM temporal.workflows
WHERE namespace = 'default'
ORDER BY history_size_bytes DESC NULLS LAST
LIMIT 10
```

### Child workflows

```sql
SELECT workflow_id, workflow_type, status, parent_workflow_id, start_time
FROM temporal.workflows
WHERE namespace = 'default'
  AND parent_workflow_id IS NOT NULL
ORDER BY start_time DESC
LIMIT 20
```

### Schedule overview

```sql
SELECT schedule_id, workflow_type, paused, notes
FROM temporal.schedules
WHERE namespace = 'default'
ORDER BY schedule_id
```

### Schedules with upcoming trigger times

```sql
SELECT schedule_id, future_action_times
FROM temporal.schedules
WHERE namespace = 'default'
  AND future_action_times IS NOT NULL
ORDER BY schedule_id
```

### Paused schedules

```sql
SELECT schedule_id, workflow_type, notes
FROM temporal.schedules
WHERE namespace = 'default'
  AND paused = true
ORDER BY schedule_id
```

### Schedules by workflow type

```sql
SELECT workflow_type, COUNT(*) AS schedule_count
FROM temporal.schedules
WHERE namespace = 'default'
GROUP BY workflow_type
ORDER BY schedule_count DESC
```

### Workflow count by task queue

```sql
SELECT task_queue, status, COUNT(*) AS execution_count
FROM temporal.workflows
WHERE namespace = 'default'
GROUP BY task_queue, status
ORDER BY task_queue, status
```

## Verification

Add the source and verify it works against a running Temporal Server:

```bash
$ TEMPORAL_ADDRESS=http://localhost:7243 TEMPORAL_API_KEY="" coral source add --file sources/community/temporal/manifest.yaml
Added source temporal (secrets: keychain)

  ✓ temporal connected successfully
  Secrets: keychain

    temporal (10 tables)
    ├─ activities
    ├─ archived_workflows
    ├─ batch_operations
    ├─ namespaces
    ├─ nexus_endpoints
    ├─ nexus_operations
    ├─ schedules
    ├─ worker_deployments
    ├─ workflow_rules
    └─ ... and 1 more
```

### Query Namespaces

```bash
$ coral sql "SELECT name, state FROM temporal.namespaces LIMIT 5"
+-----------------+----------------------------+
| name            | state                      |
+-----------------+----------------------------+
| default         | NAMESPACE_STATE_REGISTERED |
| temporal-system | NAMESPACE_STATE_REGISTERED |
+-----------------+----------------------------+
```

### Query Workflows

```bash
$ coral sql "SELECT workflow_id, workflow_type, task_queue, status FROM temporal.workflows WHERE namespace = 'default' LIMIT 3"
+---------------------------------------------+-----------------------+------------+-------------------------------------+
| workflow_id                                 | workflow_type         | task_queue | status                              |
+---------------------------------------------+-----------------------+------------+-------------------------------------+
| schedule_workflow_...-2026-05-30T08:26:00Z | SampleScheduleWorkflow | schedule   | WORKFLOW_EXECUTION_STATUS_COMPLETED |
| schedule_workflow_...-2026-05-30T08:25:55Z | SampleScheduleWorkflow | schedule   | WORKFLOW_EXECUTION_STATUS_COMPLETED |
| schedule_workflow_...-2026-05-30T08:25:50Z | SampleScheduleWorkflow | schedule   | WORKFLOW_EXECUTION_STATUS_COMPLETED |
+---------------------------------------------+-----------------------+------------+-------------------------------------+
```

### Query Schedules

```bash
$ coral sql "SELECT schedule_id, workflow_type, paused, notes FROM temporal.schedules WHERE namespace = 'default' LIMIT 5"
+-------------+---------------+--------+-------+
| schedule_id | workflow_type | paused | notes |
+-------------+---------------+--------+-------+
+-------------+---------------+--------+-------+
(No schedules in this cluster)
```

## Limits

- Requires Temporal Server v1.24+ with the HTTP frontend (`httpPort`) enabled. Earlier versions do not expose the `/api/v1` HTTP gateway.
- The `workflows` and `schedules` tables require the `namespace` filter. Omitting it causes a query error; use the `namespaces` table to discover available namespaces first.
- Workflow payload contents (input, output, activity results) are not included in the `raw` column by default — the list API returns execution metadata only, not full history. Use the Temporal SDK or CLI to retrieve workflow histories.
- The `query` filter uses [Temporal Visibility query language](https://docs.temporal.io/visibility); it requires an Elasticsearch or SQL visibility store to be configured on the server. Basic visibility (without Elasticsearch) may return errors for complex query expressions.
- Write operations (start workflow, signal, terminate, create schedule) are intentionally excluded.
