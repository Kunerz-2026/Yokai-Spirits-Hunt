# Changelog

## v0.1.6 — 2026-11-16

### Fixed: the first school is seeded when the trait level is first raised

A monster without innate traits (the majority of the roster) ended up with only **five** trait
schools lit at level 6. `_auto_unlock_slots()` only filled the schools that a level-up *newly*
opens, and the first school is never "newly opened" — it is available from level 1 — so it stayed
empty forever. Monsters that do have innate traits were fine because `ensure_features()` seeds
their first school from the data tables.

`_auto_unlock_slots()` now also seeds the first school when `old_lv <= 1`, and skips any school
that already has content, so innate traits and features the player unlocked with ghost-path points
are never overwritten.

Note: the client-side fixes shipped the same day (storing audio entries uncompressed inside the
APK, firing the five login bag requests in parallel, and a nil guard in the bag UI) are **not**
part of this repository — see README, "What is (and is not) in this repository".
## v0.1.5 — 2026-11-15

### Fixed: raising the trait level now actually unlocks a trait

Three defects behind the report *"monsters granted from the admin console can't unlock trait slots"*:

- `handlers_specialty._featlev()` defaulted to **0** for monsters with no stored `monster_featlev`
  entry, while `state.get_feat_parts()` sends **1**. Every admin-/GM-granted monster therefore showed
  "trait level 1/6, first school lit" in the client, but the server's gate computed
  `1 <= slot <= 0` → always false, so *activate* (1183) always answered `[30025]`. The default is now
  1 (an explicitly stored 0 still counts as 0), and `ensure_features()` persists the initial level.
- `1180` (total-level upgrade) only bumped the level: it returned an empty `tallentList` and never
  filled the newly opened school. The client (`SpecialtyUpgrade.doSwallow` → `unlockTips()`) reads
  `msg[4]` to announce "unlocked new trait: …", so levelling up looked like nothing had happened.
  The new `_auto_unlock_slots()` seeds one candidate trait per school crossed (`isUse=1` when the
  slot had no active entry), returns their ids in `tallentList`, and is idempotent.
- `L97_Tallents.UnLockLevel` really has **7** tiers while the client only has 6 school slots. The
  five traits with id `1024`–`1028` used to be dumped into the first empty slot (the basic school),
  where they showed up under the wrong school and could never be extended. They are now assigned to
  the advanced school (slot 6), and 1183 prefers exact-school candidates with the >6 ones as fallback.

So raising the trait level now lights up one new school **and hands over one trait of that school**
(with the unlock popup); spending 70 ghost-path points on *activate* still unlocks an extra trait in
the same school.
## v0.1.4 — 2026-11-15

### Fixed: every login bag can now be paged; oversized replies are truncated instead of dropped

The four login bags that could grow with player data (equipment / ghosts / props / skills) are all
paged by a **byte budget** now (48 KB per page, `msg[2] = "current/total"`; the client's
`NetEquipBag` / `NetGhostBag` / `NetPropBag` / `NetSkillBag` already auto-request the next page).
`monster_bag_json` keeps its page-based paging.

`state.shrink_payload()` is the last line of defence: if a reply that is *not* paged still exceeds
the framing limit (u16 body length, 65533 bytes), `pack_login_frame()` trims the trailing rows of
the top-level list until it fits and logs
`[WIRE] !! msgid=... oversized ... truncated to ...` — so the client continues with partial data
instead of waiting forever for a frame that never arrives.

### Data file required: `tables_ygtw_1.25/L91_Status.json`

The CN 1.31 table has `Tallent1/2 = 0` for every monster; static traits come from the TW 1.25
`L91_Status.json`. When it is absent, `feat_source_fingerprint()` reports `…:missing` and **every
monster's six trait slots come out empty**. `state._ygtw_table_path()` now looks in
`$YGZZ_TABLE_YGTW`, `<work>/../revival_core/tables_ygtw_1.25/` and next to `table_dir()`, and
prints a one-time warning if none of them exists. When the file appears later, the fingerprint
changes and `recompute_features_on_source_change()` fills in the trait slots of saves that were
seeded empty (`0:0 -> 1008:1009`), without touching slots the player already edited.

**Trait policy** (unchanged, documented): the trait *catalogue* is shared by all monsters (the
client reads it from `L97_Tallents`), but the server only sends the traits a monster has actually
**unlocked**. `monster_featlev` (1–6) opens that many trait slots; each slot costs 70 鬼道积分 to
unlock a trait (msg 1183). Monsters present in the tables keep their 1–2 innate traits; the rest
start with none.

---

## v0.1.3 — 2026-11-15

### Fixed: login stuck at 40% / 60% — responses over the 64 KB frame limit

Reported log:

```
FRAMING_A handler err msg(1030) error("'H' format requires 0 <= number <= 65535")
```

The framing-A body length field is **u16** (max 65533). `equip_bag_json()` sent the whole equipment
bag in one frame; a player with many items made `struct.pack('>H', ...)` raise, so **not a single
byte was sent** and the client waited forever on the third bag — the loading bar stopped at 40%
(the prop bag has the same problem and stops at 60%).

Now:

