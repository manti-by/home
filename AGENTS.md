# AGENTS.md

Umbrella repository for the home-automation platform. The three services live in
submodules; this file is the source of truth for anything that crosses submodule
boundaries.

## Layout

| Path        | Submodule                                                 | Role                                                                 |
| ----------- | --------------------------------------------------------- | -------------------------------------------------------------------- |
| `odin/`     | `git@github.com:manti-by/odin.git`                        | Central Django 5.2 / DRF server, PostgreSQL, Redis, gunicorn + worker |
| `coruscant/`| `git@github.com:manti-by/coruscant.git`                   | Raspberry Pi satellite: DS18B20 sensors, pumps/valves/servos         |
| `centax/`   | `git@github.com:manti-by/centax.git`                      | ESP-01S firmware: DHT22 temp/humidity, HTTP POST to ODIN             |

Each submodule keeps its own `AGENTS.md` with service-specific rules. **Read the
submodule's `AGENTS.md` first** whenever work is contained to one service; come
back here only for anything that touches two or three.

## Submodule Workflow

```bash
git clone --recurse-submodules git@github.com:<org>/home.git
git submodule update --init --recursive   # first time only
git submodule update --remote <name>      # pull latest from a submodule
```

Inside a submodule you work as usual — it is an independent git repo. Do **not**
edit files inside a submodule from the parent checkout; commit in the submodule
first, then bump the gitlink from the parent (`git add <submodule>`).

Pin a known-good combination of submodule commits in a dedicated branch of this
umbrella repo (e.g. `integrations/2026-Q2`) when shipping a release that
coordinates versions across services.

## Git Workflow (umbrella repo)

Same Git Flow as the submodules:

- `master` is always green; never force-push, never commit directly.
- Feature branches: `<agent-name>/feature/<id>-<descriptive-name>`.
- Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, …).
- One PR per change. PR title follows Conventional Commits.
- Linear issue tracker: `TODO` → `In Progress` → `In Review` → `Done`
  (or back to `In Progress` / `Closed` if not merged).

## Cross-Service Contracts

Anything in this section is a **public contract** shared by two or more services.
Changing it requires coordinated PRs in every affected submodule and a bump of
the umbrella repo's gitlink for each.

### 1. HTTP: sensor data ingest (`POST /api/v1/sensors/logs/`)

- **Producers:** `centax` (ESP-01S, every 5 min), `coruscant` (`sync.py`).
- **Consumer:** `odin` (DRF view in `odin/api/v1/sensors/`).
- **Request body (JSON):**

  ```json
  { "sensor_id": "<string>", "temp": <float>, "humidity": <float> }
  ```

  - `sensor_id` must already exist in `odin` (foreign key to `Sensor`).
  - `temp` is °C, `humidity` is `%RH`. Omit a field the sensor cannot provide.
- **Response:** 200 OK on success; 400 on validation; 404 on unknown `sensor_id`.
- **Versioning rule:** additive fields are non-breaking. Removing/renaming a
  field or changing its type is a breaking change → bump `/api/v2/` and keep
  v1 alive until every producer has moved.

### 2. Redis pub/sub channels

- Connection comes from `REDIS_URL`, default `redis://localhost:6379/0`.
- **ODIN → Coruscant** (relay control), channel `REDIS_RELAYS_CHANNEL`
  (default `relays:control`). ODIN sends the `RELAY_STATE_UPDATE` envelope,
  where `data.state` is the desired relay state.
- **Coruscant → ODIN** (telemetry), channel `REDIS_SENSORS_CHANNEL`
  (default `sensors:telemetry`). Coruscant sends the `SENSOR_DATA_UPDATE`
  envelope.
- **Last known relay state** — Coruscant writes the `RELAY_STATE_UPDATE`
  envelope to key `REDIS_RELAY_STATE_KEY_PREFIX<relay_id>` (default prefix
  `relays:state:`) so ODIN can show the current value without replay.

Pub/sub is fire-and-forget: no retention, no replay, no consumer-group offset.
A `RELAY_STATE_UPDATE` published while Coruscant is unsubscribed (restart,
WiFi drop, redeploy) is dropped permanently; ODIN must re-issue it after
Coruscant reconnects. ODIN detects the gap because `relays:state:<relay_id>`
keeps the last value Coruscant applied, so the dashboard shows the previous
state until the command is re-sent.

All channels and the state key carry the same envelope:

```json
{
  "type": "RELAY_STATE_UPDATE" | "SENSOR_DATA_UPDATE",
  "data": { ... },
  "timestamp": "<ISO-8601 UTC>"
}
```

