# ALAS WebUI API Document

## Overview

The ALAS WebUI provides REST APIs to manage and monitor ALAS (AzurLaneAutoScript) instances.

- **Base URL**: `http://<host>:<port>`
- **Content-Type**: `application/json`
- **API Version**: `v1` (prefixed as `/api/v1/`)

---

## Design Principles

- **RESTful**: Resources as nouns in URLs, HTTP methods as verbs
- **Consistent envelope**: All responses follow `{"ok": true/false, "data": {...}}` on success or `{"ok": false, "error": "..."}` on failure
- **Unified state model**: Same state strings across all endpoints
- **Versioned**: `/api/v1/` prefix for forward compatibility
- **Proper HTTP codes**: `200` success, `400` bad request, `404` not found, `500` internal error

---

## Response Envelope

### Success (`200`)

```json
{
  "ok": true,
  "data": { ... }
}
```

### Error (`4xx` / `5xx`)

```json
{
  "ok": false,
  "error": "human-readable message"
}
```

---

## Instance State Reference

All endpoints use the following unified state mapping:

| Integer | String      | Description                          |
|---------|-------------|--------------------------------------|
| 0       | `inactive`  | No active page / hidden              |
| 1       | `running`   | Process is alive and running tasks   |
| 2       | `stopped`   | Process is not running (clean exit)  |
| 3       | `warning`   | Process stopped unexpectedly         |
| 4       | `updating`  | Stopped for an update                |

---

## Endpoints

### 1. List Instances

```
GET /api/v1/instances
```

Returns all configured ALAS instances with their current state.

#### Response

```json
{
  "ok": true,
  "data": {
    "instances": [
      {
        "name": "alas",
        "state": "running",
        "alive": true
      },
      {
        "name": "alas2",
        "state": "stopped",
        "alive": false
      }
    ],
    "total": 2
  }
}
```

