---
type: implementation
date: 2026-10-13
services: [relays, redis-bus, consumer, api, admin, coruscant]
branch: master
tickets: []
tags: [relays, redis, mode, coruscant, consumer, contract]
session_id: ses_ecb332ec5ffe5uSn3RKdf96q7f
related:
  - 2026-08-24-redis-cutover-submodule-bump.md
  - 2026-08-24-mnt-127-crosscheck.md
---

# Relay mode round-trip through Redis

## TL;DR

`Relay.mode` was a write-only denormalized cache: the API recomputes it on
read via `Relay.get_target_state()`, but the only path that ever persisted it
was `RelayRetrieveUpdateView.perform_update`. Servo mode changes that happen
without an API write — sensor crossing `target_temp`, weather entering
`SUMMER`, the related pump switching off — never reached the column. ODIN now
sends `mode` in every `RELAY_STATE_UPDATE` envelope and coruscant echoes it
back after polling the API, so the column tracks reality on both the control
and the poll path. 401 odin tests and 50 coruscant tests pass.

## Overview

`Relay.state` and `Relay.mode` are both persisted columns. `state` is kept
current by the Redis consumer and by `Relay.refresh_state()`; `mode` had no
such path.

`RelayTargetStateService.get_target_state()` returns `(state, mode)` and is
called from exactly three places:

- **serializers** — computed on read, never persisted
- **`RelayRetrieveUpdateView.perform_update`** — persisted, but only when
  `context` or `force_state` is PATCHed
- **`RelayAdmin.save_model`** — publishes only

So the column went stale for any mode change not accompanied by one of those
writes. The pre-existing tests could not catch this: they only asserted
`get_target_state()`, never the persisted column.

The fix wires `mode` through the existing `RELAY_STATE_UPDATE` envelope in
both directions rather than adding a channel or a message type. Per the
umbrella contract in `AGENTS.md` §2 this is an additive field, so no
`/api/v2/` bump is needed.

## Step 1 — coruscant reads `mode` from the API

**File:** `coruscant/services/api.py:14`

The poll already fetched the relay document but kept only `target_state`. It
now returns both fields. Note the shape: ODIN serves these as plain strings
(`"ON"`, `"BASIC"`), not the dicts the old tests asserted.

```python
if response.ok:
    data = response.json()

    if not (target_state := data.get("target_state")):
        logger.exception(f"{relay_id}: no target_state in response")
        return None
    mode = data.get("mode")

    return Relay(relay_id=relay_id, state=target_state, mode=mode)
```

`mode` is optional by contract — an older ODIN omits it — so it defaults to
`None`, and both consumers treat `None` as "unchanged".

## Step 2 — coruscant publishes `mode` in the envelope

**File:** `coruscant/services/redis_bus.py:107`

```python
envelope = {
    "type": MessageType.RELAY_STATE_UPDATE.value,
    "data": {"relay_id": relay.relay_id, "state": relay.state, "mode": relay.mode},
    "timestamp": datetime.now(UTC).isoformat(),
}
```

The same envelope is persisted under `relays:state:<relay_id>`, so the mode is
also readable without a live subscription.

## Step 3 — ODIN applies `mode` in the consumer

**File:** `odin/apps/core/management/commands/consumer.py:142`

```python
mode = data.get("mode")
if mode is not None:
    if mode not in RelayMode.values:
        logger.warning(f"Ignoring invalid mode {mode!r} for relay {relay_id!r}")
    elif mode != relay.mode:
        relay.mode = mode
        update_fields.append("mode")

relay.save(update_fields=[*update_fields, "updated_at"])
```

`mode` is only added to `update_fields` when it actually changed, so a
steady-state poll does not write the column 96 times a day.
`RelayLogService.TRACKED_FIELDS` already includes `mode`, so the transition is
audited in the same `RelayLog` row as the state change — no schema change.

`Relay.refresh_state()` (`odin/apps/relays/models.py:155`) applies the mode
from the persisted envelope on the same terms. That is the reconciliation path
used by the admin and the SPA dashboard poll.

## Step 4 — the ODIN → coruscant direction carries `mode` too

`RedisBus.publish_relay_control` takes a **required** `mode`, so both control
publishers had to supply one. This is where the interesting bug was.

**File:** `odin/apps/relays/admin.py:126`

The first version published the computed state next to the stored column:

```python
published = RedisBus.publish_relay_control(
    relay_id=obj.relay_id, state=obj.target_state, mode=obj.mode, user=...
)
```

`obj.target_state` is computed on read; `obj.mode` is the denormalized cache
the admin never writes. A forced relay would publish `(ON, <stale mode>)` — a
fresh state paired with a mode that contradicts it. Both now come from one
call:

```python
state, mode = obj.get_target_state()
published = RedisBus.publish_relay_control(
    relay_id=obj.relay_id, state=state, mode=mode, user=getattr(request, "user", None)
)
```

