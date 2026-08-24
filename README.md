# Home Automation Platform

Umbrella repository for a home automation platform. The system is split
into three independently versioned services, each living in its own submodule
under this repository.

## Architecture

```
+-----------+        HTTP          +-----------------+
|  centax   | --------------------> |                 |
| (ESP-01S) |   POST /sensors/logs  |                 |
+-----------+                       |                 |
                                    |     odin        |      Redis bus
+-----------+        HTTP           |  (Django 5.2)   | <-------------> coruscant
| coruscant | --------------------> |  PostgreSQL     |   telemetry &
| (RPi sat) |   POST /sensors/logs  |  Redis + RQ     |   relay control
+-----------+                       +-----------------+
```

- **`odin/`** — Central Django 5.2 / DRF server, PostgreSQL, Redis, gunicorn + worker.
- **`coruscant/`** — Raspberry Pi satellite: DS18B20 sensors, pumps/valves/servos,
  Redis pub/sub client.
- **`centax/`** — ESP-01S firmware: DHT22 temp/humidity, HTTP POST to ODIN.

Each submodule keeps its own `AGENTS.md` with service-specific rules. The
umbrella `AGENTS.md` is the source of truth for anything that crosses
submodule boundaries — read it before making cross-service changes.

## Quick Start

Clone with submodules and pull the latest submodule commits:

```bash
git clone --recurse-submodules git@github.com:<org>/home.git
git submodule update --init --recursive   # first time only
git submodule update --remote <name>      # pull latest from a submodule
```

Inside a submodule you work as usual — it is an independent git repo. Do
**not** edit files inside a submodule from the parent checkout; commit in the
submodule first, then bump the gitlink from the parent (`git add <submodule>`).

## Cross-Service Contracts

The platform has a small number of public contracts shared by two or more
services. Any change to them is breaking and requires coordinated PRs in
every affected submodule plus an umbrella PR that bumps the gitlinks.

### 1. HTTP: sensor data ingest (`POST /api/v1/sensors/logs/`)

- **Producers:** `centax` (ESP-01S, every 5 min), `coruscant` (`sync.py`).
- **Consumer:** `odin` (DRF view in `odin/api/v1/sensors/`).
- **Request body (JSON):**

  ```json
  { "sensor_id": "<string>", "temp": <float>, "humidity": <float> }
  ```

  - `sensor_id` must already exist in `odin` (foreign key to `Sensor`).
  - `temp` is °C, `humidity` is `%RH`. Omit a field the sensor cannot provide.

- **Response:** 200 OK on success; 400 on validation; 404 on unknown
  `sensor_id`.
- **Versioning:** additive fields are non-breaking. Removing/renaming a field
  or changing its type is a breaking change → bump `/api/v2/` and keep v1
  alive until every producer has moved.

### 2. Redis pub/sub channels

- Connection comes from `REDIS_URL`, default `redis://localhost:6379/0`. Pub/sub delivery is
  fire-and-forget with no replay; the last known relay state is persisted separately under
  `relays:state:<relay_id>` so consumers can read it without a live subscription.
- Every message on both channels is a JSON envelope:

  ```json
  { "type": "RELAY_STATE_UPDATE" | "SENSOR_DATA_UPDATE", "data": { ... }, "timestamp": "<ISO-8601 UTC>" }
  ```

  Producers and consumers tolerate unknown `type` values.

- **ODIN → Coruscant** (relay control), channel `relays:control` (`REDIS_RELAYS_CHANNEL`):

  ```json
  { "type": "RELAY_STATE_UPDATE", "data": { "relay_id": "<string>", "state": "<string>" } }
  ```

  - `state` is `"ON"` or `"OFF"`; consumers must ignore unknown relays.

- **Coruscant → ODIN** (telemetry), channel `sensors:telemetry` (`REDIS_SENSORS_CHANNEL`). Envelope:

  ```json
  { "type": "SENSOR_DATA_UPDATE", "data": { ... }, "timestamp": "<ISO-8601 UTC>" }
  ```

