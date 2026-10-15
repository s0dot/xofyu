[← Build spec index](../STUDIO_BUILD_SPEC.md) · **Part 4 of 8** · sections §4.x · Part 0 contracts (§0.x) are in the index file

> ⚠️ **DRAFT — pre-review snapshot.** An editorial/technical review of this spec found ~200 issues and the fixed
> v1.0 is still being prepared. **Agents: do not execute any task card beyond the P0 human-setup steps (H-01…H-12)
> until this banner is gone.** Humans can start the one-time setup in Part 1 §1.2 now.

## 4. Style kit framework and vertical-slice kits

This part specifies how a Fighting Style is represented, validated and executed. It covers:
- the registry `Shared/Styles/init.luau`;
- the nine behavior modules `Shared/Behaviors/*.luau`;
- the passive/Quirk hook system;
- the Awakening and Realm content;
- complete data for the six vertical-slice kits (`hero`, `stretch`, `monarch`, `rocker`, `frost`, `decay`) and the 12 stubs;
- the placeholder-animation spec;
- presentation mapping (palettes, VFX, SFX, Style Gear);
- the checklist for adding a style.

**Relationship to Part 3 (binding).** Part 3 owns the simulation: the FSM, `Core/*` (including the status codec in `Core/Moves.luau` and the framework in `Core/Realm.luau`), hit resolution, carriers (spawned instances) and the adapter that calls behaviors. Part 4 plugs into Part 3's contracts **by name**, from Part 3 §3.4.2 and §3.9.3:
- `Fsm.CharState`, `Fsm.StepCtx`, `Fsm.InputFrame`, `Fsm.HitDef`, `Fsm.HitboxReq`, `Fsm.HitRequest`, `Fsm.SpawnRequest`, `Fsm.MotionCmd`, `Fsm.SimEvent`, `Fsm.ChassisMods`, `Fsm.StyleRuntime`;
- `BehaviorCtx`, `BehaviorOut`, `BehaviorModule {step, onContact?, onEnd?}`;
- `Moves.Status` / `Moves.STATUS` / `Moves.statusAdd`;
- `Realm.rules`, `RealmView`, `MemberView`, `RealmRules`.

Where Part 4 needs something Part 3 has not defined, it is listed in §4.9 as an **extension request**. If Part 3 later defines an equivalent under another name, Part 3's name wins and the semantics here still bind.

Conventions:
- Frames are 60 Hz. `e` is the frame elapsed in the current state (0-based, Part 3 convention): startup is `e ∈ [0, S−1]`, active is `[S, S+A−1]`, recovery is `[S+A, S+A+R−1]`. "Move frame n" (1-based) means `e + 1 = n`. Distances are in studs, times in seconds (`Sec` suffix).
- `(td)` means a tuning default chosen here because the GDD gives no number. It satisfies the cited GDD rule.
- GDD numbers are cited as `GDD §x.y` and are never changed.
- Every requirement has a **Verify** line with one of:
  - a Lune test (`tests/core/<file>.spec.luau`, test name);
  - an in-Studio scenario (Part 7 harness, `[ALG][TEST] <name> PASS|FAIL`);
  - an MCP call;
  - an `[ALG]` log line.
- Every engine API named here was checked in the offline reference snapshot (2026-10-10). Unverifiable items say "(verify in spike)".

**Files owned by this part:**

| Path | Contents |
|---|---|
| `src/Shared/Styles/init.luau` | Registry, `KitBuilder` implementation, 12 stubs, validation, config overrides, server hook dispatch, Style Moment tracker, `SFX_SPEED` table |
| `src/Shared/Styles/hero.luau` … `decay.luau` | One kit each: `StyleDef` + passive hooks + moment + animation spec (§4.5) |
| `src/Shared/Behaviors/Buff.luau` | Buff/Debuff behavior **and the kit kernel**: shared Part 4 types, status helpers, hit/aim helpers (§4.2.0, §4.2.8) |
| `src/Shared/Behaviors/Rush.luau`, `Projectile.luau`, `Grab.luau`, `Counter.luau`, `Zone.luau`, `Summon.luau`, `Mobility.luau`, `RealmCast.luau` | Behaviors (§4.2) |
| Content inside Part 3 files | `Moves.STATUS` entries in `Core/Moves.luau` (§4.3.2); `Realm.rules["decay.a4"]` in `Core/Realm.luau` (§4.4.4) |
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
5. **No canonical names** in code, identifiers, asset keys or comments (Part 1 §1.7.6, Part 8 lint). Canon references are written as pointers, e.g. `-- canon: GDD §7.3.1 row 3`.
6. **Writes only through outputs.** Behaviors return `BehaviorOut` / `CarrierOut` / `HookOut`; they never mutate their inputs (`table.clone` first).
7. **Player-visible text** comes from Skin keys `style.<id>.*` and `move.<moveId>.name`. Kit data contains no display strings.

---

### 4.1 Styles registry (`src/Shared/Styles/init.luau`)

#### 4.1.1 Module shapes

Each kit file returns a **builder function**. Kit files never require `Styles/init`, which avoids a require cycle. The registry passes in the shared constructors:

```lua
--!strict
-- src/Shared/Styles/hero.luau (shape only; full data in §4.5.2)
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
	assert(styleId ~= nil and slot ~= nil, `[ALG][Style] bad move id {id}`)
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
	get: (id: string) -> Types.StyleDef,                 -- errors "[ALG][Style] unknown style <id>"
	tryGet: (id: string) -> Types.StyleDef?,
	all: () -> { Types.StyleDef },                       -- 18 defs in Types.STYLE_IDS order (frozen)
	implemented: () -> { Types.StyleDef },               -- implemented == true and not runtime-disabled
	isPlayable: (id: string, cfg: Types.Config) -> boolean, -- implemented, valid, not in Features.DisabledStyles
	move: (moveId: string) -> Types.MoveDef?,            -- any slot incl. q, a1..a4 (and stub moves)
	hooks: (id: string) -> Kit.PassiveHooks,             -- {} for stubs
	moment: (id: string) -> Kit.MomentDef?,
	anim: (id: string) -> { [string]: Kit.AnimMoveSpec },
	behavior: (b: Types.Behavior) -> Kit.KitBehavior?,   -- nil for "Strike" (Core default, Part 3 §3.4.2)
	variants: (def: Types.StyleDef) -> { [string]: { Types.MoveDef } }, -- Kit-mode variants (§4.3.4), for Part 3 StyleRuntime
	forConfig: (cfg: Types.Config) -> KitSet,            -- defs with Kits.Overrides applied (§4.1.6); cached per cfg table
	validate: (cfg: Types.Config) -> (boolean, { Problem }),
	maxDamage: (move: Types.MoveDef) -> number,          -- per-target maximum for one use (§4.1.4)
	passiveStep: (h: Kit.HookCtx, inp: any) -> Kit.HookOut?,                  -- §4.3.1, both peers
	serverHook: (kind: Kit.HookKind, h: Kit.HookCtx, info: Kit.HitInfo?) -> Kit.HookOut?, -- §4.3.1, server
	SFX_SPEED: { [string]: number },                     -- §4.7.3 (Part 6 SfxController reads)
}
export type KitSet = { get: (id: string) -> Types.StyleDef, move: (moveId: string) -> Types.MoveDef? }
export type Problem = { rule: string, style: string, move: string?, detail: string }
```

Semantics:
- `get`/`all`/`move` return **frozen** tables (`table.freeze`, applied recursively at build). Callers that need a modified MoveDef use `table.clone`.
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

`[ALG][Style] ERROR rule=<id> style=<styleId> move=<moveId|-> detail=<text>`

The tag is `Style`, from Part 2 §2.3.6. Rules S01–S06 apply to all styles. K-rules apply only to `implemented = true` styles. Stubs are checked by S-rules only (§0.10).

Numeric thresholds come from config, not literals:
- Part 3's `Combat.Damage` (`MaxBaseMove` 18, `MaxQuirk` 6, `MaxUltimate` 30, `TellMinFrames` 8, `TellDamageAbove` 10, `CooldownBands`, `Summon`) and `Combat.Block` (`MoveGuardMin` 20, `MoveGuardMax` 60, `MoveChipDefault` 0.2);
- `Kits` (`AwakenedMaxDamage` 20, `AwakenedCooldownBand`, `PerHitGuardBand`), for the bands Part 3 does not define.

The numbers in the table are the GDD values those keys hold.

`maxDamage(move)` is the per-target maximum damage for one use. It is computed by the behavior's `maxDamage` (§4.2):
- single hit: `damage`;
- `params.hits` / `params.ticks`: their sum;
- echo: `damage + echo.damage`;
- tick zones: `min(damage, tick.damage × maxHitsPerTarget)`;
- summons with `dmgModel = "perHit"`: `damage × maxHitsPerTarget × bodies`;
- counters: `damage`.

| ID | Scope | Check | Source |
|---|---|---|---|
| S01 | registry | Exactly 18 defs; ids equal `Types.STYLE_IDS`, same order | §0.4 |
| S02 | all | `rarity` equals the §4.5.8 table | GDD §7.2, §8.3 |
| S03 | registry | `implemented == true` exactly for `hero, stretch, monarch, rocker, frost, decay`; equals `Types.IMPLEMENTED_M1G` | §0.0, §10 |
| S04 | all | `hp == 100` | GDD §6.2 |
| S05 | all | `archetype` matches `^%u%a+$`; palette has three `Color3` values | §4.7.1 |
| S06 | implemented | Primary colours pairwise Euclidean RGB distance ≥ 40 (0–255 space) (td) | GDD §12.3 readability |
| K01 | implemented | Quirk id `<id>.q`. `moves` are exactly slots `"1".."4"` in order with ids `<id>.<slot>`. Awakening id `<id>.awaken`; `awakening.moves` are slots `a1..a4` in order. Passive id `<id>.passive`. All move ids unique | §0.4 |
| K02 | implemented | Exactly one base move has `signature == true`; quirk and awakened moves do not | GDD §6.7 |
| K03 | implemented | ≥ 1 opener: a base move tagged `Projectile`, `Rush` or `Grab` | GDD §6.7 |
| K04 | implemented | ≥ 1 extender, either a base move tagged `Extender`, or one with `params.extender == true` **and** `startup + 1 + travelF ≤ m1Adv`. `travelF = ceil(5 · 60 / params.speed)` for bullets (5 studs = M1 reach, td), 0 for melee and beams. `m1Adv = hitstun − (active − 1) − recovery` of M1-3 from `cfg.Combat.M1` (= 27 − 2 − 13 = 12, GDD §6.3) | GDD §6.7 |
| K05 | implemented | ≥ 1 defensive/mobility tool among base moves + quirk: tag `Counter`, `Mobility` or `Zone`, or `params.iFrames ~= nil` | GDD §6.7 |
| K06 | implemented | `maxDamage`: base ≤ 18; quirk ≤ 6; a1–a3 ≤ 20; a4 ≤ 30 | GDD §6.7, §7.1 rule 4 |
| K07 | implemented | Cooldown bands: signature 14–22; other base 6–16 (basic 6–10 ∪ utility 10–16); quirk 3–8 (≤ 12 if `params.random == true`); a1–a3 6–22 (td); a4 = 25 if tagged Ultimate, 0 if `RealmCast` | GDD §6.7, §7.3 |
| K08 | implemented | If `maxDamage > 10`, or any single hit > 10: `startup ≥ 8`, `telegraph ~= nil`, `telegraph.leadFrames ≥ 8` and `≤ tellFrames(move)`, and `tellFrames(move) ≥ 8` | GDD §6.14 #1, §7.1 rule 5 |
| K09 | `Summon` moves | `bodies ≤ 3`; `bodies × hpEach ∈ [20, 60]`; `lifeF ∈ [360, 720]`. Exception: slot a4, tag `Ultimate`, `params.untargetable == true` → only `bodies ≤ 3` and `lifeF ≤ 360` apply (an untargetable set piece has no HP) | GDD §6.7 Summon |
| K10 | awakening | `hurtboxScale == nil or ≥ 1`; `durationSec == cfg.Combat.Awakening.DurationSec` (60) | GDD §6.2, §6.9 |
| K11 | awakening | `modifiers` keys ⊆ Part 3 recognized keys (`breakawayRefillMult`, `enderDamageAdd`, `ragdollFramesAdd`, `m1RangeAdd`, `walkSpeedMult`, `airDash`) ∪ {`visualScale`, `aura`} (render only; Part 3 ignores them) | §4.3.3, §4.4.2 |
| K12 | a4 | Rarity ∈ {legendary, mythic, cosmic} ⇔ `behavior == "RealmCast"` and tags contain `Realm`. Otherwise tags contain `Ultimate` | GDD §6.10, §7.1 |
| K13 | a4 Ultimate | `params.cinematicFrames` (default 0) ≤ 120 | GDD §6.8 |
| K14 | damaging moves | If `blockable(move)`: `guardDamage ∈ [20, 60]` (model `total`) or `∈ [8, 30]` (model `perHit`, td). Otherwise `guardDamage == 0` | GDD §6.4 |
| K15 | damaging moves | `chip == 0.2` if `blockable(move)`, else `chip == 0` | GDD §6.4 |
| K16 | all moves | `startup ≥ 1`, `active ≥ 1`, `recovery ≥ 0` (integers); total ≤ 600; `hitstun` integer 0–180; damaging moves `startup ≥ 6` | GDD §6.1, §18.2 |
| K17 | all moves | `anim/vfx/sfx == "<kind>.<style>.<slot>"` (RealmCast `anim == "anim.<style>.realm"`, Part 3 §3.11.8), matching the Assets key pattern | Part 2 §2.9.1 |
| K18 | all moves | `Moves.validate(def, slot, cfg)` (Part 3) returns no problems; `behavior(b).validate(def)` returns no problems | §4.2 |
| K19 | all moves | Tags ⊆ `Types.Tag`. Behavior↔tag: Grab⇒`Grab`; Counter⇒`Counter`; Summon⇒`Summon`; RealmCast⇒`Realm`; Projectile⇒`Projectile`; Zone⇒`Zone` or `AoE`; Rush⇒`Rush`; Mobility⇒`Mobility`; Buff⇒ one of `Buff`/`Debuff`/`Utility`/`Mobility`; Strike⇒ any | GDD §6.7 |
| K20 | implemented | `hitbox ~= nil` for Strike, Rush and Grab, except Grab with `params.sweep` | §4.2 |
| K21 | multi-hit | `Σ params.hits` and `Σ params.ticks[i].damage` each equal `damage` (± 1e-6) | §4.5.1 |
| K22 | knockback | `0 ≤ dist ≤ 60`; `−90 ≤ angleDeg ≤ 90` | (td) |
| K23 | implemented | `m1 = {anim = "anim.<id>.m1", vfx = "vfx.<id>.m1", sfx = "sfx.<id>.m1"}`; every `kitProps[i]` matches `^model%.<id>%.gear_%l+$` | §4.7 |
| K24 | implemented | `anim` has specs for `m1` (12-entry `alt`), `q`, `1..4`, `a1..a4`, `awaken`; every pose name exists in the pose library (§4.6.2) | §4.6 |
| K25 | overrides | Each `Kits.Overrides` record names an existing move and an allowed field; the overridden move passes K06–K16 | §4.1.6 |

