[← Build spec index](../STUDIO_BUILD_SPEC.md) · **Part 5 of 8** · sections §5.x · Part 0 contracts (§0.x) are in the index file

## 5. Meta systems

This part specifies every non-combat system of M1-G: style ownership and equip rules, the Wish system (odds, pity, Flairs, ledger), PolicyService gating, Choose Goose's Shop, the economy and progression tracks, mock-first monetization, the appearance data model (Character Look toggle), the telemetry wrapper and moderation basics.

It builds on Part 2's contracts and does not restate them: service APIs (§2.3.5), `Types.luau` (§2.4), attribute homes (§2.5), config publishing (§2.6), the remote catalog (§2.7.4), the DataService transaction model (§2.8.5) and the security model (§2.11). Where this part **adds** to a Part 2 shape, it says so and lists the addition in §5.12.

### 5.0 Conventions for this part

**Units.**

| Quantity | Unit | Notes |
|---|---|---|
| Odds | integer **ppm** (parts per million) | 10,000 ppm = 1%. Every odds table the server rolls from sums to exactly 1,000,000. |
| Display odds | percent | ppm / 10,000. Tiers and Flairs are shown with 4 decimals; per-style odds with 3 decimals (GDD §8.3 style). |
| Time (meta) | unix seconds from `DataService.now()` | `workspace:GetServerTimeNow()` plus the Studio dev offset (§2.8.5). Pure modules receive `now` as a parameter. |
| Day, week, period indices | integers | `Pity.dayIndex/weekIndex/periodIndex` (§5.5.8). All resets are UTC. |
| Distances | studs | — |
| Currencies | integers ≥ 0 | `Treasure`, `Wishes`, `Shards`, `CosmicSparks` (`Types.CurrencyId`) |

**Rules (MUST).**
1. Every economic operation is **one** `DataService.update` call that checks, debits and grants together (§2.8.5). No yields inside mutators.
2. Odds, rolls, rotations, curves and pity math live in the **pure** modules `Shared/Wish/Odds.luau` and `Shared/Wish/Pity.luau`. Services call them; they never re-implement them. The client uses the same modules to render odds, so the UI cannot drift from the server.
3. The client never supplies a seed, an odds value, a price, a counter or an outcome. C2S payloads carry only intents (`count`, `slot`, `styleId`…), validated per §2.7.4.
4. Every tunable is in `Defaults.luau` (§5.10) with a `GDD §x.y` or `(tuning default)` comment.
5. Log tags used by this part: `Style`, `Wish`, `Shop`, `Store`, `Policy`, `Progress`, `Appearance`, `Tel`, `Mod` (Part 2 list) plus **`TEL`** for the Studio analytics echo (GDD §18.10 wording; new, §5.12).

**Files owned by this part.**

| File | Realm | Pure | Responsibility |
|---|---|---|---|
| `src/Shared/Wish/Odds.luau` | shared | yes | Deterministic stream, tier/Flair tables, roll, effective-odds view, odds hash, shop rotation, free rotation, daily-quest pick |
| `src/Shared/Wish/Pity.luau` | shared | yes | Pity rules, ledger codec (§2.8.4 contract), base-36, account/mastery curves, UTC indices, anti-farm multipliers |
| `src/ServerScriptService/Services/StyleService.luau` | server | no | §5.1 (Part 4 owns `Shared/Styles`; this service owns ownership/equip) |
| `.../WishService.luau` | server | no | §5.2 |
| `.../PolicyGate.luau` | server | no | §5.3 |
| `.../ShopService.luau` | server | no | §5.4 |
| `.../ProgressionService.luau` | server | no | §5.5 |
| `.../MonetizationService.luau` | server | no | §5.6 |
| `.../AppearanceService.luau` | server | no | §5.7 (Part 6 renders) |
| `.../Telemetry.luau` | server | no | §5.8 (API from §2.3.5) |
| `.../ModerationService.luau` | server | no | §5.9 additions (core from §2.11) |
| `tests/wish/*.spec.luau` | host | — | Lune specs, §5.11 |

Both pure modules obey the load-purity rule (§2.10 item 2): they may require only `../Types`, `../Config/Defaults` (types), `../Config/Schema` and `../Core/Rng`, and they never touch `game`, `workspace`, `script`, `Instance`, `Enum` or `task`. Per §2.10 item 5, meta pure helpers (curves, rotations) live in these two files rather than in new files.

---

### 5.1 StyleService: ownership, equip rules, free rotation, Training trial

#### 5.1.1 Ownership

- A style is **owned** iff `data.styles[styleId] ~= nil`. Records follow `Types.StyleRecord` (§2.4).
- `hero` and `stretch` are owned by every account (`source = "starter"`). The template and `Schema.normalizePlayerData` guarantee it (§2.8.3; §0.10).
- Styles are granted only through `StyleService.grantInTx(data, styleId, source, now)`, a **pure** function called inside an `update` mutator by WishService (`"wish"`, `"cosmic_craft"`), ShopService (`"shop"`) or TestHarness (`"dev"`). It returns `true` if the style became newly owned; it never overwrites an existing record (GDD §8.2, "rolls add, never replace").
- Ownership is account-bound. There is no trading (GDD §16.2); `PolicyFlags.paidTradingAllowed` is stored but unused in M1-G.
- Styles with `implemented = false` can be owned (collectible) but not equipped (§0.10). The UI shows the "In development" badge from `Styles.get(id).implemented`.

#### 5.1.2 Equip rules (`Style.Equip`, §2.7.4 row 4)

`StyleService.canEquip(player, styleId)` returns `(true, nil)` only if **all** of the following hold. The first failing row decides the error code.

