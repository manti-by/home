# AGENTS.md

Umbrella repository for the home-automation platform. The three services live in
submodules; this file is the source of truth for anything that crosses submodule
boundaries.

## Layout

- **`odin/`** — `git@github.com:manti-by/odin.git` — Central Django 6 / DRF server,
  PostgreSQL, Redis, gunicorn + worker; ships a React/Vite SPA frontend
  (`frontend/`) and eBus boiler control (`apps/boiler`, backed by `ebusd`).
- **`coruscant/`** — `git@github.com:manti-by/coruscant.git` — Raspberry Pi satellite:
  DS18B20 sensors, pumps/valves/servos, Redis pub/sub client.
- **`centax/`** — `git@github.com:manti-by/centax.git` — ESP-01S firmware: DHT22
  temp/humidity, HTTP POST to ODIN.

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
umbrella repo (e.g. `integrations/2026-Q3`) when shipping a release that
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
  { "sensor_id": "<string>", "temp": <float>, "humidity": <float>, "created_at": "<ISO-8601>" }
  ```

  - `sensor_id` is a free-form string — `odin` does not require it to
    pre-exist; logs for unknown IDs are stored as-is (`SensorLog.sensor_id`
    is a plain column, not a foreign key).
  - `temp` is °C, `humidity` is `%RH`. Omit a field the sensor cannot provide.
  - `created_at` is optional; when omitted, `odin` stamps server time.

- **Response:** 201 Created on success; 400 on validation errors. An unknown
  `sensor_id` is not rejected.
- **Versioning rule:** additive fields are non-breaking. Removing/renaming a
  field or changing its type is a breaking change → bump `/api/v2/` and keep
  v1 alive until every producer has moved.

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
  {
    "type": "RELAY_STATE_UPDATE",
    "data": { "relay_id": "<string>", "state": "<string>", "mode": "<string|null>", "user_id": "<int|null>" }
  }
  ```

  - `state` is `"ON"` or `"OFF"`; consumers must ignore unknown relays.
  - `mode` is required and is the freshly computed mode (`Relay.get_target_state()[1]`)
    that matches the published `state` — not the stored column, which may be stale.
  - `user_id` is the acting user's pk, or `null` for system-driven updates.

- **Coruscant → ODIN** (relay status), channel `relays:control`, same `RELAY_STATE_UPDATE` type:

  ```json
  { "type": "RELAY_STATE_UPDATE", "data": { "relay_id": "<string>", "state": "<string>", "mode": "<string|null>" } }
  ```

  - `mode` is the value ODIN returned in the `mode` field of
    `GET /api/v1/relays/<relay_id>/`; it is `null` for relays driven locally
    (valves) or when the relay was switched by an older ODIN.
  - ODIN persists `mode` only when it is a valid `RelayMode`; a missing or
    `null` mode leaves the stored column untouched.

- **Coruscant → ODIN** (telemetry), channel `sensors:telemetry` (`REDIS_SENSORS_CHANNEL`). Envelope:

  ```json
  { "type": "SENSOR_DATA_UPDATE", "data": { ... }, "timestamp": "<ISO-8601 UTC>" }
  ```

`data` shape depends on `type`:

- `RELAY_STATE_UPDATE`: `{ "relay_id": "<string>", "state": "<string>", "mode": "<string|null>" }`
  — `mode` is present in both directions but consumers must treat a missing or
  `null` mode as "unchanged", since coruscant emits `null` for locally driven
  relays (valves). `user_id` is ODIN-only.
- `SENSOR_DATA_UPDATE`: `{ "sensor_id": "<string>", "value": "<string>", "unit": "<string>" }`
  — producers may add fields (coruscant also sends `temp` and `humidity`) additively.

### 3. Shared enums (must match on both sides)

Defined in two places and **must stay in lock-step**:

- **`MessageType`** — `RELAY_STATE_UPDATE`, `SENSOR_DATA_UPDATE`.
- **`PartitionKey`** — `sensors`, `relays`.

Locations:

- ODIN: `odin/apps/core/redis_bus.py`
- Coruscant: `coruscant/services/redis_bus.py`

Both enums are plain JSON-envelope discriminants now, not broker-level constructs.

Rule: never rename a value in one repo without renaming it in the other in the
same release. Add new values additively; producers and consumers tolerate
unknown values.

### 4. Environment / configuration

- **`API_URL`** — coruscant — default `http://192.168.1.100/api/v1`.
- **`SYNC_API_URL`** — coruscant — default `${API_URL}/sensors/logs/`.
- **`REDIS_URL`** — odin, coruscant — default `redis://localhost:6379/0`.
- **`REDIS_RELAYS_CHANNEL`** — odin, coruscant — default `relays:control`.
- **`REDIS_SENSORS_CHANNEL`** — odin, coruscant — default `sensors:telemetry`.
- **`REDIS_RELAY_STATE_KEY_PREFIX`** — odin, coruscant — default `relays:state:`
  (last known relay state, JSON, 24h TTL).
- **`LOGS_API_URL`** — coruscant — default
  `http://192.168.1.100/api/v1/core/logs/` (literal; unlike `SYNC_API_URL`
  it does not follow `API_URL` overrides).
- **`SERVER_ADDRESS`** — centax (firmware, hardcoded) — default
  `http://192.168.1.100/api/v1/sensors/logs/`.

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
- Tests with pytest; mock external services (Redis, GPIO, HTTP).

## Documentation Style (applies to every `.md` file in this repo and every submodule)

- **No GitHub-flavored Markdown tables.** Render tabular data as bulleted
  lists with bold lead-ins (e.g. `**Foo** — description, default, notes`).
  Tables make diffs noisy, don't reflow in narrow viewers, and are easy to
  drift out of sync across columns.
- Plain fenced code blocks for JSON, command output, file paths, and config
  snippets.
- One H1 (`#`) per file; H2 (`##`) for top-level sections; H3 (`###`) only
  for sub-sections.
- **Hard-wrap paragraph text at 120 chars.** Every line of prose — bullets,
  paragraphs, list items — must wrap so no line exceeds 120 characters. This
  matches the Python `line-length = 120` setting in every submodule and keeps
  diffs, code review, and `git log -p` readable. Code blocks and inline code
  are exempt.
- Do not paste screenshots for things a code block can express.

## Conventions Unique to a Submodule

- **odin** — Django app layout (`api/`, `apps/`, `tests/`, `static/`,
  `templates/`), URL-versioned API at `/api/v1/...`, RQ for background jobs,
  APScheduler for periodic jobs, pytest-django + Factory Boy for tests.
  See `odin/AGENTS.md`.
- **coruscant** — hardware-facing Python: GPIO, DS18B20, Redis pub/sub
  client, local SQLite/PostgreSQL. Crontab-driven (`utils/crontab.conf`),
  deployed by pulling on the Pi (`make deploy`). See `coruscant/AGENTS.md`.
- **centax** — Arduino sketch, no Python. Build with `arduino-cli` via the
  `Makefile`. No tests; verify by flashing a real ESP-01S. See `centax/README.md`
  and `centax/Makefile`.

## AI Behavior

- Prefer minimal, working output; skip commentary, no decorative comments.
- When a change spans submodules, state the contract being changed explicitly
  and list every file touched in every submodule.
- Never invent URLs; the only remotes are the three listed in `.gitmodules`.