**Verify:**
- Lune `styles.registry_18`, `styles.ids_match_types`, `styles.rarities_match_gdd`, `styles.implemented_set` (also covers Part 2's `types.implemented_matches_styles`), `styles.validate_all_clean` (`validate(Defaults)` returns `true, {}`).
- One negative test per rule, `styles.rule_<ID>_detects`. Each clones one valid kit, mutates one field (e.g. `hero.2` damage 19), and asserts the rule id appears in the problems.

#### 4.1.5 Boot behavior

- **Server.** `StyleService.init()` (Part 5 owns the service; this call is binding) runs `Styles.validate(Runtime.get())` and logs each problem.
  - In Studio, any problem calls `error("[ALG][Style] kit validation failed")`. Boot then fails loudly (Part 2 §2.3.4).
  - Outside Studio, each implemented style with a problem goes into a runtime-disabled set (so `isPlayable` is false). If `hero` or `stretch` fails, it errors (no fallback style exists).
  - On success: `[ALG][Style] registry ok styles=18 implemented=6 moves=<n> problems=0`.
- **Client.** It does not validate (same code). It logs `[ALG][Style] registry loaded styles=18` once.
- **Config change.** On `Runtime.changed`, the server re-validates `forConfig(newCfg)`. If a `Kits.Overrides` record fails, ConfigReplicator rejects that section (Part 2 §2.6.4 step 3) with `[ALG][Config] ERROR rejected section=Kits problems=<…>`.

**Verify:** scenario `style.registry_boot` expects the `registry ok` line within 5 s of play start, and zero `[ALG][Style] ERROR` lines.

#### 4.1.6 `Kits` config section (new top-level key, §4.9 item 3)

Kit numbers live in code, not config: StyleDefs contain `Vector3`/`Color3`, which are not JSON-safe (Part 2 §2.6.1). `Kits` gives live-ops a JSON-safe override layer and the carrier caps. It is **pinned with the combat config**: the character's life uses `Runtime.at(CombatCfg).Kits`.

```lua
Kits = {
	Overrides = {},              -- array of { move = "hero.3", field = "damage", value = 6 } (td: empty)
	OverrideFields = { "damage", "guardDamage", "chip", "hitstun", "cooldown", "startup", "active", "recovery", "meterGain" },
	MaxCarriersPerFighter = 12,  -- (td) oldest owned carrier despawns when exceeded
	MaxCarriersServer = 200,     -- (td) spawn refused (logged once per 10 s) when exceeded
	MomentCooldownFrames = 600,  -- (td) one Style Moment per attacker per 10 s
	StatusStayRefreshFrames = 6, -- (td) zone "stay" debuffs are re-applied every 6 f
	AwakenedMaxDamage = 20,      -- GDD §6.7 / §7.1 rule 4 (awakened moves 1–3)
	AwakenedCooldownBand = { 6, 22 }, -- (td) a1–a3 cooldown sanity band (GDD lists 8–16)
	PerHitGuardBand = { 8, 30 }, -- (td) guard damage per hit for "perHit"-model moves (summon pokes, orbit touches)
}
```

- `Overrides` is an **array** so that Part 2's `Schema.merge` accepts additions: an empty base array accepts any elements (Part 2 §2.6.3 rule 5).
- `forConfig(cfg)` applies records in order to deep copies. The result is frozen and cached in a weak-keyed table indexed by the frozen `cfg` table.
- ConfigService key `ALG_Kits` overlays `Kits` and takes effect at the next spawn (pinning, Part 2 §2.6.6).
- Schema rules:
  - `Kits.Overrides[*].move` matches `^%l+%.[%w]+$`;
  - `field ∈ OverrideFields`;
  - `value` finite in `[0, 600]`;
  - `MaxCarriersPerFighter` int 1–32;
  - `MaxCarriersServer` int 16–512;
  - `AwakenedMaxDamage` num 1–30;
  - `AwakenedCooldownBand` and `PerHitGuardBand` are 2-element num arrays with `[1] ≤ [2]`.

**Verify:** Lune `styles.overrides_apply` (override `hero.3.damage = 6` → `forConfig(cfg).move("hero.3").damage == 6`; base registry unchanged). Lune `styles.overrides_invalid_rejected` (`hero.2.damage = 19` → `validate` reports K06). Scenario `config.kits_override_next_life` (`Dev.Command setConfig Kits`; the current life keeps the old value; after respawn the new damage applies).

#### 4.1.7 Lune specs owned by this section

`tests/core/styles.spec.luau`: every test named in §4.1.4–§4.1.6, plus:
- `styles.move_ids_unique`;
- `styles.frozen` (assigning to `Styles.get("hero").hp` errors);
- `styles.behavior_strike_nil`.

---

### 4.2 Behavior modules (`src/Shared/Behaviors/*`)

#### 4.2.0 Common contract

##### 4.2.0.1 What a behavior is

A behavior turns one `MoveDef` into commands, frame by frame. It is a table of pure functions over Part 3's types. Part 3's adapter calls them on these paths:

| Function | Called by (Part 3) | Peers | When |
|---|---|---|---|
| `step(bc) -> BehaviorOut?` | `Fsm.step` (state tick) | Owner client + server | Every frame the owner is in `St = Move` |
| `onContact(bc, result) -> BehaviorOut?` | Hit resolution | Server always; owner client for BQ predictions | After a HitRequest from this move resolves on a target |
| `onEnd(bc) -> BehaviorOut?` | State exit | Both | On Move exit, including interrupts and grab breaks |
| `intercept(bc, hit, srcPos) -> Intercept?` (ext.) | Server hit pipeline, **before** `Damage.resolve` | Server | The victim is in `Move` and its def's behavior defines `intercept` |
| `stepCarrier(view, f) -> CarrierOut?` (Part 3 name, §3.11.5) | Carrier loop (phase 1b) | Server; owner client for its own predicted carriers (Part 3: `Own == localRid`), where hits are ignored | Every frame per live carrier whose MoveDef uses this behavior |
| `hurtCarrier(view, hit, attackerRid) -> CarrierOut?` (ext.) | Hit resolution | Server | A targetable summon carrier is hit |
| `reflectCarrier(view, newOwnerRid) -> CarrierOut?` (ext.) | Hit resolution | Server | A projectile carrier hits a reflecting Counter (§4.2.5) |
| `validate(def)`, `maxDamage(def)`, `tellFrames(def)`, `blockable(def)` (ext.) | `Styles.validate` | Lune + boot | Pure checks (§4.1.4) |

"(ext.)" marks an extension to Part 3's `BehaviorModule`, requested in §4.9 item 4.

##### 4.2.0.2 Inputs and outputs (Part 3 types, used verbatim)

- `BehaviorCtx = { f, e, phase, def, s, ctx, cfg, isServer }` (Part 3 §3.4.2).
  - `bc.ctx.rootCf` is the root at step start.
  - Facing follows aim during startup and holds from the first active frame (Part 3 §3.2.7).
  - Part 4 additionally needs `bc.inp: Fsm.InputFrame` (aim, moveDir, bits) and `bc.ray(origin, dir) -> (Vector3?, Vector3?)`. `ray` is a static-world raycast in collision group `WorldQuery` that excludes `workspace.Sim` (§4.9 item 5).
- `BehaviorOut = { motion?, hits?, spawns?, patch?, events? }`.
  - `patch` may write the owner's `MvA`, `MvB`, `Kit`, `Buffs`, `StDur` and `Cd1..Cd4/CdQ`. All are B or B·S class in Part 3 §3.3.1.
  - Part 3's Move entry effects set `MvA = MvB = 0` (requested in §4.9 item 15).
- `HitRequest = { key, box: HitboxReq, hit: HitDef, maxTargets }`. Behaviors build `hit` with Part 3's `Moves.hitDef(def, cfg)` and then adjust per-hit fields. `key` is `"<moveId>#<n>"`; Part 3 scopes the registry to the current action instance.
- `SpawnRequest = { kind, moveId, origin, params }`. `params` are JSON-safe carrier attributes (§4.2.0.5).

Two Part 4 extension fields ride on Part 3's types (§4.9 items 6 and 7):

```lua
-- Fsm.HitDef gains:  kit: KitHitExt?          Fsm.HitboxReq gains:  coneDeg: number?  (Sphere only)
-- Fsm.HitRequest gains: heldOnly: boolean?    (targets = victims currently Grabbed by this attacker)
export type StunFlavor = "hit" | "freeze" | "float" | "stumble" | "possess" | "flinch"
export type StatusReq = { kind: string, frames: number, n: number }        -- relative duration; Part 3 adds f
export type KitHitExt = {
	stun: StunFlavor?,                               -- Hitstun Sub flavour (table below)
	debuffs: { StatusReq }?,                         -- added to the victim's Debuffs on outcome "hit" (Moves.statusAdd)
	pull: { stopStuds: number, frames: number }?,    -- victim pulled toward the attacker's root, stops stopStuds short
	meterDrain: number?,                             -- victim server meter −n (floor 0)
	healSelf: number?,                               -- attacker HP +n on outcome "hit"
	grab: { holdF: number, sub: number, offset: Vector3 }?, -- catch: victim → Grabbed (Sub = sub) for holdF + 2 f
	release: boolean?,                               -- this hit ends the victim's Grabbed, then applies hitstun/knockback
	reflectable: boolean?,                           -- carrier projectile hits only
	firstOfUse: boolean?,                            -- Part 3 fills: first landed hit of this use on this victim
}
```

Stun flavours map to `Hitstun` Sub codes (§4.9 item 8). All of them obey Stun Decay, OTG, Shake-Off and Breakaway like normal hitstun (GDD §6.14 #3):

| Flavour | `Hitstun` Sub | Motion while stunned | Render |
|---|---|---|---|
| `hit` | 0/1 (Part 3) | Part 3 push | normal reaction |
| `freeze` | 2 | `freeze` | ice block (`vfx.status.freeze`) |
| `float` | 3 | rise 4 studs over the first 12 f, then hold (`vector`) | levitate pose |
| `stumble` | 4 | Part 3 push along `KbV` | backward stumble |
| `possess` | 5 | pull (from `kit.pull`) | spectral tether |
| `flinch` | 6 | none | small flinch |

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
	name: string,                     -- "C_<rid>_<f>_<n>" (Part 3); also the hit-registry key prefix
	kind: "projectile" | "zone" | "summon",
	moveId: string, def: Types.MoveDef,  -- static data comes from the MoveDef (Styles.forConfig(cfg).move(moveId))
	own: number, f0: number, age: number, -- attributes Own, F0; age = f − F0 (0 on the spawn step)
	cf: CFrame, hits: number,          -- carrier CFrame; attribute Hits
	p: { [string]: number | string | boolean }, -- dynamic per-carrier attributes (≤ 10 keys, Part 3 limit)
	isServer: boolean, cfg: any,
	fighters: { FighterLite },        -- live fighters, ascending rid (server roster; client: local + replicated)
	ownerCf: CFrame?, ownerAim: Vector3?, ownerAlive: boolean,
	ray: (origin: Vector3, dir: Vector3) -> (Vector3?, Vector3?), -- static world (WorldQuery), excludes workspace.Sim
}
export type CarrierOut = {
	cf: CFrame?, set: { [string]: number | string | boolean }?,
	hits: { Fsm.HitRequest }?,        -- attacker = view.own; HitboxReq.prevCf for swept casts; ignored on the client
	dead: boolean?,                   -- Part 3 sets Dead = true in the sim; destroyed in the deferred flush (≥ 2 frames later)
	newOwner: number?,                -- reflectCarrier only: Part 3 rewrites Own
	ownerHeal: number?, events: { Fsm.SimEvent }?,
}
export type KitBehavior = {
	name: Types.Behavior,
	step: (bc: any) -> any, onContact: ((bc: any, result: any) -> any)?, onEnd: ((bc: any) -> any)?,
	validate: (def: Types.MoveDef) -> { string }, maxDamage: (def: Types.MoveDef) -> number,
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
| `Buff.hitN(bc, n: number, dmg: number, over: {[string]: any}?): Fsm.HitDef` | `Moves.hitDef(bc.def, bc.cfg)` cloned. `damage = dmg`. `guardDamage = def.guardDamage × dmg / maxDamage(def)`, rounded to 0.1. Fields in `over` are copied last |
| `Buff.req(key: string, box: Fsm.HitboxReq, hit: Fsm.HitDef, maxTargets: number): Fsm.HitRequest` | Constructor |
| `Buff.once(bc, bit: number): (boolean, number)` | `(MvA & bit) == 0`, plus the new `MvA` with the bit set (`bit32`) |
| `Buff.aimPoint(bc, range: number): Vector3` | `p = root + flat(bc.inp.aim).Unit × range` (fallback `rootCf.LookVector`). Then `bc.ray(p + (0, 20, 0), (0, −60, 0))`. Returns the hit position, or `p` if nothing is hit. Q-rule rounded to 0.01 |
| `Buff.statusPatch(s: Fsm.CharState, add: { StatusReq }, f: number): string` | `Moves.statusEncode` of `Moves.statusAdd` applied over `Moves.statusDecode(s.Buffs)`, with `untilF = f + frames` |
| `Buff.statusRemove(encoded: string, kind: string): string` | Decode, drop entries of `kind`, encode |

##### 4.2.0.4 Hits, keys and one-shots

- **Key format.** `"<moveId>#<n>"`. `n` is the hit index within the use (1-based). The echo, finish and tick hits continue the numbering. The same key is never reused within one use.
- **Hit per frame.** A behavior emits a hit's request on every active frame where it may connect. Part 3's registry guarantees each target is hit at most once per key.
- **Multi-hit timing.** `params.hitFrames[i]` is the active-relative frame (1-based) of hit `i`. Hit `i`'s request (key `#i`) is emitted on every frame with `hitFrames[i] ≤ e − S + 1 < hitFrames[i] + hitWindow` (`params.hitWindow`, default 3, td). Part 3's registry stops a second contact under the same key. No scratch state is needed, and a skipped frame still leaves 2 frames in the window (P4-R3).
- **First-of-use.** Part 3 fills `hit.kit.firstOfUse`. Passive bonuses that apply "once per use per target" read it (§4.3.3).
- **Blocked outcomes.** Statuses (`kit.debuffs`), pulls, meter drain and self-heal apply **only** on `Damage.Outcome` `"hit"` (and `"interrupt"`, which continues as a hit; Part 3 §3.12.6). On `"block"`, `"guardbreak"`, `"perfect"` and `"armor"` they do not apply (td).

##### 4.2.0.5 Carriers (projectiles, zones, summons)

Lifecycle. Part 3 implements spawning, stepping and destruction (Part 3 §3.11.5); Part 4 defines the content.

1. **Spawn.** A behavior returns `SpawnRequest{kind, moveId, origin, params}` from `step`. Part 3's `Adapter.spawn` clones the character's `ALG_Proto` **inside the sim**, so it is stitched on both peers for the owner. It sets `Name = C_<rid>_<f>_<n>` and the attributes `Own, Kind, MoveId, F0, Dead, Hits` plus `params`.
   - **Server-only spawns.** Summons and homing projectiles depend on other characters' positions, so their SpawnRequest carries `serverOnly = true` (§4.9 item 9). Part 3 takes its server-only creation path for them.
   - **Targetable summons.** For `hp > 0`, Part 3 also sets on the carrier: `CanQuery = true`, `CollisionGroup = "Hurtbox"` and `Size = params.hurtbox` (default 3×5×3, td). The carrier part **is** the summon's hurtbox (Part 3 §3.12.1 lists summon hurtboxes in the `Hurtbox` group).
2. **Static vs dynamic data.**
   - Static configuration (speeds, radii, ticks, telegraph shape) is **never** copied into attributes. `stepCarrier` reads it from `view.def.params` (the MoveDef found via the `MoveId` attribute), identically on both peers.
   - `params`/`set` hold only per-instance dynamic values, **≤ 10 keys** (Part 3 limit), names ≤ 12 alphanumeric characters.
   - Every carrier includes `lifeF` (total frames). Part 3 uses it for its own lifetime cap (`Chassis.Sim.MaxCarrierLifeFrames` = 600), so every kit carrier lives ≤ 600 f.
3. **Step.** `Behaviors.<Behavior>.stepCarrier(view, f)` runs in phase 1b:
   - on the server for every live carrier, in name order;
   - on the owner client for its predicted carriers (`Own == localRid`), where only `cf`/`set` apply and hits are ignored.

   The dispatch module is the MoveDef's behavior, with one exception: a Rush echo carrier (`Kind = "zone"`, `sub = 1`) is stepped by `Zone.stepCarrier`.
4. **Death.** `dead = true` makes Part 3 set `Dead = true` in the sim. The server destroys the carrier in the deferred flush, ≥ 2 frames later, **outside** the sim (Part 3 §3.1.5).
5. **Caps.** `Kits.MaxCarriersPerFighter` evicts the oldest owned carrier; `Kits.MaxCarriersServer` refuses the spawn. Server log via the flush queue: `[ALG][Style] carrier cap owner=<rid> evicted=<name>`.
6. **Owner KO or untrack.** Part 3 marks every carrier with that `Own` as `Dead` on the next server step (td). This prevents posthumous damage.
7. **Hit bookkeeping.**
   - Carrier hit requests use keys `<view.name>#<suffix>`; Part 3 keeps carrier registries with the carrier (Part 3 §3.12.5).
   - Zone ticks use one suffix per tick (`#t<j>`), so a target can be hit once per tick.
   - Per-target caps (`maxHitsPerTarget`) are counted in **server memory** keyed by carrier name and cleared on death. This is legal per Part 3 §3.1.8: carrier hits are S-class consequences.

Rendering: the client renders a carrier from its `Kind`, `MoveId` (→ `def.vfx` or `params.model`) and dynamic attributes. Telegraphs come from §4.2.0.6.

##### 4.2.0.6 Telegraph emission

Telegraphs come from two sources. Both render on the dedicated telegraph layer in the user's telegraph colour, at 4.5:1 outline contrast, and are never reduced by Reduced VFX (GDD §13.4, §16.3; Part 6).

| Source | When | Anchor |
|---|---|---|
| `MoveDef.telegraph` (`params.teleAnchor` absent or `"self"`) | From move frame `S − leadFrames + 1` to the first active frame | The owner's root: the hitbox centre if `hitbox` is set, else `params.teleFwd` studs ahead (default `size.z/2`). It follows the facing during startup (Part 3 §3.2.7) |
| `params.teleAnchor = "carrier"` | From carrier spawn while `age < armF` (zones) or `age < flightF` (lobs). Summon attacks show while `age − a0 < attack.startupF` | Carrier `cf` (lobs: landing point `lx,ly,lz`). Shape and size come from `MoveDef.telegraph` (summon attacks: an `Arc` sized from `attack.size`) |

`tellFrames(def)` per behavior is the number of frames between the earliest visible tell and the first frame that can damage:
- self-anchored moves (Rush, Grab, Strike, beams, bullets, waves, homing, instant self zones): `startup`;
- `self`/`selfFollow` carrier zones: `startup + armF` (the self-anchored MoveDef telegraph shows during startup, then the carrier's);
- `aim` zones: `armF` (the point is only known at spawn);
- lobs: `flightF`;
- chase and marked summons: `attack.startupF`;
- line summons: `startup`.

K08 uses it (§4.1.4).

##### 4.2.0.7 Presentation hooks

- **Move start.** The client RenderController (Part 6) sees `St = Move` with `Sub` (Part 3 §3.2.1). It plays `def.anim` (`Assets.get`) and fires the `def.vfx` and `def.sfx` timing from the clip's `VFX`/`SFX` markers. A RenderStepped frame counter on `e` is the fallback when the clip is unavailable.
- **Hit and carrier effects** are keyed on attributes: `Cnt`, victim `St`/`Sub`, carrier `vis`/`tS`.
- **Undoable.** Effects started for predicted transitions follow Part 3 §3.1.7 (undo within 15 f).

##### 4.2.0.8 Interaction rules (all behaviors)

| Rule | Implementation |
|---|---|
| `superArmor` | Implemented by Part 3 (`Damage.resolve` row 8, Part 3 §3.12.6): during `params.superArmorFrames` (default startup + active), hits deal damage and meter only, with no stun or knockback. No VS move uses it |
| `projectileArmor` | Part 3 row 8: during startup, only vs `HitDef.kind == "projectile"`. Used by `hero.4` |
| Parry | `parryable = true` → Perfect Block negates the hit (Part 3 §3.6.3). Grabs are parryable only when flagged |
| Block chip | Blockable damaging moves: `chip = 0.2` (GDD §6.4). Multi-hit chip is per hit. Zone ticks are unblockable unless `tick.blockable` |
| Reflect | Only carrier projectiles with `refl = true` (§4.2.5). Beams, arms, waves and whips are not reflectable |
| Grabs | Ignore block and Counter stances; miss i-frames, airborne victims (`groundOnly = true`) and Perfect Block if parryable (GDD §6.4) |
| Summons | Take 2× M1 damage (GDD §6.7). `Summon.hurtCarrier` multiplies M1-family hits (`m1`, `ender`, `airm1`, `dashm1`) by `cfg.Combat.Damage.Summon.M1DamageMult` (2) |
| Statuses | Applied only on outcome `"hit"` (§4.2.0.4) |

#### 4.2.1 Strike (Core default, no module)

`behavior = "Strike"` uses Part 3's plain-melee path: `Moves.hitbox(def, ctx)` and `Moves.hitDef(def, cfg)` every active frame, each target once per use (Part 3 §3.4.2).

Part 4 rules for Strike moves:
- `hitbox` is required (K20).
- `params` may contain only Part 3-recognized keys: `blockstun`, `ragdollFrames`, `hangFrames`, `iFrames`, `air`, `cancelInto`, `cancelWindow`, `cancelChainIndex`, `wallSplat`, `cinematicFrames`.
- In the vertical slice, Strike is used only by `stretch.a1` (Stomp).

**Verify:** Lune `behaviors.strike_registry_nil` (`Styles.behavior("Strike") == nil`); scenario `kit.stretch.smoke` covers a1.

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
	echo: { delayF: number, damage: number, hitstun: number, knockback: Types.KnockbackDef?, ragdollFrames: number?, width: number, length: number }?,
	selfBounce: { studs: number, height: number, frames: number }?,
	teleFwd: number?,
	-- Part 3-recognized: wallSplat, ragdollFrames, hangFrames, blockstun, cinematicFrames
}
```

Per frame (`step`):

| Phase | Output |
|---|---|
| Travel window (resolved `from ≤ e+1 ≤ to`, skipped once `MvA` bit 1 "stopped" is set) | `ground`: `motion = planar` at `v = studs·60/N` along the facing, where `N = to − from + 1`. `leap`: `motion = vector` with horizontal `v` and `vy(k) = 4·height·60/N·(1 − 2k/N)`, `k = e+1−from` (symmetric arc whose summed rise is `height`, td) |
| Active (`S ≤ e < S+A`) | `hits`: single-hit moves emit `Buff.req(id.."#1", Moves.hitbox(def, ctx), Buff.hitN(bc, 1, def.damage), 4)`. Multi-hit moves follow §4.2.0.4; the last hit applies `lastHit` overrides. `coneDeg` is copied into the HitboxReq |
| Recovery | `selfBounce` (if `MvA` bit 2 "connected"): `vector` back along −facing, `studs·60/frames` horizontal plus a `height` arc over `frames` frames. Otherwise `motion = none` |

`onContact`: sets `MvA` bit 2 (connected) and, if `stopOnHit`, bit 1 (stopped).

Echo (`params.echo`, GDD §7.3.1 A3 "leaves an echo slash 0.5 s later"). On the first active frame (once-bit 4), `step` returns a **SpawnRequest**:
- `kind = "zone"`, stepped by `Zone.stepCarrier`;
- `origin` = the yaw-only root CFrame at the dash start, moved forward by `echo.length/2`;
- dynamic params `{lifeF = echo.delayF + 1, sub = 1}`.

`Zone.stepCarrier` recognizes `sub = 1`. It reads `view.def.params.echo` and, at `age ≥ echo.delayF` (once), emits a `Box (echo.width, 6, echo.length)` HitRequest with key `<name>#e`, `damage = echo.damage`, `hitstun = echo.hitstun`, `knockback = echo.knockback`, `kind = "zone"`, blockable. Then `dead`. The echo lands even after the move ends and whether or not the dash connected.

Other functions:
- `validate`: `travel.studs` in (0, 60]; `hits`/`hitFrames` same length; every `hitFrames[i] ≤ active`; `echo.delayF ≥ 8` when `echo.damage > 0`.
- `maxDamage`: `damage + (echo and echo.damage or 0)`.
- `tellFrames = startup`. `blockable = true`.

Edge cases:
- **Hitstop.** Hitstop shifts `StF` (Part 3 §3.5.6). The travel window is computed on `e`, so travel pauses during hitstop.
- **Walls.** If travel hits a wall, Part 3's mover simply stops. No special handling.
- **Interrupted move.** `onEnd` emits nothing. Carriers already spawned (echo) persist.

**Verify:** Lune `behaviors.rush_travel_distance` (Σ planar displacement over the window = `studs` ± 0.05); `behaviors.rush_leap_apex` (Σ vy·dt = 0 ± 0.05, apex = height ± 0.1); `behaviors.rush_multi_hit_sum` (hero.2 emits 6 keys with Σ damage 11); `behaviors.rush_echo_spawns_once`.

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
	reflectable: boolean,
	onHit: KitHitExt?,                                -- merged into every hit's HitDef.kit
	extender: boolean?, setPiece: string?, decayStacks: number?, teleAnchor: ("self" | "carrier")?,
}
```

| Kind | Owner `step` | Carrier (`stepCarrier`) | Hit |
|---|---|---|---|
| `beam` | Active frames: a `Cast` HitboxReq from the root along the facing. Frame `i` of `extendF` sweeps from tip `(i−1)/extendF·range` to `i/extendF·range` (instant when `extendF = 0`). Width `width`. One key `#1`, `maxTargets` = 1 (td: beams stop at the first body) | — | `kind = "projectile"`, `melee = false`, not reflectable |
| `bullet` | First active frame (once-bit 1): one SpawnRequest per shot. Shots fire at active frames `1, 1+everyF, …` (`count` shots, once-bits 1..count). `origin` = root yaw frame × (0, 0, −2) | Each frame: `to = cf + dir·speed/60`. World check `view.ray(cf.Position, to−cf.Position)` → `dead` at the wall. Emit a `Cast` HitboxReq `prevCf = cf`, `cf = to`, size `(2r, 2r, 1)`, `maxTargets = 1 + max(pierce, 0)` (all if −1). `dead` when travelled ≥ `range` or the pierce budget is used | Key `#<shot>`, `kind = "projectile"`, `kit.reflectable = refl` |
| `lob` | Shots as for bullets. The landing point is `Buff.aimPoint(bc, aimRange)` plus `pattern[shot]` rotated by the facing yaw, sent as `lx, ly, lz` | Position = parabola from origin to landing over `flightF` (apex 8 studs, td). At `age ≥ flightF` (once): `Sphere` HitboxReq radius `impactRadius` at the landing point, then `dead`. Render draws a Circle telegraph (diameter `2·impactRadius`) at `lx,ly,lz` while `age < flightF` | Key `#<shot>`, `kind = "projectile"`, not reflectable |
| `wave` | First active frame: one SpawnRequest | Moves at `speed` along `dir`, ground-snapped via `view.ray` down. Each frame a `Box` HitboxReq of size `(w(age), height, speed/60 + 1)` with `w` linear from `width` to `widthEnd or width` over `range`. Every target once (`maxTargets = 20`). `dead` at `range` | Key `#1` (each target once per use) |
| `homing` | As bullets (count/everyF), with `serverOnly = true` (steering depends on other characters) | Like bullet. Each frame `dir` rotates toward the nearest enemy (`view.fighters`, lowest angle within ±90°, ties by rid) by at most `homingDegPerSec/60` degrees. Q-rule on `dir` | Key `#<shot>`, `kind = "projectile"` |

Dynamic carrier params (≤ 10; everything else is read from `view.def.params`):
- bullet/homing: `lifeF`, `dx, dy, dz` (unit direction, Q-rule 0.01), `sh` (shot index), `trv` (studs travelled), `rf` (1 = reflected), `o0` (original owner rid);
- lob: `lifeF`, `ox, oy, oz` (launch point), `lx, ly, lz` (landing point), `sh`;
- wave: `lifeF`, `dx, dz`, `trv`.

`lifeF` = `ceil(range·60/speed) + 1` for bullets, homing and waves, and `flightF + 1` for lobs.

Reflect: `reflectCarrier(view, newOwner)` returns:
- `newOwner`;
- `set {dx = −dx, dy = −dy, dz = −dz, rf = 1, trv = 0}`;
- the carrier hit set is cleared, and the original owner is recorded in `o0`.

A carrier with `rf = 1` cannot be reflected again; a second reflecting stance absorbs it (td, no ping-pong). When a reflected carrier hits its `o0` rid, Part 3 passes `reflectedHome = true` in `HitInfo` (§4.3.1). That is `stretch`'s Style Moment.

Other functions:
- `validate`:
  - `range > 0`;
  - beam needs `width`; bullet/homing need `speed` and `radius`; lob needs `flightF ≥ 8`, `impactRadius` and `#pattern == count`; wave needs `speed`, `width`, `height`;
  - `Σ hits == damage` when `hits` is set.
- `maxDamage`: `Σ hits` or `damage`.
- `tellFrames`: `flightF` for lob, else `startup`.
- `blockable = true` for every kind.

Edge cases:
- A bullet spawned inside a wall is `dead` on its first step.
- A bullet whose owner was KO'd is removed with the owner's other carriers (§4.2.0.5).
- A beam fired during hitstop is paused by Part 3.
- A homing projectile with no enemy in the ±90° cone flies straight.

**Verify:** Lune `behaviors.projectile_beam_sweep` (stretch.1: `extendF` segments cover 0–25 studs without gaps); `behaviors.projectile_bullet_range` (`dead` after `range/speed·60` ± 1 frames); `behaviors.projectile_lob_lands_on_spot` (landing within 0.05 studs of `lx,ly,lz` at age `flightF`); `behaviors.projectile_reflect_once`; `behaviors.homing_turn_rate` (never more than `hdg/60` degrees per frame).

#### 4.2.4 Grab (`Behaviors/Grab.luau`)

```lua
export type GrabTick = { at: number, damage: number, healSelf: number? }   -- at: hold-relative frame (1-based)
export type GrabParams = {
	groundOnly: boolean,              -- Part 3-recognized param: misses airborne victims (GDD §6.4)
	holdF: number, offset: Vector3, sub: number,  -- Grabbed Sub code (table below)
	ticks: { GrabTick },              -- Σ damage == def.damage; the last tick is the finish
	finish: "throw" | "slam" | "release" | "drop",
	throwDir: ("facing" | "input")?, finishKb: Types.KnockbackDef?, finishHitstun: number?,
	finishRecovery: number,           -- attacker frames after the finish tick
	sweep: { studs: number, width: number, height: number }?,  -- moving catch box over the active window
	maxTargets: number?,              -- default 1
	carry: { riseStuds: number, riseF: number }?,
	cinematicFrames: number?, decayStacks: number?,
}
```

`Grabbed` Sub codes (Part 3 table: "grab kind (Part 4)"):

| Sub | Kind | Used by |
|---|---|---|
| 1 | hold | (generic) |
| 2 | lift | `hero.a2`, `stretch.a2` |
| 3 | coil | `stretch.4` |
| 4 | drain | `rocker.3` |
| 5 | carry | `rocker.a4` |
| 6 | thrall | `decay.2` |

Flow:

| Phase | Output |
|---|---|
| Startup | Nothing (`motion = none`; Part 3 walk speed 0) |
| Active, `MvA == 0` (not caught) | Catch request: `Moves.hitbox(def)` (or the sweep box moving `studs·60/A` per frame along the facing). `hit = Buff.hitN(bc, 0, 0, {kind = "grab", groundOnly = ground, kit = {grab = {holdF, sub, offset}}})`, key `"#c"`, `maxTargets = maxTargets or 1`. A sweep also outputs `motion = planar` along the facing |
| `onContact` (outcome `"hit"`, Part 3 attached `ALG_Hold`) | `patch {MvA = MvA + 1, MvB = e, StDur = S + A + holdF + finishRecovery}`. The move is extended to cover the hold. With a sweep, catching continues until the active window ends |
| Hold (`MvA ≥ 1`; `MvB = catchE`, `k = e − catchE`) | For each tick `i` with `ticks[i].at ≤ k + 1 < ticks[i].at + 3` (a 3-frame window, td; Part 3's registry dedupes per key): `hits = {Buff.req(id.."#"..i, holdBox, hit, maxTargets)}` with `heldOnly = true`. `holdBox` is a Sphere of radius 3 at `rootCf × offset`. The hit has `damage = tick.damage`, `kind = "grab"`, `kit.healSelf = tick.healSelf`, and for the last tick `kit.release = true` plus the finish knockback/hitstun. `carry` → `motion = vector (0, riseStuds·60/riseF, 0)` while `k < riseF`, then `vector 0`. Hold `motion = freeze` otherwise |
| Finish direction | `throwDir = "input"`: knockback direction = `bc.inp.moveDir` (else facing); `"facing"`: facing; `slam`/`drop`: `finishKb` with `angleDeg = 0` and ragdoll |
| Whiff (active ends with `MvA == 0`) | Normal recovery (`def.recovery`) |
| `onEnd` | Nothing. Part 3 detaches `ALG_Hold` on any exit of the holder's Move state (§4.9 item 10) |

Breakaway, Shake-Off or KO of the victim: Part 3 detaches the hold. The holder's Breakaway burst puts the holder into `Stagger`, which ends its Move (Part 3 §3.9.1), and `onEnd` runs. If the holder is still in Move with no held victims, the remaining ticks with `heldOnly = true` find no target and whiff harmlessly; the move ends at its `StDur`.

`cinematicFrames` (Ultimate grabs only): every hold hit sets `HitDef.cinematicLockF = cinematicFrames` (Part 3 caps it at 120 f, GDD §6.8).

Other functions:
- `validate`: `Σ ticks.damage == damage`; ticks sorted by `at` with `at ≤ holdF`; `finishRecovery ≥ 8`; `maxTargets ≤ 3`.
- `maxDamage = damage`. `tellFrames = startup`.
- `blockable = false` (grabs ignore block, GDD §6.4), hence `guardDamage = 0` and `chip = 0`.

**Verify:** Lune `behaviors.grab_ticks_sum` (stretch.4 emits 4 hold keys whose damage sums to 9); `behaviors.grab_whiff_recovery` (no contact → move length = S+A+R); `behaviors.grab_hold_extends_stdur`. Scenarios: `kit.grab_vs_jump` (`decay.2` vs a jumping dummy → whiff), `kit.grab_breakaway` (dummy with pips Breakaways at hold frame 15 → holder `St = Stagger`, no further ticks).

#### 4.2.5 Counter (`Behaviors/Counter.luau`)

```lua
export type CounterParams = {
	arcDeg: number,                   -- front arc that counts (180 = front half, Defense.isFront)
	vsMelee: "stagger" | "launch",
	staggerF: number?,                -- stagger reaction frames (Part 3 Stagger Sub 3 "melee bounce")
	counterKb: Types.KnockbackDef?, counterHitstun: number?, hangFrames: number?,
	reflectProjectiles: boolean,
	endOnTrigger: boolean, triggerRecovery: number?,
}
```

`step`: startup and recovery output `motion = none`. Active is the stance, `motion = freeze`. The behavior emits no hits of its own.

`intercept(bc, hit, srcPos)` (server; the victim is in `Move` with this def):

| Condition | Verdict |
|---|---|
| Not in active frames, or `srcPos` outside the front arc | `pass` |
| `hit.kind == "grab"` | `pass` (grabs beat counters, GDD §6.7) |
| `hit.kit.reflectable == true` and `reflectProjectiles` | `reflect`. Part 3 calls the carrier's `reflectCarrier(view, defenderRid)` and the hit is negated |
| `hit.kind == "projectile"` (beam, not reflectable) | `absorb` if `reflectProjectiles` (the wall blocks it), else `pass` |
| `hit.melee == true` | `absorb` with a reaction. `stagger`: `HitDef {damage = 0, kit.stun = "hit", hitstun = staggerF}`, which Part 3 maps to `Stagger` Sub 3 for `staggerF`. `launch`: `HitDef {damage = def.damage, knockback = counterKb, hitstun = counterHitstun, kind = "move", melee = true, unblockable = true}` |
| Zone ticks, AoE bursts (`melee == false`, not projectile) | `pass` |

If `endOnTrigger` is set and the verdict is not `pass`, `patch = {StDur = e + 1 + triggerRecovery}`. Every intercept returns `patch.MvA = MvA + 1` (trigger count), which render uses for the "boing" flash.

Other functions:
- `validate`: `active ≥ 12`; `vsMelee = "launch"` requires `counterKb`.
- `maxDamage = damage` (launch counters) or 0. `tellFrames = startup`.
- `blockable = false` (the reaction hits an attacker mid-swing; td).

**Verify:** Lune `behaviors.counter_reflects_projectile`, `behaviors.counter_staggers_melee` (reaction `hitstun == 18`), `behaviors.counter_loses_to_grab`, `behaviors.counter_back_passes`. Scenario `kit.stretch.shield_reflect` (monarch bot Gum Blaster into Stretchy Shield → the projectile hits the bot; `style_moment` event for stretch).

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
	stay: StatusReq?,                  -- debuff refreshed every Kits.StatusStayRefreshFrames on enemies inside
	ownerTick: { everyF: number, heal: number }?,
	maxHitsPerTarget: number?, pullStudsPerSec: number?, followMove: boolean?,
	dmgModel: ("total" | "perHit")?, cinematicFrames: number?, teleAnchor: ("self" | "carrier")?, hangFrames: number?,
}
```

**Instant self zone** (`anchor = "self"`, `lifeF = 0`, every burst `atF = 0`): no carrier. The owner `step` emits the burst on the active frames as a HitRequest. This gives the owner BQ-predicted contact (Part 3 §3.1.2). Shape:
- circle: `Sphere` with diameter `2·radius`;
- cone: `Sphere` with diameter `2·radius` plus `coneDeg`;
- line: `Box (width, 6, radius)` ahead.

Used by `rocker.4` and `monarch.a3`.

**Carrier zone** (every other case). On the first active frame (once-bit 1), return a SpawnRequest:
- `origin` = `Buff.aimPoint(bc, aimRange)` (aim), or the yaw-only root (self/selfFollow);
- dynamic params (≤ 10): `lifeF = armF + max(lifeF, last burst atF) + 1`, `st` (bitmask of fired bursts), `tk` (ticks done).
- Everything else (shape, radius, ticks, stay, owner tick, pull, caps) is read from `view.def.params`.
- `aim` zones use the deterministic `Buff.aimPoint` (static-world ray), so they can be predicted on the owner client.

`stepCarrier(view, f)` (server; owner client for predicted zones, hits ignored), with `k = age − armF`:

| Condition | Output |
|---|---|
| `selfFollow` | `cf = yaw(ownerCf)` every frame; `dead` if `ownerAlive == false` |
| `k < 0` | Nothing (arming; the telegraph shows) |
| Burst `b` with `k ≥ b.atF`, not yet fired (bit `b` in `st`) | HitRequest key `<name>#b<b>` with the shape box, `damage = b.damage`, `hitstun`, `kit.stun`, `knockback`, `kind = "zone"`, `unblockable = not b.blockable` (default blockable, zone centre as source) |
| Tick: `k ≥ 0`, `k % tick.everyF == 0`, `tk < count` | HitRequest key `<name>#t<j>` (j = `tk + 1`; `set.tk = j`), `damage = tick.damage`, `tick = true` (Part 3: no hitstun, hitstop or combo count), `unblockable = not tick.blockable`. Per-target cap `maxHitsPerTarget` enforced by server memory |
| Stay: `k ≥ 0`, `k % Kits.StatusStayRefreshFrames == 0` | Zero-damage `tick` HitRequest key `<name>#s<k>` with `kit.debuffs = {stay}` |
| Owner tick: `k % ownerTick.everyF == 0` and the owner is inside | `ownerHeal = ownerTick.heal` (routed through `Adapter.heal`, so realm heal multipliers apply, Part 3 §3.9.3) |
| Pull (`params.pullStudsPerSec`) | Not a hit. Part 3 applies it as an S-class planar push toward the carrier centre on enemies inside (§4.9 item 11) |
| `k ≥ lifeF` and all bursts fired | `dead = true` |

The periodic `k % N == 0` tests are the one allowed modulo use (Part 3 §3.8.7): a skipped frame skips a tick identically on both peers.

`followMove` (`stretch.a4`): during the owner's active frames, the owner `step` outputs `motion = planar` from `bc.inp.moveDir × WalkSpeed × 0.6` (td) instead of `none`.

Other functions:
- `validate`: `radius > 0`; `armF ≥ 0`; tick zones need `count` or `maxHitsPerTarget`; `cinematicFrames ≤ 120`.
- `maxDamage`: `Σ burst.damage + min(tick.damage × (count or ∞), tick.damage × maxHitsPerTarget)`, capped by `damage`. For `perHit` it is `tick.damage × maxHitsPerTarget`.
- `tellFrames`: `armF` for `aim` zones; `startup + armF` for `self`/`selfFollow` zones, instant or carrier (§4.2.0.6).
- `blockable`: any burst or tick with `blockable ~= false`.

Edge cases:
- An aim zone over a pit snaps to the first ground within 60 studs below, else spawns at aim height.
- Two zones from the same owner apply independent caps.
- Leaving a zone ends `stay` effects within `StatusStayRefreshFrames` + the status duration.

**Verify:** Lune `behaviors.zone_tick_cap` (Chemistry Flask on a stationary target for 10 s → exactly 5 ticks, 10 damage); `behaviors.zone_burst_once`; `behaviors.zone_selfFollow_tracks_owner`; `behaviors.zone_owner_heal`. Scenario `kit.decay.well_heal` (decay at 80 HP inside its own pool for 3 s → +3 HP ± 0, Part 3 regen disabled).

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
	speed: number, ai: "chase" | "line" | "marked", leash: number?,
	attack: SummonAttack?, line: { studs: number, frames: number, width: number }?,
	maxHitsPerTarget: number, once: boolean?, model: string,
}
```

`step`: on the first active frame (once-bit 1), one SpawnRequest per body, with `serverOnly = true`. `origin` = yaw-root × `spawnOffsets[i]`.
- Dynamic params (≤ 10): `lifeF`, `hp` (0 = untargetable), `bi` (body index), `st` (0 idle, 1 attacking), `a0` (attack start age), `nx` (next attack age), `tg` (target rid), `n` (attacks done), `dx, dz` (line direction).
- Static data (`speed`, `attack`, `line`, `leash`, `model`) is read from `view.def.params`.
- The summon-arrival tell is the MoveDef `Circle` telegraph drawn at each spawn offset during startup.

`Summon.stepCarrier(view, f)` (server only; `view.fighters` sorted by rid):

| AI | Behavior per frame |
|---|---|
| `chase` | Target = nearest enemy within `leash` of the owner (ties by rid). If its distance is > `aR`, move toward it at `spd/60` studs per frame (ground-snapped by `view.ray`), facing it. Otherwise, if not attacking and `age ≥ nextAttackAge`, start an attack: `st = 1`, `a0 = age` (render draws an `Arc` telegraph sized from `attack.size` while `age − a0 < attack.startupF` when `attack.tele`). At `age − a0 ≥ attack.startupF` (once per attack): a `Box` HitRequest of `size` ahead, key `#<attack#>`, `damage = def.damage`, `hitstun = def.hitstun`, `kit.stun = attack.stun`, `kind = "move"`, `melee = true`. Then `nx = age + attack.everyF`; with `once`, `dead` after the attack. Lost target → idle |
| `line` | On spawn, record `dir` = owner aim (flat). For `age < lF`: `cf += dir·lS/lF`; Box HitRequest `(lW, 6, lS/lF + 2)` swept with `prevCf`, key `#1`, every target once. Afterwards idle (blocks space, can be killed) |
| `marked` | Untargetable set piece. At `age ≥ a0` and every `aE` frames: the target is the enemy closest to the owner's aim ray (`ownerAim`) within `aBr` studs and ±30°. The telegraph is a Line from the carrier to the target's position, locked at telegraph start, for `aS` frames. Then a Cast HitRequest along the locked line (`aBw` wide), key `#<shot>`, `kind = "projectile"`, not reflectable |

