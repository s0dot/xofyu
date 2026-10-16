[← Build spec index](../STUDIO_BUILD_SPEC.md) · **Part 4 of 8** · sections §4.x · Part 0 contracts (§0.x) are in the index file

## 4. Style kit framework and vertical-slice kits

This part specifies how a Fighting Style is represented, validated and executed. It covers:
- the registry `Shared/Styles/init.luau`;
- the nine behavior modules `Shared/Behaviors/*.luau`;
- the passive/Quirk hook system;
- the Peak (Awakening) and Getaway (Realm) content;
- complete data for the six vertical-slice kits (`attendant`, `lifeguard`, `sandcastle`, `busker`, `scoop`, `dustsheet`) and the 12 stubs;
- the placeholder-animation spec;
- presentation mapping (palettes, VFX, SFX, Charm Gear, Peak pieces, summon models);
- the Base text of the style and move Skin keys (§4.5.10);
- the checklist for adding a style;
- the engine items E1 (M1-G) and E2 (M0) and the future hooks E3–E7 (§4.9).

**Relationship to Part 3 (binding).** Part 3 owns the simulation: the FSM, `Core/*` (including the status codec in `Core/Moves.luau` and the framework in `Core/Realm.luau`), hit resolution, carriers (spawned instances) and the adapter that calls behaviors. Part 4 plugs into Part 3's contracts **by name**, from Part 3 §3.4.1, §3.4.2 and §3.9.3. The shared combat types live in the Combat section of `src/Shared/Types.luau` (Part 3 §3.4.1) and `Core/Fsm.luau` re-exports them, so Part 4 code (which sits above `Fsm`) writes `Fsm.X`; Part 4 content inside Part 3's leaf modules (`Moves.STATUS`, `Realm.rules`) writes `Types.X`:
- `Fsm.CharState`, `Fsm.StepCtx`, `Fsm.InputFrame`, `Fsm.HitDef`, `Fsm.HitboxReq`, `Fsm.HitRequest`, `Fsm.SpawnRequest`, `Fsm.MotionCmd`, `Fsm.SimEvent`, `Fsm.ChassisMods`, `Fsm.StyleRuntime`;
- `BehaviorCtx`, `BehaviorOut`, `BehaviorModule {step, onContact?, onEnd?}`;
- `Moves.Status` / `Moves.STATUS` / `Moves.statusAdd`;
- `Realm.rules`, `RealmView`, `MemberView`, `RealmRules`.

Where Part 4 needs something Part 3 has not defined, it is listed in §4.9 as an **extension request**. If Part 3 later defines an equivalent under another name, Part 3's name wins and the semantics here still bind.

Conventions:
- Frames are 60 Hz. `e` is the frame elapsed in the current state (0-based, Part 3 convention): startup is `e ∈ [0, S−1]`, active is `[S, S+A−1]`, recovery is `[S+A, S+A+R−1]`. "Move frame n" (1-based) means `e + 1 = n`. Distances are in studs, times in seconds (`Sec` suffix).
- `(td)` means a tuning default chosen here because the GDD gives no number. It satisfies the cited GDD rule.
- GDD numbers are cited as `GDD §x.y` and are never changed. Kits are cited as `GDD §7.3.N` with the numbers in §0.4.
- **Names (§0.12).** Code, data and Skin keys keep their internal names: `Awakening`/`awakening` and `AwakeningDef`, `RealmCast`/`Realm.rules`, `StubbornHeart`, `moment`/`MomentDef`, `kitProps`, `ShowStyleGear`. Prose uses the player-facing names: **Peak** (filled by the **Peak Meter**), **Getaway**, **Do Not Disturb**, **Double Booking**, **Snapshot** (a style moment), **Charm Gear** (one oversized prop plus a head piece) and **Full Uniform**.
- Every requirement has a **Verify** line with one of:
  - a Lune test (`tests/core/<file>.spec.luau`, test name);
  - an in-Studio scenario (Part 7 harness, `[HB][TEST] <name> PASS|FAIL`);
  - an MCP call;
  - an `[HB]` log line.
- Every engine API named here was checked in the offline reference snapshot (2026-10-10). Unverifiable items say "(verify in spike)".

**Files owned by this part:**

| Path | Contents |
|---|---|
| `src/Shared/Styles/init.luau` | Registry, `KitBuilder` implementation, 12 stubs, validation, config overrides, server hook dispatch, Snapshot tracker, Hard Hat rate table, `SFX_SPEED` table |
| `src/Shared/Styles/attendant.luau` … `dustsheet.luau` | One kit each: `StyleDef` + passive hooks + Snapshot (`moment`) + animation spec (§4.5) |
| `src/Shared/Behaviors/Buff.luau` | Buff/Debuff behavior **and the kit kernel**: shared Part 4 types, status helpers, hit/aim helpers (§4.2.0, §4.2.8) |
| `src/Shared/Behaviors/Rush.luau`, `Projectile.luau`, `Grab.luau`, `Counter.luau`, `Zone.luau`, `Summon.luau`, `Mobility.luau`, `RealmCast.luau` | Behaviors (§4.2) |
| Content inside Part 3 files | `Moves.STATUS` entries in `Core/Moves.luau` (§4.3.2); `Realm.rules["dustsheet.a4"]` in `Core/Realm.luau` (§4.4.3) |
| `tests/core/styles.spec.luau`, `kits.spec.luau`, `behaviors.spec.luau`, `kit_status.spec.luau` | Lune specs (§4.1.7, §4.2.11, §4.3.7) |
| Data consumed by `BuildPlaceholderAnims.luau` and `BuildTemplates.luau` (Part 6 implements) | §4.6, §4.7 |

**Hard rules for all Part 4 code (P4-R1 … P4-R7):**
1. **Load-pure and Instance-free.** `Styles/*` and `Behaviors/*`:
   - never reference `game`, `workspace`, `script`, `Instance`, `Enum`, `task` or any service;
   - use only the datatypes `Vector3`, `CFrame` and `Color3`, plus the arguments they are given.
   This makes every behavior a deterministic function, testable in Lune. Extend `tools/lint_rules.luau` rule `pure` to `src/Shared/Styles/**` and `src/Shared/Behaviors/**` (§4.9 item 1).
2. **Sim-safe.** Behaviors obey Part 3 §3.1.5:
   - no yields, `os.*`, `math.random` or logging;
   - no `pairs` over keyed tables where order matters;
   - Q-rule rounding for any float that reaches an attribute (Part 3 §3.1.4).
3. **No frame-equality one-shots.** A one-shot action happens exactly once per move use. It is guarded by a scratch flag (`MvA`/`MvB` bit) and fires on the first step where `e ≥ threshold`, never on `e == X` (Part 3 §3.1.3).
4. **Citations on numbers.** Every number in kit data carries a same-line comment `GDD §x.y` or `(td)`.
5. **IDs, not display names, in code.** Identifiers, asset keys and data use style IDs, move IDs and slot pointers. Style names, epithets and move names that players see live only in the `Base` pack (Part 8). Every kit line points at its GDD row, e.g. `-- gdd: §7.3.1 row 3`; a comment may add the effect in plain words. No third-party name appears anywhere (Part 8 lints).
6. **Writes only through outputs.** Behaviors return `BehaviorOut` / `CarrierOut` / `HookOut`; they never mutate their inputs (`table.clone` first).
7. **Player-visible text** comes from Skin keys `style.<id>.*` and `move.<moveId>.name`. Kit data contains no display strings.

---

### 4.1 Styles registry (`src/Shared/Styles/init.luau`)

#### 4.1.1 Module shapes

Each kit file returns a **builder function**. Kit files never require `Styles/init`, which avoids a require cycle. The registry passes in the shared constructors:

```lua
--!strict
-- src/Shared/Styles/attendant.luau (shape only; full data in §4.5.2)
local Kit = require("../Behaviors/Buff") -- kit kernel: types + helpers (§4.2.0)
return function(K: Kit.KitBuilder): Kit.StyleModule
	return { def = (nil :: any), hooks = {}, moment = nil, anim = {} }
end
```

Kernel types used by the registry (exported from `Behaviors/Buff.luau`, §4.2.0.3):

```lua
export type MoveFields = {
	behavior: Types.Behavior, tags: { Types.Tag },
	startup: number, active: number, recovery: number,   -- frames @ 60 Hz
	damage: number, cooldown: number,                    -- GDD value; cooldown in seconds
	guardDamage: number?, chip: number?, hitstun: number?,
	knockback: Types.KnockbackDef?, hitbox: Types.HitboxDef?,
	parryable: boolean?, signature: boolean?, superArmor: boolean?, projectileArmor: boolean?,
	telegraph: { shape: "Circle" | "Line" | "Cone" | "Arc", leadFrames: number, size: Vector3 }?,
	params: { [string]: any }?, meterGain: number?,
}
export type KitBuilder = {
	move: (id: string, f: MoveFields) -> Types.MoveDef,      -- derives slot, anim/vfx/sfx keys and defaults
	stubMove: (id: string) -> Types.MoveDef,                 -- inert placeholder for implemented = false styles
	box: (w: number, h: number, d: number, fwd: number?, up: number?) -> Types.HitboxDef,
	sphere: (diameter: number, fwd: number?, up: number?) -> Types.HitboxDef,
	kb: (dist: number, angleDeg: number, ragdoll: boolean, launch: boolean?) -> Types.KnockbackDef,
	tele: (shape: "Circle" | "Line" | "Cone" | "Arc", lead: number, x: number, z: number) -> { shape: "Circle" | "Line" | "Cone" | "Arc", leadFrames: number, size: Vector3 },
	rgb: (r: number, g: number, b: number) -> Color3,
	v3: (x: number, y: number, z: number) -> Vector3,
}
export type MomentDef = { seq: { string }, windowF: number, sameVictim: boolean }
export type AnimMoveSpec = {
	windup: string, strike: string, mid: string?, alt: { string }?, finish: string?, settle: string?,
	face: { windup: string, active: string, recovery: string }?,
}
export type StyleModule = {
	def: Types.StyleDef,                 -- §0.8, verbatim type
	hooks: PassiveHooks,                 -- §4.3.1
	moment: MomentDef?,                  -- §4.3.5; nil = detected inside hooks
	anim: { [string]: AnimMoveSpec },    -- keys "m1", "q", "1".."4", "a1".."a4" (§4.6)
}
```

`KitBuilder` semantics (implemented in `Styles/init.luau`):

```lua
local function move(id: string, f: Kit.MoveFields): Types.MoveDef
	local styleId, slot = string.match(id, "^(%l+)%.(%w+)$")
	assert(styleId ~= nil and slot ~= nil, `[HB][Style] bad move id {id}`)
	local damaging = f.damage > 0
	local grab = f.behavior == "Grab"
	return {
		id = id, slot = slot :: any, behavior = f.behavior, tags = f.tags, signature = f.signature,
		startup = f.startup, active = f.active, recovery = f.recovery,
		damage = f.damage, guardDamage = f.guardDamage or 0,
		chip = if f.chip ~= nil then f.chip elseif damaging and not grab then 0.2 else 0, -- GDD §6.4 (20% chip)
		hitstun = f.hitstun or 0, knockback = f.knockback, hitbox = f.hitbox, cooldown = f.cooldown,
		parryable = if f.parryable ~= nil then f.parryable else (damaging and not grab),
		superArmor = f.superArmor, projectileArmor = f.projectileArmor, telegraph = f.telegraph,
		params = f.params, meterGain = f.meterGain or 1,
		anim = if f.behavior == "RealmCast" then `anim.{styleId}.realm` else `anim.{styleId}.{slot}`, -- Part 3 §3.11.8 key for RealmCast
		vfx = `vfx.{styleId}.{slot}`, sfx = `sfx.{styleId}.{slot}`,
	}
end
-- box: shape "Box", size (w,h,d), offset CFrame.new(0, up or 0, -(fwd or d/2)) in the attacker's yaw frame
-- sphere: shape "Sphere", size (d,d,d), offset CFrame.new(0, up or 0, -(fwd or 0))
-- kb: { dist = dist, angleDeg = angleDeg, ragdoll = ragdoll, launch = launch }
-- tele: { shape = shape, leadFrames = lead, size = Vector3.new(x, 0.2, z) }
-- rgb: Color3.fromRGB(r, g, b); v3: Vector3.new(x, y, z)
```

Telegraph `size` semantics:
- `Circle`: x = z = diameter.
- `Line` and `Arc`: x = width, z = length/depth.
- `Cone`: z = length, x = far-end width.

#### 4.1.2 Registry API

```lua
export type StylesApi = {
	get: (id: string) -> Types.StyleDef,                 -- errors "[HB][Style] unknown style <id>"
	tryGet: (id: string) -> Types.StyleDef?,
	all: () -> { Types.StyleDef },                       -- 18 defs in Types.STYLE_IDS order (frozen)
	implemented: () -> { Types.StyleDef },               -- implemented == true and not runtime-disabled
	isPlayable: (id: string, cfg: Types.Config) -> boolean, -- implemented, valid, not in Features.DisabledStyles
	move: (moveId: string) -> Types.MoveDef?,            -- any slot incl. q, a1..a4 (and stub moves)
	hooks: (id: string) -> Kit.PassiveHooks,             -- {} for stubs
	moment: (id: string) -> Kit.MomentDef?,
	anim: (id: string) -> { [string]: Kit.AnimMoveSpec },
	behavior: (b: Types.Behavior) -> Kit.KitBehavior?,   -- nil for "Strike" (Core default, Part 3 §3.4.2)
	variants: (def: Types.StyleDef) -> { [string]: { Types.MoveDef } }, -- kit-mode variants (§4.2.8); deferred: returns {} in M1-G
	forConfig: (cfg: Types.Config) -> KitSet,            -- defs with Kits.Overrides applied (§4.1.6); cached per cfg table
	validate: (cfg: Types.Config) -> (boolean, { Problem }),
	maxDamage: (move: Types.MoveDef) -> number,          -- per-target maximum for one use (§4.1.4)
	passiveStep: (h: Kit.HookCtx, inp: any) -> Kit.HookOut?,                  -- §4.3.1, both peers (runs hooks.onStep; hover deferred, §4.3.4)
	serverHook: (kind: Kit.HookKind, h: Kit.HookCtx, info: Kit.HitInfo?) -> Kit.HookOut?, -- §4.3.1, server
	SFX_SPEED: { [string]: number },                     -- §4.7.3 (Part 6 SfxController reads)
}
export type KitSet = { get: (id: string) -> Types.StyleDef, move: (moveId: string) -> Types.MoveDef? }
export type Problem = { rule: string, style: string, move: string?, detail: string }
```

Semantics:
- `get`/`all`/`move` return **frozen** tables (`table.freeze`, applied recursively at build). Callers that need a modified MoveDef use `table.clone`.
- **Base registry vs pinned kit set.** `Styles.move`/`get` return the base registry (no `Kits.Overrides`); tools, UI and Lune tests use them. Inside the sim, MoveDefs always come from the character's **pinned kit set**: Part 3's `RosterEntry.kitSet = Styles.forConfig(fullCfg)`, where `fullCfg = Runtime.at(cfgVersion)` is frozen for the life (Part 3 §3.11.1). Behaviors, carriers and hooks see it as `moves` (`bc.moves`, `view.moves`, `h.moves`) and the pinned `Kits` section as `kits`; their `cfg` is the pinned **Combat section** only (`fullCfg.Combat`, Part 3 §3.4.2).
- `isPlayable(id, cfg)` is the one gate for equipping. StyleService (Part 5) and BotService (Part 3) use it. It is false for `implemented = false` stubs (§0.10: collectible, not equippable).
- `behavior("Strike")` returns `nil`. Part 3 Core handles plain melee for Strike moves (Part 3 §3.4.2). Every other `Types.Behavior` maps to the same-named module, with `Buff` covering Buff and Debuff moves.

#### 4.1.3 Load sequence

1. **Require.** On first `require`, `Styles/init` requires `Types`, `Config/Defaults` (types only), `Core/Moves`, `Core/Realm`, the nine behavior modules, then the six kit modules via `require("@self/<id>")`. Each kit builder is called with `K`.
2. **Stubs.** Build 12 stub `StyleModule`s from the `STUBS` table (§4.5.8) with `K.stubMove`.
3. **Order.** Assemble `ALL` in `Types.STYLE_IDS` order. Index every move by id into `MOVES`.
4. **Freeze.** Deep-freeze `ALL`, `MOVES`, hooks tables and anim specs.
5. **No validation on require.** `validate(cfg)` runs at boot and in Lune (§4.1.5). This keeps `require` side-effect free.

The module works unmodified in Lune once `tests/shim` installs the globals `Vector3`, `CFrame` and `Color3` from `@lune/roblox` (Part 7). `Styles/*` uses only:
- `Vector3.new`, `Vector3.zero`;
- `CFrame.new(x, y, z)`, `CFrame.Angles`, `CFrame.lookAt`;
- `Color3.fromRGB`.

All are verified in `$XREF/datatypes/{Vector3,CFrame,Color3}.yaml`. Whether `@lune/roblox` provides each one is (verify in spike S-P4-6).

#### 4.1.4 Validation rules

`Styles.validate(cfg)` runs every rule below over all 18 styles and returns `(ok, problems)`. Each problem is logged as:

`[HB][Style] ERROR rule=<id> style=<styleId> move=<moveId|-> detail=<text>`

The tag is `Style`, from Part 2 §2.3.6. Rules S01–S06 apply to all styles. K-rules apply only to `implemented = true` styles. Stubs are checked by S-rules only (§0.10).

Numeric thresholds come from config, not literals:
- Part 3's `Combat.Damage` (`MaxBaseMove` 18, `MaxQuirk` 6, `MaxUltimate` 30, `TellMinFrames` 8, `TellDamageAbove` 10, `CooldownBands`, `Summon`) and `Combat.Block` (`MoveGuardMin` 20, `MoveGuardMax` 60, `MoveChipDefault` 0.2);
- `Kits` (`AwakenedMaxDamage` 20, `AwakenedCooldownBand`, `PerHitGuardBand`), for the bands Part 3 does not define.

The numbers in the table are the GDD values those keys hold.

`cfg` in this section is the full `Types.Config` (the boot validator runs outside the sim), so rules read `cfg.Combat.*` and `cfg.Kits.*`. Calls into Part 3's pure modules pass `cfg.Combat`, the shape Part 3 §3.4.2 expects.

`maxDamage(move)` is the per-target maximum damage for one use. It is computed by the behavior's `maxDamage` (§4.2):
- single hit: `damage`;
- `params.hits` / `params.ticks`: their sum;
- echo (deferred, §4.2.2): `damage + echo.damage`;
- tick zones: `min(damage, tick.damage × maxHitsPerTarget)`;
- summons with `dmgModel = "perHit"`: `damage × maxHitsPerTarget × bodies`;
- counters: `damage`.

| ID | Scope | Check | Source |
|---|---|---|---|
| S01 | registry | Exactly 18 defs; ids equal `Types.STYLE_IDS`, same order | §0.4 |
| S02 | all | `rarity` equals the §4.5.8 table | GDD §7.2, §8.3 |
| S03 | registry | `implemented == true` exactly for `attendant, lifeguard, sandcastle, scoop, busker, dustsheet`; equals `Types.IMPLEMENTED_M1G` | §0.0, §4, §10 |
| S04 | all | `hp == 100` | GDD §6.2 |
| S05 | all | `archetype` matches `^%u%a+$`; palette has three `Color3` values | §4.7.1 |
| S06 | implemented | Primary colours pairwise Euclidean RGB distance ≥ 40 (0–255 space) (td) | GDD §12.3 readability |
| K01 | implemented | Quirk id `<id>.q`. `moves` are exactly slots `"1".."4"` in order with ids `<id>.<slot>`. Awakening id `<id>.awaken`; `awakening.moves` are slots `a1..a4` in order. Passive id `<id>.passive`. All move ids unique | §0.4 |
| K02 | implemented | Exactly one base move has `signature == true`; quirk and Peak moves do not | GDD §6.7 |
| K03 | implemented | ≥ 1 opener: a base move tagged `Projectile`, `Rush` or `Grab` | GDD §6.7 |
| K04 | implemented | ≥ 1 extender, either a base move tagged `Extender`, or one with `params.extender == true` **and** `startup + 1 + travelF ≤ m1Adv`. `travelF = ceil(5 · 60 / params.speed)` for bullets (5 studs = M1 reach, td), 0 for melee and beams. `m1Adv = hitstun − (active − 1) − recovery` of M1-3 from `cfg.Combat.M1` (= 27 − 2 − 13 = 12, GDD §6.3) | GDD §6.7 |
| K05 | implemented | ≥ 1 defensive/mobility tool among base moves + quirk: tag `Counter`, `Mobility` or `Zone`, or `params.iFrames ~= nil` | GDD §6.7 |
| K06 | implemented | `maxDamage`: base ≤ 18; quirk ≤ 6; a1–a3 ≤ 20; a4 ≤ 30 | GDD §6.7, §7.1 rule 4 |
| K07 | implemented | Cooldown bands: signature 14–22; other base 6–16 (basic 6–10 ∪ utility 10–16); quirk 3–8 (≤ 12 if `params.random == true`); a1–a3 6–22 (td); a4 = 25 if tagged Ultimate, 0 if `RealmCast` | GDD §6.7, §7.3 |
| K08 | implemented | If `maxDamage > 10`, or any single hit > 10: `startup ≥ 8`, `telegraph ~= nil`, `telegraph.leadFrames ≥ 8` and `≤ tellFrames(move)`, and `tellFrames(move) ≥ 8` | GDD §6.14 #1, §7.1 rule 5 |
| K09 | `Summon` moves | `bodies ≤ 3`; `bodies × hpEach ∈ [20, 60]`; `lifeF ∈ [360, 720]`. Exception: slot a4, tag `Ultimate`, `params.untargetable == true` → only `bodies ≤ 3` and `lifeF ≤ 360` apply (an untargetable set piece has no HP) | GDD §6.7 Summon |
| K10 | awakening (Peak) | `hurtboxScale == nil or == 1` (no Peak scales the hurtbox); `durationSec == cfg.Combat.Awakening.DurationSec` (60) | GDD §6.2, §6.9; §0.4 |
| K11 | awakening (Peak) | `modifiers` keys ⊆ Part 3 recognized keys (`breakawayRefillMult`, `enderDamageAdd`, `ragdollFramesAdd`, `m1RangeAdd`, `walkSpeedMult`, `airDash`) | §4.3.3, §4.4.2 |
| K12 | a4 | Rarity ∈ {legendary, mythic, cosmic} ⇔ `behavior == "RealmCast"` and tags contain `Realm`. Otherwise tags contain `Ultimate` | GDD §6.10, §7.1 |
| K13 | a4 Ultimate | `params.cinematicFrames` (default 0) ≤ 120 | GDD §6.8 |
| K14 | damaging moves | If `blockable(move)`: `guardDamage ∈ [20, 60]` (model `total`) or `∈ [8, 30]` (model `perHit`, td). Otherwise `guardDamage == 0` | GDD §6.4 |
| K15 | damaging moves | `chip == 0.2` if `blockable(move)`, else `chip == 0` | GDD §6.4 |
| K16 | all moves | `startup ≥ 1`, `active ≥ 1`, `recovery ≥ 0` (integers); total ≤ 600; `hitstun` integer 0–180; damaging moves `startup ≥ 6` | GDD §6.1, §18.2 |
| K17 | all moves | `anim/vfx/sfx == "<kind>.<style>.<slot>"` (RealmCast `anim == "anim.<style>.realm"`, Part 3 §3.11.8), matching the Assets key pattern | Part 2 §2.9.1 |
| K18 | all moves | `Moves.validate(def, slot, cfg.Combat)` (Part 3) returns no problems; `behavior(b).validate(def, cfg)` (full config) returns no problems | §4.2 |
| K19 | all moves | Tags ⊆ `Types.Tag`. Behavior↔tag: Grab⇒`Grab`; Counter⇒`Counter`; Summon⇒`Summon`; RealmCast⇒`Realm`; Projectile⇒`Projectile`; Zone⇒`Zone` or `AoE`; Rush⇒`Rush`; Mobility⇒`Mobility`; Buff⇒ one of `Buff`/`Debuff`/`Utility`/`Mobility`; Strike⇒ any | GDD §6.7 |
| K20 | implemented | `hitbox ~= nil` for Strike, Rush and Grab, except Grab with `params.sweep` | §4.2 |
| K21 | multi-hit | `Σ params.hits` and `Σ params.ticks[i].damage` each equal `damage` (± 1e-6) | §4.5.1 |
| K22 | knockback | `0 ≤ dist ≤ 60`; `−90 ≤ angleDeg ≤ 90` | (td) |
| K23 | implemented | `m1 = {anim = "anim.<id>.m1", vfx = "vfx.<id>.m1", sfx = "sfx.<id>.m1"}`; every `kitProps[i]` matches `^model%.<id>%.gear_%l+$` or `^model%.<id>%.summon_%l+$`; exactly two `gear_` entries (the oversized prop, then the head piece); every Summon move's `params.model` is in `kitProps` | §4.7, §0.8 |
| K24 | implemented | `anim` has specs for `m1` (12-entry `alt`), `q`, `1..4`, `a1..a4`, `awaken`; every pose name exists in the pose library (§4.6.2) | §4.6 |
| K25 | overrides | Each `Kits.Overrides` record names an existing move and an allowed field; the overridden move passes K06–K16 | §4.1.6 |

