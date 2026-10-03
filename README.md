# Agent Relay

Agent Relay is a small FastAPI service for registering agents, delivering one
task at a time, and recording results. PostgreSQL persists the queue and
attempts, while workers execute tasks on their own machines. The included
worker deterministically returns `input.upper()`.

## Run it

Start PostgreSQL and the API together with Compose:

```bash
docker compose up --build
```

Open <http://127.0.0.1:8002/> for the token-based dashboard. To run Uvicorn
directly on your machine instead, start the Compose database service first,
then run `uv sync` and `uv run uvicorn main:app --reload`; the default database
URL connects to PostgreSQL at `localhost:5432`. Set `RELAY_DATABASE_URL` to use
a different PostgreSQL instance. `GET /health` is a liveness check and
`GET /ready` verifies database connectivity and schema.

The API examples below use the Compose address at port 8002. If running Uvicorn
directly on your machine, use port 8000 instead.

## Run on local Kubernetes with kind

With a kind cluster named `agent-relay` and the `agent-relay:local` image
available locally, load the image and apply the manifests:

```bash
kind load docker-image agent-relay:local --name agent-relay
kubectl apply -f k8s/
kubectl rollout status deployment/postgres -n agent-relay
kubectl rollout status deployment/agent-relay -n agent-relay
kubectl port-forward --address 127.0.0.1 -n agent-relay service/agent-relay 8002:8000
```

Open <http://127.0.0.1:8002/> while the port-forward command is running. The
PostgreSQL PVC retains data across pod restarts. The manifests use the local
development password `relay-local`; change it before using this setup in a
shared environment.

Register two identities and send a task:

```bash
alice=$(curl -sS -X POST http://127.0.0.1:8002/api/v1/agents \
  -H 'content-type: application/json' -d '{"name":"alice"}')
bob=$(curl -sS -X POST http://127.0.0.1:8002/api/v1/agents \
  -H 'content-type: application/json' -d '{"name":"uppercase"}')
```

The response contains each agent's secret `token` once. Keep it outside source
control. Use `Authorization: Bearer <token>` for all subsequent API calls;
registration is the only unauthenticated endpoint. For a shared installation,
set `RELAY_ENROLLMENT_SECRET` and send it as `X-Enrollment-Secret` when
registering.

## Run the deterministic worker

The worker can register itself and save credentials in a mode-0600 JSON file:

```bash
uv run python main.py worker \
  --base-url http://127.0.0.1:8002 \
  --name uppercase \
  --credentials ./uppercase-credentials.json \
  --worker-id laptop-1
```

For failure/redelivery demonstrations, make local execution intentionally slow
and stop the process after one completion:

```bash
uv run python main.py worker --credentials ./uppercase-credentials.json \
  --slow-seconds 75 --worker-id slow-laptop
```

The worker heartbeats during long work. Killing it leaves the claim leased;
after the 60-second lease expires, another worker can claim the task with a new
token and incremented attempt number. `RELAY_LEASE_SECONDS` and
`RELAY_MAX_ATTEMPTS` are configurable server settings.

An existing credential can also be supplied explicitly (the token is not
written to disk):

```bash
uv run python main.py worker --agent-id agent_123 --token agt_… --worker-id laptop-2
```

## Storage and delivery behavior

`database.py` contains SQLAlchemy models and transaction setup. `storage.py`
contains task/claim/recovery operations; routes and request models are kept in
`main.py` and `schemas.py`. PostgreSQL row locks with `FOR UPDATE SKIP LOCKED`
coordinate concurrent claims across API processes. The protocol and lifecycle
are described in `SPEC.md`.

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

Tests default to a scratch SQLite database at `/tmp/agent-relay-test.db` so
they don't need a running PostgreSQL service. The fixture drops and recreates
all tables on whatever `RELAY_DATABASE_URL` points at, so use a disposable
database URL when running tests against PostgreSQL.

The local CI workflow runs the tests, builds the image, and deploys it to kind.
The starter does not include an external broker or an LLM.

## Run CI locally with `act`

Install [`act`](https://nektosact.com/installation/) and Docker. To run the
workflow's PostgreSQL tests, build the image, load it into the existing
`agent-relay` kind cluster, and deploy it there, run this from Bash:

```bash
act workflow_dispatch \
  -W .github/workflows/ci.yml \
  -e <(printf '%s\n' '{"inputs":{"kind_cluster":"agent-relay"}}') \
  -P ubuntu-latest=catthehacker/ubuntu:act-latest \
  --container-daemon-socket unix:///var/run/docker.sock \
  --container-options "--network host -v $HOME/.kube:/act-kube" \
  --env KUBECONFIG=/act-kube/config
```

The Docker socket lets the workflow build and load its image. The host network
and mounted kubeconfig let `kubectl` and `kind` access the selected local
cluster. This grants the workflow access to Docker and your Kubernetes
credentials, so only run workflow code you trust. For another cluster, change
`kind_cluster` in the event JSON above. The default for GitHub Actions is the
separate `agent-relay-ci` cluster.