`hurtCarrier(view, hit, attackerRid)` (targetable summons only):
- `dmg = hit.damage × (if M1 family then cfg.Combat.Damage.Summon.M1DamageMult else 1)` (2, GDD §6.7 "summons take 2× M1 damage");
- `set {hp = hp − dmg}`; `dead` when `hp ≤ 0`;
- event `{k = "summon_down", rid = ownerRid}` for render.

Summons do not block, take no hitstun and are not grabbable (td).

Caps: one cast's bodies form a group. Recasting the same move despawns that owner's previous group (td).

Other functions:
- `validate`: K09 limits; `#spawnOffsets == bodies`; `attack.startupF ≥ 8` when the per-hit damage > 10; `line` requires `ai = "line"`.
- `maxDamage`: `perHit` → `damage × maxHitsPerTarget × bodies`; `total` → `damage`.
- `tellFrames`: `attack.startupF` (chase/marked) or `startup` (line).
- `blockable = true`.

**Verify:** Lune `behaviors.summon_limits` (K09 on all summon moves); `behaviors.summon_hits_cap` (monarch.2 vs a stationary target for 8 s → ≤ 18 damage); `behaviors.summon_m1_double` (a 3.5 M1 hit → hp −7); `behaviors.summon_once_despawns`.

#### 4.2.8 Buff / Debuff (`Behaviors/Buff.luau`)

```lua
export type ModeEffect = { move: string, startupDelta: number?, guardDamageMult: number? }
export type BuffParams = {
	target: "self" | "front",
	range: number?, width: number?,                    -- front: zero-damage request box
	buffs: { StatusReq }?,                             -- added to the owner's Buffs (patch)
	debuffs: { StatusReq }?,                           -- front: applied to the first enemy hit
	toggleMode: { count: number, spawn: number, effects: { ModeEffect } }?, -- cycles the owner's Kit
	analyze: { frames: number, range: number, coneDeg: number }?,            -- presentation only
	random: boolean?,
}
```

`step`, on the first active frame (once-bit 1):
- `buffs`: `patch.Buffs = Buff.statusPatch(s, buffs, f)`.
- `toggleMode`: `patch.Kit = (s.Kit + 1) % count`.
- `target = "front"`: HitRequest key `#1`, a `Box (width, 6, range)` ahead, `damage = 0`, `tick = true`, `unblockable = true`, `kit.debuffs = debuffs`, `maxTargets = 1`.
- `analyze`: no output. RenderController (Part 6) shows the analyze panel for `frames` on the owner's client: the target is the nearest enemy within `range` and `coneDeg` of the facing at the first active frame. It shows the target's `Cd1..Cd4/CdQ` (HRP) and `Meter` (Player attribute, Part 3 §3.3.3).

Other functions:
- `validate`: `toggleMode.count ≥ 2`; every `effects[i].move` exists in the same style; status kinds exist in `Moves.STATUS`.
- `maxDamage = 0`; `tellFrames = startup`; `blockable = false`.

Mode variants. `Styles.variants(def)` precomputes, for every `ModeEffect` target move, an array indexed by `Kit + 1`:
- the variant for mode `m` = the base def with `startup += startupDelta` or `guardDamage = round(guardDamage × guardDamageMult)` from `effects[m + 1]`;
- modes with no effect for that move use the base def.

Part 3's `Moves.defFor` returns `StyleRuntime.variants[id][s.Kit + 1]` when present (§4.9 item 12). Variants are validated like base moves (K06–K16).

**Verify:** Lune `behaviors.buff_toggle_cycles` (hero.q: Kit 0→1→0); `styles.variants_hero1` (mode 0: startup 11; mode 1: guardDamage 33); `behaviors.buff_front_debuff` (decay.q emits a 0-damage request with `wh`).

#### 4.2.9 Mobility (`Behaviors/Mobility.luau`)

```lua
export type MobilityParams = {
	mode: "dash" | "grapple" | "slide" | "flight",
	studs: number?, maxRange: number?, speed: number?, maxTravelF: number?,
	whiffRecoveryF: number?, whiffCooldownSec: number?,
	pitchClampDeg: number?, trail: boolean?,
	-- Part 3-recognized: air (usable airborne), iFrames = {from, to} (1-based move frames)
}
```

| Mode | Behavior |
|---|---|
| `dash` / `slide` | Active frames: `motion = planar` along the facing at `studs·60/A`. `slide` sets `vis` trail (render) |
| `flight` | First active frame (once-bit 1): store pitch `MvB = round3(clamp(asin(aim.Y), ±pitchClampDeg))`. Each active frame: `motion = vector` along the yaw/pitch direction at `studs·60/A` |
| `grapple` | First active frame (once-bit 1): `hit, n = bc.ray(root, aim.Unit × maxRange)`. If hit: `travelF = min(maxTravelF, ceil(dist·60/speed))`; `patch {MvA = 1 + 256 × travelF, MvB = round3(pitch), StDur = S + 1 + travelF + recovery}`. Yaw is the held facing. During travel (`e − S < travelF`): `motion = vector` along (yaw, pitch) at `speed`. On miss: `patch {StDur = S + 1 + whiffRecoveryF, CdQ = f + toFrames(whiffCooldownSec)}` |

`iFrames` are honoured by Part 3 (`Fsm.isInvulnerable`, Part 3 §3.2.1).

Other functions:
- `validate`: `studs > 0` for dash/slide/flight; grapple needs `maxRange`, `speed`, `maxTravelF`, `whiffRecoveryF`; `iFrames.to ≤ total frames`.
- `maxDamage = 0`; `tellFrames = startup`; `blockable = false`.

**Verify:** Lune `behaviors.mobility_flight_distance` (rocker.q: Σ displacement 25 ± 0.1); `behaviors.mobility_grapple_hit` (fake ray hits at 30 studs → travel 15 f at 120 st/s); `behaviors.mobility_grapple_whiff` (`CdQ = f + 120`). Scenario `kit.rocker.batshift_iframes` (dummy `m1_spam` hits during move frames 1–6 → whiff; frame 7+ → hit).

#### 4.2.10 RealmCast (`Behaviors/RealmCast.luau`)

Part 3 runs the whole Realm lifecycle (Part 3 §3.9.3):
- the `RealmCast` state (90 f);
- the interrupt window (e 0–44) and hyper armor (45–89);
- opening at `e ≥ 90`, membership `RlmId`, queueing (max 2), close at 12 s;
- Stubborn Heart;
- `Cd4 = AwkEndF` (once per Awakening).

The Move state is not used, so `step` is never called for RealmCast. This module provides:
- `validate(def)`:
  - `startup == cfg.Combat.Realm.CastFrames` (90, GDD §6.10), `active == 1`, `recovery == 0`, `cooldown == 0`, `damage == 0`;
  - `Realm.rules[def.id] ~= nil`;
  - telegraph `Circle` with `size.X == 2 · cfg.Combat.Realm.RadiusStuds` (80) and `leadFrames == 90`.

  Key names `CastFrames`/`RadiusStuds` are Part 3's (§3.17).
- `maxDamage = 0`, `tellFrames = 90`, `blockable = false`, `step = function() return nil end`.