| # | Condition | Error |
|---|---|---|
| 1 | `styleId ∈ Types.STYLE_IDS` (shape validator already checked) | `invalid` |
| 2 | `Styles.isPlayable(styleId, Runtime.get())` (Part 4 §4.1.2, the one equip gate). When false: `Styles.get(styleId).implemented == false` → `not_implemented`; otherwise (the kit is in Part 4's runtime-disabled set because it failed validation outside Studio, or `styleId ∈ Runtime.get().Features.DisabledStyles`) → `disabled` | `not_implemented` / `disabled` |
| 3 | Owned, **or** `styleId ∈ StyleService.freeRotation(now)`, **or** (`player:GetAttribute("MapId") == "training"` and `Progression.TrainingTrialAllStyles`) | `not_owned` |

Acceptance: a request for an unimplemented owned style returns `{ok=false, err="not_implemented"}`; an owned style listed in `Features.DisabledStyles` returns `disabled`; an unowned style outside the rotation and outside Training returns `not_owned`. StyleService never checks `implemented` or `DisabledStyles` on its own: every equip, rotation and trial decision goes through `Styles.isPlayable`.

#### 5.1.3 When an equip applies

- **Default: next respawn.** An accepted `Style.Equip` sets `NextStyle = styleId` (Player attribute) and returns `{nextStyle = styleId, applyAt = "respawn"}`. The Style Wheel opens on death with a 3 s auto-confirm (GDD §15.4; Part 6).
- **At a Style Totem: now.** GDD §4.3 allows switching mid-life at the Style Totem in each spawn hub. The server decides "totem" from position, not from the payload:
  1. `StyleService.start()` caches totem positions per map: for every Model `m` under `Workspace.Map` and `Workspace.Arena` with attribute `MapId`, each direct child `c` of `m:FindFirstChild("Totems")` contributes `c:GetPivot().Position` (`PVInstance:GetPivot` [Docs]). Part 6's builders place totems there (§2.2 tree).
  2. On `Style.Equip`, if the player's `HumanoidRootPart` is within `Progression.TotemRangeStuds` (12, tuning default) of a cached totem of the player's `MapId`, **and** the injected swap gate returns true, the request is accepted with `applyAt = "now"`: StyleService sets `NextStyle` and fires `StyleService.swapNow(player)`.
  3. The swap gate is installed by PlayerService in its `init()`: `StyleService.setSwapGate(function(p) return CombatService.canTravel(p) end)`. This follows the `Net.setReadyCheck` pattern, because StyleService (#7) may not require CombatService (#9) (§2.3.3). Until a gate is set, the gate returns false (applyAt falls back to `"respawn"`).
  4. PlayerService connects `swapNow` and calls `CombatService.spawnAsync(player)`. This is a respawn without a KO: no `ko` event, no streak reset, and no death penalty. Part 3's random safe spawn picks a hub on the same map. Respawning at the nearest spawn is a future hook.
- `StyleService.resolveForSpawn(player)` is called by `CombatService.spawnAsync` before `LoadCharacterAsync` (§3.13.4 step 2). Algorithm:

```lua
--!strict
-- StyleService.resolveForSpawn (excerpt). Spec §5.1.3.
function StyleService.resolveForSpawn(player: Player): Types.StyleId
	local data = DataService.get(player)
	local candidate: string = player:GetAttribute("NextStyle") or ""
	if candidate == "" then
		candidate = sessionPick[player] or (if data then data.equipped.styleId else "hero")
	end
	local ok = StyleService.canEquip(player, candidate)          -- re-validated: rotation may have ended, map may have changed
	local styleId: Types.StyleId = if ok then candidate :: Types.StyleId
		elseif data and StyleService.canEquip(player, data.equipped.styleId) then data.equipped.styleId
		else "hero"
	if data and data.styles[styleId] ~= nil and data.equipped.styleId ~= styleId then
		DataService.update(player, { "equipped" }, "equip", function(d) d.equipped.styleId = styleId end)
	end
	sessionPick[player] = if data and data.styles[styleId] == nil then styleId else nil -- trial/rotation picks live in memory only
	local prev = player:GetAttribute("EquippedStyle")
	local rec = data and data.styles[styleId]
	player:SetAttribute("EquippedStyle", styleId)
	player:SetAttribute("Flair", if rec then rec.equippedFlair else "standard")
	player:SetAttribute("Variant", if rec then rec.equippedVariant else "")
	player:SetAttribute("NextStyle", "")
	StyleService.lifeStyleResolved:Fire(player, styleId)          -- every spawn; handlers MUST NOT yield
	if prev ~= styleId then StyleService.equippedChanged:Fire(player, styleId) end
	Log.info("Style", "resolved", { user = player.UserId, style = styleId })
	return styleId
end
```

- `data.equipped.styleId` is persisted **only** for owned styles (P2 invariant "equipped.styleId owned"). Trial and rotation picks live in the server-memory map `sessionPick` and are re-validated at every spawn.

#### 5.1.4 Weekly free rotation (GDD §8.1 goal 2)

- **Candidates:** pool styles for which `Styles.isPlayable(id, Runtime.get())` is true and whose rarity ∉ {`starter`, `cosmic`}, in `Types.STYLE_IDS` order. In M1-G these are `monarch`, `rocker`, `frost` and `decay`. A style removed by a kill switch (`Features.DisabledStyles` or Part 4's runtime-disabled set) drops out of the candidate list, which can change that week's draw; this is accepted because the kill switch is an emergency tool.
- **Ratio note (DES-6, §5.14):** with 4 candidates and `Count = 3` (GDD §8.1, kept), 3 of the 4 rollable implemented styles (75 %) are free every week in M1-G, against the GDD's 3 of 16 at full roster. This weakens the Wish/Shop chase the slice demonstrates; the owner keeps it or disables the rotation through `ALG_Progression` (`{"FreeRotation":{"Enabled":false}}`). The default is keep.
- **Index:** `w = Pity.weekIndex(now)`, where weeks start Monday 00:00 UTC (GDD §9.5 weekly refresh). This holds when `Progression.FreeRotation.PeriodSec == 604800`. Any other value uses `Pity.periodIndex(now, PeriodSec)` (test aid).
- **Selection:** `Odds.freeRotation(w, candidates, Count, Salt)`. It seeds `Odds.stream(bit32.bxor(Odds.mul32(w % 2^32, 0x9E3779B9), Salt))` and makes `min(Count, #candidates)` uniform draws without replacement (`Odds.pickWeighted` with weights 1). The result is sorted in `STYLE_IDS` order.
- **Disabled** (`FreeRotation.Enabled == false`) → the result is the empty list.
- The client computes the same list from `Runtime` and `DataService.now()`-equivalent time for the Style Wheel badge. The server value is authoritative at equip time.

Vectors (Lune `style.free_rotation_vectors`; candidates = the 4 M1-G styles; `Count = 3`, `Salt = 0x52A1C3E7`):

| `now` (UTC) | week index | expected |
|---|---|---|
| 1791763200 (2026-10-12 00:00, Monday) | 2963 | `rocker, frost, decay` |
| +604800 | 2964 | `monarch, rocker, frost` |
| +1209600 | 2965 | `monarch, rocker, frost` |
| 1791763199 | 2962 | (week boundary check: index 2962) |

#### 5.1.5 Training trial mode

- On `MapId == "training"` with `Progression.TrainingTrialAllStyles = true`, every style that passes rule 2 (`Styles.isPlayable`) also passes rule 3 (GDD §4.5 "free trial of any style").
- Trial styles use `Flair = "standard"` and `Variant = ""`. Character Look is available for them (GDD §12.3 "own (or are trialing)").
- No Treasure, XP, Mastery or Wish progress is earned on `training` (`Economy.TrainingEarns = false`, §5.5.3).
- On leaving Training (`World.Travel` → `candy_kingdom`), the next `resolveForSpawn` re-validates and falls back to `data.equipped.styleId`.

#### 5.1.6 Flair and Variant equip

- `Style.SetFlair{styleId, flair}`: the style must be owned and `styles[id].flairs[flair] ~= nil` (any Flair ever rolled for that style is equippable, GDD §8.7). The handler does one update of `styles` (`equippedFlair = flair`). If `styleId == EquippedStyle`, it writes the `Flair` attribute immediately; Flairs are cosmetic, so a mid-life change is safe.
- `Style.SetVariant{styleId, variantId}`: `""` or an owned variant (`styles[id].variants[variantId] == true`). Same write pattern, with the `Variant` attribute.
- Log `[ALG][Style] flair user=<id> style=<id> flair=<f>` and `[ALG][Style] variant ...`.

#### 5.1.7 StyleService API (Part 2 contract plus additions)

```lua
--!strict
export type StyleServiceApi = {
	-- Part 2 §2.3.5 (unchanged)
	onPlayerReady: (player: Player) -> (),         -- writes EquippedStyle/NextStyle/Flair/Variant from data
	canEquip: (player: Player, styleId: string) -> (boolean, Types.ErrCode?),
	requestEquip: (player: Player, styleId: string, via: "wheel" | "totem" | "dev") -> Types.Response<Types.StyleEquipRes>,
	resolveForSpawn: (player: Player) -> Types.StyleId,
	equippedChanged: Signal.Signal<Player, Types.StyleId>, -- fires only when the life's style differs from the previous life
	grantInTx: (data: Types.PlayerData, styleId: Types.StyleId, source: Types.OwnSource, now: number) -> boolean,
	-- Part 5 additions (§5.12)
	lifeStyleResolved: Signal.Signal<Player, Types.StyleId>, -- fires on EVERY resolveForSpawn, before LoadCharacterAsync
	swapNow: Signal.Signal<Player>,                -- totem swap accepted; PlayerService respawns
	setSwapGate: (gate: (player: Player) -> boolean) -> (),
	freeRotation: (now: number) -> { Types.StyleId },
	isOwned: (player: Player, styleId: string) -> boolean,
}
```

The `Style.Equip` handler calls `requestEquip(player, req.styleId, "wheel")`; the `via` is upgraded to `"totem"` by the proximity rule. It logs `[ALG][Style] equip user=<id> style=<id> apply=<respawn|now> via=<wheel|totem|dev>`.

#### 5.1.8 Verify (§5.1)

- Lune: `style.free_rotation_vectors`, `time.week_index_monday`.
- Scenarios (Part 7 harness names):
  - `style.equip_owned_respawn`: equip `stretch` → `NextStyle="stretch"`; after `Dev.Command kill` and respawn → `EquippedStyle="stretch"`, `NextStyle=""`.
  - `style.equip_unimplemented_rejected`: `giveStyle sentry`; `Style.Equip{styleId="sentry"}` → `err="not_implemented"`.
  - `style.equip_unowned_rejected`: `Style.Equip{styleId="decay"}` on `candy_kingdom` with the rotation disabled (`setConfig Progression {"FreeRotation":{"Enabled":false}}`) → `not_owned`.
  - `style.training_trial`: `travel training`; equip `decay` → ok; travel `candy_kingdom`; respawn → `EquippedStyle` = the persisted owned style.
  - `style.free_rotation_equip`: `setTimeOffset` to week 2963 → `Style.Equip{styleId="rocker"}` ok on `candy_kingdom`.
  - `style.totem_swap`: teleport the character next to a totem (Part 6 builder), out of combat → response `applyAt="now"`; a new character spawns within 2 s; the log shows `via=totem`.
  - `style.disabled_kill_switch`: `setConfig Features {"DisabledStyles":["stretch"]}` while `stretch` is equipped → the next spawn resolves to the persisted fallback or `hero`.

---

### 5.2 Wish system

GDD §8.1–§8.8, §15.5 (first Wish), §17.2 (cap). The pool is the full 16-style rollable roster with `Wish.PoolMode = "Full"` (§0.10).

#### 5.2.1 Pool

- `Odds.buildPool(entries: { PoolEntry }) -> Pool`, where `entries` comes from `Styles.all()` mapped to `{styleId, rarity, implemented}`.
- It filters out `rarity == "starter"`, sorts by `STYLE_IDS` index and groups by tier.
- It errors if any of the 7 rollable tiers is empty, or if a `FirstWishGuarantee.Styles` entry is not in the pool at `FirstWishGuarantee.Rarity`. WishService catches this at `start()`, logs `[ALG][Wish] ERROR pool_invalid ...`, and its remotes return `disabled`.
- M1-G pool (GDD §8.3): common `sentry, granny, diva`; uncommon `tyrant, console, grappler`; rare `monarch, flame, butler`; epic `rocker, frost, huntress`; legendary `decay, demon`; mythic `wish`; cosmic `chaos`.
- Lune `wish.pool_matches_styles` asserts that the registry produces exactly this pool.

#### 5.2.2 Pure module API (`Shared/Wish/Odds.luau`, `Shared/Wish/Pity.luau`)

```lua
--!strict
-- src/Shared/Wish/Odds.luau — odds, rolls, effective odds, rotations. Realm: shared. Pure: yes. Spec §5.2.
local Types = require("../Types")
local Defaults = require("../Config/Defaults")
local Schema = require("../Config/Schema")
local Rng = require("../Core/Rng")
local Pity = require("./Pity")

export type WishCfg = typeof(Defaults.Wish)
export type PoolEntry = { styleId: Types.StyleId, rarity: Types.Rarity, implemented: boolean }
export type Pool = { byTier: { [string]: { PoolEntry } }, all: { PoolEntry } }
export type Stream = { state: number }
export type Context = {                        -- build once per config version; holds memo caches
	cfg: WishCfg, pool: Pool,
	tierCache: { [string]: { number } },       -- key "<hit>:<bonusPpm>" -> ppm array (TIERS order)
	flairCache: { [string]: { number } },      -- key "lucky"/"normal" -> ppm array (FLAIR_ORDER order)
	hashCache: { [string]: string },
}
export type RollInput = {
	seed: number,                              -- 1 .. 2^31-2, server-generated (§5.2.3)
	pity: Types.PityState,
	owned: { [string]: boolean },              -- StyleId -> true (current, including earlier rolls of the same batch)
	variantsOwned: { [string]: { [string]: boolean } }, -- StyleId -> VariantId -> true
	first: boolean,                            -- first-Wish guarantee pending (§5.2.9)
	lucky: boolean,                            -- Lucky Jar active (owned AND not PRI-restricted)
}
export type RollOut = {
	rarity: Types.Rarity, styleId: Types.StyleId, flair: Types.FlairId, isNew: boolean,
	variantId: string,                         -- "" = none
	pityHit: Types.PityHit, pityAfter: Types.PityState, fakeOut: boolean,
	draws: { number },                         -- d1..d5 (tests and ledger audits)
}
export type ViewState = {
	pity: Types.PityState, owned: { [string]: boolean }, first: boolean, lucky: boolean, restricted: boolean,
}
-- Exports:
-- Odds.TIERS: { Types.Rarity }  = { "common","uncommon","rare","epic","legendary","mythic","cosmic" }
-- Odds.MILLION = 1000000
-- Odds.mul32(a: number, b: number): number                 -- (a*b) mod 2^32, exact
-- Odds.fmix32(h: number): number                           -- MurmurHash3 finalizer
-- Odds.stream(seed: number): Stream ; Odds.draw(s: Stream): number   -- u in [0,1)
-- Odds.buildPool(entries: { PoolEntry }): Pool
-- Odds.context(cfg: WishCfg, pool: Pool): Context
-- Odds.largestRemainder(w: { number }, total: number): { number }
-- Odds.tierTable(ctx: Context, pity: Types.PityState, first: boolean): ({ number }, Types.PityHit)
-- Odds.flairTable(ctx: Context, lucky: boolean): { number }
-- Odds.candidates(ctx: Context, rarity: Types.Rarity, owned: { [string]: boolean }, hit: Types.PityHit): ({ Types.StyleId }, { number })
-- Odds.pickPpm(ppm: { number }, u: number): number         -- index
-- Odds.pickWeighted(weights: { number }, u: number): number -- index
-- Odds.roll(ctx: Context, input: RollInput): RollOut
-- Odds.view(ctx: Context, st: ViewState): Types.OddsView
-- Odds.oddsHash(ctx: Context, lucky: boolean): string     -- 8 hex chars
-- Odds.shopRotation(...) / Odds.parseRotationId(...)       -- §5.4
-- Odds.freeRotation(week: number, candidates: { Types.StyleId }, count: number, salt: number): { Types.StyleId }
-- Odds.dailyQuestPick(userId: number, day: number, poolSize: number, count: number): { number } -- §5.5.9
```

```lua
--!strict
-- src/Shared/Wish/Pity.luau — pity counters, ledger codec, curves, UTC indices. Realm: shared. Pure: yes. Spec §5.2/§5.5.
-- Exports:
-- Pity.floorHit(p: Types.PityState, c: PityCfg): Types.PityHit         -- "mythic" | "legendary" | "epic" | "none"
-- Pity.softBonusPpm(p: Types.PityState, c: PityCfg): number
-- Pity.advance(p: Types.PityState, rarity: Types.Rarity): Types.PityState
-- Pity.validate(p: Types.PityState, c: PityCfg): (boolean, string?)     -- bounds invariant
-- Pity.checkTransition(before: Types.PityState, after: Types.PityState, rarity: Types.Rarity, c: PityCfg): (boolean, string?)
-- Pity.b36(n: number): string ; Pity.unb36(s: string): number?         -- non-negative integers, <= 10 digits
-- Pity.encodeLedger(e: Types.LedgerEntry): string ; Pity.decodeLedger(s: string): Types.LedgerEntry?
-- Pity.ledgerPush(l: Types.Ledger, entry: string, size: number): ()  -- ring semantics of §2.8.4
-- Pity.accountLevel(xpTotal: number, c: AccountCfg): (number, number, number)  -- level, xpIntoLevel, xpToNext
-- Pity.masteryXpToReach(level: number, c: MasteryCfg): number ; Pity.masteryLevel(xp: number, c: MasteryCfg): number
-- Pity.dayIndex(t: number): number ; Pity.weekIndex(t: number): number ; Pity.periodIndex(t: number, sec: number): number
-- Pity.repeatMultiplier(nth: number, mults: { number }): number
-- Pity.rookieKoPays(log: { number }, now: number, c: { WindowSec: number, MaxTreasureKills: number }): boolean -- §5.5.4 (per earner)
-- Pity.effectiveLook(settings: Types.Settings, styleId: string): boolean  -- §5.7.1 (re-exported by AppearanceService)
```

#### 5.2.3 Server RNG and the deterministic per-roll stream

- **Seed source (server only).** WishService creates `local seedRng = Random.new()` once in `init()`. With no seed, `Random.new` uses an internal entropy source [Docs `datatypes/Random.yaml`].
  - Each roll gets `seed = seedRng:NextInteger(1, 2147483646)`.
  - Each batch gets `batchId = <6-char base-36, zero-padded> of seedRng:NextInteger(0, 2176782335)`.
  - Seeds never come from the client and are never sent to the client before the roll is committed.
- **Per-roll stream (pure, replayable).** `Odds.stream(seed)` returns `{state = Rng.new(Odds.fmix32(seed % 4294967296))}`. `Odds.draw(s)` calls `Rng.next(s.state)` (Part 3 §3.4.3 xorshift32; value = new state / 2^32), stores the new state and returns the value.
  - `fmix32` removes xorshift32's weak low-seed start. For example, seed 1 would otherwise give a first draw of 0.00006, which is a Cosmic.
  - Every roll consumes **exactly 5 draws**, in this order: `d1` tier, `d2` style, `d3` Flair, `d4` Variant, `d5` fake-out. This holds whether or not a draw is used, so the ledger seed alone reproduces the roll.
- **32-bit helpers** (exact in doubles; products stay below 2^53):

```lua
--!strict
local TWO32 = 4294967296
function Odds.mul32(a: number, b: number): number
	local aLo, aHi = a % 65536, a // 65536
	local bLo, bHi = b % 65536, b // 65536
	local mid = (aHi * bLo + aLo * bHi) % 65536            -- only the low 16 bits survive the << 16
	return (aLo * bLo + mid * 65536) % TWO32
end
function Odds.fmix32(h: number): number
	h = bit32.bxor(h, bit32.rshift(h, 16))
	h = Odds.mul32(h, 0x85EBCA6B)
	h = bit32.bxor(h, bit32.rshift(h, 13))
	h = Odds.mul32(h, 0xC2B2AE35)
	return bit32.bxor(h, bit32.rshift(h, 16))
end
function Odds.stream(seed: number): Stream
	return { state = Rng.new(Odds.fmix32(seed % TWO32)) }
end
function Odds.draw(s: Stream): number
	local u, nextState = Rng.next(s.state)
	s.state = nextState
	return u
end
```

- **Studio seam.** `WishService._setSeedMaster(seed: number?)` is Studio-only and a no-op elsewhere. With a master seed `m`, seeds come from `Odds.stream(m)`: each next seed is `1 + math.floor(Odds.draw(master) * 2147483646)`; `nil` restores `seedRng`. It is exposed through `Dev.Command setWishSeedMaster {seed}` (new DevCmd, §5.12; `seed = 0` clears).

#### 5.2.4 Tier table: exact algorithm (GDD §8.3–§8.4)

Inputs: `cfg.TierOdds` (ppm), the pity counters `p` (Wishes since the last hit, **before** this roll), `cfg.Pity`, and the `first` flag.

1. **First-Wish guarantee.** If `first` is true, return `{rare = 1,000,000; all others 0}` with hit `"first"` (§5.2.9). Pity floors and soft pity are ignored for that roll; the counters still advance.
2. **Base weights.** `w[t] = TierOdds[t]` for t in `TIERS` order.
3. **Soft pity** (Legendary+). Let `k = p.sinceLegendaryPlus + 1`, the ordinal of this roll since the last Legendary+.
   - If `k ≥ SoftStart` (60): `bonus = SoftStepPpm × (k − SoftStart + 1)`. Roll 60 gets +0.5%, roll 119 gets +30%.
   - `LplusEff = min(1e6, base.legendary + base.mythic + base.cosmic + bonus)`.
   - **Cosmic is fixed** at `base.cosmic` (GDD §8.4 "Cosmic: no pity"). The remainder `LplusEff − cosmic` is split between legendary and mythic by their base ratio (16,000 : 3,900):

     `w.legendary = (LplusEff − cosmic) × base.legendary / (base.legendary + base.mythic)`, and the same for mythic.
   - Lower tiers re-normalize: `w[t] = (1e6 − LplusEff) × base[t] / (base.common + base.uncommon + base.rare + base.epic)` for `t ∈ {common…epic}`.
4. **Hard pity floor** (`Pity.floorHit`, precedence mythic > legendary > epic). The floor applies when `p.sinceMythicPlus + 1 ≥ 400`, else `p.sinceLegendaryPlus + 1 ≥ 120`, else `p.sinceEpicPlus + 1 ≥ 30`.
   - With a floor `F`, set `w[t] = 0` for every tier below `F`.
   - Rescale the tiers from `F` to `mythic` so they fill `1e6 − cosmic` in proportion to their current weights. Cosmic keeps its 100 ppm: the floors guarantee "F or better", and Cosmic is never pity-boosted.
5. **Quantize** with `Odds.largestRemainder(w, 1e6)`. It floors each `w × 1e6 / Σw` and gives the leftover units to the largest fractional parts; ties go to the lower tier index. The result is an integer ppm array that sums to exactly 1,000,000. This is what the server rolls and what the UI shows.
6. **Roll.** `r = math.floor(d1 × 1e6)`; the tier is the first index whose cumulative ppm exceeds `r`.

```lua
--!strict
function Odds.tierTable(ctx: Context, p: Types.PityState, first: boolean): ({ number }, Types.PityHit)
	local cfg = ctx.cfg
	if first then
		local out = table.create(7, 0)
		out[table.find(Odds.TIERS, cfg.FirstWishGuarantee.Rarity) :: number] = Odds.MILLION
		return out, "first"
	end
	local hit = Pity.floorHit(p, cfg.Pity)
	local bonus = Pity.softBonusPpm(p, cfg.Pity)
	local key = `{hit}:{bonus}`
	local cached = ctx.tierCache[key]
	if cached then return cached, hit end
	local base = cfg.TierOdds
	local w = table.create(7, 0)
	for i, t in Odds.TIERS do w[i] = base[t] end
	local cosmic = w[7]
	if bonus > 0 then
		local lm = w[5] + w[6]
		local lplusEff = math.min(Odds.MILLION, base.legendary + base.mythic + base.cosmic + bonus)
		local lplusLm = lplusEff - cosmic
		w[5] = lplusLm * base.legendary / lm
		w[6] = lplusLm * base.mythic / lm
		local lower = Odds.MILLION - lplusEff
		local lowerBase = base.common + base.uncommon + base.rare + base.epic
		for i = 1, 4 do w[i] = lower * base[Odds.TIERS[i]] / lowerBase end
	end
	if hit ~= "none" then
		local fi = table.find(Odds.TIERS, hit) :: number
		for i = 1, fi - 1 do w[i] = 0 end
		local s = 0
		for i = fi, 6 do s += w[i] end
		for i = fi, 6 do w[i] = w[i] * (Odds.MILLION - cosmic) / s end
	end
	local ppm = Odds.largestRemainder(w, Odds.MILLION)
	ctx.tierCache[key] = ppm
	return ppm, hit
end

function Odds.largestRemainder(w: { number }, total: number): { number }
	local s = 0
	for _, x in w do s += x end
	local out, frac, idx = table.create(#w, 0), table.create(#w, 0), table.create(#w, 0)
	if s <= 0 then return out end
	local used = 0
	for i, x in w do
		local exact = x * total / s
		local f = math.floor(exact)
		out[i], frac[i], idx[i] = f, exact - f, i
		used += f
	end
	table.sort(idx, function(a: number, b: number): boolean
		if frac[a] ~= frac[b] then return frac[a] > frac[b] end
		return a < b
	end)
	for k = 1, total - used do out[idx[k]] += 1 end
	return out
end
```

The arithmetic order above (`a * b / c`, left-associative; sums accumulated left to right from 0) is **normative**: the test vectors in §5.2.15 depend on it.

#### 5.2.5 Style selection, Flair, Variant, fake-out

**Style (`d2`).** `Odds.candidates(ctx, rarity, owned, hit)`:

| Case | Candidates (in `STYLE_IDS` order) | Weights |
|---|---|---|
| `hit == "first"` | `FirstWishGuarantee.Styles` (`monarch`, `flame`) | `FirstWishGuarantee.Weights` (1, 1 → 50/50, GDD §15.5) |
| `hit ∈ {epic, legendary, mythic}` and `Pity.NewStylePriority` | unowned styles of the rolled tier; if none are unowned, all styles of the tier (GDD §8.4 new-style priority) | 1 each |
| otherwise | all pool styles of the rolled tier (GDD §8.3 equal per-style split) | 1 each |

`styleId = ids[Odds.pickWeighted(weights, d2)]`. `pickWeighted` computes `r = u × Σw` and returns the first index whose cumulative weight exceeds `r` (last index as a guard).

New-style priority applies whenever a hard-pity floor was active for the roll, even if the unfloored roll would have reached that tier. It is **not** applied to soft-pity hits (GDD: "when a pity triggers").

**Flair (`d3`).** `Odds.flairTable(ctx, lucky)` uses `FLAIR_ORDER` order.

```lua
--!strict
function Odds.flairTable(ctx: Context, lucky: boolean): { number }
	local key = if lucky then "lucky" else "normal"
	local cached = ctx.flairCache[key]
	if cached then return cached end
	local m = if lucky then ctx.cfg.LuckyJarFlairMultiplier else 1
	local out, sum = table.create(7, 0), 0
	for i = 2, 7 do
		out[i] = math.floor(ctx.cfg.Flair[Types.FLAIR_ORDER[i]] * m + 0.5)
		sum += out[i]
	end
	out[1] = Odds.MILLION - sum                      -- "standard" takes the remainder (re-normalized)
	ctx.flairCache[key] = out
	return out
end
```

- Base odds (GDD §8.7): shiny 20,000; golden 4,000; crystal 1,000; rainbow 200; nightosphere 40; glitched 10; standard 974,750 ppm.
- Lucky Jar (×1.5, **Flairs only**, never style rarity; GDD §17.3): shiny 30,000; golden 6,000; crystal 1,500; rainbow 300; nightosphere 60; glitched 15; standard 962,125.
- `lucky = MonetizationService.ownsPass(player, "lucky_jar") and not PolicyGate.get(player).paidRandomRestricted`. A restricted player who owns the pass gets no paid luck boost on a paid random item (GDD §8.8).

**Variant (`d4`).** Only on a duplicate (`not isNew`), and only if `d4 < VariantChance` (0.05, GDD §9.3).
- `unowned` = `"v1".."v<VariantsPerStyle>"` that are not in `variantsOwned[styleId]`.
- If it is non-empty: `variantId = unowned[min(#unowned, floor(d4 / VariantChance × #unowned) + 1)]`.
- The duplicate still grants its Shards.
- The placeholder catalog is `VariantsPerStyle = 2` (`v1`, `v2`), named by Skin keys `variant.<styleId>.<vid>.name` (Part 8).

**Fake-out (`d5`).** `fakeOut = FakeOut[rarity] ~= nil and d5 < FakeOut[rarity]`, where `FakeOut = {epic = 0.03}` (GDD §8.6: a Rare "glitches" into an Epic in 3% of Epic pulls). It is presentation only, upgrade-only, and disclosed in the odds panel footer (`ui.wish.fakeout_note`).

**Pity advance.** `Pity.advance(p, rarity)`, where rank is the index in `RARITY_ORDER`:

| Counter | Reset to 0 when the rolled rarity is | Otherwise |
|---|---|---|
| `sinceEpicPlus` | epic, legendary, mythic, cosmic | +1 |
| `sinceLegendaryPlus` | legendary, mythic, cosmic | +1 |
| `sinceMythicPlus` | mythic, cosmic | +1 |

**Bounds invariant** (`Pity.validate`): `0 ≤ sinceEpicPlus ≤ EpicPlusHard − 1`, `0 ≤ sinceLegendaryPlus ≤ LegendaryPlusHard − 1`, `0 ≤ sinceMythicPlus ≤ MythicPlusHard − 1`, all integers.

**`Pity.checkTransition`** returns false when `after ≠ Pity.advance(before, rarity)`, or when a floor was active and the rarity is below it. Either failure raises a `wish_anomaly` signal (§2.11.3).

Cosmic craft (§5.2.8) and Shop purchases do **not** touch the counters. Counters persist forever (GDD §8.4).

#### 5.2.6 Effective-odds view (GDD §8.8 "current effective odds")

`Odds.view(ctx, st)` returns a `Types.OddsView` (§2.4) for the **next single roll**. Part 5 adds the fields `nextGuarantee`, `baseTiers`, `baseFlairs`, `softStartWish` and `hardPity` (§5.12).

```lua
--!strict
-- OddsView field semantics (all pct = ppm / 10000):
-- oddsHash             Odds.oddsHash(ctx, st.lucky)
-- tiers[i]             { rarity, pct = tierTable(ctx, st.pity, st.first)[i]/1e4, styles = per-style rows }
--   styles[j]          { styleId, pct = tier pct * w_j / Σw (candidates for that tier under the current hit; 0 for excluded styles),
--                        owned, implemented }  -- every pool style is listed, including 0% rows
-- flairs               flairTable(ctx, st.lucky) as { flair, pct } rows in FLAIR_ORDER
-- baseTiers/baseFlairs the same with fresh pity, first = false, lucky = false (the published table, GDD §8.3/§8.7)
-- pity                 st.pity
-- legendaryPlusPctNow  (legendary + mythic + cosmic ppm)/1e4 of the current table
-- nextGuarantee        PityHit of the next roll ("none"|"epic"|"legendary"|"mythic"|"first")
-- softStartWish        cfg.Pity.SoftStart ; hardPity = { epic = 30, legendary = 120, mythic = 400 } (from cfg)
-- luckyJar             st.lucky ; restricted = st.restricted
```

Rules:
1. Per-style odds in a tier use the same candidate rule as the roll. For example, under the Epic+ floor with `rocker` owned, the rows are `frost` and `huntress` at 50% of the epic tier each, and `rocker` at 0%. The view is therefore exact, not an approximation.
2. The client computes the view locally from `Runtime.get().Wish`, `Styles.all()`, and the PrivateView (`pity`, `styles`, `wishStats.firstGuaranteeUsed`, `passes.lucky_jar`, `policy`). It recomputes after every store change, so the view updates live after every Wish.
3. When the Wish screen opens, the client calls `Wish.GetOdds` once and compares `oddsHash` and every tier ppm. On a mismatch it logs `[ALG][UI] WARN odds_mismatch local=<h> server=<h>` and renders the server view.
4. The footer states that each roll of a 10× Wish uses the odds in effect for that roll (pity applies per roll), and that Cosmic is never boosted by pity or luck.
5. PRI display rule (GDD §8.8): the odds button is always visible, including to restricted players, and the tier rows sum to exactly 100.0000%. The view lists every style and every Flair. Per-style rows are rounded to 3 decimals for display only.

**Odds hash.** `Odds.oddsHash(ctx, lucky) = Schema.hash32(Schema.canonical(input))`, where `input` is:

```lua
{ v = 1, tiers = cfg.TierOdds, pity = cfg.Pity, flair = <flairTable(lucky) as {flairId = ppm}>, lucky = lucky,
  first = cfg.FirstWishGuarantee, fakeOut = cfg.FakeOut, variantChance = cfg.VariantChance,
  pool = { "<styleId>:<rarity>", ... } -- STYLE_IDS order
}
```

It is memoized in `ctx.hashCache`. Any change to odds, pity rules, Flair odds, luck, the guarantee or the pool changes the hash. The ledger stores it per roll.

#### 5.2.7 10× Wish semantics

- `Wish.Make{count = 10}` performs 10 **sequential** rolls inside one `DataService.update`. Roll *i* uses the pity counters, ownership and `first` flag left by roll *i−1*. A style unlocked in roll 3 is a duplicate in roll 7, and it is "owned" for the new-style priority of later rolls.
- 10 Wishes are required. There is no discount and no partial batch.
- The result is a `WishBatchResult` with `best` = the index of the highest rarity (first occurrence on ties).
- The reveal grid (GDD §8.6) shows all 10 results with duplicates faded and "+Shards". The best result plays the full reveal (Part 6).

#### 5.2.8 Duplicates, Shards, Cosmic Sparks, Cosmic craft

| Outcome | Grant |
|---|---|
| New style | `StyleService.grantInTx(d, styleId, "wish", now)`. The Flair is counted; if the Flair is not `standard`, `equippedFlair` is set to it. |
| Duplicate | `rec.dupes += 1`; `Shards += Wish.DuplicateShards[rarity]` (common 3, uncommon 5, rare 10, epic 25, legendary 60, mythic 150, cosmic 300 — tuning default); optional Variant (§5.2.5). |
| Mythic duplicate | additionally `CosmicSparks += Wish.SparksPerMythicDupe` (1, GDD §8.4) |
| Any roll | `rec.flairs[flair] += 1`. If the Flair rank (index in `FLAIR_ORDER`) is greater than `bestFlair`, `bestFlair = flair` and `bestFlairUpgraded = true` (GDD §8.7 "keep the best Flair ever rolled"). |

Shards and Sparks are summed over the batch and credited once each at the end of the mutator (`tx.credit("Shards", total, "duplicate")`, `tx.credit("CosmicSparks", total, "mythic_dupe")`). This produces one economy event per currency per batch.

**`Wish.CraftCosmic`** (§2.7.4 row 10), as one update over `{"currencies","styles","ledger"}`:
1. Requires `CosmicSparks ≥ Wish.CosmicSparksNeeded` (10, GDD §8.4) → else `insufficient`.
2. Candidates are the pool's cosmic styles not owned. If there are none → `{ok=false, err="denied", detail="all_cosmic_owned"}`; Sparks are kept.
3. A server seed drives a stream: `d2` picks the style (uniform), `d3` rolls the Flair (Lucky Jar rules apply), and `d4`/`d5` are drawn and unused.
4. Debit Sparks (`cosmic_craft`); grant the style (source `"cosmic_craft"`); append a ledger entry (`src = "f"`, `pityHit = "c"`, `n = wishStats.total`). The pity counters and `wishStats.total` are unchanged.
5. Return a `WishBatchResult` with one result (`pityHit = "cosmic_craft"`). Announce at Cosmic level (§5.2.13).

#### 5.2.9 First-Wish guarantee and tutorial Wishes (GDD §15.5)

- `WishService.onPlayerReady(player)`: if `not wishStats.tutorialGranted`, run one update that does `tx.credit("Wishes", Economy.TutorialWishes, "tutorial")` (3) and sets `tutorialGranted = true`. This happens on the first join, whether or not the tutorial is played or skipped (GDD: "Skipping still grants the 3 wishes").
- The **first roll ever** made by the account, from any source (in 1× or 10×), has `first = Wish.FirstWishGuarantee.Enabled and not wishStats.firstGuaranteeUsed`. That roll is Rare, with style `monarch`/`flame` weighted 1:1. It sets `firstGuaranteeUsed = true` in the same mutator. `pityHit = "first"`.
- Source labels per roll:
  - `"paid"` while paid Wishes are being spent (§5.2.10);
  - else `"tutorial"` while `wishStats.total < Economy.TutorialWishes`;
  - else `"free"`.

  `wishStats.free` counts tutorial and free rolls; `wishStats.paid` counts paid rolls.

#### 5.2.10 Wallet rules: storage cap, paid Wishes, escrow, overflow (GDD §17.2, §8.8)

| Rule | Implementation |
|---|---|
| Cap 1,000 | `Wish.StorageCap = 1000`. DataService invariant: no update raises `Wishes` above the cap (§2.8.5). |
| Earned Wishes over the cap → 10 Treasure each | `tx.credit("Wishes", n, reason)` (earned) credits up to the cap and converts the excess at `Wish.OverflowTreasure` (10) with reason `wish_overflow` (§2.8.5 rule 6). Robux never becomes Treasure because purchased Wishes never overflow. |
| Purchases that would exceed the cap are blocked | `Store.Prompt` checks `Wishes + paidEscrow + pack ≤ StorageCap`, else `cap_reached` (§5.6.4). |
| Race: a receipt arrives while the room is gone | The grant credits `min(pack, room)` and puts the rest in **`wishStats.paidEscrow`** (new field). The escrow is released as `tx.credit("Wishes", m, "purchase", {purchased = true})` at the end of every `Wish.Make` and in `WishService.onPlayerReady`, with `m = min(paidEscrow, cap − Wishes)`. A paid Wish is never lost and never converted. |
| Paid Wishes are spent first | `nPaid = if restricted then 0 else min(count, paidUnspent)`; the first `nPaid` rolls of the batch are `"paid"`. |
| PRI-restricted players cannot spend paid Wishes | If `paidRandomRestricted`, spendable = `Wishes − paidUnspent`. A request that needs more → `{ok=false, err="restricted", detail="paid_wishes_locked"}`. The policy says restricted users "can not interact with paid random item generators, either via in-experience currency bought with Robux or Robux directly" [Docs `PolicyService.yaml`]. |
| Normalize | `Schema.normalizePlayerData` clamps `paidUnspent` to `0..currencies.Wishes` and `paidEscrow ≥ 0` (Part 2 addition, §5.12). |

#### 5.2.11 WishService: API, remotes, flow

```lua
--!strict
export type WishServiceApi = {
	onPlayerReady: (player: Player) -> (),         -- tutorial grant (once) + escrow release
	-- Studio-only seams (no-ops outside Studio):
	_setSeedMaster: (seed: number?) -> (),
}
```

Remotes (all from §2.7.4; no new Wish remotes):

| Remote | Handler summary | Errors |
|---|---|---|
| `Wish.Make{count}` | §5.2.11 flow below | `disabled`, `busy`, `insufficient`, `restricted`, `not_ready`, `internal` |
| `Wish.GetOdds` | `Odds.view(ctxFor(cfg), state(player))`; Telemetry wish funnel step 1 `odds_viewed` | `disabled` |
| `Wish.CraftCosmic` | §5.2.8 | `insufficient`, `denied`, `busy` |

**`Wish.Make` flow (MUST, in order):**

```lua
--!strict
-- WishService (excerpt). Spec §5.2.11. Runs inside Net's pcall'd handler.
local function make(player: Player, count: number): Types.Response<Types.WishBatchResult>
	local cfg = Runtime.get()
	if not cfg.Features.Wish then return { ok = false, err = "disabled" } end
	if busy[player] then return { ok = false, err = "busy" } end
	local data = DataService.get(player)
	if data == nil then return { ok = false, err = "not_ready" } end
	local restricted = PolicyGate.get(player).paidRandomRestricted
	if data.currencies.Wishes < count then return { ok = false, err = "insufficient" } end
	if restricted and data.currencies.Wishes - data.wishStats.paidUnspent < count then
		return { ok = false, err = "restricted", detail = "paid_wishes_locked" }
	end
	busy[player] = true
	local ctx = ctxFor(cfg)                                       -- Odds.context, cached per Runtime.version()
	local lucky = (not restricted) and MonetizationService.ownsPass(player, "lucky_jar")
	local hash = Odds.oddsHash(ctx, lucky)
	local seeds = table.create(count, 0)
	for i = 1, count do seeds[i] = nextSeed() end                 -- §5.2.3; generated BEFORE the mutator
	local batchId = nextBatchId()
	local results: { Types.WishResult } = {}
	local anomalies: { string } = {}
	local okUpdate = DataService.update(player, { "currencies", "pity", "wishStats", "ledger", "styles", "flags" }, "wish",
		function(d: Types.PlayerData, tx: DataService.Tx)
			local nPaid = if restricted then 0 else math.min(count, d.wishStats.paidUnspent)
			assert(tx.debit("Wishes", count, "wish", { sku = if count == 10 then "wish_x10" else "wish_x1" }), "debit")
			d.wishStats.paidUnspent -= nPaid
			local shardsSum, sparksSum = 0, 0
			for i = 1, count do
				local src: Types.WishSource = if i <= nPaid then "paid"
					elseif d.wishStats.total < cfg.Economy.TutorialWishes then "tutorial" else "free"
				local first = cfg.Wish.FirstWishGuarantee.Enabled and not d.wishStats.firstGuaranteeUsed
				local before = d.pity
				local r = Odds.roll(ctx, { seed = seeds[i], pity = before, owned = ownedSet(d),
					variantsOwned = variantsOf(d), first = first, lucky = lucky })
				local okT, why = Pity.checkTransition(before, r.pityAfter, r.rarity, cfg.Wish.Pity)
				if not okT then table.insert(anomalies, why or "transition") end
				d.pity = r.pityAfter
				if first then d.wishStats.firstGuaranteeUsed = true end
				local isNew = StyleService.grantInTx(d, r.styleId, "wish", tx.now)
				local rec = d.styles[r.styleId]
				rec.flairs[r.flair] = (rec.flairs[r.flair] or 0) + 1
				local upgraded = FLAIR_RANK[r.flair] > FLAIR_RANK[rec.bestFlair]
				if upgraded then rec.bestFlair = r.flair end
				if isNew and r.flair ~= "standard" then rec.equippedFlair = r.flair end
				local shards, sparks = 0, 0
				if not isNew then
					rec.dupes += 1
					shards = cfg.Wish.DuplicateShards[r.rarity]
					if r.rarity == "mythic" then sparks = cfg.Wish.SparksPerMythicDupe end
					if r.variantId ~= "" then rec.variants[r.variantId] = true end
				end
				shardsSum += shards; sparksSum += sparks
				local firstReveal = d.flags.seenReveals[r.rarity] ~= true
				d.flags.seenReveals[r.rarity] = true
				d.wishStats.total += 1
				if src == "paid" then d.wishStats.paid += 1 else d.wishStats.free += 1 end
				Pity.ledgerPush(d.ledger, Pity.encodeLedger({
					n = d.wishStats.total, t = math.floor(tx.now), src = src, batch = batchId, seed = seeds[i],
					oddsHash = hash, styleId = r.styleId, rarity = r.rarity, flair = r.flair, isNew = isNew,
					shards = shards, sparks = sparks, variantId = r.variantId, pityBefore = before, pityHit = r.pityHit,
				}), cfg.Data.LedgerSize)
				table.insert(results, { n = d.wishStats.total, styleId = r.styleId, rarity = r.rarity, flair = r.flair,
					isNew = isNew, shards = shards, sparks = sparks, variantId = if r.variantId == "" then nil else r.variantId,
					pityHit = r.pityHit, bestFlairUpgraded = upgraded, fakeOut = r.fakeOut, firstReveal = firstReveal })
			end
			if shardsSum > 0 then tx.credit("Shards", shardsSum, "duplicate") end
			if sparksSum > 0 then tx.credit("CosmicSparks", sparksSum, "mythic_dupe") end
			releaseEscrowInTx(d, tx, cfg)                       -- §5.2.10
		end)
	busy[player] = nil
	if not okUpdate then return { ok = false, err = "internal" } end
	afterCommit(player, results, batchId, hash, anomalies)      -- log, telemetry, announcements, anomaly signals (§5.2.13)
	local d = DataService.get(player) :: Types.PlayerData
	return { ok = true, data = { batchId = batchId, results = results, best = bestIndex(results),
		wishesLeft = d.currencies.Wishes, pity = d.pity, overflowTreasure = 0 } }
end
```

- `busy[player]` is also cleared in a `pcall` guard around the body. An error at any point leaves the flag false.
- `ownedSet(d)` is rebuilt per roll (≤ 18 keys), so in-batch unlocks count.
- `FLAIR_RANK` = index in `Types.FLAIR_ORDER`.
- `afterCommit` logs `[ALG][Wish] batch user=<id> n=<count> best=<rarity> src=<first roll src> pityE=<n> pityL=<n> pityM=<n> hash=<h> batch=<id>`.
- `afterCommit` emits `ModerationService.signal(player, "wish_anomaly", {why = ...})` for each anomaly (§2.11.3).
- The TestHarness command `setPity` bypasses the transition check. TestHarness writes with reason `"dev"`, and the check runs only in `Wish.Make`.

#### 5.2.12 Ledger (GDD §18.5)

- Capacity `Data.LedgerSize = 500`, using the ring and codec format v1 of §2.8.4. `Pity.encodeLedger`/`decodeLedger` implement that contract exactly.
- The index fields `style`, rarity and flair are **0-based** (style 0–17 as one base-36 char, rarity 0–7, flair 0–6).
- The "pity after" value is not stored. It is derived as `Pity.advance(pityBefore, rarity)`, which `Pity.checkTransition` already verifies at write time.
- **Replay** (support tickets, monthly Wish Report): given an entry and the config identified by its `oddsHash`, `Odds.roll` with the stored `seed`, the decoded `pityBefore`, `first = (pityHit == "first")` and `lucky` (encoded in the hash) reproduces `rarity`, `flair` and `fakeOut` exactly. It also reproduces `styleId` whenever `pityHit == "none"`. Under a floor, the style also depends on the owned set at that time, so the check is `styleId ∈ tier`.
- The ledger is never sent to clients (§2.8.5). Quick Wish's "wish-history export" is a future hook.

**Vector (Lune `ledger.vector_v1`).**

```
entry: n=1, t=1791763200, src="tutorial", batch="a1b2c3", seed=958783841, oddsHash="0badc0de",
       styleId="monarch", rarity="rare", flair="standard", isNew=true, shards=0, sparks=0,
       variantId="", pityBefore={0,0,0}, pityHit="first"
encodeLedger -> "1|1|tmrpc0|t|a1b2c3|fuu2ch|0badc0de|8|301f|0|0||0,0,0"   (53 chars)
```

`ledger.codec_roundtrip` covers 1,000 random valid entries with `decode(encode(e)) == e`. `ledger.max_len_80` asserts the worst-case entry is ≤ 80 chars. `ledger.decode_rejects_garbage` covers a wrong version, 12 fields, a non-base-36 digit, a style index > 17, and an unknown hit code.

#### 5.2.13 Announcements and post-commit effects (GDD §8.6)

| Best rarity in the batch (or craft) | Effect |
|---|---|
| epic | `Net.server.fire("Notify", player, {kind="announce", key="ui.notify.wish_epic", args={styleId, rarityId="epic"}, rarity="epic", scope="self"})` |
| legendary | `Net.server.fireAll("Notify", {kind="announce", key="ui.notify.wish_legendary", args={who=player.DisplayName, styleId, rarityId}, rarity, scope="server"})` |
| mythic, cosmic | the server-wide Notify above, plus a cross-server publish (below) |

- **Cross-server in M1-G** (decides §2.7.4's open note): WishService implements topic `Wish.Announce.CrossServerTopic = "ALG_Announce_v1"`, but it is **off by default**: both calls below run only when `Runtime.get().Wish.Announce.CrossServerEnabled == true` **and** `not RunService:IsStudio()`. `CrossServerEnabled = false` is the M1-G default (§5.10); the owner can enable it live through `ALG_Wish` (`{"Announce":{"CrossServerEnabled":true}}`). Without it, mythic/cosmic pulls get the server-wide Notify only.
  - **Publish:** `pcall(MessagingService.PublishAsync, MessagingService, topic, {jobId = game.JobId, key = "ui.notify.wish_<rarity>_global", rarityId, styleId})`. The payload is < 200 bytes (1 kB limit [Docs]). `PublishAsync` yields, so it runs in the post-commit `task.spawn` (never inside the DataService mutator).
  - **Subscribe:** `MessagingService:SubscribeAsync` yields until the backend registers the subscription [Docs `MessagingService.yaml`], and the Part 2 module contract requires `start()` to return within 1 s. So `start()` only spawns it, once, when the gate above holds at start time (enabling the flag later takes effect on the next server; publishing follows the live flag):

    ```lua
    --!strict
    -- WishService.start (excerpt). Spec §5.2.13.
    -- Module-level: `local subscribed = false`, `local announceConn: RBXScriptConnection? = nil`.
    local function subscribeAnnounce(): ()
    	local cfg = Runtime.get().Wish.Announce
    	if subscribed or RunService:IsStudio() or cfg.CrossServerEnabled ~= true then return end
    	subscribed = true
    	task.spawn(function()
    		local ok, conn = pcall(MessagingService.SubscribeAsync, MessagingService, cfg.CrossServerTopic, onAnnounce)
    		if ok then
    			announceConn = conn :: RBXScriptConnection
    		else
    			subscribed = false
    			Log.warn("Wish", "announce_subscribe_failed", { err = tostring(conn) })
    		end
    	end)
    end
    ```

    The handler `onAnnounce(msg)` drops messages whose `msg.Data.jobId == game.JobId` and `fireAll`s the rest with `scope = "global"`. A failure logs `[ALG][Wish] WARN announce_subscribe_failed err=<e>` and is not retried in M1-G.
  - Cross-server messages carry no player name (GDD: "Someone just wished…"). Delivery is best-effort (GDD §18.5).
- Telemetry (§5.8): `Telemetry.custom(player, "wish_made", count, {CustomField01 = <src of roll 1>, CustomField02 = <best rarity>, CustomField03 = Wish.BannerId, + echo extras})`, then the wish funnel step 2 `wished`.
- Server-detected FTUE step 8 `first_wish` is sent if `flags.ftueStep < 8` (§5.8.4).
- Fake-outs, reveal lengths and the skip rule are presentation (Part 6). `firstReveal` tells the client whether the rarity's reveal has been seen; `Settings.skipSeenReveals` governs repeat views.

#### 5.2.14 Policy integration summary

| Condition | Wish screen | Server |
|---|---|---|
| `paidRandomRestricted = false` | 1× / 10× / odds; paid Wishes spent first; Lucky Jar applies if owned | as specified |
| `paidRandomRestricted = true` | 1× / 10× / odds stay (free Wishes always allowed); paid Wish packs and Lucky Jar are hidden; the wallet shows "locked paid Wishes: N" | paid Wishes not spendable (`restricted`); luck off; `Store.Prompt` for PRI keys → `restricted` |
| Policy fetch failed (live) | treated as restricted (§5.3) | same as restricted |

#### 5.2.15 Lune test vectors (normative; `tests/wish/*.spec.luau`)

All vectors were generated by a reference implementation of exactly §5.2.3–§5.2.5. A mismatch means the Luau implementation deviates. Debug in this order: `mul32`/`fmix32` → stream states → tier tables → candidates → rolls.

**A. 32-bit helpers** (`wish.mul32_vectors`, `wish.fmix32_vectors`)

| Call | Expected |
|---|---|
| `mul32(0xFFFFFFFF, 0xFFFFFFFF)` | 1 |
| `mul32(123456789, 0x9E3779B9)` | 3133743405 |
| `fmix32(0)`, `(1)`, `(2)`, `(42)` | 0, 1364076727, 821347078, 142593372 |
| `fmix32(123456789)`, `(2147483646)`, `(4294967295)` | 3126909082, 1427853558, 2180083513 |

**B. Stream** (`wish.stream_vectors`): the `state` after each of the first 5 draws (draw value = state / 2^32).

| Seed | States after draws 1..5 |
|---|---|
| 1 | 524866043, 2877414208, 2380002740, 2664205378, 3890067424 |
| 2 | 3122306797, 3509455822, 447725381, 1168790822, 4007446145 |
| 42 | 3411452759, 4172666345, 51530061, 222447184, 4102354452 |
| 123456789 | 2400119906, 47325507, 2578851326, 4231046202, 4265974849 |
| 2147483646 | 3752870357, 651937808, 2766477436, 4206488233, 981041855 |

**C. Tier tables** (`wish.tier_tables`; ppm in `TIERS` order: common, uncommon, rare, epic, legendary, mythic, cosmic; pity given as E/L/M = `sinceEpicPlus/sinceLegendaryPlus/sinceMythicPlus`):

| State | E/L/M | hit | ppm |
|---|---|---|---|
| fresh | 0/0/0 | none | 500000, 280000, 140000, 60000, 16000, 3900, 100 |
| Epic+ hard | 29/29/29 | epic | 0, 0, 0, 750864, 200230, 48806, 100 |
| soft, roll 60 | 0/59/59 | none | 497449, 278571, 139286, 59694, 20020, 4880, 100 |
| soft, roll 61 | 0/60/60 | none | 494898, 277143, 138571, 59388, 24040, 5860, 100 |
| soft, roll 119 | 0/118/118 | none | 346939, 194286, 97143, 41632, 257206, 62694, 100 |
| Legendary+ hard | 5/119/119 | legendary | 0, 0, 0, 0, 803940, 195960, 100 |
| Mythic+ hard | 5/10/399 | mythic | 0, 0, 0, 0, 0, 999900, 100 |
| Epic+ hard during soft (roll 80) | 29/79/79 | epic | 0, 0, 0, 300138, 562623, 137139, 100 |
| first Wish | any | first | 0, 0, 1000000, 0, 0, 0, 0 |

`wish.tier_tables_sum_1e6`: every table for E ∈ 0..29, L ∈ 0..119, and M ∈ {0, 398, 399} sums to 1,000,000; cosmic = 100 in every non-first table; no entry is negative.

**D. Flair tables** (`wish.flair_tables`; `FLAIR_ORDER` order): normal `974750, 20000, 4000, 1000, 200, 40, 10`; lucky `962125, 30000, 6000, 1500, 300, 60, 15`.

**E. Single rolls** (`wish.roll_vectors`). Defaults: `owned = {hero, stretch}`, `variantsOwned = {}`, `lucky = false`, `first = false`.

| ID | Seed | Pity E/L/M | Other input | rarity | style | flair | isNew | variant | hit | pity after | fakeOut |
|---|---|---|---|---|---|---|---|---|---|---|---|
| R1 | 1 | 0/0/0 | — | common | diva | standard | true | "" | none | 1/1/1 | false |
| R2 | 42 | 0/0/0 | — | rare | butler | standard | true | "" | none | 1/1/1 | false |
| R3 | 123456789 | 0/0/0 | lucky | uncommon | tyrant | standard | true | "" | none | 1/1/1 | false |
| R4 | 10 | 29/29/29 | owned +rocker | epic | **frost** (priority skips owned rocker) | standard | true | "" | epic | 0/30/30 | false |
| R5 | 99 | 5/119/119 | owned +decay | legendary | **demon** (priority) | standard | true | "" | legendary | 0/0/120 | false |
| R6 | 2024 | 5/10/399 | — | mythic | wish | standard | true | "" | mythic | 0/0/0 | false |
| R7 | 5 | 0/0/0 | first | rare | monarch | standard | true | "" | first | 1/1/1 | false |
| R8 | 6 | 0/0/0 | first | rare | flame | standard | true | "" | first | 1/1/1 | false |
| R9 | 314159 | 0/118/118 | — | legendary | demon | standard | true | "" | none | 0/0/119 | false |
| R10 | 3 | 0/0/0 | owned +sentry,granny,diva | legendary | decay | standard | true | "" | none | 0/0/1 | false |
| R11 | 66 | 0/0/0 | same as R10 | epic | rocker | shiny | true | "" | none | 0/1/1 | false |
| R12 | 144 | 0/0/0 | same as R10 | common | diva | standard | **false** | **v1** | none | 1/1/1 | false |
| R13 | 289 | 0/0/0 | same as R10 | common | diva | golden | false | "" | none | 1/1/1 | false |
| R14 | 1326 | 0/0/0 | same as R10 | epic | rocker | standard | true | "" | none | 0/1/1 | **true** |
| R15 | 14 | 0/0/0 | lucky = false / true | common | diva | standard / **shiny** | true | "" | none | 1/1/1 | false |
| R16 | 1 | 5/10/399 | owned +wish | mythic | wish | standard | false | "" | mythic | 0/0/0 | false |

R4 and R5 discriminate new-style priority: without priority, R4 yields `rocker` and R5 yields `decay`. R15 discriminates luck: `d3 = 0.9683376` lies between the lucky and normal standard thresholds. R16 is a Mythic duplicate, which grants 150 Shards and 1 Spark through the service.

**F. 10× batch** (`wish.batch10_vector`). Fresh account: starters only, pity 0/0/0, first guarantee pending, `lucky = false`.
- Seeds come from a master stream `Odds.stream(777)`: `seed_i = 1 + floor(draw × 2147483646)`, giving `958783841, 607366981, 1790956215, 233935709, 1289326241, 1555924166, 1754204260, 972302372, 585530844, 1189389602`.
- Each roll uses the updated pity, owned set and first flag.

| # | rarity | style | flair | isNew | hit |
|---|---|---|---|---|---|
| 1 | rare | monarch | standard | true | first |
| 2 | uncommon | console | standard | true | none |
| 3 | uncommon | console | standard | false | none |
| 4 | uncommon | grappler | standard | true | none |
| 5 | uncommon | console | standard | false | none |
| 6 | common | diva | shiny | true | none |
| 7 | uncommon | tyrant | standard | true | none |
| 8 | common | sentry | standard | true | none |
| 9 | common | diva | standard | false | none |
| 10 | common | granny | standard | true | none |

Final pity 10/10/10. Service-level totals: Shards = 5 + 5 + 3 = 13; Sparks 0; no Variants.

**G. Other pure tests:**
- `wish.ftue_styles_match_rarity`
- `wish.cosmic_fixed` (cosmic = 100 under every floor/soft state)
- `wish.lucky_flair_only` (tier tables identical with lucky on and off)
- `wish.view_matches_roll_tables` (view pct × 1e4 == tierTable ppm for 200 random pity states)
- `wish.view_sums_100`
- `wish.view_priority_rows` (Epic+ floor with rocker owned → rocker 0, frost and huntress 375432 ppm-equivalent each)
- `wish.hash_stable` (same ctx → same hash; key order shuffled → same)
- `wish.hash_changes` (luck on/off, any TierOdds change, a pool change → different)
- `pity.floor_precedence`, `pity.soft_bonus_values` (k = 59 → 0, 60 → 5000, 119 → 300000), `pity.advance_resets`
- `pity.bounds_invariant_random_walk` (10,000 random rolls with fresh streams: counters always within bounds)
- `pity.transition_check_detects`

**H. Frequency tests** (`wish.tier_frequencies_fresh`, `wish.flair_frequencies_fresh`):
- 200,000 rolls with seeds from master stream 99, pity reset to 0/0/0 before each roll, owned = starters.
- Each tier frequency, and the shiny and golden frequencies, must lie within ±4σ of the configured probability (σ = √(p(1−p)/n)).

#### 5.2.16 Monte Carlo acceptance (§0.11; `tests/wish/montecarlo.spec.luau`)

**Procedure (normative):**
- Master stream `Odds.stream(20261012)`; 2,000 runs.
- Each run starts at pity 0/0/0 with owned = starters, `first = false` (the 3 tutorial Wishes are excluded, as in GDD §8.4) and `lucky = false`.
- Roll *n* uses `seed = 1 + floor(draw(master) × 2147483646)`. Pity and ownership update per roll.
- Day = `ceil(n / 14)` (14 Wishes/day, GDD §17.5).
- Record the day of the first Epic+, first Legendary+, first Mythic+, and full non-Cosmic roster (all 15 pool styles except `chaos`). Run until both Mythic+ and the full roster are reached.
- Median of an even count = mean of the two middle values; P95 = the sorted value at index `ceil(0.95 × n)` (1-based).

**Assertions:**

| Test | Assertion |
|---|---|
| `wish.mc_spine_tolerance` | Each mean is within ±10% of §0.11: Epic+ 1.4, Legendary+ 3.2, Mythic+ 12.6, roster 16.4 |
| `wish.mc_hard_pity_bounds` | max Epic+ day ≤ 3; max Legendary+ day ≤ 9; max Mythic+ day ≤ 29; roster P95 ≤ 29 |
| `wish.mc_reference_exact` | Bit-exact implementation check against the reference output below (means to 4 decimals) |

Reference output (2,000 runs, master seed 20261012; 449,764 rolls):

| Milestone | Mean | Median | Max | P95 |
|---|---|---|---|---|
| First Epic+ | 1.4105 | 1 | 3 | 3 |
| First Legendary+ | 3.1910 | 3 | 8 | 6 |
| First Mythic+ | 12.8620 | 11 | 29 | 29 |
| Full non-Cosmic roster | 16.5285 | 15 | 48 | 29 |

A 4,000-run reference with an independent seed gave 1.41 / 3.17 / 12.42 / 16.46. This confirms that the §5.2.4 interpretation reproduces §0.11 within tolerance:
- soft pity starts at +0.5% on roll 60;
- the boost is split legendary:mythic 16:3.9;
- Cosmic is fixed;
- new-style priority applies on hard pity only.

#### 5.2.17 Scenario tests (Studio, Part 7 harness)

| Scenario | Steps | Expected |
|---|---|---|
| `wish.tutorial_grant_once` | Fresh mock profile join; rejoin | `Wishes = 3` after first join; still 3 after rejoin; `tutorialGranted = true` |
| `wish.batch10_server` | `resetProfile`; `grant Wishes 10`; `setWishSeedMaster {seed = 777}`; `Wish.Make{count=10}` | results equal §5.2.15 F (the Lune spec of the same vector is `wish.batch10_vector`; test IDs are unique across Lune and Studio, Part 7 §7.1.4); `Shards = 13`; ledger has 10 entries; `[ALG][Wish] batch ... n=10 best=rare` |
| `wish.ftue_first_rare` | 20 iterations of: `resetProfile`; `grant Wishes 1`; `setWishSeedMaster {seed = k}` (k = 1..20); `Wish.Make{count=1}` | every roll is `rare` and `monarch` or `flame`; `pityHit = first`; both styles occur at least once |
| `wish.insufficient` | `Wishes = 5`; `Make{10}` | `err = "insufficient"`; wallet unchanged |
| `wish.busy` | two concurrent `Make{10}` from one client | one ok, one `busy` (or both ok sequentially if the first completed); never 20 rolls charged for 10 |
| `wish.pity_epic_hard` | `setPity 29 0 0`; `Make{1}` | rarity ∈ epic+; `pity.sinceEpicPlus = 0` |
| `wish.pity_mythic_hard` | `setPity 0 0 399`; `Make{1}` | rarity ∈ {mythic, cosmic} |
| `wish.restricted_paid_locked` | `simulatePurchase wishes_10` (unrestricted); `setPolicy true`; `Make{10}` with 3 free + 10 paid | `err = "restricted"`; `Make{1}` ×3 ok (free); `paidUnspent = 10` |
| `wish.restricted_no_luck` | `setPass lucky_jar true`; `setPolicy true`; `Wish.GetOdds` | `luckyJar = false`; flair standard = 97.4750% |
| `wish.lucky_odds_live` | `setPass lucky_jar true`; `Wish.GetOdds` | shiny = 3.0000%, standard = 96.2125% |
| `wish.odds_view_matches_server` | client computes `Odds.view`; server `Wish.GetOdds` | identical `oddsHash` and tier ppm |
| `wish.cosmic_craft` | `grant CosmicSparks 10`; `CraftCosmic` | owns `chaos`; Sparks 0; ledger hit `c`; second craft with 10 Sparks → `denied all_cosmic_owned` |
| `wish.escrow_release` | `grant Wishes` to 995; `simulatePurchase wishes_10` (receipt path, no cap check) | Wishes = 1000, `paidEscrow = 5`, `paidUnspent = 5`; `Make{10}` → Wishes 995 after release (990 + 5); escrow 0; `paidUnspent = 5` |
| `wish.config_change_rehash` | `setConfig Wish {"TierOdds":{"common":499900,"cosmic":200}}`; `Wish.GetOdds` | new `oddsHash`; the next ledger entry carries it |
| `wish.anomaly_signal` | Studio seam injects a pity regression | `[ALG][Mod] signal kind=wish_anomaly` |

---

### 5.3 PolicyGate (PolicyService gating)

API: §2.3.5 (`fetchAsync`, `get`, `devSet`).

**Fields used** (from `PolicyService:GetPolicyInfoForPlayerAsync(player)`, server side [Docs `PolicyService.yaml`]):

| `PolicyFlags` field | Source key | Rule |
|---|---|---|
| `paidRandomRestricted` | `ArePaidRandomItemsRestricted` | `true` unless the key is exactly `false` (missing → restricted) |
| `paidTradingAllowed` | `IsPaidItemTradingAllowed` | `true` only if exactly `true` (unused in M1-G; no trading) |
| `adsAllowed` | `AreAdsAllowed` | `true` only if exactly `true` (rewarded ads are a future hook) |
| `subscriptionEligible` | `IsEligibleToPurchaseSubscription` | `true` only if exactly `true` (Club is a future hook) |
| `source` | — | `"live"`, `"failsafe"` or `"dev"` |

Fetch algorithm (MUST):

```lua
--!strict
-- PolicyGate.fetchAsync (excerpt). Spec §5.3. Called by PlayerService join step 4 (§2.1.3 D).
function PolicyGate.fetchAsync(player: Player): Types.PolicyFlags
	local pc = Runtime.get().Economy.Policy
	for attempt = 1, pc.Attempts do                                 -- 3
		local ok, info = pcall(PolicyService.GetPolicyInfoForPlayerAsync, PolicyService, player)
		if ok and type(info) == "table" then
			return store(player, fromInfo(info, "live"))
		end
		Log.warn("Policy", "fetch_failed", { user = player.UserId, attempt = attempt, err = tostring(info) })
		if attempt < pc.Attempts then task.wait(pc.BackoffSec[attempt]) end -- 1 s, 2 s (outside any mutator)
		if player.Parent == nil then break end
	end
	local flags = failsafe()                                        -- all restrictive, source = "failsafe"
	if RunService:IsStudio() and pc.StudioFailOpen then            -- dev convenience only; never in live servers
		flags = { paidRandomRestricted = false, paidTradingAllowed = false, adsAllowed = false,
			subscriptionEligible = false, source = "dev" }
	end
	scheduleBackgroundRetries(player, pc.BackgroundRetrySec)       -- 30, 60, 120 s; on success store() again
	return store(player, flags)
end
```

- `store()` caches the flags per player, calls `DataService.setRuntime(player, "policy", flags)` (owner UI updates through the delta), and logs `[ALG][Policy] flags user=<id> restricted=<bool> source=<live|failsafe|dev>` once per change.
- `get(player)` never yields. It returns the cached flags, or `failsafe()` if none are cached yet.
- Join step 4 bounds the yield to about 10 s worst case (3 attempts × engine timeout is not ours to control). PlayerService does not wait on background retries.
- The cache is per player and dropped on leave. Flags are not refreshed mid-session except by background retries after a failsafe.
- `devSet(player, flags)` is Studio only (`Dev.Command setPolicy`) and sets `source = "dev"`.
- Never hard-code countries or ages (GDD §8.8).

**Consumers.** WishService (paid Wish spending and luck, §5.2.14), MonetizationService (store visibility and prompts, §5.6), and the UI (through `PrivateView.policy`).

**Verify.**
- Scenario `policy.failsafe_restricted`: the Studio seam `PolicyGate._forceFail = true` with `StudioFailOpen = false` → `restricted=true source=failsafe`; `Store.Get` hides `wishes_*` and `lucky_jar`.
- Scenario `policy.dev_override`: `setPolicy false` → `source=dev`; PRI items visible.
- Scenario `policy.live_in_studio` (spike, §5.14): log what `GetPolicyInfoForPlayerAsync` returns in Studio.
- Scenario `security.pri_server_rejects` (Part 2).

---

### 5.4 Choose Goose's Shop (GDD §8.5)

#### 5.4.1 Rules

- **Style slots:** `Shop.StyleSlots = 3`, drawn from pool styles with rarity ∉ `Shop.ExcludeRarities` (`starter`, `cosmic`), i.e. 15 styles. `IncludeUnimplemented = true` (§0.10: collectible).
- **Weighting toward unowned:** weight `UnownedWeight = 4` for unowned styles and `OwnedWeight = 1` for owned ones (tuning default). Owned styles can appear; they show as "Owned" and cannot be bought.
- **Cosmetic slots:** `Shop.CosmeticSlots = 4` with a placeholder catalog: **Variants of owned styles** (`v1..v<Wish.VariantsPerStyle>` not yet owned). Price `Shop.VariantPrices[rarity of the style]` (tuning default; GDD §17.5 cosmetics 500–8k). With no candidates, a slot is empty (`itemId = ""`). `CosmeticSlotsEnabled = true` (§5.12 notes the change from §2.6.2's draft). Emotes and Finales are future hooks, because PlayerData has no cosmetics inventory yet.
- **Prices** (Treasure only, GDD §8.5): common 1,500 · uncommon 3,000 · rare 6,000 · epic 12,000 · legendary 30,000 · mythic 75,000. Robux can never buy shop items (GDD §17.2).
- **Refresh:** every `Shop.RefreshSec = 43200` (12 h). Windows are aligned to 00:00 and 12:00 UTC: `period = Pity.periodIndex(now, 43200)`, `refreshAt = (period + 1) × 43200`.

#### 5.4.2 Deterministic rotation (pure)

```lua
--!strict
export type RotationInput = {
	userId: number, period: number, owned: { [string]: boolean },
	variantsOwned: { [string]: { [string]: boolean } },
	pool: Pool, shop: typeof(Defaults.Shop), variantsPerStyle: number,
}
export type Rotation = {
	styles: { Types.StyleId },                                -- length StyleSlots
	cosmetics: { { styleId: string, variantId: string } },    -- length CosmeticSlots; "" / "" = empty slot
	rotationId: string,
}
-- Odds.shopRotation(input: RotationInput): Rotation
-- Odds.parseRotationId(s: string): { period: number, styles: { Types.StyleId }, cosmetics: { { styleId: string, variantId: string } } }?
```

Algorithm (normative):
1. `seed = bit32.bxor(userId % 2^32, Odds.mul32(period % 2^32, 0x9E3779B9))`; `s = Odds.stream(seed)`. Negative Studio user IDs are valid: Luau `%` is floored, so `-1 % 2^32 = 4294967295`.
2. `cands` = eligible styles in `STYLE_IDS` order. For each style slot: `u = draw(s)`; weights per remaining candidate (owned 1, unowned 4); `i = pickWeighted(weights, u)`; take `cands[i]` and remove it.
3. `offers` = for each owned style in `STYLE_IDS` order, for `v = 1..variantsPerStyle` with `"v"..v` not owned: `{styleId, "v"..v}`. For each cosmetic slot: `u = draw(s)` (always drawn); if `offers` is non-empty, pick uniformly with `pickWeighted(ones, u)` and remove; else the slot is empty. If `CosmeticSlotsEnabled == false`, every cosmetic slot is empty and the draws still happen.
4. `rotationId = b36(period) .. ":" .. <one b36 char per style: 0-based STYLE_IDS index> .. ":" .. <comma-joined cosmetic codes>`. A code is `<b36 style index><variant digit>`, or `-` for empty. It is ≤ 32 chars and fits the `Shop.Buy` validator.

Vectors (Lune `shop.rotation_vectors`; defaults `StyleSlots 3`, `CosmeticSlots 4`, `VariantsPerStyle 2`, weights 4/1):

| userId | period | owned (besides starters) | variants owned | rotationId | styles | cosmetics |
|---|---|---|---|---|---|---|
| 1 | 41476 | — | — | `w04:97b:12,11,02,01` | flame, grappler, rocker | stretch/v2, stretch/v1, hero/v2, hero/v1 |
| 1 | 41477 | — | — | `w05:8fd:11,02,01,12` | monarch, demon, huntress | stretch/v1, hero/v2, hero/v1, stretch/v2 |
| 12345 | 41476 | — | — | `w04:c56:01,11,12,02` | frost, tyrant, console | hero/v1, stretch/v1, stretch/v2, hero/v2 |
| 12345 | 41477 | — | — | `w05:b7f:01,12,02,11` | rocker, grappler, demon | hero/v1, stretch/v2, hero/v2, stretch/v1 |
| 12345 | 41476 | monarch, rocker, sentry | rocker: v1 | `w04:c56:01,81,b2,82` | frost, tyrant, console | hero/v1, monarch/v1, rocker/v2, monarch/v2 |
| 12345 | 41477 | monarch, rocker, sentry | rocker: v1 | `w05:c7f:01,81,12,b2` | frost, grappler, demon | hero/v1, monarch/v1, stretch/v2, rocker/v2 |

Period 41476 is 2026-10-12 00:00 UTC (`now = 1791763200`). Also: `shop.rotation_id_roundtrip`, and `shop.rotation_owned_weighting` (10,000 periods for one user owning 7 of 15 styles: the share of unowned picks is within ±2 percentage points of the exact expectation, computed by enumerating the 3-step weighted draw).

#### 5.4.3 ShopService flow

- **Persistence of the composition.** On `Shop.Get` (and before `Shop.Buy`), if `Odds.parseRotationId(data.shop.rotationId)` is nil or its `period ≠ currentPeriod`:
  - compute `Odds.shopRotation(...)` with the **current** ownership;
  - store `rotationId` and reset `bought` to `false × (StyleSlots + CosmeticSlots)` in one update of `{"shop"}`;
  - log `[ALG][Shop] rotation user=<id> rid=<rotationId>`.

  The slots stay fixed for the whole window, even after purchases change ownership.
- **`Shop.Get` → `ShopView`.**
  - `rotationId`; `refreshAt`.
  - `slots[i] = {index, kind, itemId, rarity, price, currency = "Treasure", owned, bought}`. Style slots come first (1..3), then cosmetic slots (4..7). A cosmetic `itemId` is `"<styleId>/<variantId>"`.
  - When `CosmeticSlotsEnabled == false`, cosmetic slots are omitted from `slots`, but `bought` keeps its full length.
- **`Shop.Buy{slot, rotationId, price}`** (§2.7.4 row 12), as one update of `{"currencies","styles","shop"}`:

| Check (in order) | Error |
|---|---|
| `Features.Shop` | `disabled` |
| `rotationId == data.shop.rotationId` and its period is current | `stale` |
| `1 ≤ slot ≤ #slots`, slot not empty | `invalid` |
| `bought[slot] == false` | `denied` (`detail = "bought"`) |
| `price == computed price` | `stale` |
| Style slot: not owned. Cosmetic slot: style owned and variant not owned | `denied` (`detail = "owned"`) |
| `Treasure ≥ price` | `insufficient` |

  Grant inside the same mutator:
  - `tx.debit("Treasure", price, "shop_buy", {sku = "style:<id>"})` or `{sku = "variant:<id>/<v>"}`;
  - a style is granted with `StyleService.grantInTx(d, id, "shop", tx.now)`; Flair stays `standard` (the shop is deterministic and does not roll);
  - a variant is granted with `styles[id].variants[v] = true`;
  - `bought[slot] = true`.

  Response `ShopBuyRes{itemId, price, treasureLeft}`. Log `[ALG][Shop] buy user=<id> slot=<n> item=<id> price=<p>`. Notify `ui.notify.shop_bought`.
- Choose Goose's rhyming dialogue is UI text (Skin keys `ui.shop.goose_line_<n>`, n = 1…6, Part 6/8; the Original lines are plain prose, Part 8 §8.3.7).

**Verify.** Scenarios:
- `shop.rotation_stable`: two `Shop.Get` calls in the same window return the same `rotationId`; `setTimeOffset +43200` → a new one; `bought` is reset.
- `shop.buy_style`: `grant Treasure 1500`; buy a common slot → owned; Treasure 0; `bought = true`; rebuying → `denied`.
- `shop.buy_stale`: buy with an old `rotationId` → `stale`; with a wrong price → `stale`.
- `shop.buy_insufficient`.
- `shop.buy_variant`.
- `shop.unimplemented_purchasable`: buying `sentry` succeeds; equipping it → `not_implemented`.

---

### 5.5 Economy and progression

#### 5.5.1 Currencies (GDD §17.2)

| Currency | Sources (reasons) | Sinks (reasons) | Robux-purchasable |
|---|---|---|---|
| `Treasure` | `combat_ko`, `combat_assist`, `combat_damage`, `streak_bounty`, `level_up`, `daily_login`, `quest`, `world_event`, `wish_overflow`, `compensation`, `dev` | `shop_buy` (quest rerolls and Variant crafting are future hooks) | **No** |
| `Wishes` | `tutorial`, `level_up`, `daily_login`, `quest`, `world_event`, `combat_active`, `purchase`, `compensation`, `dev` | `wish` | Yes (dev products; PRI-gated) |
| `Shards` | `duplicate` | `shard_convert` (Mastery ≤ M20) | No |
| `CosmicSparks` | `mythic_dupe` | `cosmic_craft` | No |

Event tokens are out of scope.

#### 5.5.2 Earn table (GDD §17.5 casual target ≈ 14 Wishes and ≈ 1,775 Treasure per 45-min day)

| Source | Trigger | Treasure | Wishes | Account XP | Mastery XP | Config |
|---|---|---|---|---|---|---|
| KO | `CombatService.ko`, killer | 25 × decay | — | 25 × decay | 50 × decay (killer's style) | `Economy.CombatEarn.KoTreasure`, `Progression.Account.XpKo`, `Progression.Mastery.XpKo` |
| Assist | `ko.assists[*]` | 10 × decay | — | 10 × decay | — | `AssistTreasure`, `XpAssist` |
| Damage dealt | `CombatService.damage` | 5 per 100 HP (fractional accumulator) | — | — | 1 per HP (attacker's style) | `TreasurePer100Damage`, `Mastery.XpPerDamage` |
| Active minute | §5.5.3 | — | — | 10 | — | `XpPerActiveMinute` |
| Active combat Wish | every 900 cumulative active seconds | — | 1 | — | — | `CombatEarn.ActiveWishEverySec` (GDD §17.5 "1 per 15 min active") |
| Style Moment | `CombatService.styleMoment` | — | — | — | 100 | `Mastery.XpStyleMoment` |
| Streak bounty | `CombatService.streak` kind `ended`, to `endedBy` | 25 / 60 / 150 (`Combat.Streak.Bounty`, Part 3) | — | — | — | GDD §6.11 |
| Level-up | per level gained | 250 | 3 | — | — | `Account.RewardPerLevel` (GDD §9.1) |
| Daily login | first ready per UTC day | 100 | 2 | — | — | `Economy.DailyLogin` |
| Daily quest (§5.5.9; on in M1-G) | quest complete | 150 | 1 (+1 when all 3 are done) | 150 | — | `Economy.DailyQuest`, `XpDailyQuest` |
| World event (stub) | participation | 80 | 1 per 6 participations | 40 | — | `Economy.WorldEvent`, `XpWorldEvent` |
| Tutorial | first join | — | 3 | — | — | `Economy.TutorialWishes` |

**Multipliers and exclusions:**
- **Royal VIP:** every Treasure **source** except `wish_overflow`, `compensation` and `dev` is multiplied by `Economy.VipTreasureMultiplier` (1.10), then floored, by the caller (ProgressionService) when `MonetizationService.ownsPass(player, "vip")` (GDD §17.3; §2.8.5 rule 7).
- **Training:** if `MapId == "training"` and `Economy.TrainingEarns == false`, the KO, assist, damage, active-minute, active-Wish, Style Moment and bounty rows earn nothing.
- **Bots:** rewards derived from a bot victim (`victimBot ~= nil`) are multiplied by `Economy.BotRewardMultiplier` (0.5, tuning default).

**M1-G earn model (DES-7, §5.14).** With daily quests on, world events stubbed (`Features.WorldEvent = false`) and weekly quests a future hook, a casual 45-min day in M1-G earns ≈ **10.5 Wishes** (active 3 + daily quests 4 + login 2 + level-ups 1.5) and ≈ **1,025 Treasure** (combat 350 + daily quests 450 + login 100 + level-ups 125). That is ≈ 75 % of the GDD §17.5 Wishes (14) and ≈ 58 % of its Treasure (1,775): the missing rows are world events (1 Wish, ~500 Treasure) and amortized weekly quests (2.5 Wishes, 250 Treasure). The level-up row keeps the GDD amortization and slightly overstates M1-G, because world-event and weekly-quest XP are also missing. No GDD number changes; the owner accepts the gap or adds a temporary grant (DES-7).

#### 5.5.3 Activity and AFK detection (GDD §9.1 "AFK-detected and zeroed")

- ProgressionService runs one server loop every `Progression.Account.ActiveWindowSec` (60 s).
- For each ready player, the minute is **active** iff the player **dealt** at least `Progression.Account.ActiveMinDamage` (10, tuning default) damage to non-dummy targets during the window, outside Training.
- Taking damage does not count, so an AFK player being farmed earns nothing. This narrows Part 2's draft wording ("dealt or took ≥ 1"); see §5.12.
- An active minute adds `XpPerActiveMinute` and 60 s to `account.activeSec`. `account.activeSec` persists (§2.4). When `floor(activeSec / 900)` increases, 1 Wish is credited (`combat_active`).
- Inactive minutes add nothing. Partial windows at join and leave are dropped.

#### 5.5.4 Anti-farm (GDD §15.5, §16.2)

- **Repeat victim.** For each earner, keep a server-memory log of KO/assist timestamps per victim key (UserId string or bot id), pruned to `Economy.RepeatVictim.WindowSec` (600 s).
  - The *n*th reward on the same victim inside the window uses `Pity.repeatMultiplier(n, {1, 0.5, 0.25, 0})` = `mults[min(n, #mults)]`.
  - The multiplier applies to Treasure, Account XP and Mastery XP from that KO or assist. XP is included because level-ups pay Wishes (tuning default; the GDD names Treasure).
- **Rookie rule** (per earner, GDD §15.5 "no Treasure from killing Rookies more than twice per 10 minutes"). If the victim's `Rookie` attribute is true and the earner's is false, the earner gets **0 Treasure** (XP still follows the repeat multiplier) from the 3rd and later Rookie KO (**any** Rookie, not only the same one) within `Economy.RookieKill.WindowSec` (600 s, `MaxTreasureKills = 2`).
  - The server-memory log is keyed **per earner**: `rookieKos[earner] = { t1, t2, … }` holds the timestamps of the earner's **paid** Rookie KOs (all victims together), pruned to `WindowSec`. A Rookie KO pays Treasure iff fewer than `MaxTreasureKills` paid entries remain in the window; only paid KOs are appended.
  - Pure helper `Pity.rookieKoPays(log: { number }, now: number, cfg: { WindowSec: number, MaxTreasureKills: number }) -> boolean` = (count of `t` in `log` with `now − t < WindowSec`) `< MaxTreasureKills`; the caller appends `now` only when it returns true, and prunes expired entries.
  - Lune `progression.rookie_rule_per_earner`: KOs of Rookies R1, R2, R3 at t = 0, 60, 120 → paid, paid, **unpaid**; a 4th Rookie KO at t = 601 → paid (the t = 0 entry has expired).
- **Rookie status.** `flags.rookieSecLeft` (initially `Progression.Rookie.DurationSec` = 7,200) decreases by session play seconds at every flush. At 0, ProgressionService sets `Rookie = false` and sends Notify `ui.notify.rookie_ended`.
- **Spawn protection.** Protected spawn (8 s, broken by attacking) is Part 3 and applies to everyone (§3.13.5).
- **Economy velocity.** DataService raises `economy_velocity` above `Economy.VelocityAlertTreasurePerHour` (§2.11.3).

#### 5.5.5 Accumulators and flush

- Per-player server memory: `{treasure = {[reason] = number}, wishes = {[reason] = number}, xp = number, mastery = {[styleId] = number}, life = {treasure, xp, mastery}, minuteDamage, victims}`. Fractional Treasure from damage stays in memory until it reaches an integer.
- **Flush** (one `DataService.update` over `{"currencies","account","styles","flags","quests"}`) happens:
  - every `Economy.FlushSec` (30 s);
  - in the `lifeEnded` handler, which must not yield (`DataService.update` is synchronous; §3.13.7);
  - in `DataService.beforeRelease` (synchronous, §2.8.2).
- The flush mutator, in order:
  1. Credit Treasure per reason (VIP applied).
  2. Credit Wishes per reason (earned; overflow converts to Treasure, §5.2.10).
  3. Add XP. While `account.xp ≥ toNext`, level up: `+3 Wishes` and `+250 Treasure × VIP` per level (`level_up`). Cap at `MaxLevel` 100; XP beyond the cap accumulates only in `xpTotal`. Prestige is a future hook.
  4. Add Mastery XP to **owned** styles only. Trial and rotation styles accrue none. Grant Mastery rewards (§5.5.7). **Title auto-equip:** whenever this step or step 3 grants a title (`account.titles[id] = true`, newly) while `account.title == ""`, set `account.title = id` (the first title granted becomes the equipped one; later grants never replace it). There is no `title` settings key and no title picker in M1-G (Part 6 §6.15 future hook).
  5. Decrement `rookieSecLeft`.
- After commit:
  - write `Level`, `MasteryLevel`, `Rookie`, `Title` attributes on change;
  - Notify `ui.notify.level_up {level}` / `ui.notify.mastery_up {styleId, level}`;
  - Telemetry progression events;
  - log `[ALG][Progress] flush user=<id> treasure=<n> wishes=<n> xp=<n> levels=<n>` (only when non-zero) and `[ALG][Progress] level user=<id> level=<n>`.
- In `lifeEnded(player, summary)`: fill `summary.treasureEarned`, `xpEarned` and `masteryXpEarned` from `life` and reset it; flush; emit `match_summary` (§5.8).

#### 5.5.6 Account level (GDD §9.1)

- `XP_to_next(n) = 300 + 45·n` (`XpBase + XpPerLevel·n`), levels 1–100.
- `Pity.accountLevel(xpTotal, cfg)` returns `(level, xpInto, toNext)`, with `toNext = 0` at level 100.
- Vectors (`progression.level_from_xp_vectors`):

| xpTotal | Result |
|---|---|
| 0 | (1, 0, 345) |
| 344 | (1, 344, 345) |
| 345 | (2, 0, 390) |
| 252449 | (99, 4754, 4755) |
| 252450 | (100, 0, 0) |
| 10^7 | (100, 0, 0) |

  `progression.account_curve` asserts the total to reach level 100 is 252,450 (GDD "≈ 252k").
- **Rewards:** 3 Wishes and 250 Treasure per level.
- **Cosmetic every 10 levels** (data only in M1-G): `account.titles["level_<n>"] = true` for n = 10, 20, …, with Skin key `title.level_<n>`. Emote and Finale rewards are future hooks.

#### 5.5.7 Style Mastery (GDD §9.2)

- **Curve** (tuning default from §2.6.2): `XP_to_next(m) = 400 + 80·m`. `Pity.masteryXpToReach(L) = 400·(L−1) + 40·(L−1)·L`.
- Vectors (`progression.mastery_curve`):

| Query | Expected |
|---|---|
| `masteryXpToReach(2)` | 480 |
| `masteryXpToReach(20)` | 22,800 |
| `masteryXpToReach(30)` | 46,400 |
| `masteryLevel(479)` | 1 |
| `masteryLevel(480)` | 2 |
| `masteryLevel(46399)` | 29 |
| `masteryLevel(10^9)` | 30 |

- **XP sources:** §5.5.2 (damage 1/HP, KO 50, Style Moment 100).
- **`Mastery.SpendShards{styleId, shards}`** (§2.7.4 row 7), one update of `{"currencies","styles"}`:
  - the style must be owned, else `not_owned`;
  - `level = masteryLevel(xp)`; if `level ≥ ShardCapLevel` (20) → `denied` (`detail = "cap"`);
  - `remaining = masteryXpToReach(20) − xp`; `spent = min(shards, ceil(remaining / ShardXp))`; `gained = min(spent × 60, remaining)`;
  - `Shards ≥ spent`, else `insufficient`;
  - `tx.debit("Shards", spent, "shard_convert", {sku = "mastery:<styleId>"})`; `masteryXp += gained`;
  - return `{shardsSpent, xpGained, level}`.

  Vector (`progression.shard_clamp`): xp 22,790 and 5 shards requested → spent 1, gained 10, level 20. Another request → `denied cap`. Mastery 21–30 can only come from combat.
- **Rewards:** data table `Progression.Mastery.Rewards` (GDD §9.2).
  - On reaching each reward level above `masteryClaimed`, the flush sets `masteryClaimed = level`.
  - Titles are granted for real: `title.mastery_<styleId>_5` at M5 and `title.master_<styleId>` at M30, stored in `account.titles`.
  - Every other reward kind is logged `[ALG][Progress] reward_pending user=<id> style=<id> level=<n> reward=<kind>` (art pending).
- **Attribute:** `MasteryLevel` = the level of the life's style (1 for unowned trial/rotation styles), written on `lifeStyleResolved` and on level-up.

#### 5.5.8 Time, daily and weekly reset (UTC, server time)

- `Pity.dayIndex(t) = t // 86400`; `Pity.weekIndex(t) = (t // 86400 + 3) // 7` (weeks start Monday 00:00 UTC; 1970-01-01 was a Thursday); `Pity.periodIndex(t, s) = t // s`.
- Vector (`time.week_index_monday`): `t = 1791763200` (Monday 2026-10-12 00:00 UTC) → day 20738, week 2963; `t − 1` → week 2962.
- **Daily login:**
  - On `ProgressionService.onPlayerReady`, and every `Progression.DailyResetCheckSec` (60 s) for online players: if `flags.lastLoginDay < today`, credit `DailyLogin` (2 Wishes, 100 Treasure, reasons `daily_login`).
  - `loginStreak = if lastLoginDay == today − 1 then loginStreak + 1 else 1`; `lastLoginDay = today`.
  - Notify `ui.notify.daily_login`. The streak curve is a future hook; M1-G pays the flat GDD §17.5 average.
- **Quests:** reset at day/week rollover (§5.5.9). All time comes from `DataService.now()`, so `Dev.Command setTimeOffset` tests resets.

#### 5.5.9 Daily quests (`Progression.Quests.Enabled = true` in M1-G)

- **Pool:** `Progression.Quests.DailyPool` (5 entries; GDD §9.5 examples; tuning default):

| id | stat | goal |
|---|---|---|
| `deal_damage` | damage | 600 |
| `perfect_blocks` | perfectBlocks | 10 |
| `kos` | kos | 5 |
| `active_minutes` | activeMinutes | 15 |
| `awaken` | awakenings | 1 |

- **Selection:** at day rollover (or first ready of the day), `Odds.dailyQuestPick(userId, dayIndex, #DailyPool, DailyCount)`: seed `bxor(userId % 2^32, mul32(day, 0x85EBCA6B))`, uniform without replacement. Store `quests.daily = {QuestProgress}` and `lastDailyReset = day`.
- **Progress** comes from `DamageEvent` (dealt), `LifeSummary.perfectBlocks`/`kos`/`awakened`, and active minutes. It is not counted on Training.
- **Claim:** automatic when `progress ≥ goal`. Rewards: `DailyQuest.Treasure` (150 × VIP), `DailyQuest.Wishes` (1), `XpDailyQuest` (150). When all 3 are claimed, add `AllDoneBonusWishes` (1). That totals 450 Treasure and 4 Wishes, matching GDD §17.5.
- Rerolls (`DailyRerolls`, VIP +1) and weekly quests are future hooks: they need a `Quest.Reroll` remote and a weekly pool.
- **UI:** Part 6 HUD element **Quests** (§6.9.5): 3 rows from `PrivateView.quests` (`Skin.text("ui.quest.<id>", {goal})`, progress bar), header `ui.hud.quests`, hidden in Simple HUD. After the flush commit that auto-claims a quest, ProgressionService sends `Net.server.fire("Notify", player, {kind = "reward", key = "ui.notify.quest_done", scope = "self"})` and logs `[ALG][Progress] quest_done user=<id> quest=<id>` once per claimed quest. This server Notify is the only quest-done toast; Part 6 adds no client-side toast for the same claim. Scenario `hud.quests_progress` (Part 6).
- **Skin lines:** every pool `id` needs a `ui.quest.<id>` line (token `{goal}`); Part 8 §8.3.7 defines the Original lines for the five ids above, and a task that adds a `DailyPool` entry adds its line in the same task.
- **Why on in M1-G (DES-7):** with quests off, M1-G earned ≈ 6.5 Wishes/day (≈ 46 % of GDD §17.5); turning them on closes most of the gap without a hidden-reward path (a flag flip on a server without the HUD element would grant invisible rewards).
- The Studio seam `_simActivity` feeds quest progress exactly like real activity: each simulated active minute adds 1 to `activeMinutes` and `dmg` to `damage`.
- Verify: Lune `quests.pick_deterministic`; scenario `progress.daily_quest_claim` (§5.5.12).

#### 5.5.10 Streaks and world events

- `CombatService.streak` (§3.13.3):
  - `kind = "tier"` → `Net.server.fireAll("Notify", {kind = "announce", key = "ui.notify.streak_<tier>", args = {who = player.DisplayName}, scope = "server"})`.
  - `kind = "ended"` with `endedBy` → accumulate `bounty × VIP` Treasure (`streak_bounty`) for `endedBy`, and Notify `ui.notify.streak_ended`.
- World event: the M1-G stub (Features.WorldEvent false) exposes `ProgressionService.worldEventParticipation(player)`, which applies the §5.5.2 row. Part 6's stub zone calls it when enabled.

#### 5.5.11 ProgressionService API

```lua
--!strict
export type ProgressionServiceApi = {
	-- Part 2 §2.3.5
	onPlayerReady: (player: Player) -> (),         -- daily login, attributes Level/Prestige/Title/Rookie, quest rollover
	masteryLevel: (xp: number, cfg: Types.Config) -> number,                     -- = Pity.masteryLevel(xp, cfg.Progression.Mastery)
	accountLevelFromXp: (xpTotal: number, cfg: Types.Config) -> (number, number), -- = Pity.accountLevel (first two returns)
	-- Part 5 additions
	worldEventParticipation: (player: Player) -> (),
	ftueStep: (player: Player, step: number, skipped: boolean) -> (), -- §5.8.4
	-- Studio-only seams (no-ops elsewhere):
	_simActivity: (player: Player, seconds: number, damagePerMinute: number) -> (),
	_simKo: (killer: Player, victimKey: string, victimRookie: boolean) -> (),
	_flushNow: (player: Player) -> (),
}
```

- `start()` connects `CombatService.damage`, `ko`, `lifeEnded`, `streak`, `styleMoment`; `StyleService.lifeStyleResolved`; `DataService.beforeRelease`. These are module-lifetime connections (commented as such per §2.10 item 9).
- New DevCmds (§5.12): `simActivity {sec, dmg}`, `simKo {victim, rookie}`, `flushProgress {}`.

#### 5.5.12 Verify (§5.5)

| Kind | Name | Expected |
|---|---|---|
| Lune | `progression.account_curve`, `progression.level_from_xp_vectors`, `progression.mastery_curve`, `progression.shard_clamp`, `progression.repeat_victim_multipliers` (n = 1..6 → 1, 0.5, 0.25, 0, 0, 0), `progression.rookie_rule_per_earner` (§5.5.4), `time.week_index_monday`, `quests.pick_deterministic` | as stated |
| Scenario | `progress.afk_zero_xp` | `simActivity {sec=600, dmg=0}` → XP unchanged; `{sec=600, dmg=50}` → +100 XP |
| Scenario | `progress.active_wish` | `simActivity {sec=900, dmg=50}`; flush → +1 Wish (`combat_active`) |
| Scenario | `progress.level_up_rewards` | fresh profile; `simActivity {sec=2100, dmg=50}`; `flushProgress` → 35 active minutes = 350 XP ≥ 345 → level 2; Wishes +5 (3 level-up + 2 `combat_active`); Treasure +250 (`[ALG][Progress] level user=<id> level=2`) |
| Scenario | `progress.repeat_kill_decay` | 4 × `simKo {victim="b1"}` within 10 min; `flushProgress` → Treasure +21 (25 × 0.5 bot × (1 + 0.5 + 0.25 + 0) = 21.875, floored; the 0.875 remainder stays in memory) |
| Scenario | `progress.rookie_rule` | killer non-Rookie; `simKo` on three **different** Rookie victims (`{victim="-101", rookie=true}`, `"-102"`, `"-103"`) within 10 min → the first two pay, the 3rd yields 0 Treasure (per-earner rule, §5.5.4) |
| Scenario | `progress.daily_quest_claim` | quests on (default); fresh day; `simActivity {sec=900, dmg=50}` (15 active minutes, 750 damage) → `flushProgress` → every picked quest with stat `damage` (goal 600) or `activeMinutes` (goal 15) has `claimed = true` (at least one is always picked: 3 of 5); one `[ALG][Progress] quest_done` line per claim |
| Scenario | `progress.training_no_earn` | `travel training`; `simActivity` → no XP, no Treasure |
| Scenario | `progress.daily_login` | join → +2 Wishes +100 Treasure once; `setTimeOffset +86400` → granted again within 60 s; `loginStreak = 2` |
| Scenario | `progress.overflow_to_treasure` | `grant Wishes` to 999; `setTimeOffset +86400` → daily login (+2 Wishes, +100 Treasure) → Wishes 1000, Treasure +110 (100 `daily_login` + 10 `wish_overflow`) |
| Scenario | `progress.spend_shards` | `grant Shards 400`; `Mastery.SpendShards {hero, 400}` → level 20; Shards 20 (380 spent) |
| Scenario | `progress.life_summary_filled` | KO a bot → the client's `Life.Summary.treasureEarned > 0` |

Quests are on in M1-G, so simulated activity can complete a quest and add rewards. Every `progress.*` scenario with exact Wish/Treasure/XP expectations (all rows above except `progress.daily_quest_claim`, `progress.training_no_earn` and `progress.life_summary_filled`) starts with `Dev.Command setConfig Progression {"Quests":{"Enabled":false}}` and restores the overlay in cleanup.

---

### 5.6 MonetizationService (mock-first)

#### 5.6.1 Catalog (GDD §17.3, M1-G subset; keys from `Types.ProductKey` / `Types.PassKey`)

| Key | Kind | Contents | PRI | Reference price (R$) | Notes |
|---|---|---|---|---|---|
| `wishes_10` | dev product | 10 Wishes | yes | 99 | hidden when restricted |
| `wishes_55` | dev product | 55 Wishes | yes | 449 | |
| `wishes_120` | dev product | 120 Wishes | yes | 899 | equals the Legendary+ hard pity |
| `wishes_300` | dev product | 300 Wishes | yes | 1,999 | |
| `vip` | game pass | +10% Treasure, `Vip` attribute (nameplate frame), +1 quest reroll (future), VIP emote (future) | no | 299 | |
| `quick_wish` | game pass | instant reveals after each rarity's first view (GDD §8.6) and auto-10× (client, Part 6 §6.9.7); history export (future) | no | 99 | |
| `lucky_jar` | game pass | ×1.5 Flair luck (§5.2.5) | yes (paid luck boost) | 349 | hidden when restricted |

- Reference prices live in `Economy.Products/Passes[key].RefPriceRobux`. They are used **only** for the mock store display and the H-09 checklist.
- Live prices always come from `MarketplaceService:GetProductInfoAsync(id, Enum.InfoType.Product | Enum.InfoType.GamePass).PriceInRobux` [Docs], cached for `Economy.ProductInfoCacheSec` (600 s) and fetched under `pcall`. The deprecated `GetProductInfo` is never used.
- **IDs:** `Assets.product(key)` = `Assets.products[game.GameId][key]` (§2.9.1). `[0]` is the unpublished-Studio entry (mock only).
- **The agent never creates products or passes and never sets prices** (H-09). It writes the IDs the human provides into `Assets.luau`. Product names are IP-neutral ("10 Wishes", "Royal VIP", "Quick Wish", "Lucky Jar").

#### 5.6.2 Mock mode

- `MonetizationService.isMock() = Runtime.get().MockPurchases == true and _isStudio`, where `_isStudio = RunService:IsStudio()` is the test seam of §2.12 item 13.
- Outside Studio, `MockPurchases` is ignored, with a one-time log `[ALG][Store] ERROR mock_purchases_ignored_outside_studio` (§2.11.4).
- In mock mode:
  - `Store.Prompt` for a product calls the **same** `grantProduct` used by real receipts, with `purchaseId = "mock-" .. HttpService:GenerateGUID(false)` and `robuxSpent = RefPriceRobux`, and returns `{status = "mock_granted"}`.
  - For a pass, it sets `mockPasses[userId][key] = true` (server memory; it survives rejoin within the Studio session) and fires `passChanged`.
  - `Dev.Command simulatePurchase {key}` **simulates a granted receipt**. It calls `grantProduct` directly with a fresh mock PurchaseId and skips every prompt-time check (policy, cap, confirmation), exactly like a real receipt arriving late. Escrow (§5.2.10) and idempotency are tested this way.
  - `Dev.Command setPass {pass, owned}` sets or clears mock ownership.
- Log `[ALG][Store] mock grant user=<id> key=<k> pid=<id>`.

#### 5.6.3 Receipt handling (decision: `BindReceiptHandler`)

`MarketplaceService:BindReceiptHandler(Enum.ReceiptType.DeveloperProduct, handler)` is verified [Docs `MarketplaceService.yaml`]:
- the handler returns `Enum.ReceiptDecision.Processed`/`NotProcessedYet`;
- a bound handler takes precedence over `ProcessReceipt`;
- a duplicate registration throws.

MonetizationService registers **one catch-all** handler (no filter) in `start()`, inside `pcall`, and never assigns `ProcessReceipt`. If the call errors (API unavailable), it logs `[ALG][Store] WARN bind_receipt_failed` and falls back to assigning `MarketplaceService.ProcessReceipt` with the identical body, mapping the result to `Enum.ProductPurchaseDecision.PurchaseGranted`/`NotProcessedYet` [Docs]. Exactly one of the two is ever active.

```lua
--!strict
-- MonetizationService (excerpt). Spec §5.6.3.
export type GrantResult = "granted" | "duplicate" | "failed"

function MonetizationService.grantProduct(player: Player, key: Types.ProductKey, purchaseId: string,
	robuxSpent: number, origin: "receipt" | "mock"): GrantResult
	local cfg = Runtime.get()
	local product = cfg.Economy.Products[key]
	local result: GrantResult = "failed"
	local ok = DataService.update(player, { "currencies", "wishStats", "purchases" }, "purchase", function(d, tx)
		if table.find(d.purchases.ids, purchaseId) ~= nil then
			result = "duplicate"                                    -- idempotent on PurchaseId (receipts may repeat or run on 2 servers)
			return
		end
		local room = math.max(0, cfg.Wish.StorageCap - d.currencies.Wishes)
		local direct = math.min(product.Wishes, room)
		if direct > 0 then tx.credit("Wishes", direct, "purchase", { purchased = true, sku = key }) end
		d.wishStats.paidUnspent += direct
		d.wishStats.paidEscrow += product.Wishes - direct             -- §5.2.10 race fallback
		pushFifo(d.purchases.ids, purchaseId, cfg.Data.PurchaseIdCacheSize)
		pushFifo(d.purchases.wishBuyTimes, math.floor(tx.now), cfg.Data.WishBuyHistorySize)
		d.purchases.robuxSpent += robuxSpent
		local week = Pity.weekIndex(tx.now)
		if d.purchases.weekIndex ~= week then d.purchases.weekIndex, d.purchases.weekRobux = week, 0 end
		d.purchases.weekRobux += robuxSpent
		result = "granted"
	end)
	return if ok then result else "failed"
end

local function onReceipt(receipt: { [string]: any }): Enum.ReceiptDecision
	local player = Players:GetPlayerByUserId(receipt.PlayerId)
	if player == nil then return Enum.ReceiptDecision.NotProcessedYet end      -- redelivered on rejoin
	local key = keyForProductId(receipt.ProductId)                             -- reverse map of Assets.products[game.GameId]
	if key == nil then
		Log.error("Store", "unknown_product", { pid = receipt.PurchaseId, product = receipt.ProductId })
		return Enum.ReceiptDecision.NotProcessedYet
	end
	if not waitLoaded(player, Runtime.get().Economy.ReceiptLoadWaitSec) then return Enum.ReceiptDecision.NotProcessedYet end
	if PolicyGate.get(player).paidRandomRestricted then
		Log.warn("Store", "receipt_while_restricted", { user = player.UserId, key = key }) -- still granted: it is paid for
	end
	local res = MonetizationService.grantProduct(player, key, receipt.PurchaseId, receipt.CurrencySpent or 0, "receipt")
	if res == "failed" then return Enum.ReceiptDecision.NotProcessedYet end
	-- ProfileStore "Caching PurchaseId's" pattern: confirm the ID reached the saved copy before acknowledging.
	DataService.saveAsync(player)
	if waitSaved(player, receipt.PurchaseId, Runtime.get().Economy.ReceiptSaveWaitSec) then
		Log.info("Store", "receipt", { user = player.UserId, key = key, pid = receipt.PurchaseId, result = res })
		Telemetry.funnel(player, "store", sessionOf(player), 3, "granted", { CustomField01 = key })
		Net.server.fire("Notify", player, { kind = "reward", key = "ui.notify.purchase_granted", args = { productKey = key }, scope = "self" })
		return Enum.ReceiptDecision.Processed
	end
	return Enum.ReceiptDecision.NotProcessedYet
end
```

- `waitLoaded` polls `DataService.isLoaded(player)` every 0.5 s.
- `waitSaved` polls `DataService.profile(player).LastSavedData.purchases.ids` for the ID every 0.5 s, and returns false if the session ends.
- Both use `task.wait`, which is allowed because they run outside mutators. The handler may yield [Docs: "no timeout; the callback may yield"].
- `pushFifo` appends and removes index 1 while `#t > cap`, so the ring stays dense.

#### 5.6.4 Store view and prompts

- **`Store.Get` → `StoreView`.**
  - `items` for every product and pass key, each `{key, kind, priceRobux, owned, hidden}`:
    - `priceRobux` is the live price (or `RefPriceRobux` in mock);
    - `hidden = (Pri and restricted) or (not mock and Assets.product(key) == nil)`.
  - `restricted`, `mock`, `wishBuysLast24h` (count of `wishBuyTimes` within 86,400 s), plus `spentThisWeekRobux` (new, from `purchases.weekRobux` when `weekIndex` is current; GDD §8.8 "spent this week" summary).
  - Telemetry store funnel step 1 `store_view`; this starts a new `funnelSessionId = HttpService:GenerateGUID(false)` per open.
- **`Store.Prompt{key, entryPoint, confirmed?}`** (§2.7.4 row 14), checks in order:

| Check | Response |
|---|---|
| `Features.Store` | `disabled` |
| PRI key and restricted | `restricted` |
| Pass already owned | `denied` (`detail = "owned"`) |
| Wish pack and `Wishes + paidEscrow + pack > StorageCap` | `cap_reached` |
| Wish pack, `wishBuysLast24h ≥ SpendGuard.ConfirmFromPurchase − 1` (i.e. this would be the 4th purchase in 24 h) and not `confirmed` | `{ok=true, data={status="needs_confirm"}}` (GDD §8.8 soft confirmation) |
| No product ID and not mock | `disabled` (`detail = "no_product_id"`) |
| Mock | grant as in §5.6.2 → `mock_granted` |
| Live | `MarketplaceService:PromptProductPurchase(player, id)` or `PromptGamePassPurchase(player, id)` → `prompted` |

  Telemetry funnel step 2 `prompt` with `{CustomField01 = key, CustomField02 = entryPoint, CustomField03 = tostring(priceShown)}`.
- **No dark patterns** (GDD §17.1). The server never prompts on its own. `Economy.NoOfferAfterLossSec` (30 s) is a Part 6 UI rule: no unsolicited Wish offers within 30 s of the player's KO.
- `PromptProductPurchaseFinished` is UI-only and never grants (§2.11.1).

#### 5.6.5 Game passes

- **`onPlayerReady`:**
  - for each `PassKey`, in parallel threads: mock → `mockPasses`; live → `pcall(MarketplaceService.UserOwnsGamePassAsync, MarketplaceService, player.UserId, id)` [Docs; cached by the engine];
  - after 5 s, unresolved checks count as `false` and complete in the background;
  - set `DataService.setRuntime(player, "passes", map)` and the `Vip` attribute.
- **`PromptGamePassPurchaseFinished(player, gamePassId, wasPurchased)`:** if `wasPurchased`, re-check ownership with `UserOwnsGamePassAsync`, update the cache, and fire `passChanged(player, key, true)`.
- `ownsPass(player, key)` never yields: it reads the cache.
- **Effects:**
  - `vip` → Treasure multiplier (§5.5.2) and `Vip` attribute;
  - `lucky_jar` → §5.2.5;
  - `quick_wish` → `PrivateView.passes.quick_wish` (Part 6 §6.9.7: the first view of each rarity still plays in full, GDD §8.6, and every later reveal is instant; [Auto ×10] sends each batch as a normal `Wish.Make{count = 10}` only after the previous response and ≥ 2.0 s after the previous send, so it stays under the rate limit).

#### 5.6.6 MonetizationService API (Part 2 contract plus additions)

```lua
--!strict
export type MonetizationServiceApi = {
	onPlayerReady: (player: Player) -> (),
	ownsPass: (player: Player, pass: Types.PassKey) -> boolean,
	passChanged: Signal.Signal<Player, Types.PassKey, boolean>,
	-- Part 5 additions
	isMock: () -> boolean,
	grantProduct: (player: Player, key: Types.ProductKey, purchaseId: string, robuxSpent: number,
		origin: "receipt" | "mock") -> GrantResult,
	setMockPass: (player: Player, pass: Types.PassKey, owned: boolean) -> (), -- Studio only
	_isStudio: boolean,                                                         -- test seam (§2.12 item 13)
}
```

#### 5.6.7 Verify (§5.6)

| Scenario | Expected |
|---|---|
| `store.mock_grant_wishes` | `Store.Prompt{key="wishes_10", entryPoint="test"}` → `mock_granted`; Wishes +10; `paidUnspent +10`; `[ALG][Store] mock grant` |
| `store.receipt_idempotent` | Call `grantProduct` twice with the same `purchaseId` → second result `duplicate`; Wishes +10 once |
| `store.cap_blocked` | Wishes 995 → `Store.Prompt wishes_10` → `cap_reached` |
| `store.fourth_purchase_confirm` | 3 mock purchases, 4th without `confirmed` → `needs_confirm`; with `confirmed=true` → granted |
| `store.pri_hidden_restricted` | `setPolicy true`; `Store.Get` → `wishes_*` and `lucky_jar` have `hidden=true`; `vip` visible |
| `store.mock_ignored_live` | Part 2 `security.mock_ignored_live` |
| `store.pass_mock_vip` | `setPass vip true` → `Vip` attribute true; Treasure from a KO ×1.1 |
| `store.week_spend` | mock `Store.Prompt` for `wishes_10` then `wishes_55` → `spentThisWeekRobux = 548` (99 + 449, mock reference prices) |
| `store.receipt_live_spike` (human, H-09; tags `{"m1g", "spike"}`) | In the dev universe with real dev products and `MockPurchases=false`: a Studio test purchase reaches `onReceipt`; `[ALG][Store] receipt ... result=granted`. Tagged `spike`: it needs optional H-09 products, so every gate A4 selector excludes it with `-@spike` (Part 7 §7.7.2/§7.7.3); it runs only when MON-1 is investigated |

---

### 5.7 Appearance and the Character Look toggle (owner's change; GDD §12.3; §0.4)

#### 5.7.1 Data (persisted in `settings`, §2.8.6)

| Setting | Type | Default | Scope | Set via |
|---|---|---|---|---|
| `characterLook` | boolean | true | global | `Appearance.Set{characterLook}` |
| `characterLookByStyle` | `{[StyleId]: boolean}` | `{}` | per-style override (absent = inherit) | `Appearance.Set{styleOverride = {styleId, value = "on"\|"off"\|"inherit"}}` |
| `ownLook` | `"Oosona"` \| `"Avatar"` | `"Oosona"` | global | `Appearance.Set{ownLook}` |
| `showStyleGear` | boolean | true | global | `Appearance.Set{showStyleGear}` |
| `oosona` | `OosonaSpec` | §2.8.6 default | global | `Appearance.Set{oosona}` |
| `othersAppearance` | `AsChosen` \| `AlwaysCharacter` \| `AlwaysOwn` \| `Simplified` | `AsChosen` | **viewer-local**, never replicated | `Settings.Set` |

**Effective Character Look** (pure; shared helper `AppearanceService.effectiveLook` mirrored on the client): `effective(settings, styleId) = settings.characterLookByStyle[styleId]` if present, else `settings.characterLook`. It is available for every owned, trialing or rotation style from the moment it is obtained; it is not gated by Mastery (GDD §12.3).

**Ooo-sona placeholder** (graybox): blocky R15 with three colours from the player's choices.
- Head and arms use the species skin: `Appearance.OosonaSpecies[species].skin`.
- The torso uses the player-chosen `color`.
- The legs use `Appearance.OutfitColors[outfit]`, or `color` darkened by `Appearance.NoOutfitDarken` (0.65) when `outfit = 0`.
- Face, hat and outfit indices select placeholder parts from `ReplicatedStorage.Looks` (Part 6).
- The `OosonaCode` attribute carries the spec (`Schema.encodeOosona`, ≤ 30 chars).

#### 5.7.2 Replication (Player attributes, §2.5.2)

| Attribute | Written by AppearanceService when | Value |
|---|---|---|
| `CharacterLook` | ready; `Appearance.Set`; `lifeStyleResolved` | effective look for the **current life's** style |
| `ShowStyleGear` | ready; `Appearance.Set` (immediate) | `settings.showStyleGear` |
| `OosonaCode` | ready; `Appearance.Set` (immediate) | `Schema.encodeOosona(settings.oosona)` |
| `OwnLook` | ready; **`lifeStyleResolved` only** (clarification of §2.5.2, §5.12) | the base look the current character was loaded with |

`OwnLook` changes take effect on the next spawn because Avatar loading is a spawn-time property (below). The owner's settings UI shows "applies on respawn" for that one field. The other three attributes update immediately; rendering is cosmetic and the hurtbox is unaffected.

#### 5.7.3 Spawn flow (server)

AppearanceService connects `StyleService.lifeStyleResolved`. The handler is non-yielding and runs before `LoadCharacterAsync`:

```lua
--!strict
-- AppearanceService.onLifeStyleResolved (excerpt). Spec §5.7.3.
local function onLifeStyleResolved(player: Player, styleId: Types.StyleId)
	local data = DataService.get(player)
	if data == nil then return end
	local s = data.settings
	-- Player.CanLoadCharacterAppearance: script-writable; takes effect on the next LoadCharacterAsync [Docs Player.yaml].
	player.CanLoadCharacterAppearance = (s.ownLook == "Avatar")
	player:SetAttribute("OwnLook", s.ownLook)
	player:SetAttribute("CharacterLook", AppearanceService.effectiveLook(s, styleId))
end
```

- **Base body.**
  - `OwnLook = "Avatar"` → the engine loads the player's Roblox avatar (`CanLoadCharacterAppearance = true`).
  - `OwnLook = "Oosona"` → the default blocky appearance (`CanLoadCharacterAppearance = false` → "the player will spawn with a default appearance" [Docs]).
  - Avatar Settings (H-05: Consistent Gameplay, Single Collider, R15) keep the rig R15 and the collider fixed.
- **Everything else is client-rendered** (Part 6 AppearanceController), per viewer:
  - the Character look (style palette rig from `StyleDef.palette` + Style Gear props from `ReplicatedStorage.Looks`);
  - the Ooo-sona colours and parts;
  - Style Gear on own looks;
  - "Simplified".
- **Hurtbox invariance.** The hurtbox is Part 3's fixed capsule (box) welded to the HRP (§3.12.2). Appearance never changes collision, query groups or hurtbox size.
  - Client-rendered look parts use `CanQuery = false` and group `CosmeticNoQuery` (§3.18 item 8; Part 6).
  - For server-loaded avatar accessories, AppearanceService's `CharacterAdded` handler (outside the sim) sets every `Accessory` `Handle` to `CanCollide = false`, `CanQuery = false`, `CanTouch = false`, `Massless = true`. The collision group stays as Part 3's `Hurtbox.prepare` sets it.
  - Hit queries only ever target the `Hurtbox` group.
- **R15 only.** On `CharacterAdded`, AppearanceService checks `Humanoid.RigType == Enum.HumanoidRigType.R15` [Docs]. Otherwise it logs `[ALG][Appearance] ERROR rig_not_r15 user=<id>` (Avatar Settings misconfigured; H-05).
- **Fallback route B** (only if spike APP-1 fails, §5.14): build the description with `Players:GetHumanoidDescriptionFromUserIdAsync(userId)` (pcall; cached per session) and spawn with `Player:LoadCharacterWithHumanoidDescriptionAsync(desc)` [Docs]. This changes Part 3 step 6. Never use the deprecated `Humanoid:ApplyDescription`, `Players:GetHumanoidDescriptionFromUserId` or `Player:LoadCharacterWithHumanoidDescription` [Docs deprecation tags]; for in-place refresh use `Humanoid:ApplyDescriptionAsync`.

#### 5.7.4 `Appearance.Set` handler (§2.7.4 row 3)

- Checks: `Features.Appearance`; P2 shape validation; `styleOverride.styleId` must be owned, in the free rotation, or trialable on Training (`StyleService.canEquip` rules 1–4 minus implemented, so unimplemented owned styles may store an override); `oosona` must pass `Schema.validateOosona(spec, cfg)`.
- One update of `{"settings"}`; `"inherit"` removes the key.
- Then rewrite `CharacterLook` (for the current `EquippedStyle`), `ShowStyleGear` and `OosonaCode`.
- Log `[ALG][Appearance] set user=<id> look=<bool> own=<Oosona|Avatar> gear=<bool>`.
- Rate 5/1 (§2.7.4).

#### 5.7.5 Viewer override (client; Part 6 renders) — truth table

The viewer's own character always renders as the owner chose. The override applies to others only.

| Viewer `othersAppearance` | Target `CharacterLook` | Target `OwnLook` | Render |
|---|---|---|---|
| `AsChosen` | true | any | Character (palette rig + Style Gear) |
| `AsChosen` | false | `Avatar` | Avatar (+ Style Gear if `ShowStyleGear`) |
| `AsChosen` | false | `Oosona` | Ooo-sona from `OosonaCode` (+ Style Gear if `ShowStyleGear`) |
| `AlwaysCharacter` | any | any | Character |
| `AlwaysOwn` | any | `Avatar` | Avatar (+ gear per `ShowStyleGear`) |
| `AlwaysOwn` | any | `Oosona` | Ooo-sona (+ gear per `ShowStyleGear`) |
| `Simplified` | any | any | Palette-coloured graybox body, gear and accessories hidden |

The **Style Tell is always on** in every row and cannot be disabled (§0.4): the nameplate chip in the style colour and icon, and the style's VFX palette (+ Flair tint).

#### 5.7.6 AppearanceService API

```lua
--!strict
export type AppearanceServiceApi = {
	onPlayerReady: (player: Player) -> (),         -- writes the 4 attributes; sets CanLoadCharacterAppearance before the first spawn
	refresh: (player: Player) -> (),               -- recompute CharacterLook/ShowStyleGear/OosonaCode now
	effectiveLook: (settings: Types.Settings, styleId: string) -> boolean, -- pure helper
}
```

#### 5.7.7 Verify (§5.7)

| Kind | Name | Expected |
|---|---|---|
| Lune | `appearance.effective_look` | global true, override `rocker=false` → rocker false, hero true; inherit removes |
| Scenario | `appearance.attrs_replicated` | after join, client reads `CharacterLook=true`, `OwnLook="Oosona"`, `ShowStyleGear=true`, `OosonaCode` decodable |
| Scenario | `appearance.per_style_override` | `Appearance.Set{styleOverride={styleId="stretch",value="off"}}`; equip stretch; respawn → `CharacterLook=false`; equip hero → true |
| Scenario | `appearance.ownlook_on_respawn` | set `ownLook="Avatar"` → `OwnLook` attribute still `"Oosona"` until respawn, then `"Avatar"`; `CanLoadCharacterAppearance = true` |
| Scenario | `appearance.hurtbox_invariant` | Hurtbox part size and CFrame offset are identical across Character / Oosona / Avatar lives (compare via `inspect_instance`) |
| Scenario | `appearance.r15_only` | every spawned character has `RigType = R15`; no `rig_not_r15` lines |
| Scenario | `appearance.trial_override_allowed` | on Training, `styleOverride` for an unowned implemented style → accepted |
| MCP | `screen_capture` per mode (Part 6 renders) | visual check by a human at H-10 |

---

### 5.8 Telemetry wrapper (`Services/Telemetry.luau`; GDD §18.10, §20.4)

#### 5.8.1 Environments

- **Live** (`not RunService:IsStudio()` and `Telemetry.Enabled`): calls `AnalyticsService` methods. They are server-only, and events are recorded only in published games [Docs `production/analytics/event-types.md`].
- **Studio**: never calls AnalyticsService. If `Telemetry.EchoInStudio`, every call prints one echo line:

  `[ALG][TEL] <api> <name> user=<UserId> <sorted k=v of all fields>`

  e.g. `[ALG][TEL] economy Treasure amount=25 balance=125 flow=source reason=combat_ko sku= user=-1`. The line is printed via `Log.info("TEL", ...)` (GDD §18.10 wording).
- **All environments**: each call is appended to the in-memory ring (`Telemetry.RingSize` 200) as a `TelRecord` for `Telemetry.recent(n)`, which test assertions use.
- Telemetry calls **never yield**. In live mode, records go to a queue drained by a `task.defer`ed sender loop, and each AnalyticsService call is wrapped in `pcall` (failures are counted and logged once per minute).

#### 5.8.2 Budget and rate limits

- Global limit: `120 + 20 × CCU` requests/min [Docs].
- Per 60 s window: `budget = floor(BudgetFraction × (BaseRequestsPerMin + PerCcuRequestsPerMin × #Players:GetPlayers()))` (0.8 × (120 + 20·CCU)).
- Priority classes and ceilings (`Telemetry.ClassCeilings = {1.0, 0.9, 0.75, 0.5}`): a class is admitted while `used < ceiling × budget`.

| Class | Events |
|---|---|
| P0 | economy, onboarding, store funnel |
| P1 | `wish_made`, wish funnel, progression |
| P2 | `match_summary`, `mod_summary` |
| P3 | `perf_sample`, `world_event` |

- Dropped records still go to the ring and the Studio echo. Log `[ALG][Tel] WARN budget_drop n=<n> window=<id>` once per window when n > 0.
- **Cardinality** [Docs]: at most 3 custom fields; values are **strings** (the wrapper `tostring`s every value); ≤ 8,000 unique combinations; ≤ 100 custom event names (this part uses 4); ≤ 10 economy currencies (4); ≤ 20 transaction types (7 below); ≤ 100 item SKUs (≈ 60: 18 styles, 36 variants, 4 products, 2 wish SKUs).

#### 5.8.3 API (Part 2 §2.3.5 signatures) and field mapping

- Callers pass `fields` with keys `CustomField01..03` (sent) and any other keys (echo and ring only).
- The wrapper maps those keys to `Enum.AnalyticsCustomFieldKeys.CustomField0N.Name` [Docs].

**Economy transaction-type mapping** (`Enum.AnalyticsEconomyTransactionType` names [Docs]; custom names are allowed):

| Reason(s) | transactionType |
|---|---|
| `purchase` | `IAP` |
| `shop_buy` | `Shop` |
| `tutorial` | `Onboarding` |
| `daily_login` | `TimedReward` |
| `combat_*`, `streak_bounty`, `level_up`, `quest`, `world_event`, `wish`, `wish_overflow`, `duplicate`, `mythic_dupe`, `shard_convert`, `cosmic_craft` | `Gameplay` |
| `compensation`, `dev` | `Admin` (custom; `dev` never reaches live) |

**Event map (GDD §20.4):**

| GDD event | Emitter and trigger | AnalyticsService call | Value | CF01 | CF02 | CF03 | Echo-only extras |
|---|---|---|---|---|---|---|---|
| `currency_flow` | DataService, after each committed `tx.credit/debit` (§2.8.5) | `LogEconomyEvent(player, Source\|Sink, currency, amount, endingBalance, txType, sku, cf)` | amount | reason | sku or "" | — | — |
| `wish_made` | WishService `afterCommit` | `LogCustomEvent(player, "wish_made", count, cf)` | 1 or 10 | source of roll 1 | best rarity | `Wish.BannerId` | per-roll style_id, flair, pity before, odds hash, isNew |
| `wish_made` (economy part) | DataService (Wishes `debit`, sku `wish_x1`/`wish_x10`) | `LogEconomyEvent(Sink, "Wishes", …)` | count | `wish` | sku | — | — |
| wish funnel | WishService | `LogFunnelStepEvent(player, "wish", sid, step, name, cf)`: 1 `odds_viewed`, 2 `wished` | — | count | source | — | — |
| `store_view` / `purchase` | MonetizationService | `LogFunnelStepEvent(player, "store", sid, step, name, cf)`: 1 `store_view`, 2 `prompt`, 3 `granted` | — | product key | entry point | price shown | region: not sent (no field budget; dashboards break down by location) |
| `ftue_step` | ProgressionService (§5.8.4) | `LogOnboardingFunnelStepEvent(player, step, stepName, cf)` | — | `skipped:<bool>` | elapsed bucket (10 s) | — | exact elapsed_s |
| `match_summary` | ProgressionService, `lifeEnded` (skipped if `lifeSec < MatchSummaryMinLifeSec`) | `LogCustomEvent(player, "match_summary", damageDealt, cf)` | damage_dealt | style_id | map | flair | every `LifeSummary` field |
| progression | ProgressionService on level-up / mastery level-up | `LogProgressionEvent(player, "account"\|"mastery", Enum.AnalyticsProgressionType.Complete, level, "L<n>", cf)` | — | style_id (mastery) | — | — | — |
| `perf_sample` | Telemetry, `Telemetry.Perf` remote (§2.7.4 row 18; sampled clients only, see **Perf sampling** below) | `LogCustomEvent(player, "perf_sample", fpsP50, cf)` | fps_p50 | device_tier | `p5:<floor(fpsP5/5)*5>` | `vfx:<level>` | memory_mb |
| `mod_summary` | ModerationService at session end (§2.11.3) | `LogCustomEvent(player, "mod_summary", total, cf)` | total | top kind | distinct kinds | — | per-kind counts |
| `duel_result`, `realm_clash`, `world_event`, `pass_progress` | — | future hooks (no duels, Realm Clash or Pass in M1-G) | | | | | |

**Perf sampling (`perf_sample`, GDD §20.2 low-tier fps KPI; decision D-17).**
- **Server (Telemetry, at join):** in its `Players.PlayerAdded` handler Telemetry sets the Player attribute `PerfSampled = Random.new():NextNumber() < Runtime.get().Telemetry.PerfSampleRate` (boolean; §2.5.2 row). The decision is per session and is never re-rolled. Studio seam `Telemetry._setPerfSampled(player: Player, on: boolean)` overrides it (no-op outside Studio). The `Telemetry.Perf` handler drops payloads from senders whose `PerfSampled` is not true (the §2.7.4 semantic check).
- **Client (Part 6 §6.6.3, VfxController):** while `LocalPlayer:GetAttribute("PerfSampled") == true`, it records `RenderStepped` dt over a rolling `Telemetry.PerfWindowSec` (120 s) window, computes fps P50 and P5 (nearest rank over `1/dt`), `Stats:GetTotalMemoryUsageMb()`, the current device tier and VFX degrade level, and fires `Telemetry.Perf` (`TelPerfSample`) once per `Telemetry.PerfSendEverySec` (300 s), the first send after the first full window. The remote's rate (1 per 50 s) admits that cadence.
- **Scenario `tel.perf_sample_sent`:** `Dev.Command setConfig Telemetry {"PerfWindowSec":5,"PerfSendEverySec":10}`, seam `_setPerfSampled(P, true)` → within 15 s `Telemetry.recent(20)` holds a record with `name = "perf_sample"` for P and its `fpsP50 > 0`; then `_setPerfSampled(P, false)` and a forged `Telemetry.Perf` fire from the client → no new record (dropped by the semantic check).

#### 5.8.4 FTUE steps

The 9 steps follow GDD §15.5, in order:

| Step | Name |
|---|---|
| 1 | `cold_open` |
| 2 | `spawn_path` |
| 3 | `m1_log` |
| 4 | `block` |
| 5 | `moves_dash` |
| 6 | `breakaway` |
| 7 | `awakening` |
| 8 | `first_wish` |
| 9 | `first_live_fight` |

- **Server-detected:** step 8 (first `Wish.Make`) and step 9 (first spawn on `candy_kingdom` after ready).
- **Client-reported:** steps 2–7 are reported by Part 6's FTUE-lite (§6.9.10) through a new remote `Ftue.Step` (C2S event, rate 5 / 1, shape `struct{step = int(1, 9), skipped = bool}`, owner ProgressionService; §5.12): first `ui.tip.punch` shown → 2; first M1 contact → 3; first Block entered → 4; first Dash → 5; first Breakaway → 6; first Awakening → 7 (D-40). Step 1 (`cold_open`) is **not emitted in M1-G** (no cold open), so the onboarding funnel starts at step 2; Part 7 §7.6.8 notes it.
- The handler accepts only `step > flags.ftueStep`, so steps are monotonic and skips are allowed. It sets `flags.ftueStep = step` and `ftueDone = (step == 9)`, then emits the onboarding event.
- Elapsed time = `now − createdAt`.

#### 5.8.5 Verify (§5.8)

| Kind | Name | Expected |
|---|---|---|
| Scenario | `tel.echo_economy` | KO → flush → a ring record with `api="economy"`, `name="Treasure"`, reason `combat_ko`; an Output line starting `[ALG][TEL] economy Treasure` |
| Scenario | `tel.echo_wish` | `Wish.Make{1}` → `[ALG][TEL] custom wish_made ... CustomField01=tutorial` (fresh profile) |
| Scenario | `tel.no_analytics_in_studio` | the seam counter `Telemetry._liveCalls == 0` after the full scenario suite |
| Scenario | `tel.budget_drop` | seam: budget forced to 5; 20 P3 records → exactly 3 admitted (admitted while `used < 0.5 × 5`), a `budget_drop` warning, all 20 in the ring |
| Scenario | `tel.fields_stringified` | a numeric field is sent as a string (inspect via `Telemetry._lastSent`) |
| Scenario | `tel.ftue_monotonic` | `Ftue.Step{3}` then `{2}` → second ignored; `flags.ftueStep = 3` |
| Scenario | `tel.perf_sample_sent` | see **Perf sampling** (§5.8.3): sampled client sends a `perf_sample` within 15 s with the shortened window; an unsampled sender is dropped |

---

### 5.9 Moderation basics (`ModerationService` additions; core in §2.11.3)

#### 5.9.1 Reports (`Report.Submit`, §2.7.4 row 17)

- Semantic checks (P2): `Features.Reports`; the target is in this server; the target is not the reporter. A failure is silently dropped with a log.
- On accept:
  - append `{t, reporter, target, reason}` to a server-memory ring (100 entries; `ModerationService.reports(n)` for tests);
  - `signal(target, "report_received", {reason = reason})`;
  - log `[ALG][Mod] report reporter=<id> target=<id> reason=<r>`;
  - Notify the reporter with `ui.notify.report_thanks`.
- The **30 s replay snapshot** (GDD §16.2) is a future hook: ModerationService (#3) cannot require CombatService (#9), and the snapshot needs Part 3's history ring.
- No automatic action (GDD §18.10).

#### 5.9.2 Ban wrapper (dry-run in M1-G)

```lua
--!strict
export type BanRequest = {
	userIds: { number },          -- 1..50 (Players:BanAsync limit [Docs])
	durationSec: number,          -- > 0, or -1 = permanent
	displayReason: string,        -- <= 400 chars, text-filtered by Roblox
	privateReason: string,        -- <= 1000 chars, never sent to the client
	excludeAltAccounts: boolean?, -- default false = alts banned (GDD §18.10)
	applyDeviceBlock: boolean?,   -- default false (24 h device block when true)
}
local BAN_MODE: "dry_run" | "live" = "dry_run"   -- code constant; switching to "live" requires a human-approved PR (§5.14)

function ModerationService.banAsync(req: BanRequest): (boolean, string?)
	if #req.userIds < 1 or #req.userIds > 50 then return false, "invalid" end
	if BAN_MODE ~= "live" or RunService:IsStudio() then
		Log.info("Mod", "ban dry_run", { users = table.concat(req.userIds, ","), duration = req.durationSec, reason = req.privateReason })
		return false, "dry_run"
	end
	local ok, err = pcall(Players.BanAsync, Players, {
		UserIds = req.userIds, Duration = req.durationSec, DisplayReason = req.displayReason,
		PrivateReason = req.privateReason, ExcludeAltAccounts = req.excludeAltAccounts == true,
		ApplyDeviceBlock = req.applyDeviceBlock == true, ApplyToUniverse = true,
	})
	if not ok then Log.error("Mod", "ban_failed", { err = tostring(err) }) end
	return ok, if ok then nil else tostring(err)
end
```

- `Players:BanAsync` is verified [Docs]. It requires `Players.BanningEnabled` (NotScriptable, toggled in Studio by a human) and does not apply to Studio sessions.
- Nothing in M1-G calls `banAsync`; it exists for future human-reviewed tooling. This respects §2.11.3 ("no automatic kicks or bans"; Part 2 records that `Players:BanAsync` is never called in M1-G and ModerationService keeps this dry-run wrapper, OPS-1). Switching `BAN_MODE` is out of scope for M1-G.

#### 5.9.3 Rookie protection flags (GDD §15.5)

| Flag | Owner | Rule |
|---|---|---|
| `Rookie` attribute | ProgressionService | true while `flags.rookieSecLeft > 0` (7,200 s of play) |
| Rookie nameplate | Part 6 | renders from `Rookie` |
| Anti-farm | ProgressionService | §5.5.4 Rookie rule, per earner: a veteran gets no Treasure for the 3rd and later Rookie KO (any Rookie) within 10 min |
| Protected spawn | Part 3 | 8 s or until attacking, for everyone (§3.13.5) |
| Low-skill server bucket | — | future hook (matchmaking is out of scope) |

**Verify:** scenarios `mod.report_logged` (a report produces the log line, `summary(target).report_received = 1`, and a toast for the reporter), `mod.report_self_rejected`, `mod.ban_dry_run` (`banAsync` returns `false, "dry_run"` and logs; `Players:BanAsync` is never called — spy seam `ModerationService._banCalls == 0`).

---

### 5.10 Config: Part 5 sections of `Defaults.luau`

These tables replace the `{}` and placeholder values that §2.6.2 left for Part 5, and they are authoritative for these sections. Differences from §2.6.2's draft values are listed in §5.12.

```lua
--!strict
-- Part 5 sections of src/Shared/Config/Defaults.luau. Spec §5.10.
MockPurchases = true,                                         -- honoured only in Studio (§2.11.4)

Wish = {
	TierOdds = { common = 500000, uncommon = 280000, rare = 140000, epic = 60000,
		legendary = 16000, mythic = 3900, cosmic = 100 },     -- ppm; GDD §8.3 (sum 1,000,000)
	Pity = { EpicPlusHard = 30, LegendaryPlusHard = 120, MythicPlusHard = 400, -- GDD §8.4
		SoftStart = 60, SoftStepPpm = 5000,                   -- +0.5%/Wish from Wish 60 (GDD §8.4)
		NewStylePriority = true },                            -- GDD §8.4
	Flair = { shiny = 20000, golden = 4000, crystal = 1000, rainbow = 200,
		nightosphere = 40, glitched = 10 },                   -- ppm; standard = remainder 974,750 (GDD §8.7)
	LuckyJarFlairMultiplier = 1.5,                            -- GDD §17.3 (cap ×1.5, GDD §17.4)
	StorageCap = 1000, OverflowTreasure = 10,                 -- GDD §17.2
	PoolMode = "Full",                                        -- §0.10
	DuplicateShards = { common = 3, uncommon = 5, rare = 10, epic = 25,
		legendary = 60, mythic = 150, cosmic = 300 },         -- (tuning default)
	VariantChance = 0.05,                                     -- GDD §9.3
	VariantsPerStyle = 2,                                     -- placeholder catalog v1..v2 (tuning default; GDD §9.3: 2–4)
	CosmicSparksNeeded = 10, SparksPerMythicDupe = 1,         -- GDD §8.4
	FirstWishGuarantee = { Enabled = true, Rarity = "rare",
		Styles = { "monarch", "flame" }, Weights = { 1, 1 } }, -- GDD §15.5
	FakeOut = { epic = 0.03 },                                -- GDD §8.6
	BannerId = "standard",                                    -- GDD §20.4 banner_id (no banners in M1-G)
	Announce = { CrossServerEnabled = false,                  -- M1-G default (§5.2.13); ALG_Wish may enable (live servers only)
		CrossServerTopic = "ALG_Announce_v1" },               -- GDD §8.6 / §2.7.4 note
},

Shop = {
	StyleSlots = 3, CosmeticSlots = 4, CosmeticSlotsEnabled = true, -- GDD §8.5 (cosmetics = Variant offers, placeholder)
	RefreshSec = 43200,                                       -- 12 h (GDD §8.5)
	Prices = { common = 1500, uncommon = 3000, rare = 6000, epic = 12000, legendary = 30000, mythic = 75000 }, -- GDD §8.5
	ExcludeRarities = { "starter", "cosmic" },                -- GDD §8.5
	UnownedWeight = 4, OwnedWeight = 1,                       -- (tuning default)
	IncludeUnimplemented = true,                              -- §0.10
	VariantPrices = { starter = 500, common = 500, uncommon = 750, rare = 1000, epic = 1500,
		legendary = 2500, mythic = 4000, cosmic = 8000 },     -- (tuning default; GDD §17.5 cosmetics 500–8k)
},

Economy = {
	CombatEarn = { KoTreasure = 25, AssistTreasure = 10, TreasurePer100Damage = 5, -- (tuning default ≈ 350/day, GDD §17.5)
		ActiveWishEverySec = 900 },                           -- GDD §17.5
	RepeatVictim = { WindowSec = 600, Multipliers = { 1, 0.5, 0.25, 0 } }, -- GDD §16.2 (tuning default curve)
	RookieKill = { WindowSec = 600, MaxTreasureKills = 2 },   -- GDD §15.5
	DailyLogin = { Wishes = 2, Treasure = 100 },              -- GDD §17.5
	WorldEvent = { Treasure = 80, WishEveryParticipations = 6 }, -- GDD §17.5 (tuning default split)
	DailyQuest = { Treasure = 150, Wishes = 1, AllDoneBonusWishes = 1 },  -- GDD §17.5 (3 → 450 T + 4 W)
	WeeklyQuest = { Treasure = 250, Wishes = 2, AllDoneBonusWishes = 4 }, -- GDD §17.5 (future hook)
	TutorialWishes = 3,                                       -- GDD §15.5
	VipTreasureMultiplier = 1.10,                             -- GDD §17.3
	VariantCraftShards = 400,                                 -- GDD §17.2 (future hook)
	QuestRerollTreasure = 100,                                -- GDD §17.5 (future hook)
	VelocityAlertTreasurePerHour = 20000,                     -- (tuning default, §2.11.3)
	TrainingEarns = false,                                    -- (tuning default; GDD §4.5 practice mode)
	BotRewardMultiplier = 0.5,                                -- (tuning default)
	FlushSec = 30,                                            -- §2.8.5 "at least every 30 s"
	Products = {                                              -- GDD §17.3; RefPriceRobux = display in mock only
		wishes_10 = { Wishes = 10, RefPriceRobux = 99, Pri = true },
		wishes_55 = { Wishes = 55, RefPriceRobux = 449, Pri = true },
		wishes_120 = { Wishes = 120, RefPriceRobux = 899, Pri = true },
		wishes_300 = { Wishes = 300, RefPriceRobux = 1999, Pri = true },
	},
	Passes = {                                                -- GDD §17.3
		vip = { RefPriceRobux = 299, Pri = false },
		quick_wish = { RefPriceRobux = 99, Pri = false },
		lucky_jar = { RefPriceRobux = 349, Pri = true },
	},
	SpendGuard = { ConfirmFromPurchase = 4, WindowSec = 86400 }, -- GDD §8.8
	NoOfferAfterLossSec = 30,                                 -- GDD §8.8 (tuning default window; UI rule)
	ReceiptLoadWaitSec = 30, ReceiptSaveWaitSec = 30,         -- (tuning default)
	ProductInfoCacheSec = 600,                                -- (tuning default)
	Policy = { Attempts = 3, BackoffSec = { 1, 2 }, BackgroundRetrySec = { 30, 60, 120 },
		StudioFailOpen = true },                              -- (tuning default; fail-open only in Studio)
},

Progression = {
	Account = {
		MaxLevel = 100, XpBase = 300, XpPerLevel = 45,        -- GDD §9.1
		XpPerActiveMinute = 10, XpKo = 25, XpAssist = 10, XpWorldEvent = 40, XpDailyQuest = 150, XpWeeklyQuest = 600, -- GDD §9.1
		ActiveWindowSec = 60, ActiveMinDamage = 10,           -- (tuning default; AFK → 0 XP, GDD §9.1)
		RewardPerLevel = { Wishes = 3, Treasure = 250 },      -- GDD §9.1
		CosmeticEveryLevels = 10, PrestigeStarsMax = 5,       -- GDD §9.1
	},
	Mastery = {
		MaxLevel = 30, XpBase = 400, XpPerLevel = 80,         -- GDD §9.2 (curve: tuning default, §2.14 item 9)
		XpPerDamage = 1, XpKo = 50, XpStyleMoment = 100,      -- GDD §9.2
		ShardXp = 60, ShardCapLevel = 20,                     -- GDD §9.2
		Rewards = {
			{ level = 1, reward = "lore_card" }, { level = 5, reward = "title" },
			{ level = 10, reward = "finale_1_and_palette" }, { level = 15, reward = "m1_trail" },
			{ level = 20, reward = "emote" }, { level = 25, reward = "awaken_tint_and_variant_choice" },
			{ level = 30, reward = "master" },
		},                                                    -- GDD §9.2
	},
	Rookie = { DurationSec = 7200 },                          -- GDD §15.5
	FreeRotation = { Enabled = true, Count = 3, PeriodSec = 604800, Salt = 1386333159 }, -- GDD §8.1 (Salt = 0x52A1C3E7)
	TrainingTrialAllStyles = true,                            -- GDD §4.5
	TotemRangeStuds = 12,                                     -- (tuning default)
	DailyResetCheckSec = 60,                                  -- (tuning default)
	Quests = {
		Enabled = true, DailyCount = 3, WeeklyCount = 7, DailyRerolls = 1, -- GDD §9.5; on in M1-G (§5.5.9, DES-7)
		DailyPool = {
			{ id = "deal_damage", stat = "damage", goal = 600 },         -- GDD §9.5 example
			{ id = "perfect_blocks", stat = "perfectBlocks", goal = 10 }, -- GDD §9.5 example
			{ id = "kos", stat = "kos", goal = 5 },                      -- (tuning default)
			{ id = "active_minutes", stat = "activeMinutes", goal = 15 }, -- (tuning default)
			{ id = "awaken", stat = "awakenings", goal = 1 },            -- (tuning default)
		},
	},
},

Appearance = {
	OosonaSpecies = {                                         -- GDD §12.3 species templates (placeholder skins, tuning default)
		{ id = "human", skin = "#F2C9A0" }, { id = "candy", skin = "#F7A8C8" }, { id = "dog", skin = "#E8B04A" },
		{ id = "cat", skin = "#D9D9D9" }, { id = "vampire", skin = "#9AA3B5" }, { id = "flame", skin = "#FF8A3D" },
		{ id = "lumpy", skin = "#B48CD9" }, { id = "wizard", skin = "#7FB7E6" }, { id = "ghost", skin = "#EAF4FF" },
		{ id = "penguin", skin = "#2E3440" },
	},
	OosonaFaces = 6, OosonaHats = 6, OosonaOutfits = 4,       -- (tuning default) graybox catalog sizes; hat 0..6, outfit 0..4
	OutfitColors = { "#3B6EA8", "#2E8B57", "#8B3A3A", "#444444" }, -- legs colour per outfit 1..4 (tuning default)
	NoOutfitDarken = 0.65,                                    -- outfit 0: legs = colour × 0.65 (tuning default)
	OosonaColors = { "#F2C14E", "#F48FB1", "#7FD8BE", "#B39DDB", "#FFF3D6", "#4FC3F7",
		"#FF8A65", "#A1887F", "#90A4AE", "#E0E0E0", "#6D4C41", "#212121" }, -- (tuning default) Settings swatches (Part 6 §6.9.9); #F2C14E = Part 2 default
	SimplifiedLimbShade = 0.75,                               -- (tuning default) Simplified look limbs (Part 6 §6.8.2)
	FaceExpressions = 24,                                     -- GDD §12.3
},

Telemetry = {
	Enabled = true, PerfSampleRate = 0.01,                    -- GDD §20.4 (1% of clients)
	BudgetFraction = 0.8, RingSize = 200, EchoInStudio = true, -- §2.6.2 keys (tuning default)
	BaseRequestsPerMin = 120, PerCcuRequestsPerMin = 20,      -- Docs analytics limits; GDD §20.4
	ClassCeilings = { 1.0, 0.9, 0.75, 0.5 },                  -- P0..P3 (tuning default)
	MatchSummaryMinLifeSec = 5,                               -- (tuning default)
	PerfWindowSec = 120, PerfSendEverySec = 300,              -- (tuning default) client perf_sample window / send period (§5.8.3)
},
```

**Schema rules added by Part 5** (registered in `Schema.rules` / `Schema.checks`; `schema.defaults_valid` covers them):

| Path | Rule |
|---|---|
| `Wish.TierOdds.*` | int 0–1,000,000; **check** sum = 1,000,000 and every key ∈ the 7 rollable rarities |
| `Wish.Flair.*` | int 0–1,000,000; **check** `Odds.flairTable(ctx, true)[1] > 0` (with the multiplier applied, standard stays positive) |
| `Wish.LuckyJarFlairMultiplier` | num 1–1.5 (GDD §17.4 cap) |
| `Wish.Pity.*Hard` | int 1–10,000; **check** Epic ≤ Legendary ≤ Mythic |
| `Wish.Pity.SoftStart` | int 1–`LegendaryPlusHard`; `SoftStepPpm` int 0–100,000 |
| `Wish.StorageCap` | int 1–100,000; `OverflowTreasure` int 0–1,000 |
| `Wish.PoolMode` | enum `{"Full"}` |
| `Wish.DuplicateShards.*` | int 0–100,000 |
| `Wish.VariantChance`, `Wish.FakeOut.*` | num 0–1 |
| `Wish.VariantsPerStyle` | int 0–9 (`Types.VARIANT_IDS`) |
| `Wish.CosmicSparksNeeded` | int 1–1,000 |
| `Wish.FirstWishGuarantee` | `Styles[*]` styleId; `#Weights == #Styles`; weights int ≥ 1; `Rarity` a rollable rarity |
| `Shop.VariantPrices.*` | int 1–1,000,000 |
| `Economy.Products.*.Wishes` | int 1–10,000; `Economy.RepeatVictim.Multipliers[*]` num 0–1 |
| `Progression.TotemRangeStuds` | num 1–100; `Progression.Account.ActiveMinDamage` int 0–1,000 |
| `Telemetry.BudgetFraction` | num 0.1–1; `Telemetry.PerfSampleRate` num 0–1; `ClassCeilings[*]` num 0–1 |
| `Appearance.OosonaSpecies[*].skin`, `OutfitColors[*]` | hexColor; `#OutfitColors == OosonaOutfits`; **check** `#OosonaSpecies ≥ 1`; `OosonaSpecies[*].id` non-empty and unique |
| `Appearance.OosonaColors[*]` | hexColor; **check** `#OosonaColors ≥ 1` |
| `Appearance.OosonaFaces`, `OosonaHats`, `OosonaOutfits` | int ≥ 1 (Settings validator: face 1..`OosonaFaces`, hat 0..`OosonaHats`, outfit 0..`OosonaOutfits`, Part 2 §2.8.6) |
| `Appearance.SimplifiedLimbShade`, `Appearance.NoOutfitDarken` | num 0–1 |
| `Appearance.FaceExpressions` | int ≥ 1 |
| `Wish.Announce.CrossServerEnabled` | bool |
| `Telemetry.PerfWindowSec` | int 5–600; `Telemetry.PerfSendEverySec` int ≥ `PerfWindowSec`, ≤ 3,600 |

`Appearance` is the single canonical table for the whole spec (decision D-13): Part 6 §6.11 no longer defines it and points here. Species are records read via `.id` (Part 6 builders and UI, Part 8 `ui.oosona.species.<id>`). Oosona rendering follows §5.7.1 (head/arms species `skin`, torso `OosonaSpec.color`, legs `OutfitColors[outfit]` or `color × NoOutfitDarken`).

Live-ops note: changes to `ALG_Wish` take effect at the next Wish (§2.6.4). The new `oddsHash` appears in the ledger and the odds view. GDD §18.6's two-person approval is an operational rule outside the game code.

---

### 5.11 Test catalog (Part 7 runs; names are binding)

**Lune specs** (`tests/wish/`):

| File | Tests |
|---|---|
| `stream.spec.luau` | `wish.mul32_vectors`, `wish.fmix32_vectors`, `wish.stream_vectors` |
| `odds.spec.luau` | `wish.tier_tables`, `wish.tier_tables_sum_1e6`, `wish.flair_tables`, `wish.roll_vectors`, `wish.batch10_vector`, `wish.ftue_styles_match_rarity`, `wish.cosmic_fixed`, `wish.lucky_flair_only`, `wish.view_matches_roll_tables`, `wish.view_sums_100`, `wish.view_priority_rows`, `wish.hash_stable`, `wish.hash_changes`, `wish.pool_matches_styles`, `wish.tier_frequencies_fresh`, `wish.flair_frequencies_fresh` |
| `pity.spec.luau` | `pity.floor_precedence`, `pity.soft_bonus_values`, `pity.advance_resets`, `pity.bounds_invariant_random_walk`, `pity.transition_check_detects` |
| `ledger.spec.luau` (§2.8.4 contract) | `ledger.codec_roundtrip`, `ledger.vector_v1`, `ledger.max_len_80`, `ledger.decode_rejects_garbage`, `ledger.ring_semantics` |
| `montecarlo.spec.luau` | `wish.mc_spine_tolerance`, `wish.mc_hard_pity_bounds`, `wish.mc_reference_exact` |
| `rotation.spec.luau` | `shop.rotation_vectors`, `shop.rotation_id_roundtrip`, `shop.rotation_owned_weighting`, `style.free_rotation_vectors`, `quests.pick_deterministic`, `time.week_index_monday` |
| `progression.spec.luau` | `progression.account_curve`, `progression.level_from_xp_vectors`, `progression.mastery_curve`, `progression.shard_clamp`, `progression.repeat_victim_multipliers`, `progression.rookie_rule_per_earner`, `appearance.effective_look` |

`appearance.effective_look` tests the pure helper `AppearanceService.effectiveLook`. Because services are not load-pure, the helper body lives in `Pity.luau` as `Pity.effectiveLook(settings, styleId)` and AppearanceService re-exports it.

**Studio scenarios** (in-Studio harness, Part 7):
- `style.*` (§5.1.8)
- `wish.*` (§5.2.17)
- `policy.*` (§5.3)
- `shop.*` (§5.4.3)
- `progress.*` (§5.5.12)
- `store.*` (§5.6.7)
- `appearance.*` (§5.7.7)
- `tel.*` (§5.8.5)
- `mod.*` (§5.9)

Every scenario prints `[ALG][TEST] <name> PASS|FAIL <detail>` (§0.4).

**MCP verification recipe** (Part 1 snippets):
1. `start_stop_play` (start).
2. `execute_luau` with `datamodel_type = "Server"`: `return require(game.ServerScriptService.Services.TestHarness).run("wish.batch10_server")` (allowed only after spike T7-S4; otherwise use Part 7 §7.3.6's request/result protocol with selector `wish.batch10_server`).
3. `get_console_output`, filtered for `[ALG][TEST]` and `[ALG][Wish]` lines.

---

### 5.12 New contracts introduced

1. **Pure module APIs** `Shared/Wish/Odds.luau` and `Shared/Wish/Pity.luau` (§5.2.2), including:
   - the per-roll stream: `fmix32` seeding, exactly 5 draws in the order d1 tier, d2 style, d3 Flair, d4 Variant, d5 fake-out;
   - the largest-remainder quantization;
   - Cosmic fixed under pity;
   - rotation, curve and time helpers.

   Meta pure helpers (curves, rotations, `effectiveLook`) live in these files (§2.10 item 5).
2. **Config:** complete Part 5 sections (§5.10). New keys beyond §0.9 / §2.6.2:
   - `Wish.VariantsPerStyle`, `Wish.SparksPerMythicDupe`, `Wish.FakeOut`, `Wish.BannerId`, `Wish.Announce` (incl. `CrossServerEnabled = false`), `Wish.Pity.NewStylePriority`;
   - `Shop.VariantPrices`;
   - `Economy.{TrainingEarns, BotRewardMultiplier, FlushSec, Products, Passes, SpendGuard, NoOfferAfterLossSec, ReceiptLoadWaitSec, ReceiptSaveWaitSec, ProductInfoCacheSec, Policy}`;
   - `Progression.{TotemRangeStuds, DailyResetCheckSec}`, `Progression.Account.ActiveMinDamage`, `Progression.FreeRotation.Salt`, `Progression.Quests.DailyPool`;
   - `Appearance.{OutfitColors, NoOutfitDarken}`, plus Part 6's `Appearance.{OosonaColors, SimplifiedLimbShade, FaceExpressions}` merged here (D-13: §5.10 is the only `Appearance` table; Part 6 §6.11 points to it);
   - `Telemetry.{BaseRequestsPerMin, PerCcuRequestsPerMin, ClassCeilings, MatchSummaryMinLifeSec, PerfWindowSec, PerfSendEverySec}`.

   **Odds are integers in ppm.**
3. **Value decisions vs §2.6.2's draft** (§5.10 wins wherever the two differ; Part 2's draft already carries the first three values):
   - `Shop.CosmeticSlotsEnabled = true` (Variant offers as the placeholder cosmetic catalog);
   - `Progression.FreeRotation.Enabled = true` (Count stays 3, GDD §8.1; DES-6 records the M1-G ratio);
   - `Progression.Quests.Enabled = true` (M1-G; §5.5.9, DES-7);
   - `Appearance.OosonaHats = 6` (Part 2 keeps only an `Appearance` placeholder filled from §5.10; Part 6 builds `Oosona_Hat_1..6`; the Settings validator allows hat 0..6);
   - the active-minute rule is now "dealt ≥ `ActiveMinDamage` (10)" (§5.5.3) instead of the draft's "dealt or took ≥ 1".
4. **Types additions** (Part 2 adds them to `Types.luau`):
   - `WishStats.paidEscrow: number`;
   - `PurchaseState.weekIndex: number`, `PurchaseState.weekRobux: number`;
   - `WishResult.firstReveal: boolean`;
   - `OddsView.nextGuarantee: PityHit`, `OddsView.baseTiers: {OddsTier}`, `OddsView.baseFlairs: {{flair, pct}}`, `OddsView.softStartWish: number`, `OddsView.hardPity: {epic: number, legendary: number, mythic: number}`;
   - `StoreView.spentThisWeekRobux: number`.

   The template defaults are 0 / {} accordingly. `Schema.normalizePlayerData` clamps `paidUnspent` to `0..Wishes` and `paidEscrow ≥ 0`.
5. **DataService `Tx` additions:** `CreditOpts.sku: string?`, and `tx.debit(currency, amount, reason, opts: {sku: string?}?)`. The sku flows into `TelCurrencyFlow.sku` and the economy event's `itemSku`.
6. **StyleService additions:** `lifeStyleResolved` (fires every spawn, before `LoadCharacterAsync`; non-yielding handlers), `swapNow`, `setSwapGate`, `freeRotation(now)`, `isOwned`. `equippedChanged` now fires only when the style differs from the previous life.
   - PlayerService MUST call `setSwapGate(CombatService.canTravel)` in `init()` and handle `swapNow` with `CombatService.spawnAsync`.
   - Part 6 MUST place totems as direct children of `<MapModel>/Totems`.
7. **AppearanceService:** the `OwnLook` attribute is written only at ready and at `lifeStyleResolved` (the base look of the current character), and AppearanceService sets `Player.CanLoadCharacterAppearance` before each spawn. Part 6 renders per the §5.7.5 truth table.
8. **MonetizationService:**
   - `grantProduct` (the single grant path for receipts and mocks), `isMock`, `setMockPass`;
   - **`BindReceiptHandler` catch-all** with a `ProcessReceipt` fallback;
   - IDs come from `Assets.products[game.GameId]` with keys `wishes_10/55/120/300`, `vip`, `quick_wish`, `lucky_jar`;
   - Skin keys `product.<key>.name`, `pass.<key>.name`.
9. **New remote** `Ftue.Step` (C2S event, rate 5 / 1, `struct{step = int(1, 9), skipped = bool}`, owner ProgressionService).
10. **New DevCmds** (Part 2 §2.11.5 enum): `setWishSeedMaster(seed)`, `simActivity(sec, dmg)`, `simKo(victim, rookie)`, `flushProgress()`. Studio seams:
    - `WishService._setSeedMaster`;
    - `PolicyGate._forceFail`;
    - `ProgressionService._simActivity/_simKo/_flushNow`;
    - `Telemetry._liveCalls/_lastSent/_setPerfSampled`;
    - `ModerationService._banCalls`.
11. **Log tag `TEL`** for the Studio analytics echo (GDD §18.10). Fixed lines:
    - `[ALG][Wish] batch ...`
    - `[ALG][Policy] flags ...`
    - `[ALG][Store] receipt ...` / `mock grant ...`
    - `[ALG][Shop] rotation|buy ...`
    - `[ALG][Progress] flush|level|reward_pending ...`
    - `[ALG][Style] equip|resolved ...`
    - `[ALG][Appearance] set ...`
    - `[ALG][Mod] report|ban dry_run ...`
12. **Rotation ID format** `<b36 period>:<style chars>:<cosmetic codes>` and the cosmetic `itemId` format `"<styleId>/<variantId>"`.
13. **Cross-server announcements:** implemented on `ALG_Announce_v1` with payload `{jobId, key, rarityId, styleId}`, gated by `Wish.Announce.CrossServerEnabled` (false in M1-G) and skipped in Studio; the subscribe runs in `task.spawn` with `pcall` so `start()` never yields (§5.2.13).
13a. **Player attribute `PerfSampled`** (boolean, default false; written by Telemetry at join, §5.8.3). Part 2 §2.5.2 lists it; the Part 6 VfxController perf sender reads it.
13b. **Rookie anti-farm is per earner** (§5.5.4): the server-memory log of paid Rookie KOs is keyed by the earner, not by the victim.
13c. **Title auto-equip** (§5.5.5 flush step 4): the first title granted while `account.title == ""` becomes the equipped title. No `title` settings key and no picker in M1-G (Part 6 §6.15 future hook).
14. **Skin keys needed (Part 8):**
    - `ui.notify.{wish_epic, wish_legendary, wish_mythic_global, wish_cosmic_global, level_up, mastery_up, daily_login, streak_3, streak_5, streak_10, streak_ended, rookie_ended, shop_bought, purchase_granted, report_thanks, quest_done}`
    - `ui.quest.<id>` (one per `Progression.Quests.DailyPool[*].id`, token `{goal}`) and `ui.hud.quests` (decision D-42)
    - `ui.wish.fakeout_note`, `ui.shop.goose_line_<n>` (n = 1…6; Part 6's form, adopted by Part 8 §8.9 item 6)
    - `variant.<styleId>.<vid>.name`
    - `title.level_<n>`, `title.mastery_<styleId>_5`, `title.master_<styleId>`
    - `ui.oosona.species.<id>`
15. **Telemetry mapping** (§5.8.3), including the reason→transactionType table, priority classes, and string-only custom fields.
16. **Lune test names and vectors** (§5.2.15, §5.2.16, §5.11), and the scenario names in §5.1.8, §5.2.17, §5.3, §5.4.3, §5.5.12, §5.6.7, §5.7.7, §5.8.5 and §5.9.

### 5.13 Issues with spine

1. **Brief vs Part 2: Ooo-sona "three colours".** The brief describes the Ooo-sona placeholder as three player-chosen colours. §2.4/§2.8.6 fixed `OosonaSpec` with one colour plus species, face, hat and outfit (≤ 30-char code), and the GDD says "color" (singular). This part keeps Part 2's spec and derives three colours: species skin, player colour, and outfit colour (§5.7.1). If three free colours are required, extend `OosonaSpec` to v2 with `color2`/`color3`; the code is still ≤ 50 chars.
2. **Telemetry echo tag.** GDD §18.10 specifies `[ALG][TEL]` lines; Part 2's tag list has `Tel`. This part uses `TEL` for echo lines and `Tel` for operational warnings. Resolved by decision D-02: the canonical tag list (Part 2 §2.3.6) contains both, with exactly these meanings.
3. **Receipt API wording.** GDD §18.5 says purchases are granted "through `ProcessReceipt`". This part uses the newer verified `BindReceiptHandler` (precedence over `ProcessReceipt`) with `ProcessReceipt` as fallback. The idempotency behaviour is identical. GDD wording only.
4. **File layout has no home for meta pure helpers** (curves, rotations, quest picks, `effectiveLook`). Per §2.10 item 5 they live in `Wish/Odds.luau` and `Wish/Pity.luau`, whose responsibilities are therefore broader than their names suggest. Recommend renaming them, or adding `Shared/Wish/Meta.luau`, in a later spine revision.
5. **"Cosmic: no pity" vs the hard-pity renormalization.** §0.11 and the GDD do not say whether floors and soft pity boost Cosmic. This part fixes Cosmic at 0.01% in every state. The Monte Carlo (§5.2.16) confirms §0.11 is reproduced, so no spine number changes.
6. **First-Wish guarantee vs M1-G kits.** GDD §15.5 picks `monarch` or `flame` 50/50. `flame` is not implemented in M1-G, so half of new players receive a non-equippable style from their guaranteed Wish. This part keeps the GDD numbers, because the config can be changed live through `ALG_Wish` without code. Flagged for the owner (§5.14).

### 5.14 Open questions / spikes

| ID | Question | Spike / owner | Fallback |
|---|---|---|---|
| APP-1 | Does `Player.CanLoadCharacterAppearance = true` + `LoadCharacterAsync` load the avatar under `AuthorityMode = Server` in Studio (Play Solo and local multi-client test players with negative IDs)? Does `false` give the default blocky R15 rig with Avatar Settings applied? | P0 spike (Part 1 list), `appearance.ownlook_on_respawn` | Route B (§5.7.3): `GetHumanoidDescriptionFromUserIdAsync` + `LoadCharacterWithHumanoidDescriptionAsync` (requires a Part 3 step-6 change) |
| MON-1 | Does `MarketplaceService:BindReceiptHandler` fire for Studio test purchases of dev products in the dev universe (H-09)? Does ProfileStore `.Mock` update `LastSavedData` after `Save()` so `waitSaved` completes? | Human + agent with H-09 products, `store.receipt_live_spike` | `ProcessReceipt` fallback (same body); in Mock mode, skip `waitSaved` |
| POL-1 | What does `PolicyService:GetPolicyInfoForPlayerAsync` return in Studio, especially for negative test IDs? | `policy.live_in_studio` | `StudioFailOpen` + `Dev.Command setPolicy` |
| MSG-1 | Is `MessagingService` usable in Studio sessions of a published dev place? This part skips it in Studio and keeps it off in M1-G (`Wish.Announce.CrossServerEnabled = false`); confirm live behaviour after H-11 only if the owner enables it through `ALG_Wish`. | P8 | Server-wide only (default) |
| DES-1 | Owner sign-off on tuning defaults: Shards per duplicate (3/5/10/25/60/150/300), Variant prices, bot reward ×0.5, repeat-victim curve applied to XP, `ActiveMinDamage = 10`, Mastery curve `400 + 80·m` (§2.14 item 9). | H-10 (design review) | Keep defaults |
| DES-2 | First-Wish guarantee in M1-G: keep GDD `monarch`/`flame` 50/50 (half get a non-equippable style), or temporarily override via `ALG_Wish` to `monarch` only? | Owner | Keep GDD |
| DES-3 | Should a totem swap respawn at the nearest spawn (needs a `spawnAsync` option from Part 3) rather than a random safe hub? | Part 3 / owner | Random safe spawn (current) |
| DES-4 | Should Cosmic craft be allowed when every Cosmic style is owned (Sparks → duplicate Shards)? This part refuses (`all_cosmic_owned`). | Owner | Refuse |
| DES-5 | Prestige (GDD §9.1) flow: automatic at 100, or a manual claim? Data only in M1-G; XP beyond 100 accumulates in `xpTotal`. | Owner | Future hook |
| DES-6 | M1-G free rotation makes 3 of the 4 rollable implemented styles free each week (75 %, vs the GDD's 3 of 16; GDD §8.1 `Count = 3` kept). This weakens the Wish/Shop chase the slice demonstrates (§5.1.4). Owner: keep, or disable it through `ALG_Progression` (`{"FreeRotation":{"Enabled":false}}`). | Owner (H-10 design review) | Keep (`Enabled = true`, `Count = 3`) |
| DES-7 | M1-G daily earn with world events stubbed and weekly quests a future hook is below GDD §17.5: ≈ 10.5 Wishes and ≈ 1,025 Treasure per 45-min day with daily quests on (≈ 75 % / 58 % of 14 / 1,775), ≈ 6.5 Wishes and ≈ 575 Treasure if quests were off (§5.5.2, §5.5.9). Owner: accept the gap for the slice, or add a temporary slice-only grant that the owner specifies (applied as a live `ALG_*` overlay and recorded in `PROGRESS.md`; `Defaults.luau` and every GDD number stay unchanged). | Owner (H-10 design review) | Accept; daily quests on |
| TEL-1 | The analytics "region" field (GDD §20.4 store events) is not sent (3-field budget). Confirm the dashboard's built-in location breakdown is enough; otherwise use `LocalizationService:GetCountryRegionForPlayerAsync` [Docs] in a custom field. | Live-ops | Omit |
| OPS-1 | Enabling real bans (`BAN_MODE = "live"`) requires a human-approved PR and Studio's `Players.BanningEnabled`; it is out of scope for M1-G. | Human | Dry-run |
