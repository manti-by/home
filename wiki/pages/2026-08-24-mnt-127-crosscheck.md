---
type: investigation
date: 2026-08-24
related:
  - 2026-08-24-redis-cutover-submodule-bump.md
---

# MNT-127 crosscheck — tickets, PRs, submodules

Crosscheck of every ticket, PR, and submodule pointer linked to the MNT-127
Kafka → Redis cutover epic, including the post-cutover drift found in
MNT-178 and fixed by MNT-179 / MNT-180.

## TL;DR

- The epic and all six direct children landed; every code PR is merged on its
  submodule `master`; the umbrella `AGENTS.md` contract swap is merged on
  `home/master`.
- The umbrella `MNT-154` PR (manti-by/home#2) is merged, but its merged
  submodule pointers predate the `MNT-179` (odin) and `MNT-180` (coruscant)
  merges that resolved the post-cutover transport split. Working tree currently
  carries those new pointers as uncommitted gitlink bumps.
- `MNT-154` is still `In Progress` in Linear and `MNT-127` is still `In Review`;
  both need a follow-up commit on the umbrella to land the post-MNT-154
  pointers before they can be moved to `Done`.
- No `KAFKA_` references remain in either submodule's `master`; umbrella docs
  match the contract. E2E smoke flows are documented per ticket.

## Tickets — status crosscheck

- **MNT-127** — Replace Kafka with Redis (ODIN + Coruscant) — home epic —
  **In Review** — no PR.
- **MNT-149** — Update AGENTS.md contract — Kafka to Redis — home — **Done**
  — PR `manti-by/home#1` — merged (`db1ebef`).
- **MNT-150** — Add Redis deps, settings, `redis_bus.py` — coruscant —
  **Done** — PR `manti-by/coruscant#10` — merged (`3b523b7` + `88e07da`
  review fix).
- **MNT-151** — Switch to Redis, remove Kafka — coruscant — **Done** — PR
  `manti-by/coruscant#11` — merged (`10e45f9` + `5c8ddeb` review fix).
- **MNT-152** — Add Redis deps, settings, `redis_bus.py` — odin — **Done**
  — PR `manti-by/odin#26` — merged (`5ea19dd`).
- **MNT-153** — Switch `consume_sensors` + relay admin/model to Redis —
  odin — **Done** — PR `manti-by/odin#27` — merged (`44c0269`).
- **MNT-154** — Bump submodule gitlinks after Kafka to Redis cutover — home
  — **In Progress** — PR `manti-by/home#2` — merged (`e7a18ab`), but see
  drift section below.
- **MNT-178** — Decide canonical Redis transport (streams vs pub/sub) — home
  — **Done** — decision only, no PR.
- **MNT-179** — Align RedisBus publish/consume paths with canonical
  transport — odin — **Done** — PRs `manti-by/odin#30` + `#31` — merged
  (`5941996` + `c11204b` review fix).
- **MNT-180** — Align coruscant Redis transport with canonical decision —
  coruscant — **Done** — PR `manti-by/coruscant#13` — merged (`63362dc`).

## Submodule pointers

Snapshot taken from this repo's working tree (`git submodule status`):

- **odin** — `1297da9ff911558027daefc1f02d91de7c083190`
  (`v1.10.2-51-g1297da9`, `MNT-179` merge tip).
- **coruscant** — `f6eb776024476f0e3afa62b9c208a1ba673175ea`
  (`v4.1.2-45-gf6eb776`, `MNT-180` merge tip).
- **centax** — `a1eac0b4b6d654e2d3d91c7d8d9ccc6d8542c427`
  (`v1.0.0-3-ga1eac0b`, unchanged — out of scope for MNT-127, posts to HTTP).

`home/master` currently points at:

- **odin** — `9422aa0d907a92bdaa81850b7c5fad14790ab001`
  (PR manti-by/odin#29, `epic/redis` merge; predates MNT-179).
- **coruscant** — `8cd473a1a411b04e9cebd8c120ab70dd29405e7b`
  (PR manti-by/coruscant#12, `epic/redis` merge; predates MNT-180).

The `integrations/2026-Q3` release branch (`acd27e5`) was cut against those
pre-drift SHAs and currently lags `master` of both submodules.

## Drift between home/master and submodule masters

The submodule pointer bump committed in `home/master` via `195ac5d` (`MNT-154`)
captured the post-MNT-153 / MNT-151 SHAs but was authored before MNT-179 and
MNT-180 were merged. After that umbrella commit:

- MNT-179 was merged into `odin/master` at `1297da9`, replacing the stream
  consumer (`XREADGROUP` → `SUBSCRIBE`) and wrapping `publish_relay_control`
  in the standard envelope (flat `{relay_id, target_state}` → enveloped
  `{relay_id, state}` + `timestamp`). PRs #30 and #31 both closed; final SHA
  after review fixes is `c11204b`.
- MNT-180 was merged into `coruscant/master` at `f6eb776`, adding regression
  tests that guard the envelope parse (`data["data"]["relay_id"]` /
  `data["data"]["state"]`) and the `timestamp` presence in published
  envelopes. No transport change — coruscant was already pub/sub.

Both submodule merges landed after `home/master`'s `e7a18ab` merge commit, so
the working tree currently shows the new gitlinks as uncommitted changes:

```
 M coruscant
 M odin
```

`git diff HEAD`:

```
diff --git a/coruscant b/coruscant
index 8cd473a..f6eb776 160000
--- a/coruscant
+++ b/coruscant
@@ -1 +1 @@
-Subproject commit 8cd473a1a411b04e9cebd8c120ab70dd29405e7b
+Subproject commit f6eb776024476f0e3afa62b9c208a1ba673175ea
diff --git a/odin b/odin
index 9422aa0..1297da9 160000
--- a/odin
+++ b/odin
@@ -1 +1 @@
-Subproject commit 9422aa0d907a92bdaa81850b7c5fad14790ab001
+Subproject commit 1297da9ff911558027daefc1f02d91de7c083190
```

## Cross-service contract — final shape

Per the MNT-178 decision and the merged code, both channels use Redis
**pub/sub** with a single envelope:

```json
{
  "type": "RELAY_STATE_UPDATE" | "SENSOR_DATA_UPDATE",
  "data": { "relay_id": "<string>", "state": "ON" | "OFF" },
  "timestamp": "<ISO-8601 UTC>"
}
```

Key invariants verified against merged code:

- **Transport** — Redis pub/sub only. `rg -n 'xadd|xreadgroup|xautoclaim|xgroup|xack'` in
  `odin/apps/core` returns zero hits after MNT-179 (PR #30 + #31). Coruscant
  never used streams; `rg -n 'xadd|xread|xgroup|stream'` in `coruscant/`
  returns zero hits.
- **Envelope** — Both channels publish
  `{type, data:{relay_id, state}, timestamp}` with `state` = `ON` / `OFF`.
  Odin wraps relay control (MNT-179); coruscant wraps sensor data and relay
  state already (MNT-150 / MNT-151).
- **Last known state** — Coruscant writes `relays:state:<relay_id>` (JSON,
  24h TTL) on every `RELAY_STATE_UPDATE`; odin reads it via
  `RedisBus.get_relay_state`.
- **Env vars** — `REDIS_URL`, `REDIS_RELAYS_CHANNEL`,
  `REDIS_SENSORS_CHANNEL`, `REDIS_RELAY_STATE_KEY_PREFIX` in both submodules
  and umbrella `AGENTS.md` section 4.
- **Removed** — `KAFKA_SERVERS`, `KAFKA_TOPIC`, `KAFKA_ODIN_TOPIC`,
  `KAFKA_CORUSCANT_TOPIC`, `kafka-python` dep, `services/kafka.py`
  (coruscant), `core/kafka.py` (odin).

## Open items / follow-ups

1. **Commit the post-drift gitlinks in this repo.** New umbrella commit (or
   new `MNT-181`-style ticket) to bump `odin` → `1297da9` and `coruscant` →
   `f6eb776` and fast-forward `integrations/2026-Q3` so the release branch
   matches. Without it the umbrella repo is bisect-broken between
   `home/master` and the actual submodule state.
2. **Move `MNT-154` to `Done` in Linear** once the gitlink bump above lands
   and `rg -n 'KAFKA_' .` in the umbrella still returns zero hits.
3. **Move `MNT-127` (epic) to `Done`.** No further code work is expected; the
   only remaining work is the umbrella gitlink bump above.
4. **Optional operator cutover** — systemd units, `.env`, and any
   `docker-compose` for the broker removal (Kafka broker container can now be
   stopped). Not tracked as a Linear ticket.
5. **E2E smoke (still recommended)** — manual checklist from MNT-127 / MNT-151
   / MNT-153 acceptance criteria:
   - admin save on a `Relay` row → coruscant GPIO flips → next page render
     shows the new state.
   - cron sensor run on coruscant → `SensorLog` row created in odin.
   No CI smoke harness exists; verify on dev hardware.

## Verification performed

- `git log --oneline -20` on `home`, `odin`, `coruscant` — confirmed the
  ticket → commit → PR lineage in the tables above.
- `git submodule status` — submodule pointers in working tree match the post
  MNT-179 / MNT-180 submodule master tips.
- `git diff HEAD` — only the two gitlink changes; no diff inside any
  submodule from the umbrella checkout.
- `git log -1 --stat integrations/2026-Q3` — release branch commit
  `acd27e5` lists 10 lines, references pre-drift SHAs (needs fast-forward
  after the gitlink bump above).
- `git branch -a` — three local branches (`master`, release, feature); no
  stray `epic/redis` branches left behind in the umbrella checkout.
- Cross-referenced every Linear ticket's `state` against its linked PR /
  commit; no merged code without a `Done` ticket and no `Done` ticket
  without merged code.

## Files / artifacts referenced

- `AGENTS.md` (umbrella contract).
- `wiki/pages/2026-08-24-redis-cutover-submodule-bump.md` (MNT-154 wiki
  report; precursor to this crosscheck).
- `integrations/2026-Q3.md` (release branch note, SHAs to be refreshed).
- `odin/apps/core/redis_bus.py`, `odin/apps/relays/admin.py`,
  `odin/apps/relays/models.py`,
  `odin/apps/sensors/management/commands/consume_sensors.py` (post-MNT-179).
- `coruscant/services/redis_bus.py`, `coruscant/consumer.py`,
  `coruscant/tests/test_consumer.py`, `coruscant/tests/test_redis_bus.py`
  (post-MNT-180).