The rule content lives in `Core/Realm.luau` (§4.4.4).

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
	--   cfg = Defaults.Combat, isServer = true, inp = inp or { bits = 0, pressed = 0, aim = Vector3.new(0, 0, -1), moveDir = Vector3.zero },
	--   ray = <plane at groundY + vertical walls> }
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
	f: number, rid: number, styleId: string, style: Fsm.StyleRuntime, s: Fsm.CharState, cfg: any, isServer: boolean,
	moves: (moveId: string) -> Types.MoveDef?,                  -- Styles.forConfig(cfg).move (overrides applied)
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
	heal: number?,                     -- server: owner HP += heal (via Adapter.heal, realm multipliers apply)
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
	onAwaken: ((h: HookCtx) -> HookOut?)?, onAwakenEnd: ((h: HookCtx) -> HookOut?)?, -- both peers on the transition step
	onWallSplat: ((h: HookCtx, victimRid: number) -> HookOut?)?, -- server (attacker)
}
```

How Part 3 calls them (§4.9 item 13):

| Call | Where (Part 3) | Peers |
|---|---|---|
| `Styles.passiveStep(h, inp)` | After `Fsm.step`, for every character this peer simulates | Both. Outputs limited to B-class fields and motion |
| `Styles.serverHook("use", h, {moveId…})` | On `Move`/`RealmCast` entry | Server. Feeds the moment tracker only |
| `Styles.serverHook("hitDealt" / "hitTaken" / …, h, info)` | In the server consequence phase of the same step, after `Damage.resolve` | Server. Outputs are S-class |
| `Styles.serverHook("awaken"/"awakenEnd", h)` | On the transition | Both (B-class outputs only) |

`Styles.serverHook` calls the style's hook, then runs the moment tracker (§4.3.5), and merges the outputs: heals add, cooldown reductions add, events concatenate.

Determinism: hooks never read server memory except through the moment tracker and the Scientific Method table (§4.3.6), which feed S-class outputs only (Part 3 §3.1.8).

#### 4.3.2 Status registry content (`Moves.STATUS` in `Core/Moves.luau`)

Part 3 owns the codec. Part 4 owns these entries (Part 3 §3.4.2). Each entry is `{ name, list, maxStacks, mods(n) → partial StatusMods, flags }`. Two flags and one field are Part 4 extensions (§4.9 item 14):
- `consumeOn: "hit" | "block" | "attack" | nil`: removed when the owner lands a hit, raises Block, or enters any ATTACK state.
- `incomingFrom: { style, perStack, max }`: a victim debuff that adds damage only to hits from that attacker style.
- `StatusMods.m1Chain`: overrides the M1 chain length.

| Kind | Name | List | maxStacks | `mods(n)` | Flags | Source (GDD) |
|---|---|---|---|---|---|---|
| `rt` | root | Debuffs | 1 | `rooted = true` | — | monarch.1, 36 f (§7.3.9) |
| `sl` | slow | Debuffs | 60 | `speedMult = 1 − n/100` (n = percent; same-kind adds, capped at 60%, td) | — | frost.1 20%, monarch.3 15%, frost.a1 30%, decay.a2 40%, rocker.a1 50% |
| `si` | silence | Debuffs | 1 | `silenced = true` | — | rocker.a1 "can't use moves for 60f" (§7.3.12) |
| `wh` | whispered | Debuffs | 1 | `blockDelayF = 6` | `consumeOn = "block"` | decay.q (§7.3.15) |
| `dc` | decay | Debuffs | 5 | `regenBlocked = true` | `incomingFrom = {style = "decay", perStack = 1, max = 5}` | decay passive (§7.3.15) |
| `iv` | invisible | Buffs | 1 | `{}` (render reads it, §4.7.6) | `consumeOn = "attack"` | rocker.2, 120 f (§7.3.12) |
| `nh` | next-hit bonus | Buffs | 5 | `nextHitAdd = n` (owner's outgoing hits only) | `consumeOn = "hit"` | rocker.2 "+5" |
| `ff` | frenzy | Buffs | 1 | `m1Chain = 6` | — | rocker.a3, 300 f (§7.3.12) |

Rules:
- **`nextHitAdd` and `incomingFrom` (decay bonus).** Part 3's `Damage.resolve` adds the attacker's `nextHitAdd` and the victim's `incomingFrom` value (when the attacker's style matches) to `flatAdds` (Part 3 §3.8.4). Both apply **only on `hit.kit.firstOfUse`**, and are **clamped** so the move's per-target total for this use cannot exceed its band cap (base 18, a1–a3 20, a4 30, M1 family unclamped; GDD §6.7). `nh` is removed on the first landed hit.
- **`m1Chain = 6` (Fool's Frenzy).** Part 3's `M1.onPress` allows indexes up to 6 while active. Hits 4 and 5 use the M1-3 frame data and hit 6 is the Ender (td reading of GDD §7.3.12 "her M1 chain becomes 6 hits").
- **Capacity.** At most 4 entries per list (Part 3 codec). With 5 debuff kinds possible, the entry with the smallest `untilF` is evicted (Part 3 `statusAdd`).

**Verify:** Lune `kit_status.registry_kinds` (every kind used by any kit exists); `kit_status.slow_additive_cap60`; `kit_status.decay_bonus_only_from_decay` (a `dc` 5 victim hit by hero.2 → no add; by decay.1 → +5 on the first hit only); `kit_status.bonus_clamped_to_band` (decay.a1 16 + 5 → 20); `kit_status.nh_consumed_on_hit`; `kit_status.wh_consumed_on_block`; `kit_status.iv_removed_on_attack`.

#### 4.3.3 Chassis modifiers from data (`passive.params` and `awakening.modifiers`)

Part 3's `Moves.chassisMods(style, s, cfg)` builds `Fsm.ChassisMods` from data, using the keys Part 3 recognizes (Part 3 §3.18 item 12). Part 4 authors kit data in exactly that convention:

```lua
passive.params = {
	ragdollFramesAdd = -10,                    -- top-level recognized keys: always on (stretch, GDD §7.3.2)
	lowHpThreshold = 30,                       -- hero (GDD §7.3.1): while s.HP < lowHpThreshold ...
	lowHp = { breakawayRefillMult = 1.5, enderDamageAdd = 2 }, -- ... these recognized keys also apply
	hoverSec = 3, hoverMode = "hold",          -- Part 4 key (§4.3.4, passiveStep); Part 3 ignores it
	-- any other key is read only by that style's hooks; Part 3 ignores unrecognized keys
}
awakening.modifiers = { m1RangeAdd = 3 }      -- recognized keys while AwkF0 > 0; visualScale/aura are render-only
awakening.hurtboxScale = 1.5                  -- ChassisMods.hurtboxScale while awakened (≥ 1, GDD §6.2)
```

Part 3's recognized passive/awakening keys are `breakawayRefillMult`, `enderDamageAdd`, `lowHpThreshold` + `lowHp`, `ragdollFramesAdd`, `m1RangeAdd`, `walkSpeedMult` and `airDash`. The merge order and arithmetic are Part 3's. Part 4's rule K11 checks that `awakening.modifiers` keys ⊆ those keys ∪ {`visualScale`, `aura`}.

**Verify:** Lune `kit_status.chassis_from_data` (hero at HP 29 → `breakawayRefillMult 1.5`, `enderDamageAdd 2`; at 30 → defaults; awakened → `m1RangeAdd 3`; stretch awakened → `hurtboxScale 1.5`, `walkSpeedMult 0.85`).

#### 4.3.4 Hover (`passiveStep`, generic)

For styles with `passive.params.hoverSec` (`rocker` 3 s, GDD §7.3.12; `frost` 2.5 s, GDD §7.3.13), `Styles.passiveStep`:
- `St == Air` and Jump held (`Fsm.IN.Jump` bit in `inp.bits`) and `s.Kit > 0` and vertical velocity ≤ 0 →
  - `motion = vector (md.X·W, 0, md.Z·W)`, where `md = inp.moveDir` and `W = cfg.Chassis.WalkSpeed`;
  - `patch.Kit = s.Kit − 1`.
- `St == Idle` (grounded) and `s.Kit ≠ hoverSec·60` → `patch.Kit = hoverSec·60` (refill per jump).
- Otherwise nothing.

`Kit` is therefore "hover frames left" for these two styles. `hero` uses `Kit` for the Sword Swap mode. Other styles leave it at 0.

**Verify:** Lune `kit_status.hover_budget` (frost holds Jump airborne for 200 f → 150 frames of vy = 0, then gravity); scenario `kit.frost.hover`.

#### 4.3.5 Style Moments

GDD §6.9: +10 meter per landed style moment; GDD §9.2: mastery XP. The tracker lives in `Styles` server memory. Per attacker rid it keeps a ring of the last 8 tokens `{tok, victim, f}`. Tokens:
- `"<moveId>:use"` (serverHook `use`);
- `"<moveId>:hit"` (`hitDealt` with outcome `hit`);
- `"wallsplat"` (`onWallSplat`).

After each push:
1. Test the style's `MomentDef.seq` as an ordered subsequence within the last `windowF` frames.
2. With `sameVictim`, every `:hit` and `wallsplat` token must share one victim (`:use` tokens match any victim).
3. On a match: append `{k = "style_moment", rid, f, a = victim}` to `HookOut.events`, clear the ring, and block further moments for `Kits.MomentCooldownFrames`.
4. Log outside the sim: `[ALG][Style] moment style=<id> rid=<rid> victim=<rid>`.

Part 3 grants +10 meter on `style_moment` (Part 3 §3.9.2). Part 5 grants mastery XP from the flushed event.

| Style | Moment (GDD) | Definition |
|---|---|---|
| hero | Grass Sword Lash → pull → Flying Kick → Wall Splat (§7.3.1) | `seq = {"hero.3:hit", "hero.4:hit", "wallsplat"}`, `windowF = 180` (td), `sameVictim = true` |
| stretch | Reflect a projectile back into its owner (§7.3.2) | `onHitDealt`: `info.reflectedHome` → event |
| monarch | Move 1 root into move 4 beam (§7.3.9) | `seq = {"monarch.1:hit", "monarch.4:hit"}`, `windowF = 180` (td), `sameVictim = true` |
| rocker | Bat Shift → Invisible Ambush → Red Drain (§7.3.12) | `seq = {"rocker.q:use", "rocker.2:use", "rocker.3:hit"}`, `windowF = 360` (td), `sameVictim = true` |
| frost | Freeze in Ice Prison, then land the Quirk summon (§7.3.13) | `seq = {"frost.2:hit", "frost.q:hit"}`, `windowF = 180` (td), `sameVictim = true` |
| decay | Stack 5 Decay, then KO (§7.3.15) | `onKO`: the victim's `dc` stacks ≥ 5 → event |

**Verify:** Lune `kits.moments_sequence` (a token stream per style → exactly one event; an out-of-order stream → none; the second match within 600 f → none). Scenario `kit.moment_meter` (hero performs its moment on a dummy → meter +10 beyond the damage gains; `[ALG][Style] moment` line).

#### 4.3.6 Per-style passive and Quirk implementation

| Style | Passive (GDD) | Implementation | Quirk (GDD) | Implementation |
|---|---|---|---|---|
| hero | Hero's Heart: below 30 HP Breakaway refills 50% faster and the Ender deals +2 (§7.3.1) | Data: `lowHpThreshold = 30, lowHp = {breakawayRefillMult = 1.5, enderDamageAdd = 2}` (Part 3 recognized keys). `onLowHP`: event `{k = "face", s = "grit"}` (render) | Sword Swap: toggle between two swords, mode A (move 1 startup −3f) and mode B (move 1 +30% guard damage), CD 3 | `Buff` with `toggleMode` (§4.2.8). `Kit` 0 = first mode (spawn), 1 = second. Variants of `hero.1` via `Styles.variants`. Gear blade colour follows `Kit` (§4.7.4) |
| stretch | Rubber Body: ragdolls end 10f earlier (§7.3.2) | Data: `ragdollFramesAdd = -10` (Part 3 `Stun.ragdollFrames`) | Stretch Grapple to terrain ≤ 40 studs, CD 6 | `Mobility` mode `grapple` (§4.2.9) |
| monarch | Scientific Method: landing two *different* moves within 4 s reduces all cooldowns by 1 s (§7.3.9) | `onHitDealt` (server, `outcome == "hit"`, `firstOfUse`, kind ≠ M1 family). Server memory `sm[rid] = {slot, f}`. If `sm.slot ≠ nil`, `sm.slot ≠ info.slot` and `f − sm.f ≤ 240` → `cooldownsMinusSec = 1` and `sm = nil` (td: each pair triggers once). Otherwise `sm = {info.slot, f}`. Summon and zone hits count for their source move (td) | Analyze: shows the target's cooldowns and meter for 5 s, CD 8 | `Buff` with `analyze` (presentation only, §4.2.8) |
| rocker | Color Drain: each Ender she lands heals 2 HP; she can hover 3 s per jump (§7.3.12) | `onHitDealt`: `info.isEnder and outcome == "hit"` → `heal = 2`. Hover data `hoverSec = 3`, `hoverMode = "hold"` (§4.3.4) | Bat Shift: fly 25 studs, i-frames on frames 1–6, CD 5 | `Mobility` mode `flight`, `iFrames = {from = 1, to = 6}` |
| frost | Beard Flight: hold Jump to hover 2.5 s (§7.3.13) | Data `hoverSec = 2.5`, `hoverMode = "hold"` | Quirk summon: waddles to the nearest enemy and smashes a bottle (18f flinch), 3 dmg, CD 6 | `Summon` `ai = "chase"`, `once = true`, `stun = "flinch"` |
| decay | Extinction: hits apply Decay (max 5 stacks, 8 s); each stack blocks regen and adds +1 damage to his next hit (max +5) (§7.3.15) | `onHitDealt` (outcome `hit`, `firstOfUse`): `n = def.params.decayStacks or 1` for moves, 1 for the **Ender only** among M1s (td, see §4.11 Q1), 0 for `tick` hits → `debuffsOnVictim = {{kind = "dc", frames = 480, n = n}}`. The bonus is `incomingFrom` (§4.3.2). `onKO`: moment | Hypnotic Whisper: Whispered for 4 s, camera sway, next block raises 6f later, CD 8 | `Buff` `target = "front"`, `debuffs = {{kind = "wh", frames = 240, n = 1}}`. The camera sway is client-local on the victim, scaled to 0 with Reduced Motion (Part 6) |

**Verify:** Lune `kits.hero_heart_low_hp` (via `kit_status.chassis_from_data`), `kits.monarch_scientific_method` (hits 1 then 3 within 240 f → `cooldownsMinusSec = 1`; 1 then 1 → none; 1 then 3 at 241 f → none), `kits.rocker_color_drain`, `kits.decay_stacks_per_use` (decay.a3: three skulls hitting one target → +1 stack total, because only the `firstOfUse` hit counts, td), `kits.decay_m1_ender_only`.

#### 4.3.7 Lune spec `tests/core/kit_status.spec.luau`

Contains every `kit_status.*` test named above. It requires `Core/Moves` (Part 3) for the codec and checks that Part 4's `STATUS` entries round-trip through `statusEncode`/`statusDecode` within 50 characters at 4 entries.

---

### 4.4 Awakening and Realm

#### 4.4.1 Move swap

Part 3 runs activation (72 f, invulnerable, heal 10, +1 pip, shockwave), duration (60 s, +5 s per KO, max 90 s), and end (Part 3 §3.9.2). Part 4's content:
- `awakening.moves` are slots `a1..a4`. Part 3 maps keys 1–4 to them while `AwkF0 > 0` (`Move` Sub 11–14).
- The Quirk and M1 are unchanged. M1 presentation is unchanged except gear and visual scale (§4.7).
- **Cooldowns.** Part 3 zeroes `Cd1..Cd4` at start and end (both sets start ready). Mode variants (`hero.1`) apply only to base moves.
- **In-progress move.** A move in progress when Awakening ends completes with its captured MoveDef. After that, slots map to base moves.
- **Carriers.** Awakened summons, zones and projectiles persist until their own life ends (td). The Realm follows Part 3's close rules.
- **`onAwaken` / `onAwakenEnd`** (both peers). `rocker`: `onAwakenEnd` removes `ff` from `Buffs` (patch). Other VS styles: none.

**Verify:** scenario `kit.awaken.swap` (per VS style: `Dev.Command setMeter 100`, press Awaken; after 72 f, keys 1–4 start `<style>.a1..a4` (read `Sub` = 11–14 and the `MoveId` probe); after `AwkEndF`, keys map back to base moves).

#### 4.4.2 Modifiers and hurtbox scaling

| Style | Awakening (GDD) | `hurtboxScale` | `modifiers` |
|---|---|---|---|
| hero | M1 range +3 studs (§7.3.1) | 1 | `{ m1RangeAdd = 3 }` |
| stretch | Giant form: hurtbox ×1.5, walk −15%, AoEs ×1.5 (§7.3.2) | 1.5 | `{ walkSpeedMult = 0.85, visualScale = 2 }` |
| monarch | Battle armour, visual only (§7.3.9) | 1 | `{}` |
| rocker | Essence swirl, visual only (§7.3.12) | 1 | `{}` |
| frost | Crown flare, visual only (§7.3.13) | 1 | `{}` |
| decay | Towering form: hurtbox ×1.3, decay aura (§7.3.15) | 1.3 | `{ visualScale = 1.6, aura = "vfx.decay.aura" }` |

Notes:
- **stretch giant-form AoE ×1.5.** It is authored into the a1–a4 data, which exists only in that form (td reading, see §4.11 Q2). There is no runtime `aoeScale`.
- **Decay "aura".** It is cosmetic (td, §4.11 Q3).
- **Visual scale.** `visualScale` and `aura` are presentation-only. Part 6's AppearanceController scales the visual look, never the HRP or hurtbox. The hurtbox scales only by `hurtboxScale` ≥ 1 (GDD §6.2; Part 3 §3.12.2).

**Verify:** K10/K11. Scenario `kit.awaken.hurtbox` (stretch awakened → hurtbox part size = 1.5 × base on the server; decay → 1.3 ×).

#### 4.4.3 Realm content: `Realm.rules["decay.a4"]` (in `Core/Realm.luau`)

GDD §7.3.15, the decay A4 Realm (40 studs, 12 s). Inside:
- enemy healing is reversed;
- every enemy gains +1 Decay every 2 s;
- the caster's move cooldowns are −30%.

Part 3 keys rules by `RuleId` = the MoveDef id (Part 3 §3.9.3).

```lua
--!strict
-- content of Realm.rules in src/Shared/Core/Realm.luau (Part 4 fills; Part 3 owns the type and framework)
rules["decay.a4"] = {
	healMult = function(m: MemberView): number
		return if m.isCaster then 1 else -1 -- GDD §7.3.15 "enemy healing is reversed (heals deal damage instead)"
	end,
	cooldownMult = function(_casterRid: number): number
		return 0.7 -- GDD §7.3.15: the caster's move cooldowns are −30%
	end,
	tickFrames = 120, -- GDD §7.3.15 "+1 Decay every 2 s"
	tick = function(_r: RealmView, m: MemberView, f: number): { statusAdd: Moves.Status?, hit: Fsm.HitDef? }?
		if m.isCaster or m.warded then
			return nil -- Stubborn Heart negates rule effects (GDD §6.10)
		end
		return { statusAdd = { kind = "dc", untilF = f + 480, n = 1 } } -- 8 s Decay (GDD §7.3.15 passive)
	end,
}
```

Implications enforced by Part 3's framework:
- **Negative heals.** A heal with `healMult = −1` becomes damage, attributed to the caster with `hit.realm = true` (meter ×0.5, GDD §6.9) and `tick = true`.
- **Regen.** It is already off for members (Part 3 §3.8.7, GDD §6.2).
- **Decay +1 per 2 s.** It feeds the passive's `incomingFrom` bonus, so the caster's first hit per use gains up to +5 inside the Realm.

**Verify:** Lune `kits.realm_well_rules` in `kits.spec.luau` (an enemy MemberView → `healMult −1`; caster → 1; warded → `tick` nil; `cooldownMult 0.7`). Scenario `kit.decay.realm` (decay awakened casts a4 with 2 dummies at 10 and 30 studs and 1 at 45):
- after 90 f, members are 2, and `[ALG][Combat] realm_open` appears (Part 3 event log);
- each member has `dc` stacks = 1 after 2 s and 3 after 6 s;
- a dummy healed 5 by the test seam loses 5 instead;
- `Cd1` set while inside is `0.7 ×` the base cooldown;
- the realm closes at 12 s.

#### 4.4.4 Realms for stub styles

`demon`, `wish` and `chaos` are stubs (`implemented = false`). Their a4 is a stub move with `behavior = "RealmCast"`, `tags = {"Realm"}` and no `Realm.rules` entry. K12 and the RealmCast validate rule are skipped for stubs (K-rules apply to implemented styles only). Adding their rules is part of §4.8.

---

### 4.5 Kit data

#### 4.5.1 Conventions

- **Damage and cooldown** are the exact GDD §7.3 values. `damage` is the GDD "Dmg":
  - per-use total for `total`-model moves;
  - per hit for `perHit`-model moves ("3 per poke", "8 per smash", "3 per touch");
  - for "≤ N" entries it is N, the maximum, and validation confirms `maxDamage ≤ N`.
- **"N-stud area/pool/cloud/storm/circle"** is read as diameter N (radius N/2). "Within N studs" or "N-stud radius" is a radius (td reading).
- **Frame data, guard damage, hitstun, knockback, hitboxes, telegraphs and params** are tuning defaults chosen to satisfy:
  - GDD §6.14 (tell ≥ 8 f before > 10 damage);
  - GDD §7.1 rule 5;
  - GDD §6.4 (guard 20–60, chip 20%);
  - GDD §18.2 (startup ≥ 6; ≥ 8 for big hits; AoE fuses);
  - GDD §6.7 (bands).
- **Knockback** follows Part 3 §3.8.5: `ragdoll = true` → ragdoll profile (`params.ragdollFrames`); `launch = true` → `dist` is the height, with `params.hangFrames`; otherwise a planar push.
- **Canon pointers.** Every move line ends with `-- canon: GDD §7.3.N <slot>` (P4-R5).

#### 4.5.2 `hero` (GDD §7.3.1, Starter, SwordHero)

```lua
--!strict
-- src/Shared/Styles/hero.luau — hero kit. Realm: shared. Pure: load. Spec §4.5.2
local Kit = require("../Behaviors/Buff")

