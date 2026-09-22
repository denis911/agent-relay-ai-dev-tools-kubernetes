# Agent Relay

Agent Relay is a lightweight FastAPI service for registering agents, delivering one task at a time, and recording results. PostgreSQL persists the task queue, agent directory, and delivery attempts, while workers execute tasks on their own machines. The included worker deterministically returns `input.upper()`.

---

## Architecture & How Agents Interact

Agent Relay acts as a decoupled message and task broker. Agents do not execute inside the relay; registration creates an agent identity and inbox.

```text
  +------------------+         +----------------------------+         +-------------------+
  |  Alice (Sender)  |         |   Agent Relay (FastAPI)    |         | Bob's Worker (Rx) |
  |                  |         |       + PostgreSQL         |         |                   |
  +--------+---------+         +--------------+-------------+         +---------+---------+
           |                                  |                                 |
           |  1. Register Agent Identities    |                                 |
           |---- POST /api/v1/agents -------->|                                 |
           |<--- 201 Created (Token & ID) ----|                                 |
           |                                  |<--- POST /api/v1/agents --------|
           |                                  |---- 201 Created (Token & ID) -->|
           |                                  |                                 |
           |  2. Dispatch Task (Bearer Alice) |                                 |
           |---- POST /api/v1/tasks --------->|                                 |
           |<--- 201 Created ("status":queued)|                                 |
           |                                  |                                 |
           |                                  |  3. Claim Work (Bearer Bob)     |
           |                                  |<--- POST /api/v1/tasks/claim ---|
           |                                  |---- 200 OK (Task & Claim Token)->
           |                                  |                                 |
           |                                  |  [ Worker runs input.upper() ]  |
           |                                  |                                 |
           |                                  |  4. Submit Result               |
           |                                  |<--- POST .../tasks/{id}/complete|
           |                                  |---- 200 OK ("status":completed)->
           |                                  |                                 |
           |  5. Query Result (Bearer Alice)  |                                 |
           |---- GET /api/v1/tasks/{id} ----->|                                 |
           |<--- 200 OK (output: "HELLO") ----|                                 |
           v                                  v                                 v
```

### Task Lifecycle States
1. **`queued`**: The sender submitted the task. It sits safely in PostgreSQL awaiting a claim by a recipient worker.
2. **`processing`**: A worker serving the recipient claimed the task and holds an active lease (default: 60s). Long tasks extend the lease with periodic heartbeats.
3. **`completed`**: The worker submitted output with its claim token.
4. **`failed`**: The worker explicitly failed the task, or lease retries were exhausted (`attempts_exhausted`).

---

## Starting the Service

### With Docker Compose
To build and start both Agent Relay and PostgreSQL:
```bash
docker compose up --build -d
```

> **Tip - Updating Code & Resetting Data:**
> - To reload new code changes: `docker compose up --build -d` automatically rebuilds the app container image.
> - To wipe data and start completely fresh: `docker compose down -v` clears the PostgreSQL data volume.

### With Kubernetes (KinD)
Manifests are provided in `k8s/` including persistent storage (`postgres-pvc`), database credentials (`postgres-secret`), deployments, and services.

1. **Create local cluster and load image**:
   ```bash
   kind create cluster --name agent-relay
   docker build -t agent-relay:local .
   kind load docker-image agent-relay:local --name agent-relay
   ```

2. **Deploy to Kubernetes**:
   ```bash
   kubectl apply -f k8s/postgres.yaml
   kubectl apply -f k8s/agent-relay.yaml
   ```

3. **Verify pods are ready**:
   ```bash
   kubectl wait --for=condition=ready pod -l app=postgres --timeout=60s
   kubectl wait --for=condition=ready pod -l app=agent-relay --timeout=60s
   ```

4. **Port forward to test locally**:
   ```bash
   kubectl port-forward svc/agent-relay 8000:8000
   ```
   Now access the API and Dashboard at `http://127.0.0.1:8000/`.

5. **Teardown & Cleanup**:
   To delete the KinD cluster and all associated resources when finished:
   ```bash
   kind delete cluster --name agent-relay
   docker rmi agent-relay:local
   ```

### Locally with uv
```bash
uv sync
# Ensure PostgreSQL is running or set RELAY_DATABASE_URL
uv run uvicorn main:app --reload
```

- **Liveness check**: `GET http://127.0.0.1:8000/health`
- **Readiness check**: `GET http://127.0.0.1:8000/ready` (queries real DB tables)
- **Web Dashboard**: <http://127.0.0.1:8000/>

---

## Step-by-Step Manual Testing

### Option A: Bash / macOS / Linux / Git Bash

#### 1. Register Alice and Bob
```bash
alice=$(curl -sS -X POST http://127.0.0.1:8000/api/v1/agents \
  -H 'content-type: application/json' -d '{"name":"alice"}')
bob=$(curl -sS -X POST http://127.0.0.1:8000/api/v1/agents \
  -H 'content-type: application/json' -d '{"name":"uppercase"}')

# Extract tokens and agent IDs:
alice_token=$(echo "$alice" | python -c "import sys, json; print(json.load(sys.stdin)['token'])")
bob_token=$(echo "$bob" | python -c "import sys, json; print(json.load(sys.stdin)['token'])")
bob_agent_id=$(echo "$bob" | python -c "import sys, json; print(json.load(sys.stdin)['agent_id'])")

echo "Alice token: $alice_token"
echo "Bob token:   $bob_token"
echo "Bob ID:      $bob_agent_id"
```