- `state.paginate_items()` splits a list into pages by **byte budget** (48 KB, ~17 KB headroom);
- `equip_bag_json(page)` / `prop_bag_json(page)` page the response and report a real
  `"current/total"` in `msg[2]`; `game_server` reads `{"page": n}` for 1030/1033 the same way it
  already did for 1032 (the client's `NetEquipBag` / `NetPropBag` auto-request the next page);
- `pack_login_frame()` now prints `[WIRE] !! response too large msgid=... len=...` and raises a
  readable error instead of the bare `'H' format` struct error.

### Fixed: skill books could not be replaced ("default skills cannot be replaced")

`skill_bag_json()` (1506) hard-coded the sixth field `ischange` to `0`. The client's
`SkillInfo.lua:79` returns early when `ischange == 0`, showing
"角色&怪物默认技能不可替换!" — so no skill book could ever open the replace dialog, even though
`handle_skill_replace` (1505) worked. `ischange` is now derived from `skill_produced`
(skill books = 1, monster's own L91 skills = 0), the same rule `handlers_skill._skill_item()` uses.

### Note: empty trait slots are a data gap, not a bug

`static_tallents()` returns `(0, 0)` for **1014 of 1261** monsters in both the CN 1.31 and the TW
1.25 tables (e.g. every form of 坛中仙 and 天师-暮成雪); a `SourceID`-chain fallback finds talents
for exactly 0 of them. The seeding mechanism itself is fine (e.g. 哪咤 1008/1009 renders six slots).
Filling those slots would mean inventing data, so it is left empty.

---

## v0.1.2 — 2026-11-14

### Fixed: players could get stuck on the loading screen at 40%

Account saves are loaded by `accounts._load()` with a plain `json.load`, which **does not** run the
missing-key defaulting that `state.load_state()` performs for the template save. A save missing one
top-level key therefore breaks the corresponding bag handler: with `equip_id` absent,
`equip_bag_json()` raised `KeyError`, message `1030` was never answered, and the client waited
forever for the third bag — the loading bar stopped at **40%** while the server looked healthy
(a single `FRAMING_A handler err` line in the log).

Now:

- `_load()` backfills **missing** top-level keys from the template (existing values are never
  overwritten) and logs which keys were added, so an old save keeps working after an upgrade;
- `1030` / `1033` use `EQ_STATE.get(...) or {}`, so a save whose key exists but is `null` also
  produces an empty bag instead of an exception.

Same class of failure as v0.1.1's UR-shard bug: the player only sees a stuck progress bar, not an
error, so the server has to be defensive about save shape.

---

## v0.1.1 — 2026-11-14

Server-side fixes for issues found while running the private beta. No client, data tables or
assets are involved — all three changes are pure `work/` code.

### Monster bag: cap raised 21 → 200, delivered as a single page

The official client hard-codes `SQGamer:getMaxMonster() == 21`, but that value is only consumed by
a debug panel; the real capacity is whatever the server puts in the `1032` response's
`"current/total"` page field — the client auto-requests the next page when `current < total`.

**Why the page size equals the cap:** measured on a real client — **each extra page costs a fixed
5 seconds** at login (21 per page with 200 monsters = 10 pages ≈ 50 s stuck on the loading screen).
So `BAG_PAGE` is set to 200: a full bag is a single ~20 KB frame and login time is unchanged.
The paging code (`monster_bag_json(page)` + `{"page": n}` parsing in `game_server.py`) is kept for
future expansion — if you ever set `BAG_PAGE` below `MAX_BAG`, budget roughly 5 s per page.

### Fixed: composing a UR shard could brick an account (login stuck at 80%)

`L07_Item_Compose` maps the UR shard items `4000501` / `4500011` to monster id **500011**, which
exists only in the display table (`L91_StatusRender`) and has no stat row in `L91_Status`.
The old `_add_monster()` silently created it with `skin=0`/`skills=0`; the client cannot resolve
such a monster, throws while parsing `1032`, and every later login hangs at 80% (the fifth bag) —
the save is bricked with no in-game way out.

Now:

- `_add_monster()` refuses ids that are not in the monster table (returns `None`);
- `1168` validates all `ComposeID`s **before** consuming shards (all invalid → error code `114`,
  nothing consumed);
- `1032` skips unknown-id monsters already present in a save, so an affected player can at least log in;
- new `work/_fix_unknown_monsters.py` repairs existing saves (maps `500011 → 500001`, keeps
  ucode/level/exp, dry-run by default, backs up before writing, idempotent).

### Fixed: admin "one-click restore" always returned 400 in multi-account mode

Snapshots only ever covered the template `state.json`; player saves live in
`work/accounts/<aid>.json` and were never snapshotted, so `save_restore()` had to refuse —
overwriting a player's save with the template would destroy it with no way back.

Now:

- every account save is snapshotted automatically **before** the write (5-minute throttle,
  content de-duplication, 10 snapshots kept per account);
- the admin snapshot list / restore operate on account saves; a `pre-restore` copy is taken first;
- after a restore the in-memory account is evicted and online connections are dropped, so the
  client reconnects onto the restored save;
- template snapshots are rejected with a readable explanation, and the web UI shows the server's
  message instead of a bare `400`.

### Tests

Internal regression: 165/165 suites pass. The public subset in `tests/` is unchanged and still
passes (`python tests/run_tests.py`).

---

## v0.1.0 — 2026-11-13

- First public source release of the Yokai: Spirits Hunt (妖怪正传) server core.
- Server code only: no game client, installation package, assets, or data tables.
- Added configurable table directory (`YGZZ_TABLE_DIR` / `YGZZ_TABLE_DIR_ALT`).
- Added a small, self-contained public test subset.