return function(K: Kit.KitBuilder): Kit.StyleModule
	local M, box, kb, tele, v3 = K.move, K.box, K.kb, K.tele, K.v3
	local def = {
		id = "hero", rarity = "starter", archetype = "SwordHero", implemented = true, -- GDD §7.2
		hp = 100, -- GDD §6.2
		palette = { primary = K.rgb(63, 169, 245), secondary = K.rgb(245, 245, 240), accent = K.rgb(242, 193, 78) }, -- §4.7.1
		passive = { id = "hero.passive", params = {
			lowHpThreshold = 30, lowHp = { breakawayRefillMult = 1.5, enderDamageAdd = 2 }, -- GDD §7.3.1 Passive (Part 3 keys)
		} },
		quirk = M("hero.q", { -- canon: GDD §7.3.1 Quirk
			behavior = "Buff", tags = { "Buff" }, damage = 0, cooldown = 3, -- GDD §7.3.1
			startup = 6, active = 1, recovery = 10, -- (td)
			params = { toggleMode = { count = 2, spawn = 0, effects = {
				{ move = "hero.1", startupDelta = -3 },     -- mode 0: move 1 startup −3f (GDD §7.3.1)
				{ move = "hero.1", guardDamageMult = 1.3 }, -- mode 1: move 1 +30% guard damage (GDD §7.3.1)
			} } },
		}),
		moves = {
			M("hero.1", { -- canon: GDD §7.3.1 row 1
				behavior = "Rush", tags = { "Rush", "Mobility" }, damage = 8, cooldown = 7, -- GDD §7.3.1
				startup = 14, active = 5, recovery = 20, guardDamage = 25, hitstun = 30, -- (td); mode 0 → startup 11
				knockback = kb(4, 20, false), hitbox = box(6, 6, 6), -- (td)
				params = { travel = { from = 3, to = "startup", studs = 20, arc = "leap", height = 6 }, -- 20 studs: GDD §7.3.1; height (td)
					stopOnHit = true, selfBounce = { studs = 6, height = 3, frames = 12 } }, -- "small bounce" (td)
			}),
			M("hero.2", { -- canon: GDD §7.3.1 row 2
				behavior = "Rush", tags = { "Rush", "Extender" }, damage = 11, cooldown = 9, -- GDD §7.3.1
				startup = 8, active = 30, recovery = 18, guardDamage = 30, hitstun = 20, -- (td); startup 8 combos from M1-3 (K04)
				hitbox = box(5, 5, 5), telegraph = tele("Arc", 8, 5, 5), -- (td); > 10 dmg → tell ≥ 8f (GDD §6.14)
				params = { travel = { from = 1, to = "startup", studs = 6 }, -- (td)
					hits = { 1.5, 1.5, 1.5, 1.5, 1.5, 3.5 }, hitFrames = { 1, 6, 11, 16, 21, 26 }, -- 6-hit barrage = 11 (GDD §7.3.1)
					lastHit = { hitstun = 36, knockback = kb(10, 90, false, true) }, hangFrames = 20 }, -- final hit launches (GDD), (td)
			}),
			M("hero.3", { -- canon: GDD §7.3.1 row 3
				behavior = "Projectile", tags = { "Projectile", "Extender" }, damage = 6, cooldown = 12, -- GDD §7.3.1
				startup = 10, active = 4, recovery = 18, guardDamage = 20, hitstun = 34, -- (td)
				params = { kind = "beam", range = 28, width = 2, extendF = 4, reflectable = false, -- 28 studs: GDD §7.3.1
					onHit = { pull = { stopStuds = 4, frames = 12 } } }, -- "yanks the target toward" (GDD); (td)
			}),
			M("hero.4", { -- canon: GDD §7.3.1 row 4
				behavior = "Rush", tags = { "Rush", "Opener" }, signature = true, damage = 14, cooldown = 16, -- GDD §7.3.1
				startup = 16, active = 8, recovery = 24, guardDamage = 45, hitstun = 40, -- (td)
				projectileArmor = true, -- "projectile armor during startup" (GDD §7.3.1)
				knockback = kb(30, 10, true), hitbox = box(5, 5, 6), telegraph = tele("Line", 16, 4, 26), -- (td)
				params = { travel = { from = "active", to = "active", studs = 24 }, stopOnHit = true, -- (td)
					wallSplat = true, ragdollFrames = 60 }, -- "wall splat on hit" (GDD §7.3.1); 60f as Ender (GDD §6.3)
			}),
		},
		awakening = {
			id = "hero.awaken", durationSec = 60, hurtboxScale = 1, -- GDD §6.9, §6.2
			modifiers = { m1RangeAdd = 3 }, -- "M1 range +3 studs" (GDD §7.3.1)
			moves = {
				M("hero.a1", { -- canon: GDD §7.3.1 A1
					behavior = "Rush", tags = { "Rush" }, damage = 14, cooldown = 8, -- GDD §7.3.1
					startup = 10, active = 24, recovery = 16, guardDamage = 35, hitstun = 22, -- (td)
					hitbox = box(12, 6, 10), telegraph = tele("Arc", 10, 12, 10), -- forward arc (GDD); (td)
					params = { travel = { from = 1, to = "startup", studs = 4 }, hits = { 3.5, 3.5, 3.5, 3.5 }, -- 4 lashes (GDD)
						hitFrames = { 1, 7, 13, 19 }, lastHit = { hitstun = 34, knockback = kb(12, 20, false) } }, -- (td)
				}),
				M("hero.a2", { -- canon: GDD §7.3.1 A2
					behavior = "Grab", tags = { "Grab" }, damage = 16, cooldown = 12, -- GDD §7.3.1
					startup = 14, active = 6, recovery = 26, parryable = false, -- (td)
					hitbox = box(5, 6, 9, 5), telegraph = tele("Circle", 14, 8, 8), -- (td)
					params = { groundOnly = true, holdF = 40, offset = v3(0, 4, -5), sub = 2, finish = "slam", -- "grabs and slams" (GDD); (td)
						ticks = { { at = 34, damage = 16 } }, finishKb = kb(4, 0, true), ragdollFrames = 60, finishRecovery = 18 },
				}),
				M("hero.a3", { -- canon: GDD §7.3.1 A3
					behavior = "Rush", tags = { "Rush" }, damage = 12, cooldown = 12, -- GDD §7.3.1 (12 + 6)
					startup = 12, active = 8, recovery = 20, guardDamage = 40, hitstun = 40, -- (td)
					hitbox = box(5, 6, 5), telegraph = tele("Line", 12, 4, 22), -- (td)
					params = { travel = { from = "active", to = "active", studs = 18 }, passThrough = true, -- dash-through (GDD); (td)
						echo = { delayF = 30, damage = 6, hitstun = 30, knockback = kb(14, 25, true), ragdollFrames = 48, -- echo 0.5 s, +6 (GDD)
							width = 5, length = 18 } }, -- (td)
				}),
				M("hero.a4", { -- canon: GDD §7.3.1 A4
					behavior = "Zone", tags = { "Zone", "Ultimate" }, damage = 26, cooldown = 25, -- GDD §7.3.1
					startup = 12, active = 1, recovery = 30, guardDamage = 50, hitstun = 30, -- (td)
					telegraph = tele("Circle", 60, 15, 15), -- 15-stud circle, 1.0 s telegraph (GDD §7.3.1)
					params = { anchor = "aim", aimRange = 40, shape = "circle", radius = 7.5, armF = 60, lifeF = 70, -- (td) / GDD
						tick = { everyF = 10, damage = 3, count = 5, blockable = true }, -- sword rain 15 (td split of 26)
						bursts = { { atF = 60, damage = 11, hitstun = 40, knockback = kb(8, 60, true), ragdollFrames = 60 } }, -- downstrike (td)
						maxHitsPerTarget = 6, cinematicFrames = 60, teleAnchor = "carrier" }, -- (td) ≤ 120 (GDD §6.8)
				}),
			},
		},
		m1 = { anim = "anim.hero.m1", vfx = "vfx.hero.m1", sfx = "sfx.hero.m1" },
		kitProps = { "model.hero.gear_hat", "model.hero.gear_pack", "model.hero.gear_sword" },
	}
	return {
		def = def,
		hooks = {
			onLowHP = function(h, _hp)
				return { events = { { k = "face", rid = h.rid, f = h.f, s = "grit" } } } -- render-only cue (td)
			end,
		},
		moment = { seq = { "hero.3:hit", "hero.4:hit", "wallsplat" }, windowF = 180, sameVictim = true }, -- GDD §7.3.1; (td)
		anim = {
			m1 = { windup = "windup_side", strike = "slash_side",
				alt = { "slash_side", "slash_down", "slash_side", "slash_side", "slash_down", "slash_side", "kick", "slash_down", "windup_high", "slash_down", "slash_side", "slash_side" } },
			q = { windup = "ready", strike = "buff", face = { windup = "smile", active = "grit", recovery = "neutral" } },
			["1"] = { windup = "crouch", mid = "leap", strike = "slash_down" },
			["2"] = { windup = "punch_wind", strike = "punch_r", alt = { "punch_r", "punch_l" }, finish = "windup_high" },
			["3"] = { windup = "windup_side", strike = "point" },
			["4"] = { windup = "kick_wind", strike = "kick", face = { windup = "shout", active = "shout", recovery = "grit" } },
			a1 = { windup = "windup_side", strike = "slash_side", alt = { "slash_side", "slash_down" } },
			a2 = { windup = "grab_reach", strike = "grab_hold", finish = "slash_down" },
			a3 = { windup = "windup_side", strike = "slash_side", settle = "ready" },
			a4 = { windup = "cast_raise", strike = "slash_down", face = { windup = "awaken", active = "shout", recovery = "grit" } },
			awaken = { windup = "crouch", strike = "cast_raise", face = { windup = "grit", active = "awaken", recovery = "smile" } }, -- anim.<style>.awaken (72f, §4.6.1)
		},
	}
end
```

#### 4.5.3 `stretch` (GDD §7.3.2, Starter, StretchyCompanion)

```lua
--!strict
-- src/Shared/Styles/stretch.luau — stretch kit. Realm: shared. Pure: load. Spec §4.5.3
local Kit = require("../Behaviors/Buff")

return function(K: Kit.KitBuilder): Kit.StyleModule
	local M, box, sphere, kb, tele, v3 = K.move, K.box, K.sphere, K.kb, K.tele, K.v3
	local def = {
		id = "stretch", rarity = "starter", archetype = "StretchyCompanion", implemented = true, hp = 100, -- GDD §7.2, §6.2
		palette = { primary = K.rgb(242, 163, 58), secondary = K.rgb(185, 119, 46), accent = K.rgb(255, 231, 179) },
		passive = { id = "stretch.passive", params = { ragdollFramesAdd = -10 } }, -- GDD §7.3.2 Rubber Body (Part 3 key)
		quirk = M("stretch.q", { -- canon: GDD §7.3.2 Quirk
			behavior = "Mobility", tags = { "Mobility" }, damage = 0, cooldown = 6, -- GDD §7.3.2
			startup = 8, active = 1, recovery = 8, -- (td); StDur extended by travel
			params = { mode = "grapple", maxRange = 40, speed = 120, maxTravelF = 20, -- 40 studs: GDD §7.3.2; rest (td)
				whiffRecoveryF = 12, whiffCooldownSec = 2, air = true }, -- (td)
		}),
		moves = {
			M("stretch.1", { -- canon: GDD §7.3.2 row 1
				behavior = "Projectile", tags = { "Projectile", "Opener" }, damage = 7, cooldown = 6, -- GDD §7.3.2
				startup = 10, active = 6, recovery = 16, guardDamage = 22, hitstun = 30, knockback = kb(5, 0, false), -- (td)
				params = { kind = "beam", range = 25, width = 2.5, extendF = 6, reflectable = false }, -- 25 studs: GDD §7.3.2
			}),
			M("stretch.2", { -- canon: GDD §7.3.2 row 2
				behavior = "Rush", tags = { "Rush", "AoE" }, signature = true, damage = 15, cooldown = 16, -- GDD §7.3.2
				startup = 18, active = 5, recovery = 26, guardDamage = 50, hitstun = 40, -- (td)
				knockback = kb(18, 60, true), hitbox = sphere(20), -- 10-stud cone (GDD) = radius 10 + coneDeg
				telegraph = tele("Cone", 18, 11.5, 10), -- (td) x = 2·10·tan30°
				params = { travel = { from = 10, to = "startup", studs = 4 }, coneDeg = 60, ragdollFrames = 60 }, -- "launches" (GDD); (td)
			}),
			M("stretch.3", { -- canon: GDD §7.3.2 row 3
				behavior = "Counter", tags = { "Counter" }, damage = 0, cooldown = 12, -- GDD §7.3.2
				startup = 4, active = 72, recovery = 14, -- 72f = 1.2 s (GDD §7.3.2); (td)
				params = { arcDeg = 180, vsMelee = "stagger", staggerF = 18, reflectProjectiles = true, endOnTrigger = false }, -- 18f stagger (GDD)
			}),
			M("stretch.4", { -- canon: GDD §7.3.2 row 4
				behavior = "Grab", tags = { "Grab", "Extender" }, damage = 9, cooldown = 13, -- GDD §7.3.2
				startup = 9, active = 4, recovery = 22, hitbox = box(4, 5, 6), -- (td); startup 9 combos from M1 (K04)
				params = { groundOnly = true, holdF = 54, offset = v3(0, 0, -3), sub = 3, -- (td)
					ticks = { { at = 12, damage = 2 }, { at = 24, damage = 2 }, { at = 36, damage = 2 }, { at = 50, damage = 3 } }, -- 3 squeezes + throw = 9 (GDD)
					finish = "throw", throwDir = "input", finishKb = kb(22, 15, true), ragdollFrames = 60, finishRecovery = 16 }, -- "held direction" (GDD)
			}),
		},
		awakening = {
			id = "stretch.awaken", durationSec = 60, hurtboxScale = 1.5, -- GDD §6.9; ×1.5 (GDD §6.2, §7.3.2)
			modifiers = { walkSpeedMult = 0.85, visualScale = 2 }, -- walk −15%, 2× size (GDD §7.3.2)
			moves = {
				M("stretch.a1", { -- canon: GDD §7.3.2 A1
					behavior = "Strike", tags = { "AoE" }, damage = 14, cooldown = 8, -- GDD §7.3.2
					startup = 16, active = 4, recovery = 22, guardDamage = 40, hitstun = 36, -- (td)
					knockback = kb(10, 50, true), hitbox = sphere(24, 0, -2), -- 12-stud radius (GDD §7.3.2)
					telegraph = tele("Circle", 16, 24, 24), params = { ragdollFrames = 54 }, -- (td)
				}),
				M("stretch.a2", { -- canon: GDD §7.3.2 A2
					behavior = "Grab", tags = { "Grab" }, damage = 12, cooldown = 11, -- GDD §7.3.2
					startup = 12, active = 5, recovery = 24, hitbox = box(7, 8, 9), telegraph = tele("Arc", 12, 7, 9), -- (td)
					params = { groundOnly = true, holdF = 30, offset = v3(0, 6, -4), sub = 2, ticks = { { at = 28, damage = 12 } },
						finish = "throw", throwDir = "facing", finishKb = kb(30, 25, true), ragdollFrames = 60, finishRecovery = 14 }, -- tosses 30 studs (GDD)
				}),
				M("stretch.a3", { -- canon: GDD §7.3.2 A3
					behavior = "Counter", tags = { "Counter" }, damage = 10, cooldown = 12, chip = 0, parryable = false, -- GDD §7.3.2
					startup = 4, active = 48, recovery = 16, -- (td)
					params = { arcDeg = 180, vsMelee = "launch", counterKb = kb(14, 90, false, true), counterHitstun = 40, -- "launches skyward" (GDD)
						hangFrames = 12, reflectProjectiles = false, endOnTrigger = true, triggerRecovery = 12 }, -- (td)
				}),
				M("stretch.a4", { -- canon: GDD §7.3.2 A4
					behavior = "Zone", tags = { "AoE", "Ultimate" }, damage = 28, cooldown = 25, hitstun = 0, -- GDD §7.3.2 (≤ 28)
					startup = 30, active = 180, recovery = 20, guardDamage = 40, -- 3 s whirlwind (GDD); (td)
					telegraph = tele("Circle", 30, 50, 50), -- within 25 studs (GDD)
					params = { anchor = "selfFollow", shape = "circle", radius = 25, armF = 0, lifeF = 180,
						tick = { everyF = 20, damage = 3.5, count = 9, blockable = true }, maxHitsPerTarget = 8, -- 8 × 3.5 = 28 (td)
						pullStudsPerSec = 4, followMove = true }, -- (td)
				}),
			},
		},
		m1 = { anim = "anim.stretch.m1", vfx = "vfx.stretch.m1", sfx = "sfx.stretch.m1" },
		kitProps = { "model.stretch.gear_ears", "model.stretch.gear_tail" },
	}
	return {
		def = def,
		hooks = {
			onHitDealt = function(h, info)
				if info.reflectedHome and info.outcome == "hit" then -- GDD §7.3.2 Style Moment
					return { events = { { k = "style_moment", rid = h.rid, f = h.f, a = info.victimRid } } }
				end
				return nil
			end,
		},
		moment = nil,
		anim = {
			m1 = { windup = "punch_wind", strike = "punch_r",
				alt = { "punch_r", "punch_l", "punch_r", "punch_r", "punch_l", "punch_r", "kick", "cast_push", "windup_high", "stomp_down", "punch_l", "punch_l" } },
			q = { windup = "point", strike = "cast_push" },
			["1"] = { windup = "punch_wind", strike = "punch_r" },
			["2"] = { windup = "windup_high", strike = "stomp_down", face = { windup = "grit", active = "shout", recovery = "smile" } },
			["3"] = { windup = "ready", strike = "guard_wide" },
			["4"] = { windup = "grab_reach", strike = "grab_hold", finish = "throw" },
			a1 = { windup = "stomp_up", strike = "stomp_down" },
			a2 = { windup = "grab_reach", strike = "grab_hold", finish = "throw" },
			a3 = { windup = "ready", strike = "guard_wide" },
			a4 = { windup = "cast_raise", strike = "spin_a", alt = { "spin_a", "spin_b" }, face = { windup = "awaken", active = "shout", recovery = "neutral" } },
			awaken = { windup = "crouch", strike = "cast_raise", face = { windup = "grit", active = "awaken", recovery = "smile" } }, -- anim.<style>.awaken (72f, §4.6.1)
		},
	}
end
```

#### 4.5.4 `monarch` (GDD §7.3.9, Rare, CandyScienceMonarch)

```lua
--!strict
-- src/Shared/Styles/monarch.luau — monarch kit. Realm: shared. Pure: load. Spec §4.5.4
local Kit = require("../Behaviors/Buff")

return function(K: Kit.KitBuilder): Kit.StyleModule
	local M, kb, tele, v3 = K.move, K.kb, K.tele, K.v3
	local def = {
		id = "monarch", rarity = "rare", archetype = "CandyScienceMonarch", implemented = true, hp = 100, -- GDD §7.2, §6.2
		palette = { primary = K.rgb(244, 143, 177), secondary = K.rgb(216, 27, 96), accent = K.rgb(127, 216, 190) },
		passive = { id = "monarch.passive", params = { windowSec = 4, reduceSec = 1 } }, -- GDD §7.3.9 Scientific Method
		quirk = M("monarch.q", { -- canon: GDD §7.3.9 Quirk
			behavior = "Buff", tags = { "Utility" }, damage = 0, cooldown = 8, -- GDD §7.3.9
			startup = 8, active = 1, recovery = 10, -- (td)
			params = { target = "self", analyze = { frames = 300, range = 40, coneDeg = 60 } }, -- 5 s (GDD); range (td)
		}),
		moves = {
			M("monarch.1", { -- canon: GDD §7.3.9 row 1
				behavior = "Projectile", tags = { "Projectile" }, damage = 7, cooldown = 7, -- GDD §7.3.9
				startup = 8, active = 2, recovery = 16, guardDamage = 22, hitstun = 26, -- (td)
				params = { kind = "bullet", speed = 110, range = 55, radius = 1.2, pierce = 0, reflectable = true, -- (td)
					onHit = { debuffs = { { kind = "rt", frames = 36, n = 1 } } }, -- roots for 36f (GDD §7.3.9)
					extender = true }, -- K04: 8 + 1 + 3 ≤ 12
			}),
			M("monarch.2", { -- canon: GDD §7.3.9 row 2
				behavior = "Summon", tags = { "Summon" }, damage = 3, cooldown = 15, -- 3 per poke (GDD §7.3.9)
				startup = 12, active = 1, recovery = 14, guardDamage = 10, hitstun = 18, -- (td)
				telegraph = tele("Circle", 12, 6, 6), -- (td) K08 (max 18 > 10)
				params = { dmgModel = "perHit", bodies = 2, hpEach = 20, lifeF = 480, -- 2 NPCs, 20 HP, 8 s (GDD §7.3.9)
					speed = 16, ai = "chase", leash = 40, spawnOffsets = { v3(-3, 0, -2), v3(3, 0, -2) }, -- (td)
					attack = { range = 6, startupF = 12, everyF = 72, size = v3(3, 5, 7), tele = true }, -- (td)
					maxHitsPerTarget = 3, model = "model.monarch.summon_guard" }, -- 2 × 3 × 3 = 18 ≤ 18 (GDD §6.7)
			}),
			M("monarch.3", { -- canon: GDD §7.3.9 row 3
				behavior = "Zone", tags = { "Zone" }, damage = 10, cooldown = 12, chip = 0, parryable = false, -- ≤ 10 (GDD §7.3.9)
				startup = 10, active = 1, recovery = 14, -- (td)
				telegraph = tele("Circle", 20, 12, 12), -- 12-stud cloud (GDD)
				params = { anchor = "aim", aimRange = 30, shape = "circle", radius = 6, armF = 20, lifeF = 300, -- 5 s (GDD); (td)
					tick = { everyF = 60, damage = 2, count = 5 }, stay = { kind = "sl", frames = 12, n = 15 }, -- 2 dmg/s, −15% (GDD)
					maxHitsPerTarget = 5, teleAnchor = "carrier" },
			}),
			M("monarch.4", { -- canon: GDD §7.3.9 row 4
				behavior = "Projectile", tags = { "Projectile" }, signature = true, damage = 16, cooldown = 18, -- GDD §7.3.9
				startup = 42, active = 6, recovery = 24, guardDamage = 50, hitstun = 40, knockback = kb(20, 15, true), -- 42f telegraph (GDD)
				telegraph = tele("Line", 42, 4, 60), -- (td) length
				params = { kind = "beam", range = 60, width = 4, extendF = 0, reflectable = false, ragdollFrames = 60, -- (td)
					setPiece = "model.monarch.guardian_bust" },
			}),
		},
		awakening = {
			id = "monarch.awaken", durationSec = 60, hurtboxScale = 1, modifiers = {}, -- GDD §6.9
			moves = {
				M("monarch.a1", { -- canon: GDD §7.3.9 A1
					behavior = "Projectile", tags = { "Projectile" }, damage = 15, cooldown = 9, -- GDD §7.3.9
					startup = 12, active = 9, recovery = 18, guardDamage = 36, hitstun = 30, knockback = kb(6, 30, false), -- (td)
					telegraph = tele("Circle", 36, 10, 10),
					params = { kind = "lob", count = 3, everyF = 4, flightF = 36, impactRadius = 5, aimRange = 45, -- 3 shells (GDD)
						pattern = { v3(0, 0, 0), v3(5, 0, 4), v3(-5, 0, 4) }, hits = { 5, 5, 5 }, reflectable = false, teleAnchor = "carrier" }, -- (td)
				}),
				M("monarch.a2", { -- canon: GDD §7.3.9 A2
					behavior = "Summon", tags = { "Summon", "Rush" }, damage = 14, cooldown = 11, -- GDD §7.3.9
					startup = 14, active = 1, recovery = 18, guardDamage = 40, hitstun = 36, knockback = kb(14, 20, true), -- (td)
					telegraph = tele("Line", 14, 4, 40),
					params = { dmgModel = "total", bodies = 1, hpEach = 30, lifeF = 360, speed = 0, ai = "line", -- (td) K09
						line = { studs = 40, frames = 16, width = 4 }, spawnOffsets = { v3(0, 0, 3) }, maxHitsPerTarget = 1,
						ragdollFrames = 48, model = "model.monarch.summon_robot" },
				}),
				M("monarch.a3", { -- canon: GDD §7.3.9 A3
					behavior = "Zone", tags = { "Zone" }, damage = 8, cooldown = 14, -- GDD §7.3.9
					startup = 14, active = 1, recovery = 20, guardDamage = 30, hitstun = 60, -- stunned 60f (GDD)
					telegraph = tele("Circle", 14, 30, 30), -- within 15 studs (GDD)
					params = { anchor = "self", shape = "circle", radius = 15, armF = 0, lifeF = 0,
						bursts = { { atF = 0, damage = 8, hitstun = 60, stun = "float" } } }, -- float (GDD)
				}),
				M("monarch.a4", { -- canon: GDD §7.3.9 A4
					behavior = "Summon", tags = { "Summon", "Ultimate" }, damage = 6, cooldown = 25, -- ≤ 24 total (GDD §7.3.9)
					startup = 20, active = 1, recovery = 20, guardDamage = 20, hitstun = 30, knockback = kb(6, 10, false), -- (td)
					telegraph = tele("Circle", 20, 16, 16),
					params = { dmgModel = "perHit", bodies = 1, hpEach = 0, untargetable = true, lifeF = 360, -- 6 s set piece (GDD)
						speed = 0, ai = "marked", spawnOffsets = { v3(0, 0, 8) },
						attack = { kind = "beam", range = 70, beamRange = 70, beamWidth = 4, startupF = 24, everyF = 72, firstAtF = 30 }, -- (td)
						maxHitsPerTarget = 4, model = "model.monarch.summon_guardian" }, -- 4 × 6 = 24
				}),
			},
		},
		m1 = { anim = "anim.monarch.m1", vfx = "vfx.monarch.m1", sfx = "sfx.monarch.m1" },
		kitProps = { "model.monarch.gear_crown", "model.monarch.gear_coat" },
	}
	return {
		def = def,
		hooks = {}, -- Scientific Method lives in Styles.serverHook with server memory (§4.3.6), keyed by styleId "monarch"
		moment = { seq = { "monarch.1:hit", "monarch.4:hit" }, windowF = 180, sameVictim = true }, -- GDD §7.3.9
		anim = {
			m1 = { windup = "punch_wind", strike = "punch_r",
				alt = { "punch_r", "punch_l", "point", "punch_r", "punch_l", "point", "kick", "cast_push", "windup_high", "slash_down", "punch_l", "punch_l" } },
			q = { windup = "ready", strike = "point" },
			["1"] = { windup = "point", strike = "cast_push" },
			["2"] = { windup = "cast_raise", strike = "point" },
			["3"] = { windup = "windup_high", strike = "throw" },
			["4"] = { windup = "cast_raise", strike = "cast_push", face = { windup = "grit", active = "shout", recovery = "smile" } },
			a1 = { windup = "cast_raise", strike = "point", alt = { "point", "cast_push" } },
			a2 = { windup = "point", strike = "cast_push" },
			a3 = { windup = "cast_raise", strike = "cast_raise" },
			a4 = { windup = "cast_raise", strike = "point", face = { windup = "awaken", active = "shout", recovery = "smile" } },
			awaken = { windup = "crouch", strike = "cast_raise", face = { windup = "grit", active = "awaken", recovery = "smile" } }, -- anim.<style>.awaken (72f, §4.6.1)
		},
	}
