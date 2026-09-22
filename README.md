# Agent Relay

Agent Relay is a small FastAPI service for registering agents, delivering one
task at a time, and recording results. PostgreSQL persists the queue and attempts,
while workers execute tasks on their own machines. The included worker deterministically
returns `input.upper()`.

## Run it

### With Docker Compose (recommended)
Run Agent Relay and PostgreSQL together:
```bash
docker compose up -d
```

### Locally with uv
```bash
uv sync
# Ensure a PostgreSQL instance is running or set RELAY_DATABASE_URL
uv run uvicorn main:app --reload
```

Open <http://127.0.0.1:8000/> for the token-based local dashboard. The default
database connection is configured for PostgreSQL (`postgresql+psycopg://relay:relaypassword@postgres:5432/relay` in compose, or set `RELAY_DATABASE_URL`).
`GET /health` is a liveness check and `GET /ready` verifies database
connectivity and schema (it queries the real tables, so a wiped volume
reports not-ready instead of passing with zero tables).


### 1. Register two identities

Registration creates an agent identity and inbox. It returns a secret Bearer `token` **once** (keep it secure):

```bash
alice=$(curl -sS -X POST http://127.0.0.1:8000/api/v1/agents \
  -H 'content-type: application/json' -d '{"name":"alice"}')
bob=$(curl -sS -X POST http://127.0.0.1:8000/api/v1/agents \
  -H 'content-type: application/json' -d '{"name":"uppercase"}')

# Extract tokens and agent IDs:
alice_token=$(echo "$alice" | python -c "import sys, json; print(json.load(sys.stdin)['token'])")
bob_token=$(echo "$bob" | python -c "import sys, json; print(json.load(sys.stdin)['token'])")
bob_agent_id=$(echo "$bob" | python -c "import sys, json; print(json.load(sys.stdin)['agent_id'])")
```

Use `Authorization: Bearer <token>` for all subsequent API calls; registration is the only unauthenticated endpoint. For a shared installation, set `RELAY_ENROLLMENT_SECRET` and send it as `X-Enrollment-Secret` when registering.

### 2. Send a task

Alice sends a task to Bob (use Bob's `agent_id` in the `"to"` field):

```bash
task=$(curl -sS -X POST http://127.0.0.1:8000/api/v1/tasks \
  -H "Authorization: Bearer $alice_token" \
  -H 'content-type: application/json' \
  -d "{\"to\":\"$bob_agent_id\",\"input\":\"hello uppercase\"}")

task_id=$(echo "$task" | python -c "import sys, json; print(json.load(sys.stdin)['task_id'])")
```

If you check the task now:
```bash
curl -sS http://127.0.0.1:8000/api/v1/tasks/$task_id \
  -H "Authorization: Bearer $alice_token"
```
It will show `"status": "queued"`. **This is by design**: the relay is an asynchronous queue service. Registering an agent creates an inbox, but does not execute work; tasks wait in the queue until a worker process serving that agent claims them.

### 3. Run Bob's worker to process tasks

In another terminal (or background process), start the deterministic worker using Bob's credentials:

```bash
uv run python main.py worker \
  --base-url http://127.0.0.1:8000 \
  --agent-id "$bob_agent_id" \
  --token "$bob_token" \
  --worker-id bob-worker-1
```

The worker long-polls `POST /api/v1/tasks/claim`, leases the queued task, computes `input.upper()`, and reports completion back to the relay.

### 4. Retrieve the completed result

Once the worker completes the task, Alice can query the task again:

```bash
curl -sS http://127.0.0.1:8000/api/v1/tasks/$task_id \
  -H "Authorization: Bearer $alice_token"
```
Output:
```json
{
  "task_id": "...",
  "from": "agent_...",
  "to": "agent_...",
  "input": "hello uppercase",
  "status": "completed",
  "output": "HELLO UPPERCASE",
  "error": null,
  "attempt_count": 1
}
```

*(On Windows PowerShell, use `Invoke-RestMethod` with hashtable bodies converted via `ConvertTo-Json` to avoid CLI quote stripping).*

## Worker options & failure handling

### Automatic self-registration with credentials file
Instead of passing tokens manually on the CLI, the worker can self-register on its first run and save credentials in a local mode-0600 JSON file:

```bash
uv run python main.py worker \
  --base-url http://127.0.0.1:8000 \
  --name uppercase \
  --credentials ./uppercase-credentials.json \
  --worker-id laptop-1
```

### Simulating slow work and lease timeouts
To demonstrate heartbeating, redelivery, or worker crashes:

```bash
uv run python main.py worker \
  --credentials ./uppercase-credentials.json \
  --slow-seconds 75 \
  --worker-id slow-laptop
```

The worker sends periodic heartbeats to extend its lease during long operations. If the worker process is killed before finishing, its claim expires after the 60-second lease, allowing another worker process to claim the task with an incremented attempt count. `RELAY_LEASE_SECONDS` and `RELAY_MAX_ATTEMPTS` are configurable via environment variables.

## Storage and delivery behavior

`database.py` contains SQLAlchemy models, engine pooling, and transaction helpers.
`storage.py` contains task/claim/recovery operations; routes and request models
are kept in `main.py` and `schemas.py`. Claims and recovery leverage PostgreSQL's
`FOR UPDATE SKIP LOCKED` for high-throughput, non-blocking concurrent claims across
workers.

Claims are at-least-once and leased for 60 seconds by default. Heartbeats extend
an active lease. A completion or failure must include the recipient's bearer
token and claim token. Repeating the exact terminal request with that claim
token is idempotent; a stale token or different result receives `409`.

## Verify

The test suite covers the main protocol, sender/recipient access boundaries,
hashed claim-token behavior, idempotent terminal retries, concurrent claims,
lease expiry before and after recovery, pagination/error shape, and dashboard
asset serving:

```bash
uv run pytest -q
```

Tests run isolated against a scratch database (or `RELAY_DATABASE_URL` if set).
The fixture drops and recreates all tables on whatever `RELAY_DATABASE_URL` points at.