| Field               | Type    | Description                                    |
|---------------------|---------|------------------------------------------------|
| `instances`         | array   | List of instance objects                       |
| `instances[].name`  | string  | Instance config name                           |
| `instances[].state` | string  | Current state (see [state reference](#instance-state-reference)) |
| `instances[].alive` | boolean | Whether the underlying OS process is alive     |
| `total`             | integer | Total number of instances                      |

---

### 2. Get Instance Status

```
GET /api/v1/instances/{name}
```

Returns detailed status for a single instance, including a task queue summary.

#### Path Parameters

| Parameter | Type   | Required | Description                   |
|-----------|--------|----------|-------------------------------|
| `name`    | string | Yes      | Instance name (e.g. `alas`)   |

#### Response

```json
{
  "ok": true,
  "data": {
    "name": "alas",
    "state": "running",
    "alive": true,
    "task_summary": {
      "running": 1,
      "pending": 2,
      "waiting": 5
    }
  }
}
```

| Field                  | Type    | Description                              |
|------------------------|---------|------------------------------------------|
| `name`                 | string  | Instance name                            |
| `state`                | string  | Current state                            |
| `alive`                | boolean | OS process alive flag                    |
| `task_summary.running` | integer | Number of tasks currently executing      |
| `task_summary.pending` | integer | Number of tasks queued for immediate run |
| `task_summary.waiting` | integer | Number of tasks waiting for schedule     |

#### Error

| Status | Body                                              |
|--------|---------------------------------------------------|
| 400    | `{"ok": false, "error": "missing instance name"}` |
| 404    | `{"ok": false, "error": "instance 'foo' not found"}` |

---

### 3. Start Instance

```
POST /api/v1/instances/{name}/start
```

Starts the scheduler for the given instance.

#### Path Parameters

| Parameter | Type   | Required | Description                 |
|-----------|--------|----------|-----------------------------|
| `name`    | string | Yes      | Instance name (e.g. `alas`) |

#### Request Body

None required (empty body accepted).

#### Response

```json
{
  "ok": true,
  "data": {
    "name": "alas",
    "state": "running"
  }
}
```

| Field   | Type   | Description   |
|---------|--------|---------------|
| `name`  | string | Instance name |
| `state` | string | Current state |

#### Error

| Status | Body                                              |
|--------|---------------------------------------------------|
| 400    | `{"ok": false, "error": "missing instance name"}` |
| 404    | `{"ok": false, "error": "instance 'foo' not found"}` |

---

### 4. Stop Instance

```
POST /api/v1/instances/{name}/stop
```

Stops the scheduler for the given instance.

#### Path Parameters

| Parameter | Type   | Required | Description                 |
|-----------|--------|----------|-----------------------------|
| `name`    | string | Yes      | Instance name (e.g. `alas`) |

#### Request Body

None required (empty body accepted).

#### Response

```json
{
  "ok": true,
  "data": {
    "name": "alas",
    "state": "stopped"
  }
}
```

| Field   | Type   | Description   |
|---------|--------|---------------|
| `name`  | string | Instance name |
| `state` | string | Current state |

#### Error

| Status | Body                                              |
|--------|---------------------------------------------------|
| 400    | `{"ok": false, "error": "missing instance name"}` |
| 404    | `{"ok": false, "error": "instance 'foo' not found"}` |

---

### 5. Get Instance Tasks

```
GET /api/v1/instances/{name}/tasks
```

Returns the full task queue (running, pending, waiting) for an instance with per-task detail.

#### Path Parameters

| Parameter | Type   | Required | Description                 |
|-----------|--------|----------|-----------------------------|
| `name`    | string | Yes      | Instance name (e.g. `alas`) |

#### Response

```json
{
  "ok": true,
  "data": {
    "name": "alas",
    "tasks": {
      "running": [
        {
          "command": "GemsFarming",
          "enabled": true,
          "next_run": "2024-01-15T08:00:00"
        }
      ],
      "pending": [
        {
          "command": "Main",
          "enabled": true,
          "next_run": "2024-01-15T09:00:00"
        }
      ],
      "waiting": [
        {
          "command": "Daily",
          "enabled": true,
          "next_run": "2024-01-16T00:00:00"
        }
      ]
    }
  }
}
```

| Field              | Type    | Description                                     |
|--------------------|---------|-------------------------------------------------|
| `name`             | string  | Instance name                                   |
| `tasks.running`    | array   | Currently executing tasks (0-1 entries)         |
| `tasks.pending`    | array   | Tasks queued for immediate execution            |
| `tasks.waiting`    | array   | Tasks waiting for their scheduled time          |

**Task object:**

| Field     | Type    | Description                                      |
|-----------|---------|--------------------------------------------------|
| `command`  | string  | Task command identifier (e.g. `"GemsFarming"`)  |
| `enabled`  | boolean | Whether the task is enabled in config            |
| `next_run` | string \| null | ISO 8601 timestamp of next scheduled run, or `null` |

#### Error

| Status | Body                                              |
|--------|---------------------------------------------------|
| 400    | `{"ok": false, "error": "missing instance name"}` |
| 404    | `{"ok": false, "error": "instance 'foo' not found"}` |

---

### 6. Get Instance Stats (Game Resources)

```
GET /api/v1/instances/{name}/stats
```

Returns the current in-game resource values (oil, gold coins, gems, event points) with timestamps indicating when each value was last read via OCR.

> **Note**: Resource data is only available while the instance is running a task that reads the relevant game screen. Each resource reports its own availability and timestamp independently — for example, gems are read during shop visits while oil/coins are read during campaign missions. Refer to `available` and `updated_at` fields to judge data freshness.

#### Path Parameters

| Parameter | Type   | Required | Description                 |
|-----------|--------|----------|-----------------------------|
| `name`    | string | Yes      | Instance name (e.g. `alas`) |

#### Response

```json
{
  "ok": true,
  "data": {
    "name": "alas",
    "oil": {
      "value": 5832,
      "available": true,
      "updated_at": "2024-01-15T08:30:45"
    },
    "coin": {
      "value": 14250,
      "available": true,
      "updated_at": "2024-01-15T08:30:43"
    },
    "gem": {
      "value": 150,
      "available": true,
      "updated_at": "2024-01-15T07:55:12"
    },
    "event_pt": {
      "value": 5000,
      "available": true,
      "updated_at": "2024-01-15T08:30:48"
    }
  }
}
```

When a resource has not been read yet:

```json
{
  "ok": true,
  "data": {
    "name": "alas",
    "oil": {
      "value": null,
      "available": false,
      "updated_at": null
    },
    "..."
  }
}
```

#### Resource Object

| Field       | Type                  | Description                                       |
|-------------|-----------------------|---------------------------------------------------|
| `value`     | integer \| null       | Current resource amount, or `null` if never read  |
| `available` | boolean               | Whether the resource has been read at least once  |
| `updated_at`| string \| null        | ISO 8601 timestamp of last OCR reading, or `null` |

#### Resources

| Key        | OCR Source                                |
|------------|--------------------------------------------|
| `oil`      | Campaign screen fuel indicator             |
| `coin`     | Campaign screen gold coin display          |
| `gem`      | Shop screen gem counter                    |
| `event_pt` | Campaign event point tracker               |

#### Error

| Status | Body                                              |
|--------|---------------------------------------------------|
| 400    | `{"ok": false, "error": "missing instance name"}` |
| 404    | `{"ok": false, "error": "instance 'foo' not found"}` |

---

### 7. Get Task Enable Status

```
GET /api/v1/instances/{name}/tasks/enabled
```

Returns the enable/disable status of one or more tasks for an instance. Reads directly from the instance's config file.

#### Path Parameters

| Parameter | Type   | Required | Description                 |
|-----------|--------|----------|-----------------------------|
| `name`    | string | Yes      | Instance name (e.g. `alas`) |

#### Query Parameters

| Parameter | Type   | Required | Description                                           |
|-----------|--------|----------|-------------------------------------------------------|
| `tasks`   | string | No       | Comma-separated task names to filter. If omitted, all tasks are returned. |

#### Response

```json
{
  "ok": true,
  "data": {
    "name": "alas",
    "tasks": {
      "Main": true,
      "Main2": false,
      "Main3": false,
      "GemsFarming": true,
      "Event": false,
      "Event2": false,
      "EventA": true,
      "EventB": false,
      "EventC": true,
      "EventD": false,
      "EventSp": false
    }
  }
}
```

With filter: `GET /api/v1/instances/alas/tasks/enabled?tasks=Main,GemsFarming,EventA`

```json
{
  "ok": true,
  "data": {
    "name": "alas",
    "tasks": {
      "Main": true,
      "GemsFarming": true,
      "EventA": true
    }
  }
}
```

| Field        | Type               | Description                           |
|--------------|--------------------|---------------------------------------|
| `name`       | string             | Instance name                         |
| `tasks`      | object             | Map of task name to enable status     |
| `tasks.{key}`| boolean            | `true` if enabled, `false` if disabled|

#### Error

| Status | Body                                              |
|--------|---------------------------------------------------|
| 400    | `{"ok": false, "error": "missing instance name"}` |
| 404    | `{"ok": false, "error": "instance 'foo' not found"}` |

#### Key Task Names

| Task           | Description          |
|----------------|----------------------|
| `Main`         | Main campaign 1      |
| `Main2`        | Main campaign 2      |
| `Main3`        | Main campaign 3      |
| `GemsFarming`  | Gems farming (1-1)   |
| `Event`        | Current event maps   |
| `Event2`       | Second event maps    |
| `EventA`       | Event daily map A    |
| `EventB`       | Event daily map B    |
| `EventC`       | Event daily map C    |
| `EventD`       | Event daily map D    |
| `EventSp`      | Event SP map         |

---

## Web Pages

| Path      | Description                                          |
|-----------|------------------------------------------------------|
| `GET /`   | Main ALAS GUI — scheduler dashboard, task config, logs |
| `GET /manage` | Configuration management — import/export/create instances |

Both pages require password authentication when a password is configured.

---

## CLI Arguments

```
$ python module/webui/app.py [options]
```

| Argument    | Description                                              |
|-------------|----------------------------------------------------------|
| `-k, --key` | Password for the web interface. No password by default.  |
| `--cdn`     | Use jsdelivr CDN for PyWebIO static files. Self-host by default. |
| `--run`     | Run ALAS instances by config name(s) on startup.         |

---

## Authentication

If a password is configured (via `-k` or config), a login prompt appears before accessing any page. Failed logins trigger a 1.5-second delay before page reload.