end
```

#### 4.5.5 `rocker` (GDD §7.3.12, Epic, VampireRocker)

```lua
--!strict
-- src/Shared/Styles/rocker.luau — rocker kit. Realm: shared. Pure: load. Spec §4.5.5
local Kit = require("../Behaviors/Buff")

return function(K: Kit.KitBuilder): Kit.StyleModule
	local M, box, kb, tele, v3 = K.move, K.box, K.kb, K.tele, K.v3
	local def = {
		id = "rocker", rarity = "epic", archetype = "VampireRocker", implemented = true, hp = 100, -- GDD §7.2, §6.2
		palette = { primary = K.rgb(159, 180, 199), secondary = K.rgb(27, 27, 47), accent = K.rgb(229, 57, 53) },
		passive = { id = "rocker.passive", params = { enderHeal = 2, hoverSec = 3, hoverMode = "hold" } }, -- GDD §7.3.12
		quirk = M("rocker.q", { -- canon: GDD §7.3.12 Quirk
			behavior = "Mobility", tags = { "Mobility" }, damage = 0, cooldown = 5, -- GDD §7.3.12
			startup = 1, active = 15, recovery = 8, -- (td)
			params = { mode = "flight", studs = 25, iFrames = { from = 1, to = 6 }, air = true, pitchClampDeg = 30 }, -- 25 studs, i-frames 1–6 (GDD)
		}),
		moves = {
			M("rocker.1", { -- canon: GDD §7.3.12 row 1; frame data from GDD Appendix A
				behavior = "Rush", tags = { "Rush", "Extender", "Music" }, damage = 9, cooldown = 8, -- GDD §7.3.12
				startup = 10, active = 5, recovery = 20, guardDamage = 30, chip = 0.2, hitstun = 30, -- GDD App. A
				knockback = kb(18, 10, false), hitbox = box(7, 5, 6, 4), telegraph = tele("Arc", 8, 7, 6), -- GDD App. A
				params = { travel = { from = 1, to = "startup", studs = 4 } }, -- (td)
			}),
			M("rocker.2", { -- canon: GDD §7.3.12 row 2
				behavior = "Buff", tags = { "Mobility", "Buff" }, damage = 0, cooldown = 14, -- "+5" is a buff (GDD §7.3.12)
				startup = 6, active = 1, recovery = 6, -- (td)
				params = { target = "self", buffs = { { kind = "iv", frames = 120, n = 1 }, { kind = "nh", frames = 180, n = 5 } } }, -- 2 s; +5 (GDD); validity 180f (td)
			}),
			M("rocker.3", { -- canon: GDD §7.3.12 row 3
				behavior = "Grab", tags = { "Grab" }, damage = 10, cooldown = 14, -- GDD §7.3.12
				startup = 9, active = 4, recovery = 22, hitbox = box(4, 5, 6), -- (td)
				params = { groundOnly = true, holdF = 48, offset = v3(0, 0, -2.5), sub = 4, finish = "release", -- (td)
					ticks = { { at = 12, damage = 3, healSelf = 2 }, { at = 24, damage = 3, healSelf = 2 }, { at = 36, damage = 4, healSelf = 1 } }, -- 3 ticks, heal 5 (GDD)
					finishKb = kb(8, 10, false), finishHitstun = 24, finishRecovery = 14 }, -- (td)
			}),
			M("rocker.4", { -- canon: GDD §7.3.12 row 4
				behavior = "Zone", tags = { "Zone" }, signature = true, damage = 12, cooldown = 16, -- GDD §7.3.12
				startup = 14, active = 4, recovery = 24, guardDamage = 40, hitstun = 48, knockback = kb(10, 0, false), -- 48f stumble (GDD)
				telegraph = tele("Cone", 14, 32, 16), -- (td) 90° cone, 16 studs
				params = { anchor = "self", shape = "cone", radius = 16, coneDeg = 90, armF = 0, lifeF = 0,
					bursts = { { atF = 0, damage = 12, hitstun = 48, stun = "stumble", knockback = kb(10, 0, false) } } },
			}),
		},
		awakening = {
			id = "rocker.awaken", durationSec = 60, hurtboxScale = 1, modifiers = {}, -- GDD §6.9
			moves = {
				M("rocker.a1", { -- canon: GDD §7.3.12 A1
					behavior = "Projectile", tags = { "Projectile" }, damage = 8, cooldown = 12, -- GDD §7.3.12
					startup = 12, active = 4, recovery = 18, guardDamage = 20, hitstun = 18, -- (td)
					params = { kind = "beam", range = 30, width = 3, extendF = 0, reflectable = false, -- gaze (td)
						onHit = { debuffs = { { kind = "sl", frames = 60, n = 50 }, { kind = "si", frames = 60, n = 1 } } } }, -- −50%, no moves 60f (GDD)
				}),
				M("rocker.a2", { -- canon: GDD §7.3.12 A2
					behavior = "Rush", tags = { "Rush" }, damage = 16, cooldown = 9, -- GDD §7.3.12
					startup = 14, active = 6, recovery = 22, guardDamage = 45, hitstun = 40, -- (td)
					knockback = kb(16, 30, true), hitbox = box(7, 6, 7), telegraph = tele("Circle", 14, 8, 8),
					params = { travel = { from = 3, to = "startup", studs = 22, arc = "leap", height = 7 }, stopOnHit = true, -- (td)
						teleFwd = 22, ragdollFrames = 60 },
				}),
				M("rocker.a3", { -- canon: GDD §7.3.12 A3
					behavior = "Buff", tags = { "Buff" }, damage = 0, cooldown = 16, -- GDD §7.3.12
					startup = 8, active = 1, recovery = 8, -- (td)
					params = { target = "self", buffs = { { kind = "ff", frames = 300, n = 1 } } }, -- 5 s, 6-hit chain (GDD)
				}),
				M("rocker.a4", { -- canon: GDD §7.3.12 A4
					behavior = "Grab", tags = { "Grab", "Ultimate" }, damage = 26, cooldown = 25, -- GDD §7.3.12
					startup = 18, active = 20, recovery = 24, telegraph = tele("Line", 18, 8, 40), -- (td)
					params = { groundOnly = false, sweep = { studs = 40, width = 8, height = 8 }, maxTargets = 3, -- swoops through a line (GDD)
						holdF = 80, offset = v3(0, -4, 0), sub = 5, carry = { riseStuds = 20, riseF = 60 }, -- carries upward (GDD); (td)
						ticks = { { at = 78, damage = 26 } }, finish = "drop", finishKb = kb(6, 0, true), ragdollFrames = 60,
						finishRecovery = 20, cinematicFrames = 80 }, -- ≤ 120 (GDD §6.8)
				}),
			},
		},
		m1 = { anim = "anim.rocker.m1", vfx = "vfx.rocker.m1", sfx = "sfx.rocker.m1" },
		kitProps = { "model.rocker.gear_bass", "model.rocker.gear_hair" },
	}
	return {
		def = def,
		hooks = {
			onHitDealt = function(h, info)
				if info.isEnder and info.outcome == "hit" then
					return { heal = h.style.passiveParams.enderHeal } -- GDD §7.3.12 Color Drain
				end
				return nil
			end,
			onAwakenEnd = function(h)
				return { patch = { Buffs = Kit.statusRemove(h.s.Buffs, "ff") } } -- frenzy ends with the Awakening (td); helper in §4.2.0.3
			end,
		},
		moment = { seq = { "rocker.q:use", "rocker.2:use", "rocker.3:hit" }, windowF = 360, sameVictim = true }, -- GDD §7.3.12
		anim = {
			m1 = { windup = "windup_side", strike = "slash_side",
				alt = { "slash_side", "slash_side", "slash_down", "slash_side", "slash_side", "slash_down", "kick", "windup_high", "windup_high", "slash_down", "slash_side", "slash_side" } },
			q = { windup = "crouch", strike = "fly" },
			["1"] = { windup = "windup_side", strike = "slash_side" },
			["2"] = { windup = "ready", strike = "buff", face = { windup = "smile", active = "smile", recovery = "neutral" } },
			["3"] = { windup = "grab_reach", strike = "grab_hold" },
			["4"] = { windup = "crouch", strike = "cast_push", face = { windup = "grit", active = "shout", recovery = "smile" } },
			a1 = { windup = "ready", strike = "point" },
			a2 = { windup = "crouch", mid = "leap", strike = "slash_down" },
			a3 = { windup = "ready", strike = "buff" },
			a4 = { windup = "crouch", strike = "fly", finish = "slash_down", face = { windup = "awaken", active = "shout", recovery = "grit" } },
			awaken = { windup = "crouch", strike = "cast_raise", face = { windup = "grit", active = "awaken", recovery = "smile" } }, -- anim.<style>.awaken (72f, §4.6.1)
		},
	}
end
```

#### 4.5.6 `frost` (GDD §7.3.13, Epic, IceWizard)

```lua
--!strict
-- src/Shared/Styles/frost.luau — frost kit. Realm: shared. Pure: load. Spec §4.5.6
local Kit = require("../Behaviors/Buff")

return function(K: Kit.KitBuilder): Kit.StyleModule
	local M, kb, tele, v3 = K.move, K.kb, K.tele, K.v3
	local def = {
		id = "frost", rarity = "epic", archetype = "IceWizard", implemented = true, hp = 100, -- GDD §7.2, §6.2
		palette = { primary = K.rgb(79, 107, 216), secondary = K.rgb(245, 247, 250), accent = K.rgb(127, 227, 255) },
		passive = { id = "frost.passive", params = { hoverSec = 2.5, hoverMode = "hold" } }, -- GDD §7.3.13 Beard Flight
		quirk = M("frost.q", { -- canon: GDD §7.3.13 Quirk
			behavior = "Summon", tags = { "Summon" }, damage = 3, cooldown = 6, -- GDD §7.3.13
			startup = 10, active = 1, recovery = 10, guardDamage = 10, hitstun = 18, -- 18f flinch (GDD)
			params = { dmgModel = "perHit", bodies = 1, hpEach = 20, lifeF = 360, speed = 14, ai = "chase", leash = 40, -- (td) K09
				spawnOffsets = { v3(2, 0, -2) }, attack = { range = 4, startupF = 12, everyF = 0, size = v3(3, 4, 4), stun = "flinch" },
				maxHitsPerTarget = 1, once = true, model = "model.frost.summon_penguin" },
		}),
		moves = {
			M("frost.1", { -- canon: GDD §7.3.13 row 1
				behavior = "Projectile", tags = { "Projectile" }, damage = 8, cooldown = 6, -- GDD §7.3.13
				startup = 8, active = 2, recovery = 15, guardDamage = 22, hitstun = 28, -- (td)
				params = { kind = "bullet", speed = 120, range = 60, radius = 1.2, pierce = 0, reflectable = true, -- (td)
					onHit = { debuffs = { { kind = "sl", frames = 120, n = 20 } } }, extender = true }, -- −20% for 2 s (GDD)
			}),
			M("frost.2", { -- canon: GDD §7.3.13 row 2
				behavior = "Zone", tags = { "Zone" }, damage = 8, cooldown = 14, -- GDD §7.3.13
				startup = 10, active = 1, recovery = 14, guardDamage = 30, hitstun = 60, -- frozen 60f (GDD)
				telegraph = tele("Circle", 30, 6, 6), -- 0.5 s telegraph, 6-stud area (GDD)
				params = { anchor = "aim", aimRange = 35, shape = "circle", radius = 3, armF = 30, lifeF = 60,
					bursts = { { atF = 0, damage = 8, hitstun = 60, stun = "freeze" } }, teleAnchor = "carrier" },
			}),
			M("frost.3", { -- canon: GDD §7.3.13 row 3
				behavior = "Mobility", tags = { "Mobility" }, damage = 0, cooldown = 9, -- GDD §7.3.13
				startup = 4, active = 18, recovery = 10, -- (td)
				params = { mode = "slide", studs = 30, trail = true }, -- 30 studs (GDD)
			}),
			M("frost.4", { -- canon: GDD §7.3.13 row 4
				behavior = "Summon", tags = { "Summon" }, signature = true, damage = 8, cooldown = 20, -- 8 per smash (GDD §7.3.13)
				startup = 16, active = 1, recovery = 18, guardDamage = 25, hitstun = 32, knockback = kb(10, 20, false), -- (td)
				telegraph = tele("Circle", 16, 8, 8),
				params = { dmgModel = "perHit", bodies = 1, hpEach = 40, lifeF = 600, speed = 12, ai = "chase", leash = 50, -- 40 HP, 10 s (GDD)
					spawnOffsets = { v3(0, 0, -4) }, attack = { range = 7, startupF = 16, everyF = 90, size = v3(8, 7, 8), tele = true }, -- (td)
					maxHitsPerTarget = 2, model = "model.frost.summon_golem" }, -- 2 × 8 = 16 ≤ 18
			}),
		},
		awakening = {
			id = "frost.awaken", durationSec = 60, hurtboxScale = 1, modifiers = {}, -- GDD §6.9
			moves = {
				M("frost.a1", { -- canon: GDD §7.3.13 A1
					behavior = "Zone", tags = { "Zone" }, damage = 10, cooldown = 14, chip = 0, parryable = false, -- ≤ 10 (GDD)
					startup = 14, active = 1, recovery = 16, telegraph = tele("Circle", 20, 20, 20), -- 20-stud storm (GDD)
					params = { anchor = "aim", aimRange = 30, shape = "circle", radius = 10, armF = 20, lifeF = 300, -- 5 s (GDD)
						tick = { everyF = 60, damage = 2, count = 5 }, stay = { kind = "sl", frames = 12, n = 30 }, -- 2 dmg/s, −30% (GDD)
						maxHitsPerTarget = 5, teleAnchor = "carrier" },
				}),
				M("frost.a2", { -- canon: GDD §7.3.13 A2
					behavior = "Projectile", tags = { "Projectile" }, damage = 14, cooldown = 10, -- GDD §7.3.13
					startup = 14, active = 2, recovery = 18, guardDamage = 40, hitstun = 36, knockback = kb(16, 25, true), -- (td)
					telegraph = tele("Line", 14, 8, 45),
					params = { kind = "wave", speed = 45, range = 45, width = 8, height = 4, pierce = -1, reflectable = false, ragdollFrames = 48 },
				}),
				M("frost.a3", { -- canon: GDD §7.3.13 A3
					behavior = "Zone", tags = { "Zone" }, damage = 3, cooldown = 12, guardDamage = 10, hitstun = 0, -- 3 per touch (GDD)
					startup = 8, active = 1, recovery = 10, telegraph = tele("Circle", 8, 12, 12), -- (td)
					params = { dmgModel = "perHit", anchor = "selfFollow", shape = "ring", radius = 6, innerRadius = 3, armF = 0,
						lifeF = 240, tick = { everyF = 30, damage = 3, blockable = true }, maxHitsPerTarget = 6 }, -- 4 s (GDD); 6 × 3 = 18 ≤ 20
				}),
				M("frost.a4", { -- canon: GDD §7.3.13 A4
					behavior = "Zone", tags = { "AoE", "Ultimate" }, damage = 24, cooldown = 25, -- GDD §7.3.13
					startup = 16, active = 1, recovery = 20, guardDamage = 60, hitstun = 40, -- (td)
					telegraph = tele("Circle", 45, 30, 30), -- within 15 studs (GDD); 0.75 s fuse (td)
					params = { anchor = "aim", aimRange = 45, shape = "circle", radius = 15, armF = 45, lifeF = 90,
						bursts = { { atF = 0, damage = 24, hitstun = 40, knockback = kb(12, 90, false, true) } }, hangFrames = 20, -- launches (GDD)
						teleAnchor = "carrier" },
				}),
			},
		},
		m1 = { anim = "anim.frost.m1", vfx = "vfx.frost.m1", sfx = "sfx.frost.m1" },
		kitProps = { "model.frost.gear_crown", "model.frost.gear_beard" },
	}
	return {
		def = def,
		hooks = {},
		moment = { seq = { "frost.2:hit", "frost.q:hit" }, windowF = 180, sameVictim = true }, -- GDD §7.3.13
		anim = {
			m1 = { windup = "cast_push", strike = "point",
				alt = { "point", "cast_push", "point", "point", "cast_push", "point", "kick", "cast_push", "cast_raise", "slash_down", "point", "point" } },
			q = { windup = "point", strike = "point", face = { windup = "shout", active = "smile", recovery = "smile" } },
			["1"] = { windup = "cast_raise", strike = "point" },
			["2"] = { windup = "cast_raise", strike = "cast_push" },
			["3"] = { windup = "crouch", strike = "slide" },
			["4"] = { windup = "cast_raise", strike = "cast_push", face = { windup = "grit", active = "shout", recovery = "smile" } },
			a1 = { windup = "cast_raise", strike = "cast_raise" },
			a2 = { windup = "cast_raise", strike = "cast_push" },
			a3 = { windup = "ready", strike = "spin_a", alt = { "spin_a", "spin_b" } },
			a4 = { windup = "cast_raise", strike = "cast_push", face = { windup = "awaken", active = "shout", recovery = "grit" } },
			awaken = { windup = "crouch", strike = "cast_raise", face = { windup = "grit", active = "awaken", recovery = "smile" } }, -- anim.<style>.awaken (72f, §4.6.1)
		},
	}
end
```

#### 4.5.7 `decay` (GDD §7.3.15, Legendary, DecayLich)

```lua
--!strict
-- src/Shared/Styles/decay.luau — decay kit. Realm: shared. Pure: load. Spec §4.5.7
local Kit = require("../Behaviors/Buff")