**File:** `odin/api/v1/relays/views.py:78`

`perform_update` already recomputes both at line 68, so `item.state` /
`item.mode` are the right values. It also now passes the acting user, reusing
the `user` resolved for `RelayLogService` so the audit row and the Redis
envelope attribute to the same actor:

```python
user = self.request.user if self.request.user.is_authenticated else None
...
published = RedisBus.publish_relay_control(
    relay_id=item.relay_id, state=item.state, mode=item.mode, user=user
)
```

## Update — 2026-10-13 14:40

The `Relay` NamedTuple and the `get_relay` / `update_relay` renaming landed as
a parallel refactor during this session, not as part of the mode change:

- `coruscant/models.py` — new `Relay(relay_id, state, mode=None)` NamedTuple.
- `get_relay_state(relay_id) -> str | None` → `get_relay(relay_id) -> Relay | None`.
- `update_relay_state(relay_id, state, mode=None)` → `update_relay(relay: Relay)`.

Two live call sites were left broken by that rename and only surfaced once the
suite ran:

- **`coruscant/valve.py:22,24`** still called `update_relay(relay_id=..., state=...)`,
  a `TypeError` on every valve cycle. Now builds `Relay(relay_id=..., state=...)`.
- **`coruscant/consumer.py:50`** dropped the mode when echoing an inbound control
  message, discarding ODIN's mode on every control echo. Now carries
  `data["data"].get("mode")` through.

## Callers of `RelayRetrieveUpdateView`

Traced while deciding whether the control direction needed `mode`:

- **GET** (`/api/v1/relays/<relay_id>/`, `AllowAny`, no throttle) — coruscant's
  `get_relay` on the `3-59/15 * * * *` cron plus `@reboot`, for `pumps` and
  `servos`. Sends `Authorization: Token …` although retrieve is `AllowAny`.
  `relaysApi.retrieve()` exists in the SPA but has no call site.
- **PATCH** (`RelayUpdateSerializer`, `IsAuthenticated`, `ScopedRateThrottle`
  scope `relays_update` = 30/min) — `relaysApi.updateContext()` from
  `RelayScheduleModal.tsx:257`, which sends `context.schedule.periods`
  (`target_temp` for SERVO, `target_state` for PUMP). `force_state` is
  accepted by the serializer but nothing in the SPA sends it.
- The Django admin is **not** a caller — `RelayAdmin.save_model` does its own
  recompute and publish.

## Test Results

odin — **401 passed**:

- `tests/commands/test_consumer.py` — mode applied; missing, `null` and invalid
  mode all leave the column untouched; invalid mode still applies the state.
- `tests/models/test_relays.py` — `refresh_state` applies mode; absent and
  invalid mode ignored.
- `tests/api/test_relays.py`, `tests/admin/test_relay_admin.py`,
  `tests/services/` — updated for the new `mode` kwarg and the acting user.

coruscant — **50 passed**. Note that `rpi-gpio` cannot build on macOS, so the
suite needs `uv run --no-sync pytest` against the existing `.venv`; plain
`uv run pytest` tries to reinstall and fails on `sys/epoll.h`.

## Test-environment gotchas hit along the way

Two classes of failure in this session were *not* caused by the mode change and
are worth recording:

- **API tests are auto-authenticated.** `odin/tests/conftest.py:38` wraps every
  `setup_method` with a token credential, so `request.user` in an API test is a
  real `User`, not `AnonymousUser`. Assertions on the published `user` must
  compare against `Auth.objects.get().user`, not `None`.
- **The stored `mode` column is not the computed mode.** Any assertion of the
  form `mode=relay.mode` after an admin or API write passes for the wrong
  reason. The tests now pin `RelayMode.FORCED` / `RelayMode.BASIC` explicitly,
  which is what caught the admin bug in Step 4.

## Follow-ups

- Nothing is committed. Both submodules are dirty on `master`: odin has 12 staged
  files plus a `pyproject.toml` version bump to 1.25.1, coruscant has the
  `Relay` refactor plus an untracked `coruscant/models.py`. Per both
  `AGENTS.md` files this needs a feature branch and one PR per submodule,
  followed by an umbrella gitlink bump.
- This page was first written under `odin/wiki/` and moved to the umbrella
  `wiki/`, because the change spans odin and coruscant.
- `coruscant/tests/test_relay_status.py` is vestigial — it duplicates
  `test_redis_bus.py` and patched a `coruscant.services.relay.logger` that no
  longer receives those calls.

## References

- Related: [[2026-08-24-redis-cutover-submodule-bump]]
- Related: [[2026-08-24-mnt-127-crosscheck]]
- Submodule pages covering the same subsystem live in `odin/wiki/pages/`
  (2026-09-11-relay-state-redis-consumer, 2026-09-23-relay-mode-compute-on-read,
  2026-09-29-relay-log-consumer-fixes, 2026-09-18-relay-state-mode-refactor).