# Wiki Index

## Pages

_Newest first._

- [Relay mode round-trip through Redis](pages/2026-10-13-relay-mode-via-redis-round-trip.md) —
  `Relay.mode` was a write-only cache only an API write refreshed, so servo mode
  changes from sensor, weather or pump-off never reached the column; ODIN now
  publishes `mode` on every `RELAY_STATE_UPDATE` in both directions and coruscant
  echoes it back, fixing the admin publishing a fresh `target_state` next to a
  stale `obj.mode` (401 + 50 passed) (2026-10-13).
- [MNT-127 crosscheck — tickets, PRs, submodules](pages/2026-08-24-mnt-127-crosscheck.md) —
  Crosscheck of every ticket, PR, and submodule pointer linked to MNT-127 plus the
  MNT-178/179/180 drift fix; flags the uncommitted post-drift gitlink bump and the
  still-In-Progress state of MNT-154 (2026-08-24).
- [Kafka → Redis cutover: submodule gitlink bump (MNT-154)](pages/2026-08-24-redis-cutover-submodule-bump.md) —
  Bumped odin/coruscant gitlinks, restored Redis pub/sub docs, cut
  integrations/2026-Q3; resolved MNT-178 and pinned MNT-179/180 scope (2026-08-24).