return function(K: Kit.KitBuilder): Kit.StyleModule
	local M, box, kb, tele, v3 = K.move, K.box, K.kb, K.tele, K.v3
	local def = {
		id = "decay", rarity = "legendary", archetype = "DecayLich", implemented = true, hp = 100, -- GDD §7.2, §6.2
		palette = { primary = K.rgb(124, 255, 79), secondary = K.rgb(30, 42, 30), accent = K.rgb(216, 210, 184) },
		passive = { id = "decay.passive", params = { maxStacks = 5, stackFrames = 480, perStack = 1, maxBonus = 5 } }, -- GDD §7.3.15
		quirk = M("decay.q", { -- canon: GDD §7.3.15 Quirk
			behavior = "Buff", tags = { "Debuff" }, damage = 0, cooldown = 8, -- GDD §7.3.15
			startup = 8, active = 1, recovery = 10, -- (td)
			params = { target = "front", range = 20, width = 8, debuffs = { { kind = "wh", frames = 240, n = 1 } } }, -- 20 studs, 4 s (GDD)
		}),
		moves = {
			M("decay.1", { -- canon: GDD §7.3.15 row 1
				behavior = "Projectile", tags = { "Projectile" }, damage = 8, cooldown = 6, -- GDD §7.3.15
				startup = 8, active = 2, recovery = 15, guardDamage = 22, hitstun = 28, -- (td)
				params = { kind = "bullet", speed = 100, range = 60, radius = 1.3, pierce = 0, reflectable = true, -- (td)
					decayStacks = 1, extender = true }, -- +1 Decay (GDD)
			}),
			M("decay.2", { -- canon: GDD §7.3.15 row 2
				behavior = "Grab", tags = { "Grab" }, damage = 12, cooldown = 12, -- GDD §7.3.15
				startup = 10, active = 5, recovery = 24, hitbox = box(4.5, 5, 6.5), telegraph = tele("Arc", 10, 5, 7), -- (td)
				params = { groundOnly = true, holdF = 36, offset = v3(0, 1, -3), sub = 6, ticks = { { at = 30, damage = 12 } },
					finish = "throw", throwDir = "facing", finishKb = kb(12, 10, true), ragdollFrames = 54, finishRecovery = 16,
					decayStacks = 2 }, -- +2 Decay (GDD)
			}),
			M("decay.3", { -- canon: GDD §7.3.15 row 3
				behavior = "Zone", tags = { "Zone" }, damage = 12, cooldown = 16, chip = 0, parryable = false, -- ≤ 12 (GDD)
				startup = 12, active = 1, recovery = 14, telegraph = tele("Circle", 12, 10, 10), -- 10-stud pool (GDD)
				params = { anchor = "self", shape = "circle", radius = 5, armF = 0, lifeF = 360, -- 6 s (GDD)
					tick = { everyF = 60, damage = 2, count = 6 }, ownerTick = { everyF = 60, heal = 1 }, maxHitsPerTarget = 6 }, -- 2 dmg/s; heal 1/s (GDD)
			}),
			M("decay.4", { -- canon: GDD §7.3.15 row 4
				behavior = "Projectile", tags = { "Projectile" }, signature = true, damage = 12, cooldown = 20, -- GDD §7.3.15
				startup = 10, active = 2, recovery = 22, guardDamage = 30, hitstun = 72, -- 10f telegraph; possessed 72f (GDD)
				telegraph = tele("Line", 10, 3, 40),
				params = { kind = "bullet", speed = 160, range = 40, radius = 1.5, pierce = 0, reflectable = true, -- (td)
					onHit = { stun = "possess", pull = { stopStuds = 5, frames = 72 }, meterDrain = 15 } }, -- pulled, −15 meter (GDD)
			}),
		},
		awakening = {
			id = "decay.awaken", durationSec = 60, hurtboxScale = 1.3, -- GDD §6.9, §6.2
			modifiers = { visualScale = 1.6, aura = "vfx.decay.aura" }, -- towering form, decay aura (GDD §7.3.15)
			moves = {
				M("decay.a1", { -- canon: GDD §7.3.15 A1
					behavior = "Projectile", tags = { "Projectile" }, damage = 16, cooldown = 10, -- GDD §7.3.15
					startup = 14, active = 2, recovery = 20, guardDamage = 45, hitstun = 36, knockback = kb(14, 15, true), -- (td)
					telegraph = tele("Cone", 14, 34, 30),
					params = { kind = "wave", speed = 60, range = 30, width = 4, widthEnd = 34, height = 6, pierce = -1, -- cone (GDD); (td)
						reflectable = false, decayStacks = 2, ragdollFrames = 48 }, -- +2 Decay (GDD)
				}),
				M("decay.a2", { -- canon: GDD §7.3.15 A2
					behavior = "Zone", tags = { "Zone" }, damage = 15, cooldown = 14, chip = 0, parryable = false, -- ≤ 15 (GDD)
					startup = 12, active = 1, recovery = 16, telegraph = tele("Circle", 18, 14, 14), -- (td) size
					params = { anchor = "aim", aimRange = 30, shape = "circle", radius = 7, armF = 18, lifeF = 300, -- 5 s (GDD)
						tick = { everyF = 60, damage = 3, count = 5 }, stay = { kind = "sl", frames = 12, n = 40 }, -- 3 dmg/s, −40% (GDD)
						maxHitsPerTarget = 5, teleAnchor = "carrier" },
				}),
				M("decay.a3", { -- canon: GDD §7.3.15 A3
					behavior = "Projectile", tags = { "Projectile" }, damage = 15, cooldown = 10, -- GDD §7.3.15
					startup = 12, active = 9, recovery = 18, guardDamage = 36, hitstun = 24, telegraph = tele("Line", 12, 3, 40), -- (td)
					params = { kind = "homing", count = 3, everyF = 4, speed = 70, range = 60, radius = 1.2, homingDegPerSec = 120, -- 3 skulls (GDD)
						hits = { 5, 5, 5 }, pierce = 0, reflectable = true, decayStacks = 1 },
				}),
				M("decay.a4", { -- canon: GDD §7.3.15 A4
					behavior = "RealmCast", tags = { "Realm" }, damage = 0, cooldown = 0, -- rules; once per Awakening (GDD §6.10)
					startup = 90, active = 1, recovery = 0, -- 90f cast (GDD §6.10)
					telegraph = tele("Circle", 90, 80, 80), params = {}, -- 40-stud radius ring (GDD §6.10)
				}),
			},
		},
		m1 = { anim = "anim.decay.m1", vfx = "vfx.decay.m1", sfx = "sfx.decay.m1" },
		kitProps = { "model.decay.gear_horns", "model.decay.gear_cape" },
	}
	return {
		def = def,
		hooks = {
			onHitDealt = function(h, info)
				if info.outcome ~= "hit" or not info.firstOfUse or info.tick then
					return nil -- blocked hits, repeat hits of one use and zone ticks add no stacks (td)
				end
				local n = 0
				if info.kind == "ender" then
					n = 1 -- M1 family: Ender only (td, §4.11 Q1)
				elseif info.kind ~= "m1" and info.kind ~= "airm1" and info.kind ~= "dashm1" then
					local mv = h.moves(info.moveId)
					n = if mv ~= nil and mv.params ~= nil and mv.params.decayStacks ~= nil then mv.params.decayStacks else 1 -- GDD §7.3.15
				end
				if n <= 0 then
					return nil
				end
				return { debuffsOnVictim = { { kind = "dc", frames = h.style.passiveParams.stackFrames, n = n } } } -- 8 s (GDD §7.3.15)
			end,
			onKO = function(h, victimRid, victimDebuffs)
				for _, st in victimDebuffs do
					if st.kind == "dc" and st.n >= h.style.passiveParams.maxStacks then -- GDD §7.3.15 Style Moment
						return { events = { { k = "style_moment", rid = h.rid, f = h.f, a = victimRid } } }
					end
				end
				return nil
			end,
		},
		moment = nil,
		anim = {
			m1 = { windup = "windup_side", strike = "slash_side",
				alt = { "slash_side", "slash_side", "cast_push", "slash_side", "slash_side", "cast_push", "kick", "slash_down", "cast_raise", "slash_down", "slash_side", "slash_side" } },
			q = { windup = "ready", strike = "point", face = { windup = "smile", active = "smile", recovery = "neutral" } },
			["1"] = { windup = "cast_raise", strike = "cast_push" },
			["2"] = { windup = "grab_reach", strike = "grab_hold", finish = "throw" },
			["3"] = { windup = "cast_raise", strike = "cast_raise" },
			["4"] = { windup = "point", strike = "cast_push", face = { windup = "grit", active = "shout", recovery = "smile" } },
			a1 = { windup = "cast_raise", strike = "cast_push" },
			a2 = { windup = "cast_raise", strike = "throw" },
			a3 = { windup = "cast_raise", strike = "point", alt = { "point", "cast_push" } },
			a4 = { windup = "cast_raise", strike = "cast_raise", face = { windup = "awaken", active = "awaken", recovery = "smile" } },
			awaken = { windup = "crouch", strike = "cast_raise", face = { windup = "grit", active = "awaken", recovery = "smile" } }, -- anim.<style>.awaken (72f, §4.6.1)
		},
	}
end
```

Stack rule implemented by `decay.onHitDealt` (`h.moves` is `Styles.move`, passed in `HookCtx`, §4.3.1):

| Hit (outcome `"hit"`, `firstOfUse`) | Stacks added |
|---|---|
| `info.kind == "ender"` | 1 |
| Other M1-family kinds (`m1`, `airm1`, `dashm1`) | 0 |
| Zone `tick` hits | 0 |
| Any other move hit | `move.params.decayStacks or 1` (decay.1: 1, decay.2: 2, decay.a1: 2, decay.a3: 1) |

#### 4.5.8 Stub styles (12) and the rarity table

Stubs live in `Styles/init.luau`:

```lua
local STUBS: { { id: Types.StyleId, rarity: Types.Rarity, archetype: string, p: { number }, s: { number }, a: { number } } } = {
	{ id = "sentry",   rarity = "common",    archetype = "ComicSpearman",       p = { 247, 210, 62 },  s = { 255, 246, 213 }, a = { 192, 57, 43 } },
	{ id = "granny",   rarity = "common",    archetype = "BakerGranny",         p = { 139, 195, 74 },  s = { 248, 187, 208 }, a = { 128, 222, 234 } },
	{ id = "diva",     rarity = "common",    archetype = "FloatyDiva",          p = { 156, 106, 222 }, s = { 94, 53, 177 },   a = { 255, 213, 79 } },
	{ id = "tyrant",   rarity = "uncommon",  archetype = "ShriekingTyrant",     p = { 255, 241, 118 }, s = { 141, 141, 141 }, a = { 38, 50, 56 } },
	{ id = "console",  rarity = "uncommon",  archetype = "GadgetConsole",       p = { 91, 192, 190 },  s = { 44, 122, 123 },  a = { 255, 107, 107 } },
	{ id = "grappler", rarity = "uncommon",  archetype = "SuperStrongGrappler", p = { 200, 162, 122 }, s = { 93, 64, 55 },    a = { 236, 239, 241 } },
	{ id = "flame",    rarity = "rare",      archetype = "FireElemental",       p = { 255, 112, 67 },  s = { 255, 202, 40 },  a = { 216, 67, 21 } },
	{ id = "butler",   rarity = "rare",      archetype = "NecromancerButler",   p = { 250, 250, 250 }, s = { 229, 57, 53 },   a = { 33, 33, 33 } },
	{ id = "huntress", rarity = "epic",      archetype = "ForestArcherMage",    p = { 46, 125, 50 },   s = { 109, 76, 65 },   a = { 165, 214, 167 } },
	{ id = "demon",    rarity = "legendary", archetype = "DemonLord",           p = { 84, 110, 122 },  s = { 183, 28, 28 },   a = { 102, 187, 106 } },
	{ id = "wish",     rarity = "mythic",    archetype = "CosmicWishGranter",   p = { 224, 64, 251 },  s = { 64, 196, 255 },  a = { 156, 204, 101 } },
	{ id = "chaos",    rarity = "cosmic",    archetype = "GodOfChaos",          p = { 26, 26, 46 },    s = { 255, 0, 255 },   a = { 255, 215, 0 } },
}
-- Each stub: implemented = false, hp = 100, passive = { id = "<id>.passive", params = {} },
-- quirk = K.stubMove("<id>.q"), moves = { K.stubMove("<id>.1") .. "<id>.4" }, awakening = { id = "<id>.awaken",
-- durationSec = 60, hurtboxScale = 1, modifiers = {}, moves = { stubMove a1..a4 } } (a4 behavior "RealmCast", tags {"Realm"}
-- for legendary+), m1 = { anim = "anim.<id>.m1", vfx = "vfx.<id>.m1", sfx = "sfx.<id>.m1" }, kitProps = {}.
-- K.stubMove(id): behavior "Strike", tags {}, startup 1, active 1, recovery 0, damage 0, cooldown 0, parryable false,
-- guardDamage 0, chip 0, hitstun 0, keys derived as K.move.
```

Rarity table (S02; GDD §7.2, §8.3). Archetype strings are the PascalCase form of the GDD §7.2 "IP-neutral archetype" column.

| Rarity | Style IDs |
|---|---|
| starter | hero, stretch |
| common | sentry, granny, diva |
| uncommon | tyrant, console, grappler |
| rare | monarch, flame, butler |
| epic | rocker, frost, huntress |
| legendary | decay, demon |
| mythic | wish |
| cosmic | chaos |

#### 4.5.9 GDD cross-check table (`kits.spec.luau`)

`tests/core/kits.spec.luau` contains the expected values below as a literal table and asserts them against the registry (`kits.<style>.gdd_numbers`). Values in parentheses are `maxDamage`.

| Style | q dmg/CD | 1 | 2 | 3 | 4 ★ | a1 | a2 | a3 | a4 |
|---|---|---|---|---|---|---|---|---|---|
| hero | 0/3 | 8/7 | 11/9 | 6/12 | 14/16 (★4) | 14/8 | 16/12 | 12+6 (18)/12 | 26/25 |
| stretch | 0/6 | 7/6 | 15/16 (★2) | 0/12 | 9/13 | 14/8 | 12/11 | 10/12 | 28/25 |
| monarch | 0/8 | 7/7 | 3 per poke (18)/15 | 10/12 | 16/18 (★4) | 15/9 | 14/11 | 8/14 | 6 per shot (24)/25 |
| rocker | 0/5 | 9/8 | 0 (+5)/14 | 10/14 | 12/16 (★4) | 8/12 | 16/9 | 0/16 | 26/25 |
| frost | 3/6 | 8/6 | 8/14 | 0/9 | 8 per smash (16)/20 (★4) | 10/14 | 14/10 | 3 per touch (18)/12 | 24/25 |
| decay | 0/8 | 8/6 | 12/12 | 12/16 | 12/20 (★4) | 16/10 | 15/14 | 15/10 | 0/0 (Realm) |

**Verify:** Lune `kits.<style>.gdd_numbers` (6 tests), `kits.signature_slot`, `kits.extenders_resolve` (K04 resolves via tag or frame proof for every VS style).

---

### 4.6 Placeholder animation spec

#### 4.6.1 Generation algorithm (`BuildPlaceholderAnims`, Part 6 implements; data here)

For every implemented style the builder creates one `KeyframeSequence` under `ReplicatedStorage.Anim` per animation key:
- the 9 move slots (`q`, `1..4`, `a1..a4`); a RealmCast a4 uses key `anim.<style>.realm`;
- the 12 chassis M1/Ender clips of Part 3 §3.11.8 (`anim.<style>.m1_<sub>` for Sub 1, 2, 3, 11, 12, 13, 20; `anim.<style>.ender_<sub>` for Sub 1–5);
- `anim.<style>.awaken`.

That is 22 per style.

Instance and attributes:
- Name: the key's last segment prefixed by the style, e.g. `hero_4`, `hero_m1_11`, `decay_realm`.
- Attributes:
  - `AssetKey` = the key above;
  - `MoveId` (`<style>.m1` for chassis clips);
  - `Frames = S+A+R`;
  - `Gen = "placeholder-v1"`.
- Properties: `Loop = false`, `Priority = Enum.AnimationPriority.Action2` (Part 3 plays combat tracks at `Action2`, Awakening/RealmCast at `Action4`; §3.11.8). Both are `AnimationClip` members [Docs `AnimationClip.yaml`].

Frame data:
- Moves use the MoveDef. For `hero.1` the builder uses mode 0, the spawn mode.
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

The notation "`Face:<expr>`" (agent-assets §7.2) means `Name = "Face"`, `Value = "<expr>"`. Expression names are from the 24-key face set: `neutral, smile, grit, shout, awaken, …` (Part 6). Listeners use `AnimationTrack:GetMarkerReachedSignal(name)`, which is not Simulation-Access. It is used **only** in RenderController, never for gameplay timing (Part 3 §3.1.5).

Studio registration: `Assets.registerStudioClips(ReplicatedStorage.Anim)` (Part 2 §2.9.1) calls `AnimationClipProvider:RegisterAnimationClip` per sequence. `KeyframeSequenceProvider` is deprecated and banned. Online and multi-peer tests need published IDs (§0.5, H-08).

**Worked example** (`hero.4`, S=16, A=8, R=24, total 48 f):
- K1 0 `neutral` (markers Windup, Face=shout);
- K2 8 `kick_wind`;
- K3 15 `kick_wind` (Constant);
- K4 16 `kick` (markers Active, Face=shout, VFX=`vfx.hero.4`, SFX=`sfx.hero.4`);
- K5 23 `kick` (Constant), plus a marker keyframe at 24 (Recovery, Face=grit);
- K6 48 `neutral`.

Length = 48/60 = 0.8 s.

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
3. walk `KeyframeSequence:GetKeyframes()` / `Keyframe:GetMarkers()` and compare against the placeholder spec recomputed from the MoveDef.

`anim.placeholders_built` asserts that every MoveDef and M1 variant key resolves to a sequence whose `Frames` attribute equals S+A+R. It also asserts that marker counts match.

`tools/export_framedata.luau` (Lune; Part 3/4) also writes `docs/reference/framedata.json` with, per move:
- `{moveId, S, A, R, total, markers: [{name, frame}], onHitAdv, onBlockAdv}`;
- `onHitAdv = hitstun − (A − 1) − R` and `onBlockAdv = blockstun − (A − 1) − R` (GDD Appendix B).

Animators use it as the timing sheet.

**Verify:** Lune `kits.anim_specs_complete` (K24); scenarios `anim.placeholders_built` and `anim.timing_lock`; MCP `search_game_tree` under `ReplicatedStorage.Anim` lists 6 × 22 = 132 kit sequences (Part 6 adds the `anim.common.*` clips).

---

### 4.7 Presentation mapping

#### 4.7.1 Palettes

The palette feeds the always-on Style Tell: nameplate chip = `primary`; VFX tint core = `accent`, mid = `primary`, ink rim `#1A1A1A` (GDD §13.4 three tones + ink; Part 6 VfxController). Graybox character-look bodies use primary (torso), secondary (limbs) and accent (props).

| Style | Primary | Secondary | Accent | Shape language (GDD §13.4) |
|---|---|---|---|---|
| hero | (63,169,245) | (245,245,240) | (242,193,78) | crisp stars, slash arcs |
| stretch | (242,163,58) | (185,119,46) | (255,231,179) | rubbery rings, boing lines |
| monarch | (244,143,177) | (216,27,96) | (127,216,190) | hex grids, bubbles |
| rocker | (159,180,199) | (27,27,47) | (229,57,53) | bat wisps, red-to-grey drain |
| frost | (79,107,216) | (245,247,250) | (127,227,255) | zig-zag lightning, snowflake sparks |
| decay | (124,255,79) | (30,42,30) | (216,210,184) | decay petals, green smoke |
| stubs | §4.5.8 | | | — |

S06 holds for the VS set. The closest pair is hero vs frost primaries, distance ≈ 70.

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
| `VFX_Telegraph_Arc` | Flat Neon wedge-ring segment (CSG: cylinder `SubtractAsync` cylinder, then `IntersectAsync` with a wedge), 0.05 thick, telegraph colour, transparency 0.35, dark outline copy 0.4 studs larger |
| `VFX_Projectile_Orb` | Neon `Ball` 1.2 studs + `Trail` (no texture, Lifetime 0.15, WidthScale 1→0) between two attachments ±0.5 studs |
| `VFX_Zone_Disc` | Flat Neon cylinder (height 0.1) at transparency 0.6 + ring edge (cylinder subtract) at 0.2; sized by carrier `r` |
| `VFX_Wave_Wall` | Neon block sized from carrier `w`/`h`, transparency 0.4, front face brighter (second block 0.2 thick) |
| `VFX_Encase_Block` | `Ice` material block 6×7×6, transparency 0.35 (freeze status and `frost.2`) |
| `VFX_Stealth_Shimmer` | `ParticleEmitter` default sparkles texture, Rate 6, Lifetime 0.4, Size 0.6, LockedToPart |
| `VFX_Status_Pips` | `BillboardGui` + `TextLabel` with `●` × n in the status colour, `LuckiestGuy` font, offset +3.5 studs (decay stacks; root/slow/silence glyphs `⌂ ≋ ⊘`) |

Per-move map (Recipe; tint by palette). The `m1` column is the swing trail.

| Style | m1 | q | 1 | 2 | 3 | 4 | a1 | a2 | a3 | a4 |
|---|---|---|---|---|---|---|---|---|---|---|
| hero | Slash_Smear | ImpactStar | Slash_Smear | ImpactStar | Beam_Magic | Impact_Heavy | Slash_Smear | Shockwave_Ring | Slash_Smear | Zone_Disc |
| stretch | Shockwave_Ring | Beam_Magic | Beam_Magic | Shockwave_Ring | Guard_Bubble | Beam_Magic | Shockwave_Ring | ImpactStar | Guard_Bubble | Zone_Disc |
| monarch | ImpactStar | Status_Pips | Projectile_Orb | ImpactStar | Zone_Disc | Beam_Magic | Projectile_Orb | Slash_Smear | Shockwave_Ring | Beam_Magic |
| rocker | Slash_Smear | Stealth_Shimmer | Slash_Smear | Stealth_Shimmer | Beam_Magic | Shockwave_Ring | Beam_Magic | Slash_Smear | Awaken_Aura | Slash_Smear |
| frost | Beam_Magic | ImpactStar | Projectile_Orb | Encase_Block | Slash_Smear | Shockwave_Ring | Zone_Disc | Wave_Wall | Zone_Disc | Shockwave_Ring |
| decay | Slash_Smear | Beam_Magic | Projectile_Orb | Beam_Magic | Zone_Disc | Projectile_Orb | Wave_Wall | Zone_Disc | Projectile_Orb | Realm_Dome |

Extra keys:
- `vfx.decay.aura` → `VFX_Awaken_Aura` (decay tint);
- `vfx.status.freeze` → `VFX_Encase_Block`;
- `vfx.status.root`, `.slow`, `.silence`, `.decay` → `VFX_Status_Pips`;
- `vfx.status.stealth` → `VFX_Stealth_Shimmer`;
- `vfx.status.ward` → `VFX_Guard_Bubble`.

Carriers use their move key as `vis` (projectile orb, zone disc, wave wall) or a `model.*` key (summons, §4.7.5).

Per-move emitter caps by rarity (GDD §13.5: Starter 3, Rare 5, Epic 6, Legendary 7) are enforced by Part 6. Every recipe above uses ≤ 2 emitters.

#### 4.7.3 SFX per move (built-in sounds, placeholder)

Manifest: `sfx.<style>.<slot> = { id = "rbxasset://sounds/<file>", status = "placeholder", source = "builtin", kind = "audio" }`. The 11 built-in files are listed in agent-assets §2.7 / §0.5. The playback speed lives in `Styles.SFX_SPEED[key]` (default 1), which SfxController (Part 6) applies, so moves sharing a file still differ. Part 8's release lint blocks `rbxasset://` audio in releases.

Abbreviations: `jump` = action_jump.mp3, `land` = action_jump_land.mp3, `swim` = action_swim.mp3, `fall` = action_falling.ogg, `rise` = action_get_up.mp3, `steps` = action_footsteps_plastic.mp3, `boom` = impact_explosion_03.mp3, `splat` = impact_water.mp3, `tick` = volume_slider.ogg.

| Style | m1 | q | 1 | 2 | 3 | 4 | a1 | a2 | a3 | a4 |
|---|---|---|---|---|---|---|---|---|---|---|
| hero | jump@1.8 | tick@0.8 | jump@1.0 | land@1.6 | swim@1.5 | boom@1.4 | jump@2.0 | boom@0.9 | jump@1.5 | boom@0.8 |
| stretch | land@1.8 | swim@1.3 | swim@1.7 | boom@1.1 | tick@0.6 | rise@0.9 | boom@0.7 | jump@0.7 | land@0.6 | fall@1.3 |
| monarch | land@1.5 | tick@1.4 | splat@1.6 | steps@1.2 | splat@0.8 | boom@1.2 | boom@1.6 | jump@1.3 | fall@0.8 | boom@0.6 |
| rocker | jump@1.4 | fall@1.8 | boom@1.8 | rise@1.4 | splat@1.2 | fall@0.6 | tick@0.5 | jump@0.8 | tick@1.6 | fall@0.9 |
| frost | tick@1.2 | steps@1.6 | tick@1.8 | splat@0.6 | swim@0.9 | land@0.6 | fall@1.0 | steps@1.9 | tick@2.0 | boom@0.7 |
| decay | rise@0.7 | fall@0.5 | splat@1.4 | rise@0.6 | splat@0.5 | fall@0.7 | boom@0.9 | splat@0.4 | fall@1.5 | boom@0.5 |

`SFX_SPEED` holds every `@` value above, keyed by `sfx.<style>.<slot>`. `oof.ogg` and `ouch.ogg` are voices and are not used for moves.

#### 4.7.4 Style Gear prop recipes (`ReplicatedStorage.Looks`, built by Part 6 `BuildTemplates`)

Each key `model.<style>.gear_<name>` maps to `tpl:Looks/Gear_<style>_<name>`.

