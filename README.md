# Mini Distributed Cloud Object Store

A small distributed object store: **1 controller + 4 storage workers** in Docker.
Objects are split into **1 MiB chunks**, every chunk is stored on **2 different workers**,
failures and corruption are detected, and lost copies are rebuilt automatically.

* **Why was everything built this way?** → [`DECISIONS.md`](DECISIONS.md) (every design choice with reasoning)
* **Where is the code?** Every file is heavily commented; start with the map below.

---

## 1. Quick start

Requirements: Docker (with the `docker compose` plugin), Python 3.10+, Linux/macOS shell.

```bash
# 1) Start the cluster (builds 2 images, starts 1 controller + 4 workers)
docker compose up --build -d
docker compose ps                       # all 5 should become "healthy"

# 2) Create the virtual environment for the client, demo and tests
./setup.sh
source .venv/bin/activate

# 3) Use it
python client/objstore_cli.py upload ~/some-file.jpg --meta author=alice
python client/objstore_cli.py list
python client/objstore_cli.py cluster
```

Interactive API documentation (Swagger UI): <http://localhost:8000/docs>
Port 8000 busy? `CONTROLLER_PORT=9000 docker compose up -d` and use `--url http://localhost:9000`.

## 2. Run the required demonstration

```bash
source .venv/bin/activate
python demo/run_demo.py            # all 6 demos, prints evidence + PASS/FAIL for every check
python demo/run_demo.py --pause    # waits for ENTER between steps (good for presenting/recording)
python demo/run_demo.py --only 5 6 # just the failure and integrity demos
```

| Spec item | Demo step | What you will see |
|---|---|---|
| Object Storage | 1 | upload, download, SHA-256 of original == retrieved |
| Chunking & Distribution | 2 | 5.5 MiB object → 5×1 MiB + 0.5 MiB chunks, table of which worker has what |
| Replication | 3 | 2 copies per chunk on different workers, **verified on the worker disks** with `docker exec sha256sum` |
| Metadata | 4 | upload with custom metadata, read it, `PATCH` it, proof nothing was re-uploaded |
| Worker Failure | 5 | `docker stop worker2` → detected → still downloadable → auto re-replication → `docker start worker2` → back to exactly 2 copies; cluster state printed **before / during / after** |
| Data Integrity | 6 | a chunk file is deliberately corrupted → detected → good copy served → bad copy replaced |

The script exits 0 only if every check passed.

### Doing the demos by hand

```bash
CLI="python client/objstore_cli.py"
head -c 5500000 /dev/urandom > big.bin
$CLI upload big.bin --meta owner=me          # prints the object id and the chunk table
$CLI info <object_id>                        # chunk locations (which workers)
$CLI download <object_id> -o copy.bin        # prints MATCH / MISMATCH
cmp big.bin copy.bin && echo identical

# --- worker failure ---
$CLI cluster                                 # before
docker stop worker2
$CLI cluster                                 # after ~6 s: worker2 DEAD, under-replicated > 0
$CLI download <object_id> -o copy2.bin       # still works
sleep 20; $CLI cluster                       # re-replicated onto the 3 remaining workers
docker start worker2
sleep 5;  $CLI cluster                       # worker2 ALIVE, exactly 2 copies per chunk again
$CLI events                                  # the controller's log of what it decided

# --- corruption ---
$CLI info <object_id>                         # note a worker and chunk_id
scripts/corrupt_chunk.sh worker3 <chunk_id>  # flips bytes inside the container
$CLI download <object_id> -o copy3.bin       # still correct (bad copy is skipped)
$CLI scrub                                   # force the integrity scan
$CLI events                                  # corruption_detected ... repair
```

## 3. Run the automated tests (no Docker needed)

```bash
source .venv/bin/activate
python -m pytest tests -q          # 40 tests, ~30 s
```

The tests start a real controller and 4 real workers as local processes, so they exercise
the real HTTP behaviour (heartbeats, failover, repair, corruption). They also run the real
demo script end to end. See D-18 in `DECISIONS.md`.

## 4. Cleaning up when the project is over