`data` shape depends on `type`:

- `RELAY_STATE_UPDATE`: `{ "relay_id": "<string>", "state": "<string>" }`
- `SENSOR_DATA_UPDATE`: `{ "sensor_id": "<string>", "value": "<string>", "unit": "<string>" }`
  — producers may add fields (coruscant also sends `temp` and `humidity`) additively.

### 3. Shared enums

`MessageType` and `PartitionKey` are plain JSON-envelope discriminants (not broker-level
constructs) defined in two places and must stay in lock-step:

- ODIN: `odin/apps/core/redis_bus.py`
- Coruscant: `coruscant/services/redis_bus.py`

Add new values additively; producers and consumers tolerate unknown values.
Renaming a value in one repo without renaming it in the other in the same
release is forbidden.

### 4. Environment / configuration

- **`API_URL`** — coruscant — default `http://192.168.1.100/api/v1`.
- **`SYNC_API_URL`** — coruscant — default `${API_URL}/sensors/logs/`.
- **`REDIS_URL`** — odin, coruscant — default `redis://localhost:6379/0`.
- **`REDIS_RELAYS_CHANNEL`** — odin, coruscant — default `relays:control`.
- **`REDIS_SENSORS_CHANNEL`** — odin, coruscant — default `sensors:telemetry`.
- **`REDIS_RELAY_STATE_KEY_PREFIX`** — odin, coruscant — default `relays:state:`
  (last known relay state, JSON, 24h TTL).
- **`LOGS_API_URL`** — coruscant — default `${API_URL}/core/logs/`.
- **`SERVER_ADDRESS`** — centax (firmware, hardcoded) — default
  `http://192.168.1.100/api/v1/sensors/logs/`.

Keep defaults in sync. `centax` is a firmware build, so changing
`SERVER_ADDRESS` requires a recompile and re-flash.

## Development Workflow

The umbrella repo follows the same Git Flow as the submodules:

- `master` is always green; never force-push, never commit directly.
- Feature branches: `<agent-name>/feature/<id>-<descriptive-name>`.
- Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, …).
- One PR per change. PR title follows Conventional Commits.
- Linear issue tracker: `TODO` → `In Progress` → `In Review` → `Done`.

Pin a known-good combination of submodule commits in a dedicated branch of
this umbrella repo (e.g. `integrations/2026-Q2`) when shipping a release that
coordinates versions across services.

### How to Make a Cross-Service Change

1. Open an issue in the umbrella repo describing the contract change; tag
   the affected submodules.
2. Land the change in **consumers first** (e.g. add a new `MessageType`
   value in `odin` while still ignoring unknown types), then **producers**.
3. One PR per submodule. Reference the umbrella issue ID in each PR title.
4. After all submodule PRs merge, open a single umbrella PR that bumps the
   three gitlinks to the new submodule SHAs and adds a short note to the
   release branch.
5. Roll back by reverting the umbrella PR — submodule commits stay in
   history and can be reverted independently if needed.

## Shared Code Style

Applies in every submodule:

- Python 3.13+, type hints on public APIs, 120-char lines, 4-space indent.
- Ruff for lint + format (`uv run ruff check .` / `uv run ruff format .`).
- Bandit for security in Python submodules.
- No secrets in commits; configuration via env vars.
- Loggers at module level (`logger = logging.getLogger(__name__)`), no
  `print` for diagnostics.
- Tests with pytest; mock external services (Redis, GPIO, HTTP).

Per-submodule conventions (Django layout for `odin`, hardware-facing Python
for `coruscant`, Arduino sketch built with `arduino-cli` for `centax`) are
documented in each submodule's `AGENTS.md` or `README.md`.

## AI Behavior

- Prefer minimal, working output; skip commentary, no decorative comments.
- When a change spans submodules, state the contract being changed
  explicitly and list every file touched in every submodule.
- Never invent URLs; the only remotes are the three listed in `.gitmodules`.
