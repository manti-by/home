# AGENTS.md

Umbrella repository for the home-automation platform. The three services live in
submodules; this file is the source of truth for anything that crosses submodule
boundaries.

## Layout

| Path        | Submodule                                                 | Role                                                                 |
| ----------- | --------------------------------------------------------- | -------------------------------------------------------------------- |
| `odin/`     | `git@github.com:manti-by/odin.git`                        | Central Django 5.2 / DRF server, PostgreSQL, Redis, gunicorn + worker |
| `coruscant/`| `git@github.com:manti-by/coruscant.git`                   | Raspberry Pi satellite: DS18B20 sensors, pumps/valves/servos, Kafka   |
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

### 2. Kafka topics

- Bootstrap servers come from env, default `192.168.1.100:9092`.
- **ODIN → Coruscant** (relay control), topic `KAFKA_CORUSCANT_TOPIC`:

  ```json
  { "relay_id": "<string>", "target_state": "<string>" }
  ```

- **Coruscant → ODIN** (telemetry), topic `KAFKA_ODIN_TOPIC` (default `odin`).
  Envelope:

  ```json
  { "type": "RELAY_STATE_UPDATE" | "SENSOR_DATA_UPDATE", "data": { ... } }
  ```

  `data` shape depends on `type`:

  - `RELAY_STATE_UPDATE`: `{ "relay_id": "<string>", "state": "<string>" }`
  - `SENSOR_DATA_UPDATE`: `{ "sensor_id": "<string>", "value": <float>, "unit": "<string>" }`

### 3. Shared enums (must match on both sides)

Defined in two places and **must stay in lock-step**:

| Concept       | ODIN (`odin/apps/core/kafka.py`)        | Coruscant (`coruscant/services/kafka.py`) |
| ------------- | --------------------------------------- | ----------------------------------------- |
| `MessageType` | `RELAY_STATE_UPDATE`, `SENSOR_DATA_UPDATE` | identical values                       |
| `PartitionKey`| `sensors`, `relays`                     | identical values                          |

Rule: never rename a value in one repo without renaming it in the other in the
same release. Add new values additively; producers and consumers tolerate
unknown values.

### 4. Environment / configuration

| Variable               | Used by        | Default (dev)                                    |
| ---------------------- | -------------- | ------------------------------------------------ |
| `API_URL`              | coruscant      | `http://192.168.1.100/api/v1`                    |
| `SYNC_API_URL`         | coruscant      | `${API_URL}/sensors/logs/`                       |
| `KAFKA_SERVERS`        | odin, coruscant| `192.168.1.100:9092`                             |
| `KAFKA_TOPIC`          | coruscant      | `odin`                                           |
| `KAFKA_ODIN_TOPIC`     | odin           | `odin`                                           |
| `KAFKA_CORUSCANT_TOPIC`| odin           | `coruscant`                                      |
| `LOGS_API_URL`         | coruscant      | `${API_URL}/core/logs/`                          |
| `SERVER_ADDRESS`       | centax (firmware, hardcoded) | `http://192.168.1.100/api/v1/sensors/logs/` |

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
- Tests with pytest; mock external services (Kafka, GPIO, HTTP).

## Conventions Unique to a Submodule

- **odin** — Django app layout (`api/`, `apps/`, `tests/`, `static/`,
  `templates/`), URL-versioned API at `/api/v1/...`, RQ for background jobs,
  APScheduler for periodic jobs, pytest-django + Factory Boy for tests.
  See `odin/AGENTS.md`.
- **coruscant** — hardware-facing Python: GPIO, DS18B20, Kafka producer,
  local SQLite/PostgreSQL. Crontab-driven, deployed via `scp`. See
  `coruscant/AGENTS.md`.
- **centax** — Arduino sketch, no Python. Build with `arduino-cli` via the
  `Makefile`. No tests; verify by flashing a real ESP-01S. See `centax/README.md`
  and `centax/Makefile`.

## AI Behavior

- Prefer minimal, working output; skip commentary, no decorative comments.
- When a change spans submodules, state the contract being changed explicitly
  and list every file touched in every submodule.
- Never invent URLs; the only remotes are the three listed in `.gitmodules`.