`data` shape depends on `type`:

- `RELAY_STATE_UPDATE`: `{ "relay_id": "<string>", "state": "<string>" }`
- `SENSOR_DATA_UPDATE`: `{ "sensor_id": "<string>", "value": <float>, "unit": "<string>" }`

### 3. Shared enums (must match on both sides)

Defined in two places and **must stay in lock-step**:

| Concept       | ODIN (`odin/apps/core/redis_bus.py`)    | Coruscant (`coruscant/services/redis_bus.py`) |
| ------------- | --------------------------------------- | --------------------------------------------- |
| `MessageType` | `RELAY_STATE_UPDATE`, `SENSOR_DATA_UPDATE` | identical values                           |
| `PartitionKey`| `sensors`, `relays`                     | identical values                              |

`MessageType` and `PartitionKey` are not Kafka types. `MessageType` maps to the
envelope's `type` field on the wire; `PartitionKey` is not a wire field — it
selects the routing destination:

- `relays` — channel `relays:control` and state keys `relays:state:<relay_id>`
- `sensors` — channel `sensors:telemetry`

Both sides must use the routing above (via the `REDIS_*` env vars), so the
envelopes they produce and consume stay wire-compatible.

Rule: never rename a value in one repo without renaming it in the other in the
same release. Add new values additively; producers and consumers tolerate
unknown values.

### 4. Environment / configuration

| Variable                      | Used by          | Default (dev)                             |
| ----------------------------- | ---------------- | ----------------------------------------- |
| `API_URL`                     | coruscant        | `http://192.168.1.100/api/v1`             |
| `SYNC_API_URL`                | coruscant        | `${API_URL}/sensors/logs/`                |
| `REDIS_URL`                   | odin, coruscant  | `redis://localhost:6379/0`                |
| `REDIS_RELAYS_CHANNEL`        | odin, coruscant  | `relays:control`                          |
| `REDIS_SENSORS_CHANNEL`       | odin, coruscant  | `sensors:telemetry`                       |
| `REDIS_RELAY_STATE_KEY_PREFIX`| odin, coruscant  | `relays:state:`                           |
| `LOGS_API_URL`                | coruscant        | `${API_URL}/core/logs/`                   |
| `SERVER_ADDRESS`              | centax (firmware, hardcoded) | `http://192.168.1.100/api/v1/sensors/logs/` |

Keep defaults in sync. `centax` is a firmware build, so changing
`SERVER_ADDRESS` requires a recompile and re-flash.

## How to Make a Cross-Service Change

1. Open an issue in the umbrella repo describing the contract change; tag the
   affected submodules.
2. Land the change in **consumers first** (e.g. add a new `MessageType` value
   in `odin` while still ignoring unknown types), then **producers**.
3. One PR per submodule. Reference the umbrella issue ID in each PR title.
4. After all submodule PRs merge, open a single umbrella PR that bumps the
   three gitlinks to the new submodule SHAs and adds a short note to the
   release branch.
5. Roll back by reverting the umbrella PR — submodule commits stay in history
   and can be reverted independently if needed.

## Shared Code Style (applies in every submodule)

- Python 3.13+, type hints on public APIs, 120-char lines, 4-space indent.
- Ruff for lint + format (`uv run ruff check .` / `uv run ruff format .`).
- Bandit for security in Python submodules.
- No secrets in commits; configuration via env vars.
- Loggers at module level (`logger = logging.getLogger(__name__)`), no
  `print` for diagnostics.
- Tests with pytest; mock external services (redis-py, GPIO, HTTP).

## Conventions Unique to a Submodule

- **odin** — Django app layout (`api/`, `apps/`, `tests/`, `static/`,
  `templates/`), URL-versioned API at `/api/v1/...`, RQ for background jobs,
  APScheduler for periodic jobs, pytest-django + Factory Boy for tests.
  See `odin/AGENTS.md`.
- **coruscant** — hardware-facing Python: GPIO, DS18B20, local
  SQLite/PostgreSQL. Crontab-driven, deployed via `scp`. See
  `coruscant/AGENTS.md`.
- **centax** — Arduino sketch, no Python. Build with `arduino-cli` via the
  `Makefile`. No tests; verify by flashing a real ESP-01S. See `centax/README.md`
  and `centax/Makefile`.

## AI Behavior

- Prefer minimal, working output; skip commentary, no decorative comments.
- When a change spans submodules, state the contract being changed explicitly
  and list every file touched in every submodule.
- Never invent URLs; the only remotes are the three listed in `.gitmodules`.