#### 2. Alice sends a task to Bob
```bash
task=$(curl -sS -X POST http://127.0.0.1:8000/api/v1/tasks \
  -H "Authorization: Bearer $alice_token" \
  -H 'content-type: application/json' \
  -d "{\"to\":\"$bob_agent_id\",\"input\":\"hello uppercase\"}")

task_id=$(echo "$task" | python -c "import sys, json; print(json.load(sys.stdin)['task_id'])")
echo "Task ID: $task_id"
```

If you query the task now:
```bash
curl -sS http://127.0.0.1:8000/api/v1/tasks/$task_id \
  -H "Authorization: Bearer $alice_token"
```
It shows `"status": "queued"`.

#### 3. Start Bob's worker to process the task
In a separate terminal (or background job):
```bash
uv run python main.py worker \
  --base-url http://127.0.0.1:8000 \
  --agent-id "$bob_agent_id" \
  --token "$bob_token" \
  --worker-id bob-worker-1
```
The worker claims the task, computes `input.upper()`, and marks it completed.

#### 4. Check the completed result
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

---

### Option B: Windows PowerShell

#### 1. Register Alice and Bob
```powershell
$alice = Invoke-RestMethod -Uri "http://127.0.0.1:8000/api/v1/agents" -Method Post -ContentType "application/json" -Body '{"name":"alice"}'
$bob = Invoke-RestMethod -Uri "http://127.0.0.1:8000/api/v1/agents" -Method Post -ContentType "application/json" -Body '{"name":"uppercase"}'

$alice_token = $alice.token
$bob_token = $bob.token
$bob_agent_id = $bob.agent_id

Write-Host "Alice token: $alice_token"
Write-Host "Bob token:   $bob_token"
Write-Host "Bob ID:      $bob_agent_id"
```

#### 2. Alice sends a task to Bob
```powershell
$headers = @{ "Authorization" = "Bearer $alice_token" }
$body = @{ to = $bob_agent_id; input = "hello uppercase from powershell" } | ConvertTo-Json

$task = Invoke-RestMethod -Uri "http://127.0.0.1:8000/api/v1/tasks" -Method Post -Headers $headers -ContentType "application/json" -Body $body
$task_id = $task.task_id
Write-Host "Task ID: $task_id"
```

#### 3. Start Bob's worker
```powershell
uv run python main.py worker `
  --base-url http://127.0.0.1:8000 `
  --agent-id "$bob_agent_id" `
  --token "$bob_token" `
  --worker-id bob-worker-1
```

#### 4. Retrieve result
```powershell
$result = Invoke-RestMethod -Uri "http://127.0.0.1:8000/api/v1/tasks/$task_id" -Headers $headers
$result | Format-List *
```

---

## Real-Time Inspection via Web Dashboard

Open <http://127.0.0.1:8000/> in your browser:
1. Paste either `$alice_token` or `$bob_token` into the Token prompt.
2. The dashboard displays:
   - Registered agent directory and last-seen timestamps.
   - Live task status (`queued`, `processing`, `completed`, `failed`).
   - Delivery attempts, worker IDs, and task inputs/outputs.

---

## Worker Options & Resilience Testing

### Self-registration with credentials file
Instead of passing tokens manually on the CLI, the worker can self-register once and persist credentials to disk:
```bash
uv run python main.py worker \
  --base-url http://127.0.0.1:8000 \
  --name uppercase \
  --credentials ./bob-credentials.json \
  --worker-id laptop-1
```
On subsequent runs, only `--credentials ./bob-credentials.json` is needed.

### Simulating slow execution and lease timeouts
```bash
uv run python main.py worker \
  --credentials ./bob-credentials.json \
  --slow-seconds 75 \
  --worker-id slow-laptop
```
- During slow work, the worker automatically sends periodic heartbeats to renew its 60-second lease.
- If you terminate the worker process midway, the lease expires after 60 seconds; the relay automatically requeues the task for another worker to pick up with an incremented attempt count.

---

## Storage & Delivery Guarantees

- **PostgreSQL Concurrency**: Task claims leverage PostgreSQL's `FOR UPDATE SKIP LOCKED`. Multiple workers polling concurrently receive distinct tasks without blocking each other or causing deadlocks.
- **At-least-once Delivery**: Tasks are leased to workers. If a worker fails or loses connectivity without heartbeating, the lease expires and the task is safely redelivered.
- **Idempotent Retries**: Terminating requests (`complete` or `fail`) require the unique claim token issued during claim. Repeating the same result is idempotent; conflicting terminal results receive `409 Conflict`.
- **Task Submission Idempotency**: Senders can supply an `Idempotency-Key` header on `POST /tasks` to ensure network retries do not duplicate tasks.

---

## Running Tests

Run the test suite using `uv`:
```bash
uv run pytest -q
```
The test suite runs against an isolated scratch database, verifying idempotency, token hashing, sender/recipient isolation, claim token lifecycle, and concurrent race conditions.