Build rules:
- **Container.** Each template is an `Accessory` [Docs `Accessory.yaml`] whose `Handle` (a Part or union) contains an `Attachment` named for the body attachment it mates with. Names from the rigid-accessory spec [Docs `avatar/rigid-accessories/specifications.md`]: `HatAttachment`, `HairAttachment`, `FaceFrontAttachment`, `BodyBackAttachment`, `WaistBackAttachment`.
- **Unions.** `GeometryService:UnionAsync(part, parts, {SplitApart = false})` (yields; security None; returns an array [Docs `GeometryService.yaml`]) runs at **Edit time** inside the builder. Never at runtime.
- **Flags.** All parts: `CanCollide = false`, `CanQuery = false`, `CanTouch = false`, `Massless = true`, `CastShadow = false`, Material `SmoothPlastic`.
- **Colours.** From the palette unless stated.
- **Attaching.** Part 6's AppearanceController attaches gear client-side (`Humanoid:AddAccessory` on the look model). It is shown only when `ShowStyleGear` is on and Character Look is off (§0.4 Appearance).

| Key | Attach | Parts (shape, size in studs, colour, offset from Handle) |
|---|---|---|
| `model.hero.gear_hat` | HatAttachment | Union of Ball (1.35,1.2,1.35) white (Handle) + Ball (0.5,0.5,0.5) ×2 at (±0.45, 0.55, 0): the bear-ear hat |
| `model.hero.gear_pack` | BodyBackAttachment | Block (1.5,1.7,0.7) green (76,175,80) + Block flap (1.5,0.5,0.75) at (0,0.6,−0.05) darker green |
| `model.hero.gear_sword` | RightGripAttachment (verify in spike S-P4-5; fallback: `RigidConstraint` to `RightHand`) | Blade Block (0.15,3.2,0.45) + Wedge tip (0.15,0.6,0.45) at (0,1.9,0) + crossguard Block (0.9,0.15,0.3) at (0,0,0) + grip Cylinder (0.2,0.7) at (0,−0.45,0). Blade colour is set by the client from `Kit`: 0 → accent (242,193,78), 1 → (200,30,30) |
| `model.stretch.gear_ears` | HatAttachment | Block band (1.3,0.15,1.3) + Ball (0.45,0.8,0.3) ×2 at (±0.7,−0.2,0), secondary |
| `model.stretch.gear_tail` | WaistBackAttachment | Cylinder (0.25 × 1.2) rotated 45°, primary |
| `model.monarch.gear_crown` | HatAttachment | Cylinder (0.6 dia × 0.25) gold (242,193,78) + Wedge ×3 + Ball gem (0.2) blue (64,128,255) |
| `model.monarch.gear_coat` | BodyBackAttachment | Block (1.6,2.0,0.1) white (250,250,250) + Block collar (1.6,0.3,0.15) |
| `model.rocker.gear_bass` | BodyBackAttachment | Body: Block (1.6,2.2,0.3) accent unioned with Wedge ×2 (axe-blade flare); neck Block (0.25,2.4,0.2) secondary at (0,2.2,0); rotated 30° about Z (slung) |
| `model.rocker.gear_hair` | HairAttachment | Block (1.3,3.0,0.3) secondary at (0,−1.2,0.4) + Ball cap (1.3) |
| `model.frost.gear_crown` | HatAttachment | Cylinder ring (1.2 dia × 0.5) gold (242,193,78) minus Cylinder (1.0 dia) via `SubtractAsync` + Wedge spikes ×3 + Ball gems ×3 (200,30,30) |
| `model.frost.gear_beard` | FaceFrontAttachment | Wedge (1.4,1.8,0.6) white + Ball (1.2,0.8,0.6) at (0,0.5,0) |
| `model.decay.gear_horns` | HatAttachment | Cylinder (0.25 × 1.6) ×2 tilted ±35° + Wedge tips, accent (bone) |
| `model.decay.gear_cape` | BodyBackAttachment | Block (1.8,2.8,0.1) secondary minus 3 small Wedges along the bottom edge via `SubtractAsync` (tattered) |

GDD §12.3 rule: every style reads in pure black silhouette at 64 px. Part 6 scenario `look.gear_silhouette` takes `screen_capture`s for human review (H-10).

#### 4.7.5 Summon and set-piece models (`ReplicatedStorage.Looks`)

Visual only. The server carrier is invisible (Part 3). The client attaches the template to the carrier by `vis`.

| Key | Recipe |
|---|---|
| `model.monarch.summon_guard` | Block torso (1.4,2.2,1) yellow (247,210,62) + Ball head (1.2) + spear: Cylinder (0.15 × 4) + Wedge tip |
| `model.monarch.summon_robot` | Block body (1.6,2.6,1.2) grey (170,170,180) + Ball head (1.2) red + Block blade (0.2,2.5,0.4) |
| `model.monarch.summon_guardian` | Ball body (8) pink (244,143,177) + Ball eye (2) at (0,2,−3.5) white + Block base (6,3,6); 16 studs tall in total |
| `model.monarch.guardian_bust` | The guardian's upper half only, at 60% scale, behind the caster for `monarch.4` (set piece) |
| `model.frost.summon_penguin` | Ball body (1.2,1.5,1.1) black + Ball belly (0.9) white at the front + Wedge beak orange |
| `model.frost.summon_golem` | Block torso (3.5,4,2.5) Ice material (127,227,255) + Block arms (1.2,3,1.2) ×2 + Ball head (1.8) |

#### 4.7.6 Status visuals and stealth

RenderController (Part 6) reads `Buffs`/`Debuffs` (decoded with `Moves.statusDecode`) and `Hitstun` Sub:
- **Debuffs.** `rt`, `sl`, `si` and `dc` show `VFX_Status_Pips` above the head; `dc` shows n pips. The freeze flavour (`Hitstun` Sub 2) shows `VFX_Encase_Block`. `wh` sways the victim's own camera by ±1.5° at 0.5 Hz, scaled by the camera-shake setting and set to 0 with Reduced Motion (GDD §16.3).
- **Stealth `iv`.** For viewers who are enemies of the owner and farther than 12 studs (GDD §7.3.12), hide the look (`LocalTransparencyModifier = 1`) and the nameplate (td), keeping `VFX_Stealth_Shimmer`. The owner and allies see the owner at 50% transparency. Telegraphs and hit flashes are never hidden.

**Verify:** scenario `look.stealth_ranges` (two clients: an enemy at 20 studs sees the shimmer only; at 8 studs it sees the look).

---

### 4.8 Adding a new style (checklist the agent follows)

1. **Design input.** Copy the style's GDD §7.3 table into the task card. Confirm its rarity, archetype and Style Moment. Never invent damage or cooldown numbers.
2. **Behaviors.** Map each move to an existing behavior and params (§4.2). If a mechanic fits no behavior:
   - prefer new params on the closest behavior;
   - add a new `Moves.STATUS` kind only through §4.3.2 (2-char kind, `mods`, flags) plus a Lune test;
   - a new behavior module would change §0.3, so open a spine issue instead.
3. **Kit file.** Write `src/Shared/Styles/<id>.luau` following §4.5:
   - every number has a `GDD §` or `(td)` comment and a `canon:` pointer;
   - set `implemented = true` and remove the stub row from `STUBS` in `Styles/init.luau`;
   - add the id to `Types.IMPLEMENTED_M1G` (test aid).
4. **Passive.** Use data (Part 3 chassis keys, `lowHpThreshold`/`lowHp`, `hoverSec`) where possible; write hooks only for the rest (§4.3). Add the Style Moment (`moment` or a hook).
5. **Realm (legendary+).** Add `Realm.rules["<id>.a4"]` in `Core/Realm.luau` (§4.4.3 pattern) plus `kits.realm_<id>_rules` Lune tests.
6. **Statuses.** Add any new kinds to `Moves.STATUS` with tests in `kit_status.spec.luau`.
7. **Animation spec.** Fill `anim` for `m1, q, 1–4, a1–a4` using pose-library names (add poses to §4.6.2 if needed). Run `BuildPlaceholderAnims`, then `Verify`.
8. **Presentation.**
   - palette (S06 distance ≥ 40 against implemented styles);
   - VFX recipe map row (§4.7.2);
   - SFX row and `SFX_SPEED` entries (§4.7.3);
   - Style Gear recipes (§4.7.4), summon models (§4.7.5);
   - manifest entries for every `anim./vfx./sfx./model.` key (Part 8 `tools/check_manifest.luau` must pass).
9. **Skin keys.** `style.<id>.name`, `style.<id>.epithet`, `move.<id>.{1..4,q,a1..a4,passive,awaken}.name` in the `Codename` and `Original` packs (Part 8). `lint_strings` must be clean.
10. **Tests.**
    - `lune run tests/run.luau`: `styles.*`, `kits.<id>.gdd_numbers` (add the row to §4.5.9's table), `behaviors.*`, `kit_status.*`;
    - scenario `kit.<id>.smoke`: each slot used on a training dummy; HP delta = `maxDamage` for single-target hits (± 0.1, chip excluded); cooldown attribute = `f + toFrames(cooldown)`; no `[ALG][Style] ERROR`;
    - `kit.awaken.swap`, `anim.placeholders_built`.
11. **Docs.**
    - `lune run tools/export_framedata.luau` (regenerates `docs/reference/framedata.json`);
    - update the `ARCHITECTURE.md` rows for the kit file (status `wip` → `done`);
    - append evidence to `docs/agent/PROGRESS.md`.
12. **Human checkpoint.** H-10 feel playtest of the kit. H-08 publish placeholder animations before multi-peer tests.

---

### 4.9 New contracts introduced

1. **Lint scope.** `tools/lint_rules.luau` rule `pure` also covers `src/Shared/Styles/**` and `src/Shared/Behaviors/**` (P4-R1). Part 3's `standards.sim_banned_tokens` already covers `Behaviors/**`.
2. **Style module shape.** `Styles/<id>.luau` returns `(K: KitBuilder) -> StyleModule {def, hooks, moment, anim}`. The Styles API is in §4.1.2: `get, tryGet, all, implemented, isPlayable, move, hooks, moment, anim, behavior, variants, forConfig, validate, maxDamage, passiveStep, serverHook, SFX_SPEED`. Part 1's skill `algebraic-style-kit.md` must say "returns a builder function producing a StyleModule" instead of "returning a StyleDef".
3. **Config.** New top-level `Kits` section (§4.1.6) with ConfigService key `ALG_Kits`, pinned with `CombatCfg` (Part 2 §2.6.6).
   - `Combat.Realm` keys read here: `CastFrames` (90), `RadiusStuds` (40), as Part 3 §3.17 names them.
   - `Combat.Awakening.DurationSec` (60) is read by K10.
4. **`BehaviorModule` extensions** (Part 3 §3.4.2 / §3.18 item 10): optional `name`, `validate`, `maxDamage`, `tellFrames`, `blockable`, `intercept`, `hurtCarrier`, `reflectCarrier`, plus Part 3's `stepCarrier(view, f)` on `Projectile`, `Zone` **and `Summon`** (§4.2.0.1).
   - Part 3 calls `intercept` before `Damage.resolve` when the victim is in `Move` and its behavior defines it.
   - Part 3 calls `hurtCarrier` when a carrier hurtbox is hit, and `reflectCarrier` on a `reflect` verdict.
   - `Styles.behavior("Strike")` returns `nil`.
5. **`BehaviorCtx` additions:** `inp: Fsm.InputFrame` and `ray(origin, dir) -> (Vector3?, Vector3?)` (static world, `WorldQuery` group, excluding `workspace.Sim`).
   - **Carrier view content:** the `view` argument of `stepCarrier` is `CarrierView`, with `CarrierOut` and `FighterLite` (§4.2.0.3).
   - **Carrier params:** dynamic only, ≤ 10, always including `lifeF`; static data is read from the MoveDef.
   - **Death:** `dead` → `Dead = true`.
   - **Rush echo:** an echo carrier (`Kind = "zone"`, `sub = 1`) is dispatched to `Zone.stepCarrier`.
6. **`Fsm.HitDef.kit: KitHitExt?`** (§4.2.0.2):
   - `stun`, `debuffs`, `pull`, `meterDrain`, `healSelf`, `grab`, `release`, `reflectable`, `firstOfUse`;
   - Part 3 applies them in the server consequence phase, only on outcome `"hit"` (`grab` on catch);
   - Part 3 sets `firstOfUse`.
7. **`Fsm.HitboxReq.coneDeg`** (Sphere angular filter) and **`Fsm.HitRequest.heldOnly`** (targets = victims Grabbed by this attacker).
8. **`Hitstun` Sub codes** 2 freeze, 3 float, 4 stumble, 5 possess, 6 flinch (§4.2.0.2), and **`Grabbed` Sub codes** 1–6 (§4.2.4). `Stagger` Sub 3 is used for counter bounces with the reaction's `hitstun`.
9. **`SpawnRequest.serverOnly: boolean?`**: when true, Part 3 creates the carrier on the server branch only (Part 3 §3.11.5 "server-only creation"). Used by summons and homing projectiles. Targetable summon carriers (`hp > 0`) also get `CanQuery = true`, `CollisionGroup = "Hurtbox"` and `Size = params.hurtbox` (default 3×5×3).
10. **Grab release.** Part 3 detaches `ALG_Hold` whenever the holder leaves `Move`, and whenever a hit with `kit.release = true` resolves.
11. **Zone pull.** A zone MoveDef with `params.pullStudsPerSec` makes Part 3 apply an S-class planar push toward the carrier centre on enemies inside, every step. Grabs: on catch, Part 3 sets the attacker's `ALG_Grab.CFrame = CFrame.new(kit.grab.offset)` and the victim's `ALG_Hold.Attachment1` to it; on release it resets `ALG_Grab` to (0, 0, −2.5).
12. **Kit-mode variants.** `Fsm.StyleRuntime.variants: { [moveId]: { MoveDef } }` from `Styles.variants(def)`. `Moves.defFor` returns `variants[id][s.Kit + 1]` when present.
13. **Hook dispatch.**
    - Part 3 calls `Styles.passiveStep(h, inp)` after `Fsm.step` (both peers); its `motion` applies only when the FSM motion is `none` and `St == Air`.
    - Part 3 calls `Styles.serverHook(kind, h, info)` for `use, hitDealt, hitTaken, blocked, perfectBlocked, ko, lowHp, wallsplat` (server, consequence phase) and `awaken/awakenEnd` (both).
    - `HookCtx` includes `moves = Styles.move` for lookups.
    - `HookOut` fields are applied as in §4.3.1.
14. **`Moves.STATUS` content** (§4.3.2), the `consumeOn` flag, the `incomingFrom` field and `StatusMods.m1Chain`.
    - `nextHitAdd` applies to the owner's outgoing hits only.
    - Both adds apply on `firstOfUse`, clamped to the move's band cap.
    - Part 3's `M1.onPress` honours `m1Chain`.
15. **Move entry and chassis data.** Part 3's `Move` entry effects zero `MvA` and `MvB`. **Chassis data convention:** kits use exactly Part 3's recognized keys (`ragdollFramesAdd`, `lowHpThreshold` + `lowHp`, `m1RangeAdd`, `walkSpeedMult`, …). Part 4-only keys are `hoverSec`/`hoverMode` (read by `passiveStep`) and the render-only `visualScale`/`aura` in `awakening.modifiers`, which Part 3 must ignore (§4.3.3). Behavior `patch` writes `MvA`, `MvB`, `Kit` (B) and `Buffs`, `StDur`, `CdQ` (B·S). Part 3's adapter must accept these from behavior and hook outputs (Part 3 §3.11.4).
16. **`Kit` attribute semantics per style:** hero = Sword Swap mode; rocker and frost = hover frames left; others = 0. `MvA`/`MvB` usage per behavior: once-bits, hit counters, grab hold data (§4.2).
17. **SimEvent kinds added:** `"face"` (render-only face cue), `"summon_down"`. `"style_moment"` uses `a = victim rid`.
18. **Asset keys.**
    - `anim.<style>.<slot>`, `anim.<style>.realm` (RealmCast), `anim.<style>.awaken`, `anim.<style>.m1_<sub>` / `anim.<style>.ender_<sub>` (Part 3 §3.11.8 keys; Part 4 supplies poses), `vfx.<style>.<slot>`, `vfx.status.<kind>`, `vfx.decay.aura`, `sfx.<style>.<slot>`, `model.<style>.gear_<name>`, `model.<style>.summon_<name>`, `model.monarch.guardian_bust`.
    - New VFX recipes: `VFX_Telegraph_Arc`, `VFX_Projectile_Orb`, `VFX_Zone_Disc`, `VFX_Wave_Wall`, `VFX_Encase_Block`, `VFX_Stealth_Shimmer`, `VFX_Status_Pips`.
    - `Styles.SFX_SPEED`.
19. **Placeholder animation contract** (§4.6): naming, attributes (`AssetKey`, `MoveId`, `Frames`, `Gen`), marker names and values, the timing-lock rule, and `docs/reference/framedata.json`.
20. **Log lines:** `[ALG][Style] registry ok …`, `[ALG][Style] registry loaded …`, `[ALG][Style] ERROR rule=…`, `[ALG][Style] moment …`, `[ALG][Style] carrier cap …`.
21. **Lune specs and test files:** `tests/core/styles.spec.luau`, `kits.spec.luau`, `behaviors.spec.luau`, `kit_status.spec.luau`, `tests/shim/FakeCtx.luau`. **Scenarios:** `style.registry_boot`, `config.kits_override_next_life`, `kit.<style>.smoke` ×6, `kit.awaken.swap`, `kit.awaken.hurtbox`, `kit.grab_vs_jump`, `kit.grab_breakaway`, `kit.stretch.shield_reflect`, `kit.rocker.batshift_iframes`, `kit.frost.hover`, `kit.decay.well_heal`, `kit.decay.realm`, `kit.moment_meter`, `anim.placeholders_built`, `anim.timing_lock`, `look.stealth_ranges`.
22. **Realm content:** `Realm.rules["decay.a4"]` (§4.4.3), keyed by MoveDef id (Part 3 §3.9.3).

### 4.10 Issues with spine

1. **`Strike` has no module.** §0.8's `Behavior` includes `"Strike"`, but the §0.3 layout lists no `Behaviors/Strike.luau`. Resolved as Part 3 does: Strike is Core's plain-melee default and `Styles.behavior("Strike") == nil`.
2. **"Every GDD tunable in Defaults" vs kit data in code.** §0.4 says `Defaults.luau` holds every tunable. StyleDefs use `Vector3`/`Color3` and cannot live in JSON config (Part 2 §2.6.1). Move numbers therefore live in `Styles/*`, and live tuning goes through the new `Kits.Overrides` layer. `Kits` is a new top-level key not in §0.9's fixed list; it needs a spine amendment.
3. **No file for shared kit types.** §0.3 fixes the file list. Part 4's shared types and helpers live in `Behaviors/Buff.luau` (the "kit kernel"), because a separate module would add a file.
4. **Core ownership.** Part 2 lists `Core/Realm.luau` as Part 4's; Part 3 claims the framework and leaves only `Realm.rules` to Part 4. This part follows Part 3, and also writes `Moves.STATUS` content inside Part 3's `Core/Moves.luau`.
5. **Canon references.** The brief asks for canon references in comments, but Part 1/Part 8 forbid canonical names in code. Comments use `canon: GDD §7.3.N <slot>` pointers.
6. **Extender rule vs GDD kits.** GDD §6.7 requires ≥ 1 extender per base kit. The GDD kit tables for `monarch`, `frost` and `decay` (§7.3.9, §7.3.13, §7.3.15) tag no move `Extender`. K04 accepts a frame-verified `params.extender` (monarch.1, frost.1, decay.1) without changing GDD tags. Design should confirm.
7. **Log tag.** The brief says `[ALG][STYLE]`; Part 2's tag list uses `Style`, so lines read `[ALG][Style]`.

### 4.11 Open questions / spikes

Spikes (proposed for P0):
- **S-P4-1 Predicted carriers.** This is Part 3 spike SA-09 (stitching when two characters spawn carriers in the same frame). If it fails, Part 3 falls back to server-only carriers plus a cosmetic ghost. Part 4 content works either way: `stepCarrier` is deterministic and hits are server-only. To support ghosts, `Projectile` exports the pure `Projectile.trajectory(def, origin, age) -> CFrame`, which RenderController (Part 6) uses.
- **S-P4-2 Grab hold smoothness** under SA via `ALG_Hold` (AlignPosition) for the victim's client at 150 ms. Measure misprediction count per hold.
- **S-P4-3 Status write cost.** An S-class `Debuffs` write per hit plus zone `stay` refreshes every 6 f: measure rollbacks per second on a victim standing in a zone. Fallback: a refresh every 12 f with `frames = 24`.
- **S-P4-5** `RightGripAttachment` exists on the default R15 `RightHand` (not in the docs snapshot).
- **S-P4-6** `@lune/roblox` exposes `Vector3.new`, `CFrame.Angles`, `CFrame.lookAt` and `Color3.fromRGB` for the Lune shim.
- **S-P4-7** Pose sign conventions (§4.6.2) via `Animator:StepAnimations` + `screen_capture`.

Design questions:
- **Q1 Decay interpretation.** "His hits apply Decay … adds +1 damage to his next hit". Implemented as:
  - stacks are not consumed;
  - the bonus applies to the first landed hit of each use, clamped to the band;
  - among M1s only the Ender applies a stack;
  - ticks apply none.
  The literal reading (every hit +1 stack, every hit +n) would exceed GDD §6.6 combo targets.
- **Q2 stretch giant-form AoE ×1.5.** Read as already included in the GDD's A1–A4 radii (stretch.a1 12, stretch.a4 25). The alternative is to multiply them at runtime (18 / 37.5).
- **Q3 decay awakening "decay aura".** Cosmetic or a gameplay pulse? Currently cosmetic.
- **Q4 Diameter reading.** "N-stud pool/cloud/storm/area/circle" is read as a diameter (decay.3 radius 5, frost.2 radius 3).
- **Q5 Scientific Method.** Do summon and zone hits count as "landing" a move (currently yes, once per use)? Does one pair trigger once or chain (currently once per pair)?
- **Q6 Fool's Frenzy** frame data for M1 hits 5–6 (currently: hits 4–5 reuse M1-3, hit 6 is the Ender).
- **Q7 Unspecified sizes.** Grey Slime size (14-stud pool, td) and the monarch.a2 robot's post-dash life (idles until 6 s to satisfy the Summon tag's 6–12 s rule).
- **Q8 Slow stacking.** Additive slows capped at 60% (td) vs strongest-wins.
- **Q10 Archetype strings.** They are the GDD §7.2 "IP-neutral archetype" column in PascalCase (e.g. `DecayLich`, `VampireRocker`). If Part 8's denylist flags a generic word inside one of them, rename that archetype (e.g. `DecayUndead`). The id is not player-visible, and S05 only checks the pattern.
- **Q9 Stealth vs Style Tell.** Hiding the nameplate beyond 12 studs while invisible conflicts with "Style Tell always on" (§0.4). Proposal: stealth suppresses the nameplate only, and the shimmer keeps the style colour.