```bash
./cleanup.sh          # asks first; removes containers, network, volumes (ALL DATA),
                      # the built images, .venv, caches
./cleanup.sh -y --base-image   # also removes the shared python:3.12-slim image
```
Then delete this folder to remove the source. Nothing is installed outside `.venv` and Docker
(`setup.sh` disables pip's cache, which would otherwise live in `~/.cache/pip`).
Stop without deleting data: `docker compose down` (data stays in the volumes).

---

## 5. Architecture

```
                         host
  client CLI / curl ──────────────┐
  (runs in .venv)                 │ :8000  (only published port)
 ─────────────────────────────────▼──────────────── Docker network "objstore"
                       ┌─────────────────────┐
                       │     controller      │  REST API, metadata (SQLite on a volume),
                       │  FastAPI + SQLite   │  placement, failure detection, repair, scrub
                       └─────────┬───────────┘
          register/heartbeat ▲   │ PUT/GET/DELETE chunks (internal API)
        ┌───────────┬────────┴───┴───┬───────────┐
   ┌────▼───┐  ┌────▼───┐       ┌────▼───┐  ┌────▼───┐
   │worker1 │  │worker2 │       │worker3 │  │worker4 │   each: FastAPI + its own
   │ /data  │  │ /data  │       │ /data  │  │ /data  │   persistent volume of chunk files
   └────────┘  └────────┘       └────────┘  └────────┘
```

**Upload:** client → controller → split into 1 MiB chunks → each chunk SHA-256'd and written to the
2 least-loaded workers in parallel → object becomes visible only when all chunks are stored.
**Download:** controller reads each chunk from any live copy, verifies its checksum, streams it on.
**Self-healing:** heartbeats detect dead workers; a repair loop restores 2 copies; a scrub
loop finds corruption; rejoining workers are reconciled with the database first.

### Code map

| Path | Purpose |
|---|---|
| `controller/app/main.py` | app wiring, start/stop of background loops |
| `controller/app/routes_client.py` | the 6 client REST endpoints (spec §5) |
| `controller/app/routes_internal.py` | worker register/heartbeat + admin endpoints |
| `controller/app/cluster.py` | placement, store/fetch chunk, bad-copy handling, replication report |
| `controller/app/maintenance.py` | failure monitor, reconcile, repair, scrub loops |
| `controller/app/db.py` | every SQL statement; schema |
| `controller/app/worker_client.py` | controller → worker HTTP calls |
| `controller/app/errors.py`, `config.py` | uniform JSON errors; env-driven settings |
| `worker/app/main.py` | worker internal API + heartbeat loop |
| `worker/app/store.py` | atomic, checksum-verified chunk files on disk |
| `client/objstore_cli.py` | command-line client |
| `demo/run_demo.py` | the required demonstration |
| `tests/` | integration tests |
| `docker-compose.yml`, `*/Dockerfile` | deployment |
| `setup.sh`, `cleanup.sh`, `scripts/` | venv setup, full cleanup, corruption helper |

## 6. REST API summary

Client API (JSON; binary for object data). Errors are always `{"error": "...", "message": "..."}`.

| Method | Endpoint | Success | Notes |
|---|---|---|---|
| POST | `/api/v1/objects` | 201 | multipart: `file` (required), `metadata` (optional JSON string) |
| GET | `/api/v1/objects/{id}` | 200 | binary body; headers `X-Object-SHA256`, `ETag`; 404 if unknown; 503 if no live copy |
| GET | `/api/v1/objects/{id}/metadata` | 200 | system + custom metadata + chunk locations |
| PATCH | `/api/v1/objects/{id}/metadata` | 200 | JSON object merged in; `null` deletes a key |
| DELETE | `/api/v1/objects/{id}` | 204 | |
| GET | `/api/v1/objects?limit=&offset=` | 200 | paginated list |

Other status codes: `400` invalid request · `404` not found · `500` internal error ·
`503` storage unavailable (also: fewer than 2 live workers on upload). `409` is not used (see D-27).

Internal (controller): `POST /internal/workers/register`, `POST /internal/workers/heartbeat`,
`GET /internal/workers/{id}/chunks` (controller's records).
Internal (worker): `PUT|GET|DELETE /internal/chunks/{chunk_id}`, `GET /internal/workers/{id}/chunks` (what is on disk).
Admin (extra): `GET /api/v1/admin/cluster`, `GET /api/v1/admin/events`, `POST /api/v1/admin/scrub`, `POST /api/v1/admin/repair`.

## 7. Configuration

Set in `docker-compose.yml` (controller section). Defaults shown.

| Variable | Default | Meaning |
|---|---|---|
| `CHUNK_SIZE` | 1048576 | chunk size in bytes (spec: 1 MiB) |
| `REPLICATION_FACTOR` | 2 | copies per chunk (spec: 2) |
| `HEARTBEAT_TIMEOUT` | 6 | seconds of silence before a worker is DEAD |
| `REPAIR_GRACE_SECONDS` | 15 | wait before re-replicating a dead worker's data |
| `SCRUB_INTERVAL` | 60 | seconds between integrity scans (0 = off) |

## 8. Troubleshooting

| Symptom | Fix |
|---|---|
| `cannot reach the controller` | `docker compose ps`; `docker compose logs controller` |
| Workers not "healthy" | `docker compose logs worker1`; the controller must be healthy first |
| Upload says `insufficient_workers` | fewer than 2 workers alive: `docker start workerN` |
| Port 8000 in use | `CONTROLLER_PORT=9000 docker compose up -d` |
| Start completely fresh | `docker compose down -v && docker compose up --build -d` |
| Look inside a worker | `docker exec worker2 ls -l /data/chunks` |

## 9. Verification status (please read)

* The controller, workers, client, demo script and tests were **executed and pass** (40 tests) with the real
  services running as local processes.
* Docker itself was **not available** where this was written, so the `Dockerfile`s, `docker-compose.yml` and the
  literal `docker stop/start/exec` commands were validated statically (YAML parsed, scripts syntax-checked) but not run.
  Run `python demo/run_demo.py` once against your Docker cluster to confirm; any problem will be a small
  deployment detail, not a logic issue.
