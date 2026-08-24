---
type: implementation
session_id: ses_fcd45a4befferUSqM66UMeumQL
date: 2026-08-24
---

# Kafka → Redis cutover: submodule gitlink bump (MNT-154)

## TL;DR

Closed the umbrella half of the MNT-127 Kafka → Redis epic. Verified the
`odin` / `coruscant` submodule gitlinks point at the post-cutover `master`
tips, restored the Redis pub/sub contract docs that MNT-149 had lost when the
repo was re-initialized, and cut the `integrations/2026-Q3` release branch.
Also resolved the follow-up transport decision (MNT-178 → Done) and filed
MNT-179 / MNT-180 as implementation follow-ups.

## Context

- **MNT-154** — bump submodule gitlinks after the Kafka → Redis cutover and cut
  a release branch. Blocked by MNT-149 (docs), MNT-151 (coruscant), MNT-153
  (odin), all already Done.
- The umbrella repo had only an `Initial commit`; the MNT-149 doc changes had
  been lost, so `AGENTS.md` / `README.md` still carried the full Kafka contract.

## What changed

- **AGENTS.md** — Layout table → bullets; section 2 "Kafka topics" → "Redis
  pub/sub channels"; section 3 enum paths → `redis_bus.py` with JSON-envelope
  discriminant note; section 4 env table → bullets with the four `REDIS_*`
  vars replacing the four `KAFKA_*` rows; coruscant conventions / code-style
  bullets drop "Kafka". GFM tables converted to bullets per Documentation
  Style.
- **README.md** — mirrored the same Redis contract changes; architecture
  diagram "Kafka" → "Redis bus".
- **odin** gitlink `ec23d24` → `9422aa0` (PR manti-by/odin#29).
- **coruscant** gitlink `d00cfe9` → `8cd473a` (PR manti-by/coruscant#12).
- **centax** untouched (`a1eac0b`).
- **integrations/2026-Q3.md** — 10-line release note listing pinned SHAs.
- **Makefile** + **opencode.json** — pre-existing staged tooling committed in a
  separate trailing commit (`bde3e63`), droppable from the PR.

## Branch / commits

- Feature branch: `ox-alpha/feature/mnt-154-bump-submodule-gitlinks`
  - `d8e83c0` — `docs: replace Kafka contract with Redis pub/sub channels`
  - `195ac5d` — `chore: bump odin and coruscant gitlinks after Redis cutover`
  - `bde3e63` — `chore: add submodule sync Makefile target and opencode config`
- Release branch: `integrations/2026-Q3` (`acd27e5`)
- PR: https://github.com/manti-by/home/pull/2 (Linear → In Review)

## Verification

- `rg -n 'KAFKA_'` in umbrella → zero hits.
- `git submodule status` SHAs match the release note; no diffs inside any
  submodule.
- Working tree clean; acceptance criteria from MNT-154 all pass.

## Contract drift found (→ MNT-178)

The merged submodules did not interoperate: odin used Redis **streams**
(`XADD` / `XREADGROUP` consumer groups) while coruscant used **pub/sub**
(`PUBLISH` / `SUBSCRIBE`), and odin's relay-control payload was flat
`{relay_id, target_state}` vs coruscant's expected envelope
`data.data.relay_id` / `data.data.state`.

Resolved (MNT-178 → Done): canonical transport is **pub/sub**, single
envelope `{type, data:{relay_id, state}, timestamp}` with `state` = ON/OFF.
odin drops streams (MNT-179); coruscant verify-only (MNT-180). Umbrella docs
already match — no further doc change.

## Files touched

- `AGENTS.md`
- `README.md`
- `odin` (gitlink)
- `coruscant` (gitlink)
- `integrations/2026-Q3.md` (release branch)

## Update — 2026-08-24

Resolved MNT-178 (transport decision) and rechecked/updated the two follow-up
implementation tickets against it.

- **MNT-178 → Done.** Decision: canonical transport is Redis **pub/sub**
  (epic-faithful); single envelope `{type, data:{relay_id, state}, timestamp}`
  with `state` = `ON`/`OFF` on both channels; unknown `type` tolerated.
- **MNT-179 (odin) → Todo**, scope pinned: drop Redis Streams —
  `publish_message` `XADD` → `PUBLISH`; `publish_relay_control` flat
  `{relay_id, target_state}` → enveloped `{relay_id, state}` + `timestamp`;
  `consume_sensors` `XREADGROUP` consumer group → plain `SUBSCRIBE`/`listen()`
  loop, dropping `drain_pending` / `reclaim_pending` / `dead_letter_or_retry` /
  attempts-hash machinery; update tests.
- **MNT-180 (coruscant) → Todo**, scope pinned: verify-only + test hardening.
  Transport (pub/sub) and envelope (`data.data.relay_id` / `data.data.state`,
  `timestamp` present) already match MNT-178; add regression tests guarding
  the flat-payload bug and `timestamp` presence.
- Umbrella docs unchanged — AGENTS.md section 2 already matched the decision.

## Follow-ups

- MNT-179 — odin: drop streams, `consume_sensors` → SUBSCRIBE loop, envelope
  control payload (Todo, scope pinned).
- MNT-180 — coruscant: verify transport + add regression tests (Todo, scope
  pinned).
- MNT-154 — awaiting review/merge of PR #2.