**Verify:**
- Lune `styles.registry_18`, `styles.ids_match_types`, `styles.rarities_match_gdd`, `styles.implemented_set` (also covers Part 2's `types.implemented_matches_styles`), `styles.validate_all_clean` (`validate(Defaults)` returns `true, {}`).
- One negative test per rule, `styles.rule_<ID>_detects`. Each clones one valid kit, mutates one field (e.g. `attendant.2` damage 19), and asserts the rule id appears in the problems.

#### 4.1.5 Boot behavior

- **Server.** `StyleService.init()` (Part 5 owns the service; this call is binding) runs `Styles.validate(Runtime.get())` and logs each problem.
  - In Studio, any problem calls `error("[HB][Style] kit validation failed")`. Boot then fails loudly (Part 2 §2.3.4).
  - Outside Studio, each implemented style with a problem goes into a runtime-disabled set (so `isPlayable` is false). If `attendant` or `lifeguard` fails, it errors (no fallback style exists).
  - On success: `[HB][Style] registry ok styles=18 implemented=6 moves=<n> problems=0`.
- **Client.** It does not validate (same code). It logs `[HB][Style] registry loaded styles=18` once.
- **Config change.** On `Runtime.changed`, the server re-validates `forConfig(newCfg)`. If a `Kits.Overrides` record fails, ConfigReplicator rejects that section (Part 2 §2.6.4 step 3) with `[HB][Config] ERROR rejected section=Kits problems=<…>`.

**Verify:** scenario `style.registry_boot` expects the `registry ok` line within 5 s of play start, and zero `[HB][Style] ERROR` lines.

#### 4.1.6 `Kits` config section (new top-level key, §4.9 item 3)

Kit numbers live in code, not config: StyleDefs contain `Vector3`/`Color3`, which are not JSON-safe (Part 2 §2.6.1). `Kits` gives live-ops a JSON-safe override layer and the carrier caps. It is **pinned with the combat config** (Part 2 §2.6.6, Part 3 §3.11.1): at spawn Part 3 resolves `fullCfg = Runtime.at(CombatCfg)` (the full snapshot, frozen for the life) outside the sim and stores it with `kitSet = Styles.forConfig(fullCfg)` in the `RosterEntry`. Core receives only `fullCfg.Combat`. Behaviors, carriers and hooks also get `kits = fullCfg.Kits` and `moves = kitSet.move` (§4.2.0.2, §4.2.0.3, §4.3.1). `fullCfg.Features.DisabledMoves` feeds Part 3's per-move kill switch (`StyleRuntime.disabled`, `Moves.canUse`; Part 3 §3.4.2). Sim code never reads `Runtime.get()` or a bare `Kits.X`.

```lua
Kits = {
	Overrides = {},              -- array of { move = "attendant.3", field = "damage", value = 5 } (td: empty)
	OverrideFields = { "damage", "guardDamage", "chip", "hitstun", "cooldown", "startup", "active", "recovery", "meterGain" },
	MaxCarriersPerFighter = 12,  -- (td) oldest owned carrier despawns when exceeded
	MaxCarriersServer = 200,     -- (td) spawn refused (logged once per 10 s) when exceeded
	MomentCooldownFrames = 600,  -- (td) one Snapshot per attacker per 10 s
	StatusStayRefreshFrames = 6, -- (td) zone "stay" debuffs are re-applied every 6 f
	AwakenedMaxDamage = 20,      -- GDD §6.7 / §7.1 rule 4 (Peak moves 1–3)
	AwakenedCooldownBand = { 6, 22 }, -- (td) a1–a3 cooldown sanity band (GDD lists 8–16)
	PerHitGuardBand = { 8, 30 }, -- (td) guard damage per hit for "perHit"-model moves (summon pokes, orbit touches)
}
```

- `Overrides` is an **array** so that Part 2's `Schema.merge` accepts additions: an empty base array accepts any elements (Part 2 §2.6.3 rule 5).
- `forConfig(cfg)` applies records in order to deep copies. The result is frozen and cached in a weak-keyed table indexed by the frozen `cfg` table.
- ConfigService key `HB_Kits` overlays `Kits` and takes effect at the next spawn (pinning, Part 2 §2.6.6).
- Schema rules:
  - `Kits.Overrides[*].move` matches `^%l+%.[%w]+$`;
  - `field ∈ OverrideFields`;
  - `value` finite in `[0, 600]`;
  - `MaxCarriersPerFighter` int 1–32;
  - `MaxCarriersServer` int 16–512;
  - `AwakenedMaxDamage` num 1–30;
  - `AwakenedCooldownBand` and `PerHitGuardBand` are 2-element num arrays with `[1] ≤ [2]`.

**Verify:** Lune `styles.overrides_apply` (override `attendant.3.damage = 5` → `forConfig(cfg).move("attendant.3").damage == 5`; the base registry still says 6). Lune `styles.overrides_invalid_rejected` (`attendant.2.damage = 19` → `validate` reports K06). Scenario `config.kits_override_next_life` (`Dev.Command setConfig Kits`; the current life keeps the old value; after respawn the new damage applies).

#### 4.1.7 Lune specs owned by this section

`tests/core/styles.spec.luau`: every test named in §4.1.4–§4.1.6, plus:
- `styles.move_ids_unique`;
- `styles.frozen` (assigning to `Styles.get("attendant").hp` errors);
- `styles.behavior_strike_nil`.

---

### 4.2 Behavior modules (`src/Shared/Behaviors/*`)

#### 4.2.0 Common contract

##### 4.2.0.1 What a behavior is

A behavior turns one `MoveDef` into commands, frame by frame. It is a table of pure functions over Part 3's types. Part 3's adapter calls them on these paths:

| Function | Called by (Part 3) | Peers | When |
|---|---|---|---|
| `step(bc) -> BehaviorOut?` | `Fsm.step` (state tick) | Owner client + server | Every frame the owner is in `St = Move` |
| `onContact(bc, result) -> BehaviorOut?` | Hit resolution | Server always; owner client for BQ predictions | After a HitRequest from this move's own `step` resolves on a target. Never for carrier hits (`req.carrier` set): they change no owner state (Part 3 §3.12.6, §4.2.0.5 item 7) |
| `onEnd(bc) -> BehaviorOut?` | State exit | Both | On Move exit, including interrupts and grab breaks |
| `intercept(bc, hit, srcPos) -> Intercept?` (ext.) | Server hit pipeline, **before** `Damage.resolve` | Server | The victim is in `Move` and its def's behavior defines `intercept` |
| `stepCarrier(view, f) -> CarrierOut?` (Part 3 name, §3.11.5) | Carrier loop (phase 1b) | Server; owner client for its own predicted carriers (Part 3: `Own == localRid`), where hits are ignored | Every frame per live carrier whose MoveDef uses this behavior |
| `hurtCarrier(view, hit, attackerRid) -> CarrierOut?` (ext.) | Hit resolution | Server | A targetable summon carrier is hit |
| `reflectCarrier(view, newOwnerRid) -> CarrierOut?` (ext.; **deferred**, no M1-G user) | Hit resolution | Server | A projectile carrier hits a reflecting Counter (§4.2.5) |
| `validate(def, cfg)` (`cfg` = full `Types.Config`), `maxDamage(def)`, `tellFrames(def)`, `blockable(def)` (ext.) | `Styles.validate` | Lune + boot | Pure checks (§4.1.4) |

"(ext.)" marks an extension to Part 3's `BehaviorModule`, requested in §4.9 item 4.

##### 4.2.0.2 Inputs and outputs (Part 3 types, used verbatim)

- `BehaviorCtx = { f, e, phase, def, s, ctx, cfg, kits, moves, isServer }` (Part 3 §3.4.2).
  - `bc.cfg` is the pinned **Combat section** (`fullCfg.Combat`), so behaviors read `bc.cfg.Chassis.X`, `bc.cfg.Damage.X`, never `cfg.Combat.X`.
  - `bc.kits` is the pinned `Kits` section (`fullCfg.Kits`, §4.1.6); `bc.moves(moveId)` is the pinned kit set's `move` (overrides applied). Behaviors never read a bare `Kits.X` or `Runtime`.
  - `bc.ctx.rootCf` is the root at step start.
  - Facing follows aim during startup and holds from the first active frame (Part 3 §3.2.7).
  - Part 4 additionally needs `bc.inp: Fsm.InputFrame` (aim, moveDir, bits) and `bc.ray(origin, dir) -> (Vector3?, Vector3?)`. `ray` is a static-world raycast in collision group `WorldQuery` that excludes `workspace.Sim` (§4.9 item 5).
- `BehaviorOut = { motion?, hits?, spawns?, patch?, events? }`.
  - `patch` may write the owner's `MvA`, `MvB`, `Kit`, `Buffs`, `StDur` and `Cd1..Cd4/CdQ`. All are B or B·S class in Part 3 §3.3.1.
  - Part 3's Move entry effects set `MvA = MvB = 0` (requested in §4.9 item 15).
- `HitRequest = { key, box: HitboxReq, hit: HitDef, maxTargets }`. Behaviors build `hit` with Part 3's `Moves.hitDef(def, bc.cfg)` (Combat section) and then adjust per-hit fields. `key` is `"<moveId>#<n>"`; Part 3 scopes the registry to the current action instance.
- `SpawnRequest = { kind, moveId, origin, params }`. `params` are JSON-safe carrier attributes (§4.2.0.5).

Part 4's extension fields on Part 3's types (§4.9 items 6 and 7) are declared by Part 3 in the Combat section of `Types.luau` (Part 3 §3.4.1). `HitDef.kit` is typed `any?` there, so `Types` stays a leaf; its shape is `KitHitExt` below (in `Behaviors/Buff.luau`):

```lua
-- Fsm.HitDef.kit: any? (= KitHitExt)          Fsm.HitboxReq.coneDeg: number?  (Sphere only)
-- Fsm.HitboxReq.innerRadius: number?          (Sphere only: skip targets whose planar distance from the centre is below it)
-- Fsm.HitRequest.heldOnly: boolean?           (targets = victims currently Grabbed by this attacker)
export type StunFlavor = "hit" | "freeze" | "float" | "stumble" | "possess" | "flinch"
export type StatusReq = { kind: string, frames: number, n: number }        -- relative duration; Part 3 adds f
export type KitHitExt = {
	stun: StunFlavor?,                               -- Hitstun Sub flavour (table below)
	debuffs: { StatusReq }?,                         -- added to the victim's Debuffs on outcome "hit" (Moves.statusAdd)
	pull: { stopStuds: number, frames: number }?,    -- one KbV write toward the attacker on outcome "hit"; victim pushes itself (B), see "Pull" below
	meterDrain: number?,                             -- victim server meter −n (floor 0); busker.3 (Peak Meter)
	healSelf: number?,                               -- attacker HP +n on outcome "hit" (no slice user)
	grab: { holdF: number, sub: number, offset: Vector3 }?, -- catch: victim → Grabbed (Sub = sub) for holdF + 2 f
	release: boolean?,                               -- this hit ends the victim's Grabbed, then applies hitstun/knockback
	reflectable: boolean?,                           -- carrier projectile hits only
	firstOfUse: boolean?,                            -- Part 3 fills: first landed hit of this use on this victim
}
```

Stun flavours map to `Hitstun` Sub codes (§4.9 item 8). All of them obey Stun Decay, OTG, Shake-Off and Breakaway like normal hitstun (GDD §6.14 #3). The slice kits use `hit`, `stumble` and `flinch`. `freeze`, `float` and `possess` keep their enum values and Sub codes; they have no M1-G user, so their motion, render and tests are deferred to the first style that needs them (§4.9 item 26):

| Flavour | `Hitstun` Sub | Motion while stunned | Render | Slice users |
|---|---|---|---|---|
| `hit` | 0/1 (Part 3) | Part 3 push | normal reaction | most moves |
| `freeze` | 2 | `freeze` | held still in a frame (`vfx.status.freeze`, deferred) | none (deferred) |
| `float` | 3 | rise 4 studs over the first 12 f, then hold (`vector`) | lifted pose | none (deferred) |
| `stumble` | 4 | Part 3 push along `KbV` | backward stumble | `scoop.2` finish, `scoop.a2`, `dustsheet.3` |
| `possess` | 5 | Part 3 push along `KbV` for the whole `StDur` (the pull, below) | tether line | none (deferred) |
| `flinch` | 6 | none | small flinch | `attendant.q`, `busker.q`, `busker.2`, `busker.a3`, `scoop.q` |

**Pull (`kit.pull`).** Server Authority rule: the server writes the cause once and both peers derive the motion (Part 3 §3.1.2).
- **Cause (S, once).** On outcome `"hit"` only (§4.2.0.4), in the server consequence phase, the server writes the victim's `KbV = d.Unit · (max(dist − stopStuds, 0) · 60 / frames)` with the Q-rule, where `d = Buff.flat(attRoot − vicRoot)` and `dist = d.Magnitude` (roots at resolution time). If `d` is zero, `KbV = Vector3.zero` (no pull). This replaces the hit's own knockback push, so moves with `kit.pull` carry no `knockback`.
- **Effect (B, every frame).** The victim's FSM moves it at constant `KbV` with Part 3's existing **Push** profile (Part 3 §3.7, §3.8.5) for `N` frames: `N = StDur` for the `possess` flavour (`Hitstun` Sub 5), else `N = min(StDur, Stun.MoveKbFramesMax)` (12). Both peers derive `N` from replicated state, so after the one rollback the victim's client predicts the rest.
- **Data rule.** `pull.frames` equals that `N` for the move's own hitstun: a move with hitstun 34 uses `frames = 12` (`min(34, 12)`); a `possess` move uses its full hitstun. Projectile `validate` checks it (§4.2.3).
- **Users.** No slice move uses `kit.pull`. The cause/effect rule stays in the behaviour library for later kits; Lune `behaviors.projectile_pull_frames_rule` checks the data rule on a synthetic MoveDef.
- Zone pull (`params.pullStudsPerSec`) is a different, per-frame B-class effect (§4.2.6).

##### 4.2.0.3 Kit kernel (`Behaviors/Buff.luau` exports)

`Buff.luau` holds the shared Part 4 types and helpers, because the Part 0 layout has no extra file (§0.3). Every other behavior and every kit file requires it. Buff requires only `Types`, `Core/Fsm`, `Core/Moves`, `Core/Frames` and `Core/Movement` (pure Part 3 modules), so no cycle exists.

```lua
export type Phase = "startup" | "active" | "recovery"
export type Intercept = {
	verdict: "pass" | "absorb" | "reflect",
	reaction: Fsm.HitDef?,          -- applied by Part 3 to the attacker (melee counters)
	patch: { [string]: any }?,      -- defender B·S fields (e.g. StDur for endOnTrigger)
}
export type FighterLite = { rid: number, cf: CFrame, team: number, styleId: string, st: number, airborne: boolean }
-- `view` argument of Part 3's `Behaviors.<B>.stepCarrier(view, f)` (Part 3 §3.11.5). Part 3 builds it per carrier per step.
export type CarrierView = {
	name: string,                     -- "C_<rid>_<f>_<n>" (Part 3; read from attribute Cid); also the hit-registry key prefix
	kind: "projectile" | "zone" | "summon",
	moveId: string, def: Types.MoveDef,  -- static data comes from the MoveDef (= view.moves(moveId), the owner's pinned kit set)
	own: number, f0: number, age: number, -- attributes Own, F0; age = f − F0 (0 on the spawn step)
	cf: CFrame, hits: number,          -- carrier CFrame; attribute Hits
	p: { [string]: number | string | boolean }, -- dynamic per-carrier attributes (≤ 10 keys, Part 3 limit)
	isServer: boolean,
	cfg: any,                          -- the owner's pinned Combat section (fullCfg.Combat, Part 3 §3.4.2)
	kits: any,                         -- the owner's pinned Kits section (fullCfg.Kits, §4.1.6)
	moves: (moveId: string) -> Types.MoveDef?, -- the owner's pinned kit set (RosterEntry.kitSet.move)
	fighters: { FighterLite },        -- live fighters, ascending rid (server roster; client: local + replicated)
	ownerCf: CFrame?, ownerAim: Vector3?, ownerAlive: boolean,
	ray: (origin: Vector3, dir: Vector3) -> (Vector3?, Vector3?), -- static world (WorldQuery), excludes workspace.Sim
}
export type CarrierOut = {
	cf: CFrame?, set: { [string]: number | string | boolean }?,
	hits: { Fsm.HitRequest }?,        -- attacker = view.own (credit only); Part 3 sets req.carrier; HitboxReq.prevCf for swept casts; ignored on the client
	dead: boolean?,                   -- Part 3 sets Dead = true in the sim; destroyed in the deferred flush (≥ 2 frames later)
	newOwner: number?,                -- reflectCarrier only: Part 3 rewrites Own
	ownerHeal: number?, events: { Fsm.SimEvent }?,
}
export type KitBehavior = {
	name: Types.Behavior,
	step: (bc: any) -> any, onContact: ((bc: any, result: any) -> any)?, onEnd: ((bc: any) -> any)?,
	validate: (def: Types.MoveDef, cfg: Types.Config) -> { string }, -- boot validator: cfg is the FULL config (§4.1.4)
	maxDamage: (def: Types.MoveDef) -> number,
	tellFrames: (def: Types.MoveDef) -> number, blockable: (def: Types.MoveDef) -> boolean,
	intercept: ((bc: any, hit: Fsm.HitDef, srcPos: Vector3) -> Intercept?)?,
	stepCarrier: ((view: CarrierView, f: number) -> CarrierOut?)?,              -- Part 3 name (§3.11.5)
	hurtCarrier: ((view: CarrierView, hit: Fsm.HitDef, attackerRid: number) -> CarrierOut?)?,
	reflectCarrier: ((view: CarrierView, newOwnerRid: number) -> CarrierOut?)?,
}
-- (bc/result typed with Part 3's BehaviorCtx / BehaviorOut / Damage.HitResult in code)
```

Helpers (pure):

| Helper | Contract |
|---|---|
| `Buff.flat(v: Vector3): Vector3` | `Vector3.new(v.X, 0, v.Z)`; returns `Vector3.zero` when the magnitude < 1e-4 |
| `Buff.yawCf(rootCf: CFrame): CFrame` | `CFrame.new(rootCf.Position) * CFrame.Angles(0, yaw, 0)`, with `yaw` from `Movement.yawOf(rootCf.LookVector)` (Part 3) |
| `Buff.hitN(bc, n: number, dmg: number, over: {[string]: any}?): Fsm.HitDef` | `Moves.hitDef(bc.def, bc.cfg)` cloned (`bc.cfg` = Combat section). `damage = dmg`. `guardDamage = if maxDamage(def) > 0 then round1(def.guardDamage × dmg / maxDamage(def)) else 0` (`round1` = round to 0.1; zero-damage moves such as dustsheet.q's front request get 0, never NaN). Fields in `over` are copied last |
| `Buff.req(key: string, box: Fsm.HitboxReq, hit: Fsm.HitDef, maxTargets: number): Fsm.HitRequest` | Constructor |
| `Buff.once(bc, bit: number): (boolean, number)` | `(MvA & bit) == 0`, plus the new `MvA` with the bit set (`bit32`) |
| `Buff.aimPoint(bc, range: number): Vector3` | `p = root + flat(bc.inp.aim).Unit × range` (fallback `rootCf.LookVector`). Then `bc.ray(p + (0, 20, 0), (0, −60, 0))`. Returns the hit position, or `p` if nothing is hit. Q-rule rounded to 0.01 |
| `Buff.statusPatch(s: Fsm.CharState, add: { StatusReq }, f: number): string` | `Moves.statusEncode` of `Moves.statusAdd` applied over `Moves.statusDecode(s.Buffs)`, with `untilF = f + frames` |
| `Buff.statusRemove(encoded: string, kind: string): string` | Decode, drop entries of `kind`, encode |
| `Buff.appliesStatus(def: Types.MoveDef, kind: string): boolean` | True if the move itself can add `kind` to a victim: `params.onHit.debuffs`, `params.debuffs` (front Buff), `params.stay` or any `params.ticks[i]`/`bursts[i]` status names it. Used by the `scoop` passive (§4.3.6) |

##### 4.2.0.4 Hits, keys and one-shots

- **Key format.** `"<moveId>#<n>"`. `n` is the hit index within the use (1-based). The echo, finish and tick hits continue the numbering. The same key is never reused within one use.
- **Hit per frame.** A behavior emits a hit's request on every active frame where it may connect. Part 3's registry guarantees each target is hit at most once per key.
- **Multi-hit timing.** `params.hitFrames[i]` is the active-relative frame (1-based) of hit `i`. Hit `i`'s request (key `#i`) is emitted on every frame with `hitFrames[i] ≤ e − S + 1 < hitFrames[i] + hitWindow` (`params.hitWindow`, default 3, td). Part 3's registry stops a second contact under the same key. No scratch state is needed, and a skipped frame still leaves 2 frames in the window (P4-R3).
- **First-of-use.** Part 3 fills `hit.kit.firstOfUse`. Passive bonuses that apply "once per use per target" read it (§4.3.3).
- **Blocked outcomes.** Statuses (`kit.debuffs`), pulls, meter drain and self-heal apply **only** on `Damage.Outcome` `"hit"` (and `"interrupt"`, which continues as a hit; Part 3 §3.12.6). On `"block"`, `"guardbreak"`, `"perfect"` and `"armor"` they do not apply (td).

##### 4.2.0.5 Carriers (projectiles, zones, summons)

Lifecycle. Part 3 implements spawning, stepping and destruction (Part 3 §3.11.5); Part 4 defines the content.

1. **Spawn.** A behavior returns `SpawnRequest{kind, moveId, origin, params}` from `step`. Part 3's `Adapter.spawn` clones the character's `HB_Proto` **inside the sim**, so it is stitched on both peers for the owner. It sets `Name = C_<rid>_<f>_<n>` and the attributes `Own, Kind, MoveId, F0, Dead, Hits, Cid` (`Cid` = the name; the sim reads `Cid`, because `Name` is not Simulation-Access, Part 3 §3.11.5) plus `params`.
   - **Server-only spawns.** Summons and homing projectiles depend on other characters' positions, so their SpawnRequest carries `serverOnly = true` (§4.9 item 9). Part 3 takes its server-only creation path for them.
   - **Targetable summons.** For `hp > 0`, Part 3 also sets on the carrier: `CanQuery = true`, `CollisionGroup = "Hurtbox"` and `Size = params.hurtbox` (default 3×5×3, td). The carrier part **is** the summon's hurtbox (Part 3 §3.12.1 lists summon hurtboxes in the `Hurtbox` group).
2. **Static vs dynamic data.**
   - Static configuration (speeds, radii, ticks, telegraph shape) is **never** copied into attributes. `stepCarrier` reads it from `view.def.params` (the MoveDef found via the `MoveId` attribute in the owner's pinned kit set, `view.moves`), identically on both peers.
   - `params`/`set` hold only per-instance dynamic values, **≤ 10 keys** (Part 3 limit), names ≤ 12 alphanumeric characters. The attribute names of item 1 (`Own, Kind, MoveId, F0, Dead, Hits, Cid`) are reserved and are never a `params`/`set` key.
   - Every carrier includes `lifeF` (total frames). Part 3 caps it with `Chassis.Sim.MaxCarrierLifeFrames` = 780 (the GDD 12 s summon maximum plus a 60 f arm margin, Part 3 §3.11.5), so no GDD-valid summon (K09: `lifeF ≤ 720`) or zone is cut short.
3. **Step.** `Behaviors.<Behavior>.stepCarrier(view, f)` runs in phase 1b:
   - on the server for every live carrier, in name order (Part 3 enumerates `workspace.Sim` and sorts by `Cid`, §3.11.5);
   - on the owner client for its predicted carriers (`Own == localRid`), where only `cf`/`set` apply and hits are ignored.

   The dispatch module is the MoveDef's behavior, with one exception: a Rush echo carrier (`Kind = "zone"`, `sub = 1`; deferred, §4.2.2) is stepped by `Zone.stepCarrier`.
4. **Death.** `dead = true` makes Part 3 set `Dead = true` in the sim. The server destroys the carrier in the deferred flush, ≥ 2 frames later, **outside** the sim (Part 3 §3.1.5).
5. **Caps.** Read from the spawning owner's pinned `Kits` (`RosterEntry.fullCfg.Kits`, the same table as `bc.kits`): `MaxCarriersPerFighter` evicts the oldest owned carrier; `MaxCarriersServer` refuses the spawn. Server log via the flush queue: `[HB][Style] carrier cap owner=<rid> evicted=<name>`.
6. **Owner KO or untrack.** Part 3 marks every carrier with that `Own` as `Dead` on the next server step (td). This prevents posthumous damage.
7. **Hit bookkeeping.**
   - Carrier hit requests use keys `<view.name>#<suffix>`; Part 3 keeps carrier registries with the carrier (Part 3 §3.12.5).
   - Zone ticks use one suffix per tick (`#t<j>`), so a target can be hit once per tick.
   - Per-target caps (`maxHitsPerTarget`) are counted in **server memory** keyed by carrier name and cleared on death. This is legal per Part 3 §3.1.8: carrier hits are S-class consequences.
   - **No owner state.** Part 3's adapter tags every carrier hit with `req.carrier` in phase 1b. Such hits credit the owner (`Own`) but change none of its state: no `Cnt`, no hitstop or `StF` shift, no Uppercut follow, no `onContact` call. Only server-memory meta applies (meter dealt, ledger, `lastHit`), plus the owner's `CombatF = f` when `f − CombatF ≥ 60` (td). Victim effects, including victim hitstop, apply as usual (Part 3 §3.12.6).

Rendering: the client renders a carrier from its `Kind`, `MoveId` (→ `def.vfx` or `params.model`) and dynamic attributes. Telegraphs come from §4.2.0.6.

##### 4.2.0.6 Telegraph emission

Telegraphs come from two sources. Both render on the dedicated telegraph layer in the user's telegraph colour, at 4.5:1 outline contrast, and are never reduced by Reduced VFX (GDD §13.4, §16.3; Part 6).

| Source | When | Anchor |
|---|---|---|
| `MoveDef.telegraph` (`params.teleAnchor` absent or `"self"`) | From move frame `S − leadFrames + 1` to the first active frame | With `params.teleFwd`: a point `teleFwd` studs ahead of the root position the renderer saw at Move entry, along the current facing (leap landing spots such as `lifeguard.2`). Otherwise the owner's root: the hitbox centre if `hitbox` is set, else `size.z/2` studs ahead. It follows the facing during startup (Part 3 §3.2.7) |
| `params.teleAnchor = "carrier"` | From carrier spawn while `age < armF` (zones) or `age < flightF` (lobs). Summon attacks show while `age − a0 < attack.startupF` | Carrier `cf` (lobs: landing point `lx,ly,lz`). Shape and size come from `MoveDef.telegraph` (summon attacks: an `Arc` sized from `attack.size`) |

Render-side carrier age: for carriers the local player owns, `age = fNow − F0`; for carriers not owned by the local player, RenderController uses the observation age `ageR = fNow − seenF(carrier)` (Part 6 §6.6.1) in these `age` windows, so a remote carrier's telegraph is shown for its full length.

`tellFrames(def)` per behavior is the number of frames between the earliest visible tell and the first frame that can damage:
- self-anchored moves (Rush, Grab, Strike, beams, bullets, waves, homing, instant self zones): `startup`;
- `self`/`selfFollow` carrier zones: `startup + armF` (the self-anchored MoveDef telegraph shows during startup, then the carrier's);
- `aim` zones: `armF` (the point is only known at spawn);
- lobs: `flightF`;
- chase summons (and marked summons, deferred): `attack.startupF`;
- line summons: `startup`.

K08 uses it (§4.1.4).

##### 4.2.0.7 Presentation hooks

- **Move start.** Part 3's `Adapter.animate` plays `def.anim`. Combat animations are played only there, inside `RunService:BindToAnimation(fn, Enum.StepFrequency.Hz60)` on both peers (Part 3 §3.11.8): never on RenderStepped, never gated on `RunService:IsResimulating()`, and AnimationTracks are never cached. RenderController (Part 6) sees `St = Move` with `Sub` (Part 3 §3.2.1) and fires `def.vfx`/`def.sfx` at the data frames: `e ≥ S`, and `e ≥ S + hitFrames[i] − 1` for barrage hit `i` (one-shots, P4-R3). VFX/SFX are render-side (RenderStepped) and skipped while `RunService:IsResimulating()`. Their timing comes from MoveDef frame data, never from clip markers. Clip markers exist only for the §4.6.4 timing-lock check.
- **Hit and carrier effects** are keyed on attributes: the attacker's `Cnt` (melee hits from the owner's `step` only; carrier hits never set the owner's `Cnt`, §4.2.0.5 item 7), victim `St`/`Sub`/`HP`, carrier `vis`/`tS`.
- **Undoable.** Effects started for predicted transitions follow Part 3 §3.1.7 (undo within 15 f).

##### 4.2.0.8 Interaction rules (all behaviors)

| Rule | Implementation |
|---|---|
| `superArmor` | Implemented by Part 3 (`Damage.resolve` row 8, Part 3 §3.12.6): during `params.superArmorFrames` (default startup + active), hits deal damage and meter only, with no stun or knockback. No slice move uses it |
| `projectileArmor` | Part 3 row 8: during startup, only vs `HitDef.kind == "projectile"`. No slice move uses it |
| Parry | `parryable = true` → Perfect Block negates the hit (Part 3 §3.6.3). Grabs are parryable only when flagged |
| Block chip | Blockable damaging moves: `chip = 0.2` (GDD §6.4). Multi-hit chip is per hit. Zone ticks are unblockable unless `tick.blockable == true`; zone bursts are blockable unless `b.blockable == false` (§4.2.6) |
| Reflect | **Deferred** (no M1-G user). Carrier projectiles keep the `reflectable` flag (`refl`); the reflect path (§4.2.5) is built with the first reflecting style. Beams, waves and lobs are never reflectable |
| Grabs | Ignore block and Counter stances; miss i-frames, airborne victims (`groundOnly = true`) and Perfect Block if parryable (GDD §6.4) |
| Summons | Take 2× M1 damage (GDD §6.7). `Summon.hurtCarrier` multiplies M1-family hits (`m1`, `ender`, `airm1`, `dashm1`) by `view.cfg.Damage.Summon.M1DamageMult` (2; `view.cfg` is the Combat section) |
| Statuses | Applied only on outcome `"hit"` (§4.2.0.4) |

#### 4.2.1 Strike (Core default, no module)

`behavior = "Strike"` uses Part 3's plain-melee path: `Moves.hitbox(def, ctx)` and `Moves.hitDef(def, cfg)` every active frame, each target once per use (Part 3 §3.4.2).

Part 4 rules for Strike moves:
- `hitbox` is required (K20).
- `params` may contain only Part 3-recognized keys: `blockstun`, `ragdollFrames`, `hangFrames`, `iFrames`, `air`, `cancelInto`, `cancelWindow`, `cancelChainIndex`, `wallSplat`, `cinematicFrames`.
- No slice kit uses Strike. Stub moves are inert Strike moves (§4.5.8), and later kits may use it for plain melee.

**Verify:** Lune `behaviors.strike_registry_nil` (`Styles.behavior("Strike") == nil`) and `behaviors.strike_stub_inert` (a stub move validates and never reaches the sim, because stubs are not playable).

#### 4.2.2 Rush (`Behaviors/Rush.luau`)

The user travels and strikes. Rush also covers multi-hit barrages.

```lua
export type TravelDef = {
	from: number | "active",          -- 1-based move frame, or the first active frame
	to: number | "startup" | "active",-- 1-based move frame, the last startup frame, or the last active frame
	studs: number, arc: ("ground" | "leap")?, height: number?,
}
export type RushParams = {
	travel: TravelDef?,
	stopOnHit: boolean?,              -- travel stops on the first contact
	passThrough: boolean?,            -- informational: characters do not collide (Part 3 groups); only disables stopOnHit
	hits: { number }?, hitFrames: { number }?, hitWindow: number?,
	lastHit: { hitstun: number?, knockback: Types.KnockbackDef? }?,
	coneDeg: number?,                 -- Sphere hitbox angular filter (HitboxReq.coneDeg)
	echo: { delayF: number, damage: number, hitstun: number, knockback: Types.KnockbackDef?, ragdollFrames: number?, width: number, length: number }?, -- deferred (no M1-G user)
	selfBounce: { studs: number, height: number, frames: number }?,
	teleFwd: number?,                 -- telegraph anchor ahead of the Move-entry root (§4.2.0.6)
	dustStacks: number?,              -- read by the dustsheet passive (§4.5.7)
	-- Part 3-recognized: wallSplat, ragdollFrames, hangFrames, blockstun, cinematicFrames
}
```

Per frame (`step`):

| Phase | Output |
|---|---|
| Travel window (resolved `from ≤ e+1 ≤ to`, skipped once `MvA` bit 1 "stopped" is set) | `ground`: `motion = planar` at `v = studs·60/N` along the facing, where `N = to − from + 1`. `leap`: `motion = vector` with horizontal `v` and `vy(k) = 4·height·60/N·(1 − (2k+1)/N)` for `k = e+1−from ∈ [0, N−1]` (td). `Σ vy/60 = 0` exactly (net rise 0); the apex, reached at the end of frame `k = N/2 − 1`, equals `height` exactly for even `N` and is within `height/N²` for odd `N`. Example attendant.1: `N = 12`, `height = 6` → apex 6.0 after frame k = 5, back to the start height after k = 11 |
| Active (`S ≤ e < S+A`) | `hits`: single-hit moves emit `Buff.req(id.."#1", Moves.hitbox(def, ctx), Buff.hitN(bc, 1, def.damage), 4)`. Multi-hit moves follow §4.2.0.4; the last hit applies `lastHit` overrides. `coneDeg` is copied into the HitboxReq |
| Recovery | `selfBounce` (if `MvA` bit 2 "connected"): `vector` back along −facing, `studs·60/frames` horizontal plus a `height` arc over `frames` frames (the leap `vy(k)` formula with `N = frames`, `k = e − (S + A)`; nothing after `k = N − 1`). Otherwise `motion = none` |

`onContact`: sets `MvA` bit 2 (connected) and, if `stopOnHit`, bit 1 (stopped).

Slice users: leaps `attendant.1`, `lifeguard.q`, `lifeguard.2`; ground travel `lifeguard.a3`, `dustsheet.a2` (both `passThrough`); barrages `attendant.2` (4 hits), `busker.a2` (4 beats while travelling).

Echo (`params.echo`, **deferred**: no M1-G user; the type stays). When a style first needs it: on the first active frame (once-bit 4), `step` returns a **SpawnRequest**:
- `kind = "zone"`, stepped by `Zone.stepCarrier`;
- `origin` = the yaw-only root CFrame at the dash start, moved forward by `echo.length/2`;
- dynamic params `{lifeF = echo.delayF + 1, sub = 1}`.

`Zone.stepCarrier` recognizes `sub = 1`. It reads `view.def.params.echo` and, at `age ≥ echo.delayF` (once), emits a `Box (echo.width, 6, echo.length)` HitRequest with key `<name>#e`, `damage = echo.damage`, `hitstun = echo.hitstun`, `knockback = echo.knockback`, `kind = "zone"`, blockable. Then `dead`. The echo lands even after the move ends and whether or not the dash connected. Its implementation and its test (`behaviors.rush_echo_spawns_once`) land with that style.

Other functions:
- `validate`: `travel.studs` in (0, 60]; `hits`/`hitFrames` same length; every `hitFrames[i] ≤ active`; `echo == nil` in M1-G (deferred; when it lands, `echo.delayF ≥ 8` when `echo.damage > 0`).
- `maxDamage`: `damage + (echo and echo.damage or 0)`.
- `tellFrames = startup`. `blockable = true`.

Edge cases:
- **Hitstop.** Hitstop shifts `StF` (Part 3 §3.5.6). The travel window is computed on `e`, so travel pauses during hitstop.
- **Walls.** If travel hits a wall, Part 3's mover simply stops. No special handling.
- **Interrupted move.** `onEnd` emits nothing.
- **Travelling barrage.** In `busker.a2` the non-final beats carry the def's `knockback` (a 4-stud push), so the target drifts forward with the drummer and stays in reach (td).

**Verify:** Lune `behaviors.rush_travel_distance` (Σ planar displacement over the window = `studs` ± 0.05; `dustsheet.a2` 20 studs); `behaviors.rush_leap_apex` (attendant.1 and lifeguard.2: Σ vy·dt = 0 ± 0.05, apex = height ± 0.1 with `vy(k) = 4·height·60/N·(1 − (2k+1)/N)`); `behaviors.rush_multi_hit_sum` (attendant.2 emits 4 keys with Σ damage 11; busker.a2 emits 4 keys with Σ 16); `behaviors.rush_self_bounce` (attendant.1 bounces back 6 studs only after a connect).

#### 4.2.3 Projectile (`Behaviors/Projectile.luau`)

```lua
export type ProjectileParams = {
	kind: "beam" | "bullet" | "lob" | "wave" | "homing",
	range: number,
	width: number?, radius: number?, speed: number?,  -- studs, studs/s
	extendF: number?,                                 -- beam: frames to extend to full range (0 = instant)
	count: number?, everyF: number?, hits: { number }?,
	pattern: { Vector3 }?, flightF: number?, impactRadius: number?, aimRange: number?,
	widthEnd: number?, height: number?,
	pierce: number?,                                  -- bullets: targets passed through (0 default; -1 = all)
	homingDegPerSec: number?,
	reflectable: boolean,                             -- kept on every carrier projectile; the reflect path is deferred
	onHit: KitHitExt?,                                -- merged into every hit's HitDef.kit
	extender: boolean?, setPiece: string?, dustStacks: number?, teleAnchor: ("self" | "carrier")?,
	-- Part 3-recognized: ragdollFrames, cancelInto, cancelWindow, cancelChainIndex (Move → M1 cancel, busker.1)
}
```

Slice users: beam `dustsheet.a1`; bullets `attendant.3`, `lifeguard.3`, `sandcastle.1`, `busker.q`, `busker.1`, `scoop.a3` (3 shots), `dustsheet.1`; lobs `scoop.1`, `scoop.a1` (3 shots), `dustsheet.4`; waves `attendant.a2`, `lifeguard.a2`, `sandcastle.4`, `busker.a1` (widening). `homing` and `setPiece` have no slice user; they stay in the library with synthetic-def tests.

| Kind | Owner `step` | Carrier (`stepCarrier`) | Hit |
|---|---|---|---|
| `beam` | Active frames: a `Cast` HitboxReq from the root along the facing. Frame `i` of `extendF` sweeps from tip `(i−1)/extendF·range` to `i/extendF·range` (instant when `extendF = 0`). Width `width`. One key `#1`, `maxTargets` = 1 (td: beams stop at the first body) | — | `kind = "projectile"`, `melee = false`, not reflectable |
| `bullet` | First active frame (once-bit 1): one SpawnRequest per shot. Shots fire at active frames `1, 1+everyF, …` (`count` shots, once-bits 1..count). `origin` = root yaw frame × (0, 0, −2) | Each frame: `to = cf + dir·speed/60`. World check `view.ray(cf.Position, to−cf.Position)` → `dead` at the wall. Emit a `Cast` HitboxReq `prevCf = cf`, `cf = to`, size `(2r, 2r, 1)`, `maxTargets = 1 + max(pierce, 0)` (all if −1). `dead` when travelled ≥ `range` or the pierce budget is used | Key `#<shot>`, `kind = "projectile"`, `kit.reflectable = refl` |
| `lob` | Shots as for bullets. The landing point is `Buff.aimPoint(bc, aimRange)` plus `pattern[shot]` rotated by the facing yaw, sent as `lx, ly, lz` | Position = parabola from origin to landing over `flightF` (apex 8 studs, td). At `age ≥ flightF` (once): `Sphere` HitboxReq radius `impactRadius` at the landing point, every target in it (`maxTargets = 20`), then `dead`. Render draws a Circle telegraph (diameter `2·impactRadius`) at `lx,ly,lz` while `age < flightF` | Key `#<shot>`, `kind = "projectile"`, not reflectable |
| `wave` | First active frame: one SpawnRequest | Moves at `speed` along `dir`, ground-snapped via `view.ray` down. Each frame a `Box` HitboxReq of size `(w(age), height, speed/60 + 1)` with `w` linear from `width` to `widthEnd or width` over `range`. Every target once (`maxTargets = 20`). `dead` at `range` | Key `#1` (each target once per use) |
| `homing` | As bullets (count/everyF), with `serverOnly = true` (steering depends on other characters) | Like bullet. Each frame `dir` rotates toward the nearest enemy (`view.fighters`, lowest angle within ±90°, ties by rid) by at most `homingDegPerSec/60` degrees. Q-rule on `dir` | Key `#<shot>`, `kind = "projectile"` |

Dynamic carrier params (≤ 10; everything else is read from `view.def.params`):
- bullet/homing: `lifeF`, `dx, dy, dz` (unit direction, Q-rule 0.01), `sh` (shot index), `trv` (studs travelled), `rf` (1 = reflected), `o0` (original owner rid);
- lob: `lifeF`, `ox, oy, oz` (launch point), `lx, ly, lz` (landing point), `sh`;
- wave: `lifeF`, `dx, dz`, `trv`.

`lifeF` = `ceil(range·60/speed) + 1` for bullets, homing and waves, and `flightF + 1` for lobs.

Reflect (**deferred**: no M1-G user; built with the first style whose Counter reflects). The contract stays so the flag and the `rf`/`o0` params keep their meaning. `reflectCarrier(view, newOwner)` returns:
- `newOwner`;
- `set {dx = −dx, dy = −dy, dz = −dz, rf = 1, trv = 0}`;
- the carrier hit set is cleared, and the original owner is recorded in `o0`.

A carrier with `rf = 1` cannot be reflected again; a second reflecting stance absorbs it (td, no ping-pong). When a reflected carrier hits its `o0` rid, Part 3 passes `reflectedHome = true` in `HitInfo` (§4.3.1). In M1-G no hit is ever reflected, so `reflectedHome` is always false.

Other functions:
- `validate`:
  - `range > 0`;
  - beam needs `width`; bullet/homing need `speed` and `radius`; lob needs `flightF ≥ 8`, `impactRadius` and `#pattern == count`; wave needs `speed`, `width`, `height`;
  - `Σ hits == damage` when `hits` is set;
  - `onHit.pull` (§4.2.0.2): `knockback == nil`, `stopStuds ≥ 0`, and `pull.frames == hitstun` when `onHit.stun == "possess"`, else `pull.frames == min(hitstun, cfg.Combat.Stun.MoveKbFramesMax)`.
- `maxDamage`: `Σ hits` or `damage`.
- `tellFrames`: `flightF` for lob, else `startup`.
- `blockable = true` for every kind.

Edge cases:
- A bullet spawned inside a wall is `dead` on its first step.
- A bullet whose owner was KO'd is removed with the owner's other carriers (§4.2.0.5).
- A beam fired during hitstop is paused by Part 3.
- A homing projectile with no enemy in the ±90° cone flies straight.

**Verify:** Lune `behaviors.projectile_beam_sweep` (dustsheet.a1: `extendF` segments cover 0–25 studs without gaps); `behaviors.projectile_bullet_range` (attendant.3: `dead` after `range/speed·60` ± 1 frames); `behaviors.projectile_lob_lands_on_spot` (dustsheet.4: landing within 0.05 studs of `lx,ly,lz` at age `flightF`; every target inside `impactRadius` is in the request); `behaviors.projectile_wave_widens` (busker.a1: box width 4 at age 0, 16 at `range`); `behaviors.projectile_pull_frames_rule` (synthetic def, §4.2.0.2); `behaviors.homing_turn_rate` (synthetic def: never more than `hdg/60` degrees per frame). `behaviors.projectile_reflect_once` is deferred with the reflect path.

#### 4.2.4 Grab (`Behaviors/Grab.luau`)

```lua
export type GrabTick = {
	at: number, damage: number,       -- at: hold-relative frame (1-based)
	healSelf: number?,                -- kit.healSelf on this tick (no slice user)
	meterDrain: number?,              -- kit.meterDrain on this tick: the victim loses n Peak Meter (busker.3)
}
export type GrabParams = {
	groundOnly: boolean,              -- Part 3-recognized param: misses airborne victims (GDD §6.4)
	holdF: number, offset: Vector3, sub: number,  -- Grabbed Sub code (table below)
	ticks: { GrabTick },              -- Σ damage == def.damage; the last tick is the finish
	finish: "throw" | "slam" | "release" | "drop",
	throwDir: ("facing" | "input")?, finishKb: Types.KnockbackDef?, finishHitstun: number?,
	finishStun: StunFlavor?,          -- kit.stun on the finish tick (scoop.2: "stumble")
	finishRecovery: number,           -- attacker frames after the finish tick
	sweep: { studs: number, width: number, height: number }?,  -- moving catch box over the active window (no slice user)
	maxTargets: number?,              -- default 1
	carry: { riseStuds: number, riseF: number }?, -- no slice user
	cinematicFrames: number?, dustStacks: number?,
}
```

Slice users: `attendant.4` (hold, sit, tip out), `lifeguard.4` (shoulder lift, throw toward input), `busker.3` (hold, meter drain, release), `scoop.2` (hold, release with a stumble).

`Grabbed` Sub codes (Part 3 table: "grab kind (Part 4)"). Subs 3–6 keep their enum values; they have no M1-G user, so their render and tests are deferred (§4.9 item 26):

| Sub | Kind | Used by |
|---|---|---|
| 1 | hold | `attendant.4`, `busker.3`, `scoop.2` |
| 2 | lift | `lifeguard.4` |
| 3 | coil | none (deferred) |
| 4 | drain | none (deferred) |
| 5 | carry | none (deferred) |
| 6 | thrall | none (deferred) |

Flow:

| Phase | Output |
|---|---|
| Startup | Nothing (`motion = none`; Part 3 walk speed 0) |
| Active, `MvA == 0` (not caught) | Catch request: `Moves.hitbox(def)` (or the sweep box moving `studs·60/A` per frame along the facing). `hit = Buff.hitN(bc, 0, 0, {kind = "grab", groundOnly = ground, kit = {grab = {holdF, sub, offset}}})`, key `"#c"`, `maxTargets = maxTargets or 1`. A sweep also outputs `motion = planar` along the facing |
| `onContact` (outcome `"hit"`, Part 3 attached `HB_Hold`) | `patch {MvA = MvA + 1, MvB = e, StDur = S + A + holdF + finishRecovery}`. The move is extended to cover the hold. With a sweep, catching continues until the active window ends |
| Hold (`MvA ≥ 1`; `MvB = catchE`, `k = e − catchE`) | For each tick `i` with `ticks[i].at ≤ k + 1 < ticks[i].at + 3` (a 3-frame window, td; Part 3's registry dedupes per key): `hits = {Buff.req(id.."#"..i, holdBox, hit, maxTargets)}` with `heldOnly = true`. `holdBox` is a Sphere of radius 3 at `rootCf × offset`. The hit has `damage = tick.damage`, `kind = "grab"`, `kit.healSelf = tick.healSelf`, `kit.meterDrain = tick.meterDrain`, and for the last tick `kit.release = true` plus the finish knockback/hitstun and `kit.stun = finishStun`. `carry` → `motion = vector (0, riseStuds·60/riseF, 0)` while `k < riseF`, then `vector 0`. Hold `motion = freeze` otherwise |
| Finish direction | `throwDir = "input"`: knockback direction = `bc.inp.moveDir` (else facing); `"facing"`: facing; `slam`/`drop`: `finishKb` with `angleDeg = 0` and ragdoll |
| Whiff (active ends with `MvA == 0`) | Normal recovery (`def.recovery`) |
| `onEnd` | Nothing. Part 3 detaches `HB_Hold` on any exit of the holder's Move state (§4.9 item 10) |

Breakaway, Shake-Off or KO of the victim: Part 3 detaches the hold. The holder's Breakaway burst puts the holder into `Stagger`, which ends its Move (Part 3 §3.9.1), and `onEnd` runs. If the holder is still in Move with no held victims, the remaining ticks with `heldOnly = true` find no target and whiff harmlessly; the move ends at its `StDur`.

`cinematicFrames` (Ultimate grabs only): every hold hit sets `HitDef.cinematicLockF = cinematicFrames` (Part 3 caps it at 120 f, GDD §6.8).

Other functions:
- `validate`: `Σ ticks.damage == damage`; ticks sorted by `at` with `at ≤ holdF`; `finishRecovery ≥ 8`; `maxTargets ≤ 3`.
- `maxDamage = damage`. `tellFrames = startup`.
- `blockable = false` (grabs ignore block, GDD §6.4), hence `guardDamage = 0` and `chip = 0`.

**Verify:** Lune `behaviors.grab_ticks_sum` (lifeguard.4 emits 2 hold keys whose damage sums to 9); `behaviors.grab_whiff_recovery` (no contact → move length = S+A+R); `behaviors.grab_hold_extends_stdur`; `behaviors.grab_meter_drain` (busker.3's finish tick carries `kit.meterDrain = 15`, the first tick none); `behaviors.grab_finish_stun` (scoop.2's finish tick carries `kit.stun = "stumble"` and `hitstun = 30`). Scenarios: `kit.grab_vs_jump` (`attendant.4` vs a jumping dummy → whiff), `kit.grab_breakaway` (dummy with pips Breakaways at hold frame 15 → holder `St = Stagger`, no further ticks).

#### 4.2.5 Counter (`Behaviors/Counter.luau`)

```lua
export type CounterParams = {
	arcDeg: number,                   -- front arc that counts (180 = front half, Defense.isFront)
	vsMelee: "stagger" | "launch",
	staggerF: number?,                -- stagger reaction frames (Part 3 Stagger Sub 3 "melee bounce")
	counterKb: Types.KnockbackDef?, counterHitstun: number?, hangFrames: number?,
	reflectProjectiles: boolean,      -- must be false in M1-G (the reflect path is deferred)
	endOnTrigger: boolean, triggerRecovery: number?,
}
```

Slice users: `attendant.a1` (launch), `sandcastle.a3` and `dustsheet.2` (stagger; projectiles pass). None reflects.

`step`: startup and recovery output `motion = none`. Active is the stance, `motion = freeze`. The behavior emits no hits of its own.

`intercept(bc, hit, srcPos)` (server; the victim is in `Move` with this def):

| Condition | Verdict |
|---|---|
| Not in active frames, or `srcPos` outside the front arc | `pass` |
| `hit.kind == "grab"` | `pass` (grabs beat counters, GDD §6.7) |
| `hit.kit.reflectable == true` and `reflectProjectiles` (deferred) | `reflect`. Part 3 calls the carrier's `reflectCarrier(view, defenderRid)` and the hit is negated. Unreachable in M1-G: `validate` rejects `reflectProjectiles = true` until the reflect path exists |
| `hit.kind == "projectile"` (beam, not reflectable) | `absorb` if `reflectProjectiles` (the wall blocks it), else `pass` |
| `hit.melee == true` | `absorb` with a reaction. `stagger`: `HitDef {damage = 0, kit.stun = "hit", hitstun = staggerF}`, which Part 3 maps to `Stagger` Sub 3 for `staggerF`. `launch`: `HitDef {damage = def.damage, knockback = counterKb, hitstun = counterHitstun, kind = "move", melee = true, unblockable = true}` |
| Zone ticks, AoE bursts (`melee == false`, not projectile) | `pass` |

If `endOnTrigger` is set and the verdict is not `pass`, `patch = {StDur = e + 1 + triggerRecovery}`. Every intercept returns `patch.MvA = MvA + 1` (trigger count), which render uses for the "boing" flash.

Other functions:
- `validate`: `active ≥ 12`; `vsMelee = "launch"` requires `counterKb`; `reflectProjectiles == false` (M1-G).
- `maxDamage = damage` (launch counters) or 0. `tellFrames = startup`.
- `blockable = false` (the reaction hits an attacker mid-swing; td).

**Verify:** Lune `behaviors.counter_staggers_melee` (sandcastle.a3: reaction `hitstun == 24`; dustsheet.2: 30), `behaviors.counter_launches_melee` (attendant.a1: reaction damage 14, launch, `StDur` cut to `e + 1 + 12`), `behaviors.counter_projectile_passes` (a bullet hit on sandcastle.a3 → `pass`), `behaviors.counter_loses_to_grab`, `behaviors.counter_back_passes`. `behaviors.counter_reflects_projectile` and the reflect scenario are deferred with the reflect path.

#### 4.2.6 Zone (`Behaviors/Zone.luau`)

```lua
export type ZoneBurst = { atF: number, damage: number, hitstun: number?, stun: StunFlavor?, knockback: Types.KnockbackDef?, ragdollFrames: number?, blockable: boolean? }
export type ZoneParams = {
	anchor: "self" | "aim" | "selfFollow", aimRange: number?,
	shape: "circle" | "cone" | "line" | "ring",
	radius: number,                    -- circle/ring outer radius; cone length; line length
	innerRadius: number?, coneDeg: number?, width: number?,
	armF: number,                      -- frames from spawn to the first effect (telegraph lead)
	lifeF: number,                     -- frames the zone persists after arming (0 = bursts only)
	bursts: { ZoneBurst }?,
	tick: { everyF: number, damage: number, count: number?, blockable: boolean? }?,
	stay: StatusReq?,                  -- debuff refreshed every view.kits.StatusStayRefreshFrames on enemies inside
	ownerTick: { everyF: number, heal: number }?, -- no slice user
	maxHitsPerTarget: number?, pullStudsPerSec: number?, followMove: boolean?, -- followMove: no slice user
	dmgModel: ("total" | "perHit")?, cinematicFrames: number?, teleAnchor: ("self" | "carrier")?, hangFrames: number?,
}
```

**Hit shapes** (instant and carrier zones alike):
- circle: `Sphere` with diameter `2·radius`;
- cone: `Sphere` with diameter `2·radius` plus `coneDeg`;
- line: `Box (width, 6, radius)` ahead of the anchor along its yaw (the box starts at the anchor);
- ring: `Sphere` with diameter `2·radius` plus `innerRadius` (Part 3's `HitboxReq.innerRadius`, §4.9 item 7): targets nearer the centre than `innerRadius` (planar) are skipped.

**Instant self zone** (`anchor = "self"`, `armF = 0`, `lifeF = 0`, every burst `atF = 0`): no carrier. The owner `step` emits the burst on the active frames as a HitRequest (same fields and `unblockable` rule as the carrier burst row below). This gives the owner BQ-predicted contact (Part 3 §3.1.2).

Used by `scoop.q`, `busker.4`, `sandcastle.a1` (line) and `scoop.a2` (cone).

**Carrier zone** (every other case). Slice users: `aim` circles `lifeguard.1`, `lifeguard.a1`, `lifeguard.a4` (pull), `scoop.4`; `self` lines `attendant.a4` (one burst after a 1 s arm) and `sandcastle.3` (ticks and a slow); `self` circle `busker.a4` (ticks, a slow, then a burst); `self` ring `sandcastle.a4` (three bursts); `selfFollow` ring `dustsheet.a3`. On the first active frame (once-bit 1), return a SpawnRequest:
- `origin` = `Buff.aimPoint(bc, aimRange)` (aim), or the yaw-only root (self/selfFollow);
- dynamic params (≤ 10): `lifeF = armF + max(lifeF, last burst atF) + 1`, `st` (bitmask of fired bursts), `tk` (ticks done).
- Everything else (shape, radius, ticks, stay, owner tick, pull, caps) is read from `view.def.params`.
- `aim` zones use the deterministic `Buff.aimPoint` (static-world ray), so they can be predicted on the owner client.

`stepCarrier(view, f)` (server; owner client for predicted zones, hits ignored), with `k = age − armF`:

| Condition | Output |
|---|---|
| `selfFollow` | `cf = yaw(ownerCf)` every frame; `dead` if `ownerAlive == false` |
| `k < 0` | Nothing (arming; the telegraph shows) |
| Burst `b` with `k ≥ b.atF`, not yet fired (bit `b` in `st`) | HitRequest key `<name>#b<b>` with the shape box, `damage = b.damage`, `hitstun`, `kit.stun`, `knockback`, `kind = "zone"`, `unblockable = (b.blockable == false)` (default **blockable**; zone centre as source) |
| Tick: `k ≥ 0`, `k % tick.everyF == 0`, `tk < count` | HitRequest key `<name>#t<j>` (j = `tk + 1`; `set.tk = j`), `damage = tick.damage`, `tick = true` (Part 3: no hitstun, hitstop or combo count), `unblockable = (tick.blockable ~= true)` (default **unblockable**). Per-target cap `maxHitsPerTarget` enforced by server memory |
| Stay: `k ≥ 0`, `k % view.kits.StatusStayRefreshFrames == 0` | Zero-damage `tick` HitRequest key `<name>#s<k>` with `kit.debuffs = {stay}`, `unblockable = true` (tick default) |
| Owner tick: `k % ownerTick.everyF == 0` and the owner is inside | `ownerHeal = ownerTick.heal` (routed through `Adapter.heal`, Part 3 §3.9.3). No slice zone sets it |
| Pull (`params.pullStudsPerSec`) | Not a hit and not an output of `stepCarrier`. **B-class**, computed in each victim's own step (both peers), see "Zone pull" below |
| `k ≥ lifeF` and all bursts fired | `dead = true` |

The periodic `k % N == 0` tests are the one allowed modulo use (Part 3 §3.8.7): a skipped frame skips a tick identically on both peers.

**Zone pull (B-class, §4.9 item 11).** Per-frame mover writes on another character from the server would mispredict every frame on the victim's client, so the pull is derived by the victim's own step instead (Part 3 §3.1.2). In each character's step (phase 1, after the behavior hook), on both peers, Part 3's adapter (Part 3 §3.11.5 "Zone pull" implements this):
1. scans the live carriers in `workspace.Sim` with the same enumeration `Carriers.step` uses in phase 1b (sorted by `Cid`, Part 3 §3.11.5), for zone carriers (attributes `Kind == "zone"`, `Dead == false`, `MoveId`) whose MoveDef (looked up with `moves`, the pinned kit set; `params` are not overridable, §4.1.6, so every peer agrees) has `params.pullStudsPerSec`, and whose owner (`Own`) is not this character (party membership is server-only data, so party allies are not exempt; the pull deals no damage; attribute and `CFrame` reads only);
2. for each, calls the pure helper **`Zone.pullVelocity(def, carrierPos: Vector3, rootPos: Vector3): Vector3`**: `Vector3.zero` if the planar distance from the carrier centre to the root is > `radius` or < 0.5 studs; otherwise `Buff.flat(carrierPos − rootPos).Unit · pullStudsPerSec`, each component rounded to 0.01 (Q-rule);
3. sums the contributions, rounds each component to 0.01 (Q-rule) and adds the result to the FSM's `MotionCmd` for that frame: `none` becomes `planar` (`moveDir · walkSpeed` + pull), `planar` adds to `vx`/`vz`, `vector`/`freeze` (knockback, hitstop, Wall Splat) are unchanged (Part 3 §3.11.5).

`pullStudsPerSec` is valid only with `shape = "circle"` (Zone `validate`).

Spike S-P4-3 measures the rollback rate this causes on a victim inside `lifeguard.a4`, the slice's one pulling zone (§4.11).

`followMove` (no slice user): during the owner's active frames, the owner `step` outputs `motion = planar` from `bc.inp.moveDir × bc.cfg.Chassis.WalkSpeed × 0.6` (td) instead of `none`.

Other functions:
- `validate`: `radius > 0`; `armF ≥ 0`; tick zones need `count` or `maxHitsPerTarget`; `cinematicFrames ≤ 120`; `pullStudsPerSec` requires `shape == "circle"`.
- `pullVelocity(def, carrierPos, rootPos)`: zone pull helper (above), pure, called by Part 3's adapter.
- `maxDamage`: `Σ burst.damage + min(tick.damage × (count or ∞), tick.damage × maxHitsPerTarget)`, capped by `damage`. For `perHit` it is `tick.damage × maxHitsPerTarget`.
- `tellFrames`: `armF` for `aim` zones; `startup + armF` for `self`/`selfFollow` zones, instant or carrier (§4.2.0.6).
- `blockable(def)`: true if any burst has `b.blockable ~= false`, or the tick has `tick.blockable == true`. This matches the runtime defaults above (bursts blockable unless `false`; ticks unblockable unless `true`), so K14/K15 judge exactly what Part 3 resolves. Examples: sandcastle.3 (ticks only, no flag) → false (`chip = 0`, no guard damage); scoop.q, busker.4, sandcastle.a1, sandcastle.a4, lifeguard.1, lifeguard.a1, scoop.4, attendant.a4 (bursts) → true; lifeguard.a4, busker.a4, dustsheet.a3 (`tick.blockable = true`) → true.

Edge cases:
- An aim zone over a pit snaps to the first ground within 60 studs below, else spawns at aim height.
- Two zones from the same owner apply independent caps.
- Leaving a zone ends `stay` effects within `StatusStayRefreshFrames` + the status duration.
- A burst and a tick due on the same `k` both fire (e.g. `lifeguard.a1` at `k = 0`: the 4-damage spike and the first 2-damage sting).

**Verify:** Lune `behaviors.zone_tick_cap` (sandcastle.3 on a stationary target inside the trench for 10 s → exactly 5 ticks, 5 damage); `behaviors.zone_burst_once`; `behaviors.zone_selfFollow_tracks_owner` (dustsheet.a3); `behaviors.zone_owner_heal` (synthetic def); `behaviors.zone_line_ahead` (attendant.a4: a target 30 studs ahead is hit once at `k = 0`; a target 2 studs behind is not); `behaviors.zone_ring_inner` (sandcastle.a4: targets at 5 and 26 studs are never hit; a target at 15 studs takes three bursts of 8 at `k = 0`, 45 and 90); `behaviors.zone_ticks_then_burst` (busker.a4: ticks at `k = 0`, 45, 90, 135, then the burst at 180; Σ on one target 26); `behaviors.zone_blockable_defaults` (`Zone.blockable`: sandcastle.3 → false, busker.4 → true, lifeguard.a4 → true; a tick request without `blockable` has `unblockable = true`, a burst request without `blockable` has `unblockable = false`); `behaviors.zone_pull_b_class` (`Zone.pullVelocity(lifeguard.a4, centre, root)`: root 6 studs from the centre → planar magnitude 4 ± 0.02 toward the centre, components rounded to 0.01; 12 studs → zero; 0.3 studs → zero; lifeguard.a4's `stepCarrier` outputs never contain a pull).

#### 4.2.7 Summon (`Behaviors/Summon.luau`)

```lua
export type SummonAttack = {
	range: number, startupF: number, everyF: number,  -- everyF = 0: one attack only
	size: Vector3?, kind: ("melee" | "beam")?, beamRange: number?, beamWidth: number?, firstAtF: number?,
	stun: StunFlavor?, tele: boolean?,
}
export type SummonParams = {
	dmgModel: "perHit" | "total",
	bodies: number, hpEach: number, lifeF: number, untargetable: boolean?,
	spawnOffsets: { Vector3 },        -- owner yaw space
	speed: number, ai: "chase" | "line" | "marked", leash: number?, -- "marked": deferred (no M1-G user)
	attack: SummonAttack?, line: { studs: number, frames: number, width: number }?,
	maxHitsPerTarget: number, once: boolean?, model: string,
}
```

Slice users: `chase` at speed 0 (traps and turrets, E2) `attendant.q` (one snap, `once`), `sandcastle.2`, `sandcastle.a2`; moving `chase` `busker.2`, `busker.a3` (3 bodies), `dustsheet.3` (2 bodies); `line` `scoop.a4` (rolls 40 studs, then parks with 40 HP).

`step`: on the first active frame (once-bit 1), one SpawnRequest per body, with `serverOnly = true`. `origin` = yaw-root × `spawnOffsets[i]`.
- Dynamic params (≤ 10): `lifeF`, `hp` (0 = untargetable), `bi` (body index), `st` (0 idle, 1 attacking), `a0` (attack start age), `nx` (next attack age), `tg` (target rid), `n` (attacks done), `dx, dz` (line direction).
- Static data (`speed`, `attack`, `line`, `leash`, `model`) is read from `view.def.params`.
- The summon-arrival tell is the MoveDef `Circle` telegraph drawn at each spawn offset during startup.

`Summon.stepCarrier(view, f)` (server only; `view.fighters` sorted by rid):

| AI | Behavior per frame |
|---|---|
| `chase` | **E2:** target = the enemy nearest the **carrier** (planar distance from `view.cf`), among enemies within `leash` studs of the carrier (ties by rid). With `speed = 0` the summon never moves, so it waits and attacks whoever comes within `attack.range`: a trap (`once`) or a turret (repeating). If the target's distance is > `aR`, move toward it at `spd/60` studs per frame (ground-snapped by `view.ray`; nothing when `spd = 0`), facing it. Otherwise, if not attacking and `age ≥ nextAttackAge`, start an attack: `st = 1`, `a0 = age` (render draws an `Arc` telegraph sized from `attack.size` while `age − a0 < attack.startupF` when `attack.tele`). At `age − a0 ≥ attack.startupF` (once per attack): a `Box` HitRequest of `size` ahead, key `#<attack#>`, `damage = def.damage`, `hitstun = def.hitstun`, `kit.stun = attack.stun`, `kind = "move"`, `melee = true`. Then `nx = age + attack.everyF`; with `once`, `dead` after the attack. Lost target → idle |
| `line` | On spawn, record `dir` = owner aim (flat). For `age < lF`: `cf += dir·lS/lF`; Box HitRequest `(lW, 6, lS/lF + 2)` swept with `prevCf`, key `#1`, every target once. Afterwards idle (blocks space, can be killed) |
| `marked` (**deferred**) | Untargetable set piece; implementation and tests land with the first style that needs it. At `age ≥ a0` and every `aE` frames: the target is the enemy closest to the owner's aim ray (`ownerAim`) within `aBr` studs and ±30°. The telegraph is a Line from the carrier to the target's position, locked at telegraph start, for `aS` frames. Then a Cast HitRequest along the locked line (`aBw` wide), key `#<shot>`, `kind = "projectile"`, not reflectable |

`hurtCarrier(view, hit, attackerRid)` (targetable summons only):
- `dmg = hit.damage × (if M1 family then view.cfg.Damage.Summon.M1DamageMult else 1)` (2, GDD §6.7 "summons take 2× M1 damage"; `view.cfg` is the owner's pinned Combat section);
- `set {hp = hp − dmg}`; `dead` when `hp ≤ 0`;
- event `{k = "summon_down", rid = ownerRid}` for render.

Summons do not block, take no hitstun and are not grabbable (td).

Caps: one cast's bodies form a group. Recasting the same move despawns that owner's previous group (td). Different moves keep separate groups, so `sandcastle.2`'s castle and `sandcastle.a2`'s Grand Castle can stand together.

Other functions:
- `validate`: K09 limits; `#spawnOffsets == bodies`; `attack.startupF ≥ 8` when the per-hit damage > 10; `line` requires `ai = "line"`; `ai ~= "marked"` in M1-G (deferred); with `speed = 0`, `leash ≥ attack.range` (td: a trap must be able to see what it can reach).
- `maxDamage`: `perHit` → `damage × maxHitsPerTarget × bodies`; `total` → `damage`.
- `tellFrames`: `attack.startupF` (chase) or `startup` (line).
- `blockable = true`.

**Verify:** Lune `behaviors.summon_limits` (K09 on all summon moves); `behaviors.summon_hits_cap` (sandcastle.2 vs a stationary target 4 studs from the castle for 8 s → ≤ 18 damage, at most 6 bonks); `behaviors.summon_chase_nearest_carrier` (E2: a speed-0 summon with enemy A 3 studs from the owner but 9 from the carrier and enemy B 3.5 studs from the carrier → targets B; with only A present it never attacks); `behaviors.summon_line_parks` (scoop.a4: travels 40 studs in 24 f, hits each target once, then idles with `hp = 40` until `lifeF`); `behaviors.summon_m1_double` (a 3.5 M1 hit → hp −7); `behaviors.summon_once_despawns` (attendant.q is `dead` after its one snap). Scenario `kit.attendant.trap` (E2 in Studio: plant `attendant.q`, walk a dummy to 6 studs from the chair, outside the 4-stud trigger → no snap; to 3.5 studs, inside it → the snap lands within `attack.startupF + 2` frames: dummy HP −3 and `Hitstun` Sub 6 (flinch) for 18 f, exactly once; the chair is then `Dead`. With a second dummy nearer the caster but outside the trigger, only the dummy at 3.5 studs from the chair is hit).

#### 4.2.8 Buff / Debuff (`Behaviors/Buff.luau`)

```lua
export type ModeEffect = { move: string, startupDelta: number?, guardDamageMult: number? }
export type BuffParams = {
	target: "self" | "front",
	range: number?, width: number?,                    -- front: zero-damage request box
	buffs: { StatusReq }?,                             -- added to the owner's Buffs (patch)
	debuffs: { StatusReq }?,                           -- front: applied to the first enemy hit
	toggleMode: { count: number, spawn: number, effects: { ModeEffect } }?, -- deferred (no M1-G user)
	random: boolean?,
}
```

Slice users: self `nh` buffs `attendant.a3` (+5) and `sandcastle.q` (+4); front debuff `dustsheet.q` (root 30 f).

`step`, on the first active frame (once-bit 1):
- `buffs`: `patch.Buffs = Buff.statusPatch(s, buffs, f)`.
- `target = "front"`: HitRequest key `#1`, a `Box (width, 6, range)` ahead, `damage = 0`, `tick = true`, `unblockable = true`, `kit.debuffs = debuffs`, `maxTargets = 1`.
- `toggleMode` (**deferred**): `patch.Kit = (s.Kit + 1) % count`. Built with the first style that has a mode toggle.

Other functions:
- `validate`: status kinds exist in `Moves.STATUS`; `toggleMode` is rejected in M1-G (deferred); when it lands: `toggleMode.count ≥ 2` and every `effects[i].move` exists in the same style.
- `maxDamage = 0`; `tellFrames = startup`; `blockable = false`.

Mode variants (**deferred** with `toggleMode`). `Styles.variants(def)` returns `{}` for every M1-G style, and Part 3's `StyleRuntime.variants` is `{}`. When a mode toggle lands, `Styles.variants(def)` precomputes, for every `ModeEffect` target move, an array indexed by `Kit + 1`. Part 3 calls it with the pinned kit set's StyleDef (`kitSet.get(styleId)`, overrides applied) when it builds `StyleRuntime` for the life:
- the variant for mode `m` = the base def with `startup += startupDelta` or `guardDamage = round(guardDamage × guardDamageMult)` from `effects[m + 1]`;
- modes with no effect for that move use the base def.

Part 3's `Moves.defFor` returns `StyleRuntime.variants[id][s.Kit + 1]` when present (§4.9 item 12). Variants are validated like base moves (K06–K16).

**Verify:** Lune `behaviors.buff_self_nh` (sandcastle.q adds `nh` n = 4 for 180 f; attendant.a3 adds n = 5); `behaviors.buff_front_debuff` (dustsheet.q emits one 0-damage, unblockable request with `kit.debuffs = {{kind = "rt", frames = 30, n = 1}}`); `styles.variants_empty_m1g` (`Styles.variants(def)` is `{}` for all six slice styles). `behaviors.buff_toggle_cycles` and the variant tests are deferred with `toggleMode`.

#### 4.2.9 Mobility (`Behaviors/Mobility.luau`)

```lua
export type MobilityParams = {
	mode: "dash" | "grapple" | "slide" | "flight",       -- M1-G implements "dash"; the others are deferred
	studs: number?, maxRange: number?, speed: number?, maxTravelF: number?,
	whiffRecoveryF: number?, whiffCooldownSec: number?,
	pitchClampDeg: number?, trail: boolean?,
	-- Part 3-recognized: air (usable airborne), iFrames = {from, to} (1-based move frames)
}
```

Slice user: `scoop.3` (dash 18 studs, i-frames on move frames 1–8).

| Mode | Behavior |
|---|---|
| `dash` | Active frames: `motion = planar` along the facing at `studs·60/A` |
| `slide` (**deferred**) | As `dash`, plus a `vis` trail for render |
| `flight` (**deferred**) | First active frame (once-bit 1): store the pitch in **radians**, `MvB = round3(math.clamp(math.asin(math.clamp(aim.Y, -1, 1)), -math.rad(pitchClampDeg), math.rad(pitchClampDeg)))` with `aim = bc.inp.aim` (unit). `math.asin` returns radians and `pitchClampDeg` is in degrees, so the clamp converts with `math.rad` (e.g. ±30° = ±0.524 rad). Each active frame: `motion = vector` along the yaw/pitch direction at `studs·60/A` |
| `grapple` (**deferred**) | First active frame (once-bit 1): `hit, n = bc.ray(root, aim.Unit × maxRange)`. If hit: `travelF = min(maxTravelF, ceil(dist·60/speed))`; `patch {MvA = 1 + 256 × travelF, MvB = round3(pitch), StDur = S + 1 + travelF + recovery}` (`pitch = math.asin(math.clamp(dir.Unit.Y, -1, 1))` in radians, `dir` = hit point − root). Yaw is the held facing. During travel (`e − S < travelF`): `motion = vector` along (yaw, pitch) at `speed`. On miss: `patch {StDur = S + 1 + whiffRecoveryF, CdQ = f + toFrames(whiffCooldownSec)}` |

The deferred modes keep their rows so the first style that needs one has a contract; their implementation and tests (`behaviors.mobility_flight_*`, `behaviors.mobility_grapple_*`) land with that style. `validate` rejects them in M1-G.

`iFrames` are honoured by Part 3 (`Fsm.isInvulnerable`, Part 3 §3.2.1).

Other functions:
- `validate`: `mode == "dash"` (M1-G); `studs > 0`; `iFrames.to ≤ total frames`.
- `maxDamage = 0`; `tellFrames = startup`; `blockable = false`.

**Verify:** Lune `behaviors.mobility_dash_distance` (scoop.3: Σ planar displacement 18 ± 0.1 over the active frames). Scenario `kit.scoop.meltaway_iframes` (a dummy running `m1_spam` hits during move frames 1–8 → whiff; frame 9 onward → hit).

#### 4.2.10 RealmCast (`Behaviors/RealmCast.luau`)

A Getaway is cast through `RealmCast`. Part 3 runs the whole lifecycle (Part 3 §3.9.3):
- the `RealmCast` state (90 f);
- the interrupt window (e 0–44) and hyper armor (45–89);
- opening at `e ≥ 90`, membership `RlmId`, queueing (max 2), close at 12 s;
- Do Not Disturb (internally Stubborn Heart);
- `Cd4 = AwkEndF` (one Getaway per Peak).

The Move state is not used, so `step` is never called for RealmCast. This module provides:
- `validate(def, cfg)` (`cfg` = the full `Types.Config`, boot validator):
  - `startup == cfg.Combat.Realm.CastFrames` (90, GDD §6.10), `active == 1`, `recovery == 0`, `cooldown == 0`, `damage == 0`;
  - `Realm.rules[def.id] ~= nil`;
  - telegraph `Circle` with `size.X == 2 · cfg.Combat.Realm.RadiusStuds` (80) and `leadFrames == 90`.

  Key names `CastFrames`/`RadiusStuds` are Part 3's (§3.17).
- `maxDamage = 0`, `tellFrames = 90`, `blockable = false`, `step = function() return nil end`.

The rule content lives in `Core/Realm.luau` (§4.4.3). `params` may carry render-only data (`dustsheet.a4` keeps its rule timing table in `params.getaway`, §4.4.3); `validate` ignores it.

#### 4.2.11 Lune harness for behaviors

`tests/core/behaviors.spec.luau` drives behaviors with a fake `BehaviorCtx`/`CarrierView`. It uses `tests/shim/FakeCtx.luau` (new file in the `tests/shim/` folder):

```lua
--!strict
-- tests/shim/FakeCtx.luau — minimal BehaviorCtx/CarrierView factory for Part 4 specs. Spec §4.2.11.
export type FakeWorld = { groundY: number, walls: { { x: number } }, fighters: { any } }
local FakeCtx = {}
function FakeCtx.bc(def: any, e: number, s: any?, inp: any?, world: FakeWorld?): any
	-- returns { f = 1000 + e, e = e, phase = <from def>, def = def, s = s or Fsm.initial(100, 1000, cfg),
	--   ctx = { f = 1000 + e, rid = 1, grounded = true, rootCf = CFrame.new(0, 3, 0), vel = Vector3.zero, ... },
	--   cfg = Defaults.Combat (the Combat section, as Part 3 passes it), kits = Defaults.Kits,
	--   moves = Styles.forConfig(Defaults).move, isServer = true,
	--   inp = inp or { bits = 0, pressed = 0, aim = Vector3.new(0, 0, -1), moveDir = Vector3.zero },
	--   ray = <plane at groundY + vertical walls> }
	return {} :: any
end
function FakeCtx.view(def: any, age: number, p: { [string]: any }?, world: FakeWorld?): any
	-- returns a CarrierView (§4.2.0.3) with the same cfg/kits/moves/ray as FakeCtx.bc, own = 1, f0 = 1000, age = age,
	--   cf = CFrame.new(0, 3, -3), fighters = (world and world.fighters) or {}
	return {} :: any
end
function FakeCtx.member(rid: number, pos: Vector3, isCaster: boolean?, warded: boolean?): any
	-- returns a Getaway MemberView (Part 3 §3.9.3, incl. E1 pos) for Realm rule tests (§4.4.3)
	return {} :: any
end
function FakeCtx.run(def: any, frames: number, world: FakeWorld?): { any } -- calls step for e = 0..frames-1, applying patch to s
	return {}
end
return FakeCtx
```

The test names are listed in §4.2.2–§4.2.9. The file also contains `behaviors.all_pure` (every behavior function returns the same output for the same input twice, and never mutates its input: a deep-compare snapshot before and after).

---

### 4.3 Passive and Quirk hook system

#### 4.3.1 Hook points

Passives are **data first** (§4.3.2–§4.3.4). Kit-specific logic goes in `StyleModule.hooks`. Hooks are pure functions returning `HookOut`, and Part 3 applies the outputs.

```lua
export type HookKind = "use" | "hitDealt" | "hitTaken" | "blocked" | "perfectBlocked" | "ko" | "lowHp" | "wallsplat" | "awaken" | "awakenEnd"
export type HookCtx = {
	f: number, rid: number, styleId: string, style: Fsm.StyleRuntime, s: Fsm.CharState, isServer: boolean,
	cfg: any,                                                   -- pinned Combat section (fullCfg.Combat, Part 3 §3.4.2)
	kits: any,                                                  -- pinned Kits section (fullCfg.Kits, §4.1.6)
	moves: (moveId: string) -> Types.MoveDef?,                  -- pinned kit set: RosterEntry.kitSet.move (overrides applied)
}
export type HitInfo = {
	attackerRid: number, victimRid: number, attackerStyle: string, victimStyle: string,
	moveId: string, slot: string, kind: string,                 -- Fsm.HitDef.kind
	outcome: string,                                            -- Damage.Outcome (Part 3 §3.12.6) or "absorbed"/"reflected" (Counter)
	damage: number, firstOfUse: boolean, reflectedHome: boolean, isEnder: boolean,
	tick: boolean,                                              -- Fsm.HitDef.tick (zone/DoT ticks)
	victimDebuffs: { Moves.Status },                             -- after this hit's statuses were applied
}
export type HookOut = {
	patch: { [string]: any }?,         -- owner B-class fields (passiveStep only)
	motion: Fsm.MotionCmd?,            -- passiveStep only; Part 3 applies it only when FSM motion is "none" and St == Air
	heal: number?,                     -- server: owner HP += heal (via Adapter.heal)
	cooldownsMinusSec: number?,        -- server: every owner Cd1..Cd4/CdQ -= sec (not below f)
	debuffsOnVictim: { StatusReq }?,   -- server: hitDealt only; added to info.victimRid's Debuffs
	events: { Fsm.SimEvent }?,         -- e.g. { k = "style_moment", rid = rid, f = f, a = victimRid }
}
export type PassiveHooks = {
	onStep: ((h: HookCtx, inp: Fsm.InputFrame) -> HookOut?)?,   -- both peers (owner), every step after Fsm.step
	onHitDealt: ((h: HookCtx, info: HitInfo) -> HookOut?)?,     -- server
	onHitTaken: ((h: HookCtx, info: HitInfo) -> HookOut?)?,     -- server
	onBlocked: ((h: HookCtx, info: HitInfo) -> HookOut?)?,      -- server (this fighter blocked)
	onPerfectBlocked: ((h: HookCtx, info: HitInfo) -> HookOut?)?, -- server
	onKO: ((h: HookCtx, victimRid: number, victimDebuffs: { Moves.Status }) -> HookOut?)?, -- server (killer)
	onLowHP: ((h: HookCtx, hp: number) -> HookOut?)?,           -- server; edge when HP crosses below passive.params.lowHpThreshold
	onAwaken: ((h: HookCtx) -> HookOut?)?, onAwakenEnd: ((h: HookCtx) -> HookOut?)?, -- both peers on the Peak transition step
	onWallSplat: ((h: HookCtx, victimRid: number) -> HookOut?)?, -- server (attacker)
}
```

Slice users: `onHitDealt` (`lifeguard`, `busker`, `scoop`, `dustsheet`), `onPerfectBlocked` (`attendant`), and the `sandcastle` block heal, which runs inside `Styles.serverHook` on `blocked` and `perfectBlocked` (§4.3.6). No slice style uses `onStep`, `onKO`, `onLowHP`, `onAwaken`, `onAwakenEnd` or `onWallSplat`; the hook points stay for later kits.

How Part 3 calls them (§4.9 item 13):

| Call | Where (Part 3) | Peers |
|---|---|---|
| `Styles.passiveStep(h, inp)` | After `Fsm.step`, for every character this peer simulates | Both. Outputs limited to B-class fields and motion |
| `Styles.serverHook("use", h, {moveId…})` | On `Move`/`RealmCast` entry | Server. Feeds the Snapshot tracker only |
| `Styles.serverHook("hitDealt" / "hitTaken" / …, h, info)` | In the server consequence phase of the same step, after `Damage.resolve` | Server. Outputs are S-class |
| `Styles.serverHook("awaken"/"awakenEnd", h)` | On the Peak transition | Both (B-class outputs only) |

`Styles.serverHook` calls the style's hook, then runs the Snapshot tracker (§4.3.5), and merges the outputs: heals add, cooldown reductions add, events concatenate.

Determinism: hooks never read server memory except through the Snapshot tracker and the Hard Hat rate table (§4.3.6), which feed S-class outputs only (Part 3 §3.1.8).

#### 4.3.2 Status registry content (`Moves.STATUS` in `Core/Moves.luau`)

Part 3 owns the codec. Part 4 owns these entries (Part 3 §3.4.2). Each entry is `{ name, list, maxStacks, mods(n) → partial StatusMods, flags }`. Two flags and one field are Part 4 extensions (§4.9 item 14):
- `consumeOn: "hit" | "block" | "attack" | nil`: removed when the owner lands a hit, raises Block, or enters any ATTACK state. M1-G uses `"hit"` only.
- `incomingFrom: { style, perStack, max }`: a victim debuff that adds damage only to hits from that attacker style.
- `StatusMods.m1Chain` (Part 3's field): overrides the M1 chain length. No M1-G status sets it; the field stays for later kits.

| Kind | Name | List | maxStacks | `mods(n)` | Flags | Sources (GDD) |
|---|---|---|---|---|---|---|
| `rt` | root | Debuffs | 1 | `rooted = true` | — | sandcastle.1 36 f (GDD §7.3.9); lifeguard.3 36 f (GDD §7.3.2); dustsheet.q 30 f, dustsheet.4 48 f (GDD §7.3.15) |
| `sl` | slow | Debuffs | 60 | `speedMult = 1 − n/100` (n = percent; same-kind adds, capped at 60%, td) | — | lifeguard passive 20% 60 f, lifeguard.1 15%, lifeguard.a1 30% (GDD §7.3.2); sandcastle.3 30% (GDD §7.3.9); scoop.1 20% 120 f, scoop.4 40% (GDD §7.3.10); busker.a4 40% (GDD §7.3.12); `dustsheet.a4` Dust Sheets 30% 60 f (GDD §7.3.15) |
| `si` | silence | Debuffs | 1 | `silenced = true` | — | `dustsheet.a4` Quiet Hours "no moves for 0.5 s", 30 f (GDD §7.3.15) |
| `du` | Dust | Debuffs | 5 | `regenBlocked = true` | `incomingFrom = {style = "dustsheet", perStack = 1, max = 5}` | dustsheet passive, 480 f (GDD §7.3.15) |
| `nh` | next-hit bonus | Buffs | 5 | `nextHitAdd = n` (owner's outgoing hits only) | `consumeOn = "hit"` | attendant.a3 "+5" (GDD §7.3.1); sandcastle.q "+4" (GDD §7.3.9); 180 f |
| `iv` | stealth | Buffs | 1 | `{}` (render reads it, §4.7.6) | `consumeOn = "attack"` | **deferred**: no M1-G user; the entry and its render land with the first stealth style |

Future kinds (E3, §4.9 item 25): `dz` Dazzled and `fg` Fogged.

Rules:
- **`nextHitAdd` and `incomingFrom` (Dust bonus).** Part 3's `Damage.resolve` adds the attacker's `nextHitAdd` and the victim's `incomingFrom` value (when the attacker's style matches) to `flatAdds` (Part 3 §3.8.4). Both apply **only on `hit.kit.firstOfUse`**, and are **clamped** so the move's per-target total for this use cannot exceed its band cap (base 18, a1–a3 20, a4 30, M1 family unclamped; GDD §6.7). `nh` is removed on the first landed hit.
- **Capacity.** At most 4 entries per list (Part 3 codec). With 4 debuff kinds possible in M1-G (`rt`, `sl`, `si`, `du`), the list never overflows; when later kinds add a fifth, the entry with the smallest `untilF` is evicted (Part 3 `statusAdd`).

**Verify:** Lune `kit_status.registry_kinds` (every kind used by any kit or Getaway rule exists); `kit_status.slow_additive_cap60`; `kit_status.dust_bonus_only_from_dustsheet` (a `du` 5 victim hit by attendant.2 → no add; by dustsheet.1 → +5 on the first hit only); `kit_status.bonus_clamped_to_band` (dustsheet.a1 16 + 5 → 20); `kit_status.nh_consumed_on_hit` (sandcastle.q then sandcastle.1: +4 on the first hit, none on the next); `kit_status.si_blocks_moves` (a silenced character cannot enter `Move`, Part 3 `silenced`).

#### 4.3.3 Chassis modifiers from data (`passive.params` and `awakening.modifiers`)

Part 3's `Moves.chassisMods(style, s, cfg)` builds `Fsm.ChassisMods` from data, using the keys Part 3 recognizes (Part 3 §3.18 item 12). Part 4 authors kit data in exactly that convention:

```lua
passive.params = {
	ragdollFramesAdd = -10,                    -- example: top-level recognized keys are always on
	lowHpThreshold = 30,                       -- example: while s.HP < lowHpThreshold ...
	lowHp = { breakawayRefillMult = 1.5 },     -- ... these recognized keys also apply
	perfectBlockCdSec = 1,                     -- a Part 4 key: read only by that style's hook; Part 3 ignores it
}
awakening.modifiers = { walkSpeedMult = 1.1 } -- the Peak's recognized keys while AwkF0 > 0 (attendant, GDD §7.3.1)
awakening.hurtboxScale = 1                    -- every kit: no Peak scales the hurtbox (GDD §6.2; §0.4)
```

Part 3's recognized passive/Peak keys are `breakawayRefillMult`, `enderDamageAdd`, `lowHpThreshold` + `lowHp`, `ragdollFramesAdd`, `m1RangeAdd`, `walkSpeedMult` and `airDash`. The merge order and arithmetic are Part 3's. Rule K11 checks that `awakening.modifiers` keys ⊆ those keys. No slice passive uses a recognized chassis key; every slice passive is a hook (§4.3.6), and each Peak sets exactly one modifier (§4.4.2).

**Verify:** Lune `kit_status.chassis_from_data` (in Peak: attendant → `walkSpeedMult 1.1`; lifeguard → `breakawayRefillMult 1.5`; sandcastle → `enderDamageAdd 2`; busker → `airDash true`; scoop → `breakawayRefillMult 1.25`; dustsheet → `walkSpeedMult 1.1`; every style → `hurtboxScale 1`; out of Peak → defaults).

#### 4.3.4 Hover (`passiveStep`, generic; deferred)

**Deferred:** no slice style hovers. The hover passive (`passive.params.hoverSec`, `hoverMode`) and its tests (`kit_status.hover_budget`, scenario for the first hovering style) land with the first style that needs it. The contract, kept for that style:
- `St == Air` and Jump held (`Fsm.IN.Jump` bit in `inp.bits`) and `s.Kit > 0` and vertical velocity ≤ 0 →
  - `motion = vector (md.X·W, 0, md.Z·W)`, where `md = inp.moveDir` and `W = h.cfg.Chassis.WalkSpeed` (`h.cfg` is the pinned Combat section);
  - `patch.Kit = s.Kit − 1`.
- `St == Idle` (grounded) and `s.Kit ≠ hoverSec·60` → `patch.Kit = hoverSec·60` (refill per jump).
- Otherwise nothing.

`Kit` is "hover frames left" for a hovering style. Every slice style leaves `Kit` at 0 (§4.9 item 16).

#### 4.3.5 Snapshots (style moments)

A **Snapshot** is a style's signature sequence. GDD §6.9: +10 Peak Meter per landed Snapshot; GDD §9.2: mastery XP. The tracker lives in `Styles` server memory. Per attacker rid it keeps a ring of the last 8 tokens `{tok, victim, f}`. Tokens:
- `"<moveId>:use"` (serverHook `use`);
- `"<moveId>:hit"` (`hitDealt` with outcome `hit` and `damage > 0`; zone ticks and carrier hits count, td, so a moat tick can open `sandcastle`'s Snapshot);
- `"wallsplat"` (`onWallSplat`).

After each push:
1. Test the style's `MomentDef.seq` as an ordered subsequence within the last `windowF` frames.
2. With `sameVictim`, every `:hit` and `wallsplat` token must share one victim (`:use` tokens match any victim).
3. On a match: append `{k = "style_moment", rid, f, a = victim}` to `HookOut.events`, clear the ring, and block further Snapshots for `h.kits.MomentCooldownFrames` (the attacker's pinned `Kits`).
4. Log outside the sim: `[HB][Style] moment style=<id> rid=<rid> victim=<rid>`.

Part 3 grants +10 Peak Meter on `style_moment` (Part 3 §3.9.2). Part 5 grants mastery XP from the flushed event.

Every slice Snapshot is a two-step `seq` on one Guest (`sameVictim = true`); no slice style detects its Snapshot inside a hook.

| Style | Snapshot (GDD) | Definition |
|---|---|---|
| `attendant` | *Front Row Seat*: Hop In, then Ticket Punch on the same Guest (GDD §7.3.1) | `seq = {"attendant.1:hit", "attendant.2:hit"}`, `windowF = 180` |
| `lifeguard` | *Ring and Rescue*: Ring Toss, then Rescue Hold (GDD §7.3.2) | `seq = {"lifeguard.3:hit", "lifeguard.4:hit"}`, `windowF = 180` |
| `sandcastle` | *Flood the Moat*: Dig a Moat, then Let the Tide In! on a Guest in the trench (GDD §7.3.9) | `seq = {"sandcastle.3:hit", "sandcastle.4:hit"}`, `windowF = 180` (td: a moat tick is the `:hit` token) |
| `busker` | *Snatch and Crash*: Snack Snatch, then Cymbal Crash! (GDD §7.3.12) | `seq = {"busker.3:hit", "busker.4:hit"}`, `windowF = 180` (the Snatch release pushes only 5 studs, td, so the 10-stud crash reaches) |
| `scoop` | *Brain Freeze Special*: Scoop Shot, then Brain Freeze (GDD §7.3.10) | `seq = {"scoop.1:hit", "scoop.2:hit"}`, `windowF = 240` (td) |
| `dustsheet` | *Put Away for Winter*: Draped!, then Moth Puff (GDD §7.3.15) | `seq = {"dustsheet.4:hit", "dustsheet.1:hit"}`, `windowF = 180` |

**Verify:** Lune `kits.moments_sequence` (a token stream per style → exactly one event; an out-of-order stream → none; the second match within 600 f → none; `sandcastle`: a `sandcastle.3` tick token then `sandcastle.4:hit` → one event; a zero-damage stay refresh pushes no token). Scenario `kit.moment_meter` (attendant lands Hop In then Ticket Punch on a dummy → Peak Meter +10 beyond the damage gains; `[HB][Style] moment` line).

#### 4.3.6 Per-style passive and Quirk implementation

| Style | Passive (GDD) | Implementation | Quirk (GDD) | Implementation |
|---|---|---|---|---|
| `attendant` | **Seen It All**: each Perfect Block knocks 1 s off his move cooldowns (GDD §7.3.1) | `onPerfectBlocked` → `cooldownsMinusSec = passive.params.perfectBlockCdSec` (1) | **Reserved**: plants a deckchair (20 HP, 6 s); the first enemy within 4 studs gets the chair snapped on its ankles, 18 f flinch, 3 dmg, CD 6 | `Summon` `ai = "chase"`, `speed = 0`, `once = true`, `attack.stun = "flinch"` (E2) |
| `lifeguard` | **Walk, Don't Run!**: her Ender blows the whistle; the target is slowed 20% for 1 s (GDD §7.3.2) | `onHitDealt`: `info.isEnder and outcome == "hit"` → `debuffsOnVictim = {{kind = "sl", frames = 60, n = 20}}` | **Pole Vault**: vaults 20 studs in a high arc and lands feet-first, 3 dmg, CD 6 | `Rush` leap, tags `Rush`, `Mobility` |
| `sandcastle` | **Hard Hat**: at most once per second, a hit she blocks heals her 1 HP (GDD §7.3.9) | Inside `Styles.serverHook` for style `sandcastle`, kinds `blocked` and `perfectBlocked` (td: a Perfect Block is a block). Server memory `hh[rid] = lastF` (the Hard Hat rate table). If `hh[rid] == nil or f − hh[rid] ≥ passive.params.blockHealEveryF` (60) → `heal = passive.params.blockHeal` (1) and `hh[rid] = f`. Cleared on despawn | **Spirit Level**: her next hit within 3 s deals +4, CD 8 | `Buff` self, `buffs = {{kind = "nh", frames = 180, n = 4}}` |
| `busker` | **Tip Jar**: each Ender he lands drops a coin in his hat; his move cooldowns tick 1 s down. No heal (GDD §7.3.12) | `onHitDealt`: `info.isEnder and outcome == "hit"` → `cooldownsMinusSec = passive.params.enderCdSec` (1) | **Pass the Hat**: skims his cap 30 studs; 18 f flinch, 3 dmg, CD 5. Grounded, no i-frames | `Projectile` bullet, `onHit.stun = "flinch"` |
| `scoop` | **Toppings**: once per move use, a hit on a target that was already slowed knocks 0.5 s off her cooldowns (GDD §7.3.10) | `onHitDealt`: outcome `hit`, `firstOfUse`, not `tick`, `damage > 0`, not M1 family, the hit's own move does not slow (`Buff.appliesStatus(h.moves(info.moveId), "sl") == false`, td: the slowing hit itself does not count), and `info.victimDebuffs` holds `sl` → `cooldownsMinusSec = passive.params.toppingsCdSec` (0.5) | **Ding-Ding!**: rings her kiosk-trike bell; enemies within 8 studs flinch 18 f, 3 dmg, CD 6 | `Zone` instant self circle, burst `stun = "flinch"` |
| `dustsheet` | **Gathering Dust**: her move hits and her Ender add Dust (max 5, 8 s); Dust blocks regen; each stack adds +1 to her first hit on that target per move (max +5, clamped to the band) (GDD §7.3.15) | `onHitDealt` (outcome `hit`, `firstOfUse`, not `tick`, `damage > 0`): `n = def.params.dustStacks or 1` for moves, 1 for the **Ender only** among M1s (td, §4.11 Q1) → `debuffsOnVictim = {{kind = "du", frames = 480, n = n}}`. The bonus is `incomingFrom` (§4.3.2) | **Hands Off!**: hangs a HANDS OFF tag on a target within 20 studs; rooted 0.5 s, CD 8 | `Buff` `target = "front"`, `debuffs = {{kind = "rt", frames = 30, n = 1}}` |

**Verify:** Lune `kits.attendant_seen_it_all` (a `perfectBlocked` call → `cooldownsMinusSec = 1`; a `blocked` call → nil); `kits.lifeguard_whistle` (an Ender hit → `debuffsOnVictim` `sl` 20 for 60 f; an M1-2 hit or a blocked Ender → nil); `kits.sandcastle_hard_hat` (two blocks 30 f apart → one heal of 1; 60 f apart → two heals; a Perfect Block counts); `kits.busker_tip_jar` (an Ender hit → `cooldownsMinusSec = 1` and no `heal`; a blocked Ender → nil); `kits.scoop_toppings` (scoop.2 on a victim slowed by scoop.1 → 0.5; scoop.2 on an unslowed victim → nil; scoop.1 on an already slowed victim → nil (its own slow); the second hit of one scoop.a1 use → nil; an M1 hit → nil); `kits.dustsheet_stacks_per_use` (dustsheet.3: two pieces of luggage bumping one target → +1 stack total, because only the `firstOfUse` hit counts, td); `kits.dustsheet_m1_ender_only`; `kits.dustsheet_no_stacks_from_ticks` (dustsheet.a3 ticks and the dustsheet.q front request add none).

#### 4.3.7 Lune spec `tests/core/kit_status.spec.luau`

Contains every `kit_status.*` test named above. It requires `Core/Moves` (Part 3) for the codec and checks that Part 4's `STATUS` entries round-trip through `statusEncode`/`statusDecode` within 50 characters at 4 entries.

---

### 4.4 Peak (Awakening) and Getaways (Realm)

#### 4.4.1 Move swap

Part 3 runs Peak activation (72 f, invulnerable, heal 10, +1 pip, shockwave), duration (60 s, +5 s per KO, max 90 s), and end (Part 3 §3.9.2). Part 4's content:
- `awakening.moves` are slots `a1..a4`. Part 3 maps keys 1–4 to them while `AwkF0 > 0` (`Move` Sub 11–14).
- The Quirk and M1 are unchanged. The style's uniform appears for the Peak, with its Peak piece (§4.7.4) and the gilt trim (render only, Part 6 §6.8.2; §0.4). Nobody grows.
- **Cooldowns.** Part 3 zeroes `Cd1..Cd4` at start and end (both sets start ready).
- **In-progress move.** A move in progress when the Peak ends completes with its captured MoveDef. After that, slots map to base moves.
- **Carriers.** Peak summons, zones and projectiles persist until their own life ends (td). A Getaway follows Part 3's close rules.
- **`onAwaken` / `onAwakenEnd`** (both peers): no slice style uses them.

**Verify:** scenario `kit.awaken.swap` (per slice style: `Dev.Command setMeter 100`, press Awaken; after 72 f, keys 1–4 start `<style>.a1..a4` (read `Sub` = 11–14 and the `MoveId` probe); after `AwkEndF`, keys map back to base moves).

#### 4.4.2 Peak modifiers (no hurtbox scaling)

Each slice Peak sets exactly one recognized modifier (§4.3.3). **No Peak scales the hurtbox:** `hurtboxScale = 1` for every style (GDD §6.2; §0.4; K10). The look column is render-only: Part 6 draws the uniform and the gilt trim, and attaches the Peak piece that §4.7.4 defines (GDD §6.9, §12.3).

| Style | Peak (GDD) | Look (render only; Peak piece) | `hurtboxScale` | `modifiers` |
|---|---|---|---|---|
| `attendant` | **Rush Hour**: walks 10% faster (GDD §7.3.1) | striped blazer and megaphone (`model.attendant.peak_blazer`) | 1 | `{ walkSpeedMult = 1.1 }` |
| `lifeguard` | **Red Flag Day**: Breakaway refills 50% faster; no size change (GDD §7.3.2) | a red flag on a pole on her back (`model.lifeguard.peak_flag`) | 1 | `{ breakawayRefillMult = 1.5 }` |
| `sandcastle` | **Topping Out**: her Ender deals +2 (GDD §7.3.9) | hard-hat lamp on (`model.sandcastle.peak_lamp`); the blueprint swirl is VFX | 1 | `{ enderDamageAdd = 2 }` |
| `busker` | **One-Man Band**: he can dash once in mid-air (GDD §7.3.12) | bass drum on his back, cymbals at his knees (`model.busker.peak_drum`) | 1 | `{ airDash = true }` |
| `scoop` | **Soft Serve**: Breakaway refills 25% faster (GDD §7.3.10) | kiosk shutters up and bunting out, worn as a backpack (`model.scoop.peak_kiosk`) | 1 | `{ breakawayRefillMult = 1.25 }` |
| `dustsheet` | **All Covered**: glides 10% faster; no size change (GDD §7.3.15) | moth lanterns glow on the sheet (`model.dustsheet.peak_lanterns`) | 1 | `{ walkSpeedMult = 1.1 }` |

**Verify:** K10/K11; Lune `kit_status.chassis_from_data` (§4.3.3).

#### 4.4.3 Getaway content: `Realm.rules["dustsheet.a4"]` (in `Core/Realm.luau`)

GDD §7.3.15, the `dustsheet` A4 Getaway **Greetings from the Closed Season** (40 studs, 12 s). A postcard frame slams down, sheets cover the arena and a sign flips to CLOSED. Three house rules apply to every enemy member who is not warded:

| # | Rule | When (`k = f − r.openF`) | Effect | Engine |
|---|---|---|---|---|
| 1 | **Dust Sheets**: sheets tangle round the ankles | every 4 s: `k = 240, 480, 720` | slow 30% for 1 s (`sl` n = 30, 60 f) | existing tick |
| 2 | **Quiet Hours**: a bell tinkles 0.5 s ahead, then each enemy is hushed | alternating with rule 1, so a pulse every 2 s: `k = 120, 360, 600` (bell at `k − 30`) | no moves for 0.5 s (`si`, 30 f) | existing tick |
| 3 | **Lights Out**: the lights go out in one quarter of the ring, clockwise, with the next quarter flickering 1 s ahead | every 3 s: `k = 180, 360, 540, 720` (flicker from `k − 60`) | enemies in the dark quarter bump into covered furniture: a 3-damage hit | **E1** (member position) |

Counters: **Do Not Disturb** (Stubborn Heart: warded members get no rule effects), staying out of the ring as it closes (members cannot walk out, Part 3 §3.9.3), or **Double Booking** (not in M1-G). There is no healing rule, no Dust ticking and no caster cooldown cut. Getaway damage is tagged `realm = true` (Peak Meter ×0.5, GDD §6.9).

Dark quarter at pulse `n = k / 180` (1–4): `q = ((n − 1) % 4) + 1`, compared with Part 3's `Realm.quadrant(r.center, m.pos)` (1–4 clockwise, E1). The first quarter is 1; it is dark from `k = 180` until the next pulse. **If E1 slips**, rule 3 ships as darkness only: the render still shows the dark and flickering quarters, and the rule returns no hit (the `LIGHTS_DAMAGE` switch below is `false`).

Part 3 keys rules by `RuleId` = the MoveDef id (Part 3 §3.9.3). Part 3 calls `tick` for each unwarded member on every step where `k` is a multiple of `tickFrames` (60; Part 3 §3.9.3). The rule derives all timing from `f − r.openF`, and every pulse below falls on a multiple of 60 f.

```lua
--!strict
-- content of Realm.rules in src/Shared/Core/Realm.luau (Part 4 fills; Part 3 owns the type, the framework and Realm.quadrant)
-- Realm is a leaf Core module: it requires only Types/Frames/Rng, so it writes Types.Status/Types.HitDef (Part 3 §3.4.1)
local LIGHTS_DAMAGE = true -- E1 (Part 3 MemberView.pos); false ships Lights Out as darkness only (§4.4.3)

local function closedSeasonHit(): Types.HitDef
	return {
		moveId = "dustsheet.a4", kind = "zone", damage = 3, -- GDD §7.3.15 Lights Out: 3 damage
		guardDamage = 0, chip = 0, hitstun = 0, blockstun = 0, hitstop = 0, knockback = nil, enderVariant = 0,
		parryable = false, unblockable = true, melee = false, groundOnly = false,
		tick = true, realm = true, meterGain = 1, signature = false, cinematicLockF = 0, kit = nil,
	}
end

rules["dustsheet.a4"] = {
	tickFrames = 60, -- every pulse below falls on a multiple of 60 f (GDD §7.3.15 timings)
	tick = function(r: RealmView, m: MemberView, f: number): { statusAdd: Types.Status?, hit: Types.HitDef? }?
		if m.isCaster or m.warded then
			return nil -- the caster is never affected; Do Not Disturb negates rule effects (GDD §6.10)
		end
		local k = f - r.openF
		if k <= 0 then
			return nil
		end
		local statusAdd: Types.Status? = nil
		if k % 240 == 0 then
			statusAdd = { kind = "sl", untilF = f + 60, n = 30 } -- Dust Sheets: every 4 s, slow 30% for 1 s (GDD §7.3.15)
		elseif k % 240 == 120 then
			statusAdd = { kind = "si", untilF = f + 30, n = 1 } -- Quiet Hours: 2 s offset, no moves for 0.5 s (GDD §7.3.15)
		end
		local hit: Types.HitDef? = nil
		if LIGHTS_DAMAGE and k % 180 == 0 then -- Lights Out: every 3 s (GDD §7.3.15)
			local dark = ((k // 180 - 1) % 4) + 1 -- clockwise, quarter 1 first (td)
			if Realm.quadrant(r.center, m.pos) == dark then
				hit = closedSeasonHit()
			end
		end
		if statusAdd == nil and hit == nil then
			return nil
		end
		return { statusAdd = statusAdd, hit = hit }
	end,
}
```

Implications enforced by Part 3's framework:
- **Pulse collisions.** At `k = 360` Quiet Hours and Lights Out fall together, and at `k = 720` Dust Sheets and Lights Out do. One tick returns both a status and a hit; two statuses never fall on the same frame.
- **The last frame.** `k = 720` is also the close frame (`EndF = OpenF + 720`). Part 3 runs the Active row before the Close row on that step (Part 3 §3.9.3), so its pulses (Dust Sheets and Lights Out) still fire.
- **Regen.** It is already off for members (Part 3 §3.8.7, GDD §6.2). Rule hits set `tick = true` (no hitstun) and `realm = true`.
- **Dust.** The rules add no Dust; the passive's `incomingFrom` bonus still applies to the caster's hits inside the ring.

**Timing data for render.** The same numbers live in the MoveDef as `params.getaway` (§4.5.7), which RenderController (Part 6) reads with the marker's `OpenF` to draw the tells: the bell (`sfx.dustsheet.getaway_bell`) 30 f before each Quiet Hours pulse, the dark quarter (`vfx.dustsheet.getaway_dark`) from each Lights Out pulse until the next, and the flicker on the next quarter for the 60 f before it. Lune `kits.getaway_schedule_matches` drives `tick` over `k = 0..720` and asserts that statuses and hits fall exactly on the frames `params.getaway` lists.

**Verify:** Lune `kits.getaway_rules` in `kits.spec.luau` (with `FakeCtx.member`: an enemy at `k = 240` → `sl` 30 for 60 f; at `k = 120` → `si` for 30 f; at `k = 100` → nil; the caster or a warded member → nil at every `k`; at `k = 180` a member at offset (5, 0, −5) (quarter 1) gets the 3-damage hit and a member at (5, 0, 5) (quarter 2) does not; at `k = 360` quarter 2 is dark and the result carries `si` plus the hit; with `LIGHTS_DAMAGE = false` no result ever carries a hit) and `kits.getaway_schedule_matches`. Scenario `kit.dustsheet.getaway` (dustsheet in Peak casts a4 with 2 dummies at 10 and 30 studs and 1 at 45):
- after 90 f, members are **3**: `RlmId` is set on the caster and on the 2 dummies inside 40 studs (members include the caster, Part 3 §3.9.3), not on the dummy at 45; `[HB][Combat] realm_open` appears (Part 3 event log);
- each enemy member carries `si` from `k = 120` for 30 f and `sl` 30 from `k = 240` for 60 f; the caster never does;
- E1 in Studio: the dummy placed in quarter 1 loses 3 HP at `k = 180`; the dummy in quarter 3 loses 3 HP at `k = 540` and nothing at 180 or 360;
- a dummy holding Block for 1 s (Do Not Disturb) gets no further rule effects;
- the Getaway closes at 12 s.

#### 4.4.4 Getaways for stub styles

`raincloud`, `serpent` and `ferry` are stubs (`implemented = false`). Their a4 is a stub move with `behavior = "RealmCast"`, `tags = {"Realm"}` and no `Realm.rules` entry. K12 and the RealmCast validate rule are skipped for stubs (K-rules apply to implemented styles only). Their designed Getaways need the Year-1 engine items E3–E6 (§4.9 item 25). Adding their rules is part of §4.8.

---

### 4.5 Kit data

#### 4.5.1 Conventions

- **Damage and cooldown** are the exact GDD §7.3 values. `damage` is the GDD "Dmg":
  - per-use total for `total`-model moves;
  - per hit for `perHit`-model moves ("3 per bonk", "3 per peck", "3 per bump", "3 per touch");
  - for "0 (+N)" buffs it is 0, and the `nh` status carries the +N.
- **"N-stud circle/area"** is read as diameter N (radius N/2). "Within N studs", "N-stud radius" and "between A and B studs" are radii (td reading).
- **Frame data, guard damage, hitstun, knockback, hitboxes, telegraph sizes, status lengths not given and params** are tuning defaults chosen to satisfy:
  - GDD §6.14 (tell ≥ 8 f before > 10 damage);
  - GDD §7.1 rule 5;
  - GDD §6.4 (guard 20–60, chip 20%);
  - GDD §18.2 (startup ≥ 6; ≥ 8 for big hits; AoE fuses);
  - GDD §6.7 (bands: base moves ≤ 18, Quirk ≤ 6, A1–A3 ≤ 20, A4 ≤ 30; cooldowns basic 6–10 s, utility 10–16 s, signature 14–22 s, Quirk 3–8 s, A4 25 s).
- **Knockback** follows Part 3 §3.8.5: `ragdoll = true` → ragdoll profile (`params.ragdollFrames`); `launch = true` → `dist` is the height, with `params.hangFrames`; otherwise a planar push.
- **GDD pointers.** Every move line ends with `-- gdd: §7.3.N <slot>` (P4-R5). Hook code points at the passive or Snapshot row the same way.
- **Peak.** Every kit sets `hurtboxScale = 1` and one recognized modifier (§4.4.2).
- **Charm Gear and summons.** `kitProps` lists the oversized prop, then the head piece, then each summon model key (K23; §4.7.4, §4.7.5). The optional Peak piece (`model.<style>.peak_<name>`, §4.7.4) is not listed in `kitProps`.
- **Rule check.** Every slot uses a behaviour from §4.2, passives use §4.3 hooks, and Peaks use §4.3.3 keys. Per kit: one signature (K02), an opener (K03), an extender by tag or frame proof (K04), a defence or mobility tool (K05), bands (K06, K07) and a ≥ 8 f tell on every move above 10 damage (K08). All A4s of non-Legendary styles are tagged `Ultimate`; `dustsheet.a4` is `RealmCast` (K12).

| Style | Opener (K03) | Extender (K04) | Defence or mobility (K05) | Signature (K02, K07, K08) | Summons (K09) |
|---|---|---|---|---|---|
| `attendant` | 1 Rush | 2 tagged `Extender`, startup 8 | 1 tagged `Mobility` | 4 Grab, CD 16, 14 dmg, startup 14 + arc telegraph | Quirk: 1 × 20 HP, 360 f |
| `lifeguard` | 2 Rush, 3 Projectile, 4 Grab | 4 tagged `Extender`, startup 9 | Quirk Mobility; 1 Zone | 2 Rush, CD 16, 15 dmg, startup 18 + circle telegraph | — |
| `sandcastle` | 1, 4 Projectile | 1 `params.extender`: 8 + 1 + 3 ≤ 12 | 3 Zone | 4 wave, CD 18, 16 dmg, startup 42 + line telegraph | 2: 1 × 40 HP, 480 f; A2: 1 × 60 HP, 480 f |
| `busker` | 1 Projectile, 3 Grab | 1 tagged `Extender` (GDD Appendix A), with a Move → M1 cancel | 4 Zone | 4 Zone, CD 16, 12 dmg, startup 14 + circle telegraph | 2: 1 × 20 HP, 360 f; A3: 3 × 7 HP, 360 f |
| `scoop` | 1 Projectile, 2 Grab | 2 tagged `Extender`, startup 9 | Quirk Zone; 3 Mobility (i-frames) | 4 aim Zone, CD 20, 12 dmg, arm 30 f + circle telegraph | A4: 1 × 40 HP, 360 f |
| `dustsheet` | 1, 4 Projectile | 1 `params.extender`: 8 + 1 + 3 ≤ 12 | 2 Counter | 4 lob, CD 20, 12 dmg, flight 24 f + circle telegraph | 3: 2 × 10 HP, 360 f |

#### 4.5.2 `attendant` (GDD §7.3.1, Starter, `DeckchairAttendant`)

Sonny, *Take a Seat*: an otter deckchair attendant who is sure the beach is his. M1: chair jabs; the Ender snaps the chair open under the target.

```lua
--!strict
-- src/Shared/Styles/attendant.luau — attendant kit. Realm: shared. Pure: load. Spec §4.5.2
local Kit = require("../Behaviors/Buff")

return function(K: Kit.KitBuilder): Kit.StyleModule
	local M, box, kb, tele, v3 = K.move, K.box, K.kb, K.tele, K.v3
	local def = {
		id = "attendant", rarity = "starter", archetype = "DeckchairAttendant", implemented = true, -- GDD §7.2
		hp = 100, -- GDD §6.2
		palette = { primary = K.rgb(214, 58, 48), secondary = K.rgb(245, 236, 214), accent = K.rgb(110, 70, 45) }, -- §4.7.1
		passive = { id = "attendant.passive", params = {
			perfectBlockCdSec = 1, -- GDD §7.3.1 Passive: each Perfect Block knocks 1 s off his move cooldowns
		} },
		quirk = M("attendant.q", { -- gdd: §7.3.1 Quirk
			behavior = "Summon", tags = { "Summon" }, damage = 3, cooldown = 6, -- GDD §7.3.1
			startup = 10, active = 1, recovery = 10, guardDamage = 10, hitstun = 18, -- 18f flinch (GDD §7.3.1); rest (td)
			params = { dmgModel = "perHit", bodies = 1, hpEach = 20, lifeF = 360, -- one deckchair, 20 HP, 6 s (GDD §7.3.1)
				speed = 0, ai = "chase", leash = 4, spawnOffsets = { v3(0, 0, -3) }, -- speed-0 trap, 4 studs (GDD §7.3.1; E2); offset (td)
				attack = { range = 4, startupF = 8, everyF = 0, size = v3(4, 4, 5), stun = "flinch", tele = true }, -- snaps on the ankles (GDD §7.3.1); (td)
				maxHitsPerTarget = 1, once = true, model = "model.attendant.summon_chair" }, -- the first enemy only (GDD §7.3.1)
		}),
		moves = {
			M("attendant.1", { -- gdd: §7.3.1 row 1
				behavior = "Rush", tags = { "Rush", "Mobility" }, damage = 8, cooldown = 7, -- GDD §7.3.1
				startup = 14, active = 5, recovery = 20, guardDamage = 25, hitstun = 30, -- (td)
				knockback = kb(4, 20, false), hitbox = box(6, 6, 6), -- lands seat-first (GDD §7.3.1); (td)
				params = { travel = { from = 3, to = "startup", studs = 20, arc = "leap", height = 6 }, -- 20 studs: GDD §7.3.1; height (td)
					stopOnHit = true, selfBounce = { studs = 6, height = 3, frames = 12 } }, -- "small bounce back" (GDD §7.3.1); (td)
			}),
			M("attendant.2", { -- gdd: §7.3.1 row 2
				behavior = "Rush", tags = { "Rush", "Extender" }, damage = 11, cooldown = 9, -- GDD §7.3.1
				startup = 8, active = 22, recovery = 18, guardDamage = 30, hitstun = 20, -- (td); startup 8 combos from M1-3 (K04)
				hitbox = box(5, 5, 5), telegraph = tele("Arc", 8, 5, 5), -- (td); > 10 dmg → tell ≥ 8f (GDD §6.14)
				params = { travel = { from = 1, to = "startup", studs = 4 }, -- (td)
					hits = { 2.5, 2.5, 2.5, 3.5 }, hitFrames = { 1, 7, 13, 19 }, -- three clips of 2.5, then a stamp of 3.5 (GDD §7.3.1)
					lastHit = { hitstun = 36, knockback = kb(12, 15, false) } }, -- the stamp spins the target away (GDD §7.3.1); (td)
			}),
			M("attendant.3", { -- gdd: §7.3.1 row 3
				behavior = "Projectile", tags = { "Projectile" }, damage = 6, cooldown = 12, -- GDD §7.3.1
				startup = 10, active = 2, recovery = 18, guardDamage = 20, hitstun = 30, -- (td)
				knockback = kb(5, 10, true), -- knocks the target flat (GDD §7.3.1); (td)
				params = { kind = "bullet", speed = 90, range = 40, radius = 1.5, pierce = 0, reflectable = true, -- 40 studs: GDD §7.3.1; rest (td)
					ragdollFrames = 36 }, -- "short ragdoll" (GDD §7.3.1); (td)
			}),
			M("attendant.4", { -- gdd: §7.3.1 row 4
				behavior = "Grab", tags = { "Grab", "Opener" }, signature = true, damage = 14, cooldown = 16, -- GDD §7.3.1
				startup = 14, active = 4, recovery = 24, -- 14f startup (GDD §7.3.1); rest (td)
				hitbox = box(4, 5, 6), telegraph = tele("Arc", 14, 5, 6), -- arc telegraph (GDD §7.3.1); size (td)
				params = { groundOnly = true, holdF = 44, offset = v3(0, 0, -3), sub = 1, -- (td)
					ticks = { { at = 16, damage = 4 }, { at = 40, damage = 10 } }, -- sits on it (4), tips them out (10) (GDD §7.3.1)
					finish = "throw", throwDir = "facing", finishKb = kb(8, 10, true), ragdollFrames = 60, finishRecovery = 18 }, -- onto the sand, ragdoll (GDD §7.3.1); (td)
			}),
		},
		awakening = {
			id = "attendant.awaken", durationSec = 60, hurtboxScale = 1, -- GDD §6.9, §6.2
			modifiers = { walkSpeedMult = 1.1 }, -- walks 10% faster (GDD §7.3.1 Peak)
			moves = {
				M("attendant.a1", { -- gdd: §7.3.1 A1
					behavior = "Counter", tags = { "Counter" }, damage = 14, cooldown = 8, chip = 0, parryable = false, -- GDD §7.3.1
					startup = 8, active = 48, recovery = 16, -- 8f startup, holds 0.8 s (GDD §7.3.1); recovery (td)
					telegraph = tele("Circle", 8, 6, 6), -- self circle telegraph (GDD §7.3.1, K08); size (td)
					params = { arcDeg = 180, vsMelee = "launch", counterKb = kb(12, 90, false, true), counterHitstun = 40, -- launched (GDD §7.3.1); (td)
						hangFrames = 12, reflectProjectiles = false, endOnTrigger = true, triggerRecovery = 12 }, -- (td)
				}),
				M("attendant.a2", { -- gdd: §7.3.1 A2
					behavior = "Projectile", tags = { "Projectile" }, damage = 16, cooldown = 12, -- GDD §7.3.1
					startup = 14, active = 2, recovery = 20, guardDamage = 40, hitstun = 36, knockback = kb(10, 15, true), -- knocks everyone over (GDD §7.3.1); (td)
					telegraph = tele("Line", 14, 6, 30), -- (td)
					params = { kind = "wave", speed = 45, range = 30, width = 6, height = 4, pierce = -1, reflectable = false, -- a 30-stud line (GDD §7.3.1); (td)
						ragdollFrames = 48 }, -- (td)
				}),
				M("attendant.a3", { -- gdd: §7.3.1 A3
					behavior = "Buff", tags = { "Buff" }, damage = 0, cooldown = 12, -- "0 (+5)" (GDD §7.3.1)
					startup = 8, active = 1, recovery = 10, -- (td)
					params = { target = "self", buffs = { { kind = "nh", frames = 180, n = 5 } } }, -- next hit within 3 s deals +5 (GDD §7.3.1)
				}),
				M("attendant.a4", { -- gdd: §7.3.1 A4
					behavior = "Zone", tags = { "Zone", "Ultimate" }, damage = 26, cooldown = 25, -- GDD §7.3.1
					startup = 12, active = 1, recovery = 30, guardDamage = 50, hitstun = 40, -- (td)
					telegraph = tele("Line", 60, 6, 40), -- the tune plays along a 40-stud line for 1 s (GDD §7.3.1); width (td)
					params = { anchor = "self", shape = "line", radius = 40, width = 6, armF = 60, lifeF = 0, -- 40 studs, 1 s (GDD §7.3.1); width (td)
						bursts = { { atF = 0, damage = 26, hitstun = 40, knockback = kb(12, 90, false, true) } }, -- one 26-damage launch (GDD §7.3.1)
						hangFrames = 20, teleAnchor = "carrier" }, -- (td)
				}),
			},
		},
		m1 = { anim = "anim.attendant.m1", vfx = "vfx.attendant.m1", sfx = "sfx.attendant.m1" },
		kitProps = { "model.attendant.gear_chair", "model.attendant.gear_boater", "model.attendant.summon_chair" }, -- §4.7.4, §4.7.5
	}
	return {
		def = def,
		hooks = {
			onPerfectBlocked = function(h, _info)
				return { cooldownsMinusSec = h.style.passiveParams.perfectBlockCdSec } -- gdd: §7.3.1 Passive
			end,
		},
		moment = { seq = { "attendant.1:hit", "attendant.2:hit" }, windowF = 180, sameVictim = true }, -- gdd: §7.3.1 Snapshot
		anim = {
			m1 = { windup = "punch_wind", strike = "punch_r",
				alt = { "punch_r", "punch_l", "slash_down", "punch_r", "punch_l", "slash_down", "kick", "slash_down", "windup_high", "stomp_down", "slash_side", "slash_side" } },
			q = { windup = "crouch", strike = "slash_down", face = { windup = "smile", active = "grit", recovery = "neutral" } },
			["1"] = { windup = "crouch", mid = "leap", strike = "sit" },
			["2"] = { windup = "punch_wind", strike = "point", alt = { "point", "punch_r" }, finish = "slash_down" },
			["3"] = { windup = "windup_high", strike = "throw" },
			["4"] = { windup = "grab_reach", strike = "grab_hold", finish = "sit", face = { windup = "shout", active = "shout", recovery = "grit" } },
			a1 = { windup = "ready", strike = "guard_wide" },
			a2 = { windup = "kick_wind", strike = "kick" },
			a3 = { windup = "ready", strike = "buff" },
			a4 = { windup = "cast_raise", strike = "slash_down", face = { windup = "awaken", active = "shout", recovery = "grit" } },
			awaken = { windup = "crouch", strike = "cast_raise", face = { windup = "grit", active = "awaken", recovery = "smile" } }, -- anim.<style>.awaken (72f, §4.6.1)
		},
	}
end
```

#### 4.5.3 `lifeguard` (GDD §7.3.2, Starter, `HippoLifeguard`)

Paddles, *The Lifeguard Who Can't Swim*: a hippo lifeguard who wades, vaults and hauls. M1: buoy slaps; the Ender is a whistle-blast shove.

```lua
--!strict
-- src/Shared/Styles/lifeguard.luau — lifeguard kit. Realm: shared. Pure: load. Spec §4.5.3
local Kit = require("../Behaviors/Buff")

return function(K: Kit.KitBuilder): Kit.StyleModule
	local M, box, sphere, kb, tele, v3 = K.move, K.box, K.sphere, K.kb, K.tele, K.v3
	local def = {
		id = "lifeguard", rarity = "starter", archetype = "HippoLifeguard", implemented = true, hp = 100, -- GDD §7.2, §6.2
		palette = { primary = K.rgb(0, 150, 150), secondary = K.rgb(230, 60, 50), accent = K.rgb(245, 245, 240) }, -- §4.7.1
		passive = { id = "lifeguard.passive", params = {
			enderSlowPct = 20, enderSlowFrames = 60, -- GDD §7.3.2 Passive: her Ender slows the target 20% for 1 s
		} },
		quirk = M("lifeguard.q", { -- gdd: §7.3.2 Quirk
			behavior = "Rush", tags = { "Rush", "Mobility" }, damage = 3, cooldown = 6, -- GDD §7.3.2
			startup = 12, active = 4, recovery = 16, guardDamage = 20, hitstun = 18, -- (td)
			knockback = kb(3, 0, false), hitbox = box(5, 5, 5), -- lands feet-first (GDD §7.3.2); (td)
			params = { travel = { from = 3, to = "startup", studs = 20, arc = "leap", height = 9 }, -- 20 studs, high arc (GDD §7.3.2); height (td)
				stopOnHit = true }, -- (td)
		}),
		moves = {
			M("lifeguard.1", { -- gdd: §7.3.2 row 1
				behavior = "Zone", tags = { "Zone" }, damage = 7, cooldown = 6, -- GDD §7.3.2
				startup = 10, active = 1, recovery = 14, guardDamage = 20, hitstun = 24, -- (td)
				telegraph = tele("Circle", 18, 8, 8), -- (td)
				params = { anchor = "aim", aimRange = 30, shape = "circle", radius = 4, armF = 18, lifeF = 240, -- up to 30 studs, bobs 4 s, 4 studs (GDD §7.3.2); armF (td)
					bursts = { { atF = 0, damage = 7, hitstun = 24 } }, -- bonks on landing (GDD §7.3.2)
					stay = { kind = "sl", frames = 12, n = 15 }, teleAnchor = "carrier" }, -- slowed 15% within 4 studs (GDD §7.3.2)
			}),
			M("lifeguard.2", { -- gdd: §7.3.2 row 2
				behavior = "Rush", tags = { "Rush", "AoE" }, signature = true, damage = 15, cooldown = 16, -- GDD §7.3.2
				startup = 18, active = 5, recovery = 26, guardDamage = 50, hitstun = 40, -- 18f startup (GDD §7.3.2); rest (td)
				knockback = kb(10, 40, true), hitbox = sphere(16), -- everyone within 8 studs is splashed flat (GDD §7.3.2); (td)
				telegraph = tele("Circle", 18, 16, 16), -- circle telegraph on the landing spot (GDD §7.3.2)
				params = { travel = { from = 3, to = "startup", studs = 12, arc = "leap", height = 6 }, -- leaps 12 studs (GDD §7.3.2); height (td)
					teleFwd = 12, ragdollFrames = 60 }, -- landing-spot anchor (§4.2.0.6); (td)
			}),
			M("lifeguard.3", { -- gdd: §7.3.2 row 3
				behavior = "Projectile", tags = { "Projectile" }, damage = 4, cooldown = 12, -- GDD §7.3.2
				startup = 9, active = 2, recovery = 16, guardDamage = 20, hitstun = 20, -- (td)
				params = { kind = "bullet", speed = 100, range = 40, radius = 1.5, pierce = 0, reflectable = true, -- 40 studs: GDD §7.3.2; rest (td)
					onHit = { debuffs = { { kind = "rt", frames = 36, n = 1 } } } }, -- hoops the target: rooted 0.6 s (GDD §7.3.2)
			}),
			M("lifeguard.4", { -- gdd: §7.3.2 row 4
				behavior = "Grab", tags = { "Grab", "Extender" }, damage = 9, cooldown = 13, -- GDD §7.3.2
				startup = 9, active = 4, recovery = 22, hitbox = box(4, 5, 6), -- (td); startup 9 combos from M1 (K04)
				params = { groundOnly = true, holdF = 48, offset = v3(0, 3, -1), sub = 2, -- fireman's carry on her shoulder (GDD §7.3.2); (td)
					ticks = { { at = 20, damage = 4 }, { at = 44, damage = 5 } }, -- jogs a step (4), then heaves (5) (GDD §7.3.2)
					finish = "throw", throwDir = "input", finishKb = kb(20, 15, true), ragdollFrames = 60, finishRecovery = 16 }, -- the way you hold the stick (GDD §7.3.2)
			}),
		},
		awakening = {
			id = "lifeguard.awaken", durationSec = 60, hurtboxScale = 1, -- GDD §6.9; no size change (GDD §7.3.2)
			modifiers = { breakawayRefillMult = 1.5 }, -- Breakaway refills 50% faster (GDD §7.3.2 Peak)
			moves = {
				M("lifeguard.a1", { -- gdd: §7.3.2 A1
					behavior = "Zone", tags = { "Zone" }, damage = 14, cooldown = 8, -- GDD §7.3.2
					startup = 10, active = 1, recovery = 14, guardDamage = 24, hitstun = 18, -- (td)
					telegraph = tele("Circle", 20, 16, 16), -- 16-stud circle (GDD §7.3.2)
					params = { anchor = "aim", aimRange = 30, shape = "circle", radius = 8, armF = 20, lifeF = 300, -- 5 s (GDD §7.3.2); aimRange, armF (td)
						bursts = { { atF = 0, damage = 4, hitstun = 18 } }, -- the flag spike: 4 (GDD §7.3.2)
						tick = { everyF = 60, damage = 2, count = 5 }, stay = { kind = "sl", frames = 12, n = 30 }, -- stings 2/s, slows 30% (GDD §7.3.2)
						maxHitsPerTarget = 5, teleAnchor = "carrier" }, -- 4 + 5 × 2 = 14
				}),
				M("lifeguard.a2", { -- gdd: §7.3.2 A2
					behavior = "Projectile", tags = { "Projectile" }, damage = 12, cooldown = 11, -- GDD §7.3.2
					startup = 14, active = 2, recovery = 18, guardDamage = 36, hitstun = 30, knockback = kb(16, 10, false), -- washes everyone back (GDD §7.3.2); (td)
					telegraph = tele("Line", 14, 10, 30), -- (td)
					params = { kind = "wave", speed = 40, range = 30, width = 10, height = 5, pierce = -1, reflectable = false }, -- rolls 30 studs (GDD §7.3.2); (td)
				}),
				M("lifeguard.a3", { -- gdd: §7.3.2 A3
					behavior = "Rush", tags = { "Rush" }, damage = 10, cooldown = 12, -- GDD §7.3.2
					startup = 10, active = 18, recovery = 18, guardDamage = 30, hitstun = 30, -- (td)
					knockback = kb(10, 30, true), hitbox = box(5, 5, 5), telegraph = tele("Line", 10, 5, 24), -- bowls over everyone in her path (GDD §7.3.2); (td)
					params = { travel = { from = "active", to = "active", studs = 24 }, passThrough = true, ragdollFrames = 40 }, -- belly-surfs 24 studs (GDD §7.3.2); (td)
				}),
				M("lifeguard.a4", { -- gdd: §7.3.2 A4
					behavior = "Zone", tags = { "Zone", "Ultimate" }, damage = 28, cooldown = 25, hitstun = 0, -- GDD §7.3.2
					startup = 16, active = 1, recovery = 20, guardDamage = 40, -- (td)
					telegraph = tele("Circle", 30, 20, 20), -- (td)
					params = { anchor = "aim", aimRange = 40, shape = "circle", radius = 10, armF = 30, lifeF = 180, -- within 40 studs, 3 s (GDD §7.3.2); radius, armF (td)
						tick = { everyF = 22, damage = 3.5, count = 8, blockable = true }, maxHitsPerTarget = 8, -- 8 ticks of 3.5 (GDD §7.3.2)
						pullStudsPerSec = 4, teleAnchor = "carrier" }, -- drags enemies toward its middle (GDD §7.3.2); speed (td)
				}),
			},
		},
		m1 = { anim = "anim.lifeguard.m1", vfx = "vfx.lifeguard.m1", sfx = "sfx.lifeguard.m1" },
		kitProps = { "model.lifeguard.gear_buoy", "model.lifeguard.gear_visor" }, -- §4.7.4
	}
	return {
		def = def,
		hooks = {
			onHitDealt = function(h, info)
				if info.isEnder and info.outcome == "hit" then -- gdd: §7.3.2 Passive
					local p = h.style.passiveParams
					return { debuffsOnVictim = { { kind = "sl", frames = p.enderSlowFrames, n = p.enderSlowPct } } }
				end
				return nil
			end,
		},
		moment = { seq = { "lifeguard.3:hit", "lifeguard.4:hit" }, windowF = 180, sameVictim = true }, -- gdd: §7.3.2 Snapshot
		anim = {
			m1 = { windup = "windup_side", strike = "slash_side",
				alt = { "slash_side", "slash_side", "slash_down", "slash_side", "slash_side", "slash_down", "kick", "cast_push", "windup_high", "flop", "slash_side", "slash_side" } },
			q = { windup = "crouch", mid = "leap", strike = "stomp_down" },
			["1"] = { windup = "windup_high", strike = "throw" },
			["2"] = { windup = "crouch", mid = "leap", strike = "flop", face = { windup = "grit", active = "shout", recovery = "smile" } },
			["3"] = { windup = "windup_side", strike = "throw" },
			["4"] = { windup = "grab_reach", strike = "carry", finish = "throw" },
			a1 = { windup = "cast_raise", strike = "stomp_down" },
			a2 = { windup = "ready", strike = "cast_push" },
			a3 = { windup = "crouch", strike = "flop" },
			a4 = { windup = "cast_raise", strike = "spin_a", face = { windup = "awaken", active = "shout", recovery = "neutral" } },
			awaken = { windup = "crouch", strike = "cast_raise", face = { windup = "grit", active = "awaken", recovery = "smile" } }, -- anim.<style>.awaken (72f, §4.6.1)
		},
	}
end
```

#### 4.5.4 `sandcastle` (GDD §7.3.9, Rare, `MoleBuilder`)

Burrows, *The Tide-Proof Architect*: a mole who builds the perfect sandcastle every day; the tide takes it every night. M1: spade swipes; the Ender upends a bucket on the target's head.

```lua
--!strict
-- src/Shared/Styles/sandcastle.luau — sandcastle kit. Realm: shared. Pure: load. Spec §4.5.4
local Kit = require("../Behaviors/Buff")

return function(K: Kit.KitBuilder): Kit.StyleModule
	local M, kb, tele, v3 = K.move, K.kb, K.tele, K.v3
	local def = {
		id = "sandcastle", rarity = "rare", archetype = "MoleBuilder", implemented = true, hp = 100, -- GDD §7.2, §6.2
		palette = { primary = K.rgb(40, 60, 120), secondary = K.rgb(222, 196, 140), accent = K.rgb(200, 40, 40) }, -- §4.7.1
		passive = { id = "sandcastle.passive", params = {
			blockHeal = 1, blockHealEveryF = 60, -- GDD §7.3.9 Passive: a blocked hit heals 1 HP, at most once per second
		} },
		quirk = M("sandcastle.q", { -- gdd: §7.3.9 Quirk
			behavior = "Buff", tags = { "Buff" }, damage = 0, cooldown = 8, -- "0 (+4)" (GDD §7.3.9)
			startup = 8, active = 1, recovery = 10, -- (td)
			params = { target = "self", buffs = { { kind = "nh", frames = 180, n = 4 } } }, -- next hit within 3 s deals +4 (GDD §7.3.9)
		}),
		moves = {
			M("sandcastle.1", { -- gdd: §7.3.9 row 1
				behavior = "Projectile", tags = { "Projectile" }, damage = 7, cooldown = 7, -- GDD §7.3.9
				startup = 8, active = 2, recovery = 16, guardDamage = 22, hitstun = 26, -- (td)
				params = { kind = "bullet", speed = 110, range = 45, radius = 1.2, pierce = 0, reflectable = true, -- (td)
					onHit = { debuffs = { { kind = "rt", frames = 36, n = 1 } } }, -- roots the target 36f (GDD §7.3.9)
					extender = true }, -- K04: 8 + 1 + 3 ≤ 12
			}),
			M("sandcastle.2", { -- gdd: §7.3.9 row 2
				behavior = "Summon", tags = { "Summon" }, damage = 3, cooldown = 15, -- 3 per bonk (GDD §7.3.9)
				startup = 12, active = 1, recovery = 14, guardDamage = 10, hitstun = 18, -- (td)
				telegraph = tele("Circle", 12, 6, 6), -- (td) K08 (max 18 > 10)
				params = { dmgModel = "perHit", bodies = 1, hpEach = 40, lifeF = 480, -- one sandcastle, 40 HP, 8 s (GDD §7.3.9)
					speed = 0, ai = "chase", leash = 6, spawnOffsets = { v3(3, 0, 0) }, -- beside her, never moves (GDD §7.3.9; E2)
					attack = { range = 6, startupF = 12, everyF = 72, size = v3(5, 5, 7), tele = true }, -- within 6 studs, every 1.2 s (GDD §7.3.9); size (td)
					maxHitsPerTarget = 6, model = "model.sandcastle.summon_castle" }, -- max 6 bonks a target: 6 × 3 = 18 (GDD §7.3.9)
			}),
			M("sandcastle.3", { -- gdd: §7.3.9 row 3
				behavior = "Zone", tags = { "Zone" }, damage = 5, cooldown = 12, chip = 0, parryable = false, -- GDD §7.3.9
				startup = 10, active = 1, recovery = 14, -- (td)
				telegraph = tele("Line", 10, 6, 20), -- (td)
				params = { anchor = "self", shape = "line", radius = 20, width = 6, armF = 12, lifeF = 300, -- 20-stud trench, 5 s (GDD §7.3.9); width, armF (td)
					tick = { everyF = 60, damage = 1, count = 5 }, stay = { kind = "sl", frames = 12, n = 30 }, -- 1/s, slowed 30% (GDD §7.3.9)
					maxHitsPerTarget = 5 },
			}),
			M("sandcastle.4", { -- gdd: §7.3.9 row 4
				behavior = "Projectile", tags = { "Projectile" }, signature = true, damage = 16, cooldown = 18, -- GDD §7.3.9
				startup = 42, active = 2, recovery = 24, guardDamage = 50, hitstun = 40, knockback = kb(16, 15, true), -- 42f line telegraph; ragdoll (GDD §7.3.9)
				telegraph = tele("Line", 42, 6, 45), -- GDD §7.3.9; width (td)
				params = { kind = "wave", speed = 60, range = 45, width = 6, height = 5, pierce = -1, reflectable = false, -- surges 45 studs, everyone in the line once (GDD §7.3.9)
					ragdollFrames = 60 }, -- (td)
			}),
		},
		awakening = {
			id = "sandcastle.awaken", durationSec = 60, hurtboxScale = 1, -- GDD §6.9, §6.2
			modifiers = { enderDamageAdd = 2 }, -- her Ender deals +2 (GDD §7.3.9 Peak)
			moves = {
				M("sandcastle.a1", { -- gdd: §7.3.9 A1
					behavior = "Zone", tags = { "Zone" }, damage = 15, cooldown = 9, -- GDD §7.3.9
					startup = 12, active = 2, recovery = 18, guardDamage = 40, hitstun = 36, -- (td)
					telegraph = tele("Line", 12, 4, 12), -- 12 × 4 studs (GDD §7.3.9)
					params = { anchor = "self", shape = "line", radius = 12, width = 4, armF = 0, lifeF = 0, -- instant burst (GDD §7.3.9)
						bursts = { { atF = 0, damage = 15, hitstun = 36, knockback = kb(6, 0, true), ragdollFrames = 48 } } }, -- slams down (GDD §7.3.9); (td)
				}),
				M("sandcastle.a2", { -- gdd: §7.3.9 A2
					behavior = "Summon", tags = { "Summon" }, damage = 7, cooldown = 11, -- 7 per bonk (GDD §7.3.9)
					startup = 12, active = 1, recovery = 14, guardDamage = 16, hitstun = 24, -- (td)
					telegraph = tele("Circle", 12, 8, 8), -- (td) K08 (max 14 > 10)
					params = { dmgModel = "perHit", bodies = 1, hpEach = 60, lifeF = 480, -- the deluxe castle, 60 HP, 8 s (GDD §7.3.9)
						speed = 0, ai = "chase", leash = 6, spawnOffsets = { v3(-3, 0, 0) }, -- never moves (GDD §7.3.9; E2); offset (td)
						attack = { range = 6, startupF = 12, everyF = 72, size = v3(6, 6, 8), tele = true }, -- (td)
						maxHitsPerTarget = 2, model = "model.sandcastle.summon_grandcastle" }, -- max 2 bonks a target: 2 × 7 = 14 (GDD §7.3.9)
				}),
				M("sandcastle.a3", { -- gdd: §7.3.9 A3
					behavior = "Counter", tags = { "Counter" }, damage = 0, cooldown = 14, -- GDD §7.3.9
					startup = 4, active = 48, recovery = 14, -- sandbags up for 0.8 s (GDD §7.3.9); (td)
					params = { arcDeg = 180, vsMelee = "stagger", staggerF = 24, reflectProjectiles = false, endOnTrigger = false }, -- stagger 24f; projectiles get through (GDD §7.3.9)
				}),
				M("sandcastle.a4", { -- gdd: §7.3.9 A4
					behavior = "Zone", tags = { "Zone", "Ultimate" }, damage = 24, cooldown = 25, -- GDD §7.3.9
					startup = 16, active = 1, recovery = 20, guardDamage = 50, hitstun = 30, -- (td)
					telegraph = tele("Circle", 30, 50, 50), -- (td)
					params = { anchor = "self", shape = "ring", radius = 25, innerRadius = 8, armF = 30, lifeF = 0, -- between 8 and 25 studs (GDD §7.3.9); armF (td)
						bursts = { -- three tide waves of 8, each pushing them out (GDD §7.3.9); spacing (td)
							{ atF = 0, damage = 8, hitstun = 30, knockback = kb(10, 0, false) },
							{ atF = 45, damage = 8, hitstun = 30, knockback = kb(10, 0, false) },
							{ atF = 90, damage = 8, hitstun = 30, knockback = kb(10, 0, false) },
						}, teleAnchor = "carrier" },
				}),
			},
		},
		m1 = { anim = "anim.sandcastle.m1", vfx = "vfx.sandcastle.m1", sfx = "sfx.sandcastle.m1" },
		kitProps = { "model.sandcastle.gear_spade", "model.sandcastle.gear_hardhat", -- §4.7.4
			"model.sandcastle.summon_castle", "model.sandcastle.summon_grandcastle" }, -- §4.7.5
	}
	return {
		def = def,
		hooks = {}, -- the block heal lives in Styles.serverHook with server memory (§4.3.6), keyed by styleId "sandcastle"
		moment = { seq = { "sandcastle.3:hit", "sandcastle.4:hit" }, windowF = 180, sameVictim = true }, -- gdd: §7.3.9 Snapshot; (td)
		anim = {
			m1 = { windup = "windup_side", strike = "slash_side",
				alt = { "slash_side", "slash_side", "slash_down", "slash_side", "slash_side", "slash_down", "kick", "slash_down", "windup_high", "stomp_down", "slash_side", "slash_side" } },
			q = { windup = "ready", strike = "point" },
			["1"] = { windup = "dig", strike = "throw" },
			["2"] = { windup = "dig", strike = "slash_down" },
			["3"] = { windup = "dig", strike = "dig" },
			["4"] = { windup = "dig", strike = "cast_push", face = { windup = "grit", active = "shout", recovery = "smile" } },
			a1 = { windup = "windup_high", strike = "slash_down" },
			a2 = { windup = "dig", strike = "cast_raise" },
			a3 = { windup = "ready", strike = "guard_wide" },
			a4 = { windup = "cast_raise", strike = "stomp_down", face = { windup = "awaken", active = "shout", recovery = "smile" } },
			awaken = { windup = "crouch", strike = "cast_raise", face = { windup = "grit", active = "awaken", recovery = "smile" } }, -- anim.<style>.awaken (72f, §4.6.1)
		},
	}
end
```

#### 4.5.5 `busker` (GDD §7.3.12, Epic, `GullBusker`)

Bellows, *The Pier Busker*: an accordion-busking gull and snack thief. He squeezes the accordion; he never swings it. M1: wing slaps; the Ender is a wheezing accordion blast.

```lua
--!strict
-- src/Shared/Styles/busker.luau — busker kit. Realm: shared. Pure: load. Spec §4.5.5
local Kit = require("../Behaviors/Buff")

return function(K: Kit.KitBuilder): Kit.StyleModule
	local M, box, kb, tele, v3 = K.move, K.box, K.kb, K.tele, K.v3
	local def = {
		id = "busker", rarity = "epic", archetype = "GullBusker", implemented = true, hp = 100, -- GDD §7.2, §6.2
		palette = { primary = K.rgb(222, 170, 40), secondary = K.rgb(150, 156, 165), accent = K.rgb(40, 140, 140) }, -- §4.7.1
		passive = { id = "busker.passive", params = {
			enderCdSec = 1, -- GDD §7.3.12 Passive: each Ender he lands ticks his move cooldowns 1 s down; no heal
		} },
		quirk = M("busker.q", { -- gdd: §7.3.12 Quirk
			behavior = "Projectile", tags = { "Projectile" }, damage = 3, cooldown = 5, -- GDD §7.3.12
			startup = 8, active = 2, recovery = 12, guardDamage = 20, hitstun = 18, -- 18f flinch (GDD §7.3.12); rest (td)
			params = { kind = "bullet", speed = 80, range = 30, radius = 1.2, pierce = 0, reflectable = true, -- skims his cap 30 studs (GDD §7.3.12); (td)
				onHit = { stun = "flinch" } }, -- grounded, no i-frames (GDD §7.3.12)
		}),
		moves = {
			M("busker.1", { -- gdd: §7.3.12 row 1
				behavior = "Projectile", tags = { "Projectile", "Extender", "Music" }, damage = 9, cooldown = 8, -- GDD §7.3.12, Appendix A
				startup = 10, active = 5, recovery = 20, guardDamage = 30, hitstun = 30, -- GDD Appendix A
				knockback = kb(6, 0, false), telegraph = tele("Line", 8, 3, 20), -- shoves the target back 6 studs; Line20, lead 8 (GDD Appendix A); width (td)
				params = { kind = "bullet", speed = 90, range = 20, radius = 1.5, pierce = 0, reflectable = false, -- a puff of air, 20 studs (GDD §7.3.12); speed, radius (td, GDD Appendix A)
					cancelInto = { "m1" }, cancelWindow = { 21, 30 }, cancelChainIndex = 2 }, -- cancels into M1-2 on e 21–30 (GDD Appendix A)
			}),
			M("busker.2", { -- gdd: §7.3.12 row 2
				behavior = "Summon", tags = { "Summon" }, damage = 3, cooldown = 14, -- 3 per peck (GDD §7.3.12)
				startup = 10, active = 1, recovery = 12, guardDamage = 8, hitstun = 12, -- (td)
				params = { dmgModel = "perHit", bodies = 1, hpEach = 20, lifeF = 360, -- one gull, 20 HP, 6 s, always visible (GDD §7.3.12)
					speed = 14, ai = "chase", leash = 40, spawnOffsets = { v3(2, 0, -2) }, -- heckles the nearest enemy (GDD §7.3.12); (td)
					attack = { range = 4, startupF = 10, everyF = 72, size = v3(3, 4, 4), stun = "flinch" }, -- a peck every 1.2 s (GDD §7.3.12); (td)
					maxHitsPerTarget = 2, model = "model.busker.summon_heckler" }, -- max 2: 2 × 3 = 6 (GDD §7.3.12)
			}),
			M("busker.3", { -- gdd: §7.3.12 row 3
				behavior = "Grab", tags = { "Grab" }, damage = 10, cooldown = 14, -- GDD §7.3.12
				startup = 9, active = 4, recovery = 22, hitbox = box(4, 5, 6), -- (td)
				params = { groundOnly = true, holdF = 36, offset = v3(0, 0, -2.5), sub = 1, finish = "release", -- (td)
					ticks = { { at = 12, damage = 3 }, { at = 30, damage = 7, meterDrain = 15 } }, -- two pecks 3 + 7; −15 Peak Meter; no heal (GDD §7.3.12)
					finishKb = kb(5, 0, false), finishHitstun = 30, finishRecovery = 14 }, -- release pushes 5 studs so the crash reaches (td; GDD §7.3.12 Snapshot)
			}),
			M("busker.4", { -- gdd: §7.3.12 row 4
				behavior = "Zone", tags = { "Zone" }, signature = true, damage = 12, cooldown = 16, -- GDD §7.3.12
				startup = 14, active = 2, recovery = 24, guardDamage = 40, hitstun = 30, knockback = kb(12, 0, false), -- 14f startup; blasted back 12 studs (GDD §7.3.12)
				telegraph = tele("Circle", 14, 20, 20), -- everyone within 10 studs (GDD §7.3.12)
				params = { anchor = "self", shape = "circle", radius = 10, armF = 0, lifeF = 0, -- instant burst (GDD §7.3.12)
					bursts = { { atF = 0, damage = 12, hitstun = 30, knockback = kb(12, 0, false) } } },
			}),
		},
		awakening = {
			id = "busker.awaken", durationSec = 60, hurtboxScale = 1, -- GDD §6.9, §6.2
			modifiers = { airDash = true }, -- he can dash once in mid-air (GDD §7.3.12 Peak)
			moves = {
				M("busker.a1", { -- gdd: §7.3.12 A1
					behavior = "Projectile", tags = { "Projectile" }, damage = 8, cooldown = 12, -- GDD §7.3.12
					startup = 12, active = 2, recovery = 18, guardDamage = 24, hitstun = 24, knockback = kb(10, 10, false), -- knocks everyone back (GDD §7.3.12); (td)
					telegraph = tele("Cone", 12, 16, 30), -- (td)
					params = { kind = "wave", speed = 50, range = 30, width = 4, widthEnd = 16, height = 5, pierce = -1, -- a widening wave, 30 studs (GDD §7.3.12); widths (td)
						reflectable = false },
				}),
				M("busker.a2", { -- gdd: §7.3.12 A2
					behavior = "Rush", tags = { "Rush" }, damage = 16, cooldown = 9, -- GDD §7.3.12
					startup = 10, active = 28, recovery = 18, guardDamage = 40, hitstun = 20, knockback = kb(4, 0, false), -- (td); a 4-stud push keeps the target in reach
					hitbox = box(6, 6, 5), telegraph = tele("Line", 10, 6, 16), -- (td)
					params = { travel = { from = "active", to = "active", studs = 16 }, -- drums his way 16 studs forward (GDD §7.3.12)
						hits = { 4, 4, 4, 4 }, hitFrames = { 1, 8, 15, 22 }, -- four beats of 4 (GDD §7.3.12)
						lastHit = { hitstun = 32, knockback = kb(12, 20, false) } }, -- (td)
				}),
				M("busker.a3", { -- gdd: §7.3.12 A3
					behavior = "Summon", tags = { "Summon" }, damage = 2, cooldown = 16, -- 2 per peck (GDD §7.3.12)
					startup = 12, active = 1, recovery = 14, guardDamage = 8, hitstun = 12, -- (td)
					telegraph = tele("Circle", 12, 4, 4), -- (td) K08 (max 12 > 10)
					params = { dmgModel = "perHit", bodies = 3, hpEach = 7, lifeF = 360, -- three gulls, 7 HP each, 6 s (GDD §7.3.12)
						speed = 16, ai = "chase", leash = 40, spawnOffsets = { v3(-3, 0, -2), v3(0, 0, -3), v3(3, 0, -2) }, -- (td)
						attack = { range = 4, startupF = 12, everyF = 60, size = v3(3, 4, 4), stun = "flinch", tele = true }, -- (td)
						maxHitsPerTarget = 2, model = "model.busker.summon_gull" }, -- max 2 pecks each: 3 × 2 × 2 = 12 (GDD §7.3.12)
				}),
				M("busker.a4", { -- gdd: §7.3.12 A4
					behavior = "Zone", tags = { "Zone", "Ultimate" }, damage = 26, cooldown = 25, -- GDD §7.3.12
					startup = 14, active = 1, recovery = 24, guardDamage = 50, hitstun = 40, -- (td)
					telegraph = tele("Circle", 20, 36, 36), -- everyone within 18 studs (GDD §7.3.12)
					params = { anchor = "self", shape = "circle", radius = 18, armF = 20, lifeF = 180, -- stationary, 3 s (GDD §7.3.12); armF (td)
						tick = { everyF = 45, damage = 3, count = 4, blockable = true }, maxHitsPerTarget = 4, -- 3 on each of four beats (GDD §7.3.12)
						stay = { kind = "sl", frames = 12, n = 40 }, -- sways along, −40% speed (GDD §7.3.12)
						bursts = { { atF = 180, damage = 14, hitstun = 40, knockback = kb(8, 20, true), ragdollFrames = 60 } }, -- the big finish knocks them flat (14) (GDD §7.3.12)
						teleAnchor = "carrier" },
				}),
			},
		},
		m1 = { anim = "anim.busker.m1", vfx = "vfx.busker.m1", sfx = "sfx.busker.m1" },
		kitProps = { "model.busker.gear_accordion", "model.busker.gear_cap", -- §4.7.4
			"model.busker.summon_heckler", "model.busker.summon_gull" }, -- §4.7.5
	}
	return {
		def = def,
		hooks = {
			onHitDealt = function(h, info)
				if info.isEnder and info.outcome == "hit" then -- gdd: §7.3.12 Passive (no heal)
					return { cooldownsMinusSec = h.style.passiveParams.enderCdSec }
				end
				return nil
			end,
		},
		moment = { seq = { "busker.3:hit", "busker.4:hit" }, windowF = 180, sameVictim = true }, -- gdd: §7.3.12 Snapshot
		anim = {
			m1 = { windup = "windup_side", strike = "slash_side",
				alt = { "slash_side", "slash_side", "slash_down", "slash_side", "slash_side", "slash_down", "kick", "squeeze", "windup_high", "slash_down", "slash_side", "slash_side" } },
			q = { windup = "windup_side", strike = "throw" },
			["1"] = { windup = "squeeze", strike = "cast_push" },
			["2"] = { windup = "cast_raise", strike = "point" },
			["3"] = { windup = "grab_reach", strike = "grab_hold", face = { windup = "smile", active = "smile", recovery = "neutral" } },
			["4"] = { windup = "guard_wide", strike = "cast_push", face = { windup = "grit", active = "shout", recovery = "smile" } },
			a1 = { windup = "squeeze", strike = "cast_push" },
			a2 = { windup = "ready", strike = "punch_r", alt = { "punch_r", "punch_l" }, finish = "stomp_down" },
			a3 = { windup = "cast_raise", strike = "point" },
			a4 = { windup = "squeeze", strike = "guard_wide", face = { windup = "awaken", active = "shout", recovery = "grit" } },
			awaken = { windup = "crouch", strike = "cast_raise", face = { windup = "grit", active = "awaken", recovery = "smile" } }, -- anim.<style>.awaken (72f, §4.6.1)
		},
	}
end
```

#### 4.5.6 `scoop` (GDD §7.3.10, Rare, `SealScooper`)

Dollop, *Every Flavour Ever*: a cheerful, slightly flustered spotted harbour seal who stocks every flavour a guest ever asked for. Her moves are ice-cream verbs (scoop, drip, melt, sprinkle, wafer); she never freezes anyone. One mechanic: sticky slows that feed her passive. M1: cone bops; the Ender plops a scoop on the head.

```lua
--!strict
-- src/Shared/Styles/scoop.luau — scoop kit. Realm: shared. Pure: load. Spec §4.5.6
local Kit = require("../Behaviors/Buff")

return function(K: Kit.KitBuilder): Kit.StyleModule
	local M, box, kb, tele, v3 = K.move, K.box, K.kb, K.tele, K.v3
	local def = {
		id = "scoop", rarity = "rare", archetype = "SealScooper", implemented = true, hp = 100, -- GDD §7.2, §6.2
		palette = { primary = K.rgb(200, 40, 110), secondary = K.rgb(160, 225, 195), accent = K.rgb(250, 240, 215) }, -- §4.7.1
		passive = { id = "scoop.passive", params = {
			toppingsCdSec = 0.5, -- GDD §7.3.10 Passive: once per move use, a hit on a slowed target knocks 0.5 s off her cooldowns
		} },
		quirk = M("scoop.q", { -- gdd: §7.3.10 Quirk
			behavior = "Zone", tags = { "Zone" }, damage = 3, cooldown = 6, -- GDD §7.3.10
			startup = 8, active = 2, recovery = 14, guardDamage = 20, hitstun = 18, -- 18f flinch (GDD §7.3.10); rest (td)
			params = { anchor = "self", shape = "circle", radius = 8, armF = 0, lifeF = 0, -- enemies within 8 studs (GDD §7.3.10)
				bursts = { { atF = 0, damage = 3, hitstun = 18, stun = "flinch" } } }, -- instant burst (GDD §7.3.10)
		}),
		moves = {
			M("scoop.1", { -- gdd: §7.3.10 row 1
				behavior = "Projectile", tags = { "Projectile" }, damage = 8, cooldown = 6, -- GDD §7.3.10
				startup = 8, active = 1, recovery = 16, guardDamage = 22, hitstun = 26, -- (td)
				params = { kind = "lob", range = 35, aimRange = 35, count = 1, flightF = 24, impactRadius = 3, -- a marked spot up to 35 studs away (GDD §7.3.10); rest (td)
					pattern = { v3(0, 0, 0) }, reflectable = false, teleAnchor = "carrier",
					onHit = { debuffs = { { kind = "sl", frames = 120, n = 20 } } } }, -- splats: −20% speed for 2 s (GDD §7.3.10)
			}),
			M("scoop.2", { -- gdd: §7.3.10 row 2
				behavior = "Grab", tags = { "Grab", "Extender" }, damage = 8, cooldown = 14, -- GDD §7.3.10
				startup = 9, active = 4, recovery = 22, hitbox = box(4, 5, 6), -- (td); startup 9 combos from M1 (K04)
				params = { groundOnly = true, holdF = 36, offset = v3(0, 0, -2.5), sub = 1, finish = "release", -- (td)
					ticks = { { at = 12, damage = 3 }, { at = 30, damage = 5 } }, -- two ticks, 3 + 5 (GDD §7.3.10)
					finishKb = kb(6, 0, false), finishHitstun = 30, finishStun = "stumble", finishRecovery = 14 }, -- stagger back, stumble 30f (GDD §7.3.10)
			}),
			M("scoop.3", { -- gdd: §7.3.10 row 3
				behavior = "Mobility", tags = { "Mobility" }, damage = 0, cooldown = 9, -- GDD §7.3.10
				startup = 1, active = 14, recovery = 10, -- (td)
				params = { mode = "dash", studs = 18, iFrames = { from = 1, to = 8 } }, -- slides out 18 studs, untouchable for the first 8 frames (GDD §7.3.10)
			}),
			M("scoop.4", { -- gdd: §7.3.10 row 4
				behavior = "Zone", tags = { "Zone" }, signature = true, damage = 12, cooldown = 20, -- GDD §7.3.10
				startup = 10, active = 1, recovery = 18, guardDamage = 36, hitstun = 30, -- (td)
				telegraph = tele("Circle", 30, 10, 10), -- 0.5 s telegraph (GDD §7.3.10); size (td)
				params = { anchor = "aim", aimRange = 40, shape = "circle", radius = 5, armF = 30, lifeF = 240, -- a sticky puddle for 4 s (GDD §7.3.10); range, radius (td)
					bursts = { { atF = 0, damage = 12, hitstun = 30 } }, -- a giant scoop drops (12) (GDD §7.3.10)
					stay = { kind = "sl", frames = 12, n = 40 }, teleAnchor = "carrier" }, -- −40% speed (GDD §7.3.10)
			}),
		},
		awakening = {
			id = "scoop.awaken", durationSec = 60, hurtboxScale = 1, -- GDD §6.9, §6.2
			modifiers = { breakawayRefillMult = 1.25 }, -- Breakaway refills 25% faster (GDD §7.3.10 Peak)
			moves = {
				M("scoop.a1", { -- gdd: §7.3.10 A1
					behavior = "Projectile", tags = { "Projectile" }, damage = 10, cooldown = 14, -- GDD §7.3.10
					startup = 12, active = 9, recovery = 18, guardDamage = 30, hitstun = 24, -- (td)
					params = { kind = "lob", range = 35, aimRange = 35, count = 3, everyF = 4, flightF = 30, impactRadius = 4, -- three handfuls on telegraphed spots (GDD §7.3.10); (td)
						pattern = { v3(0, 0, 0), v3(5, 0, 4), v3(-5, 0, 4) }, hits = { 3, 3, 4 }, -- 3 + 3 + 4 (GDD §7.3.10)
						reflectable = false, teleAnchor = "carrier" },
				}),
				M("scoop.a2", { -- gdd: §7.3.10 A2
					behavior = "Zone", tags = { "Zone" }, damage = 14, cooldown = 10, -- GDD §7.3.10
					startup = 12, active = 2, recovery = 18, guardDamage = 40, hitstun = 36, knockback = kb(8, 0, false), -- (td)
					telegraph = tele("Cone", 12, 14, 12), -- a 12-stud cone (GDD §7.3.10); x = 2·12·tan30°
					params = { anchor = "self", shape = "cone", radius = 12, coneDeg = 60, armF = 0, lifeF = 0, -- instant burst (GDD §7.3.10); angle (td)
						bursts = { { atF = 0, damage = 14, hitstun = 36, stun = "stumble", knockback = kb(8, 0, false) } } }, -- stumble back 36f (GDD §7.3.10)
				}),
				M("scoop.a3", { -- gdd: §7.3.10 A3
					behavior = "Projectile", tags = { "Projectile" }, damage = 12, cooldown = 12, -- GDD §7.3.10
					startup = 10, active = 9, recovery = 18, guardDamage = 30, hitstun = 20, -- (td)
					telegraph = tele("Line", 10, 3, 40), -- (td) K08 (12 > 10)
					params = { kind = "bullet", count = 3, everyF = 4, speed = 100, range = 40, radius = 1, -- three wafers (GDD §7.3.10); (td)
						hits = { 4, 4, 4 }, pierce = 0, reflectable = true }, -- 4 each (GDD §7.3.10)
				}),
				M("scoop.a4", { -- gdd: §7.3.10 A4
					behavior = "Summon", tags = { "Summon", "Ultimate" }, damage = 24, cooldown = 25, -- GDD §7.3.10
					startup = 16, active = 1, recovery = 20, guardDamage = 50, hitstun = 40, knockback = kb(14, 15, true), -- flattens everyone in its path (GDD §7.3.10)
					telegraph = tele("Line", 16, 6, 40), -- (td)
					params = { dmgModel = "total", bodies = 1, hpEach = 40, lifeF = 360, speed = 0, ai = "line", -- then parks (40 HP) until it melts away (GDD §7.3.10); life (td, K09)
						line = { studs = 40, frames = 24, width = 6 }, spawnOffsets = { v3(0, 0, 3) }, maxHitsPerTarget = 1, -- rolls down a 40-stud line (GDD §7.3.10); (td)
						ragdollFrames = 48, model = "model.scoop.summon_van" }, -- (td)
				}),
			},
		},
		m1 = { anim = "anim.scoop.m1", vfx = "vfx.scoop.m1", sfx = "sfx.scoop.m1" },
		kitProps = { "model.scoop.gear_cone", "model.scoop.gear_cap", "model.scoop.summon_van" }, -- §4.7.4, §4.7.5
	}
	return {
		def = def,
		hooks = {
			onHitDealt = function(h, info)
				if info.outcome ~= "hit" or not info.firstOfUse or info.tick or info.damage <= 0 then
					return nil -- once per move use; zone ticks and zero-damage requests never count (td)
				end
				if info.kind == "m1" or info.kind == "ender" or info.kind == "airm1" or info.kind == "dashm1" then
					return nil -- move hits only (td, §4.11 Q2)
				end
				local mv = h.moves(info.moveId)
				if mv == nil or Kit.appliesStatus(mv, "sl") then
					return nil -- the slowing hit itself does not count (td)
				end
				for _, st in info.victimDebuffs do
					if st.kind == "sl" then -- gdd: §7.3.10 Passive
						return { cooldownsMinusSec = h.style.passiveParams.toppingsCdSec }
					end
				end
				return nil
			end,
		},
		moment = { seq = { "scoop.1:hit", "scoop.2:hit" }, windowF = 240, sameVictim = true }, -- gdd: §7.3.10 Snapshot; window (td)
		anim = {
			m1 = { windup = "punch_wind", strike = "punch_r",
				alt = { "punch_r", "punch_l", "punch_r", "punch_r", "punch_l", "punch_r", "kick", "slash_down", "windup_high", "stomp_down", "punch_l", "punch_l" } },
			q = { windup = "ready", strike = "point", face = { windup = "smile", active = "shout", recovery = "smile" } },
			["1"] = { windup = "windup_high", strike = "throw" },
			["2"] = { windup = "grab_reach", strike = "grab_hold", finish = "cast_push" },
			["3"] = { windup = "crouch", strike = "slide" },
			["4"] = { windup = "cast_raise", strike = "slash_down", face = { windup = "grit", active = "shout", recovery = "smile" } },
			a1 = { windup = "windup_high", strike = "throw" },
			a2 = { windup = "ready", strike = "cast_push" },
			a3 = { windup = "windup_side", strike = "throw" },
			a4 = { windup = "cast_raise", strike = "point", face = { windup = "awaken", active = "shout", recovery = "smile" } },
			awaken = { windup = "crouch", strike = "cast_raise", face = { windup = "grit", active = "awaken", recovery = "smile" } }, -- anim.<style>.awaken (72f, §4.6.1)
		},
	}
end
```

#### 4.5.7 `dustsheet` (GDD §7.3.15, Legendary, `DustSheetSpirit`)

Mothball, *Under Wraps*: a dust-sheet spirit who covers things so they keep until next season. Not evil. M1: sheet flicks; the Ender wraps and spins the target.

```lua
--!strict
-- src/Shared/Styles/dustsheet.luau — dustsheet kit. Realm: shared. Pure: load. Spec §4.5.7
local Kit = require("../Behaviors/Buff")

return function(K: Kit.KitBuilder): Kit.StyleModule
	local M, box, kb, tele, v3 = K.move, K.box, K.kb, K.tele, K.v3
	local def = {
		id = "dustsheet", rarity = "legendary", archetype = "DustSheetSpirit", implemented = true, hp = 100, -- GDD §7.2, §6.2
		palette = { primary = K.rgb(150, 120, 200), secondary = K.rgb(235, 228, 210), accent = K.rgb(230, 160, 50) }, -- §4.7.1
		passive = { id = "dustsheet.passive", params = {
			maxStacks = 5, stackFrames = 480, perStack = 1, maxBonus = 5, -- GDD §7.3.15 Passive: Dust, max 5, 8 s, +1 per stack (max +5)
		} },
		quirk = M("dustsheet.q", { -- gdd: §7.3.15 Quirk
			behavior = "Buff", tags = { "Debuff" }, damage = 0, cooldown = 8, -- GDD §7.3.15
			startup = 8, active = 1, recovery = 10, -- (td)
			params = { target = "front", range = 20, width = 6, debuffs = { { kind = "rt", frames = 30, n = 1 } } }, -- within 20 studs, rooted 0.5 s (GDD §7.3.15)
		}),
		moves = {
			M("dustsheet.1", { -- gdd: §7.3.15 row 1
				behavior = "Projectile", tags = { "Projectile" }, damage = 8, cooldown = 6, -- GDD §7.3.15
				startup = 8, active = 2, recovery = 15, guardDamage = 22, hitstun = 28, -- (td)
				params = { kind = "bullet", speed = 100, range = 40, radius = 1.3, pierce = 0, reflectable = true, -- (td)
					dustStacks = 1, extender = true }, -- +1 Dust (GDD §7.3.15); K04: 8 + 1 + 3 ≤ 12
			}),
			M("dustsheet.2", { -- gdd: §7.3.15 row 2
				behavior = "Counter", tags = { "Counter" }, damage = 0, cooldown = 12, -- GDD §7.3.15
				startup = 4, active = 60, recovery = 14, -- ducks under her sheet for 1 s (GDD §7.3.15); (td)
				params = { arcDeg = 180, vsMelee = "stagger", staggerF = 30, reflectProjectiles = false, endOnTrigger = false }, -- tangled: stagger 30f; projectiles pass through (GDD §7.3.15)
			}),
			M("dustsheet.3", { -- gdd: §7.3.15 row 3
				behavior = "Summon", tags = { "Summon" }, damage = 3, cooldown = 16, -- 3 per bump (GDD §7.3.15)
				startup = 12, active = 1, recovery = 14, guardDamage = 8, hitstun = 18, knockback = kb(3, 0, false), -- trips them (GDD §7.3.15); (td)
				telegraph = tele("Circle", 12, 4, 4), -- (td) K08 (max 12 > 10)
				params = { dmgModel = "perHit", bodies = 2, hpEach = 10, lifeF = 360, -- two pieces of luggage, 10 HP each, 6 s (GDD §7.3.15)
					speed = 12, ai = "chase", leash = 40, spawnOffsets = { v3(-2.5, 0, -2), v3(2.5, 0, -2) }, -- shuffle after the nearest enemy (GDD §7.3.15); (td)
					attack = { range = 4, startupF = 12, everyF = 72, size = v3(3, 3, 4), stun = "stumble", tele = true }, -- (td)
					maxHitsPerTarget = 2, model = "model.dustsheet.summon_luggage" }, -- max 2 bumps each: 2 × 2 × 3 = 12 (GDD §7.3.15)
			}),
			M("dustsheet.4", { -- gdd: §7.3.15 row 4
				behavior = "Projectile", tags = { "Projectile" }, signature = true, damage = 12, cooldown = 20, -- GDD §7.3.15
				startup = 10, active = 1, recovery = 22, guardDamage = 36, hitstun = 30, -- (td)
				telegraph = tele("Circle", 24, 10, 10), -- landing circle shown in flight (GDD §7.3.15); size (td)
				params = { kind = "lob", range = 35, aimRange = 35, count = 1, flightF = 24, impactRadius = 5, -- a marked spot (GDD §7.3.15); (td)
					pattern = { v3(0, 0, 0) }, reflectable = false, teleAnchor = "carrier",
					onHit = { debuffs = { { kind = "rt", frames = 48, n = 1 } } }, -- draped where they stand, rooted 48f (GDD §7.3.15)
					dustStacks = 2 }, -- +2 Dust (GDD §7.3.15)
			}),
		},
		awakening = {
			id = "dustsheet.awaken", durationSec = 60, hurtboxScale = 1, -- GDD §6.9; no size change (GDD §7.3.15)
			modifiers = { walkSpeedMult = 1.1 }, -- glides 10% faster (GDD §7.3.15 Peak)
			moves = {
				M("dustsheet.a1", { -- gdd: §7.3.15 A1
					behavior = "Projectile", tags = { "Projectile" }, damage = 16, cooldown = 10, -- GDD §7.3.15
					startup = 14, active = 6, recovery = 20, guardDamage = 45, hitstun = 36, knockback = kb(10, 15, true), -- knockdown (GDD §7.3.15); (td)
					telegraph = tele("Line", 14, 3, 25), -- (td)
					params = { kind = "beam", range = 25, width = 3, extendF = 6, reflectable = false, -- flaps a long sheet out 25 studs (GDD §7.3.15); (td)
						dustStacks = 2, ragdollFrames = 48 }, -- +2 Dust (GDD §7.3.15); (td)
				}),
				M("dustsheet.a2", { -- gdd: §7.3.15 A2
					behavior = "Rush", tags = { "Rush" }, damage = 15, cooldown = 14, -- GDD §7.3.15
					startup = 12, active = 14, recovery = 18, guardDamage = 40, hitstun = 36, knockback = kb(8, 20, false), -- (td)
					hitbox = box(5, 6, 5), telegraph = tele("Line", 12, 5, 20), -- (td)
					params = { travel = { from = "active", to = "active", studs = 20 }, passThrough = true, -- drifts through everyone on a 20-stud line (GDD §7.3.15)
						dustStacks = 1 }, -- +1 Dust (GDD §7.3.15)
				}),
				M("dustsheet.a3", { -- gdd: §7.3.15 A3
					behavior = "Zone", tags = { "Zone" }, damage = 3, cooldown = 10, guardDamage = 10, hitstun = 0, -- 3 per touch (GDD §7.3.15)
					startup = 8, active = 1, recovery = 10, telegraph = tele("Circle", 8, 12, 12), -- (td)
					params = { dmgModel = "perHit", anchor = "selfFollow", shape = "ring", radius = 6, innerRadius = 3, armF = 0, -- sheets whirl round her (GDD §7.3.15); radii (td)
						lifeF = 240, tick = { everyF = 30, damage = 3, blockable = true }, maxHitsPerTarget = 6 }, -- 4 s (GDD §7.3.15); 6 × 3 = 18 ≤ 20
				}),
				M("dustsheet.a4", { -- gdd: §7.3.15 A4
					behavior = "RealmCast", tags = { "Realm" }, damage = 0, cooldown = 0, -- the Getaway; once per Peak (GDD §6.10)
					startup = 90, active = 1, recovery = 0, -- 90f cast (GDD §6.10)
					telegraph = tele("Circle", 90, 80, 80), -- 40-stud radius ring (GDD §6.10)
					params = { getaway = { -- render timing of Realm.rules["dustsheet.a4"] (§4.4.3)
						quietFirstF = 120, quietEveryF = 240, quietLeadF = 30, -- Quiet Hours at 2 s, then every 4 s; bell 0.5 s ahead (GDD §7.3.15)
						dustFirstF = 240, dustEveryF = 240, -- Dust Sheets every 4 s (GDD §7.3.15)
						lightsFirstF = 180, lightsEveryF = 180, lightsLeadF = 60, -- Lights Out every 3 s; next quarter flickers 1 s ahead (GDD §7.3.15)
					} },
				}),
			},
		},
		m1 = { anim = "anim.dustsheet.m1", vfx = "vfx.dustsheet.m1", sfx = "sfx.dustsheet.m1" },
		kitProps = { "model.dustsheet.gear_sheet", "model.dustsheet.gear_tag", "model.dustsheet.summon_luggage" }, -- §4.7.4, §4.7.5
	}
	return {
		def = def,
		hooks = {
			onHitDealt = function(h, info)
				if info.outcome ~= "hit" or not info.firstOfUse or info.tick or info.damage <= 0 then
					return nil -- blocked hits, repeat hits of one use, zone ticks and zero-damage requests add no Dust (td)
				end
				local n = 0
				if info.kind == "ender" then
					n = 1 -- M1 family: Ender only (td, §4.11 Q1)
				elseif info.kind ~= "m1" and info.kind ~= "airm1" and info.kind ~= "dashm1" then
					local mv = h.moves(info.moveId)
					n = if mv ~= nil and mv.params ~= nil and mv.params.dustStacks ~= nil then mv.params.dustStacks else 1 -- gdd: §7.3.15 Passive
				end
				if n <= 0 then
					return nil
				end
				return { debuffsOnVictim = { { kind = "du", frames = h.style.passiveParams.stackFrames, n = n } } } -- 8 s (GDD §7.3.15)
			end,
		},
		moment = { seq = { "dustsheet.4:hit", "dustsheet.1:hit" }, windowF = 180, sameVictim = true }, -- gdd: §7.3.15 Snapshot
		anim = {
			m1 = { windup = "windup_side", strike = "slash_side",
				alt = { "slash_side", "slash_side", "cast_push", "slash_side", "slash_side", "cast_push", "kick", "spin_a", "cast_raise", "slash_down", "spin_a", "spin_b" } },
			q = { windup = "ready", strike = "point", face = { windup = "smile", active = "smile", recovery = "neutral" } },
			["1"] = { windup = "cast_raise", strike = "cast_push" },
			["2"] = { windup = "ready", strike = "duck" },
			["3"] = { windup = "point", strike = "cast_push" },
			["4"] = { windup = "windup_high", strike = "throw", face = { windup = "grit", active = "shout", recovery = "smile" } },
			a1 = { windup = "windup_side", strike = "slash_side" },
			a2 = { windup = "ready", strike = "cast_push" },
			a3 = { windup = "ready", strike = "spin_a" },
			a4 = { windup = "cast_raise", strike = "cast_raise", face = { windup = "awaken", active = "awaken", recovery = "smile" } },
			awaken = { windup = "crouch", strike = "cast_raise", face = { windup = "grit", active = "awaken", recovery = "smile" } }, -- anim.<style>.awaken (72f, §4.6.1)
		},
	}
end
```

Dust rule implemented by `dustsheet.onHitDealt` (`h.moves` is the attacker's pinned kit set `move`, passed in `HookCtx`, §4.3.1):

| Hit (outcome `"hit"`, `firstOfUse`, `damage > 0`) | Dust added |
|---|---|
| `info.kind == "ender"` | 1 |
| Other M1-family kinds (`m1`, `airm1`, `dashm1`) | 0 |
| Zone `tick` hits (`dustsheet.a3`, Getaway rule hits) and zero-damage requests (`dustsheet.q`, the `dustsheet.2` stagger) | 0 |
| Any other move hit | `move.params.dustStacks or 1` (dustsheet.1: 1, dustsheet.3: 1 per use, dustsheet.4: 2, dustsheet.a1: 2, dustsheet.a2: 1) |

#### 4.5.8 Stub styles (12) and the rarity table

Stubs live in `Styles/init.luau`. Their palettes are the designed GDD §7.3 palettes (the comment on each row names the colours and the kit). Across all 18 styles the closest pair of primaries is `angler`–`raincloud` at about 55, so S06's ≥ 40 rule already holds when a stub is implemented.

```lua
local STUBS: { { id: Types.StyleId, rarity: Types.Rarity, archetype: string, p: { number }, s: { number }, a: { number } } } = {
	{ id = "skimstone",   rarity = "common",    archetype = "SkimmingStone",     p = { 84, 120, 160 }, s = { 240, 242, 238 }, a = { 196, 140, 60 } }, -- slate blue, tide-mark white, ammonite ochre (GDD §7.3.3)
	{ id = "masseur",     rarity = "common",    archetype = "BottleMasseur",     p = { 226, 110, 140 }, s = { 248, 236, 220 }, a = { 120, 84, 60 } }, -- rose pink, cream, spa-mud brown (GDD §7.3.4)
	{ id = "bellhop",     rarity = "common",    archetype = "HamsterBellhop",    p = { 30, 110, 70 }, s = { 200, 160, 70 }, a = { 215, 140, 75 } }, -- bottle green, brass, hamster ginger (GDD §7.3.5)
	{ id = "angler",      rarity = "uncommon",  archetype = "HeronAngler",       p = { 128, 132, 124 }, s = { 245, 205, 50 }, a = { 220, 60, 50 } }, -- heron grey, rain-hat yellow, float red (GDD §7.3.6)
	{ id = "kite",        rarity = "uncommon",  archetype = "NervousKite",       p = { 245, 115, 30 }, s = { 40, 70, 190 }, a = { 250, 245, 230 } }, -- kite orange, cobalt, paper white (GDD §7.3.7)
	{ id = "boxcamera",   rarity = "uncommon",  archetype = "BoxCamera",         p = { 190, 130, 70 }, s = { 35, 32, 36 }, a = { 200, 160, 70 } }, -- saddle tan, camera black, brass (GDD §7.3.8)
	{ id = "diver",       rarity = "rare",      archetype = "DolphinDiver",      p = { 40, 200, 215 }, s = { 245, 120, 95 }, a = { 250, 250, 245 } }, -- aqua, coral, white (GDD §7.3.11)
	{ id = "lamplighter", rarity = "epic",      archetype = "PuffinLamplighter", p = { 255, 215, 90 }, s = { 34, 34, 44 }, a = { 250, 250, 245 } }, -- lamp glow, puffin black, lens white (GDD §7.3.13)
	{ id = "cablecar",    rarity = "epic",      archetype = "CuckooClock",       p = { 70, 160, 50 }, s = { 245, 235, 210 }, a = { 200, 40, 40 } }, -- alpine green, cream, gondola red (GDD §7.3.14)
	{ id = "raincloud",   rarity = "legendary", archetype = "StormCloud",        p = { 90, 95, 110 }, s = { 250, 220, 50 }, a = { 110, 160, 220 } }, -- storm grey, lightning yellow, rain blue (GDD §7.3.16)
	{ id = "serpent",     rarity = "mythic",    archetype = "SeaSerpent",        p = { 0, 100, 120 }, s = { 235, 230, 220 }, a = { 120, 200, 170 } }, -- deep teal, pearl, sea-glass green (GDD §7.3.17)
	{ id = "ferry",       rarity = "cosmic",    archetype = "LastFerry",         p = { 245, 245, 240 }, s = { 200, 160, 70 }, a = { 150, 160, 170 } }, -- hull white, brass, fog grey (GDD §7.3.18)
}
-- Each stub: implemented = false, hp = 100, passive = { id = "<id>.passive", params = {} },
-- quirk = K.stubMove("<id>.q"), moves = { K.stubMove("<id>.1") .. "<id>.4" }, awakening = { id = "<id>.awaken",
-- durationSec = 60, hurtboxScale = 1, modifiers = {}, moves = { stubMove a1..a4 } } (a4 behavior "RealmCast", tags {"Realm"}
-- for legendary+), m1 = { anim = "anim.<id>.m1", vfx = "vfx.<id>.m1", sfx = "sfx.<id>.m1" }, kitProps = {}.
-- K.stubMove(id): behavior "Strike", tags {}, startup 1, active 1, recovery 0, damage 0, cooldown 0, parryable false,
-- guardDamage 0, chip 0, hitstun 0, keys derived as K.move.
```

Rarity table (S02; GDD §7.2, §8.3; §0.4). Archetype strings are fixed by the §0.4 roster table; S05 checks their pattern.

| Rarity | Style IDs |
|---|---|
| starter | `attendant`, `lifeguard` |
| common | `skimstone`, `masseur`, `bellhop` |
| uncommon | `angler`, `kite`, `boxcamera` |
| rare | `sandcastle`, `scoop`, `diver` |
| epic | `busker`, `lamplighter`, `cablecar` |
| legendary | `dustsheet`, `raincloud` |
| mythic | `serpent` |
| cosmic | `ferry` |

`busker` is the slice's Epic, so it covers the Epic reveal tests (Part 5, Part 7).

#### 4.5.9 GDD cross-check table (`kits.spec.luau`)

`tests/core/kits.spec.luau` contains the expected values below as a literal table and asserts them against the registry (`kits.<style>.gdd_numbers`). Values in parentheses are `maxDamage`.

| Style | q dmg/CD | 1 | 2 | 3 | 4 ★ | a1 | a2 | a3 | a4 |
|---|---|---|---|---|---|---|---|---|---|
| `attendant` | 3/6 | 8/7 | 11/9 | 6/12 | 14/16 (★4) | 14/8 | 16/12 | 0 (+5)/12 | 26/25 |
| `lifeguard` | 3/6 | 7/6 | 15/16 (★2) | 4/12 | 9/13 | 14/8 | 12/11 | 10/12 | 28/25 |
| `sandcastle` | 0 (+4)/8 | 7/7 | 3 per bonk (18)/15 | 5/12 | 16/18 (★4) | 15/9 | 7 per bonk (14)/11 | 0/14 | 24/25 |
| `busker` | 3/5 | 9/8 | 3 per peck (6)/14 | 10/14 | 12/16 (★4) | 8/12 | 16/9 | 2 per peck (12)/16 | 26/25 |
| `scoop` | 3/6 | 8/6 | 8/14 | 0/9 | 12/20 (★4) | 10/14 | 14/10 | 12/12 | 24/25 |
| `dustsheet` | 0/8 | 8/6 | 0/12 | 3 per bump (12)/16 | 12/20 (★4) | 16/10 | 15/14 | 3 per touch (18)/10 | 0/0 (Getaway) |

**Verify:** Lune `kits.<style>.gdd_numbers` (6 tests), `kits.signature_slot`, `kits.extenders_resolve` (K04 resolves via tag or frame proof for every slice style), `kits.peak_one_modifier` (each slice Peak has exactly one modifier key and `hurtboxScale == 1`), `kits.kitprops_cover_summons` (K23: every Summon move's `params.model` is in `kitProps`).

#### 4.5.10 Skin keys and Base text

Kit data holds no display strings (P4-R7). Part 4 owns the `Base` text of the style and move keys below; Part 8 §8.3.2–§8.3.3 copies it into `src/Shared/Skin/Packs/Base.luau` word for word.

- `style.<id>.name` / `style.<id>.epithet`: the Name and Epithet columns of the §0.4 roster table, for all 18 IDs (e.g. `attendant` → "Sonny" / "Take a Seat").
- `move.<moveId>.name` for the six slice kits. The `m1` names are tuning defaults (the GDD describes each M1 chain by its action); every other name is the GDD §7.3.N kit name.

| Slot | `attendant` | `lifeguard` | `sandcastle` | `busker` | `scoop` | `dustsheet` |
|---|---|---|---|---|---|---|
| m1 | Chair Jabs | Buoy Slaps | Spade Swipes | Wing Slaps | Cone Bops | Sheet Flicks |
| passive | Seen It All | Walk, Don't Run! | Hard Hat | Tip Jar | Toppings | Gathering Dust |
| q | Reserved | Pole Vault | Spirit Level | Pass the Hat | Ding-Ding! | Hands Off! |
| 1 | Hop In | Buoy Toss | Wet Sand | Squeezebox Puff | Scoop Shot | Moth Puff |
| 2 | Ticket Punch | Belly Flop | Sandcastle | Heckler | Brain Freeze | Peekaboo |
| 3 | Chair Toss | Ring Toss | Dig a Moat | Snack Snatch | Melt Away | Left Behind |
| 4 | Take a Seat! | Rescue Hold | Let the Tide In! | Cymbal Crash! | Meltdown | Draped! |
| awaken | Rush Hour | Red Flag Day | Topping Out | One-Man Band | Soft Serve | All Covered |
| a1 | Not on My Beach | No Swimming! | Drawbridge! | Off-Key | Sprinkle Shower | Sheet Flap |
| a2 | Domino Deckchairs | Wave Machine | Grand Castle | Drum Roll | Squirty Cream | Haunt |
| a3 | Tea Break | Bodyboard | Sandbags | Gull Gang | Wafer Volley | Linen Whirl |
| a4 | Musical Chairs! | Whirlpool | Castle Keep | Sing-Along! | Last Call | Greetings from the Closed Season |

**Verify:** `lint_strings` key coverage (L02, Part 8 §8.4.3): every key above exists in `Base` for each implemented style.

---

### 4.6 Placeholder animation spec

#### 4.6.1 Generation algorithm (`BuildPlaceholderAnims`, Part 6 implements; data here)

For every implemented style the builder creates one `KeyframeSequence` under `ReplicatedStorage.Anim` per animation key:
- the 9 move slots (`q`, `1..4`, `a1..a4`); a RealmCast a4 uses key `anim.<style>.realm`;
- the 12 chassis M1/Ender clips of Part 3 §3.11.8 (`anim.<style>.m1_<sub>` for Sub 1, 2, 3, 11, 12, 13, 20; `anim.<style>.ender_<sub>` for Sub 1–5);
- `anim.<style>.awaken` (the Peak activation).

That is 22 per style.

Instance and attributes:
- Name: the key's last segment prefixed by the style, e.g. `attendant_4`, `attendant_m1_11`, `dustsheet_realm`.
- Attributes:
  - `AssetKey` = the key above;
  - `MoveId` (`<style>.m1` for chassis clips);
  - `Frames = S+A+R`;
  - `Gen = "placeholder-v1"`.
- Properties: `Loop = false`, `Priority = Enum.AnimationPriority.Action2` (Part 3 plays combat tracks at `Action2`, Peak activation and RealmCast at `Action4`; §3.11.8). Both are `AnimationClip` members [Docs `AnimationClip.yaml`].

Frame data:
- Moves use the MoveDef. Grabs use their whiff length (`S+A+R`); the hold extension is not animated by the placeholder.
- Chassis clips use Part 3's `Combat.M1` table (Part 3 §3.17):
  - `m1_1..m1_3`: `Hits[1..3]`;
  - `m1_11..m1_13`: `Air`;
  - `m1_20`: `DashM1`;
  - `ender_1`, `ender_2`, `ender_4`, `ender_5`: `Ender`;
  - `ender_3`: `Downslam`.
- `awaken` uses 24/36/12.

Keyframes (time `t(n) = n/60` s; frame numbers are 0-based `e`). Each is a `Keyframe` with `Time` and poses added via `Keyframe:AddPose`; the pose hierarchy follows R15 (`HumanoidRootPart › LowerTorso › UpperTorso › …`, [Docs `Pose.yaml`]):

| # | Frame | Pose | `PoseBase.EasingStyle` |
|---|---|---|---|
| K1 | 0 | `neutral` | `CubicV2` |
| K2 | `floor(S/2)` (skipped if 0) | `windup` (or `mid` when S ≥ 10 and `mid` is set) | `CubicV2` |
| K3 | `S − 1` | `windup` (snappy hold) | `Constant` |
| K4 | `S` | `strike`, or `alt[1]` | `Linear` |
| K4b | each later hit frame `S + hitFrames[i] − 1`, i ≥ 2 | `alt[(i−1) % #alt + 1]`; the last hit uses `finish` when set | `Linear` |
| K5 | `S + A − 1` | the last strike pose (hold) | `Constant` |
| K6 | `S + A + R` | `settle` or `neutral` | `CubicV2` |

`Enum.PoseEasingStyle.Cubic` is deprecated; use `CubicV2` [Docs `enums/PoseEasingStyle.yaml`]. Deduplicate keyframes with equal time (keep the later row). The result has 3–6 key poses for single-hit moves and one extra per hit for barrages.

Markers (`KeyframeMarker` via `Keyframe:AddMarker`; `Name` + `Value`, [Docs `KeyframeMarker.yaml`]):

| Name | Value | On keyframe |
|---|---|---|
| `Windup` | `MoveId` | K1 |
| `Face` | `face.windup` (default `grit`) | K1 |
| `Active` | `MoveId` | K4 |
| `Face` | `face.active` (default `shout`) | K4 |
| `VFX` | `def.vfx` | K4 (and each K4b) |
| `SFX` | `def.sfx` | K4 (and each K4b) |
| `Recovery` | `MoveId` | K5 (time `S + A`; the builder adds a pose-less keyframe if needed) |
| `Face` | `face.recovery` (default `neutral`) | K5 |

The notation "`Face:<expr>`" (agent-assets §7.2) means `Name = "Face"`, `Value = "<expr>"`. Expression names are from the 24-key face set: `neutral, smile, grit, shout, awaken, …` (Part 6). **No runtime code listens to these markers**: `AnimationTrack:GetMarkerReachedSignal` is not Simulation-Access, and VFX/SFX/face timing comes from MoveDef frame data (§4.2.0.7, Part 3 §3.1.5). The markers exist for the §4.6.4 timing-lock check and as the animator's timing sheet.

Studio registration: `Assets.registerStudioClips(ReplicatedStorage.Anim)` (Part 2 §2.9.1) calls `AnimationClipProvider:RegisterAnimationClip` per sequence. `KeyframeSequenceProvider` is deprecated and banned. Online and multi-peer tests need published IDs (§0.5, H-08).

**Worked example** (`busker.4`, S=14, A=2, R=24, total 40 f):
- K1 0 `neutral` (markers Windup, Face=grit);
- K2 7 `guard_wide`;
- K3 13 `guard_wide` (Constant);
- K4 14 `cast_push` (markers Active, Face=shout, VFX=`vfx.busker.4`, SFX=`sfx.busker.4`);
- K5 15 `cast_push` (Constant), plus a marker keyframe at 16 (Recovery, Face=smile);
- K6 40 `neutral`.

Length = 40/60 ≈ 0.667 s.

#### 4.6.2 Pose library (in `BuildPlaceholderAnims.luau`)

Angles are degrees `(X, Y, Z)` applied as `CFrame.Angles(rad X, rad Y, rad Z)` on the joint's Pose. `LT.pos` is a root offset in studs. Sign conventions are (verify in spike S-P4-7): pose each one in Edit mode with `Animator:StepAnimations` (PluginOrOpenCloud capability, agent-assets §5.4) and `screen_capture` it. If an axis reads mirrored, flip the sign in this table, not per move.

Joint keys: `LT` LowerTorso, `UT` UpperTorso, `H` Head, `RUA/RLA` right upper/lower arm, `LUA/LLA` left, `RUL/RLL` right upper/lower leg, `LUL/LLL` left. Unlisted joints are 0.

| Pose | Joints |
|---|---|
| `neutral` | — |
| `ready` | UT(5,0,0) RUA(20,0,−10) LUA(20,0,10) RLA(30,0,0) LLA(30,0,0) |
| `crouch` | LT.pos(0,−0.8,0) UT(20,0,0) RUL(50,0,0) RLL(−70,0,0) LUL(50,0,0) LLL(−70,0,0) |
| `leap` | LT.pos(0,1.2,0) RUL(60,0,0) RLL(−90,0,0) LUL(30,0,0) LLL(−50,0,0) RUA(150,0,−20) LUA(120,0,20) |
| `windup_high` | UT(−15,0,0) H(−5,0,0) RUA(170,0,−15) RLA(20,0,0) LUA(40,0,25) |
| `slash_down` | UT(30,0,0) H(10,0,0) RUA(40,0,10) RLA(5,0,0) LUA(−20,0,15) |
| `windup_side` | UT(0,50,0) RUA(80,40,−60) RLA(30,0,0) LUA(30,0,30) |
| `slash_side` | UT(0,−50,0) RUA(85,−50,−20) LUA(10,0,20) |
| `punch_wind` | UT(0,30,0) RUA(30,0,−25) RLA(100,0,0) LUA(70,0,10) LLA(80,0,0) |
| `punch_r` | UT(0,−25,0) RUA(90,0,0) LUA(40,0,15) LLA(90,0,0) RUL(−10,0,0) LUL(20,0,0) |
| `punch_l` | UT(0,25,0) LUA(90,0,0) RUA(40,0,−15) RLA(90,0,0) LUL(−10,0,0) RUL(20,0,0) |
| `kick_wind` | UT(−10,0,0) RUL(−30,0,0) RLL(−80,0,0) RUA(40,0,−30) LUA(40,0,30) |
| `kick` | LT(−15,0,0) UT(−15,0,0) RUL(85,0,0) LUL(−10,0,0) RUA(−30,0,−40) LUA(−30,0,40) |
| `cast_raise` | UT(−10,0,0) H(−15,0,0) RUA(170,0,−20) LUA(170,0,20) |
| `cast_push` | UT(10,0,0) RUA(90,0,−10) LUA(90,0,10) |
| `point` | UT(0,−15,0) RUA(90,0,0) LUA(10,0,10) |
| `grab_reach` | UT(15,0,0) RUA(90,0,−15) LUA(90,0,15) RLA(10,0,0) LLA(10,0,0) |
| `grab_hold` | UT(5,0,0) RUA(70,0,−35) LUA(70,0,35) RLA(70,0,0) LLA(70,0,0) |
| `throw` | UT(0,−60,0) RUA(110,−30,−30) LUA(30,0,30) RUL(20,0,0) |
| `stomp_up` | UT(−5,0,0) RUL(80,0,0) RLL(−90,0,0) RUA(60,0,−40) LUA(60,0,40) |
| `stomp_down` | LT.pos(0,−0.5,0) UT(25,0,0) RUL(10,0,0) RLL(−10,0,0) RUA(20,0,−60) LUA(20,0,60) |
| `fly` | LT(70,0,0) RUA(−20,0,−60) LUA(−20,0,60) |
| `slide` | LT.pos(0,−0.6,0) LT(−20,0,0) RUL(70,0,0) RLL(−30,0,0) LUL(−20,0,0) RUA(20,0,−70) LUA(20,0,70) |
| `buff` | UT(−10,0,0) H(−10,0,0) RUA(0,0,−100) RLA(120,0,0) LUA(0,0,100) LLA(120,0,0) |
| `guard_wide` | UT(−5,0,0) RUA(90,0,−80) LUA(90,0,80) |
| `spin_a` | LT(0,120,0) RUA(0,0,−90) LUA(0,0,90) |
| `spin_b` | LT(0,240,0) RUA(0,0,−90) LUA(0,0,90) |
| `sit` | LT.pos(0,−1.2,0) UT(−10,0,0) RUL(90,0,0) RLL(−90,0,0) LUL(90,0,0) LLL(−90,0,0) RUA(20,0,−30) LUA(20,0,30) |
| `carry` | UT(10,0,0) RUA(170,0,−30) RLA(70,0,0) LUA(40,0,20) LLA(60,0,0) |
| `dig` | UT(40,0,0) H(15,0,0) RUA(70,0,−10) RLA(20,0,0) LUA(80,0,10) LLA(40,0,0) RUL(30,0,0) RLL(−30,0,0) |
| `squeeze` | UT(5,0,0) RUA(70,0,−60) RLA(80,0,0) LUA(70,0,60) LLA(80,0,0) |
| `duck` | LT.pos(0,−1.4,0) UT(45,0,0) H(20,0,0) RUL(80,0,0) RLL(−110,0,0) LUL(80,0,0) LLL(−110,0,0) RUA(160,0,−10) LUA(160,0,10) |
| `flop` | LT.pos(0,−1.6,0) LT(80,0,0) RUA(0,0,−110) LUA(0,0,110) RUL(−10,0,0) LUL(−10,0,0) |

The last six poses (`sit`, `carry`, `dig`, `squeeze`, `duck`, `flop`) serve the slice kits' own verbs: sitting on the chair, the shoulder carry, digging, squeezing the accordion, ducking under a sheet and a belly flop. Their angles are tuning defaults like the rest.

#### 4.6.3 M1 per style

`anim.m1.alt` lists the 12 strike poses in Part 3's clip order:

`m1_1, m1_2, m1_3, m1_11, m1_12, m1_13, m1_20, ender_1 (Normal), ender_2 (Uppercut), ender_3 (Downslam), ender_4 (Side-L), ender_5 (Side-R)`

The windup is `anim.m1.windup`. Faces are the defaults. The `awaken` clip uses synthetic frame data S = 24, A = 36, R = 12 (td), totalling `Combat.Awakening.ActivationFrames` (72, GDD §6.9).

#### 4.6.4 Timing-lock rule

A final (artist) animation for key `anim.<style>.<slot>` MUST:
1. have length within ±1 frame of the placeholder: `|length − Frames/60| ≤ 1/60`;
2. contain the same multiset of marker names (`Windup`, `Active`, `Recovery`, `Face` ×3, `VFX` ×hits, `SFX` ×hits);
3. place `Windup`, `Active` and `Recovery` within ±1 frame of the placeholder times.

`Face` values may differ (art choice). Gameplay never depends on clips (Part 3 §3.1.5). The lock keeps presentation aligned with frame data (GDD §13.2: "timing is locked").

Check: Studio scenario `anim.timing_lock` (server, Studio only). For every manifest key with `kind = "anim"` and `status = "final"`:
1. `AnimationClipProvider:GetAnimationClipAsync(id)` (yields, security None [Docs `AnimationClipProvider.yaml`]), wrapped in `pcall`;
2. read `AnimationClip.Length`, polling until non-zero, timeout 5 s [Docs];
3. collect `(name, time)` markers by the clip's class, then compare against the placeholder spec recomputed from the MoveDef:
   - `clip:IsA("KeyframeSequence")`: `KeyframeSequence:GetKeyframes()`, then `Keyframe:GetMarkers()` on each; name = `KeyframeMarker.Name`, time = the parent `Keyframe.Time` [Docs `KeyframeSequence.yaml`, `Keyframe.yaml`, `KeyframeMarker.yaml`];
   - `clip:IsA("CurveAnimation")` (common for final artist clips; it has no keyframes): every descendant `MarkerCurve`; name = the `MarkerCurve`'s `Name`, times from `MarkerCurve:GetMarkers()`, which returns the time and string value of each marker [Docs `CurveAnimation.yaml`, `MarkerCurve.yaml`]. The record field names and the one-curve-per-marker-name layout are checked on the first final clip (verify in spike);
   - any other class: `[HB][TEST] anim.timing_lock FAIL key=<key> unsupported clip class <ClassName>`.

`anim.placeholders_built` asserts that every MoveDef and M1 variant key resolves to a sequence whose `Frames` attribute equals S+A+R. It also asserts that marker counts match.

`tools/export_framedata.luau` (Lune; Part 3/4) also writes `docs/reference/framedata.json` with, per move:
- `{moveId, S, A, R, total, markers: [{name, frame}], onHitAdv, onBlockAdv}`;
- `onHitAdv = hitstun − (A − 1) − R` and `onBlockAdv = blockstun − (A − 1) − R` (GDD Appendix B).

Animators use it as the timing sheet.

**Verify:** Lune `kits.anim_specs_complete` (K24, including the six new poses); scenarios `anim.placeholders_built` and `anim.timing_lock`; MCP `search_game_tree` under `ReplicatedStorage.Anim` lists 6 × 22 = 132 kit sequences (Part 6 adds the `anim.common.*` clips).

---

### 4.7 Presentation mapping

#### 4.7.1 Palettes

The palette feeds the always-on Style Tell: nameplate chip = `primary`; VFX tint core = `accent`, mid = `primary`, ink rim = the Theme `Ink` token (`#241C33`, Part 6 §6.9.4; GDD §13.4 three tones plus ink). Graybox Full Uniform bodies use primary (torso) and secondary (limbs and head); props take the colour roles of §4.7.4. The slice palettes are saturated primaries so the Style Tell reads on cream, chalk and sand maps.

| Style | Primary | Secondary | Accent | Shape language (GDD §13.4) |
|---|---|---|---|---|
| `attendant` | (214, 58, 48) stripe red | (245, 236, 214) cream | (110, 70, 45) chocolate | canvas stripes, snap lines |
| `lifeguard` | (0, 150, 150) sea-teal | (230, 60, 50) lifeguard red | (245, 245, 240) white | rings, splash crowns |
| `sandcastle` | (40, 60, 120) navy | (222, 196, 140) sand | (200, 40, 40) flag red | crenellations, sand puffs |
| `busker` | (222, 170, 40) mustard | (150, 156, 165) gull grey | (40, 140, 140) teal | coins, sound rings |
| `scoop` | (200, 40, 110) raspberry | (160, 225, 195) mint | (250, 240, 215) cream | drips, sprinkles |
| `dustsheet` | (150, 120, 200) lavender | (235, 228, 210) linen | (230, 160, 50) amber | folds, moth wings |
| stubs | §4.5.8 | | | — |

S06 holds for the slice set. The closest pair is `attendant` vs `scoop` primaries, distance ≈ 66 (S06 needs ≥ 40).

#### 4.7.2 VFX recipes per move

Templates are built by Part 6's `BuildVfxTemplates` under `ReplicatedStorage.VFX` (agent-assets §7.4 names; template = recipe name). Manifest entries:

`vfx.<style>.<slot> = { id = "tpl:VFX/<Recipe>", status = "placeholder", source = "agent", kind = "vfx", note = "<what>" }`

Generic effects are added by Part 6 automatically, so they are not per move:
- `VFX_HitSpark_Light` on every hit;
- `VFX_Impact_Heavy` (with impact frame) on signature and ragdoll hits;
- `VFX_Dash_Puff` on Rush travel and Mobility;
- telegraphs `VFX_Telegraph_*`.

**New recipes** requested from Part 6 (same rules as §7.4: `LightInfluence = 0`, no collision/query, template under `ReplicatedStorage.VFX`):

| Recipe | Spec |
|---|---|
| `VFX_Telegraph_Arc` | Flat Neon wedge-ring segment (CSG: cylinder `SubtractAsync` cylinder, then `IntersectAsync` with a wedge), 0.05 thick, telegraph colour, transparency 0.35, outline copy 0.4 studs larger in the outline colour Part 6 picks per telegraph (Theme `Ink` or white, for 4.5:1 contrast; Part 6 §6.6.3) |
| `VFX_Projectile_Orb` | Neon `Ball` 1.2 studs + `Trail` (no texture, Lifetime 0.15, WidthScale 1→0) between two attachments ±0.5 studs |
| `VFX_Zone_Disc` | Flat Neon cylinder (height 0.1) at transparency 0.6 + ring edge (cylinder subtract) at 0.2; sized by carrier `r`; for `line` zones a flat Neon block of `width × radius`; for `ring` zones the edge only, with the inner disc cut out |
| `VFX_Wave_Wall` | Neon block sized from carrier `w`/`h`, transparency 0.4, front face brighter (second block 0.2 thick) |
| `VFX_Stealth_Shimmer` | `ParticleEmitter` default sparkles texture, Rate 6, Lifetime 0.4, Size 0.6, LockedToPart (also used by Part 6's spawn shimmer; the stealth status render is deferred) |
| `VFX_Status_Pips` | `BillboardGui` + `TextLabel` with `●` × n in the status colour, `LuckiestGuy` font, offset +3.5 studs (Dust stacks; root/slow/silence glyphs `⌂ ≋ ⊘`) |
| `VFX_Getaway_Quarter` | Flat quarter-disc, radius 40 (cylinder `IntersectAsync` with a box covering one quadrant), height 0.1, the Theme `Ink` colour (`#241C33`, never black) at transparency 0.35 for the dark quarter; the flicker plays transparency 0.6 ↔ 0.9 at 6 Hz (static 0.75 with Reduced Motion). Placed per quarter on the Getaway centre (§4.4.3) |
| `VFX_Encase_Block` | **Deferred** with the `freeze` stun flavour: a block 6×7×6 at transparency 0.35 |

Per-move map (Recipe; tint by palette). The `m1` column is the swing trail.

| Style | m1 | q | 1 | 2 | 3 | 4 | a1 | a2 | a3 | a4 |
|---|---|---|---|---|---|---|---|---|---|---|
| `attendant` | Slash_Smear | ImpactStar | ImpactStar | ImpactStar | Projectile_Orb | Impact_Heavy | Guard_Bubble | Wave_Wall | ImpactStar | Zone_Disc |
| `lifeguard` | Shockwave_Ring | Shockwave_Ring | Zone_Disc | Shockwave_Ring | Projectile_Orb | ImpactStar | Zone_Disc | Wave_Wall | Slash_Smear | Zone_Disc |
| `sandcastle` | Slash_Smear | ImpactStar | Projectile_Orb | ImpactStar | Zone_Disc | Wave_Wall | Shockwave_Ring | ImpactStar | Guard_Bubble | Zone_Disc |
| `busker` | Slash_Smear | Projectile_Orb | Projectile_Orb | ImpactStar | ImpactStar | Shockwave_Ring | Wave_Wall | ImpactStar | ImpactStar | Zone_Disc |
| `scoop` | ImpactStar | Shockwave_Ring | Projectile_Orb | ImpactStar | Slash_Smear | Zone_Disc | Projectile_Orb | Beam_Magic | Projectile_Orb | Shockwave_Ring |
| `dustsheet` | Slash_Smear | Beam_Magic | Projectile_Orb | Guard_Bubble | ImpactStar | Zone_Disc | Beam_Magic | Slash_Smear | Zone_Disc | Getaway_Frame |

Extra keys:
- `vfx.status.root`, `.slow`, `.silence`, `.dust` → `VFX_Status_Pips`;
- `vfx.status.ward` → `VFX_Guard_Bubble` (Do Not Disturb);
- `vfx.dustsheet.getaway_dark` → `VFX_Getaway_Quarter` (dark and flickering quarters, §4.4.3);
- deferred with their first user: `vfx.status.freeze` → `VFX_Encase_Block`; `vfx.status.stealth` → `VFX_Stealth_Shimmer`.

Carriers use their move key as `vis` (projectile orb, zone disc, wave wall) or a `model.*` key (summons, §4.7.5).

Per-move emitter caps by rarity (GDD §13.5: Starter 3, Rare 5, Epic 6, Legendary 7) are enforced by Part 6. Every recipe above uses ≤ 2 emitters.

#### 4.7.3 SFX per move (built-in sounds, placeholder)

Manifest: `sfx.<style>.<slot> = { id = "rbxasset://sounds/<file>", status = "placeholder", source = "builtin", kind = "audio" }`. The 11 built-in files are listed in agent-assets §2.7 / §0.5. The playback speed lives in `Styles.SFX_SPEED[key]` (default 1), which SfxController (Part 6) applies, so moves sharing a file still differ. Part 7's release check (`check_manifest --release`, rule M08) blocks these placeholder sounds in releases. The final sounds follow the kit sound notes (GDD §7.3) and the audio direction (GDD §14): Sonny's *clack!*, Paddles' whistle and *sploosh*, Burrows' *shink*, Bellows' wheezy sting, Dollop's kiosk chime, Mothball's soft "shhh" and moth flutter. That is art work, not graybox.

Abbreviations: `jump` = action_jump.mp3, `land` = action_jump_land.mp3, `swim` = action_swim.mp3, `fall` = action_falling.ogg, `rise` = action_get_up.mp3, `steps` = action_footsteps_plastic.mp3, `boom` = impact_explosion_03.mp3, `splat` = impact_water.mp3, `tick` = volume_slider.ogg.

| Style | m1 | q | 1 | 2 | 3 | 4 | a1 | a2 | a3 | a4 |
|---|---|---|---|---|---|---|---|---|---|---|
| `attendant` | land@1.6 | tick@0.7 | jump@1.0 | tick@1.6 | swim@1.4 | boom@1.3 | tick@0.5 | land@0.8 | splat@1.8 | boom@0.8 |
| `lifeguard` | splat@1.7 | jump@0.8 | splat@1.2 | splat@0.7 | swim@1.7 | rise@0.9 | tick@0.9 | swim@0.6 | steps@1.6 | swim@0.5 |
| `sandcastle` | steps@1.4 | tick@1.4 | splat@1.5 | land@1.2 | steps@0.8 | swim@0.8 | boom@1.5 | land@0.9 | land@1.4 | boom@0.6 |
| `busker` | jump@1.4 | fall@1.8 | rise@1.6 | jump@1.9 | rise@1.3 | boom@1.8 | rise@0.6 | steps@1.9 | jump@1.6 | tick@0.6 |
| `scoop` | tick@1.2 | tick@1.9 | splat@1.3 | splat@0.9 | swim@1.0 | splat@0.6 | tick@1.8 | splat@1.1 | fall@1.2 | steps@0.6 |
| `dustsheet` | fall@0.7 | rise@0.7 | fall@1.4 | fall@0.5 | steps@0.7 | fall@0.6 | rise@0.5 | fall@0.9 | fall@1.5 | boom@0.5 |

Extra key: `sfx.dustsheet.getaway_bell` = tick@2.0 (the Quiet Hours bell, 30 f ahead of each pulse, §4.4.3).

`SFX_SPEED` holds every `@` value above, keyed by `sfx.<style>.<slot>` (and `sfx.dustsheet.getaway_bell`). `oof.ogg` and `ouch.ogg` are voices and are not used for moves.

#### 4.7.4 Charm Gear and Peak-piece recipes (`ReplicatedStorage.Looks`, built by Part 6 `BuildTemplates`)

**Charm Gear** is one oversized held or back prop that defines the silhouette, plus a head piece (§0.4; §0.12). Each key `model.<style>.gear_<name>` maps to `tpl:Looks/Gear_<style>_<name>`. The first `gear_` entry in `kitProps` is the prop, the second the head piece (K23). Each slice style also has one optional **Peak piece** (`model.<style>.peak_<name>` → `tpl:Looks/Peak_<style>_<name>`, table below), worn only at Peak.

Build rules (Charm Gear and Peak pieces alike):
- **Container.** Each template is an `Accessory` [Docs `Accessory.yaml`] whose `Handle` (a Part or union) contains an `Attachment` named for the body attachment it mates with. Names from the rigid-accessory spec [Docs `avatar/rigid-accessories/specifications.md`]: `HatAttachment`, `HairAttachment`, `FaceFrontAttachment`, `BodyBackAttachment`, `BodyFrontAttachment`, `WaistBackAttachment`; held props use `RightGripAttachment` (verify in spike S-P4-5; fallback: `RigidConstraint` to `RightHand`).
- **Unions.** `GeometryService:UnionAsync(part, parts, {SplitApart = false})` (yields; security None; returns an array [Docs `GeometryService.yaml`]) runs at **Edit time** inside the builder. Never at runtime.
- **Flags.** All parts: `CanCollide = false`, `CanQuery = false`, `CanTouch = false`, `Massless = true`, `CastShadow = false`, Material `SmoothPlastic`.
- **Colour roles.** Every part is a primitive `{shape, size, offset, role}` with `role` ∈ `primary`, `secondary`, `accent`, `ink` (the form Part 6 §6.8.6 stores). `primary`, `secondary` and `accent` take the style palette through `Theme.stylePalette` (so an Off-Duty Look recolours the gear); `ink` takes the Theme `Ink` token (`#241C33`, Part 6 §6.9.4). There are no fixed colours. At Peak, every Charm Gear and Peak-piece part with role `accent` takes `Theme.Gilt` (the gilt trim, Part 6 §6.8.2). Summon models (§4.7.5) use the same roles.
- **Attaching.** Part 6's AppearanceController attaches gear client-side: it welds each Handle so that its named attachment matches the same-named attachment on the proxy or real part (Part 6 §6.8.3). The look model is a Humanoid-less `LookRig_R15` proxy, so `Humanoid:AddAccessory` is not used. It attaches only the `gear_` entries of `kitProps` as Charm Gear; `summon_` entries are summon models (§4.7.5). When Charm Gear shows is Part 6's rule (§6.8.1; §0.4: the `ShowStyleGear` setting and the Full Uniform toggle). Peak pieces are attached the same way, only while the owner is at Peak (Part 6 §6.8.2).

| Key | Attach | Parts (shape, size in studs, colour role, offset from Handle) |
|---|---|---|
| `model.attendant.gear_chair` | BodyBackAttachment | The folded striped deckchair: frame Blocks (0.15, 3.0, 0.15) ×2 at (±0.8, 0, 0) accent + crossbar Cylinder (0.12 × 1.6) at (0, 1.4, 0) accent + canvas Blocks (0.5, 2.6, 0.08) ×3 at x = −0.5, 0, 0.5 alternating primary/secondary/primary; tilted 10° about X (leaning on the back) |
| `model.attendant.gear_boater` | HatAttachment | Straw boater: brim Cylinder (2.0 dia × 0.08) secondary + crown Cylinder (1.25 dia × 0.5) at (0, 0.28, 0) secondary + band Cylinder (1.27 dia × 0.14) at (0, 0.12, 0) primary |
| `model.lifeguard.gear_buoy` | BodyBackAttachment | Big ring buoy: Cylinder (2.8 dia × 0.6) minus Cylinder (1.4 dia) via `SubtractAsync`, accent (white), with 4 Block bands (0.7, 0.62, 0.5) at 90° steps, secondary (lifeguard red); axis along Z (flat on the back) |
| `model.lifeguard.gear_visor` | HatAttachment | Visor: band Cylinder (1.35 dia × 0.15) minus Cylinder (1.2 dia) secondary + peak Wedge (1.2, 0.1, 0.7) at (0, 0, −0.8) secondary. The red flag is the Peak piece (below); the float belt (at the waist, never at the neck) belongs to the Full Uniform, not to Charm Gear |
| `model.sandcastle.gear_spade` | RightGripAttachment | Long spade: shaft Cylinder (0.18 × 4.0) secondary + blade Block (0.9, 1.1, 0.1) at (0, −2.4, 0) primary + T-grip Block (0.6, 0.15, 0.15) at (0, 2.0, 0) secondary |
| `model.sandcastle.gear_hardhat` | HatAttachment | Hard hat: Ball (1.45, 0.9, 1.55) secondary (sand) cut in half with `SubtractAsync` (flat base) + brim Cylinder (1.75 dia × 0.08) secondary + pennant pole Cylinder (0.06 × 1.2) at (0.5, 0.9, 0) ink + Wedge flag (0.05, 0.4, 0.6) accent (flag red). Never a bucket on the head |
| `model.busker.gear_accordion` | BodyFrontAttachment (verify in spike S-P4-5) | Accordion on the chest: end Blocks (0.4, 1.6, 1.2) ×2 at (±0.9, 0, 0) primary + bellows Blocks (0.2, 1.5, 1.1) ×5 between them, alternating secondary/accent + keyboard Block (0.1, 1.4, 0.3) secondary on the right end |
| `model.busker.gear_cap` | HatAttachment | Feathered flat cap: Cylinder (1.3 dia × 0.3) secondary + peak Wedge (1.0, 0.12, 0.5) at (0, −0.05, −0.7) secondary + feather Wedge (0.05, 0.3, 0.9) at (0.5, 0.2, 0.2) accent |
| `model.scoop.gear_cone` | RightGripAttachment | Giant cone held up: cone body Wedges (0.9, 2.4, 0.45) ×2 back to back, ink (a dark wafer) + scoop Ball (1.4) at (0, 1.6, 0) primary + scoop Ball (1.2) at (0, 2.5, 0) secondary + drip Ball (0.4) at (0.5, 1.2, 0) primary |
| `model.scoop.gear_cap` | HatAttachment | Paper cap: Wedges (1.4, 0.6, 0.4) ×2 back to back forming a peak, accent (cream) + band Block (1.42, 0.15, 0.82) at (0, −0.2, 0) primary |
| `model.dustsheet.gear_sheet` | HatAttachment | Hooded dust sheet: hood Ball (1.7, 1.6, 1.7) secondary minus a front Block (`SubtractAsync`, so the avatar's face shows) + drape Block (2.0, 3.0, 0.12) at (0, −1.9, 0.8) hanging down the back, secondary (linen) + two moth Wedges (0.05, 0.25, 0.3) on the hood, primary |
| `model.dustsheet.gear_tag` | RightGripAttachment | HANDS OFF tag: card Block (0.9, 0.5, 0.05) secondary with a stripe Block (0.8, 0.12, 0.06) accent + string Cylinder (0.03 × 0.8) at (0, 0.6, 0) ink. No text in graybox |

GDD §12.3 rule: every style reads from its oversized prop in pure black silhouette at **48 px**. Scenario `look.gear_silhouette` (owned here; it uses Part 6's `silhouette = true` template build, Part 6 §6.8.6) takes one `screen_capture` per implemented style (at most 3 per run) for the T8 blind art comparison. The agent appends one `pending` row per still to `docs/agent/RECOGNIZABILITY.md` (Part 8 §8.5.3) and never fills in a result; humans run the test and sign it.

**Peak uniform and trim.** The Peak shows the style's uniform for 60 s (render only, Part 6 §6.8.2; §4.4.2 lists each look): the Full Uniform for every viewer except `Simplified`, the style's Peak piece below, and the gilt trim on every `accent` part of the Charm Gear and the Peak piece. Nothing grows and the hurtbox never changes (K10). Part 6 attaches every `Looks.Peak_<style>_*` template it finds for the style, so a style without a Peak piece shows the uniform and the gilt trim only.

**Peak pieces** (optional, one per slice style; same build rules; not listed in `kitProps`, so K23 is unchanged). Each is a graybox stand-in for the GDD Peak look (GDD §6.9, §12.3); the final uniform pieces are art. Peak pieces sit on the look rig like Charm Gear: client-local, massless, never part of the hurtbox.

| Key | Attach | Parts (shape, size in studs, colour role, offset from Handle) |
|---|---|---|
| `model.attendant.peak_blazer` | BodyFrontAttachment | Striped blazer front and megaphone: lapel Blocks (0.7, 1.8, 0.1) ×2 at (±0.45, 0, 0) primary, each with two stripe Blocks (0.12, 1.8, 0.11) secondary + megaphone on a strap: strap Block (0.1, 1.2, 0.05) at (0.9, −0.3, 0) ink, mouthpiece Cylinder (0.4 dia × 0.6) at (0.9, −1.0, −0.3) accent, bell Cylinder (1.0 dia × 0.15) at (0.9, −1.0, −0.7) accent |
| `model.lifeguard.peak_flag` | BodyBackAttachment | Red flag on a pole, standing up beside the buoy: pole Cylinder (0.12 × 5.0) at (0.9, 1.8, 0.4) ink + flag Block (0.05, 1.0, 1.4) at (0.9, 3.8, 1.1) secondary (lifeguard red) + finial Ball (0.3) at (0.9, 4.4, 0.4) accent |
| `model.sandcastle.peak_lamp` | HatAttachment | Hard-hat lamp switched on, on the front of the hard hat: housing Cylinder (0.45 dia × 0.3) at (0, 0.45, −0.75) ink, facing forward + lens Ball (0.4) at (0, 0.45, −0.92) accent (gilt at Peak: the lamp is lit) |
| `model.busker.peak_drum` | BodyBackAttachment | Bass drum on his back, cymbals at his knees: shell Cylinder (2.2 dia × 1.0) at (0, −0.2, 0.6) primary, heads facing left and right + hoop Cylinders (2.3 dia × 0.12) ×2 at the shell's ends accent + cymbal Cylinders (0.9 dia × 0.05) ×2 at (±0.6, −2.6, −0.9) accent, each on a rod Cylinder (0.06 × 2.4) ink running down from the drum |
| `model.scoop.peak_kiosk` | BodyBackAttachment | Kiosk backpack with its shutter up and bunting out: box Block (1.8, 1.6, 0.9) at (0, 0, 0.5) secondary (mint) + raised shutter Block (1.9, 0.9, 0.08) at (0, 1.3, 0.1) primary + bunting string Block (2.4, 0.04, 0.04) at (0, 1.9, 0.5) ink with five flag Wedges (0.05, 0.35, 0.35) hanging from it, alternating accent/primary |
| `model.dustsheet.peak_lanterns` | HatAttachment | Moth lanterns glowing on the sheet: lantern Balls (0.45) ×3 at (−0.6, −1.4, 0.95), (0, −2.0, 0.95), (0.6, −1.4, 0.95) accent, each hung from a string Cylinder (0.03 × 0.5) ink, with a moth Wedge (0.05, 0.2, 0.25) primary on the middle lantern |

**Verify:** Part 6 `BuildTemplates.verify()` (every gear, Peak and summon key resolves through `Assets.template`; every `Gear_*` and `Peak_*` template has its mount attachment); scenario `look.peak_uniform` (Part 6: at Peak the owner renders the uniform, the gilt trim on `accent` parts and any `Peak_<style>_*` piece; root and hurtbox sizes never change); scenario `look.gear_silhouette` (above).

#### 4.7.5 Summon models (`ReplicatedStorage.Looks`)

Visual only. The server carrier is invisible (Part 3). The client attaches the template to the carrier by `vis`. Each key is also listed in the style's `kitProps` (K23). Parts use the colour roles of §4.7.4 (`primary`, `secondary`, `accent`, `ink`); the gilt trim never applies to summons.

| Key | Recipe |
|---|---|
| `model.attendant.summon_chair` | An open striped deckchair, 2.4 × 2.6 × 3.2 studs: frame Blocks (0.15, 0.15, 3.0) and legs, accent + canvas seat Blocks (0.7, 0.08, 2.4) ×3 alternating primary/secondary, sloped 30° |
| `model.sandcastle.summon_castle` | Keep Block (3, 3, 3) secondary + corner Cylinder towers (1 × 4) ×4 secondary + Wedge crenellations on top secondary + flag-spade: Cylinder pole (0.12 × 2) ink with Block blade (0.6, 0.7, 0.08) primary and a Wedge pennant accent; about 5 studs tall |
| `model.sandcastle.summon_grandcastle` | The castle at 1.5× with a second tier Block (3, 2, 3) secondary on top and two pennants accent; about 7.5 studs tall |
| `model.busker.summon_heckler` | A gull: body Ball (1.2, 1.0, 1.6) secondary (gull grey) + head Ball (0.8) secondary + beak Wedge primary (mustard) + wing Wedges ×2 ink + a small flat cap Cylinder primary; always visible |
| `model.busker.summon_gull` | A smaller gull (0.8×), no cap; neck scarf Block accent |
| `model.scoop.summon_van` | Ice-cream van: body Block (4, 3.5, 7) secondary (mint) + cab Block (4, 2.5, 2) at the front secondary + wheel Cylinders ×4 ink + stripe Block (4.02, 0.5, 7.02) primary + a roof cone (Ball scoop on a Wedge) primary; fits the 6-stud line |
| `model.dustsheet.summon_luggage` | Unclaimed luggage: case Block (1.8, 1.3, 0.7) accent (amber) + handle Block ink + label Block (0.4, 0.25, 0.05) secondary on a string ink. Both bodies use this model |

#### 4.7.6 Status visuals

RenderController (Part 6) reads `Buffs`/`Debuffs` (decoded with `Moves.statusDecode`) and `Hitstun` Sub:
- **Debuffs.** `rt`, `sl`, `si` and `du` show `VFX_Status_Pips` above the head; `du` shows n pips (Dust).
- **Flinch and stumble** (`Hitstun` Sub 6 and 4) use the normal reaction poses; no extra effect.
- **Deferred.** The `freeze` flavour (`Hitstun` Sub 2, `VFX_Encase_Block`) and the stealth status `iv` (hidden look and nameplate beyond 12 studs for enemies, `VFX_Stealth_Shimmer`; §4.11 Q9) have no M1-G user. Their render and the `look.stealth_ranges` scenario land with the first style that needs them.
- **Getaway tells.** The Quiet Hours bell, the dark quarter and the flicker of the next quarter for `dustsheet.a4` are drawn from `params.getaway` and the Getaway marker's `OpenF` (§4.4.3). Telegraphs and hit flashes are never hidden.

---

### 4.8 Adding a new style (checklist the agent follows)

1. **Design input.** Copy the style's GDD §7.3 table into the task card. Confirm its rarity, archetype string (§0.4) and Snapshot. Never invent damage or cooldown numbers.
2. **Behaviors.** Map each move to an existing behavior and params (§4.2). If a mechanic fits no behavior:
   - prefer new params on the closest behavior;
   - if the style needs a deferred feature (§4.9 item 26) or a future engine item (§4.9 item 25), build it first as its own task with a deterministic rule and a Lune test;
   - add a new `Moves.STATUS` kind only through §4.3.2 (2-char kind, `mods`, flags) plus a Lune test;
   - a new behavior module would change §0.3, so open a spine issue instead.
3. **Kit file.** Write `src/Shared/Styles/<id>.luau` following §4.5:
   - every number has a `GDD §` or `(td)` comment and every move line a `-- gdd: §7.3.N <slot>` pointer;
   - set `implemented = true` and remove the stub row from `STUBS` in `Styles/init.luau`;
   - add the id to `Types.IMPLEMENTED_M1G` (test aid).
4. **Passive.** Use data (Part 3 chassis keys, `lowHpThreshold`/`lowHp`) where possible; write hooks only for the rest (§4.3). Add the Snapshot (`moment` or a hook).
5. **Peak.** One or more recognized modifiers (§4.3.3), `hurtboxScale = 1`.
6. **Getaway (legendary+).** Add `Realm.rules["<id>.a4"]` in `Core/Realm.luau` (§4.4.3 pattern) plus its cases in Lune `kits.getaway_rules` (and `kits.getaway_schedule_matches` when the a4 MoveDef mirrors the timing for render). Timing comes from `f − r.openF` only; position tests use `m.pos` (E1).
7. **Statuses.** Add any new kinds to `Moves.STATUS` with tests in `kit_status.spec.luau`.
8. **Animation spec.** Fill `anim` for `m1, q, 1–4, a1–a4, awaken` using pose-library names (add poses to §4.6.2 if needed). Run `BuildPlaceholderAnims`, then `Verify`.
9. **Presentation.**
   - palette (S06 distance ≥ 40 against implemented styles);
   - VFX recipe map row (§4.7.2);
   - SFX row and `SFX_SPEED` entries (§4.7.3);
   - Charm Gear recipes (one oversized prop and one head piece, §4.7.4) and summon models (§4.7.5), all listed in `kitProps`; optionally one Peak piece (§4.7.4, not in `kitProps`); every part with a colour role;
   - manifest entries for every `anim./vfx./sfx./model.` key (Part 7 `tools/check_manifest.luau`, including Part 8's rules M09–M13, must pass).
10. **Skin keys.** `style.<id>.name`, `style.<id>.epithet`, `move.<id>.{m1,passive,q,1..4,awaken,a1..a4}.name` go into the `Base` pack (`src/Shared/Skin/Packs/Base.luau`, Part 8 §8.3) in the same task, written from the kit's GDD §7.3.N table as in §4.5.10 (the `m1` name is a tuning default). `lint_strings` and `lint_retired_keys` must be clean.
11. **Tests.**
    - `lune run tests/run.luau`: `styles.*`, `kits.<id>.gdd_numbers` (add the row to §4.5.9's table), `behaviors.*`, `kit_status.*`;
    - scenario `kit.<id>.smoke`: each slot used on a training dummy. Before each slot, clear carried-over flat adds: `CombatService._queuePatch(dummy, {Debuffs = ""})` and `CombatService._queuePatch(attacker, {Buffs = "", Kit = 0})` (Studio seams; or use a fresh dummy slot). Otherwise Dust from dustsheet.1 adds +1 to the next dustsheet slot, and the `nh` from attendant.a3 or sandcastle.q adds +5 or +4 to the next hit (§4.3.2). Expected HP delta = `maxDamage` for single-target hits (± 0.1, chip excluded), or `maxDamage` + the active flat adds clamped to the band cap when the scenario deliberately chains a slot; cooldown attribute = `f + toFrames(cooldown)`; no `[HB][Style] ERROR`;
    - `kit.awaken.swap`, `anim.placeholders_built`.
12. **Docs.**
    - `lune run tools/export_framedata.luau` (regenerates `docs/reference/framedata.json`);
    - update the `ARCHITECTURE.md` rows for the kit file (status `wip` → `done`);
    - append evidence to `docs/agent/PROGRESS.md`.
13. **Human checkpoints.** H-10 feel playtest of the kit. H-08 publish placeholder animations before multi-peer tests. Before any public footage of the style, the producer runs the T11 gameplay-clip test (Part 8; the agent only prepares the captures). A later change to what a move does or shows, a palette, or a Charm Gear, Peak-piece or summon recipe is a drift change: the same task appends `pending` T8 rows (and a T11 row for the style) to `docs/agent/RECOGNIZABILITY.md` (Part 8 §8.5.6).

---

### 4.9 New contracts introduced

1. **Lint scope.** `tools/lint_rules.luau` rule `pure` also covers `src/Shared/Styles/**` and `src/Shared/Behaviors/**` (P4-R1). Part 3's `standards.sim_banned_tokens` already covers `Behaviors/**`.
2. **Style module shape.** `Styles/<id>.luau` returns `(K: KitBuilder) -> StyleModule {def, hooks, moment, anim}`. The Styles API is in §4.1.2: `get, tryGet, all, implemented, isPlayable, move, hooks, moment, anim, behavior, variants, forConfig, validate, maxDamage, passiveStep, serverHook, SFX_SPEED`. Part 1's skill `hullabrawl-style-kit.md` must say "returns a builder function producing a StyleModule" instead of "returning a StyleDef".
3. **Config.** New top-level `Kits` section (§4.1.6) with ConfigService key `HB_Kits`, pinned with `CombatCfg` (Part 2 §2.6.6).
   - `Combat.Realm` keys read here: `CastFrames` (90), `RadiusStuds` (40), as Part 3 §3.17 names them.
   - `Combat.Awakening.DurationSec` (60) is read by K10.
4. **`BehaviorModule` extensions** (Part 3 §3.4.2 / §3.18 item 10): optional `name`, `validate(def, cfg)` (`cfg` = full `Types.Config`; boot validator), `maxDamage`, `tellFrames`, `blockable`, `intercept`, `hurtCarrier`, `reflectCarrier` (deferred, item 26), plus Part 3's `stepCarrier(view, f)` on `Projectile`, `Zone` **and `Summon`** (§4.2.0.1), and the pure helper `Zone.pullVelocity(def, carrierPos, rootPos)` (§4.2.6).
   - Part 3 calls `intercept` before `Damage.resolve` when the victim is in `Move` and its behavior defines it.
   - Part 3 calls `hurtCarrier` when a carrier hurtbox is hit, and `reflectCarrier` on a `reflect` verdict (unreachable in M1-G).
   - `Styles.behavior("Strike")` returns `nil`.
5. **`BehaviorCtx` additions:** `inp: Fsm.InputFrame` and `ray(origin, dir) -> (Vector3?, Vector3?)` (static world, `WorldQuery` group, excluding `workspace.Sim`). Pinned config (Part 3 §3.11.1 `RosterEntry.fullCfg`/`kitSet`, §3.4.2): `cfg` stays the pinned **Combat section**; Part 3 adds `kits` (= `fullCfg.Kits`) and `moves` (= `kitSet.move`). Behaviors and hooks never read `cfg.Combat.X`, a bare `Kits.X` or `Runtime`.
   - **Carrier view content:** the `view` argument of `stepCarrier` is `CarrierView`, with `CarrierOut` and `FighterLite` (§4.2.0.3). `view.cfg`, `view.kits` and `view.moves` are the carrier owner's pinned Combat section, `Kits` section and kit set.
   - **Carrier params:** dynamic only, ≤ 10, always including `lifeF`; static data is read from the MoveDef.
   - **Death:** `dead` → `Dead = true`.
   - **Rush echo** (deferred): an echo carrier (`Kind = "zone"`, `sub = 1`) is dispatched to `Zone.stepCarrier`.
6. **`Fsm.HitDef.kit: KitHitExt?`** (§4.2.0.2; Part 3 §3.4.1 declares it as `kit: any?` so `Types` stays a leaf):
   - `stun`, `debuffs`, `pull`, `meterDrain`, `healSelf`, `grab`, `release`, `reflectable`, `firstOfUse`;
   - Part 3 applies them in the server consequence phase, only on outcome `"hit"` (`grab` on catch); `pull` is a single `KbV` write whose motion the victim derives (item 11);
   - Part 3 sets `firstOfUse`.
   - M1-G users: `stun` (flinch, stumble), `debuffs`, `meterDrain` (`busker.3`), `grab`, `release`, `firstOfUse`. `pull` and `healSelf` have no slice user.
7. **`Fsm.HitboxReq.coneDeg`** (Sphere angular filter), **`Fsm.HitboxReq.innerRadius`** (new: Sphere planar inner exclusion for `ring` zones, used by `sandcastle.a4` and `dustsheet.a3`; Part 3's hit query skips targets whose planar distance from the centre is below it) and **`Fsm.HitRequest.heldOnly`** (targets = victims Grabbed by this attacker), all declared in Part 3 §3.4.1. Part 3 adds `HitRequest.carrier` itself (§4.2.0.5 item 7).
8. **`Hitstun` Sub codes** 2 freeze, 3 float, 4 stumble, 5 possess, 6 flinch (§4.2.0.2), and **`Grabbed` Sub codes** 1–6 (§4.2.4). M1-G uses `Hitstun` 4 and 6 and `Grabbed` 1 and 2; the other codes keep their enum values and their tests are deferred (item 26). `Stagger` Sub 3 is used for counter bounces with the reaction's `hitstun`.
9. **`SpawnRequest.serverOnly: boolean?`** (declared in Part 3 §3.4.1): when true, Part 3 creates the carrier on the server branch only (Part 3 §3.11.5 "server-only creation"). Used by summons and homing projectiles. Targetable summon carriers (`hp > 0`) also get `CanQuery = true`, `CollisionGroup = "Hurtbox"` and `Size = params.hurtbox` (default 3×5×3).
10. **Grab release.** Part 3 detaches `HB_Hold` whenever the holder leaves `Move`, and whenever a hit with `kit.release = true` resolves. Grab ticks may carry `meterDrain`; the finish tick may carry `finishStun` as `kit.stun` (§4.2.4).
11. **Pulls (Server Authority: cause once, effect derived).**
    - **`kit.pull`** (§4.2.0.2; no slice user): on outcome `"hit"` the server writes the victim's `KbV` once, `flat(attRoot − vicRoot).Unit · (max(dist − stopStuds, 0)·60/frames)` (Q-rule). The victim's FSM applies it as a planar push with the existing Push profile (B-class) for `N` frames: `N = StDur` for `Hitstun` Sub 5 (`possess`), else `min(StDur, Stun.MoveKbFramesMax)`.
    - **Zone pull** (§4.2.6; user `lifeguard.a4`): B-class, implemented by Part 3 §3.11.5. In each character's own step (both peers), Part 3's adapter scans the live zone carriers in `workspace.Sim` whose MoveDef has `params.pullStudsPerSec` and whose `Own` is not this character, and adds `Zone.pullVelocity(def, carrierPos, rootPos)` (planar, toward the carrier centre, rounded to 0.01) to the character's motion. There is no per-frame S-class mover write.
    - **Grab attachments:** on catch, Part 3 sets the attacker's `HB_Grab.CFrame = CFrame.new(kit.grab.offset)` and the victim's `HB_Hold.Attachment1` to it; on release it resets `HB_Grab` to (0, 0, −2.5).
12. **Kit-mode variants** (deferred). `Fsm.StyleRuntime.variants: { [moveId]: { MoveDef } }` (declared in Part 3 §3.4.1) from `Styles.variants(def)`, which returns `{}` in M1-G. `Moves.defFor` returns `variants[id][s.Kit + 1]` when present.
13. **Hook dispatch.**
    - Part 3 calls `Styles.passiveStep(h, inp)` after `Fsm.step` (both peers); its `motion` applies only when the FSM motion is `none` and `St == Air`.
    - Part 3 calls `Styles.serverHook(kind, h, info)` for `use, hitDealt, hitTaken, blocked, perfectBlocked, ko, lowHp, wallsplat` (server, consequence phase) and `awaken/awakenEnd` (both). M1-G users: `hitDealt` (`lifeguard`, `busker`, `scoop`, `dustsheet`), `perfectBlocked` (`attendant`, and the `sandcastle` block heal), `blocked` (the `sandcastle` block heal), `use` and `hitDealt` (the Snapshot tracker).
    - `HookCtx` includes `cfg` (pinned Combat section), `kits` (pinned `Kits`) and `moves` (= `RosterEntry.kitSet.move`, the pinned kit set) for lookups.
    - `HookOut` fields are applied as in §4.3.1.
14. **`Moves.STATUS` content** (§4.3.2): `rt`, `sl`, `si`, `du`, `nh` (and `iv`, deferred), the `consumeOn` flag and the `incomingFrom` field.
    - `nextHitAdd` applies to the owner's outgoing hits only.
    - Both adds apply on `firstOfUse`, clamped to the move's band cap.
    - Part 3's `StatusMods.m1Chain` stays in the type; no M1-G status sets it.
15. **Move entry and chassis data.** Part 3's `Move` entry effects zero `MvA` and `MvB`. **Chassis data convention:** kits use exactly Part 3's recognized keys (`ragdollFramesAdd`, `lowHpThreshold` + `lowHp`, `m1RangeAdd`, `walkSpeedMult`, `breakawayRefillMult`, `enderDamageAdd`, `airDash`). Other `passive.params` keys (`perfectBlockCdSec`, `enderSlowPct`, `enderSlowFrames`, `blockHeal`, `blockHealEveryF`, `enderCdSec`, `toppingsCdSec`, `maxStacks`, `stackFrames`, `perStack`, `maxBonus`) are read only by Part 4 hooks; Part 3 ignores them. `awakening.modifiers` holds recognized keys only (K11). Behavior `patch` writes `MvA`, `MvB`, `Kit` (B) and `Buffs`, `StDur`, `CdQ` (B·S). Part 3's adapter must accept these from behavior and hook outputs (Part 3 §3.11.4).
16. **`Kit` attribute semantics per style:** 0 for every slice style (mode toggles and hover are deferred). `MvA`/`MvB` usage per behavior: once-bits, hit counters, grab hold data (§4.2).
17. **SimEvent kinds added:** `"summon_down"`. `"style_moment"` (a Snapshot) uses `a = victim rid`.
18. **Asset keys.**
    - `anim.<style>.<slot>`, `anim.<style>.realm` (RealmCast), `anim.<style>.awaken`, `anim.<style>.m1_<sub>` / `anim.<style>.ender_<sub>` (Part 3 §3.11.8 keys; Part 4 supplies poses), `vfx.<style>.<slot>`, `vfx.status.<kind>` (`root`, `slow`, `silence`, `dust`, `ward`; `freeze` and `stealth` deferred), `vfx.dustsheet.getaway_dark`, `sfx.<style>.<slot>`, `sfx.dustsheet.getaway_bell`, `model.<style>.gear_<name>`, `model.<style>.summon_<name>`, and the optional `model.<style>.peak_<name>` (one per slice style: `peak_blazer`, `peak_flag`, `peak_lamp`, `peak_drum`, `peak_kiosk`, `peak_lanterns`; templates `Looks.Peak_<style>_<name>`, §4.7.4).
    - Recipe data: every Charm Gear, Peak-piece and summon primitive carries a colour role (`primary`, `secondary`, `accent`, `ink`); `accent` parts of Charm Gear and Peak pieces take `Theme.Gilt` at Peak (§4.7.4). `vfx.dustsheet.a4` uses Part 6's `VFX_Getaway_Frame`.
    - New VFX recipes: `VFX_Telegraph_Arc`, `VFX_Projectile_Orb`, `VFX_Zone_Disc`, `VFX_Wave_Wall`, `VFX_Stealth_Shimmer`, `VFX_Status_Pips`, `VFX_Getaway_Quarter`; `VFX_Encase_Block` deferred.
    - `Styles.SFX_SPEED`.
    - `kitProps`: the two Charm Gear keys, then the summon model keys (K23; §0.8).
19. **Placeholder animation contract** (§4.6): naming, attributes (`AssetKey`, `MoveId`, `Frames`, `Gen`), marker names and values, the timing-lock rule, the six new poses (`sit`, `carry`, `dig`, `squeeze`, `duck`, `flop`) and `docs/reference/framedata.json`.
20. **Log lines:** `[HB][Style] registry ok …`, `[HB][Style] registry loaded …`, `[HB][Style] ERROR rule=…`, `[HB][Style] moment …`, `[HB][Style] carrier cap …`.
21. **Lune specs and test files:** `tests/core/styles.spec.luau`, `kits.spec.luau`, `behaviors.spec.luau`, `kit_status.spec.luau`, `tests/shim/FakeCtx.luau` (incl. `FakeCtx.member`). **Scenarios:** `style.registry_boot`, `config.kits_override_next_life`, `kit.<style>.smoke` ×6 (`attendant`, `lifeguard`, `sandcastle`, `busker`, `scoop`, `dustsheet`), `kit.attendant.trap` (E2), `kit.scoop.meltaway_iframes`, `kit.dustsheet.getaway` (incl. E1), `kit.awaken.swap`, `kit.grab_vs_jump` (`attendant.4`), `kit.grab_breakaway`, `kit.moment_meter`, `anim.placeholders_built`, `anim.timing_lock`, `look.gear_silhouette` (§4.7.4; stills for T8). Lune tests that pin the zone blockable defaults, the B-class zone pull, the ring filter and E2: `behaviors.zone_blockable_defaults`, `behaviors.zone_pull_b_class`, `behaviors.zone_ring_inner`, `behaviors.summon_chase_nearest_carrier`. Old kit scenarios with no slice user are removed; the deferred ones (reflect, hover, stealth ranges) return with their first user.
22. **Getaway content:** `Realm.rules["dustsheet.a4"]` (§4.4.3), keyed by MoveDef id (Part 3 §3.9.3); its timing table is mirrored in `dustsheet.a4` `params.getaway` for render, and Lune `kits.getaway_schedule_matches` keeps the two equal.
23. **E1 (M1-G): a Getaway rule tick reads the member's position.** Part 3 sets `MemberView.pos` (the member's root position at the tick) and provides `Realm.quadrant(center, pos)` (1–4 clockwise; Part 3 §3.9.3). Rules derive timing from `f − r.openF` only, so a tick is deterministic for a given frame and position. User: `dustsheet.a4` *Lights Out* (quarter test). Later user (Year 1): `serpent.a4` (*Hump Rings*: an inside-a-ring test with `Realm.contains`, GDD §7.3.17). Tests: Part 3 `realm.member_pos_quadrant`; Part 4 `kits.getaway_rules`, scenario `kit.dustsheet.getaway`. **If E1 slips,** `LIGHTS_DAMAGE = false` ships Lights Out as darkness only. Cost 0.5–1 day.
24. **E2 (M0): `chase` targets the enemy nearest the carrier.** Target = the enemy nearest the carrier among those within `leash` studs of the carrier (ties by rid); `speed = 0` makes the summon a trap (`once`) or a turret (§4.2.7). No new param is needed. It lands with the Summon behavior, before the M0 kits, because `attendant.q` (*Reserved*) is an M0 move. Users: `attendant.q`, `sandcastle.2`, `sandcastle.a2`; Year-1 users (GDD §7.3): `bellhop.3`, `angler.2`, `boxcamera.2`, `diver.3`, `serpent.3`. Tests: `behaviors.summon_chase_nearest_carrier`, scenario `kit.attendant.trap` (a speed-0 summon fires at 4 studs). Cost 0–0.5 day.
25. **Future hooks E3–E7 (Year 1; not built in M1-G).** Each lands as its own task, with a deterministic rule and a Lune test, before its first user:
    - **E3** two statuses through §4.3.2: `dz` Dazzled (the victim's next block raises 6 f late, `blockDelayF = 6`, removed on that block; a flash render) and `fg` Fogged (render only: hides enemy nameplates and the Style Tell beyond 12 studs; telegraphs and hit flashes are never hidden). Users: `boxcamera`, `lamplighter`; `serpent`, `ferry`.
    - **E4** Getaway rule phases: the rule set changes on a fixed cycle derived from `f − r.openF`, with the next phase shown 1 s ahead. User: `ferry`.
    - **E5** Getaway strikes at positions locked 1 s ahead (the framework records each enemy's position at the lock frame, server memory, S-class outputs only), and caster repositioning on a fixed pattern. Users: `raincloud`, `serpent`.
    - **E6** Getaway-wide drift (a slow planar push on every member, B-class like the zone pull; optional, cut if costly). User: `ferry`.
    - **E7** Leaving a Getaway early costs a Breakaway pip (optional polish; it needs the framework to let members leave). User: `dustsheet`.
26. **Deferred and dropped features.**
    - **Deferred** (types and enum values kept; implementation and tests land with the first style that needs them): Buff `toggleMode` and `Styles.variants`; hover `passiveStep`; Rush `echo`; Mobility `grapple`, `flight`, `slide`; Summon `marked` AI; the Counter reflect path and `Projectile.reflectCarrier` (the `reflectable` flag stays on every carrier projectile); the stealth status `iv` render; the hitstun flavours `freeze`, `float` and `possess` and `Grabbed` Subs 3–6 (enum values kept, tests deferred). `validate` rejects a deferred feature in M1-G data, so nothing half-built can ship.
    - **Dropped:** the Buff `analyze` panel; statuses `wh` and `ff`; heal reversal in Getaway rules (heal multipliers are never negative, Part 3 §3.9.3); the Peak modifiers `visualScale` and `aura`; the scenario `kit.awaken.hurtbox`.
    - **No Peak scales the hurtbox:** `hurtboxScale = 1` everywhere (K10).

### 4.10 Issues with spine

1. **`Strike` has no module.** §0.8's `Behavior` includes `"Strike"`, but the §0.3 layout lists no `Behaviors/Strike.luau`. Resolved as Part 3 does: Strike is Core's plain-melee default and `Styles.behavior("Strike") == nil`.
2. **"Every GDD tunable in Defaults" vs kit data in code.** §0.4 says `Defaults.luau` holds every tunable. StyleDefs use `Vector3`/`Color3` and cannot live in JSON config (Part 2 §2.6.1). Move numbers therefore live in `Styles/*`, and live tuning goes through the new `Kits.Overrides` layer. `Kits` is a new top-level key not in §0.9's fixed list; it needs a spine amendment.
3. **No file for shared kit types.** §0.3 fixes the file list. Part 4's shared types and helpers live in `Behaviors/Buff.luau` (the "kit kernel"), because a separate module would add a file.
4. **Core ownership.** Part 2 lists `Core/Realm.luau` as Part 4's; Part 3 claims the framework and leaves only `Realm.rules` to Part 4. This part follows Part 3, and also writes `Moves.STATUS` content inside Part 3's `Core/Moves.luau`.
5. **`kitProps` holds gear and summon keys.** Resolved: §0.8 now states the order (the two `gear_` keys, then the summon keys; K23) and that Part 6's AppearanceController attaches only the `gear_` entries as Charm Gear. Peak pieces are not listed in `kitProps`.
6. **Extender rule vs GDD kits.** GDD §6.7 requires ≥ 1 extender per base kit. The GDD kit tables for `sandcastle` and `dustsheet` (GDD §7.3.9, §7.3.15) tag no move `Extender`. K04 accepts a frame-verified `params.extender` (sandcastle.1, dustsheet.1) without changing GDD tags. `busker.1` carries the `Extender` tag and its frame data from GDD Appendix A. **Resolved:** the GDD §7.2 move-table legend confirms that a move marked extender combos from the M1 chain, whether Part 4 proves it with the tag or with frame data.
7. **Log tag.** The brief says `[HB][STYLE]`; Part 2's tag list uses `Style`, so lines read `[HB][Style]`.

### 4.11 Open questions / spikes

Spikes (proposed for P0):
- **S-P4-1 Predicted carriers.** This is Part 3 spike SA-09 (stitching when two characters spawn carriers in the same frame). If it fails, Part 3 falls back to server-only carriers plus a cosmetic ghost. Part 4 content works either way: `stepCarrier` is deterministic and hits are server-only. To support ghosts, `Projectile` exports the pure `Projectile.trajectory(def, origin, age) -> CFrame`, which RenderController (Part 6) uses.
- **S-P4-2 Grab hold smoothness** under Server Authority via `HB_Hold` (AlignPosition) for the victim's client at 150 ms. Measure misprediction count per hold (`attendant.4` hold, `lifeguard.4` shoulder lift).
- **S-P4-3 Status write and pull cost.** An S-class `Debuffs` write per hit plus zone `stay` refreshes every 6 f: measure rollbacks per second on a victim standing in a zone (`busker.a4`, `scoop.4`). Also measure rollbacks per second on a victim inside `lifeguard.a4` (B-class zone pull toward an `aim` carrier, §4.2.6). Fallback for statuses: a refresh every 12 f with `frames = 24`. Fallback for the zone pull: lower `pullStudsPerSec` (td value); if that is not enough, keep the drag as render only and record it in `PROGRESS.md` as a `spec-gap` blocker (Part 1 §1.8.1), because the GDD asks for the drag.
- **S-P4-5** Charm Gear and Peak-piece mounts on the default R15 rig: `RightGripAttachment` on `RightHand`, and `BodyFrontAttachment` and `BodyBackAttachment` on `UpperTorso` (the back mount carries the `attendant` deckchair, the `lifeguard` buoy and three Peak pieces). The docs snapshot does not confirm these on the default rig. Fallback: a `RigidConstraint` to `RightHand` / `UpperTorso` for each one that is missing.
- **S-P4-6** `@lune/roblox` exposes `Vector3.new`, `CFrame.Angles`, `CFrame.lookAt` and `Color3.fromRGB` for the Lune shim.
- **S-P4-7** Pose sign conventions (§4.6.2) via `Animator:StepAnimations` + `screen_capture`, including the six slice poses.

Design questions (the owner signs these off at H3):
- **Q1 Dust interpretation.** "Her move hits and her Ender add Dust … each stack adds +1 to her first hit on that target per move." Implemented as:
  - stacks are not consumed;
  - the bonus applies to the first landed hit of each use, clamped to the band;
  - among M1s only the Ender adds a stack;
  - zone ticks, Getaway rule hits and zero-damage requests add none;
  - a summon cast adds its stacks once per use, however many bodies land.
- **Q2 Toppings "already slowed".** Read as: the hit's own move applies no slow, M1-family hits never count, and one cast procs at most once. The alternative (any hit on a slowed target) would let a slowing move refund itself.
- **Q3 Hard Hat and Perfect Blocks.** A Perfect Block also heals 1 HP (still at most once per second). The alternative is blocks only.
- **Q4 Diameter reading.** "N-stud circle" is a diameter (lifeguard.a1's 16-stud circle → radius 8); "within N studs" and "between A and B studs" are radii (busker.4 10, busker.a4 18, sandcastle.a4 8–25).
- **Q5 Unspecified sizes.** Buoy Toss bonks inside its own 4-stud slow radius; the No Swimming! spike (4) hits the whole 16-stud circle; Whirlpool radius 10; Meltdown puddle radius 5; Draped! sheet radius 5; Castle Keep waves 0.75 s apart; the Grand Castle can stand beside the small castle. All (td).
- **Q6 Snapshot tokens from zone ticks.** A zone tick pushes a `:hit` token, so a moat tick opens *Flood the Moat*. The alternative is only the first tick of each use.
- **Q7 Lights Out order.** Quarter 1 (Part 3's `Realm.quadrant` numbering) goes dark first, then clockwise every 3 s; the caster is not hidden in the dark. The alternative is to start with the quarter the caster faces.
- **Q8 Slow stacking.** Additive slows capped at 60% (td) vs strongest-wins. Example: Sing-Along's −40% plus a Buoy Toss −15% gives −55% today.
- **Q9 Stealth vs Style Tell** (deferred with `iv`). Hiding the nameplate beyond 12 studs while stealthed conflicts with "Style Tell always on" (§0.4). Proposal for the first stealth style: stealth suppresses the nameplate only, and the shimmer keeps the style colour.
- **Q10 E2 scope.** Every `chase` summon targets from the carrier, with `leash` measured from the carrier. The alternative keeps owner-based targeting for moving summons and adds one param for speed-0 traps and turrets.
