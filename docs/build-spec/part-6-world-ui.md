[← Build spec index](../STUDIO_BUILD_SPEC.md) · **Part 6 of 8** · sections §6.x · Part 0 contracts (§0.x) are in the index file

## 6. World, UI and presentation

This part specifies everything the player sees and hears in *Hullabrawl*, and the places on the Wishaway Isles where they fight:
- the map builder framework and the three maps: the Bandstand (`arena_duel_m0`, the M0 duel arena), Elsewhere-on-Sea (`hub`, the M1-G battleground, with its tide) and the Staff Room (`training`);
- lighting presets and the per-map colour scripts;
- the client presentation layer: RenderController, AnimController, VfxController, SfxController, CameraController, the cosmetic ragdoll, and appearance rendering (the Full Uniform, the Postcard Look and Charm Gear, the Peak uniform, the Style Tell, UI-drawn faces);
- the UI system (`UI/Create.luau`, `UI/Theme.luau`, every M1-G screen: HUD, Charm Ring and Scrapbook, Wish, Sundry's Rack and the Store, Settings, Results);
- the placeholder audio map;
- the `Defaults.Perf` config section (VFX and presentation budgets); `Defaults.Appearance` is owned by Part 5 §5.10 and includes the keys this part reads.

**Relationship to other parts (binding).**
- Part 3 owns the simulation and the RenderController *contract* (Part 3 §3.11.8): the transition table, the 15-frame undo rule, `RenderController.onRollback(f)`, and "no `GetAttributeChangedSignal` for one-shots". Part 3 also owns combat animation playback (`Adapter.animate` inside `BindToAnimation`). This part implements the render side and never writes HRP attributes.
- Part 4 owns kit presentation data: palettes, per-move VFX recipe and SFX maps, Charm Gear, Peak-piece and summon recipes, the placeholder-animation algorithm and pose library (Part 4 §4.6–§4.7). This part builds and plays them.
- Part 2 owns the DataModel tree, client init order, PlayerGui root names, `UiController`/`SettingsController` APIs, `Types`, the remote catalog and Player attributes. This part fills the screens and the roots Part 2 assigns to it.
- Part 5 owns appearance data and replication (`AppearanceService`), Wish/Shop/Store semantics and payloads (including the stamp-first reveal data and the Cosmic Arrival event). This part renders them.

**Names.** Code, Skin keys, config keys and instance names keep their internal identifiers (`StyleWheel`, `Totem`, `CharacterLook`, `Awakening`, `Realm`, `Flair`, `Variant`, `Treasure`, `Shards`, `CosmicSparks`, `Vol = "Spring"`). Prose about what the player sees uses the player-facing names of §0.12 (§0.12): Charm Ring, Charm Rack, Scrapbook, Full Uniform, Charm Gear, Postcard Look, Peak and the Peak Meter, Getaway, Finish, Off-Duty Look, Shells, Stamps, Messages in a Bottle, Mailbag, Guaranteed Delivery, Sundry's Rack, trampolines, "Checked out!".

**Conventions.**
- Frames are 60 Hz. `fNow = Frames.fromTime(time())` sampled once per render step. `e = fNow − StF` (Part 3) for the **local** character's view only; remote views, bots and carriers not owned by the local player use observation time `eR`/`ageR` (§6.6.1 "Timing per view").
- Distances are studs, times seconds (`Sec` suffix). Map coordinates are **map-local**: relative to the map Model's origin (§6.1.1), with **+Y up, North = −Z, East = +X**. A centre `(x, y, z)` is a Part's `Position`.
- UI sizes are **reference units** on the 1334 × 750 canvas (§6.9.3).
- `(td)` marks a tuning default chosen here because the GDD gives no number. GDD numbers are cited "GDD §x.y" and never changed.
- Every engine API named here was checked in the offline creator-docs snapshot (2026-10-10). Items that could not be verified carry "(verify in spike S-P6-n)"; the spikes are listed in §6.15.
- Every requirement has a **Verify** line: a Lune test (`tests/...`), an in-Studio scenario (Part 7 harness, printed `[HB][TEST] <name> PASS|FAIL`), an MCP call against `Roblox_Studio`, or an `[HB]` log line.

**Files owned by this part.**

| Path | Contents |
|---|---|
| `src/DevTools/Builders/BuildArena.luau`, `BuildMap_Hub.luau`, `BuildTraining.luau`, `BuildLighting.luau` | World builders (§6.1–§6.5) |
| `src/DevTools/Builders/BuildTemplates.luau`, `BuildVfxTemplates.luau`, `BuildPlaceholderAnims.luau` | Presentation templates (§6.6.7, §6.6.8, §6.8.6) |
| Additive "MapKit" functions in `src/DevTools/Builders/Util.luau` | Shared map helpers (§6.1.2); Part 1 owns the file |
| `src/StarterPlayerScripts/Controllers/RenderController, AnimController, VfxController, SfxController, CameraController, AppearanceController, HudController` | Presentation (§6.6–§6.8) |
| Part 6 sections of `UiController.luau`, `SettingsController.luau`, `InputController.luau` | Screen router, toasts, settings application, touch layout (§6.9) |
| `src/StarterPlayerScripts/UI/Create.luau`, `Theme.luau`, `Screens/*.luau` | UI system (§6.9) |
| `Defaults.Perf` (complete) in `src/Shared/Config/Defaults.luau` (`Defaults.Appearance` is Part 5 §5.10's, including this part's keys) | §6.11 |
| `tools/studio/map_stats.luau`, `tools/studio/export_anim.luau`, `tools/upload_anims.luau`, `tools/base64.luau`, `tests/core/upload_anims.spec.luau` | Map stats snippet (§6.1.9); animation upload pipeline, H-07 route (§6.6.9; built by conditional card T143) |
| Specified here, implemented inside Part 3 files: `Movement.volumeEffect` and `Movement.tidePhase` (`Core/Movement.luau`), volume and tide application (`Sim/Adapter.luau`), the world runtime block (`Services/CombatService.luau`) | §6.1.7, §6.1.8, §6.3.8 |

---

### 6.1 Map builder framework

#### 6.1.1 Instance layout and ownership

Every map is a `Model` with fixed child folders. Ownership follows Part 1 §1.6.1 and Part 2 §2.2. Owned roots carry `HbBuilder = "<Builder>"`.

| Map ID | Player-facing name | Root instance | Owner | Map origin (world) | Part 2 tree |
|---|---|---|---|---|---|
| `arena_duel_m0` | The Bandstand (M0 duel arena) | `Workspace.Arena` (Folder) › `DuelArena` (Model) | `BuildArena` | `(−2600, 0, 0)` | `Workspace.Arena.DuelArena` |
| `hub` | Elsewhere-on-Sea (M1-G battleground, 20 players) | `Workspace.Map.Hub` (Model) | `BuildMap_Hub` | `(0, 0, 0)` | `Workspace.Map.Hub` |
| `training` | The Staff Room (training area) | `Workspace.Map.Training` (Model) | `BuildTraining` | `(2600, 0, 0)` | `Workspace.Map.Training` |

`gullcliff` (Gullcliff) and `mountcocoa` (Mount Cocoa) are reserved Year-1 map IDs (§0.4); no builder exists for them in M1-G.

The origins keep every pair of map footprints more than 2,000 studs apart (Part 2 §2.2), so StreamingEnabled never streams two maps to one client.

**Root creation.** `Workspace.Map` is a **container** owned by `BuildAll`'s containers step (Part 1 §1.6.1, §1.6.6 step 0): a Folder stamped `HbBuilder = "BuildAll"`, never pruned and never destroyed by `reset`. The map builders own only their Model under it and never prune or stamp the container:
- `BuildMap_Hub`: `local root = Util.ensureRootOf(Util.ensureContainer(workspace, "Map", c), "Hub", "Model", NAME, c, opts.reset)`.
- `BuildTraining`: `local root = Util.ensureRootOf(Util.ensureContainer(workspace, "Map", c), "Training", "Model", NAME, c, opts.reset)`.
- `BuildArena`: `Workspace.Arena` is its own Folder root (`Util.ensureRoot`), and `DuelArena` is an owned child Model inside it.

`Util.ensureContainer(parent, name, c, quiet?)` (Part 1 §1.6.4) finds or creates the Folder (stamped `HbBuilder = "BuildAll"`, ignoring `reset`). The map builders call it with three arguments: if `Workspace.Map` was missing, `ensureContainer` itself adds the warning `container Workspace.Map missing; run BuildAll` to the builder's counter (only `BuildAll`'s containers step passes `quiet = true`); the map builder adds no second warning. Idempotency rule 2 (prune unproduced children) applies to the owned Model roots only, so `BuildMap_Hub` never deletes `Workspace.Map.Training` and vice versa.

Children of each map Model (every name is fixed):

| Child | Class | Contents |
|---|---|---|
| `Geometry` | Folder | Every static gameplay part (kinds `Floor`, `Wall`, `Cover`, `Decor`, `Boundary`, visual trampoline pads), grouped in sub-Folders by zone (`Geometry.<ZoneId>`); parts outside every zone (boundary, the Haar Line, backdrop) go in `Geometry.edge` |
| `Spawns` | Folder | Spawn marker Parts, attribute `SpawnFor = <MapId>` (Part 2 §2.5.1, Part 3 §3.13.4) |
| `Totems` | Folder | Charm Rack Models (internally Style Totems, §6.1.4) |
| `Breakables` | Folder | Breakable Models (§6.1.8) |
| `Zones` | **Model** with `ModelStreamingMode = Persistent` | Sub-folders `Volumes`, `KillVolumes`, `EventZones`, `ZoneBounds`. Persistent so that every client has every volume regardless of streaming, because both peers' simulations read volumes (§6.1.7) |
| `DummyPoints` | Folder | Training only (§6.4) |

Part 2 §2.2 lists `Zones` as a Folder and "Part 6 owns contents". This part makes it a Model so it can be streamed persistently, and adds `Breakables` (§6.13 item 1).

Map Model attributes (set by the builder):

| Attribute | Type | Meaning |
|---|---|---|
| `MapId` | string | Map ID (§0.4) |
| `KillY` | number | Map-local Y below which a character is KO'd by the void (§6.1.8) |
| `Origin` | Vector3 | World origin of the map (for map-local math at runtime) |
| `BuildVersion` | number | Bumped by hand when the layout tables change; logged by `verify()` |

#### 6.1.2 MapKit: additive functions in `Util.luau`

Map builders share these helpers. They go into Part 1's leaf `Util.luau` (still requiring nothing) so the §0.3 file list is unchanged. Each helper upserts through `Util.upsert`, so idempotency rules (Part 1 §1.6.2) hold.

```lua
--!strict
-- Additions to src/DevTools/Builders/Util.luau (leaf: requires nothing). Spec §6.1.2.
export type Kind = "Floor" | "Wall" | "Cover" | "Breakable" | "Hazard" | "Spawn" | "Event" | "KillVolume"
	| "Decor" | "Volume" | "Totem" | "Boundary" | "Trim"
export type Shape = "Block" | "Ball" | "Cylinder"
export type PartSpec = {
	kind: Kind,
	size: { number },                 -- {x, y, z} studs. Cylinder: {diameter, height, diameter} (upright)
	pos: { number },                  -- map-local centre {x, y, z}
	yaw: number?,                     -- degrees about +Y (default 0)
	shape: Shape?,                    -- default "Block"
	color: string?,                   -- "#RRGGBB" zone-palette override (kinds Floor/Wall/Cover/Decor only)
	attrs: { [string]: string | number | boolean | Vector3 }?,
}
export type KindStyle = {
	color: string, material: Enum.Material, transparency: number,
	collide: boolean, query: boolean, group: string, castShadow: boolean,
}

-- Semantic colour code (agent-assets §7.3) + flags. Kinds marked "zone" may take a palette override.
Util.KIND = {
	Floor      = { color = "#8C8C8C", material = Enum.Material.SmoothPlastic, transparency = 0,    collide = true,  query = true,  group = "Default",     castShadow = true },  -- zone
	Wall       = { color = "#4A4A4A", material = Enum.Material.SmoothPlastic, transparency = 0,    collide = true,  query = true,  group = "Default",     castShadow = true },  -- zone
	Cover      = { color = "#3A7BD5", material = Enum.Material.SmoothPlastic, transparency = 0,    collide = true,  query = true,  group = "Default",     castShadow = true },  -- zone
	Breakable  = { color = "#F28C28", material = Enum.Material.SmoothPlastic, transparency = 0,    collide = true,  query = true,  group = "Default",     castShadow = true },
	Hazard     = { color = "#E53935", material = Enum.Material.Neon,          transparency = 0.2,  collide = false, query = false, group = "Default",     castShadow = false },
	Spawn      = { color = "#43A047", material = Enum.Material.SmoothPlastic, transparency = 0.5,  collide = false, query = false, group = "Default",     castShadow = false },
	Event      = { color = "#FDD835", material = Enum.Material.SmoothPlastic, transparency = 0.6,  collide = false, query = false, group = "Default",     castShadow = false },
	KillVolume = { color = "#8E24AA", material = Enum.Material.SmoothPlastic, transparency = 0.8,  collide = false, query = false, group = "Default",     castShadow = false },
	Decor      = { color = "#F48FB1", material = Enum.Material.SmoothPlastic, transparency = 0.3,  collide = false, query = false, group = "Default",     castShadow = false }, -- zone
	Volume     = { color = "#7FD8BE", material = Enum.Material.SmoothPlastic, transparency = 1,    collide = false, query = false, group = "Default",     castShadow = false },
	Totem      = { color = "#00BCD4", material = Enum.Material.SmoothPlastic, transparency = 0,    collide = true,  query = true,  group = "Default",     castShadow = true },
	Boundary   = { color = "#000000", material = Enum.Material.SmoothPlastic, transparency = 1,    collide = true,  query = false, group = "MapBoundary", castShadow = false },
	Trim       = { color = "#2B2B2B", material = Enum.Material.SmoothPlastic, transparency = 0,    collide = false, query = false, group = "Default",     castShadow = false },
} :: { [string]: KindStyle }

-- Signatures of the new helpers (bodies follow the comments; all upserts go through Util.upsert).
export type MapKitApi = {
	-- Creates/updates one anchored Part under `parent`: props from Util.KIND[spec.kind] + spec; CanTouch always false;
	-- Shape via Part.Shape (upright cylinders get CFrame.Angles(0, 0, math.rad(90)) because Roblox cylinders run along X);
	-- writes spec.attrs + { Kind = spec.kind } with SetAttribute only when a value differs (counted as `updated`).
	mapPart: (parent: Instance, name: string, spec: PartSpec, origin: Vector3, c: Counter) -> BasePart,
	-- WedgePart ramp from `low` (map-local {x, z} at height y0) up to `high` ({x, z} at y0 + rise), `width` studs wide.
	-- Self-check: two downward workspace:Raycast probes (low end ≈ y0, high end ≈ y0 + rise, tolerance 0.5);
	-- if they read swapped, rotate the wedge 180° about Y. Deterministic.
	ramp: (parent: Instance, name: string, low: { number }, high: { number }, y0: number, rise: number,
		width: number, origin: Vector3, c: Counter) -> WedgePart,
	-- Pure: splits rect {x0, x1, z0, z1} around axis-aligned holes into ≤ 4 rects per hole, fixed order
	-- (west, east, north, south strips). Used for floor slabs with trapdoors.
	slabRects: (rect: { number }, holes: { { number } }) -> { { number } },
	-- Find-or-create an owned root of any class (Model or Folder) stamped with HbBuilder (generalises ensureRoot).
	ensureRootOf: (parent: Instance, name: string, className: string, owner: string, c: Counter, reset: boolean?) -> Instance,
}
```

Rules for every map builder:
1. Layout lives in **data tables at the top of the builder** (one row per part, as in §6.2–§6.4). Logic only iterates tables. Gameplay tunables (trampoline speed, slow multiplier, tide period, break HP, rebuild time) come from `Defaults.Perf.World` (§6.11) and are written as **attributes** on the volume, `Zones` or breakable instances, so both peers read identical values from the place.
2. Parts are created in table order. Names are unique within their folder (`<Zone>_<Thing>_<n>`).
3. Map geometry is in collision group `Default` (Part 3 §3.5.5, §3.12.1). Only `Boundary` parts use `MapBoundary` (§6.1.5).
4. `Anchored = true` on every part. `CanTouch = false` everywhere (the world runtime uses AABB tests, not `Touched`).
5. No scripts, no asset IDs. AI-generated props from `ServerStorage.AIProps` (Part 1 C10) may replace `Decor` parts only, with a primitive fallback when the folder is missing. Prompts for them name generic seaside objects only (§0.5).

#### 6.1.3 Zone palettes (Elsewhere-on-Sea)

Gameplay-relevant kinds (`Breakable`, `Hazard`, `Spawn`, `Event`, `KillVolume`, `Volume`, `Totem`) always use the semantic colour code. `Floor`, `Wall`, `Cover` and `Decor` take the zone palette below, so a player knows the zone at a glance (GDD §11.1) while blockout meaning is preserved. The palette follows the hub's colour script (§6.5.3): cream stucco and sea-glass, with coral bunting and lemon as accents and periwinkle in the shade. The Bandstand and the Staff Room use their own palettes (§6.2, §6.4).

| Zone (`ZoneId`) | Floor | Wall | Cover | Accent (Decor) | Source |
|---|---|---|---|---|---|
| `under_pier` | `#3B4A4F` (wet slate) | `#2E3A3F` | `#5A4A3A` (tarred pier piles) | `#FFE9A8` (sun through the plank gaps) | GDD §11.2 A "Under the Pier" (td) |
| `pavilion` | `#F3E9D2` (cream stucco paving) | `#FFF3D6` | `#8FD3C1` (sea-glass tubs) | `#FF7F66` (coral bunting) | GDD §12.6 hub colour script |
| `promenade` | terrain Pavement | `#F3E9D2` | `#D9826E` (stacked deckchairs) | `#F7E06B` (lemon) | GDD §12.6 |
| `beach` | terrain Sand | `#FFF3D6` | `#D9B870` (sandcastles) | `#FF7F66` (parasols) | GDD §11.2 A (td) |
| `bandstand_gardens` | terrain LeafyGrass | `#4F7D4A` (clipped hedge) | `#C97B63` (terracotta beds) | `#9FA8D6` (periwinkle) | GDD §11.2 A (td) |
| `jubilee_clock` | terrain Ground | `#E8E4D8` (chalk) | `#9A9486` (sea-worn rock) | `#2F5DA8` (cobalt clock face) | GDD §11.2 A (td) |

Trampoline pads (kind `Floor`, colour override `#2F5DA8` cobalt) use one colour on every map, so a trampoline reads the same everywhere.

**Wall-splat marking** (GDD §11.1): every part with `WallSplat = true` gets a `Trim` part: a dark band `#2B2B2B`, 0.6 studs tall, 0.2 studs proud of each face at its base, non-colliding. Beach huts use one trim block per hut (§6.3.4).

#### 6.1.4 Attribute tagging contract

| Attribute | On | Type | Writer | Meaning / rule |
|---|---|---|---|---|
| `Kind` | every map BasePart | string (`Util.Kind`) | builder | Semantic kind; `verify()` checks flags against `Util.KIND` |
| `ZoneId` | `ZoneBounds` parts; `Geometry.<ZoneId>` folders | string | builder | One of the IDs in §6.3.1; HUD zone label (§6.9.5) |
| `WallSplat` | Wall/Cover/Breakable parts | boolean | builder | Marks a deliberate splat surface (trim added). Informational: Part 3's wall probe hits any steep `Default` surface |
| `Breakable` | Breakable **Model** | boolean | builder | §6.1.8 |
| `BreakHp`, `BreakFromAir`, `RebuildSec` | Breakable Model | number / boolean / number | builder (from `Perf.World.Breakable`) | §6.1.8 |
| `State`, `Hp`, `BrokenAt`, `RebuildAt` | Breakable Model | string / number / number / number | server world runtime | §6.1.8; `BrokenAt`/`RebuildAt` are `workspace:GetServerTimeNow()` seconds |
| `T0` | Breakable chunk parts | number | builder | Starting transparency, restored at rebuild |
| `Hazard`, `HazardDamage`, `HazardTelegraphSec`, `HazardPeriodSec` | Hazard parts | boolean / number / number / number | builder | GDD §11.1: damage ≤ 15, telegraph ≥ 1.5 s. **No hazard instances exist in M1-G maps** (§6.13 item 12) |
| `SpawnFor` | spawn Parts | string (MapId) | builder | Part 3 `pickSpawn`; the part's CFrame faces the map centre |
| `HubId` | spawn Parts, Totem Models | string `"N"`/`"S"`/`"E"`/`"W"`/`"T"` | builder | Groups spawns with their hub |
| `StyleTotem` | Totem Model (a Charm Rack) and its `Pillar` part | boolean | builder | §6.3.6 |
| `TotemRadius` | Totem Model | number | builder (from `Progression.TotemRangeStuds`, 12, Part 5 §5.10) | Used **only** for the ProximityPrompt `MaxActivationDistance`. The server range rule is Part 5 §5.1.3 (distance from the requester's HRP to the Totem **Model pivot** ≤ `Progression.TotemRangeStuds`); the server never reads `TotemRadius` (§6.3.6) |
| `Vol` | `Zones.Volumes` parts | string `"Spring"` (a trampoline) / `"Slow"` / `"Boundary"` | builder | §6.1.7 |
| `LaunchVy` / `SlowMult` / `Inward`, `BandStuds`, `MinMult` | volume parts | number / number / Vector3, number, number | builder | §6.1.7 |
| `Tide` | `Zones.Volumes` parts (optional) | string `"low"`/`"high"` | builder | The volume is active only in that tide phase; absent = always active (§6.1.7, §6.3.8) |
| `TidePeriodSec` | the map's `Zones` Model (hub only) | number | builder (from `Perf.World.Tide.PeriodSec`, 240) | Tide schedule read by both peers (§6.3.8); absent = no tide. Only the Studio seam `setTidePeriod` changes it at runtime |
| `TidePhase` | the map Model (hub only) | string `"low"`/`"high"` | server world runtime | Mirror of the current phase for the HUD fallback, tests and MCP inspection; **never read by the sim** (§6.3.8) |
| `KillVolume` | `Zones.KillVolumes` parts | boolean | builder | §6.1.8 |
| `EventZone` | `Zones.EventZones` parts | string | builder | World-event stub zones (§6.3.7) |
| `GateStub`, `PatrolStub` | stub Models | boolean | builder | Future-hook markers (§6.3.5, §6.3.7) |
| `DummySlot` | `DummyPoints` parts | number | builder | Part 3 §3.15.2 |
| `Mode` | `DummyPoints` parts | string | builder (`"idle"`), then BotService | Current dummy mode, replicated for the Staff Room UI (§6.4, §6.13 item 7) |

#### 6.1.5 Collision groups

Part 3 §3.12.1 lists all eight groups (8 of 32), and `Combat/Hurtbox.ensureGroups()` (Part 3's single registration point) registers them and sets the full matrix, so the matrix lives in one place and persists in the place file (§6.13 item 2). Two of the eight exist for this part; Part 6 adds no second registration point:

| Group | Members | Collidable with | Why |
|---|---|---|---|
| `MapBoundary` | invisible boundary walls and caps (`Kind = "Boundary"`) | `Characters`, `RagdollFx` | Keeps characters in the map without being a wall-splat surface: `WorldQuery` (the wall probe) does not collide with it |
| `RagdollFx` | client-only ragdoll clones and breakable debris (§6.7, §6.1.8) | `Default`, `MapBoundary` | Debris and ragdolls must land on floors. They must never touch `Characters` (pushing a predicted character mispredicts) |

Client visuals that must not collide at all (proxy look parts, gear, rings, VFX parts) use Part 3's `CosmeticNoQuery` with `CanQuery = false`, `CanCollide = false`, `CanTouch = false`.

**Verify:** `BuildArena.verify()` and `BuildMap_Hub.verify()` report `[builder:<Name>] group MapBoundary missing (run BuildInputs)` when `workspace:IsCollisionGroupRegistered("MapBoundary")` is false, and check the matrix with `workspace:CollisionGroupsAreCollidable` (`MapBoundary`×`Characters` true, `MapBoundary`×`WorldQuery` false, `RagdollFx`×`Characters` false, `RagdollFx`×`Default` true).

#### 6.1.6 Terrain usage

Only `hub` uses Terrain. The arena and Training are part-only.
- **Materials and colours** are set once per build with `Terrain:SetMaterialColor(material, color)`. This recolours the material everywhere, so no other map may use these terrain materials.

  | Material | Colour | Use |
  |---|---|---|
  | `Grass` | `#A9C47F` (dune grass) | Outer ring outside the boundary |
  | `LeafyGrass` | `#8FC27A` (mown lawn) | Bandstand Gardens |
  | `Pavement` | `#E8DCC4` (cream paving) | The Promenade |
  | `Ground` | `#CDBB8C` (packed dune sand) | The Jubilee Clock corner |
  | `Sand` | `#EAD9A8` | The Beach; the dunes |
  | `Rock` | `#B7B1A6` (chalk and flint) | Decorative headlands outside the boundary |

- **`Terrain.Decoration` is human-set at H-04** (unchecked; GDD §12.5: no realistic 3D grass). The property is tagged `NotScriptable` (`Terrain.yaml`), so no Luau (builder, command bar or `execute_luau`) can read or write it; the builder never touches it.
- **No Terrain water.** The shallows are flat sand with a translucent `Decor` water sheet and `Slow` volumes (§6.3.5, §6.3.8); the sea beyond the north boundary is a `Decor` plane (§6.3.7). Swimming would change Humanoid state and physics under prediction.
- Fill order (deterministic; a `reset` build first clears the full region with `Terrain:FillBlock(cf, size, Enum.Material.Air)`):
  1. `Grass` slab: `FillBlock` region x[−360, 360] × z[−360, 360], y[−4, 0].
  2. Zone materials over the same y range: Promenade x[−300,−100] z[−300,100] `Pavement`; Beach x[−100,300] z[−300,−100] `Sand`; Gardens x[100,300] z[−100,300] and x[−100,100] z[100,300] `LeafyGrass`; Jubilee Clock corner x[−300,−100] z[100,300] `Ground`.
  3. Clear (`Air`) the part-floored regions: the Pavilion square x[−100,100] z[−100,100]; the promenade deck over Under the Pier x[−200,−100] z[−70,70].
  4. Dunes: `FillBall` with `Sand`: centres `(−250,−6,150)` r 16, `(−180,−8,230)` r 18, `(−130,−6,160)` r 14, `(−240,−8,270)` r 16. Outside the boundary, `Rock` headlands at the four corners `(±340, −10, ±340)` r 30.
- Terrain voxels are not instances, so terrain writes do not increment `created`/`updated`. The idempotency test (Part 1 §1.6.8) still passes.

**Verify:** `verify()` checks `Terrain:GetMaterialColor(m)` for the six materials (tolerance 1/255) and reads 6 sample voxels with `Terrain:ReadVoxels` (one each for the Promenade, the Beach, Bandstand Gardens, the Jubilee Clock corner and the outer ring, plus the Pavilion square hole, which must be `Air`). It does **not** read `Terrain.Decoration` (NotScriptable); it adds the non-failing reminder line `[HB][BUILD] reminder Terrain.Decoration is human-set at H-04 (NotScriptable)` to its output (not a problem string).

#### 6.1.7 Predicted volumes (trampolines, the shallows and the tide, soft boundary)

Movement effects must be **predicted** (computed identically on the owner client and the server inside the sim). A server-only launch would rubber-band. Volumes are therefore static data read by both peers, and the tide that switches the shallows is a pure function of the simulation frame (§6.3.8).

**Data.** Each volume is an invisible Part under `<Map>.Zones.Volumes` whose world AABB (from `CFrame`/`Size`; volumes are never rotated) defines the region:

| `Vol` | Attributes | Effect (applied by the Adapter, inside the sim) |
|---|---|---|
| `Spring` (a trampoline) | `LaunchVy` (studs/s) | If the root is inside, `St ∈ NEUTRAL` and `AssemblyLinearVelocity.Y ≤ 1`: `Humanoid:ChangeState(Enum.HumanoidStateType.Freefall)` then `root.AssemblyLinearVelocity = Vector3.new(v.X, LaunchVy, v.Z)` |
| `Slow` | `SlowMult`; optional `Tide` | While inside and grounded: walk speed × `SlowMult`. A volume with `Tide` is active only while the tide phase equals it (§6.3.8) |
| `Boundary` | `Inward` (unit Vector3), `BandStuds`, `MinMult` | While inside the band and moving outward (`moveDir · Inward < 0`): walk speed × `1 − (1 − MinMult) · clamp(depth / BandStuds, 0, 1)` |

Any volume kind may carry `Tide`; in M1-G only the two shallows volumes do.

**Pure functions** (added to Part 3's `Core/Movement.luau`; Lune-testable):

```lua
--!strict
-- Additions to src/Shared/Core/Movement.luau (pure; no Instances). Spec §6.1.7, §6.3.8.
export type VolumeKind = "Spring" | "Slow" | "Boundary"
export type TidePhase = "low" | "high"
export type Volume = {
	kind: VolumeKind, min: Vector3, max: Vector3,   -- world AABB
	launchVy: number?, slowMult: number?,
	inward: Vector3?, bandStuds: number?, minMult: number?,
	tide: TidePhase?,                               -- active only in this tide phase; nil = always active
}
export type VolumeEffect = { speedMult: number, launchVy: number? }

local function inside(p: Vector3, v: Volume): boolean
	return p.X >= v.min.X and p.X <= v.max.X and p.Y >= v.min.Y and p.Y <= v.max.Y
		and p.Z >= v.min.Z and p.Z <= v.max.Z
end

-- Tide phase at absolute simulation frame f (§6.3.8). periodF <= 0 means "no tide" (always "low").
function Movement.tidePhase(f: number, periodF: number): TidePhase
	if periodF <= 0 or math.floor(f / periodF) % 2 == 0 then
		return "low"
	end
	return "high"
end

-- Returns the walk-speed multiplier (rounded to 0.01, Q-rule) and an optional trampoline launch speed.
-- tidePhase = nil (a map without a tide) skips every volume that carries a Tide tag.
function Movement.volumeEffect(pos: Vector3, vel: Vector3, grounded: boolean, neutral: boolean,
	moveDir: Vector3, vols: { Volume }, tidePhase: TidePhase?): VolumeEffect
	local mult = 1
	local launch: number? = nil
	for _, v in vols do -- array order is deterministic (builder order)
		if (v.tide == nil or v.tide == tidePhase) and inside(pos, v) then
			if v.kind == "Spring" then
				if launch == nil and neutral and vel.Y <= 1 then
					launch = v.launchVy
				end
			elseif v.kind == "Slow" then
				if grounded then
					mult *= (v.slowMult or 1)
				end
			elseif v.kind == "Boundary" then
				local inward = v.inward or Vector3.zero
				if moveDir:Dot(inward) < 0 then
					local c = (v.min + v.max) / 2
					local half = (v.max - v.min) / 2
					local face = c + inward * math.abs(half:Dot(inward)) -- inner face of the band
					local depth = (face - pos):Dot(inward)
					local band = v.bandStuds or 1
					local frac = math.clamp(depth / band, 0, 1)
					mult *= 1 - (1 - (v.minMult or 1)) * frac
				end
			end
		end
	end
	return { speedMult = math.floor(mult * 100 + 0.5) / 100, launchVy = launch }
end
```

**Adapter integration (Part 3 file `Sim/Adapter.luau`, both peers):**
1. Outside the sim, at `track` time, the roster entry caches `entry.vols: { Movement.Volume }` and `entry.tidePeriodF: number` for the entry's map. The list is built once per map from `<Map>.Zones.Volumes` children sorted by `Name` (each part's `Tide` attribute fills `tide`); `tidePeriodF = (Zones:GetAttribute("TidePeriodSec") or 0) × 60`. Both are cached per `MapId` and rebuilt on `Zones.Volumes.ChildAdded` and on a `TidePeriodSec` attribute change (client streaming safety; `Zones` is Persistent).
2. Inside phase 1, after `Fsm.step` and before `Loco.apply`: `local tide = if entry.tidePeriodF > 0 then Movement.tidePhase(f, entry.tidePeriodF) else nil`; `local ve = Movement.volumeEffect(root.Position, root.AssemblyLinearVelocity, ctx.grounded, inNeutral, inp.moveDir, entry.vols, tide)`; `loco.walkSpeed *= ve.speedMult`; if `ve.launchVy` then change state and write the velocity as in the table. `f` is the step's absolute frame (`Frames.fromTime(time())`, Part 3 §3.1.3). `BasePart.AssemblyLinearVelocity` and `Humanoid:ChangeState` are Simulation-Access [Docs `BasePart.yaml`, api-sheet §1.5]. Whether the trampoline launch predicts without rollback is spike S-P6-5; whether the tide swap does is spike S-P6-15.

**Verify:** Lune `tests/core/movement_volumes.spec.luau`: `movement.volume_spring_neutral_only` (Idle → launch; Hitstun → none; vy = 5 → none), `movement.volume_slow_grounded` (0.6 grounded, 1.0 airborne), `movement.volume_boundary_depth` (depth 0 → 1.0, 10/20 → 0.6, 20/20 → 0.2, moving inward → 1.0), `movement.volume_order_deterministic`, `movement.volume_tide_gating` (a `tide = "high"` Slow volume applies only when `tidePhase == "high"`; an untagged volume applies in both phases; `tidePhase = nil` skips every tagged volume), `movement.tide_phase` (§6.3.8). Scenario `world.spring_launch` (character walks onto the Staff Room trampoline `Tramp_Test`; within 30 frames the server reads root Y ≥ pad top + 20; the client `RunService:GetPredictionStatus(root)` stays `Predicted` and the Part 7 misprediction counter for `St` stays 0). Scenario `world.shallows_slow` (on the hub, a character standing in the low-tide strip at (240, ·, −262), which is wet at both tides: WalkSpeed reads 18 × 0.6 = 10.8 on both peers).

#### 6.1.8 Server world runtime (void kills, breakables, rebuild, the tide mirror)

The layout has no world service. The runtime is therefore a clearly delimited block inside Part 3's `Services/CombatService.luau` (`-- ===== World runtime (Spec §6.1.8) =====`), exposed as `CombatService.world`. It runs **outside the sim** on the server, on `RunService.Heartbeat` throttled to `Perf.World.ScanHz` (10 Hz).

```lua
--!strict
-- CombatService.world (server; outside the simulation). Spec §6.1.8.
export type BreakState = "intact" | "cracked" | "broken"
export type WorldApi = {
	scan: () -> (),                                  -- (re)index Breakables and KillVolumes under Workspace.Map / Workspace.Arena
	breakNow: (model: Model, cause: string) -> (),   -- also used by Dev.Command breakAll (§6.13 item 9)
	rebuildNow: (model: Model) -> (),
	stats: () -> { breakables: number, broken: number, voidKills: number },
	tidePhase: (mapId: string) -> string?,           -- "low" | "high" for a map with TidePeriodSec, else nil (§6.3.8)
}
```

**Void and kill volumes.** On each scan tick, for every ready roster entry (`Adapter.list()`) whose `St ≠ KO` and that has not already been killed this life:
- if `root.Position.Y − mapOrigin.Y < map:GetAttribute("KillY")`, or the root is inside any `KillVolume` AABB of its map → `CombatService.killAsync(target, "environment")` once, and log `[HB][World] void_kill rid=<n> map=<mapId>`.
- KO credit: GDD §11.1 says hazards and environment KOs credit "the last attacker within 5 s". Part 3 owns the rule: `Combat.Streak.EnvKoCreditWindowSec = 5` (Part 3 §3.17) and §3.13.2 credit `lastHit.rid` for `killAsync(…, "environment")` only if `f − lastHit.f ≤ toFrames(EnvKoCreditWindowSec)` (the general 10 s `KoCreditWindowSec` applies to all other KOs). The world runtime only passes `cause = "environment"`; `Perf.World.Hazard` holds no credit window (§6.11, §6.13 item 4). Scenario `combat.env_ko_credit` (Part 3 §3.13 Verify).

**Tide mirror (hub only, §6.3.8).** On each scan tick the runtime computes `Movement.tidePhase(Frames.fromTime(time()), TidePeriodSec × 60)` for every map whose `Zones` Model carries `TidePeriodSec`. When the result differs from the map Model's `TidePhase` attribute it writes the new value and logs `[HB][World] tide phase=<low|high> f=<f> map=<mapId>`. This attribute is a mirror for the HUD fallback, tests and MCP inspection only. The simulation never reads it: both peers compute the phase themselves from `f` (§6.1.7), so a late attribute can never cause a misprediction.

**Breakables.** A Breakable is a `Model` with `Breakable = true` holding pre-split chunk Parts (kind `Breakable`).

| Step | Rule |
|---|---|
| Contact (server) | Every scan tick, for each entry whose root lies inside the model's bounding box inflated by `BreakInflateStuds` (1.5, td), the entry is a *breaker* when `St ∈ {Ragdoll, Launched, WallSplat}` (the character was slammed into it), or when the model has `BreakFromAir = true` and `St == Air` with `|AssemblyLinearVelocity.Y| ≥ BreakMinVy` (40, td; trapdoors only). Dedup key `"<rid>:<StF>"` per model, so one slam deals one point |
| Damage | `Hp −= 1`. At `Hp > 0`: `State = "cracked"`. At `Hp ≤ 0`: break |
| Break | Each chunk: `CanCollide = false`, `CanQuery = false`, `Transparency = 1`. Model: `State = "broken"`, `BrokenAt = now`, `RebuildAt = now + RebuildSec` (90 s, GDD §11.1). Log `[HB][World] break id=<name> cause=<slam\|air\|dev>` |
| Rebuild | At `now ≥ RebuildAt`: if any root is inside the inflated box, postpone by 1 s; else restore chunk `CanCollide = true`, `CanQuery = true`, `Transparency = T0`, `State = "intact"`, `Hp = BreakHp`. Log `[HB][World] rebuild id=<name>` |
| Client | VfxController watches `State` through `GetAttributeChangedSignal` (allowed: this is not a predicted one-shot). On `cracked`: darken chunks 15 % locally plus a puff. On `broken`: spawn pooled debris (clones of the chunks, unanchored, group `RagdollFx`, outward impulse, lifetime random in `Perf.Vfx.DebrisLifeSec` = 6–10 s, GDD §11.1). During the last `Perf.Vfx.RebuildRedrawSec` (3 s, td) before `RebuildAt`, the "fresh paint" effect (GDD §11.1): local ghost copies of the chunks fade from transparency 1 to 0.5, each with an ink outline made from a second copy 0.1 stud larger in the ink colour `Perf.Vfx.Ink` (`#241C33`) at transparency 0.7, as if the Staff were painting the hut back up overnight |

`now` is `workspace:GetServerTimeNow()` (§0.4 meta clock).

Placement rules (`verify()` enforces them):
- No breakable within `Perf.World.Breakable.SpawnClearStuds` (60) of any spawn hub centre. GDD §11.1: destruction never opens sightlines into spawns.
- A trapdoor has an open drop under it and a trampoline (`Spring` volume) on the floor below, so nobody is ever trapped.

**Optional fracture (GDD §18.8).** By default, chunks are deterministic quarters (§6.3.4). With `build({fragment = true})` the builder runs `GeometryService:GenerateFragmentSites(part, {SiteSpacing = 6})` and `GeometryService:FragmentAsync(part, sites, {SplitApart = true})` once per **template** (one beach hut, one trapdoor) and stores the MeshParts under `ServerStorage.Templates.Fragments.<Template>`. Later builds reuse the cache unless `opts.refragment = true`. Site generation is not repeatable [api-sheet §18.6], so idempotency holds only because the cache is reused.

**Verify:** scenarios `world.void_kill` (character teleported to `KillY − 5` → `St = KO` within 0.2 s; log line present), `world.hut_break_rebuild` (bot Ender into the beach hut `Hut_2_0` twice → `State` `cracked` then `broken`; debris count > 0 on the client; `Dev.Command setTimeOffset 90` → `State = "intact"`), `world.trapdoor_fall_and_trampoline` (break `Trapdoor_E`; a character on it falls to y ≈ −33, Under the Pier; standing on the trampoline `PierTramp_E` launches it above y = 0 and breaks the rebuilt trapdoor from below), `world.rebuild_postponed_when_occupied`, `world.tide_swap` (§6.3.8).

#### 6.1.9 Budgets and common `verify()` checks

Budgets (from `Perf.World.Budget`, §6.11):

| Map | Max BaseParts | Max breakables | Spawns | Totems | Source |
|---|---|---|---|---|---|
| `arena_duel_m0` | 400 | 0 | 8 | 0 | (td) |
| `training` | 400 | 0 | 2 | 1 | (td) |
| `hub` | 2,500 | 42 | 16 | 4 | (td); GDD §11.1 (4 hubs) |

On-screen triangles and draw calls (GDD §12.8 map ≤ 220k / 450k / 800k triangles; GDD §18.7 draw calls ≤ 600 / 900 / 1,500) are measured by Part 7's perf pass with the Render Stats overlay. Graybox parts are cheap, so part count is the builder-side proxy.

Every map builder's `verify()` runs these checks; each problem string starts with `[builder:<Name>]`:
1. Root exists, is owned, has `MapId`, `KillY`, `Origin`.
2. Every BasePart has `Kind`; its flags (`CanCollide`, `CanQuery`, `CanTouch = false`, `CollisionGroup`, `Anchored = true`) match `Util.KIND`.
3. Counts match the budget table (spawns and totems exactly; parts and breakables at most).
4. Spawn parts face the map centre within 5°.
5. Breakable placement rules (§6.1.8).
6. Hazards (if any): `HazardDamage ≤ 15`, `HazardTelegraphSec ≥ 1.5` (GDD §11.1).
7. `Zones` is a Model with `ModelStreamingMode == Enum.ModelStreamingMode.Persistent`; every part under `Zones.Volumes` has a valid `Vol` and that kind's attributes, and a `Tide` value (if any) of `"low"` or `"high"`; a map with tide-tagged volumes has `Zones.TidePeriodSec > 0`; every `ZoneBounds` part has a `ZoneId`.
8. **Wall splat and verticality per zone** (GDD §11.1; Elsewhere-on-Sea only): each zone has ≥ 4 parts with `WallSplat = true`, and ≥ 1 walkable top surface at least 8 studs above the zone's floor (layer 2). Under the Pier is exempt from the layer-2 half: its ceiling is the street level above, and its trampolines are its vertical play.
9. **Sightline audit** (GDD §11.1, ≤ 150 studs; Elsewhere-on-Sea only): from every point of a 20-stud grid inside the playable square, at local Y = 5, cast `workspace:Raycast` in 8 compass directions with length 160, using `RaycastParams` with `CollisionGroup = "WorldQuery"` and `ExcludeInstances = {Zones}`. A ray that travels more than `Perf.World.Sightline.MaxStuds` (150) without a hit and whose end stays inside the playable square is a problem: `sightline <len> from (x,z) dir <d>`. **Fix rule for the agent:** add a `Cover` block (6 × 6 × 6, zone colour) to the layout table at the ray midpoint, snapped to the nearest promenade lane centre or open area, then re-run. The fix is a data change, so the build stays deterministic.

Studio snippet `tools/studio/map_stats.luau` (Edit; ARGS `{mapId}`; ≤ 1.5 KB): returns `{parts, byKind, byZone, breakables, spawns, totems, worstSightline}` as JSON.

**Verify (all builders):** Part 1 §1.6.8 tests (`build_fresh`, `build_idempotent`, `build_reset_equal`, `verify_detects`, `no_sync_side_effects`) for each of `BuildArena`, `BuildTraining`, `BuildMap_Hub`, `BuildLighting`, `BuildTemplates`, `BuildVfxTemplates`, `BuildPlaceholderAnims`.

---

### 6.2 M0 duel arena `arena_duel_m0` (the Bandstand)

The Bandstand is the competitive standard of GDD §11.2 D and the M0 arena: a 120 × 120 lawn inside high clipped hedges, a raised bandstand stage in the middle, four painted kiosks in the corners, and **no hazards**. Display name: `Skin.text("map.arena_duel_m0.name")` (the Bandstand).

#### 6.2.1 Layout (map-local; origin `(−2600, 0, 0)`)

| Name | Kind | Shape / size | Centre | Notes |
|---|---|---|---|---|
| `Floor_<i>_<j>` (36) | Floor | Block 20 × 4 × 20 | x, z ∈ {−50, −30, −10, 10, 30, 50}, y −2 | Mown-lawn stripes: colour `#B9D9A0` when `(i+j)` is even, else `#C7E3B0`. The tiles are the scale reference (agent-assets §7.3) |
| `Wall_N` / `Wall_S` | Wall, `WallSplat` | 128 × 30 × 4 | (0, 15, ∓62) | Clipped hedge, colour `#5E8C5A` |
| `Wall_W` / `Wall_E` | Wall, `WallSplat` | 4 × 30 × 120 | (∓62, 15, 0) | |
| `Trim_N/S/W/E` | Trim | 120 × 0.6 × 0.4 (N/S); 0.4 × 0.6 × 120 (W/E) | inner face base: (0, 0.3, ∓59.8), (∓59.8, 0.3, 0) | GDD §11.1 ink border |
| `Cap_N/S/W/E` | Boundary | 128 × 60 × 4 / 4 × 60 × 120 | (0, 60, ∓62) / (∓62, 60, 0) | Invisible caps from y 30 to 90: uppercut follow-ups (20 studs + hang) cannot land on hedge tops |
| `Stage` | Cover, `WallSplat` | 16 × 12 × 16 | (0, 6, 0) | The bandstand stage; top = layer 2 at y 12. Colour `#F2EDE0` (white-painted timber) |
| `Kiosk_NE/NW/SE/SW` | Cover, `WallSplat` | 12 × 12 × 12 | (±36, 6, ±36) | Painted kiosks, colour `#8FD3C1` |
| `Ramp_<corner>` (4) | Floor (WedgePart via `Util.ramp`) | width 8, rise 12 | low (±12, ·, ±36) → high (±30, ·, ±36), same sign as the kiosk | Climbs along X to each kiosk's inner face |
| `Bridge_<corner>` (4) | Floor | 4 × 1 × 31.1 | (±19, 11.5, ±19), yaw ±45° along the diagonal | From each kiosk's inner corner (±30, ±30) to the stage's corner (±8, ±8); layer 2 is connected |
| `Bunting_<n>` (4) | Decor `#FF7F66` | Block 0.4 × 0.4 × 46 | (0, 26, ±31) along Z; (±31, 26, 0) along X (yaw 90°) | Bunting lines from above the stage toward each hedge; floating, non-colliding |
| `EmptyDeckchair` | Decor `#E8B4A0` | Block 3 × 3 × 4 | (52, 1.5, 52) | One empty deckchair with an open book: a quiet landmark that every map carries; never interactive |
| `Spawn_<n>` (8) | Spawn, `SpawnFor = "arena_duel_m0"` | 4 × 1 × 4 | (−24, 0.5, −50), (24, 0.5, −50), (−24, 0.5, 50), (24, 0.5, 50), (−50, 0.5, −24), (−50, 0.5, 24), (50, 0.5, −24), (50, 0.5, 24) | `CFrame.lookAt(pos, origin-height centre)`. 8 spawns serve 8 clients + bots (§0.0 M0) |
| `Zones.KillVolumes.Kill_Floor` | KillVolume | 200 × 10 × 200 | (0, −50, 0) | Safety net |

Map attributes: `MapId = "arena_duel_m0"`, `KillY = −40`. Lighting preset `arena_duel_m0.Day` (§6.5).

**Out of bounds.** Hedges plus caps make leaving impossible. Falling through geometry hits `KillY`. The arena has no volumes and no tide.

**Sightlines.** The duel arena is exempt from the 150-stud rule (GDD §11.1 applies to battleground maps), but the stage and the kiosks still break every centre-to-corner line.

#### 6.2.2 M0 spawn selection

Part 2's default `MapId` is `"hub"`, which does not exist in M0. PlayerService must set the initial `MapId` to `"hub"` when `Workspace.Map.Hub` exists, else `"arena_duel_m0"` (§6.13 item 6). The HUD travel button sends `World.Travel {mapId = "training"}` (to the Staff Room) and back. The return target is `"hub"`, or `"arena_duel_m0"` when the hub is absent, which needs `"arena_duel_m0"` added to the `World.Travel` enum (§6.13 item 6).

**Verify:** `BuildArena.verify()` `{ok = true}` with 36 + 4 + 4 + 4 + 5 + 4 + 4 + 4 + 1 + 8 + 1 = 75 BaseParts. Scenario `map.arena_spawn_8` (8 bots + 1 player spawn; every spawn CFrame faces the centre; no two characters spawn within 4 studs). Scenario `map.arena_no_escape` (scripted uppercut follow at a hedge: the attacker's root Y never exceeds 31 while |x| or |z| > 58). MCP `screen_capture` from `(0, 40, 90)` looking at the centre, for human review of layer 2 readability.

---

### 6.3 Elsewhere-on-Sea graybox `hub`

Elsewhere-on-Sea is the flagship battleground (GDD §11.2 A): a European pier town of the Wishaway Isles, for 20 players. The graybox is playable over 600 × 600 studs (x, z ∈ [−300, 300]) with a soft boundary, 6 zones, 4 spawn hubs with Charm Racks, the tide, and graybox interactables (trampolines, the shallows, breakable beach huts and trapdoors; GDD §11.1, §11.2 A). Display names: `map.hub.name` (Elsewhere-on-Sea) and `map.hub.zone.<ZoneId>`.

**Orientation.** North (−Z) is the sea. The Beach and the shallows run along the north edge; the Pavilion stands in the middle on the pier deck, with Under the Pier below it; the Promenade's beach huts fill the west; Bandstand Gardens fill the south-east; the Jubilee Clock stands in the south-west corner, under the chalk cliff with the funicular. The Haar Line (a painted sea-fog wall) closes every horizon.

**Scope.** Every prop is generic seaside furniture. The town's other landmarks and interactables (the pier deck's breakable railings and ring-outs, the funicular car on its timetable, Pip-Pip's sorting office, Sundry's Rack beside the Pavilion, the Summer Board and the picnic hampers; GDD §11.2 A) are future hooks (§6.15). There is no lighthouse here (it belongs to Gullcliff), and no boardwalk, amusement arcade or fairground ride.

#### 6.3.1 Zones

`ZoneBounds` parts (invisible, `Kind = "Volume"`, no `Vol` attribute) carry `ZoneId`. A point belongs to the first matching box in this order (Under the Pier first, because it lies under other zones):

| Order | `ZoneId` | Name | Boxes: x range, z range, y range | Silhouette / verticality |
|---|---|---|---|---|
| 1 | `under_pier` | Under the Pier | [−190, 70], [−60, 60], [−40, −4] | Low hall of pier piles and rock pools under the Pavilion and the promenade deck |
| 2 | `pavilion` | The Pavilion | [−100, 100], [−100, 100], [−4, 90] | Tiered Pavilion with a cupola; terraces at y 20 / 36 / 50 |
| 3 | `promenade` | The Promenade | [−300, −100], [−300, 100], [−4, 40] | Dense rows of beach huts; hut roofs at y 14 |
| 4 | `beach` | The Beach | [−100, 300], [−300, −100], [−4, 40] | Open sand: the bathing station and the shelter (roofs y 16), two lifeguard towers (y 24), sandcastles; the shallows and the tide (§6.3.8) |
| 5 | `bandstand_gardens` | Bandstand Gardens | [100, 300], [−100, 300] and [−100, 100], [100, 300], y [−4, 40] | Palm canopies (y 20), garden rooms of clipped hedges, the bandstand roof (y 13) |
| 6 | `jubilee_clock` | The Jubilee Clock | [−300, −100], [100, 300], [−4, 40] | Dunes (up to y ≈ 10), sea-worn rocks, the clock's plinth (y 10) and tower |

The six zones are the graybox cut of GDD §11.2 A's Elsewhere-on-Sea. The chalk cliff and the funicular are a decorative silhouette outside the boundary (§6.3.7).

#### 6.3.2 The Pavilion (`Geometry.pavilion`)

| Name | Kind | Shape / size | Centre / ends | Notes |
|---|---|---|---|---|
| `Pavilion_Floor_<k>` | Floor `#F3E9D2` | 4 blocks from `Util.slabRects({−100,100,−100,100}, {{35,45,35,45}})`, thickness 4 | top at y 0 | The pier-deck square; leaves the `Trapdoor_E` hole at (40, 40) |
| `Pavilion_T1` | Wall `#FFF3D6`, `WallSplat` | 64 × 20 × 64 | (0, 10, 0) | Top = terrace 1 (y 20) |
| `Pavilion_T2` | Wall, `WallSplat` | 44 × 16 × 44 | (0, 28, 0) | Top y 36 |
| `Pavilion_T3` | Wall, `WallSplat` | 28 × 14 × 28 | (0, 43, 0) | Top y 50 |
| `Pavilion_Cupola` | Wall | Cylinder Ø10 × 24 | (0, 62, 0) | Landmark, top y 74. The `Pavilion` Model (tiers and cupola) sets `LevelOfDetail = Enum.ModelLevelOfDetail.StreamingMesh` so it reads from afar |
| `Ramp_N`, `Ramp_S` | Floor | width 12, rise 20 | low (0, ·, ∓72) → high (0, ·, ∓32) | Ground to terrace 1 |
| `Ramp_T12` | Floor | width 8, rise 16 | low (27, ·, 14) → high (27, ·, −14), y0 = 20 | On the east ring of terrace 1, beside tier 2 |
| `Ramp_T23` | Floor | width 6, rise 14 | low (−18, ·, −12) → high (−18, ·, 12), y0 = 36 | On the west ring of tier 2 |
| `Tramp_PavilionE`, `Tramp_PavilionW` pads | Floor `#2F5DA8` | Cylinder Ø8 × 1 | (±44, 0.5, 0) | Trampoline mats; volumes in §6.3.6 |
| `Planter_<n>` (8) | Cover `#8FD3C1`, `WallSplat` | 4 × 6 × 12 at x = ±70; 12 × 6 × 4 at z = ±70 | (±70, 3, ±30), (±30, 3, ±70) | Painted flower tubs; line-of-sight breakers |
| `EmptyDeckchair` | Decor `#E8B4A0` | 3 × 3 × 4 | (0, 51.5, 10) | On terrace 3, the top balcony, south of the cupola: one empty deckchair with an open book (GDD §11.2 A; the quiet landmark every map carries) |
| `Arch_N_L/R`, `Arch_N_Lintel` | Wall | 8 × 24 × 8 pillars at (∓16, 12, −100); lintel 40 × 4 × 8 at (0, 26, −100) | | North arch |
| `Arch_S_L/R`, `Arch_S_Lintel` | Wall | mirror at z = +100 | | South arch |
| `Lamp_<n>` (8) | Decor `#3F4A5A` | Cylinder Ø1 × 10 | 6 studs outward from each planter | Cast-iron lamp posts; non-colliding |

#### 6.3.3 Under the Pier (`Geometry.under_pier`)

| Name | Kind | Shape / size | Centre | Notes |
|---|---|---|---|---|
| `UnderPier_Floor` | Floor `#3B4A4F` | 260 × 4 × 120 | (−60, −38, 0) | Top y −36 |
| `UnderPier_Wall_N/S` | Wall `#2E3A3F`, `WallSplat` | 268 × 32 × 4 | (−60, −20, ∓62) | |
| `UnderPier_Wall_W/E` | Wall, `WallSplat` | 4 × 32 × 120 | (−192, −20, 0), (72, −20, 0) | |
| `Prom_Deck_<k>` | Floor (zone `promenade` colour) | `Util.slabRects({−200,−100,−70,70}, {{−155,−145,−5,5}})`, thickness 4, top y 0 | | The promenade deck over Under the Pier, with the `Trapdoor_W` hole |
| `Pile_<n>` (10) | Cover `#5A4A3A`, `WallSplat` | 6 × 32 × 6 | x ∈ {−160, −110, −60, −10, 40}, z = ±25, y −20 | Pier piles |
| `PlankGap_<n>` (6) | Decor `#FFE9A8`, Neon, transparency 0.4 | 8 × 0.2 × 8 | ceiling underside y −4.2 at x ∈ {−150, −100, −50, 0, 40, 60}, z = 0 (40 for x = 40) | Sunlight through the gaps in the planks; Neon stands in for light (GDD §12.4: no dynamic lights) |
| `RockPool_<n>` (3) | Decor `#4E9E8C`, transparency 0.3 | Cylinder Ø12 × 0.2 | (−85, −35.9, 40), (15, −35.9, −40), (−135, −35.9, −40) | Rock pools on the floor; decoration only |
| `PierTramp_W/E` pads | Floor `#2F5DA8` | Cylinder Ø8 × 1 | (−150, −35.5, 0), (40, −35.5, 40) | Trampolines under the trapdoors: the only ways back up |

Trapdoors (`Breakables`):

| Model | Chunks | Centre | Attributes |
|---|---|---|---|
| `Trapdoor_W` | 4 × Block 5 × 1 × 5 | (−150, −0.5, 0) | `Breakable = true`, `BreakHp = Perf.World.Breakable.TrapdoorHp` (1), `BreakFromAir = true`, `RebuildSec = 90` |
| `Trapdoor_E` | 4 × Block 5 × 1 × 5 | (40, −0.5, 40) | same |

A trapdoor breaks when a character is slammed into it (Ender, Downslam spike), lands on it from height (|vy| ≥ 40), or is launched into it from the trampoline below. This makes Under the Pier the GDD's "high-risk shortcut" between the Promenade and the Pavilion.

**Under the Pier lighting overlay.** While the local root is inside the `under_pier` ZoneBounds box, CameraController blends in the `hub.UnderPier` ColorCorrection overlay (§6.5) over 0.5 s.

#### 6.3.4 The Promenade (`Geometry.promenade`, `Breakables`)

**Beach-hut generator** (deterministic; implemented in the builder as data plus one loop):
- Rows `j = 0..11`: `z = −282 + 34·j`. Columns `i = 0..4`: `x = −272 + 34·i + (17 if j is odd else 0)` (staggered, so north–south lanes dog-leg every row).
- Skip a cell when (a) its centre is within 68 studs of hub W `(−270, 0)`; (b) it is within 24 studs of `Trapdoor_W (−150, 0)`; (c) `(i + 2·j) % 5 == 0` (small squares).
- Result: **exactly 40 huts**: `(i,j)` = (1,0) (2,0) (3,0) (4,0) (0,1) (1,1) (2,1) (4,1) (0,2) (2,2) (3,2) (4,2) (0,3) (1,3) (2,3) (3,3) (0,4) (1,4) (3,4) (4,4) (1,5) (2,5) (3,5) (4,5) (0,6) (1,6) (2,6) (4,6) (2,7) (3,7) (4,7) (3,9) (4,9) (2,10) (3,10) (4,10) (0,11) (1,11) (2,11) (4,11).

Each beach hut `Hut_<i>_<j>` is a Breakable Model (`BreakHp = Perf.World.Breakable.HutHp` (2), `BreakFromAir = false`, `RebuildSec = 90`):
- 4 chunks `Q1..Q4`: Block 8 × 14 × 8 at `(x ± 4, 7, z ± 4)`, kind `Breakable`, `WallSplat = true`. Flat roofs at y 14 are layer 2.
- `Stripe`: Decor band 16.4 × 2 × 16.4 at `(x, 9, z)` (0.2 studs proud of each wall), colour cycling coral `#FF7F66` / sea-glass `#8FD3C1` / lemon `#F7E06B` by `(i + j) % 3`: the painted stripe that tells huts apart. It hides with the hut.
- A `Trim` block (16.6 × 0.6 × 16.6 at `(x, 0.3, z)`) sits in `Geometry`, outside the Model, so the foundation border stays when the hut breaks.

Two breaks are needed (the first slam cracks, the second breaks). Wall combos into huts therefore still work at least once (GDD §11.2 A: the Promenade is "good for wall combos" and its huts are breakable).

Other Promenade parts:

| Name | Kind | Size | Centre | Notes |
|---|---|---|---|---|
| `ChairStack_<n>` | Cover `#D9826E`, `WallSplat` | 6 × 6 × 6 | (−204, 3, zs) for each east–west lane centre `zs = row centre + 17` ∈ {−265, −231, −197, −163, −129, −95, −61, −27, 41, 75} | Stacks of folded deckchairs that break the 200-stud east–west lane lines. The lane at z = 7 is skipped (inside the hub clearance) |
| `PromWall_<k>` (3) | Wall `#F3E9D2`, `WallSplat` | 2 × 12 × L | x = −101; z spans [−100, −50], [−38, 52], [64, 100] | The promenade wall between the Promenade and the Pavilion square, with gaps at z ≈ −44 and z ≈ 58. Huts `(4,7)` and `(4,10)` stand behind the gaps, so no straight line runs from the Promenade into the Pavilion square |
| `Tramp_P1`, `Tramp_P2` pads | Floor `#2F5DA8` | Cylinder Ø8 × 1 | (−153, 0.5, −163), (−255, 0.5, −95) | Rooftop access |
| `PromLamp_<n>` (10) | Decor `#3F4A5A` | Cylinder Ø1 × 9 | lane intersections at x = −187, z ∈ {−265, −197, −129, −61, 41} and x = −255, same z | |

#### 6.3.5 The Beach, the shallows and Bandstand Gardens

**The Beach** (`Geometry.beach`; the sea is to the north):

| Name | Kind | Size | Centre / ends | Notes |
|---|---|---|---|---|
| `BathingStation` | Wall `#FFF3D6`, `WallSplat` | 80 × 16 × 24 | (150, 8, −270) | The bathing station at the water's edge; roof y 16 (dry at high tide) |
| `Ramp_BathingStation` | Floor | width 10, rise 16 | low (230, ·, −270) → high (190, ·, −270) | |
| `Shelter` | Wall, `WallSplat` | 24 × 16 × 60 | (260, 8, −180) | The seafront shelter; roof reached by `Tramp_B` |
| `LifeguardTower_1`, `LifeguardTower_2` | Wall `#E06A5A`, `WallSplat` | 10 × 24 × 10 | (40, 12, −150), (120, 12, −230) | Tops y 24. The lifeguard towers where lifeguards call world events (GDD §11.2 A; the calling is a future hook) |
| `Sandcastle_<n>` (8) | Cover `#D9B870`, `WallSplat` | 8 × 8 × 8 | (−60, 4, −250), (0, 4, −200), (70, 4, −180), (100, 4, −280), (180, 4, −130), (200, 4, −210), (250, 4, −120), (280, 4, −260) | Sightline breakers on the open sand. Static in M1-G (they never wash away or rebuild) |
| `Parasol_<n>` (6) | Cover `#FF7F66` | Cylinder Ø2 × 5 | x ∈ {60, 80, …, 160}, z = −250, y 2.5 | Parasol poles at the high-tide line (static props) |
| `PatrolStub` | Decor `#F7E06B` | Cylinder Ø2 × 5 | (100, 2.5, −200) | `PatrolStub = true`; a lifeguard patrol by the Staff is a future hook |
| `Tramp_B`, `Tramp_T1`, `Tramp_T2` pads | Floor `#2F5DA8` | Cylinder Ø8 × 1 | (230, 0.5, −180), (40, 0.5, −132), (120, 0.5, −212) | `Tramp_T2` sits inside the high-tide water and still launches (§6.3.8) |
| `Shallows_LowWater` | Decor `#6FBFB0`, transparency 0.5 | 240 × 0.2 × 24 | (160, 0.1, −268) | The strip that is wet at both tides |
| `Shallows_HighWater` | Decor `#6FBFB0`, transparency 0.5 | 240 × 0.2 × 60 | (160, 0.1, −226) | The band the high tide covers; shown by each client only at high tide (§6.3.8) |

**Bandstand Gardens** (`Geometry.bandstand_gardens`):

| Name | Kind | Size | Centre | Notes |
|---|---|---|---|---|
| `Palm_<n>_Trunk` (11) | Cover `#8B6F4E`, `WallSplat` | Cylinder Ø2 × 14 | y 7 at (130,−70) (190,−60) (250,−80) (130,−20) (220,−20) (130,30) (200,35) (−70,140) (70,140) (−40,200) (40,200) | Cabbage palms |
| `Palm_<n>_Canopy` (11) | Floor `#5E9C5A` (odd n `#7DB36A`) | Cylinder Ø14 × 6 | same x, z; y 17 | Walkable fronds at y 20 (layer 2) |
| `Tramp_G1`, `Tramp_G2` pads | Floor `#2F5DA8` | Cylinder Ø8 × 1 | (160, 0.5, −45), (0, 0.5, 225) | |
| `Hedge_<n>` (15) | Wall `#4F7D4A`, `WallSplat` | height 8, thickness 2 | see the hedge table | Garden rooms of clipped hedges in x [150, 290] × z [100, 260] |
| `Bandstand_Pillar_<n>` (4) | Wall | 2 × 12 × 2 | (±9, 6, 191), (±9, 6, 209) | |
| `Bandstand_Roof` | Floor | 22 × 1 × 22 | (0, 12.5, 200) | The bandstand roof; layer 2 |
| `Bed_<n>` (4) | Cover `#C97B63` | 4 × 6 × 20 | (±50, 3, 180), (±50, 3, 240) | Raised flower beds |

Hedge segments (centre; size):

| # | Centre | Size | # | Centre | Size |
|---|---|---|---|---|---|
| 1 | (180, 4, 100) | 60 × 8 × 2 | 9 | (180, 4, 150) | 2 × 8 × 60 |
| 2 | (260, 4, 100) | 60 × 8 × 2 | 10 | (210, 4, 180) | 60 × 8 × 2 |
| 3 | (200, 4, 260) | 100 × 8 × 2 | 11 | (240, 4, 215) | 2 × 8 × 50 |
| 4 | (280, 4, 260) | 20 × 8 × 2 | 12 | (205, 4, 220) | 50 × 8 × 2 |
| 5 | (150, 4, 140) | 2 × 8 × 80 | 13 | (260, 4, 140) | 2 × 8 × 40 |
| 6 | (150, 4, 230) | 2 × 8 × 60 | 14 | (245, 4, 130) | 30 × 8 × 2 |
| 7 | (290, 4, 180) | 2 × 8 × 160 | 15 | (190, 4, 200) | 40 × 8 × 2 |
| 8 | (215, 4, 245) | 2 × 8 × 30 | | | |

The ways into the garden rooms are the gaps in segments 1/2 (x 210–230), 3/4 (x 250–270) and 5/6 (z 180–200).

#### 6.3.6 Spawn hubs, Charm Racks and volumes

**Hubs.** Four hubs at the edge midpoints (GDD §11.1): `N (0, −270)` on the Beach, `S (0, 270)`, `W (−270, 0)` on the Promenade, `E (270, 0)`. Each hub has:
- a pad `Hub_<id>_Pad`: Spawn-kind visual, 36 × 0.4 × 36 at `(hx, 0.2, hz)`, colour `#43A047`, transparency 0.6, non-colliding;
- 4 spawn Parts `Spawn_<id>_<k>` (4 × 1 × 4) at `(hx ± 8, 0.5, hz ± 8)` with `SpawnFor = "hub"` and `HubId = <id>`, each `CFrame.lookAt(pos, Vector3.new(0, pos.Y, 0) + origin)`;
- a Charm Rack Model `Totem_<id>` at the hub centre.

There are no cover or breakables within 30 studs of a hub centre, and no breakables within 60. The shallows start 40 studs east of hub N, so nobody spawns in the water.

**Charm Rack** (internally a Style Totem; `Totems.Totem_<id>`; also used in the Staff Room). In the fiction it is a souvenir rack of charms; the graybox is a post and one charm:
- `Base`: Totem kind, Cylinder Ø4 × 1 at y 0.5.
- `Pillar`: Totem kind, 2 × 8 × 2 at y 5. Attributes `StyleTotem = true`, `HubId`.
- `Orb`: Decor Ball Ø3 at y 10.5 (the charm). The client cycles its colour through the palettes of the viewer's owned styles at 1 Hz.
- `HB_TotemPrompt` (ProximityPrompt) under `Pillar`: `KeyboardKeyCode = Enum.KeyCode.E`, `GamepadKeyCode = Enum.KeyCode.DPadLeft` (free in Part 3's map), `HoldDuration = 0`, `MaxActivationDistance = Progression.TotemRangeStuds` (12, Part 5 §5.10; read from `Defaults` the same way as `Perf.World`, rule 1 of §6.1.2), `RequiresLineOfSight = false`, `ActionText = ""`, `ObjectText = ""`. The client fills the texts from `Skin.text("ui.totem.action")` / `Skin.text("ui.totem.object")`, because builders never write player-visible literals.
- Model attributes `StyleTotem = true`, `HubId`, `TotemRadius = Progression.TotemRangeStuds` (12). `TotemRadius` exists only to size the prompt (§6.1.4).
- Client: `ProximityPromptService.PromptTriggered` with a prompt named `HB_TotemPrompt` → `UiController.open("StyleWheel", {mode = "totem", hubId = …})` (the Charm Ring, §6.9.6).
- Server (Part 5 §5.1.3 is canonical): StyleService treats an equip as a Charm Rack equip when the distance from the requester's HRP to a Totem **Model pivot** (`Totem_<id>:GetPivot().Position`) of its current map is ≤ `Progression.TotemRangeStuds` (12) and the swap gate passes (`CombatService.canTravel(player)`). A Charm Rack equip applies **now**: the swap respawns the player through Part 3's random safe spawn on the same map (not necessarily that rack's hub; Part 5 DES-3 pending) (§6.13 item 8).

**Volumes** (`Zones.Volumes`; sorted by `Name` when cached):

| Name | `Vol` | AABB (x, y, z ranges) | Attributes |
|---|---|---|---|
| `Tramp_PavilionE`, `Tramp_PavilionW` | Spring | [±40, ±48], [0, 5], [−4, 4] | `LaunchVy = Perf.World.Spring.LaunchVy` (101) |
| `Tramp_P1`, `Tramp_P2`, `Tramp_B`, `Tramp_T1`, `Tramp_T2`, `Tramp_G1`, `Tramp_G2` | Spring | pad centre ± 4 in x/z; y [0, 5] | `LaunchVy = 101` |
| `PierTramp_W`, `PierTramp_E` | Spring | pad centre ± 4; y [−36, −31] | `LaunchVy = Perf.World.Spring.LaunchVyTrapdoor` (130) |
| `Shallows_Low` | Slow | [40, 280], [−1, 4], [−280, −256] | `SlowMult = Perf.World.Slow.Mult` (0.6), `Tide = "low"` |
| `Shallows_High` | Slow | [40, 280], [−1, 4], [−280, −196] | `SlowMult = 0.6`, `Tide = "high"` |
| `Band_N` | Boundary | [−300, 300], [−50, 200], [−300, −280] | `Inward = (0, 0, 1)`, `BandStuds = 20`, `MinMult = 0.2` |
| `Band_S` | Boundary | [−300, 300], [−50, 200], [280, 300] | `Inward = (0, 0, −1)` |
| `Band_W` | Boundary | [−300, −280], [−50, 200], [−300, 300] | `Inward = (1, 0, 0)` |
| `Band_E` | Boundary | [280, 300], [−50, 200], [−300, 300] | `Inward = (−1, 0, 0)` |

`Zones` Model attribute: `TidePeriodSec = Perf.World.Tide.PeriodSec` (240; §6.3.8).

Trampoline speeds (td) come from apex = v² / (2g) with g = 196.2 (Part 3 `Chassis.Gravity`): 101 studs/s gives an apex of ≈ 26 studs (terrace 1, hut roofs, palm fronds, tower tops); 130 studs/s gives ≈ 43 studs (out of Under the Pier through a trapdoor). On sand the root stands at y ≈ 3, inside the shallows' y range; a root on a roof or in the air is never slowed.

#### 6.3.7 Soft boundary, kill volume, the Jubilee Clock corner and stubs

| Name | Kind | Size | Centre | Notes |
|---|---|---|---|---|
| `Boundary_N/S` | Boundary | 620 × 200 × 4 | (0, 100, ∓306) | Invisible, group `MapBoundary` |
| `Boundary_W/E` | Boundary | 4 × 200 × 620 | (∓306, 100, 0) | |
| `HaarLine_N/S/W/E` | Decor `#DDE3E8`, transparency 0.55 | 660 × 60 × 2 / 2 × 60 × 660 | (0, 30, ∓318), (∓318, 30, 0) | The Haar Line: a painted sea-fog wall on every horizon (GDD §11.1 "a painted wall of sea fog"; §11.3) |
| `Sea_N` | Decor `#6FBFB0`, transparency 0.2 | 660 × 0.4 × 12 | (0, 0.3, −312) | The open sea between the north boundary and the Haar Line |
| `Zones.KillVolumes.Kill_Under` | KillVolume | 800 × 10 × 800 | (0, −70, 0) | Map `KillY = −60` |
| `TicketBooth_N`, `TicketBooth_S` (Models) | `Booth`: Cover `#8FD3C1`, `WallSplat`; `Awning`: Decor `#FF7F66` | `Booth` Block 12 × 14 × 12 at y 7; `Awning` Block 16 × 1 × 16 at y 15.5 | (−40, ·, −108), (40, ·, 108) | Ticket booths beside the Pavilion's north and south arches. `GateStub = true` marks them as reserved world-event markers at the Pavilion gates (no GDD §10.4 event uses them yet; `Features.WorldEvent = false`). 2 parts per Model |
| `Zones.EventZones.Gate_N/S` | Event | 30 × 1 × 30 | (0, 0.05, ∓120) | `EventZone = "gate_n" / "gate_s"` (reserved) |
| `Zones.EventZones.GullRaid` | Event | 200 × 1 × 200 | (0, 0.05, 0) | `EventZone = "gull_raid"`: the M1-G world-event stub zone over the Pavilion square, where the Gull Raid's gull lands on the Pavilion dome (the cupola) (GDD §10.4, §11.2 A; off while `Features.WorldEvent = false`; when on, it calls Part 5's `ProgressionService.worldEventParticipation`) |
| `Rock_<n>` (3) | Cover `#9A9486`, `WallSplat` | 10 × 10 × 10 | (−200, 5, 140), (−150, 5, 280), (−270, 5, 210) | Sea-worn rocks among the dunes |
| `Clock_Plinth` | Cover `#E8E4D8`, `WallSplat` | 16 × 10 × 16 | (−130, 5, 250) | Top y 10: a 4-stud walk round the tower. `Ramp_Clock` low (−130, ·, 282) → high (−130, ·, 258), width 8, rise 10 |
| `Clock_Tower` | Wall `#F3E9D2`, `WallSplat` | 8 × 30 × 8 | (−130, 25, 250) | The Jubilee Clock, stopped at closing time; a landmark from y 10 to 40. A `Decor` face `Clock_Face` (Cylinder Ø6 × 0.2, cobalt `#2F5DA8`) sits on its north side at y 34 |
| `Cliff_Main`, `Funicular_Track`, `Funicular_Car`, `Funicular_Top` | Decor | `Cliff_Main` 60 × 80 × 40 at (−340, 40, 340), `#E8E4D8`; `Funicular_Track` 6 × 84 × 2 at (−322, 42, 318), `#5A4A3A`; `Funicular_Car` 8 × 6 × 6 at (−322, 30, 314), `#FF7F66`; `Funicular_Top` 14 × 10 × 14 at (−330, 85, 330), `#FFF3D6` | outside the boundary | The chalk cliff with the funicular railway, seen from the south-west corner; *The Haar Rolls In* begins at its clifftop lookout (GDD §10.4, §11.2 A; a future hook). The car is a static silhouette; its timetable is a future hook (§6.15) |

**Verify (§6.3):** `BuildMap_Hub.verify()` `{ok = true}` with: 16 spawns, 4 Charm Racks (totems), 42 breakables (40 beach huts + 2 trapdoors), 11 trampolines (`Vol = "Spring"`), 2 tide-gated slow volumes (`Shallows_Low` with `Tide = "low"`, `Shallows_High` with `Tide = "high"`), 4 boundary bands, 6 zones, `Zones.TidePeriodSec == Perf.World.Tide.PeriodSec`, the sightline audit clean, per-zone wall-splat and layer-2 checks passing, ≤ 2,500 BaseParts, and `TicketBooth_N/S` each holding exactly `Booth` and `Awning`. Scenarios `map.hub_zone_label` (teleport to one point per zone → the HUD zone label shows `map.hub.zone.<id>`), `map.hub_boundary_soft` (walk outward from (0, 0, 270): WalkSpeed falls to ≤ 3.6 at the wall, the root never goes beyond z = 304), `map.hub_totem_prompt` (prompt within 12 studs of a Charm Rack; triggering it opens the Charm Ring in totem mode), `world.shallows_slow` (§6.1.7), `world.tide_swap` (§6.3.8). MCP `screen_capture` ×3 for human review (H-10): the Pavilion from the south ramp, the Promenade huts at head height, the Beach at high tide from a lifeguard tower. The same three captures are blind-art material (T8, GDD §5.5): the agent appends one `pending` T8 row with subject `map:hub` whose Material cell lists all three captures (one row per subject and preparation run, Part 8 §8.5.3) to `docs/agent/RECOGNIZABILITY.md` and never fills in a result.

#### 6.3.8 The tide

GDD §11.2 A: on Elsewhere-on-Sea the tide moves every 4 minutes. In the graybox the tide is a **volume swap**: two `Slow` volumes cover the shallows on the Beach, and exactly one of them is active at any frame.

| Phase | Active volume | Wet area (AABB x, y, z) | Visual |
|---|---|---|---|
| `low` | `Shallows_Low` (`Tide = "low"`) | [40, 280], [−1, 4], [−280, −256] | `Shallows_LowWater` only |
| `high` | `Shallows_High` (`Tide = "high"`) | [40, 280], [−1, 4], [−280, −196] | `Shallows_LowWater` and `Shallows_HighWater` |

**Schedule** (deterministic, server-owned, never driven by client input):
- The builder writes `TidePeriodSec = Perf.World.Tide.PeriodSec` (240 s, GDD §11.2 A) on the hub's `Zones` Model, so both peers read the same number from the place (§6.1.2 rule 1). A map without the attribute has no tide.
- The phase is a pure function of the simulation frame: `Movement.tidePhase(f, periodF)` with `periodF = TidePeriodSec × 60` returns `"low"` while `floor(f / periodF)` is even and `"high"` while it is odd (§6.1.7). Under Server Authority, `f` comes from `time()`, which is the server's clock, synchronized with every client and rewound with resimulation (Part 3 §3.1.3). The server therefore owns the schedule: swaps fall on the fixed 4-minute boundaries of its clock (the first 240 s after the server starts), and the owning client and the server swap on the same simulation frame. Nothing is replicated at the swap, so nothing can arrive late.
- Inside the sim, the Adapter passes the phase to `Movement.volumeEffect`, which skips the volume tagged with the other phase (§6.1.7).
- Outside the sim, the server world runtime mirrors the phase into the hub Model's `TidePhase` attribute and logs `[HB][World] tide phase=<low|high> f=<f> map=hub` on each change (§6.1.8). The sim never reads that attribute.

**Warning** (render side, client): when `periodF − (fNow % periodF) ≤ Perf.World.Tide.WarnSec × 60` (10 s, td; well above GDD §11.1's 1.5 s telegraph minimum):
- HUD: the tide chip (§6.9.5) switches to the countdown `ui.hud.tide_in {s}` (low to high) or `ui.hud.tide_out {s}` (high to low);
- SFX: `sfx.tide.bell` plays once at the start of the warning, for every client on the hub;
- VFX: `vfx.tide.foam` (a white foam line) slides from the current waterline (z = −256 at low tide, −196 at high tide) to the next one over the warning window, and `Shallows_HighWater` tweens its `LocalTransparencyModifier` from 1 to 0 (coming in) or from 0 to 1 (going out). With Reduced Motion the line does not slide: it appears at the next waterline.

The render side uses `fNow`, the local predicted frame (§6.6.1), so the local character's slow starts exactly when its warning ends.

**Rules.** The tide deals no damage and moves nobody: it only slows grounded characters (`SlowMult` 0.6, §6.1.7). Trampolines, roofs and sandcastle tops inside the high-tide band keep working (an airborne or raised root is never slowed). No hazard attribute is involved (§6.13 item 12).

**Verify:** Lune `movement.tide_phase` (`periodF = 14,400`: f 0 → low, 14,399 → low, 14,400 → high, 28,800 → low; `periodF = 0` → low) and `movement.volume_tide_gating` (§6.1.7). Scenario `world.tide_swap`: the Studio seam `Dev.Command setTidePeriod 20` (§6.13 item 9; it rewrites `Zones.TidePeriodSec` on the server, then waits 1 s for replication); a character stands at (160, ·, −226), inside the high-tide band only. Across two swaps, WalkSpeed reads 18 while `Movement.tidePhase` is low and 10.8 while it is high, on both peers; the HUD shows `ui.hud.tide_in` during the 10 s before the low-to-high swap; the hub's `TidePhase` attribute matches within one scan tick; at the swap frames `RunService:GetPredictionStatus(root)` stays `Predicted` and the Part 7 misprediction counter for `St` stays 0. The scenario ends with `setTidePeriod 240`. Spike S-P6-15 runs the same check under the `M0Gate` net profile.

---

### 6.4 The Staff Room `training`

The Staff Room is the training area (GDD §4.5): the back-of-house room where the Staff practise between shows. The graybox is a 200 × 200 walled room with lino floor, five practice dummies, a splat wall, an air deck, a trampoline and a slow tray. Display name: `Skin.text("map.training.name")` (the Staff Room).

#### 6.4.1 Layout (map-local; origin `(2600, 0, 0)`)

| Name | Kind | Size | Centre | Notes |
|---|---|---|---|---|
| `Floor_<i>_<j>` (100) | Floor | 20 × 4 × 20 | x, z ∈ {−90, −70, …, 90}, y −2 | Lino checks `#E6DCC8` / `#F0E8D8` alternating |
| `Wall_N/S` | Wall `#C8B89A`, `WallSplat` | 208 × 20 × 4 | (0, 10, ∓102) | Tongue-and-groove panelling |
| `Wall_W/E` | Wall, `WallSplat` | 4 × 20 × 200 | (∓102, 10, 0) | |
| `Cap_N/S/W/E` | Boundary | as the walls, 60 tall | y 50 | |
| `SpawnPad` | Spawn (visual) | 24 × 0.2 × 12 | (0, 0.1, 70) | |
| `Spawn_1`, `Spawn_2` | Spawn, `SpawnFor = "training"`, `HubId = "T"` | 4 × 1 × 4 | (±6, 0.5, 70), facing −Z | |
| `Totem_T` | Totem (a Charm Rack, §6.3.6) | | (0, ·, 84) | Opens the Charm Ring in `trial` mode |
| `DummyPoints.Slot_<n>` (5) | Volume (invisible marker) | 2 × 1 × 2 | x ∈ {−40, −20, 0, 20, 40}, y 0.5, z −20, facing +Z | `DummySlot = n`, `Mode = "idle"` (Part 3 §3.15.2) |
| `Placard_<n>` (5) | Decor `#FFFFFF`, transparency 0 | 8 × 0.2 × 3 | (x, 0.1, −12) | Flat floor placards. `SurfaceGui` (`Face = Top`, `SizingMode = PixelsPerStud`, `PixelsPerStud = 40`, `LightInfluence = 0`) showing slot number and mode; the client fills the text |
| `SplatWall` | Wall `#A8957A`, `WallSplat` | 100 × 20 × 4 | (0, 10, −44) | Face at z = −42: 22 studs behind the dummies, inside the 38-stud Ender knockback, reached at about e = 10, well inside the 18f splat window |
| `AirDeck` | Cover, `WallSplat` | 30 × 8 × 20 | (70, 4, 30) | Air-combo practice platform |
| `Ramp_AirDeck` | Floor | width 8, rise 8 | low (70, ·, 58) → high (70, ·, 40) | |
| `Tramp_Test` pad and volume | Floor `#2F5DA8` / Volume `Spring` | Ø8 pad at (−70, 0.5, 30); AABB ± 4, y [0, 5] | | A trampoline; `LaunchVy = 101` |
| `Slow_Test` pad and volume | Floor `#6FBFB0` / Volume `Slow` | 20 × 0.2 × 20 at (−70, 0.1, −60); AABB y [0, 4] | | A paddling tray; `SlowMult = 0.6`; no `Tide` (always active) |
| `EmptyDeckchair` | Decor `#E8B4A0` | 3 × 3 × 4 | (90, 1.5, 90) | The quiet landmark every map carries |
| `Zones.KillVolumes.Kill_Floor` | KillVolume | 220 × 10 × 220 | (0, −50, 0) | Map `KillY = −40` |

Lighting preset `training.Day`. The Staff Room has no tide.

#### 6.4.2 Staff Room runtime and UI

All Staff Room UI lives in `UI/Screens/Hud.luau`, shown only while `Player.MapId == "training"` (no extra files; §0.3).

| Panel | Anchor / position (ref units) | Contents | Actions |
|---|---|---|---|
| `TrainingPanel` | left-middle `(0, 0.5)`, offset (16, 0), 300 × 380 | Nearest dummy slot (by distance to `DummyPoints`, ≤ 40 studs) with its current `Mode`; 5 mode buttons (`idle`, `block`, `perfect_block`, `m1_spam`, `dash_spam`; labels `ui.training.mode_<mode>`); a **Damage numbers** toggle (`ui.training.damage_numbers`); [Try Charms]; [Reset]; [Back to battle] | Mode → `Net.client.fire("Training.Dummy", {behavior = <mode>})` (Part 3 enum). Damage numbers → `SettingsController.set("damageNumbers", not current)`. [Try Charms] → `UiController.open("StyleWheel", {mode = "trial"})`. [Reset] → `World.Travel {mapId = "training"}`. [Back] → `World.Travel {mapId = <battle map>}` (§6.2.2) |
| `FrameDataPanel` | right-middle `(1, 0.5)`, offset (−16, 0), 340 × 320 | Last action of the local player (below) | [Pin] keeps the panel open on Elsewhere-on-Sea, only while in Studio (dev aid) |
| Damage numbers | world BillboardGuis | Damage per contact (HP delta of the victim) | **Off by default** (GDD §15.3: off by default; can be enabled in the Staff Room and in the accessibility settings). Shown when the settings key `damageNumbers` (boolean, default `false`, Part 2 §2.8.6) is true, in any map. Two toggles write that one key through `Settings.Set`: the `TrainingPanel` row `ui.training.damage_numbers` and Settings ▸ Accessibility `ui.settings.damage_numbers` (§6.9.9). RenderController reads it (§6.9.11); `Perf.Ui.DamageNumbersDefault = false` is the fallback before settings load |
| Placards | world SurfaceGuis | `Skin.text("ui.training.slot", {n})` and the slot `Mode` | Updated on the `Mode` attribute change |

**Frame-data readout.** Updated when the local character leaves an `ATTACK` state, and when a contact is observed. Data comes from the same source as `tools/export_framedata.luau` (GDD §18.9): the MoveDef (`Styles.forConfig(cfg).move(id)`) or Part 3's `Combat.M1` tables from the character's pinned config (`Runtime.at(CombatCfg)`).

| Row | Source |
|---|---|
| Name | `Skin.move(moveId)`, or `ui.training.m1_<sub>` / `ui.training.ender_<sub>` / `ui.training.dash_<sub>` |
| Startup / Active / Recovery / Total (f) | MoveDef `startup/active/recovery`; M1 rows from `Combat.M1.Hits[n]`, `Ender`, `Air`, `Downslam`, `DashM1` |
| On hit / on block (f) | `onHitAdv = hitstun − (A − 1) − R`, `onBlockAdv = blockstun − (A − 1) − R` (GDD Appendix B; same formula as Part 4 §4.6.4). For M1 hits the block advantage is `BlockAdv` from config |
| Observed first contact | `HsF − StF`, read on the render frame where the local `Cnt` becomes non-zero (`hit @ e = <n>`). The contact patch sets `HsF = f + hs` and `StF = StF_old + hs` in the same step (Part 3 §3.5.6), so `HsF − StF = f − StF_old` is exactly the contact `e` on any frame during hitstop; `fNow − StF` would read `startup − hs` and must not be used. Should equal `startup` (shown red when it does not) |
| Damage / combo | Victim HP delta for this action; the dummy's `CHits` and the combo total (max HP − HP at combo end) |
| Cooldown | MoveDef `cooldown` (s) |

**Charm trial selector.** The Charm Ring in `trial` mode (§6.9.6) lists every `Styles.implemented()` style with a "Trial" badge (GDD §4.5: a free trial of any Charm in the Staff Room). Selecting one sends `Style.Equip {styleId}` (accepted in the Staff Room when `Progression.TrainingTrialAllStyles` is true, Part 2 §2.7.4 row 4), then `World.Travel {mapId = "training"}` so the trial applies immediately.

**Reset.** `World.Travel {mapId = "training"}` respawns at a Staff Room spawn (HP, cooldowns and Breakaway reset by a new life). Part 3's `canTravel` blocks travel for 8 s after combat (`Chassis.CombatTagSec`). This part asks that travel **from `training` to `training`** skip the combat-tag check, keeping only the STUN/KO check (§6.13 item 6). Dummies reset themselves (Part 3 `Respawn.Dummy`).

**Verify:** `BuildTraining.verify()` `{ok = true}` (100 tiles, 5 dummy points with `DummySlot` 1–5, 2 spawns, 1 Charm Rack). Scenarios `training.dummy_mode_ui` (pressing [Block] near slot 3 → slot 3 `Mode` attribute `"block"` within 0.5 s; placard text updates), `training.framedata_m1` (four M1 presses on an idle dummy → panel rows 6/3/12, 6/3/12, 7/3/13, 10/4/22 and observed contacts at e = 6, 6, 7, 10), `training.trial_equip` (select `sandcastle` → respawn with `EquippedStyle = "sandcastle"` in ≤ 6 s), `training.damage_numbers_toggle` (fresh profile: hitting a dummy shows no damage BillboardGui; the panel toggle → `settings.damageNumbers == true` in the next delta and the next hit shows one; Settings ▸ Accessibility shows the same value), `training.reset_travel` (Reset 2 s after hitting a dummy succeeds once §6.13 item 6 is implemented).

---

### 6.5 Lighting presets and colour scripts as data

GDD §11.4 / §12.4: `LightingStyle = Soft`, boosted ambient, `EnvironmentSpecularScale ≈ 0`, mild shadows, per-map `ColorCorrectionEffect`, mild `BloomEffect` (it only affects Neon "magic" parts because its threshold is high), `Atmosphere` for the painted sea haze. No depth of field, no film grain. Painted skies need uploads, so graybox uses the default `Sky` (agent-assets §2.4).

#### 6.5.1 Ownership and application

- `BuildLighting` (Edit) owns `Lighting` properties plus children `HB_Atmosphere` (Atmosphere), `HB_Sky` (Sky, defaults), `HB_CC` (ColorCorrectionEffect, base), `HB_CC_Fx` (ColorCorrectionEffect, neutral; runtime effects tween it) and `HB_Bloom` (BloomEffect). It also owns `Workspace.Terrain.HB_Clouds` (Clouds must be under Terrain). Each child carries `HbBuilder = "BuildLighting"`. Other Lighting children are left alone; `verify()` warns about them.
- The Edit-time preset is `Perf.Lighting.EditPreset` (`"arena_duel_m0.Day"` until the Elsewhere-on-Sea task switches it to `"hub.Day"`).
- **Runtime (client):** `Lighting` is global, and maps share one place, so CameraController applies the preset for the local player's `MapId` on every `MapId` change. These are local writes. It tweens numeric and colour values over `Perf.Lighting.TweenSec` (1 s on map change). The day/golden/night cycle (GDD §11.4: 45 min) is data-ready but off in M1-G (`CycleEnabled = false`). When on, phase = `floor((GetServerTimeNow() % CycleSec) / (CycleSec / 3))` → Day / Golden / Night, with transitions tweened over 10 s.
- **Zone overlay** (Under the Pier, `hub.UnderPier`) changes only `HB_CC_Fx`.
- **Cosmic Arrival overlay** (`hub.Fog`; Part 5 §5.2.18). Scope: in the design the first Cosmic pull anywhere sets off the Arrival on every server (GDD §8.6); M1-G fires it once per server, and the universe-wide record is a future hook (Part 5 ARR-1). This part renders whatever `CosmicArrivalUntil` says. The client checks `ReplicatedStorage.Runtime:GetAttribute("CosmicArrivalUntil")` against `workspace:GetServerTimeNow()` on the attribute's change and once per second (outside the render step). While the Arrival is on and the local `MapId == "hub"`, CameraController tweens `HB_Atmosphere` and `HB_CC_Fx` to the `hub.Fog` values over `Perf.Lighting.FogTweenSec` (3 s), and back to the map preset over 3 s when the time passes; SfxController plays the ferry-horn sting `sfx.wish.horn` once (on the first observation, so a player who joins mid-Arrival hears it too) and mutes the Music bus (`duck("cosmic", true)`) until the end. Other maps show only the Notify message. The fog never touches parts, attributes, query groups or the hurtbox; telegraphs (always-on-top outlines), hit flashes, nameplates and the Style Tell stay fully visible (the `hub.Fog` density is capped at 0.6, checked in the `cam.arrival_fog` capture).

#### 6.5.2 Presets (`Defaults.Perf.Lighting.Presets`; all `LightingStyle = "Soft"`, `EnvironmentSpecularScale = 0`, `GlobalShadows = true`, `PrioritizeLightingQuality = false`; values td unless cited)

| Preset | ClockTime | Brightness | Ambient | OutdoorAmbient | EnvDiffuse | ShadowSoftness | Exposure | Atmosphere (Density, Offset, Color, Decay, Glare, Haze) | ColorCorrection (Brightness, Contrast, Saturation, Tint) | Bloom (Intensity, Size, Threshold) | Clouds (Cover, Density, Color) |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `hub.Day` | 14 | 2.4 | `#9AA3B8` | `#DCE0EC` | 0.25 | 0.75 | 0.15 | 0.26, 0.15, `#E6F2EE`, `#9FA8D6`, 0, 1.0 | 0.02, 0.06, 0.15, `#FFF8EE` | 0.35, 24, 1.7 | 0.5, 0.5, `#FFFFFF` |
| `hub.Golden` | 17.6 | 1.8 | `#A68E86` | `#F2CBA8` | 0.25 | 0.75 | 0.10 | 0.30, 0.15, `#FFDDBA`, `#C99A9A`, 0, 1.6 | 0.02, 0.06, 0.20, `#FFEEDC` | 0.35, 24, 1.7 | 0.5, 0.55, `#FFE3C8` |
| `hub.Night` | 21.5 | 1.2 | `#5A6290` | `#7C86B8` | 0.2 | 0.8 | 0.25 | 0.32, 0.1, `#8C98C8`, `#3C4670`, 0, 0.8 | 0.03, 0.05, 0.10, `#E2E6FF` | 0.45, 24, 1.6 | 0.4, 0.5, `#AEB6D8` |
| `hub.UnderPier` (overlay, `HB_CC_Fx` only) | — | — | — | — | — | — | — | — | −0.12, 0.10, −0.20, `#C0D8D8` | — | — |
| `hub.Fog` (overlay, `HB_Atmosphere` and `HB_CC_Fx` only) | — | — | — | — | — | — | — | 0.6, 0.1, `#DCE3E6`, `#B8C2C8`, 0, 2.5 | −0.04, −0.05, −0.25, `#EEF2F4` | — | — |
| `arena_duel_m0.Day` | 15 | 2.6 | `#A9B8A0` | `#E2ECD8` | 0.3 | 0.9 | 0.25 | 0.35, 0, `#EEF4E6`, `#C8D8B8`, 0, 0.4 | 0.03, 0, −0.03, `#FFFBF0` | 0.25, 24, 1.8 | 0.3, 0.4, `#FFFFFF` |
| `training.Day` | 13 | 2.0 | `#A49C8C` | `#DCD3C0` | 0.25 | 0.75 | 0.10 | 0.25, 0.1, `#EFE8DA`, `#C8BCA4`, 0, 1.0 | 0, 0.05, 0.05, `#FFF9EE` | 0.3, 24, 1.7 | 0.4, 0.5, `#FFFFFF` |

All properties above were verified on `Lighting`, `Atmosphere`, `ColorCorrectionEffect`, `BloomEffect` and `Clouds` [Docs class YAMLs]. `Lighting.LightingStyle` is script-writable (security None) [Docs `Lighting.yaml`].

#### 6.5.3 Colour scripts

Every map is a different holiday, like a page of an old travel brochure. Each colour script (GDD §11.2, §12.6) sets the dominant, accent and shadow colours that the zone palettes (§6.1.3), the lighting presets (§6.5.2) and later art follow. Rules for every scene: six to eight sun-faded poster flats plus salt and rust; three or four flat background planes with Atmosphere haze; pattern taken from the place (stripes, bunting, pier ironwork, sunbursts); gold appears only where the gilt rule allows it (§6.9.4).

| Map | Dominant | Accent | Shadow | Mood | Status |
|---|---|---|---|---|---|
| Elsewhere-on-Sea (`hub`) | Cream stucco, sea-glass | Coral bunting, lemon | Periwinkle | Breezy, nostalgic | Built (M1-G) |
| The Bandstand (`arena_duel_m0`) | Mown-lawn green, white-painted timber | Coral bunting, sea-glass kiosks | Hedge green | Bright, tidy (td) | Built (M0) |
| The Staff Room (`training`) | Cream lino, warm timber | Sea-glass, cobalt | Umber | Quiet, workmanlike (td) | Built (M0) |
| Gullcliff (`gullcliff`, reserved) | Sand gold, chalk | Marram green, brass | Slate teal | Wild, lonely | Year 1; no builder in M1-G |
| Mount Cocoa (`mountcocoa`, reserved) | Timber, snow cream | Red check, pine | Peach alpenglow | Cosy, crisp | Year 1; no builder in M1-G |
| The Tidings (Season 1; no key yet) | Faded mint, linen | Brass, rose | Dusty lavender | Hushed, spooky-sweet | Season 1 |

**Verify:** `BuildLighting.verify()` (`LightingStyle == Enum.LightingStyle.Soft`, `EnvironmentSpecularScale == 0`, the five owned children plus `Terrain.HB_Clouds` present). Lune `perf.lighting_presets_complete` (every preset has every key; colours match `^#%x%x%x%x%x%x$`; the `hub.Fog` Atmosphere density ≤ 0.6). Scenario `cam.lighting_follows_map` (travel to the Staff Room → `Lighting.ClockTime == 13` on the client within 1.5 s; back → 14). Scenario `cam.arrival_fog` (on the hub, after Part 5's `wish.cosmic_arrival_once` steps fire an Arrival: within 3.5 s `HB_Atmosphere.Density ≥ 0.55` on the client, `sfx.wish.horn` played exactly once and the Music bus volume reads 0; on the Staff Room no fog is applied; after the seam `WishService._resetArrival()` sets the attribute to 0, the preset values return within 3.5 s; one `screen_capture` for human review that a chip at 100 studs stays readable).

---

### 6.6 Presentation architecture

#### 6.6.0 Overview and rules

```
RunService (client)
  BindToSimulation / BindToAnimation   ── Part 3: sim + combat animation (Adapter.animate)
  RenderStepped (BindToRenderStep)
    "HB_Render"   Character+1  RenderController: snapshot diff → transitions → Vfx/Sfx/Camera/Appearance/Hud
    "HB_Camera"   Camera+1     CameraController: shift-lock offset, shake, kick, cinematic
    "HB_Hud"      Last         HudController: bars, cooldowns, meters
```

Rules (all controllers in this section):
1. Never write HRP attributes, constraint properties or Humanoid properties of any character. Presentation writes only client-local instances, `LocalTransparencyModifier`, `Lighting`/`SoundService` locally, and the local `Animator.PreferLodEnabled` override on remote characters (§6.6.1 "Animator LOD").
2. Never use `GetAttributeChangedSignal` for combat one-shots (Part 3 §3.11.8). Poll snapshots once per frame.
3. Guard every render step with `if RunService:IsResimulating() then return end` (the resim replays sim steps only, but the guard is cheap and protects shared handlers).
4. Effects started for **predicted** transitions (the local character's own state, or `Cnt` contact) are undoable within `UNDO_WINDOW = 15` frames (Part 3 `Render.UndoWindowFrames`; a constant, not config).
5. No yields in render handlers (Part 2 §2.10 item 6). Asset loading (`ContentProvider:PreloadAsync`) runs in `task.spawn` at `start()`.
6. All client-created instances live under `Workspace.HB_Local` (Folder, created by RenderController `init()`; sub-folders `Vfx`, `Looks`, `Ragdolls`, `Debris`, `Sfx`). It never exists in Edit (`Verify` reports it as a leftover).
7. Timing comes from **frame data, not animation markers.** Keyframe markers (Part 4 §4.6.1) are kept for artists and the timing-lock check only. The render side computes `e` (remote views: `eR`, §6.6.1) and fires VFX/SFX/faces at the data-defined frames. Part 4 §4.2.0.7 follows the same rule (decision D-03: clip markers exist only for the §4.6.4 timing-lock check); combat animations themselves are played by Part 3's `Adapter.animate` in `BindToAnimation` on both peers, never from RenderStepped and never gated on `IsResimulating()`.

#### 6.6.1 RenderController

```lua
--!strict
-- src/StarterPlayerScripts/Controllers/RenderController.luau — render-side transition dispatcher. Realm: client. Spec §6.6.1.
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Shared = ReplicatedStorage.Shared
local Adapter = require(Shared.Sim.Adapter)
local Frames = require(Shared.Core.Frames)

export type Snap = {
	St: number, StF: number, Sub: number, M1Idx: number, Cnt: number, HsF: number, HP: number, BkP: number,
	BkProg: number, BkLockF: number, AwkF0: number, AwkEndF: number, RlmId: number, Flg: number, CFlags: number,
	CHits: number, Grd: number, ProtF: number, Buffs: string, Debuffs: string, Kit: number,
}
export type Effect = {
	actionId: number,          -- the view's action instance when the effect started (§6.6.1 "Action identity")
	st: number, stF: number, startF: number,
	endF: number?,             -- natural end frame of the effect state (StF + duration), when known
	contactF: number?,         -- contact effects only: the frame of the Cnt 0→1/2 transition
	predicted: boolean, kind: string, stop: (fade: number?) -> (),
}
export type CharView = {
	model: Model, root: BasePart, rid: number, isLocal: boolean, isBot: boolean,
	styleId: string, last: Snap?, cur: Snap?, effects: { Effect }, dist: number, lastHitF: number,
	actionId: number,          -- +1 only on entry (St or Sub changed, or M1Idx for St = M1); never on a hitstop shift
	entryStF: number,          -- StF observed at entry; StF − entryStF = accumulated hitstop shift
	seenF: number,             -- fNow when this action (or reaction refresh) was first observed
	firedMax: number,          -- highest one-shot threshold already fired for actionId (−1 after entry)
	reachedActive: boolean,    -- the action reached its first active frame (telegraph is over)
}
export type RenderControllerApi = {
	init: () -> (), start: () -> (),
	onRollback: (f: number) -> (),               -- Part 3 contract; connected to RunService.Rollback
	views: () -> { CharView },
	viewOf: (model: Model) -> CharView?,
	localView: () -> CharView?,
	focusTarget: () -> CharView?,                -- lock/last-hit target (nameplates, combo counter, full-quality LOD)
	frameNow: () -> number,
	trackEffect: (v: CharView, e: Effect) -> (), -- registers a predicted/undoable effect
}

local UNDO_WINDOW = 15 -- Part 3 §3.18 #14 (constant)
local RenderController = {} :: RenderControllerApi
local fNow = 0
local views: { [Model]: CharView } = {}

local function read(root: BasePart): Snap
	local a = root:GetAttributes()
	return {
		St = a.St or 1, StF = a.StF or 0, Sub = a.Sub or 0, M1Idx = a.M1Idx or 0, Cnt = a.Cnt or 0, HsF = a.HsF or 0,
		HP = a.HP or 100, BkP = a.BkP or 2, BkProg = a.BkProg or 0, BkLockF = a.BkLockF or 0, AwkF0 = a.AwkF0 or 0,
		AwkEndF = a.AwkEndF or 0, RlmId = a.RlmId or 0,
		Flg = a.Flg or 0, CFlags = a.CFlags or 0, CHits = a.CHits or 0, Grd = a.Grd or 100, ProtF = a.ProtF or 0,
		Buffs = a.Buffs or "", Debuffs = a.Debuffs or "", Kit = a.Kit or 0,
	}
end

local function step(_dt: number)
	if RunService:IsResimulating() then
		return
	end
	fNow = Frames.fromTime(time())
	for _, entry in Adapter.list() do -- client roster (local + remote + bots), Part 3 §3.11.1
		local v = views[entry.model]
		if v ~= nil and entry.ready then
			v.last, v.cur = v.cur, read(entry.root)
			-- 1) undo check for predicted effects (rule 4); 2) dispatch transitions (table below)
		end
	end
end

function RenderController.onRollback(f: number)
	-- Contact effects with contactF >= f are undone at once (the rollback rewound past the contact).
	-- Other predicted effects with startF > f are marked unverified; the next step applies the undo rule
	-- (§6.6.1 "Undo rule", Part 3 §3.11.8): stop when the actionId is gone and the new StF <= effect.stF,
	-- or when the view entered a STUN/KO state before effect.endF, regardless of StF ordering.
end

function RenderController.start()
	RunService.Rollback:Connect(function(t: number)
		RenderController.onRollback(Frames.fromTime(t))
	end)
	RunService:BindToRenderStep("HB_Render", Enum.RenderPriority.Character.Value + 1, step)
end

return RenderController
```

**Entry detection** (Part 3 §3.11.8). Entry = `St` or `Sub` changed, or, for `St = M1`, `M1Idx` changed (under an `m1Chain` status M1 hits 4–5 reuse `Sub = 3`, Part 3 §3.5.1): `entered = cur.St ≠ last.St or cur.Sub ≠ last.Sub or (cur.St == S.M1 and cur.M1Idx ≠ last.M1Idx)`. `refreshed = same St, Sub (and M1Idx) and cur.StF > last.StF`. A refresh in an `ATTACK` state is the hitstop shift (Part 3 §3.5.6: the attacker's `StF += hs`) and is **ignored**. A refresh in `Hitstun`/`BlockStun`/`Stagger` is a hit refresh and replays the hit reaction. This avoids treating hitstop as a new action.

**Action identity and one-shot keys.** Each view carries an action instance id `actionId`. It is incremented **only when `entered`**, never by an ATTACK refresh. On increment: `entryStF = cur.StF`, `seenF = fNow`, `firedMax = −1`, `reachedActive = false`. (A hit refresh in a STUN state only resets `seenF`, `entryStF` and the reaction one-shots.)
- Every one-shot (swing, `def.vfx`/`def.sfx`, barrage hits, face changes, the M1 swing at `e = startup`) is keyed by **`(actionId, threshold)`**. It fires on the first render frame where the view's elapsed frame (`e` local, `eR` remote; below) is `≥ threshold`, and only when `threshold > firedMax`; then `firedMax = threshold`. A threshold `≤ firedMax` never fires again for the same `actionId`, so the hitstop shift (which drops `e` back by `hs`) cannot re-fire it.
- The telegraph is drawn only while `cur.Cnt == 0` and `reachedActive == false`. `reachedActive` becomes true on the first frame where elapsed `≥ S` (the move's first active frame) and stays true for that `actionId`, so the telegraph never reappears during or after hitstop.

**Timing per view.** Under Server Authority the client's `time()` is its predicted frame, which runs ahead of the server, while remote characters' attributes arrive about one RTT plus buffering later (Part 3 §3.11.8; creator-docs `server-authority`: other characters "render slightly in the past"). Using `fNow − StF` for them would put the first observation ~9–12 frames into the action at 150 ms, skipping 8–10f telegraphs (GDD §6.14: a tell is shown for ≥ 8f).
- **Local view** (`isLocal == true`; predicted): `e = fNow − StF`.
- **Remote views and bots** (`isLocal == false`): `eR = fNow − seenF − (cur.StF − entryStF)`. `seenF` is the render frame where the new `actionId` was first observed; subtracting the accumulated `StF` shift pauses `eR` during hitstop exactly like `e`. Telegraph windows and one-shot thresholds use `eR`. Other absolute-frame comparisons on a remote view (the `ProtF > fNow` shimmer, `AwkEndF`, `BkLockF` greying) use `fNow − lagR`, where `lagR = (seenF − entryStF)` is the view's observation lag measured at its last entry (0 for the local view).
- **Carriers** (`Workspace.Sim` children): own carriers use `age = fNow − F0` (carrier attribute `F0`, Part 3 §3.11.5); carriers not owned by the local player use `ageR = fNow − seenF(carrier)`, where `seenF(carrier)` is the render frame of the carrier's `ChildAdded` observation. Carrier telegraphs keyed on `age < armF` / `age < flightF` (Part 4 §4.2.0.6) use `ageR` for them.

**Undo rule** (Part 3 §3.11.8 contract; rule 4 of §6.6.0). Within `UNDO_WINDOW` (15f) of its start:
- A **contact** effect (`Cnt` 0→1/2) is undone only if `Cnt` returns to 0 **while `actionId` is unchanged**, or if `RunService.Rollback` fires with a frame `≤ contactF`. A `Cnt` reset caused by entering a new action (an air M1 or Hop Cancel ~6–7f after contact) is **not** an undo.
- Any **predicted** effect is undone when its `actionId` is gone and the new `StF ≤ effect.stF` (rollback), **or** when, within 15f, the view enters a `STUN` or `KO` state before the effect state's natural end (`effect.endF = StF + duration`), regardless of `StF` ordering (the server pre-empted the action with a same-frame hit: `StF = f + hs + 1` is larger).

**Haptics** (GDD §18.4 "`HapticEffect` waveforms for hits on phones and pads"). RenderController `init()` creates one client-local `HapticEffect` `HB_Haptic` under `Workspace.HB_Local` with `Type = Enum.HapticEffectType.GameplayCollision` [Docs `HapticEffect.yaml`, `input/gamepad.md`]. It calls `HB_Haptic:Play()` on the local attacker's `Cnt` 0→1 and on a local-victim `HP` decrease, at most once per 6 frames, only when `UserInputService.GamepadEnabled or UserInputService.TouchEnabled`, and never with Reduced Motion on. It is cosmetic and skipped while `IsResimulating()` like every render handler.

**Animator LOD** (GDD §18.4; Part 3 §3.11.2 contract). The server sets `Animator.PreferLodEnabled = false` on every player and bot character at `Hurtbox.prepare`, so attacker startups (the telegraphs) are never LOD-throttled. Every 0.5 s, from the `HB_Render` step (outside the sim; the property is not Sim-Access), RenderController overrides it locally on each **remote** view's Animator (other players and bots; never the local character): `false` when the view's root is within 60 studs of the current `Camera.CameraSubject`'s position, `true` beyond. It writes only when the value changes, and skips the pass while `IsResimulating()`.

**Transition dispatch** (implements Part 3 §3.11.8; P = predicted for the local character):

| Detect | P | Vfx | Sfx (§6.10) | Camera | Appearance / face | Notes |
|---|---|---|---|---|---|---|
| enter `M1`/`Ender` | yes | `vfx.<style>.m1` swing (Slash_Smear on the right hand) at e = startup | `sfx.<style>.m1` at e = startup | — | `grit` → `shout` at active | |
| enter `Move` (Sub → slot) | yes | telegraph (§6.6.3) from `e = S − leadFrames` to `S − 1`, anchored per Part 4 §4.2.0.6 (with `params.teleFwd`: `teleFwd` studs ahead, along the current facing, of the root position this view saw at Move entry; otherwise the hitbox centre); `def.vfx` at hitbox centre at `e ≥ S` (and at each `S + hitFrames[i] − 1` for barrages); `vfx.dash.puff` for Rush/Mobility | `sfx.tele.windup` at telegraph start if `damage > 10` (GDD §14.4); `def.sfx` at `e ≥ S` | — | `face.windup/active/recovery` from `Styles.anim(style)[slot].face` (Part 4) | One-shots keyed on `(actionId, threshold)`, fired on the first frame where `e ≥ threshold` (remote: `eR`) and `threshold > firedMax`; telegraph only while `Cnt == 0` and not `reachedActive` (see "Action identity") |
| enter `Dash` / `TechRoll` | yes | `vfx.dash.puff` | `sfx.dash.whoosh` | — | — | |
| enter `Block` | yes | `vfx.guard.bubble` (owner + everyone ≤ 40 studs) | `sfx.block.raise` | — | `block` | Bubble alpha follows `Grd` (0.2 → 0.6) |
| `Cnt` 0→1 (local attacker) | yes | `vfx.hit.light` (M1) or `vfx.hit.heavy` (Ender/move) at hitbox centre; impact frame if `def.signature` (§6.6.6) | `sfx.hit.light`/`heavy`, pitch rising with the local combo count | `light` (M1) / `medium` (Ender/move); hitstop kick (§6.6.6) | — | Undo: `Cnt` back to 0 within 15f **with `actionId` unchanged**, or `Rollback` to a frame ≤ `contactF` → fade (a `Cnt` reset by a new action is not an undo). Haptic pulse (see "Haptics") |
| `Cnt` 0→2 (blocked) | yes | `vfx.hit.block` | `sfx.hit.block` | `light` | — | |
| victim `HP` decrease | no | Highlight hit flash (3f, §6.6.3); damage number when the `damageNumbers` setting is true (§6.4.2, §6.9.11); local victim: haptic pulse | local victim: `sfx.vo.hurt` (throttled as a bark) | local victim: `light` if < 8 damage else `medium` | `hurt_light` / `hurt_heavy` | Combo counter update if this view is the focus target |
| enter `Hitstun`/`Stagger`/`Launched`/`Grabbed` | no | — | — | — | `hurt_*` | |
| `Flg` gains `PBHIT` (defender) | no | `vfx.pb.star` + lemon outline flash (6f) | `sfx.block.perfect` (top priority after telegraphs, GDD §14.6) | `light` both | `perfect_block` | |
| enter `GuardBroken` | no | `vfx.guard.break` | `sfx.guard.break` | `medium` if local | `dizzy` | |
| enter `WallSplat` | no | `vfx.wallsplat.splat` at the wall contact (root + facing × −1.1) | `sfx.wallsplat.splat` | `medium` if local attacker/victim | `hurt_heavy` | |
| enter `Ragdoll` | no | — | `sfx.ragdoll.land` on first landing (`Sub` 0→n) | — | — | Ragdoll presenter `begin` at e ≥ 12 (§6.7) |
| leave `Ragdoll` → `Idle` | no | — | — | — | — | Presenter `finish(blend)` (wake-up, §6.7.1) |
| leave `Ragdoll` → any other state except `KO`; or Ragdoll re-entry (`Sub` reset to 0, or `StF` increased) | no | — | — | — | — | Presenter `finish(instant)`, then the normal `begin` rule for a new Ragdoll (§6.7.1) |
| enter `Breakaway` | yes | `vfx.breakaway.burst` | `sfx.breakaway.burst` | — | `determined` | Presenter `finish(instant)` |
| enter `Awakening` (Peak) | yes | `vfx.awaken.burst` (palette) | `sfx.awaken.sting` + bark `ui.vo.awaken` ("Go Peak!") | local: 1.2 s cinematic (GDD §6.9) + `heavy`; others ≤ 30 studs: `medium` | `awaken`; AppearanceController shows the **Peak uniform** (§6.8.2) | The uniform stays while `AwkF0 > 0` (60 s, GDD §6.9). Nobody grows: no scale change anywhere |
| `AwkF0` > 0 → 0 | no | power-down puff | — | — | `neutral`; the Peak uniform goes | |
| enter `KO` | no | `vfx.ko.poof` at `BodyHideSec` (1.2 s, Part 3): a puff of sand and a waving hankie; hide the look | `sfx.ko.poof` + bark `ui.vo.ko` ("Checked out!") | — | `ko` | The "Checked out!" call-out (`ui.hud.checked_out`, a BillboardGui above the body for 1.5 s, viewers ≤ 60 studs). Local KO opens the Charm Ring (§6.9.6) |
| `ProtF > fNow` | no | `vfx.spawn.shimmer` while true | `sfx.spawn.ready` when it ends | — | — | Part 3 §3.13.5 |
| `Buffs`/`Debuffs` change | no | status visuals (Part 4 §4.7.6): `vfx.status.root`, `.slow`, `.silence` and `.dust` (`VFX_Status_Pips`; Dust shows n pips) and `vfx.status.ward` (`VFX_Guard_Bubble`, Do Not Disturb) | — | — | — | Decoded with `Moves.statusDecode`. The `freeze` and stealth renders are deferred with their first user (Part 4 §4.7.6) |
| Realm marker `Realm_<id>` appears / `Dead` (a Getaway) | no | `vfx.realm.frame`: the giant "Greetings from…" postcard frame slams down round the ring, captioned with `Skin.move(moveId)`; `vfx.realm.ring`; the kit's own Getaway visuals (Part 4 §4.7); 12f time dilation (`HB_CC_Fx` saturation −0.4 and VFX `TimeScale` 0.2) | `sfx.realm.open` + the caster's bark `ui.vo.fully_booked` ("Fully booked!", GDD §14.5); Music duck −8 dB while the local player is a member (GDD §14.6) | `heavy` if a member | — | Members see the banner `ui.hud.fully_booked` ("Fully booked!", §6.9.5). Reduced Motion: no colour-correction pulse and no slam (the frame fades in). **Getaway tells** (`RuleId = "dustsheet.a4"`; Part 4 §4.4.3): from `Styles.move("dustsheet.a4").params.getaway` and the marker's `OpenF`, with `k = fNow − OpenF`: `sfx.dustsheet.getaway_bell` 30 f before each Quiet Hours pulse, the dark quarter `vfx.dustsheet.getaway_dark` (`VFX_Getaway_Quarter`) from each Lights Out pulse until the next, and the flicker on the next quarter for the 60 f before it. Telegraphs and hit flashes are never hidden |
| `Workspace.Sim` carrier added / `Dead` | own: yes | attach `vfx.<style>.<slot>` (or a `model.*` summon) by `MoveId`; hide at once on `Dead` | `def.sfx` | — | — | Part 3 §3.11.5 |
| Player `Streak ≥ 3` | no | `vfx.streak.aura` (coral, low rate) | — | — | — | Part 3 §3.13.3 |

**Focus target** (GDD §15.3 "current lock/last-hit target"): the view whose `HP` or `CHits` changed within 6 frames after the local attacker's `Cnt` 0→1, picked as the nearest such view in front of the attacker (dot > 0) within 15 studs. It is held for `Perf.Ui.FocusHoldSec` (4 s) after the last contact. HUD, nameplates and VFX LOD use it.

**Verify:** Lune lint `render.no_attr_signal` (on `RenderController.luau`: no `GetAttributeChangedSignal(` text; a Lune check, not a Studio scenario); scenarios `render.undo_spark` (Part 3), `render.hitstop_not_reentry` (local M1 hits a dummy → exactly one swing effect per M1 state despite the `StF` shift), `render.no_double_move_vfx` (`attendant.3` hits a dummy → exactly one `vfx.attendant.3` and one `sfx.attendant.3` play, and no telegraph is drawn after the contact frame), `render.remote_telegraph_full` (under the Part 7 `M0Gate` net profile, ≈ 150 ms RTT: a bot uses `attendant.4`, Take a Seat!, which has an Arc telegraph; the observer client draws the Arc telegraph for `leadFrames ± 1` render frames, at least 8 (GDD §6.14), before the move's VFX), `render.focus_target` (hit a dummy → `focusTarget()` is that dummy for 4 s), `render.haptic_gate` (a test seam counts `HB_Haptic:Play()` calls: gamepad emulation + one M1 contact → 1; with `reducedMotion = true` → 0; KBM-only → 0).

#### 6.6.2 AnimController

Part 3 plays every combat track from `Adapter.animate` (`BindToAnimation`, both peers for simulated entries; server-played tracks replicate to observers). AnimController handles everything else:
1. **Studio clip registration check.** `Main.client` already calls `Assets.registerStudioClips(ReplicatedStorage.Anim)` (Part 2 §2.9.1). AnimController's `start()` logs `[HB][Anim] clips n=<count> studio=<bool>` and warns when the count is 0 in Studio.
2. **Cosmetic tracks** (render-only, never gameplay): `anim.common.getup` after a ragdoll, Scrapbook / Settings preview rigs, Wish reveal poses, the Staff Room mirror.
3. **The track budget.** Playing tracks per Animator ≤ `Perf.Anim.MaxPlayingTracks` (8, pending Part 3 spike SA-05; GDD §18.2's figure is unverified [api-sheet §1 #17]). A cosmetic play that would exceed it is skipped, with one `[HB][Anim] WARN track_budget` per model.

```lua
--!strict
-- AnimController.trackFor — never caches tracks (Part 3 §3.11.8); caches Animation instances per key.
local Assets = require(game:GetService("ReplicatedStorage").Shared.Assets)
local animByKey: { [string]: Animation } = {}

local function trackFor(animator: Animator, key: string): AnimationTrack?
	local id = Assets.get(key)
	if id == "" then
		return nil -- unpublished placeholder outside Studio: play nothing (timing never depends on clips)
	end
	local existing = animator:GetTrackByAnimationId(id)
	if existing then
		return existing
	end
	local anim = animByKey[key]
	if anim == nil then
		local a = Instance.new("Animation")
		a.AnimationId = id
		animByKey[key] = a
		anim = a
	end
	return animator:LoadAnimation(anim :: Animation) -- loaded once per Animator; later calls hit GetTrackByAnimationId
end
```

API: `play(model, key, {fade, speed, priority, looped}?) -> AnimationTrack?`, `stop(model, key, fade?)`, `trackFor(animator, key)`, `playingCount(model) -> number` (`#animator:GetPlayingAnimationTracks()`), `lengthFrames(key) -> number?` (the `Frames` attribute of `ReplicatedStorage.Anim.<Name>`, Part 4 §4.6.1, never `AnimationTrack.Length`).

**Verify:** scenario `anim.cosmetic_budget` (force 9 cosmetic plays on one rig → the 9th is skipped with one WARN line); scenario `anim.no_track_cache` (Lune lint: no table assignment of `LoadAnimation(` results in `src/`).

#### 6.6.3 VfxController

**Templates** come from `ReplicatedStorage.VFX` (built by `BuildVfxTemplates`, §6.6.7). They are cloned into per-key pools under `Workspace.HB_Local.Vfx`.

```lua
--!strict
export type Layer = "fx" | "telegraph"
export type Tint = { core: Color3, mid: Color3, ink: Color3 }
export type VfxSpec = {
	key: string,                          -- Assets key, e.g. "vfx.hit.light" → "tpl:VFX/VFX_HitSpark_Light"
	at: CFrame?, attach: BasePart?,       -- world placement (stays anchored), or weld to a part (auras): un-anchored first (below)
	owner: Model?,                        -- whose effect (LOD, tint, Finish, Reduced VFX opacity)
	tint: Tint?, flair: string?, scale: number?, lifeSec: number?,
	predicted: boolean?,                  -- registers with RenderController.trackEffect
}
export type VfxHandle = { stop: (fade: number?) -> (), alive: () -> boolean }
export type TelegraphSpec = {
	shape: "Circle" | "Line" | "Cone" | "Arc", size: Vector3, cf: CFrame, owner: Model?, predicted: boolean?,
}
export type VfxControllerApi = {
	play: (spec: VfxSpec) -> VfxHandle?,                    -- nil when skipped by budget/LOD
	telegraph: (spec: TelegraphSpec) -> VfxHandle,          -- never budgeted or reduced
	flash: (model: Model, kind: "hit" | "pb" | "crack" | "lock" | "impact" | "pulse", frames: number) -> (),
	freeze: (model: Model, frozen: boolean) -> (),          -- hitstop: TimeScale 0.05 on effects attached to model
	debris: (breakable: Model) -> (),
	setTier: (tier: "Low" | "Mid" | "High") -> (),
	stats: () -> { emitters: number, particles: number, beamsTrails: number, highlights: number, level: number },
}
```

**Attached effects (`spec.attach`).** Every VFX template part is built anchored (§6.6.7), and welding an anchored part into a character makes it the assembly root and implicitly anchors the whole character [Docs `physics/assemblies.md`: "When one of an assembly's parts is anchored … all of the other parts become implicitly anchored with it"]. The predicted local character would then stop moving client-side and rubber-band. So when `spec.attach` is set, VfxController takes the pooled clone and, **before** parenting or welding it, sets on every `BasePart` descendant: `Anchored = false`, `Massless = true`, `CanCollide = false`, `CanQuery = false`, `CanTouch = false`, `CollisionGroup = "CosmeticNoQuery"`. It then positions the root at `spec.attach.CFrame` and welds it with one `WeldConstraint` (`Part0 = spec.attach`, `Part1 = <template root>`) parented under the clone. World-placed effects (`spec.at`) stay anchored. A pooled clone returned to the pool keeps its un-anchored flags and is re-anchored only if it is reused with `at` (pools are keyed per key and placement mode). This rule covers `vfx.streak.aura`, `vfx.spawn.shimmer`, status visuals, the Peak uniform's gilt trim (§6.8.2) and the Style Tell ring (§6.8.4).

**Perf sample sender** (GDD §20.4 `perf_sample`; Part 5 §5.8.3; Part 2 §2.7.4 row 18). While `Players.LocalPlayer:GetAttribute("PerfSampled") == true` (written by Telemetry at join; Part 2 §2.5.2), VfxController records `RenderStepped` dt into a ring covering `Runtime.get().Telemetry.PerfWindowSec` (120 s). Once per `Telemetry.PerfSendEverySec` (300 s), the first send only after the first full window, it computes fps P50 and P5 by nearest rank over `1/dt`, reads `Stats:GetTotalMemoryUsageMb()` [Docs `Stats.yaml`], and fires `Net.client.fire("Telemetry.Perf", {deviceTier = <current tier>, fpsP50 = …, fpsP5 = …, memoryMb = …, vfxDegradeLevel = <degrade level 0–5>})` (`TelPerfSample`). The work runs in the existing render step (no extra binding, no yields). When the attribute is false or absent, nothing is recorded or sent.

**Palette and Finish.** The tint is `core = palette.accent`, `mid = palette.primary`, `ink = Perf.Vfx.Ink` (`#241C33`, the warm navy-plum ink; never black) (Part 4 §4.7.1; GDD §13.4 three tones plus an ink rim). Emitters get `ColorSequence(core → mid)`; ink parts keep ink. The Finish overlay (the `Flair` field; GDD §8.7) is applied to the owner's effects only and never changes shape (GDD §8.7, §13.4):

| Finish (`Flair` ID) | Overlay |
|---|---|
| Plain (`plain`) | none |
| Glossy (`glossy`) | extra sparkle emitter (default sparkles texture, Rate 4) + brighter outline: a fresh coat of gloss paint |
| Gilt (`gilt`) | tint lerp 50 % toward the gilt colour `Theme.Gilt` (`#D9A93A`); ink → `#6E4E12` |
| Sea-Glass (`seaglass`) | tint lerp 40 % toward `#A8E6CF`; +0.15 transparency (frosted, sea-worn) |
| Pearl (`pearl`) | pearly sheen: the tint's hue cycles at 0.5 Hz at low saturation (`Color3.fromHSV(h, 0.25, 1)`) |
| Glowtide (`glowtide`) | core `#4FFFE0` (glowing surf), mid `#0B2545` (night sea) |
| Misprint (`misprint`) | the colour plates are out of register: core alternates `#00B8D9`/`#E5007E` at 6 Hz; with `misprintCalm` (GDD §16.3) static `#00B8D9`. Never faster than 3 full-screen flashes per second (it is not full-screen anyway) |

**Budgets** (`Perf.Tiers[tier]`; GDD §13.5). Every template root has attributes `Emitters`, `ParticlesEst` (max simultaneous particles: Rate × Lifetime.Max + Emit bursts), `BeamsTrails`, `Layer`, `Lod1` (variant name or `""`). The controller keeps running totals of active `fx`-layer templates:
1. When `play` would exceed `MaxEmitters`, `MaxParticles` or `MaxBeamsTrails` for the tier (× `ReducedVfxBudgetMult` 0.4 when Reduced VFX is on, GDD §16.3): effects of the local player and the focus target still play; others play their `Lod1` variant if it fits, else are skipped.
2. **LOD by distance** (GDD §13.5): the local player's and focus target's effects are always full quality. Others beyond `LodSimplifiedStuds` (80) use `Lod1`; beyond `LodTelegraphOnlyStuds` (160) only telegraphs render.
3. **Degrade scheduler:** a frame-time EMA (α = 0.1) compared with `1 / TargetFps`. Above `DegradeFrameTimeHigh` (×1.15) for `DegradeUpAfterSec` (2 s) → level + 1 (max 5); below ×0.85 for 5 s → level − 1. Level `n` multiplies every emitter `Rate` and `Emit` count by `DegradeRateMult[n + 1]` = {1, 0.8, 0.6, 0.45, 0.3, 0.2}. It lowers emitter rates first and **never** touches telegraphs (GDD §13.5). The level is reported in `Telemetry.Perf` (`vfxDegradeLevel`, Part 2 §2.7.4 row 18).
4. Reduced VFX also sets enemy-owned effects to 60 % opacity (`ReducedVfxEnemyOpacity`, GDD §16.3): transparency `t → 1 − 0.6·(1 − t)`.
5. **Per-move emitter caps** by rarity (GDD §13.5: Starter–Common 3, Uncommon–Rare 5, Epic 6, Legendary 7, Mythic–Cosmic 8) are enforced at build time by `BuildVfxTemplates.verify()` and asserted in Studio at `play`.

**Telegraph layer** (GDD §13.4, §16.3). `telegraph()` builds from `VFX_Telegraph_<Shape>`:
- a flat Neon fill (thickness 0.05, `Transparency = TelegraphFillTransparency` 0.35) in the user's `telegraphColor`;
- an outline copy `TelegraphOutlineStuds` (0.4) larger underneath. Its colour is chosen at spawn: raycast down from the telegraph centre (`WorldQuery`), take the ground colour (`RaycastResult.Instance.Color`, or `Terrain:GetMaterialColor(result.Material)` for Terrain), then pick ink `#241C33` if `Theme.contrast(ink, ground) ≥ 4.5`, else `#FFFFFF`;
- an **always-on-top overlay**: a `SurfaceGui` (`Face = Top`, `AlwaysOnTop = true`, `LightInfluence = 0`) on the fill part, holding the shape's outline drawn with Frames (`UIStroke` 3 px, outline colour). Circle: one frame with `UICorner` 0.5. Line: a rectangle. Cone and Arc: the two edges plus the far arc as 8 rotated thin frames. This keeps the hit area's edge visible above particles and beams (spike S-P6-4).

Telegraphs are never pooled out, never LOD'd, never reduced, never tinted by a Finish. Telegraph colour swatches and colour-blind variants are in §6.9.4.

**Highlight pool** (GDD §12.4: gameplay feedback only, pooled, cap 12). The docs allow 255 per client and count disabled ones [Docs `Highlight.yaml`]; 12 is the GDD's design cap. The pool creates 12 Highlights at `start()` under `HB_Local.Vfx` and toggles `Adornee`/`Enabled` (no add/remove hitches). Priority when full: `impact` > `pb` > `crack` > `hit` > `lock` (evict the lowest-priority, oldest one).

| Kind | Fill / outline | Frames |
|---|---|---|
| `hit` | Fill `#FFFFFF` at 0.5, outline off | 3 |
| `pb` | Fill off, outline `#F7E06B` (lemon) | 6 |
| `crack` | Fill off, outline lerps `#FFFFFF` → `#E53935` as `Grd` falls from 40 to 0 | while `Grd < 40` and blocking |
| `lock` | Fill off, outline `Theme.Color.Focus` at 0.5 | while focus target |
| `impact` | Fill ink `#241C33` at 0, outline `#FFFFFF`, `DepthMode = Occluded` | 2 (§6.6.6) |
| `pulse` | Fill off, outline `#FFFFFF`, outline transparency 0 → 1 | 15 (Reduced Motion replacement) |

**Verify:** scenario `vfx.budget_tiers` (spawn 30 hit sparks per frame from bots for 3 s at tier Low → `stats().emitters ≤ 120` and `particles ≤ 1500`; with Reduced VFX ≤ 48 / 600); `vfx.telegraph_never_reduced` (Reduced VFX + degrade level 5 → a move telegraph still renders with fill transparency 0.35); `vfx.highlight_cap` (20 simultaneous hits → never more than 12 Highlights with `Enabled = true`); `vfx.lod_distance` (an enemy effect at 120 studs uses `Lod1`; at 170 studs only its telegraph exists); `vfx.attach_unanchored` (attach `vfx.streak.aura` and the Style Tell ring to the local character → every attached `BasePart` has `Anchored == false` and `Massless == true`, `HRP.AssemblyRootPart == HRP`, `HRP.Anchored == false`, and a 2 s scripted walk moves the root ≥ 10 studs with `RunService:GetPredictionStatus(root)` still `Predicted`); `tel.perf_sample_sent` (Part 5 §5.8.3: seam `Telemetry._setPerfSampled(P, true)` with `setConfig Telemetry {"PerfWindowSec":5,"PerfSendEverySec":10}` → a `perf_sample` record within 15 s); Lune `theme.contrast` (ink `#241C33` vs the Pavilion paving `#F3E9D2` ≥ 4.5; ink vs the Under the Pier floor `#3B4A4F` < 4.5 → white chosen).

#### 6.6.4 SfxController

- **Buses:** `SoundService.Music`, `Sfx`, `Vo`, `Ui` (SoundGroups, client-created at `init()`; Part 2 §2.2). `Volume = settings.vol<Bus> × settings.volMaster`.
- **Pool:** `Perf.Audio.PoolSize` (32) `Sound` instances. Spatial sounds are parented to pooled `Attachment`s on one anchored, invisible part `HB_Local.Sfx.Anchor`; 2D sounds are parented to `SoundService`. `Sound.SoundId` takes the manifest ID (`rbxasset://sounds/<file>` placeholders, §0.5). `RollOffMaxDistance = 80`, `RollOffMinDistance = 8` (td).
- **Voice cap** `Perf.Tiers[tier].MaxSounds` (16 / 24 / 32). When full, the lowest-priority, oldest sound is stopped. Priorities follow GDD §14.6: 1 telegraph cue, 2 Perfect Block / Guard Break, 3 hits on or from you, 4 your VO, 5 nearby hits, 6 music, 7 ambience.
- **Speed:** `Styles.SFX_SPEED[key]` for per-move keys and `sfx.dustsheet.getaway_bell` (Part 4 §4.7.3), else `Perf.Audio.Map[key].speed`, times a uniform jitter of ±`PitchJitter` (5 %, td) from a module `Random.new()` (presentation randomness is allowed; it never reaches the sim), times the combo rise `1 + ComboPitchStep × min(hits − 1, ComboPitchMaxSteps)` (0.03 × ≤ 10, td; GDD §6.13 "rising pitch on combo counters").
- **Barks** (GDD §14.5): ≤ 1 per `BarkMinIntervalSec` (4 s) per player, only within `BarkRangeStuds` (40) of the listener, own barks only when `quietBarks` is on (§6.13 item 10), always subtitled when `subtitles` is on (§6.9.10). Placeholder voice is `AudioTextToSpeech` wired to an `AudioDeviceOutput` with a `Wire` (`SourceInstance`/`TargetInstance`) [Docs `audio/objects.md`], using `Skin.text("ui.vo.<line>")`. TTS is throttled by the engine (1 + 6 × CCU per minute per experience; 4 per second per client [Docs `AudioTextToSpeech.yaml`]); a refused TTS falls back to the subtitle only. TTS runs only in Studio (`Perf.Audio.TtsInStudioOnly = true`). Part 8's release lint blocks it anyway.
- **Ducking** (GDD §14.6): Music bus −4 dB (×0.631) while the local player's combo `CHits ≥ 3` on the focus target; −8 dB (×0.398) while a member of an active Getaway; Cosmic Wish reveal: Music muted (GDD §14.4); Cosmic Arrival on the hub: Music muted until it ends (§6.5.1). Tweened over 0.2 s.
- **Music direction** (GDD §14): accordion and melodica over a seaside brass band, a different genre per island (a music-hall waltz for Elsewhere-on-Sea). M1-G ships no tracks; the bus, the ducking and the hub's horn sting are ready for stems.

API: `play(key, {at, owner, speed, volume, prio, comboStep}?) -> Sound?`, `bark(owner: Player | Model, lineKey: string) -> boolean`, `duck(reason: "combo" | "realm" | "cosmic", on: boolean)`, `applySettings(s: Types.Settings)`.

**Verify:** scenario `sfx.voice_cap` (40 hit sounds in one frame at tier Low → ≤ 16 playing; the PB ding survives); `sfx.bark_throttle` (two barks 1 s apart from one player → the second returns false); `sfx.bus_volumes` (settings volMusic 0.5 → `SoundService.Music.Volume == 0.5 × volMaster`).

#### 6.6.5 CameraController

**Shift-lock** (GDD §15.2: "Mouse (Shift-Lock default ON)"). The default PlayerModule mouse lock is disabled: `Player.DevEnableMouseLock = false` is set **by the server** at join (it cannot be set client-side [Docs `Player.yaml`]; §6.13 item 6). CameraController implements its own:
- State is owned by InputController (`setShiftLock(on)` / `shiftLock()`, §6.13 item 7), so InputController can synthesize `AimLock` (Part 3 §3.10.3) without requiring a later controller. The default is `Perf.Camera.ShiftLockDefault` (true). The `ShiftLock` UI action (LeftShift) and the touch `ShiftLockBtn` toggle it.
- `BindToRenderStep("HB_Camera", Enum.RenderPriority.Camera.Value + 1, fn)`. When on and `UserInputService.PreferredInput == Enum.PreferredInput.KeyboardAndMouse`: `UserInputService.MouseBehavior = Enum.MouseBehavior.LockCenter`. On any input: `camera.CFrame = camera.CFrame * CFrame.new(ShiftLockOffset)` ({1.75, 0, 0}, td). Character facing comes from the sim (`AimLock` + `Aim`, Part 3), never from the camera script. Whether the PlayerModule overrides `MouseBehavior` after Camera priority is spike S-P6-8.

**Shake** wraps the vendored Sleitnick `Shake` 1.1.0 (`Shared/Vendor/Shake`; API verified in the cached repo: `Shake.new()`, fields `Amplitude`, `Frequency`, `FadeInTime`, `FadeOutTime`, `SustainTime`, `Sustain`, `PositionInfluence`, `RotationInfluence`, `TimeFunction`; methods `Start`, `Stop`, `Update() -> (Vector3, Vector3, boolean)`).

```lua
--!strict
-- CameraController shake (excerpt). Spec §6.6.5.
local Shake = require(game:GetService("ReplicatedStorage").Shared.Vendor.Shake)
type Preset = { Amplitude: number, Frequency: number, FadeInTime: number, SustainTime: number, FadeOutTime: number,
	Pos: { number }, Rot: { number } }
local active: { any } = {}

local function shake(preset: Preset, scale: number)
	if scale <= 0 then
		return -- Reduced Motion or cameraShake = 0 (GDD §16.3)
	end
	local s = Shake.new()
	s.Amplitude = preset.Amplitude * scale
	s.Frequency = preset.Frequency
	s.FadeInTime, s.SustainTime, s.FadeOutTime = preset.FadeInTime, preset.SustainTime, preset.FadeOutTime
	s.PositionInfluence = Vector3.new(preset.Pos[1], preset.Pos[2], preset.Pos[3])
	s.RotationInfluence = Vector3.new(preset.Rot[1], preset.Rot[2], preset.Rot[3])
	s.TimeFunction = os.clock -- smooth at render rate; non-outcome timing (§0.4)
	s:Start()
	table.insert(active, s)
end

-- Called from the HB_Camera render step after the shift-lock offset:
local function applyShakes(cam: Camera)
	for i = #active, 1, -1 do
		local pos, rot, done = active[i]:Update()
		cam.CFrame *= CFrame.new(pos) * CFrame.Angles(rot.X, rot.Y, rot.Z)
		if done then
			table.remove(active, i)
		end
	end
end
```

- `scale = settings.cameraShake × (reducedMotion and 0 or 1)`. Events not involving the local player use linear falloff to 0 at 30 studs (td).
- Presets (`Perf.Camera.Shake`, td, GDD §6.13 light/medium/heavy):

| Preset | Used for | Amplitude | Frequency | FadeIn / Sustain / FadeOut (s) | Pos influence | Rot influence (rad) |
|---|---|---|---|---|---|---|
| `light` | M1 contact, blocked hit, light hit taken | 1 | 0.06 | 0 / 0.03 / 0.12 | {0.3, 0.3, 0} | {0.01, 0.01, 0.012} |
| `medium` | Ender/move contact, wall splat, guard break, heavy hit taken | 1 | 0.07 | 0 / 0.06 / 0.22 | {0.7, 0.7, 0} | {0.02, 0.02, 0.03} |
| `heavy` | Peak ("Go Peak!"), a Getaway opening | 1 | 0.08 | 0.04 / 0.25 / 0.45 | {1.4, 1.4, 0.4} | {0.04, 0.04, 0.06} |

**Other camera duties.**
- `kick(frames)`: hitstop FOV punch `−Perf.Camera.HitstopFovKick` (2°) for `frames` frames, then restore. Skipped with Reduced Motion.
- `cinematic("awaken")`: the GDD §6.9 local 1.2 s "Go Peak!" mini-cinematic, framing the Peak uniform as it appears. `CameraType = Scriptable`; orbit 150° around the root at distance 10 and height 3 (`TweenService` on a NumberValue driving the CFrame), then restore `Custom`. With Reduced Motion: a static three-quarter framing for 0.6 s, no orbit.
- `flash(color, alpha, frames)`: local screen-space flash (ScreenGui frame). Rate-limited to ≤ `MaxFlashesPerSec` (3, GDD §16.3) and never `#FFFFFF` above alpha 0.5. Disabled with Reduced Motion (replaced by `VfxController.flash(model, "pulse")`).
- `applyLighting(mapId, phase?)` and `overlay(name: "UnderPier" | "Fog", on: boolean)` (§6.5.1).

**Verify:** scenario `cam.shift_lock_default` (KBM client: `UserInputService.MouseBehavior == LockCenter` after spawn; LeftShift toggles; the `AimLock` action reads true); `cam.reduced_motion` (`reducedMotion = true` → `shake()` creates no instance; the awaken cinematic does not orbit); `cam.flash_rate` (10 flash requests in 1 s → ≤ 3 applied).

#### 6.6.6 Hitstop presentation and impact frames

- **Hitstop** (GDD §6.13: M1 3f, Ender 6f, signature 8–10f, Perfect Block 6f; a Getaway opening 12f render-only). The sim freezes the state, and `Adapter.animate` calls `AdjustSpeed(0)` (Part 3 §3.5.6). Render adds: while `fNow ≤ HsF` for a view, `VfxController.freeze(model, true)` (effects attached to that model get `ParticleEmitter.TimeScale = 0.05`); `CameraController.kick(HsF − fNow)` for the local attacker or victim.
- **Impact frames** (GDD §6.13, §13.4): on contact by a move with `signature = true`, for 2 frames, "local to the impact, never full-screen", off with Reduced Motion. Implementation: Highlight `impact` on the victim (fill ink, outline white) plus `VFX_ImpactStar` with the ink starburst variant behind the victim (BillboardGui, 8 studs). Shown to the local attacker (predicted), the local victim, and observers ≤ 40 studs. With Reduced Motion: Highlight `pulse` (outline only, 15f).

**Verify:** scenario `vfx.impact_frame` (`attendant.4` signature hit on a dummy → a Highlight with `FillTransparency = 0` exists for 2 ± 1 render frames; with Reduced Motion, none with fill; a pulse instead).

#### 6.6.7 `BuildVfxTemplates` (`ReplicatedStorage.VFX`)

Every template is a Model named after its recipe. Rules:
- An invisible root part `Anchor` (0.2 stud cube, `Transparency = 1`, `Anchored = true`, all collision/query/touch off, `CastShadow = false`, `CollisionGroup = "CosmeticNoQuery"`) with Attachments `A0`/`A1`, plus visual parts with the same flags. Templates stay anchored for world-placed (`at`) effects; VfxController un-anchors and sets `Massless = true` on every part of a clone **before** welding it to a character (`attach`, §6.6.3). Templates never contain welds or constraints to a character.
- `LightInfluence = 0` on every ParticleEmitter, Beam and Trail. `LightEmission` only for magic (Beam_Magic, Awaken_Aura). Default particle texture only (agent-assets §2.6).
- Root attributes `Recipe`, `Layer`, `Emitters`, `ParticlesEst`, `BeamsTrails`, `Lod1`, `Tintable`.
- CSG at **Edit time only** (`GeometryService:SubtractAsync`, `UnionAsync`, `IntersectAsync`, `FragmentAsync`; security None, yields [Docs `GeometryService.yaml`]). Results are cached in the template and never computed at runtime.

| Recipe | Source | Spec (beyond agent-assets §7.4 / Part 4 §4.7.2) |
|---|---|---|
| `VFX_HitSpark_Light`, `VFX_Impact_Heavy`, `VFX_ImpactStar`, `VFX_Slash_Smear`, `VFX_Beam_Magic`, `VFX_Shockwave_Ring`, `VFX_Telegraph_Circle`, `_Line`, `_Cone`, `VFX_Guard_Bubble`, `VFX_Guard_Break`, `VFX_Dash_Puff`, `VFX_Awaken_Aura` | agent-assets §7.4 | As listed there. `VFX_Guard_Break` = the bubble Ball fragmented at build time (`GenerateFragmentSites` + `FragmentAsync`, cached), in sand `#EAD9A8` and sea-glass `#8FD3C1` chips: the graybox stand-in for GDD §13.4's guard that bursts like a kicked sandcastle. The telegraph templates add the `SurfaceGui` overlay of §6.6.3. `VFX_Awaken_Aura` is used only by the streak aura (Peak shows the uniform instead of an aura) |
| `VFX_Telegraph_Arc`, `VFX_Projectile_Orb`, `VFX_Zone_Disc`, `VFX_Wave_Wall`, `VFX_Stealth_Shimmer`, `VFX_Status_Pips`, `VFX_Getaway_Quarter` and every other Part 4 §4.7.2 recipe | Part 4 §4.7.2 | As listed there. `VFX_Encase_Block` is deferred with the `freeze` stun flavour (not built in M1-G) |
| `VFX_KO_Burst` | new | "Checked out!": a puff of sand (ParticleEmitter, default texture, `Emit(30)`, Lifetime 0.4–0.7, Speed 6–10, colour `#EAD9A8` → `#F3E9D2`) plus a waving hankie (a white Block 1.2 × 1.2 × 0.05 hinged at one edge, rotated ±30° twice over 0.8 s by the controller, then faded). No explosion and no fire |
| `VFX_Getaway_Frame` | new | The Getaway postcard frame: four standing panels (Paper `#FFF8EC`, 0.4 thick, 10 tall, inner span = the Getaway ring diameter) with an ink edge, forming a square border round the ring; a stamp Block (3 × 3.6 × 0.2) in the caster's palette `primary` on the north-east corner; a `SurfaceGui` caption on the south panel, filled by the client with `Skin.move(moveId)`. The controller drops it from +20 studs over 8 frames (the slam) |
| `VFX_Tide_Foam` | new | A Beam (white, `Width0 = Width1 = 1.5`, transparency 0.3, `LightInfluence = 0`) between two attachments 240 studs apart along X; the controller moves the template between waterlines (§6.3.8) |
| `VFX_BlockSpark` | new | ParticleEmitter, default texture, `Emit(6)`, Lifetime 0.1–0.15, Speed 12–18, white→`#B3E5FC` |
| `VFX_PerfectStar` | new | BillboardGui (`AlwaysOnTop = false`, 6 × 6 studs) with a `★` TextLabel in LuckiestGuy, lemon `#F7E06B`, ink UIStroke; plus a ring (Shockwave_Ring at 0.5 scale, lemon). The ★ stands in for GDD §13.4's starfish-shaped flash |
| `VFX_Breakaway_Burst` | new | Shockwave_Ring (white, Size ×4 over 0.2 s) + sparkles `Emit(24)`, Lifetime 0.3: the graybox ring of sea spray (GDD §13.4) |
| `VFX_Splat` | new | Flat Neon disc Ø4 (0.05 thick) oriented to the wall normal, palette core, fading over 0.6 s, + ImpactStar |
| `VFX_Tell_Ring` | new | CSG ring (Cylinder Ø4.5 minus Cylinder Ø3.7, 0.1 thick), SmoothPlastic, transparency 0.35; the Style Tell ground ring (§6.8.4) |
| `VFX_Telegraph_*_Lod1`, `VFX_HitSpark_Light_Lod1`, … | derived | Every fx recipe with emitters gets a `_Lod1` copy: emitter Rate and Emit halved, Trails disabled |

Asset keys (Part 2 §2.9.1 format) for generic effects, all `{id = "tpl:VFX/<Recipe>", status = "placeholder", source = "agent", kind = "vfx"}`:

| Key | Recipe | Key | Recipe |
|---|---|---|---|
| `vfx.hit.light` | VFX_HitSpark_Light | `vfx.awaken.burst` | VFX_Shockwave_Ring (palette; the graybox postmark ring of the Peak reveal, GDD §13.4) |
| `vfx.hit.heavy` | VFX_Impact_Heavy | `vfx.ko.poof` | VFX_KO_Burst (sand puff and hankie) |
| `vfx.hit.star` | VFX_ImpactStar | `vfx.wallsplat.splat` | VFX_Splat |
| `vfx.hit.block` | VFX_BlockSpark | `vfx.realm.frame` | VFX_Getaway_Frame |
| `vfx.pb.star` | VFX_PerfectStar | `vfx.realm.ring` | VFX_Shockwave_Ring |
| `vfx.guard.bubble` | VFX_Guard_Bubble | `vfx.spawn.shimmer` | VFX_Stealth_Shimmer (white) |
| `vfx.guard.break` | VFX_Guard_Break | `vfx.tell.ring` | VFX_Tell_Ring |
| `vfx.dash.puff` | VFX_Dash_Puff | `vfx.streak.aura` | VFX_Awaken_Aura (coral, Rate × 0.4) |
| `vfx.breakaway.burst` | VFX_Breakaway_Burst | `vfx.spring.boing` | VFX_Shockwave_Ring (cobalt; the trampoline bounce) |
| `vfx.telegraph.circle` / `.line` / `.cone` / `.arc` | VFX_Telegraph_* | `vfx.tide.foam` | VFX_Tide_Foam |

**Verify:** `BuildVfxTemplates.verify()`: every recipe above plus every recipe referenced by `vfx.<style>.<slot>` manifest entries and Part 4's extra keys (`vfx.status.*`, `vfx.dustsheet.getaway_dark`) exists; `Emitters` on each template ≤ the per-move cap for the lowest rarity using it; no part has `CanCollide`/`CanQuery`/`CanTouch` true; every emitter has `LightInfluence == 0`. Scenario `vfx.templates_all_play` (play each key once at a fixed point; `[HB][Vfx]` logs no missing template).

#### 6.6.8 `BuildPlaceholderAnims` (`ReplicatedStorage.Anim`)

Implements Part 4 §4.6 exactly (22 sequences per implemented style; the §4.6.2 pose library, including the slice kits' poses `sit`, `carry`, `dig`, `squeeze`, `duck` and `flop`; naming, attributes `AssetKey`, `MoveId`, `Frames`, `Gen`; markers; `PoseBase.EasingStyle` `CubicV2`/`Constant`/`Linear`; `AnimationClip.Loop`/`Priority`). Pose hierarchy: `HumanoidRootPart › LowerTorso › UpperTorso › Head`, `UpperTorso › <Side>UpperArm › <Side>LowerArm › <Side>Hand`, `LowerTorso › <Side>UpperLeg › <Side>LowerLeg › <Side>Foot`, built with `Keyframe:AddPose` / `Pose:AddSubPose`. Joint names are read from `ServerStorage.Templates.DummyRig` (§6.8.6), hence the builder order (Part 1 §1.6.6). This part adds the common clips of Part 3 §3.11.8 and the render-only get-up.

New poses (added to Part 4's library; same sign conventions, spike S-P4-7):

| Pose | Joints (X, Y, Z degrees) |
|---|---|
| `hurt_light` | UT(−15,0,0) H(−10,0,0) RUA(20,0,−35) LUA(20,0,35) |
| `hurt_heavy` | LT(−20,0,0) UT(−30,0,0) H(−20,0,0) RUA(60,0,−70) LUA(60,0,70) RUL(−20,0,0) LUL(15,0,0) |
| `tumble` | LT(−80,0,20) UT(−20,0,0) RUA(120,0,−40) LUA(120,0,40) RUL(40,0,0) LUL(−30,0,0) |
| `splat` | UT(10,0,0) RUA(0,0,−110) LUA(0,0,110) RUL(0,0,−25) LUL(0,0,25) |
| `dizzy` | UT(8,20,0) H(15,0,10) RUA(10,0,−20) LUA(10,0,20) |
| `ko_lie` | LT.pos(0,−2.2,0) LT(−90,0,0) RUA(0,0,−80) LUA(0,0,80) |
| `getup_kneel` | LT.pos(0,−1.2,0) UT(25,0,0) RUL(80,0,0) RLL(−90,0,0) LUL(−10,0,0) LLL(−60,0,0) RUA(30,0,−20) LUA(30,0,20) |

Common clips (`Name` = key's last two segments joined, e.g. `common_dash_1`; `MoveId = "common"`; one `Face` marker at frame 0; Priority `Action2`):

| Key | `Frames` | Keyframes (frame: pose) | Loop |
|---|---|---|---|
| `anim.common.block` | 8 | 0 neutral, 3 guard_wide, 8 guard_wide | true |
| `anim.common.block_hit` | 12 | 0 guard_wide, 3 hurt_light, 12 guard_wide | false |
| `anim.common.dash_1` | 18 | 0 crouch, 4 fly, 14 fly, 18 ready | false |
| `anim.common.dash_2` | 15 | 0 ready, 3 spin_a, 12 spin_b, 15 ready | false |
| `anim.common.dash_3` | 15 | 0 ready, 3 spin_b, 12 spin_a, 15 ready | false |
| `anim.common.dash_4` | 14 | 0 crouch, 5 hurt_light, 14 ready | false |
| `anim.common.dash_5` | 12 | 0 leap, 4 fly, 12 fly | false |
| `anim.common.techroll` | 18 | 0 crouch, 6 spin_a, 12 spin_b, 18 ready | false |
| `anim.common.hitstun` | 27 | 0 neutral, 2 hurt_light, 20 hurt_light, 27 ready | false |
| `anim.common.stagger` | 18 | 0 hurt_light, 9 dizzy, 18 ready | false |
| `anim.common.launched` | 66 | 0 hurt_heavy, 18 tumble, 66 tumble | false |
| `anim.common.wallsplat` | 30 | 0 splat, 30 splat | false |
| `anim.common.guardbroken` | 84 | 0 hurt_heavy, 12 dizzy, 84 dizzy | false |
| `anim.common.grabbed` | 30 | 0 hurt_heavy, 30 hurt_heavy | true |
| `anim.common.tumble` | 12 | 0 hurt_heavy, 12 tumble | false |
| `anim.common.breakaway` | 20 | 0 crouch, 4 buff, 20 ready | false |
| `anim.common.shakeoff` | 18 | 0 dizzy, 6 spin_a, 12 spin_b, 18 ready | false |
| `anim.common.ko` | 30 | 0 hurt_heavy, 12 ko_lie, 30 ko_lie | false |
| `anim.common.getup` | 24 | 0 ko_lie, 10 getup_kneel, 24 ready | false (render-only, §6.7) |

Counts: M0 = 2 × 22 + 19 = 63 sequences; M1-G = 6 × 22 + 19 = 151.

**Verify:** `BuildPlaceholderAnims.verify()` (count, unique `AssetKey`, `Frames` equals the last keyframe frame); Part 4 scenario `anim.placeholders_built`; MCP `search_game_tree` under `ReplicatedStorage.Anim` returns 151 KeyframeSequences in M1-G.

#### 6.6.9 Animation upload pipeline (H-07 route)

Placeholder KeyframeSequences only play in Studio until they are published (§0.5). Part 1 H-08 publishes them by hand; this pipeline is the alternative the owner may pick at T014 (Part 1 H-07 key; built by card **T143**, conditional: `skipped (H-08 chosen)` otherwise). The agent never creates products, never publishes the place, and never sees the key.

| Step | Where | What |
|---|---|---|
| 1. Export | Studio, **Edit** datamodel: snippet `tools/studio/export_anim.luau` (ARGS `{key, offset?, len?}`; ≤ 1.5 KB) | Resolves `Assets.entry(key)` → `ReplicatedStorage.Anim.<Name>` (a KeyframeSequence). `local buf = SerializationService:SerializeInstancesAsync({seq})` (returns a `buffer` of `.rbxm` content; yields; usable from plugins, so from `execute_luau`) [Docs `SerializationService.yaml`]. Base64-encodes the bytes `[offset, offset + len)` with a small pure-Luau encoder over `buffer.readu8` (not `HttpService:JSONEncode(buffer)`, which may compress). Returns `{ok, key, size = buffer.len(buf), offset, b64}` as JSON. Read-only: it changes nothing in the place |
| 2. Save | Host (agent) | The agent writes the returned `b64` chunks, in order, to `build/anims/<key>.b64` (`build/` is git-ignored, Part 1 §1.2.7) and records `size`. Chunk length is fixed by spike S-P6-14 (default one chunk) |
| 3. Upload | Host: Lune `tools/upload_anims.luau [--only <key>]` | Reads `HB_OPEN_CLOUD_API_KEY` from `process.env` and **never prints, logs or writes it** (missing → `[HB][CHECK] upload_anims FAIL missing HB_OPEN_CLOUD_API_KEY`, exit 1). Reads `groupId` from `docs/agent/dev-place.json`. For each `build/anims/*.b64` without a line in `build/anims/ids.txt`: decode base64 (pure Luau) → `build/anims/<key>.rbxm` (byte length must equal the recorded `size`), then `POST https://apis.roblox.com/assets/v1/assets` with header `x-api-key` and a `multipart/form-data` body: part `request` = `{"assetType":"Animation","displayName":"<key>","description":"Hullabrawl placeholder","creationContext":{"creator":{"groupId":"<groupId>"}}}`, part `fileContent` = the `.rbxm` bytes with content type `model/x-rbxm` [Docs `cloud/guides/usage-assets.md`: Animation accepts `.rbxm`, `model/x-rbxm`]. The response `{"path":"operations/<id>"}` is polled with `GET https://apis.roblox.com/assets/v1/operations/<id>` every 2 s up to 60 s (td) until `done == true`; then `response.assetId` is appended to `build/anims/ids.txt` as `key=id`. One upload at a time, ≥ 1 s apart (td). Prints `[HB][CHECK] upload_anims key=<key> id=<id>` or `… FAIL key=<key> reason=<status message>`; finally `[HB][CHECK] upload_anims SUMMARY ok=<n> fail=<m>` |
| 4. Patch | Host (agent) | For each `key=id` line, the agent edits `src/Shared/Assets.luau`: `id = "rbxassetid://<id>"`, `status = "placeholder"` (content is still placeholder), `source = "agent"`, `kind = "anim"`. Then the host gate and the sync gate |

Rules:
- **Every run of step 3 is human-approved.** The tool refuses to upload unless invoked with `--approve <n>` where `n` equals the number of pending keys it prints in a dry pass without the flag (so the human sees exactly what will be uploaded). Part 1's example permissions file keeps `lune run tools/upload_anims.luau` on **ask** (§6.13 item 23).
- The uploaded clips belong to the dedicated dev group that owns the dev universe (H-01; Part 8 §8.2.6). They are placeholders and never ship: a release build fails on any `placeholder` asset (§0.5), and the production group receives only final assets, re-uploaded at release (Part 8 §8.2.6).
- `lint_rules` `secrets` covers `tools/upload_anims.luau`; the env var name is the only key-related literal in it.
- The tool writes `build/anims/<key>.rbxm` and `ids.txt` only through `tools/lib/SafeWrite.luau` (Part 8 §8.4.5: it refuses the three protected files `docs/agent/TITLE_CLEARANCE.md`, `tools/denylist.local.txt` and `tools/retired_keys.txt`, and the Lune test `ip.no_tool_writes_signed_files` scans `tools/**`).
- Re-running is idempotent: keys already in `ids.txt` are skipped. A failed key is retried only on the next approved run.

**Verify:** spike **S-P6-14** (export size vs the `execute_luau` result limit and the 2 KB payload cap: in T143 export the shortest and the longest existing clip (`anim.common.*`; `anim.attendant.m1_1` does not exist before T174, which repeats the check on it); pass when one chunk round-trips byte-exactly, i.e. the decoded `.rbxm` length equals `size`; otherwise set the chunk length so every call returns ≤ the measured limit; if neither works, fall back to H-08 and mark T143 `skipped`). Lune `anims.base64_roundtrip` in `tests/core/upload_anims.spec.luau` (pure helper `tools/base64.luau`, loaded with `@tools/`, used by `upload_anims.luau`; encoding then decoding a 3-byte, a 4-byte and a 1 KB seeded-random buffer is lossless); Lune `anims.upload_refuses_without_approve` (a dry pass with a fake key prints the pending count and makes no request). After a real run (T143): `[HB][CHECK] upload_anims SUMMARY ok=<n> fail=0`, `Assets.isPlaceholder` unchanged, and a two-client playtest plays an uploaded clip on the observer (`AnimationTrack.IsPlaying` true on the remote rig; in T143 an `anim.common.*` clip, from T174 also `anim.attendant.m1_1`).

---

### 6.7 Cosmetic ragdoll presenter

GDD §13.3: a hybrid ragdoll. The authored tumble (10–14f) plays first, then a physics ragdoll with BallSocket limits and high damping ("noodle limbs flop instead of breaking"), then an authored get-up. Ragdoll physics is **client-side and cosmetic only**; the authoritative capsule stays upright and kinematic in the sim (Part 3 §3.8.6). The presenter is a section of `RenderController.luau` (`RenderController.ragdoll`), because §0.3 fixes the file list.

#### 6.7.1 Lifecycle

| Step | Trigger | Action |
|---|---|---|
| Tumble | enter `Ragdoll` | Nothing here: `Adapter.animate` plays `anim.common.tumble` for the first 12 frames (`Combat.Ragdoll.TumbleAnimFrames`, Part 3) |
| Begin | `St == Ragdoll` and `e ≥ Perf.Ragdoll.TumbleSwitchFrames` (12) | If the view is within `Perf.Ragdoll.LodStuds` (120) and the active ragdoll count < `Perf.Tiers[tier].MaxRagdolls` (4 / 8 / 12): build the clone (§6.7.2). Otherwise keep the tumble pose (no physics) |
| Simulate | each frame | `AlignPosition` on the clone's `LowerTorso` pulls toward `root.Position − (0, Perf.Ragdoll.FloorOffset, 0)` in **XZ only** (`ForceLimitMode = PerAxis`, `MaxAxesForce = (F, 0, F)` with `F = AlignForcePerMass × clone mass`, `Responsiveness = 25`). The flop never drifts from the authoritative hurtbox; gravity handles Y |
| Finish (blend) | leaving `Ragdoll` to `Idle` (wake-up) | Anchor every clone part; over `GetUpBlendSec` (0.25 s) lerp each clone part's CFrame from its captured pose to the matching real (or proxy) part's live CFrame; then destroy the clone, restore visibility, and `AnimController.play(model, "anim.common.getup", {speed = 24 / (GetUpBlendSec × 60)})` |
| Finish (instant) | leaving `Ragdoll` to **any other state except `KO`**: `WallSplat` (it can start at e < 18 while the clone exists from e ≥ 12), `Hitstun`, `Launched`, `Stagger`, `Grabbed`, `Breakaway`, `TechRoll`, `ShakeOff`; **and** a Ragdoll re-entry (still `Ragdoll` but `Sub` reset to 0 or `StF` increased: a juggle hit on an airborne ragdolled victim, or `WallSplat` → `Ragdoll`) | Destroy the clone and restore visibility at once (the new state's VFX/animation hides the pop). On a re-entry the presenter then runs `begin` again under the normal rule (`e ≥ 12` of the new Ragdoll). `begin` never runs on a view that still has a clone: it first calls `finish(instant)` |
| KO | `KO` while ragdolled | Keep the clone until the poof at `BodyHideSec`, then destroy |
| Safety | clone age > `MaxLifeSec` (6 s), or the model is removed | Destroy and restore |

Undo: `Ragdoll` is an authoritative (S-class) state, so there is no predicted ragdoll and no undo path.

#### 6.7.2 Clone construction

```lua
--!strict
-- RenderController.ragdoll.begin (excerpt). Spec §6.7.2. R15 rig attachment names are standard
-- (verify in spike S-P6-1 on DummyRig).
type Joint = { a: string, b: string, att: string, upper: number, twistLo: number, twistHi: number }
local JOINTS: { Joint } = {
	{ a = "UpperTorso", b = "Head", att = "NeckRigAttachment", upper = 45, twistLo = -60, twistHi = 60 },
	{ a = "LowerTorso", b = "UpperTorso", att = "WaistRigAttachment", upper = 30, twistLo = -30, twistHi = 30 },
	{ a = "UpperTorso", b = "LeftUpperArm", att = "LeftShoulderRigAttachment", upper = 110, twistLo = -90, twistHi = 90 },
	{ a = "LeftUpperArm", b = "LeftLowerArm", att = "LeftElbowRigAttachment", upper = 100, twistLo = -10, twistHi = 10 },
	{ a = "LeftLowerArm", b = "LeftHand", att = "LeftWristRigAttachment", upper = 60, twistLo = -30, twistHi = 30 },
	-- … Right arm mirrors Left; legs: Hip 90/±45, Knee 110/±5, Ankle 40/±20 (values from Perf.Ragdoll.Joints)
}

local function begin(visual: { [string]: BasePart }, folder: Folder, rootVel: Vector3): { [string]: BasePart }
	local clone: { [string]: BasePart } = {}
	for name, src in visual do -- 15 body parts (proxies or avatar parts, §6.8.3)
		local p = src:Clone()
		for _, ch in p:GetChildren() do
			if not (ch:IsA("Attachment") and string.sub(ch.Name, -13) == "RigAttachment") and not ch:IsA("SurfaceGui") then
				ch:Destroy()
			end
		end
		p.Anchored = false
		p.CanCollide = true
		p.CanQuery = false
		p.CanTouch = false
		p.Massless = false
		p.CollisionGroup = "RagdollFx" -- collides with Default + MapBoundary only (§6.1.5)
		p.Transparency = src.Transparency
		p.LocalTransparencyModifier = 0
		p.CFrame = src.CFrame
		p.AssemblyLinearVelocity = rootVel
		p.Parent = folder
		clone[name] = p
	end
	for _, j in JOINTS do
		local pa, pb = clone[j.a], clone[j.b]
		local a0 = pa and pa:FindFirstChild(j.att)
		local a1 = pb and pb:FindFirstChild(j.att)
		if a0 and a1 and a0:IsA("Attachment") and a1:IsA("Attachment") then
			local bs = Instance.new("BallSocketConstraint")
			bs.Attachment0, bs.Attachment1 = a0, a1
			bs.LimitsEnabled, bs.UpperAngle = true, j.upper
			bs.TwistLimitsEnabled, bs.TwistLowerAngle, bs.TwistUpperAngle = true, j.twistLo, j.twistHi
			bs.MaxFrictionTorque = 15 -- "high damping" (td; Perf.Ragdoll.FrictionTorque)
			bs.Parent = pa
			local nc = Instance.new("NoCollisionConstraint")
			nc.Part0, nc.Part1 = pa, pb
			nc.Parent = pa
		end
	end
	return clone
end
```

- The originals are hidden with `AppearanceController.setHidden(model, true)` (sets `LocalTransparencyModifier = 1` on the visual set and Charm Gear). Charm Gear follows the clone's parts while ragdolled.
- The face `SurfaceGui` travels with the cloned head, so expressions keep working (`ko`, `hurt_heavy`).
- Under Server Authority, client-created unanchored parts are simulated locally; this is spike S-P6-9.

**Verify:** scenario `ragdoll.clone_lifecycle` (bot Ender on a dummy within 30 studs: a clone Model appears under `HB_Local.Ragdolls` 12 ± 2 frames after `Ragdoll` entry; every clone part has `CanQuery == false` and `CollisionGroup == "RagdollFx"`; it is gone ≤ 0.3 s after the dummy's `Ragdoll` ends; the real rig's parts have `LocalTransparencyModifier == 0` again); `ragdoll.cap` (6 dummies ragdolled at once at tier Low → ≤ 4 clones); `ragdoll.wallsplat_handoff` (the Staff Room: an Ender toward `SplatWall` from 18 studs, splat at e ≈ 14 → on the first render frame where the dummy's `St == WallSplat`, or the next one, no clone exists under `HB_Local.Ragdolls` for it and its real rig parts have `LocalTransparencyModifier == 0`); `ragdoll.juggle_rebegin` (a Ragdoll refresh by an air hit → the old clone is destroyed at once and at most one clone exists per view at any time); `ragdoll.no_misprediction` (the local player is ragdolled 10 times; the Part 7 misprediction log shows no entries attributable to `RagdollFx` parts).

---

### 6.8 Appearance rendering (Full Uniform, Postcard Look, Charm Gear, Peak)

§0.4 fixes the settings: `CharacterLook` (the **Full Uniform** toggle, default **Off**), `OwnLook` (`"Postcard"`, the **Postcard Look**, or `"Avatar"`), `ShowStyleGear` (**Charm Gear**, default on) and the viewer-local `OthersAppearance`; the always-on Style Tell; the fixed hurtbox; and the Peak uniform (60 s; nobody grows). Part 5 owns data and replication (Player attributes `CharacterLook`, `OwnLook`, `ShowStyleGear`, `PostcardCode`, `EquippedStyle`, `Flair`, `Variant`; Part 2 §2.5.2; Part 5 §5.7). This section is the graybox renderer, implemented in `AppearanceController.luau`.

In a fight a player is their own look plus Charm Gear: **one oversized held or back prop that defines the silhouette, plus a head piece** (GDD §12.3). The whole staff member appears only as the Full Uniform (when the player turns it on), as the Peak uniform, on the postcard in the Wish reveal and in the Scrapbook.

#### 6.8.1 Effective mode per viewer

```lua
--!strict
export type LookMode = "Character" | "Postcard" | "Avatar" | "Simplified" -- "Character" renders the Full Uniform
-- owner attributes: CharacterLook (bool), OwnLook ("Postcard"|"Avatar"); bots have neither → Character.
-- atPeak: the owner's root shows AwkF0 > 0 (RenderController calls AppearanceController.setPeak, §6.6.1).
local function effectiveMode(characterLook: boolean?, ownLook: string?, viewer: string, atPeak: boolean): LookMode
	if viewer == "Simplified" then
		return "Simplified"
	elseif viewer == "AlwaysCharacter" or atPeak then
		return "Character" -- at Peak the uniform appears whatever the Full Uniform setting (§0.4)
	end
	local own: LookMode = if ownLook == "Avatar" then "Avatar" else "Postcard"
	if viewer == "AlwaysOwn" then
		return own
	end
	-- "AsChosen"
	if characterLook == false then
		return own
	end
	return "Character"
end
```

The local player's own character uses `AsChosen` regardless of `OthersAppearance`, and still switches to the uniform at Peak. Missing attributes mean `Character`. This is deliberate and does not override the Full Uniform default: bots never carry the attributes, so they always show their style's uniform, and in M0, before Part 5 lands, there is no Postcard Look data to draw. From M1-G every player gets `CharacterLook = false` at join (Part 2 §2.5.2, Part 5 §5.7), so the default Off holds. With Part 5's defaults (`CharacterLook = false`, `OwnLook = "Postcard"`, `ShowStyleGear = true`) everyone sees the Postcard Look plus Charm Gear.

#### 6.8.2 What each mode renders

| Mode | Body | Props | Face | Source |
|---|---|---|---|---|
| `Character` (the Full Uniform) | Proxy rig (§6.8.3) in `Theme.stylePalette(styleId, variantId)` (§6.9.4; `variantId` = the owner's `Variant` attribute, `""` = base): torso `primary`, limbs `secondary`, head `secondary`; `classic` proportions | **Always** the style's Charm Gear: only the `model.<style>.gear_*` entries of `StyleDef.kitProps` (Part 4 §4.7.4; its `summon_*` entries are summon models, §4.7.5, never worn): the silhouette read (GDD §12.3). Gear parts take their colour role: `primary`, `secondary` and `accent` from `Theme.stylePalette(styleId, variantId)`, `ink` from the Theme `Ink` token | UI face (§6.8.5), expressions on | GDD §12.3 "Full Uniform" |
| `Postcard` (the Postcard Look) | The player's own look drawn in flat postcard colours on the proxy rig. Proportions from the body style: `Appearance.PostcardStyles[spec.body].id` (`classic`, `round`, `tall`, `tiny`) → `Perf.Looks.BodyScale[id]` (below). Colours per the Part 5 §5.7.1 three-colour rule: head and arms the skin tone `Appearance.PostcardSkins[spec.skin]`; torso `spec.color`; legs `Appearance.OutfitColors[spec.outfit]`, or `spec.color` darkened by `Appearance.NoOutfitDarken` (0.65) when `outfit = 0`. Hat and outfit props | Charm Gear if `ShowStyleGear` | UI face, variant `PostcardSpec.face` | GDD §12.3 "Postcard Look"; Part 5 §5.7.1 |
| `Avatar` | The real Roblox avatar as loaded (no proxies) | Charm Gear if `ShowStyleGear`, attached to the real parts | The avatar's own face (UI face hidden) | GDD §12.3 "Roblox Avatar". A postcard toon pass over real avatars is a future hook |
| `Simplified` | Proxy rig: torso `primary`, limbs `primary` × `Appearance.SimplifiedLimbShade` (0.75, Part 5 §5.10), head `secondary` (palette from `Theme.stylePalette(styleId, variantId)`) | None | UI face, `neutral` only | §0.4 viewer override; Part 4 §4.7.6 |

All modes keep the **Style Tell** (§6.8.4). The hurtbox is never touched (§0.4, GDD §6.2).

**Body styles** (Postcard Look only; render proxies only, never the hurtbox or the collider). `Perf.Looks.BodyScale[id] = {core, limbs}` multiplies the proxy sizes of the head and torso (`core`) and of the arms and legs (`limbs`). Each proxy stays centred on its real part, so a scaled proxy may overlap its neighbour or leave a small gap at a joint; the graybox accepts both.

| Body style | `core` (x, y, z) | `limbs` (x, y, z) | Read |
|---|---|---|---|
| `classic` | 1, 1, 1 | 1, 1, 1 | default proportions |
| `round` | 1.25, 1, 1.25 | 1.1, 1, 1.1 | rounder and broader |
| `tall` | 0.95, 1.12, 0.95 | 0.95, 1.12, 0.95 | longer and leaner |
| `tiny` | 0.85, 0.85, 0.85 | 0.85, 0.85, 0.85 | smaller all over |

All body styles are people; there are no creature options (Part 5 §5.7.1).

**Peak uniform** (§0.4; GDD §6.9). When a character goes Peak (RenderController: enter `Awakening`), AppearanceController calls `setPeak(model, true)`:
1. the effective mode becomes `Character` for every viewer except `Simplified` (§6.8.1), so the staff member's uniform appears on the Guest whatever the Full Uniform setting;
2. the style's optional Peak piece is attached: every `ReplicatedStorage.Looks.Peak_<style>_*` template found for the style (Part 4 §4.4.2, §4.7.4: one per slice style, keys `model.attendant.peak_blazer`, `model.lifeguard.peak_flag`, `model.sandcastle.peak_lamp`, `model.busker.peak_drum`, `model.scoop.peak_kiosk`, `model.dustsheet.peak_lanterns`; for example Sonny's striped blazer and megaphone, or the red flag on a pole on Paddles' back). A style without one shows the uniform and the gilt trim only;
3. the gilt trim: every part of the Charm Gear and Peak pieces built with role `accent` takes `Theme.Gilt` for the Peak (gold marks the Peak; §6.9.4). In `Simplified` the pieces are skipped and the torso proxy turns gilt instead.

Nothing scales: no visual scale, no size change, and `hurtboxScale = 1` for every kit. On `AwkF0` > 0 → 0, `setPeak(model, false)` restores the previous mode at once.

**Charm Gear is static.** One template per key and no alternate states: Charm Gear never folds, unfolds or swaps models on a move or on the `Kit` attribute. Move visuals come from the move's VFX and summon models (Part 4 §4.7). The plush keyring sold with each Charm is a cosmetic extra; it has no graybox template and is never part of the silhouette read.

**Off-Duty Looks (graybox render).** An Off-Duty Look (internally a Variant) is a palette change only; shapes never change (GDD §8.7). `Theme.stylePalette(styleId, variantId)` returns `StyleDef.palette` (Part 4 §4.7.1) for `""`; for `v1` it swaps `secondary` and `accent`; for `v2` it darkens every colour by 25 % (`Color3:Lerp(black, 0.25)`, applied to `primary`, `secondary` and `accent`). Proxy body, Charm Gear, the Style Tell ring and chip, and the owner's VFX tint (§6.6.3) all read it, so an Off-Duty Look is visible in every mode where the style palette shows. Ids beyond `v2` (future content) fall back to `v2`'s rule until Part 4 defines per-look palettes.

**Capture staging (silhouette mode, Studio only).** While `ReplicatedStorage.Looks:GetAttribute("Silhouette") == true` (set only by `BuildTemplates.build({silhouette = true})` in Edit, §6.8.6), AppearanceController renders every character for the human blind tests: proxies, Charm Gear, Peak pieces and summon models in pure black `#000000`; no UI face; the nameplate chip, name and info rows hidden; the ground ring black; the owner's VFX tinted `#3A3A3A`. Telegraphs stay as they are. This is the only exception to the always-on Style Tell besides deferred stealth (§6.8.4). `BuildTemplates.verify()` warns while it is on, so it never reaches a commit or a publish.

#### 6.8.3 Proxy rig and attachment strategy

- `ReplicatedStorage.Looks.LookRig_R15` (built by `BuildTemplates`): 15 primitive Parts named like the R15 body parts. Sizes are taken from `DummyRig` at build time; the head is a Ball, the rest Blocks. All `CosmeticNoQuery`, `Massless`, no collision/query/touch, `CastShadow = true`. Each proxy part carries copies of its source part's `*RigAttachment`s (same names and CFrames), so the ragdoll (§6.7) and Charm Gear attach the same way.
- For a non-`Avatar` mode, the controller clones the proxy rig into `HB_Local.Looks.<rid>`, sizes each proxy to the real part's `Size` (times the body-style scale in `Postcard` mode, §6.8.2), welds it to the matching real body part with a client-local `WeldConstraint` (`Part0 = proxy`, `Part1 = real`), and hides the real body parts and accessories (`LocalTransparencyModifier = 1`). Clothing and avatar heads are hidden with the parts they render on.
- Charm Gear and Peak-piece Accessories (Part 4 §4.7.4) are cloned per character: for Charm Gear only the `gear_` entries of `kitProps`, never its `summon_` entries. Each `Handle` is welded so that its named attachment matches the same-named attachment on the target part (proxy or real). This is the `Humanoid:AddAccessory` alignment rule done by hand, because the proxy is not in the Humanoid's model. Handles are `Massless`, `CosmeticNoQuery`.
- The welds make the engine handle interpolation of remote characters. Whether a client-local, massless part welded into the **local predicted** character causes mispredictions is spike S-P6-2. The fallback is no welds: a single `workspace:BulkMoveTo(parts, cframes, Enum.BulkMoveMode.FireCFrameChanged)` per frame in the `HB_Render` step.
- **LOD** (`Perf.Looks`): beyond `ProxyDetailStuds` (100) the face SurfaceGui is disabled and Charm Gear hidden. Beyond `TellOnlyStuds` (160) proxies are hidden too (ring and chip remain).
- **Refresh triggers:** the owner's `EquippedStyle`, `CharacterLook`, `OwnLook`, `ShowStyleGear`, `PostcardCode`, `Flair`, `Variant` attribute changes; Peak start and end (`setPeak`, from RenderController); the viewer's `othersAppearance` setting; `CharacterAdded`; the `ReplicatedStorage.Looks` attribute `Silhouette`; `Skin.onChanged` (nameplate text only, §6.8.4). These are not combat one-shots, so attribute-changed signals are fine here.
- Part 3's `Hurtbox.prepare` is asked to set `Humanoid.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.None` server-side (§6.13 item 11), so the default name/health display never competes with the nameplate.

#### 6.8.4 Style Tell (always on)

Rendered for every character in every mode and for every viewer (§0.4; GDD §12.3 readability guarantee):
1. **Nameplate chip.** BillboardGui `HB_Nameplate` on the head (proxy or real): `AlwaysOnTop = false`, `MaxDistance = Perf.Looks.TellChipStuds` (150, matching the GDD §11.1 sightline cap), `LightInfluence = 0`, `StudsOffset = (0, 2.6, 0)`, `Size = UDim2.fromOffset(220, 72)`. Rows:
   - **Chip row (always):** a circle in the style `primary` with the rarity shape glyph (§6.9.4) and `Skin.style(id).name` in Body font 14; a Finish glint (✦ in the Finish colour) when `Flair ≠ "plain"`.
   - **Name row (≤ 60 studs):** `Player.DisplayName`, title (`Skin.text("title." .. Title)` **only when `Title ~= ""`**; otherwise the title element is omitted, so no `title.` key is ever resolved), Rookie tag (`ui.nameplate.rookie` when `Rookie`), prestige stars, First Class stroke (a cobalt `UIStroke` when `Vip`, the First Class pass).
   - **Info row (GDD §15.3: focus target, or ≤ `NameplateInfoStuds` 25 studs):** thin HP bar, 2 Breakaway pips (`BkP`, refill sweep from `BkProg / 1500`), block-crack icon when `Grd < 40` while blocking, broken icon in `GuardBroken`.
2. **Ground ring.** `vfx.tell.ring` in `primary`, placed at foot level (`−(Humanoid.HipHeight + root.Size.Y / 2) + 0.05` below the HRP) through `VfxController.play{attach = HRP}`, so the §6.6.3 attach rule applies: every ring part is set `Anchored = false`, `Massless = true`, `CanCollide/CanQuery/CanTouch = false`, `CollisionGroup = "CosmeticNoQuery"` **before** the `WeldConstraint` to the HRP is created (an anchored ring would anchor the whole character). If spike S-P6-2 shows welds into the local predicted character mispredict, the ring uses the same fallback as the proxies: no weld, moved once per frame with `workspace:BulkMoveTo` in the `HB_Render` step. Hidden beyond `TellOnlyStuds` 160.
3. **VFX palette.** All of the owner's effects are tinted from `Theme.stylePalette(styleId, variantId)` (§6.6.3, §6.8.2).
4. **Text refresh.** Chip and name rows re-render on `Skin.onChanged` (a later locale or seasonal text override; Part 2 §2.9.2), on the owner's `Title`, `Rookie`, `Prestige`, `Vip` attribute changes, and on the refresh triggers of §6.8.3.

**Stealth (deferred).** The stealth status `iv` and its render are deferred until their first user (Shutter, GDD §7.3.8, Year 1; Part 4 §4.9). When it lands: for enemy viewers beyond `Perf.Looks.StealthRevealStuds` (12) of an owner with status `iv`, the nameplate name and info rows and the ring are hidden; the chip is hidden too and the Tell is carried by `VFX_Stealth_Shimmer` in the owner's style colour. This is an exception to "always on" (§6.14 item 3). No M1-G kit uses it.

#### 6.8.5 UI-drawn faces and FaceController

- `ReplicatedStorage.Looks.Face_UI`: a `SurfaceGui` (`Face = Front`, `SizingMode = PixelsPerStud`, `PixelsPerStud = 100`, `LightInfluence = 0`, `AlwaysOnTop = false`) holding `Eyes.L`, `Eyes.R` (Frames with `UICorner` 0.5, ink, each with a small white `Glint` Frame) and `Mouth` (a Frame inside a clipping Frame with `ClipsDescendants = true`, so a stroked rounded frame shows only its upper or lower half). It is parented to the proxy head (or nothing in Avatar mode).
- Eyes follow GDD §12.3: glossy oval eyes like enamel pins, each with one highlight (the `Glint`, shown for the `dot` and `wide` shapes); never pie-cut eyes. Lines use the warm navy-plum ink (§6.9.4), never black.
- Final art later drops into the same SurfaceGui as an `ImageLabel` atlas (`ImageRectOffset`/`ImageRectSize`, agent-assets §7.2): same container, same API.

Expression table (24 keys, GDD §12.3 "24 expressions"). Eye shapes: `dot` (an oval, height 1.2 × width), `wide` (larger oval), `line` (flat bar), `happy` (upper half-ring), `sad` (lower half-ring), `x` (two crossed bars), `spiral` (ring with stroke), `star` (★ TextLabel). Mouth shapes: `line`, `smile` (lower half-ring), `frown` (upper half-ring), `open` (filled ellipse), `o` (small ring), `grit` (bar with 3 tick marks), `wave` (3 rotated bars), `flat_wide`.

| Expr | Eyes (shape, scaleY, offsetY px) | Mouth (shape, w, h) | Expr | Eyes | Mouth |
|---|---|---|---|---|---|
| `neutral` | dot, 1, 0 | line, 16, 3 | `hurt_light` | line, 1, 0 | frown, 14, 6 |
| `smile` | dot, 1, 0 | smile, 18, 8 | `hurt_heavy` | x, 1, 0 | open, 14, 12 |
| `grin` | happy, 1, 0 | smile, 24, 10 | `dizzy` | spiral, 1, 0 | wave, 18, 6 |
| `grit` | line, 0.6, 2 | grit, 20, 6 | `ko` | x, 1, 0 | o, 8, 8 |
| `shout` | wide, 1.2, −2 | open, 18, 16 | `block` | line, 0.7, 2 | flat_wide, 20, 3 |
| `focus` | dot, 0.8, 1 | line, 12, 3 | `perfect_block` | star, 1, 0 | smile, 20, 8 |
| `smug` | happy, 0.6, 2 | smile, 14, 5 | `awaken` (Peak) | star, 1.2, −2 | open, 20, 14 |
| `surprised` | wide, 1.3, −3 | o, 10, 10 | `victory` | happy, 1, 0 | open, 22, 12 |
| `scared` | wide, 1.3, 0 | wave, 18, 6 | `sleepy` | line, 0.4, 4 | o, 6, 6 |
| `sad` | sad, 1, 2 | frown, 16, 6 | `blank` | dot, 0.8, 0 | line, 10, 2 |
| `angry` | line, 0.7, 2 (rotated ±15°) | frown, 18, 6 | `wink` | dot + line | smile, 18, 8 |
| `determined` | dot, 0.8, 2 | flat_wide, 18, 3 | `laugh` | happy, 1, 0 | open, 24, 14 |

```lua
--!strict
export type Expr = "neutral" | "smile" | "grin" | "grit" | "shout" | "focus" | "smug" | "surprised" | "scared" | "sad"
	| "angry" | "determined" | "hurt_light" | "hurt_heavy" | "dizzy" | "ko" | "block" | "perfect_block" | "awaken"
	| "victory" | "sleepy" | "blank" | "wink" | "laugh"
export type FaceHandle = {
	set: (self: FaceHandle, expr: Expr, holdSec: number?) -> (), -- reverts to base after holdSec (nil = until next set)
	setBase: (self: FaceHandle, expr: Expr) -> (),
	destroy: (self: FaceHandle) -> (),
}
-- AppearanceController.face(model: Model): FaceHandle?   (nil in Avatar mode or beyond ProxyDetailStuds)
```

The Postcard face variants 1–6 change the `neutral`/`blank` eye shapes only: 1 dot, 2 wide, 3 tall oval (dot with scaleY 1.4), 4 happy, 5 line (sleepy), 6 small dot (dot at 0.7 size). RenderController drives expressions (§6.6.1 table). Expression changes for characters beyond `ProxyDetailStuds` are skipped.

#### 6.8.6 `BuildTemplates`

| Output | Contents |
|---|---|
| `ServerStorage.Templates.DummyRig`, `BotRig` | `Players:CreateHumanoidModelFromDescriptionAsync(desc, Enum.HumanoidRigType.R15)` with a default `HumanoidDescription` (body colours `#9E9E9E` dummy / `#BDBDBD` bot). Edit mode via `execute_luau` is spike S-P6-1 (the docs show it in a Studio script); fallback: the human inserts one R15 rig via Avatar ▸ Rig Builder into `ServerStorage.Templates.__RigSeed`, and the builder clones it. Then `PrimaryPart = HumanoidRootPart`, Humanoid `DisplayDistanceType = None`, every part `Anchored = false` |
| `ServerStorage.Templates.Fragments.*` | Optional fracture cache (§6.1.8) |
| `ReplicatedStorage.Looks.LookRig_R15` | §6.8.3 |
| `ReplicatedStorage.Looks.Face_UI` | §6.8.5 |
| `ReplicatedStorage.Looks.Gear_<style>_<name>`, `Peak_<style>_<name>`, `Summon_<style>_<name>` | Part 4 §4.7.4 / §4.7.5 recipes (Edit-time `GeometryService:UnionAsync`/`SubtractAsync`). The builder keeps every recipe as plain data (per key, a list of primitives `{shape, size, offset, role}`, `role` ∈ `primary`/`secondary`/`accent`/`ink`), so recipes can be checked in Lune without Studio. One static template per key (no alternate states, §6.8.2) |
| `ReplicatedStorage.Looks.Postcard_Hat_<n>` (6), `Postcard_Outfit_<n>` (4) | Primitive Accessories (table below). Body styles have no template: they are proxy scales (§6.8.2) |

**Charm Gear build list** (GDD §7.3; recipes and exact mount attachments: Part 4 §4.7.4–§4.7.5). Each implemented style has exactly two `gear_` keys, the oversized prop first; summon models follow in `kitProps`; the optional Peak piece is not in `kitProps` (Part 4 K23). M1-G builds 12 `Gear_*`, 6 `Peak_*` and 7 `Summon_*` templates:

| Style | Oversized prop (the silhouette read) | Second piece | Summon models | Peak piece (worn only at Peak) |
|---|---|---|---|---|
| `attendant` (Sonny) | `model.attendant.gear_chair`: folded striped deckchair on the back (`BodyBackAttachment`) | `model.attendant.gear_boater`: straw boater | `model.attendant.summon_chair` | `model.attendant.peak_blazer` |
| `lifeguard` (Paddles) | `model.lifeguard.gear_buoy`: big ring buoy on the back | `model.lifeguard.gear_visor`: visor | — | `model.lifeguard.peak_flag` |
| `sandcastle` (Burrows) | `model.sandcastle.gear_spade`: long spade (`RightGripAttachment`) | `model.sandcastle.gear_hardhat`: hard hat with a pennant | `model.sandcastle.summon_castle`, `model.sandcastle.summon_grandcastle` | `model.sandcastle.peak_lamp` |
| `scoop` (Dollop) | `model.scoop.gear_cone`: giant cone, held | `model.scoop.gear_cap`: paper cap | `model.scoop.summon_van` | `model.scoop.peak_kiosk` |
| `busker` (Bellows) | `model.busker.gear_accordion`: accordion on the chest (`BodyFrontAttachment`; spike S-P4-5) | `model.busker.gear_cap`: feathered flat cap | `model.busker.summon_heckler`, `model.busker.summon_gull` | `model.busker.peak_drum` |
| `dustsheet` (Mothball) | `model.dustsheet.gear_sheet`: hooded dust sheet draped over the avatar (it also covers the head) | `model.dustsheet.gear_tag`: HANDS OFF tag, held | `model.dustsheet.summon_luggage` | `model.dustsheet.peak_lanterns` |

**Silhouette option.** `build({silhouette = true})` rebuilds every `Gear_*`, `Peak_*` and `Summon_*` template and `LookRig_R15` in pure black `#000000` (SmoothPlastic) and sets the attribute `Silhouette = true` on `ReplicatedStorage.Looks`; a normal build restores the colours and clears it. The render rules are in §6.8.2 ("Capture staging"). It serves two staging scenarios owned by Parts 4 and 7: `look.gear_silhouette` (one still per implemented style; each style must read from its oversized prop in pure black at **48 px**, GDD §12.3) and `look.clip_stage` (the staged 30 s demo that the producer records as a masked-silhouette clip for the T11 gameplay-clip test). These captures feed the human blind tests T7–T11 (GDD §5.5): the agent appends `pending` rows to `docs/agent/RECOGNIZABILITY.md`, one per subject and preparation run (Part 8 §8.5.3), and never fills in a result; humans run the tests and sign the results. `verify()` warns `[builder:BuildTemplates] silhouette templates active (rebuild before commit)` while the attribute is set.

Postcard Look catalog (GDD §12.3; graybox primitives; keys `model.postcard.hat_<n>`, `model.postcard.outfit_<n>`). Hats and outfits are generic holiday wear and never copy a style's Charm Gear piece:

| # | Hat (`Appearance.PostcardHats = 6`) | Outfit (`PostcardOutfits = 4`) |
|---|---|---|
| 1 | cap (Cylinder + Wedge brim) | striped vest (Block front/back + 2 stripe Blocks) |
| 2 | sun hat (wide Cylinder brim + Ball crown) | neckerchief (Wedge) |
| 3 | bucket hat (Cylinder with a narrower Cylinder top) | cardigan (Block) |
| 4 | knotted handkerchief (half Ball + 4 small Ball knots at the corners) | bum bag (Block on a thin band) |
| 5 | flower (Ball + 4 Balls) | |
| 6 | sailor cap (Cylinder + Block band) | |

**Verify:** `BuildTemplates.verify()` (both rigs have 15 body parts and a Humanoid; `LookRig_R15` has 15 parts with RigAttachments; every Part 4 gear, Peak and summon key and every Postcard key resolves through `Assets.template`; every `Gear_*` and `Peak_*` template has its mount attachment and every part is massless, non-collidable and in `CosmeticNoQuery`; the silhouette warning while `Silhouette` is set); scenarios `look.modes_matrix` (level **`multi`**: runs only through `multi(2, Baseline30, look.modes_matrix)`, never in solo `studio(…)`; owner sets each of Full Uniform / Postcard Look / Avatar × Charm Gear on/off; viewer sets each `OthersAppearance` → the expected mode per §6.8.1; the hurtbox `Size` is unchanged in all 16 cases), `look.style_tell_always` (in every mode, the viewer sees the chip and ring for an enemy at 100 studs), `look.postcard_three_colour` (spec `{body = 1, skin = 2, color = "#F48FB1", outfit = 0}` → head/arm proxies `Appearance.PostcardSkins[2]`, torso `#F48FB1`, legs `#F48FB1` × 0.65 within 1/255; outfit 3 → legs `#8B3A3A`), `look.body_styles` (for each of `classic`, `round`, `tall`, `tiny`: every proxy size equals the real part's size × `Perf.Looks.BodyScale[id]` within 0.01; the hurtbox `Size` and the root size never change), `look.peak_uniform` (owner on `attendant` with `CharacterLook = false`: `Dev.Command setMeter 100`, go Peak → within 0.2 s the owner renders `Character` with the gilt trim on its `accent` parts and any `Peak_attendant_*` pieces; after `AwkEndF` the Postcard Look returns; root and hurtbox sizes never change), `look.variant_render` (owner on `attendant` in Full Uniform mode; the server-side harness step sets the owner's Player attribute `Variant = "v1"` (render-only check; ownership is covered by `ui.wheel_variant_equip`) → the torso proxy stays `primary` and limb proxies show the base `accent`; `v2` → the torso is `primary` darkened 25 %; switching back to base restores the colours within 1 s), Part 4 `look.gear_silhouette` and Part 7 `look.clip_stage` (both with the silhouette option), `look.face_expressions` (each of the 24 `set` calls is asserted **programmatically**: eye/mouth shape, size and offset of the `Face_UI` frames match the table; then **one** `screen_capture` of a 3×3 grid of sample expressions goes to human review, within the 3-captures-per-task cap).

---

### 6.9 UI system

All UI is built at runtime into `PlayerGui` (StarterGui stays empty, Part 2 §2.2). Every player-visible string comes from `Skin.text`/`Skin.style`/`Skin.move` (§0.4) and uses the player-facing names of §0.12. No image assets: shapes are Frames, glyphs come from built-in fonts. The look is postcard and ticket stock (GDD §12.7): paper panels with ink borders, stamp-shaped rarity badges, and a navigation-chart style for any map view.

#### 6.9.1 Roots, router and screen contract

| Root (`ScreenGui`) | Creator | `DisplayOrder` | `ScreenInsets` | Contents |
|---|---|---|---|---|
| `HB_Hud` | HudController | 10 | `CoreUISafeInsets` | HUD, nameplate-independent overlays, Staff Room panels |
| `HB_Touch` | InputController (Part 3) | 11 | `DeviceSafeInsets` | Touch cluster (§6.9.5) |
| `HB_Screens` | UiController | 20 | `CoreUISafeInsets` | Modal screens |
| `HB_Toasts` | UiController | 30 | `CoreUISafeInsets` | Toasts, banners, subtitles |

All roots use `ResetOnSpawn = false`, `ZIndexBehavior = Sibling`, `IgnoreGuiInset = false`.

Router (Part 6 section of `UiController`): `open(screen: Types.ScreenId, args: { [string]: any }?)` (the optional `args` is added to Part 2's signature; backward compatible), `close(screen)`, `top() -> ScreenId?`. Modal screens (`StyleWheel` (the Charm Ring and the Scrapbook), `Wish`, `Shop` (Sundry's Rack and the Store), `Settings`) call `InputController.setCombatEnabled(false)` while open, except the Charm Ring in `death` mode (the character is KO'd anyway).

Screen module contract (`UI/Screens/<Name>.luau`):

```lua
--!strict
export type ScreenCtx = { args: { [string]: any }, trove: any, close: () -> () } -- trove: Shared/Vendor/Trove instance
export type Screen = {
	id: string,                                  -- Types.ScreenId
	modal: boolean,
	mount: (root: Frame, ctx: ScreenCtx) -> (),
	unmount: () -> (),
	onData: ((sections: { string }) -> ())?,     -- UiController.changed forwarding (Types.Section names)
}
```

**Text refresh** (Part 2 §2.9.2 `Skin.activePack()` / `Skin.onChanged`). One pack ships (`Base`), but the Skin API keeps `onChanged` so a later locale or seasonal text override needs no screen changes. UiController registers one `Skin.onChanged` callback at `start()`. When it fires, UiController unmounts and re-mounts the open screens with the same `args` (screens read every string at mount time), re-renders toasts' pending text, and HudController rebuilds its labels; nameplates refresh themselves (§6.8.4). No screen caches `Skin.text` results across mounts. **Verify:** scenario `ui.skin_refresh` (open the Charm Ring and the HUD; the client harness calls the Part 2 seam `Skin._notify()` → each open screen re-mounts exactly once within 1 s, its labels are rebuilt, and `Skin.missing()` stays empty).

Load rule: UiController **requires screen modules lazily** inside `open()`, never at load. Screens may then require any controller by relative path (`require("../../Controllers/SettingsController")`) without breaking the init-order rule (Part 2 §2.3.3), because they load after every controller has started.

#### 6.9.2 `UI/Create.luau` (typed factory)

```lua
--!strict
-- src/StarterPlayerScripts/UI/Create.luau — typed UI factory. Realm: client. Spec §6.9.2.
local Theme = require("./Theme")
export type Props = { [string]: any }
export type TextStyle = "Header" | "Impact" | "Body" | "Number" | "Hand"
export type TextSize = "S" | "M" | "L" | "XL" | "Huge"
export type BarHandle = { root: Frame, set: (frac: number, drain: boolean?) -> () }
export type RadialHandle = { root: Frame, set: (remainingFrac: number) -> () }
export type CreateApi = {
	new: (className: string, props: Props?, children: { Instance }?) -> Instance,
	frame: (props: Props?, children: { Instance }?) -> Frame,
	panel: (props: Props?, children: { Instance }?) -> Frame,          -- postcard stock: paper fill, 3 px ink UIStroke, UICorner 14, UIPadding 12
	label: (text: string, style: TextStyle, size: TextSize, props: Props?) -> TextLabel,
	button: (spec: { text: string?, style: TextStyle?, size: Vector2?, touch: boolean?, sfx: string?,
		onActivated: () -> () }) -> TextButton,                       -- touch=true → UISizeConstraint MinSize 88×88
	toggle: (value: boolean, onChange: (boolean) -> ()) -> Frame,
	segmented: (options: { string }, index: number, onChange: (number) -> ()) -> Frame,
	slider: (min: number, max: number, step: number, value: number, onChange: (number) -> ()) -> Frame,
	bar: (spec: { width: number, height: number, fill: Color3 }) -> BarHandle,   -- HP bar with white drain
	radial: (diameter: number) -> RadialHandle,                    -- radial cooldown wipe
	rarity: (rarity: string, size: number) -> Frame,               -- stamp-shaped badge: glyph + colour + border (§6.9.4)
	styleChip: (styleId: string, size: number) -> Frame,
	boing: (gui: GuiObject) -> (),                                 -- GDD §12.7 2-frame squash; no-op with Reduced Motion
	canvas: (root: ScreenGui) -> Frame,                            -- reference-unit canvas (§6.9.3)
	restyleAll: () -> (),                                          -- re-apply fonts/text sizes after settings change
}
```

Rules:
- `label` sets `FontFace = Theme.font(style)` (with the dyslexia substitution for Body and Hand when `dyslexiaFont` is on), `TextSize = max(14, round(Theme.TextSize[size] × textScale))` (GDD §16.3: 80–150 %, minimum 14 px at the reference size), and `TextColor3` = ink or paper, whichever has contrast ≥ 4.5 against the parent background (`Theme.contrast`). Labels are registered in a weak table (`setmetatable({}, {__mode = "k"})`) for `restyleAll`.
- `button` plays `sfx.ui.click`, calls `boing`, and is `Selectable` for gamepad (`GuiService.SelectedObject` is set to the first button when a screen opens and `PreferredInput == Gamepad`).
- `radial`: two clipped half-frames (`ClipsDescendants = true`), each holding a full circle with a `UIGradient` whose `Transparency` sequence steps at 0.5; the halves' gradient `Rotation` sweeps 0–180° each. `set(frac)` maps `frac` to the two rotations. A centred `Number` label shows whole seconds remaining when ≥ 1.
- `bar`: fill frame plus a white drain frame behind it. `set(frac, true)` snaps the fill and tweens the drain to the new width over `Perf.Ui.HpDrainSec` (0.4 s, GDD §15.3) after a 0.1 s hold (td).

#### 6.9.3 Layout rules (1334 × 750 first)

- **Reference canvas.** Every root gets `Create.canvas(root)`: a full-screen Frame whose `UIScale.Scale = s = clamp(min(W / 1334, H / 750), 0.5, 2)`, where `(W, H)` is the root's `AbsoluteSize` (already inside the safe insets). The canvas `Size = UDim2.fromScale(1 / s, 1 / s)`, so children anchored to edges with offsets in reference units land correctly at any resolution. Recomputed on `AbsoluteSize` change. GDD §15.1: designed at 1334 × 750 first. On an iPhone SE-class device the GUI viewport may report points (667 × 375); s = 0.5 then preserves the physical design (14 reference px = 14 physical px). Spike S-P6-10 confirms this on the H-12 device.
- **Safe areas.** `CoreUISafeInsets` keeps HUD and screens clear of the top bar and notches; the touch cluster uses `DeviceSafeInsets` to reach the bottom corners [Docs `ScreenGui.ScreenInsets`].
- **Touch targets** ≥ 88 × 88 reference units (44 pt; td).
- **Thumb zones.** Nothing interactive in the left 40 % × bottom 45 % (thumbstick) on touch.
- **Text** ≥ 14 reference px; body 18; headers 24–36; combo 48–72 (Impact).
- **Motion.** Every appear animation is `Create.boing`. Reduced Motion disables boing, panel slides and UI shakes.
- **Simple HUD** (GDD §16.3 "Cognitive"): hides the KO feed, combo counter, zone label, the Quests panel, nameplate name rows and edge indicators; keeps HP, pips, the Peak Meter, hotbar/cluster, Wallet & level, the tide chip (without its pulse) and toasts.

#### 6.9.4 `UI/Theme.luau` (tokens and pure helpers)

Theme is **load-pure** (hex strings and numbers; `Color3`/`Font` values are built lazily inside functions), so Lune can test its helpers.

| Token | Value | Source |
|---|---|---|
| Fonts | Header `LuckiestGuy`; Impact `Bangers`; Body `FredokaOne`; Number `BuilderSansBold`; Hand `PatrickHand`; Dyslexia `Font.fromName("ComicNeueAngular")`, fallback `Enum.Font.Arimo` | GDD §12.7 (a sign-painter display face for headers and a highly legible sans for body text and numbers), §16.3 (dyslexia-friendly option); agent-assets §2.5. All enum members verified in `Font.yaml`; `ComicNeueAngular` is not in the enum, so it is loaded by family name (spike S-P6-12). An original sign-painter display face replaces Header at art lock (final art, GDD §12.7) |
| Ink | `#241C33` (warm navy-plum; never black) | GDD §12.4 (the one ink, tuning default); §13.4 ink rim |
| Paper / PaperShade | `#FFF8EC` / `#F3E6CF` (postcard and ticket stock) | GDD §12.7 |
| Seaside accents | SeaGlass `#8FD3C1`, Coral `#FF7F66`, Lemon `#F7E06B`, Cobalt `#2F5DA8`, Periwinkle `#9FA8D6`, Cream `#FFF3D6`; weathered: Driftwood `#B8A58A`, Rust `#B5583A`, Verdigris `#4E9E8C` | GDD §12.5 (salt and rust), §12.6 (the Elsewhere-on-Sea colour script) |
| Gilt | `#D9A93A` | GDD §12.7 "gold is meaning": one gilt colour, used only for Legendary-and-up stamps and rarity frames, the Gilt Finish, the Peak trim and currency icons |
| HP | Fill `#6BD06B`; low (≤ 30 HP) `#FF6F61`; drain `#FFFFFF`; guard `#4FC3F7`; pip `#F7E06B`; focus `#FFFFFF` | (td); drain white per GDD §15.3 |
| Telegraph swatches | `#FF8A1F` (default "warm orange"), `#D55E00`, `#F0E442`, `#56B4E9`, `#CC79A7`, `#FFFFFF` | GDD §16.3; settings default `#FF8A1F` (Part 2 §2.8.6) |
| Text sizes | S 14, M 18, L 24, XL 36, Huge 64 | GDD §16.3 min 14 |
| Stroke / corner / pad / gap | 3 / 14 / 12 / 8 | (td); GDD §12.7 postcard and ticket stock |

**Rarity encoding** (GDD §8.3 shapes and colours; GDD §16.3: colour is never the only cue). Each badge is a **stamp**: a small paper square with a perforated edge (12 `PaperShade` circles of `UICorner` 0.5 cut round the border), holding the shape glyph in the rarity colour inside a border whose style also varies, so three cues differ. Legendary-and-up badges use a gilt frame (`Theme.Gilt`). Rarity is always shown by its plain name (`rarity.<id>.name`: Starter, Common … Cosmic); the postal zone printed on Wish stamps (`ui.wish.zone_<rarity>`, Part 5 §5.2.13) is flavour only.

| Rarity | Glyph | Colour name and default colour | CVD colour (all three modes; Okabe–Ito) | Border |
|---|---|---|---|---|
| `starter` | ♥ | Stripe Red `#E06A5A` | `#E69F00` | solid 2 px |
| `common` | ○ | Seafoam `#5FC9A8` | `#009E73` | solid 2 px |
| `uncommon` | ◇ | Sky `#5AAEE8` | `#56B4E9` | solid 3 px |
| `rare` | △ | Coral `#FF7F66` | `#CC79A7` | double (2 strokes) |
| `epic` | ☆ | Twilight `#6E5BC4` | `#0072B2` | double, thick |
| `legendary` | ✦ | Sun Gold `#F2B630`, gilt frame | `#E69F00` | dashed (4 segments), gilt |
| `mythic` | ❂ | Deep Sea: `UIGradient` `#06283D` → `#0E6E7E` → `#9FE3DC`, gilt frame | `#F0E442` | gradient stroke, gilt |
| `cosmic` | ✺ | Fog-and-Starlight: `#C9CED6` with an animated gradient `#2B3442` → `#C9CED6` → `#FFF6D8` (Rotation +90°/s; static with Reduced Motion), gilt frame | `#D55E00` | animated stroke, gilt |

Glyph coverage in the built-in fonts is spike S-P6-6. The fallback draws shapes with Frames: circle (UICorner), diamond (square rotated 45°), triangle (three thin rotated frames), star (★ via `Bangers`).

Pure helpers (Lune-tested):

```lua
--!strict
-- Theme pure helpers (no Roblox globals). Spec §6.9.4.
local Theme = {}

local function channel(c: number): number
	return if c <= 0.03928 then c / 12.92 else ((c + 0.055) / 1.055) ^ 2.4
end
local function luminance(hex: string): number
	local r = tonumber(string.sub(hex, 2, 3), 16) :: number / 255
	local g = tonumber(string.sub(hex, 4, 5), 16) :: number / 255
	local b = tonumber(string.sub(hex, 6, 7), 16) :: number / 255
	return 0.2126 * channel(r) + 0.7152 * channel(g) + 0.0722 * channel(b)
end
-- WCAG contrast ratio of two "#RRGGBB" colours (1..21).
function Theme.contrast(a: string, b: string): number
	local la, lb = luminance(a), luminance(b)
	local hi, lo = math.max(la, lb), math.min(la, lb)
	return (hi + 0.05) / (lo + 0.05)
end
-- GDD §8.6 reveal theatre: returns (seconds, skippable). rows = Perf.Ui.Reveal[rarity] = {first, repeat}.
function Theme.revealDuration(rows: { number }, seenBefore: boolean, skipSeen: boolean): (number, boolean)
	if not seenBefore then
		return rows[1], false -- first view: full length, not skippable
	elseif skipSeen then
		return rows[2], true -- repeat view (default)
	end
	return rows[1], true
end
-- Cooldown wipe fraction from absolute ready frame (Part 3 Cd* attributes).
function Theme.cooldownFrac(readyAtF: number, totalF: number, fNow: number): number
	if readyAtF <= fNow or totalF <= 0 then
		return 0
	end
	return math.clamp((readyAtF - fNow) / totalF, 0, 1)
end
return Theme
```

**Style palette helper** (canonical here). `Theme.stylePalette(styleId, variantId?)` applies the Off-Duty Look rule of §6.8.2 to `StyleDef.palette` (Part 4 §4.7.1). The Skin pack holds strings only (Part 2 §2.9.2), so palettes come from the style registry alone. Every presentation colour that depends on a style (proxy rig, Charm Gear, nameplate chip and ring, Charm Ring chips, VFX tints, Sundry's Rack cards) goes through it; nothing reads `StyleDef.palette` directly.

```lua
--!strict
-- Theme.stylePalette — style colours for presentation (canonical). Spec §6.9.4.
-- Excerpt of UI/Theme.luau: the require goes at the top of the module; `Theme` is the module table above.
-- Off-Duty Look rule: §6.8.2 (v1 swaps secondary/accent; v2 and later darken 25 %).
local Styles = require("@game/ReplicatedStorage/Shared/Styles")

export type StylePalette = { primary: Color3, secondary: Color3, accent: Color3 }
function Theme.stylePalette(styleId: string, variantId: string?): StylePalette
	local BLACK = Color3.new(0, 0, 0) -- built inside the function so Theme stays load-pure
	local base: StylePalette = Styles.get(styleId).palette -- Part 4 §4.7.1
	if variantId == nil or variantId == "" then
		return base
	elseif variantId == "v1" then
		return { primary = base.primary, secondary = base.accent, accent = base.secondary } -- swap secondary/accent
	end
	return { -- "v2" and any later id: every colour darkened 25 %
		primary = base.primary:Lerp(BLACK, 0.25),
		secondary = base.secondary:Lerp(BLACK, 0.25),
		accent = base.accent:Lerp(BLACK, 0.25),
	}
end
```

`Color3:Lerp` is verified in the offline docs [Docs `ui/3D-drag-detectors.md` (`COLOR2:Lerp(COLOR1, t)` on `Color3` values)]. The require is a pure module, so Theme stays load-pure.

**Verify:** Lune `tests/core/theme.spec.luau`: `theme.contrast` (ink/paper ≥ 15; the telegraph default vs ink ≥ 4.5), `theme.reveal_durations` (all 7 rarities from `Defaults.Perf.Ui.Reveal` match GDD §8.6), `theme.cooldown_frac`, `theme.touch_no_overlap` (§6.9.5 table: no two circles overlap; all inside 1334 × 750; none in the jump reserve), `theme.style_palette_variants` (base = `Styles.get("attendant").palette`; `v1` swaps secondary/accent; `v2` = each channel × 0.75 within 1e-6), `theme.rarity_tokens` (8 rarities, each with a glyph, a colour, a CVD colour and a border; exactly `legendary`, `mythic` and `cosmic` carry the gilt frame).

#### 6.9.5 HUD (`UI/Screens/Hud.luau`, mounted by `HudController.start()`)

Data: the local view snapshot (`RenderController.localView()`), Player attributes (`Meter`, `Streak`, `MapId`, `EquippedStyle`), the pinned combat config (`Runtime.at(CombatCfg)`), the store (`UiController.state()`). Updated in the `HB_Hud` render step (priority `Last`).

| Element | KBM / Gamepad placement (anchor; offset; size) | Touch placement | Data and behaviour | GDD |
|---|---|---|---|---|
| KO feed | (1, 0); (−16, 16); 420 × 140 | same | `Combat.Feed`: 4 lines, newest on top, 6 s each; killer and victim names (`Players:GetPlayerByUserId` → `DisplayName`; bots `ui.feed.bot`), style chips, a Finish glint from the killer's `Flair`, assists `+n`, streak tag | §15.3 |
| Event banner | (0.5, 0); (0, 16); 640 × 56 | same | `Notify kind = "announce"`: 4 s each, queued. Local call-outs use the same banner: `ui.hud.fully_booked` ("Fully booked!") for every member when a Getaway opens round them (§6.6.1) | §15.3 |
| Menu strip | (0, 0); (16, 16); 5 × 64 + gaps | same | Buttons: Scrapbook (the `StyleWheel` screen in `menu` mode, §6.9.6), Wish, Shop (Sundry's Rack), Settings, Travel (the Staff Room ↔ battle). Wish shows a badge with the `Wishes` count. **Gamepad access:** see "Gamepad menu focus" below | §15.4 |
| Wallet & level | (0, 0); (16, 96); 360 × 56 | same | Row 1: `Skin.text("ui.hud.level", {level})` plus a thin XP bar (`Create.bar`, 200 × 8) filled `xpInto / toNext` from `Pity.accountLevel(account.xpTotal, Runtime.get().Progression.Account)` (full bar at level 100, `toNext = 0`). Row 2: the Shells, Stamps and Messages in a Bottle balances, `ui.hud.treasure {n}`, `ui.hud.shards {n}`, `ui.hud.sparks {n}`, from `PrivateView.currencies` (internal fields `Treasure`, `Shards`, `CosmicSparks`), Number font, each beside a small currency icon with a gilt rim. Updated on `UiController.changed` for `currencies`/`account`; a `+n` float (boing) on increase. Kept in Simple HUD | §0.0 (Account Level "simple UI"); GDD §17.2 |
| Quests | (0, 0); (16, 160); 360 × 132 | same | Header `ui.hud.quests`; 3 rows from `PrivateView.quests.daily` (in order): `Skin.text("ui.quest." .. q.id, {goal = q.goal})`, a progress bar `q.progress / q.goal`, a tick when `q.claimed`. Rewards are auto-claimed by the server (Part 5 §5.5.9), which also sends the `Notify` reward toast `ui.notify.quest_done`; the HUD only animates the row's tick (no second toast). Hidden when `Progression.Quests.Enabled == false`, when the list is empty, and in Simple HUD | GDD §17.5; Part 5 §5.5.9 (`Quests.Enabled = true` in M1-G) |
| Zone label | (0.5, 0); (0, 80); 400 × 28 | (0.5, 0); (0, 150) | `map.<id>.zone.<ZoneId>` from the local root position; fades after 2 s; shown on zone change | §11.1 |
| Tide chip | (0.5, 0); (0, 112); 260 × 24 | (0.5, 0); (0, 178) | Only while `MapId == "hub"`: the current phase `ui.hud.tide_low` / `ui.hud.tide_high` from `Movement.tidePhase(fNow, TidePeriodSec × 60)` (fallback before `Zones` streams in: the hub's `TidePhase` attribute). During the last `Perf.World.Tide.WarnSec` (10 s) before a swap it shows the countdown `ui.hud.tide_in {s}` / `ui.hud.tide_out {s}` with a 1 Hz pulse (no pulse with Reduced Motion or in Simple HUD) (§6.3.8) | §11.2 A |
| HP bar | (0.5, 1); (0, −120); 480 × 28 | (0.5, 0); (0, 88); 420 × 24 (**deviation**, see below) | `HP / maxHp`; white drain over 0.4 s; low colour ≤ 30; shield overlay when a shield buff is active | §15.3 |
| Breakaway pips | (0.5, 1); (0, −96); 2 × (40 × 14) | (0.5, 0); (0, 118) (**deviation**) | `BkP`; a full pip sparkles (boing at 1 Hz); an empty pip shows a refill sweep `BkProg / 1500`; pips are tinted grey while `fNow < BkLockF` (Breakaway lockout, HRP attribute) | §15.3, §6.8 |
| Protection tag | (0.5, 1); (0, −150); 220 × 24 | (0.5, 0); (0, 140) | `ProtF > fNow` → `ui.hud.protected` with whole seconds | §6.2 |
| Hotbar | (0.5, 1); (0, −16); 6 × 72 + 5 × 8 | hidden | Slots M1, 1, 2, 3, 4, Quirk: move name tooltip on hover, key label from the binding (`settings.binds["<Action>.KBM"]` / `["<Action>.Pad"]` override for the current device, else Part 3 defaults; gamepad glyph text `X Y B RB LB →`), radial wipe `Theme.cooldownFrac(Cd<n>, toFrames(def.cooldown), fNow)` from the Peak move set when `AwkF0 > 0`. **Kill-switched move:** when the slot's move id is in `Runtime.get().Features.DisabledMoves`, the slot (and its touch-cluster button) is greyed (60 % transparency) with a lock glyph (Frame-drawn padlock: a rounded body plus a `UICorner` shackle; no image, no text); cosmetic only (the sim refuses the move through `StyleRuntime.disabled`, Part 3 §3.4.2) | §15.3; GDD §18.6 |
| Peak button (`AwakenBtn` on touch) | (0.5, 1); (300, −16); 88 × 88 | in the cluster | The Peak Meter: the ring fills with `Meter / 100` (Player attribute, Part 3). At `AwkRdy` a gilt `UIGradient` sweeps round the ring (the Peak trim; static with Reduced Motion) and the label `ui.hud.go_peak` ("Go Peak!") appears. While at Peak, the ring shows the remaining time `(AwkEndF − fNow) / (AwkEndF − AwkF0 − 72)` | §15.3, §6.9 |
| Combo counter | (1, 0.5); (−260, −120); 320 × 96 | (1, 0); (−16, 170) | Shown when the focus target's `CHits ≥ 3` (GDD §15.3): `ui.hud.combo {n}`, `ui.hud.combo_big {n}` at ≥ 10 (for example "12 HITS! WHAT A HULLABRAWL!"), Impact font growing 48 → 72; hides 1 s after the combo ends | §15.3 |
| Toasts | `HB_Toasts` (0, 1); (16, −220); 360 wide | (0, 0.5); (16, 0) | `Notify` toast/reward/error: auto-stacking, max 4, 4 s each | §15.3 |
| Subtitles | `HB_Toasts` (0.5, 1); (0, −170); 800 × 32 | (0.5, 0.5); (0, 120) | Bark and cinematic lines | §16.3 |
| Edge indicators | canvas edges, 48 × 48 arrows, max 4 | same | Off-screen Getaways (Realm markers); enemies ≤ 30 studs in an `ATTACK` startup outside the camera's ±100° cone ("attacks from behind"); active `EventZones` | §16.3 |
| Staff Room panels | §6.4.2 | same | Only when `MapId == "training"` | §4.5 |

**Dash/Breakaway swap** (Part 3 §3.10.5): while the local `St ∈ STUN` and `BkP ≥ 1`, the Dash slot (cluster button and hotbar hint) shows `ui.hud.breakaway` with a sparkle border.

**Touch HP and pips deviation (owner sign-off).** GDD §15.3 places HP and Breakaway pips bottom-centre above the hotbar, and GDD §15.1 says mobile defines the layout. On touch, the bottom-centre band between the thumbstick zone (left 40 %) and the cluster (`BlockBtn` left edge at x ≈ 686 of 1334) is only ≈ 150 reference units wide, too narrow for a 420-wide bar. The touch HUD therefore puts them top-centre. This is a recorded deviation from GDD §15.3 that the owner signs off at the H-10 feel gate (§6.14 item 9); if rejected, the fallback is a 140-wide HP bar at (0.5, 1); (0, −40) with pips above it, re-checked by `theme.touch_no_overlap`.

**Gamepad menu focus** (Part 3 §3.10.1: `InputController.init()` disables CoreGui `Backpack` and `PlayerList` and sets `GuiService.AutoSelectGuiEnabled = false`, so `Tab`/`ButtonSelect` no longer toggle the player list or UI-selection mode). The `UI` context gains a Bool action `MenuFocus` (Pad `DPadDown`, free in Part 3's map). Pressing it while no modal screen is open calls `InputController.setCombatEnabled(false)` and sets `GuiService.SelectedObject` to the menu strip's Scrapbook button; the strip buttons are `Selectable` with `NextSelectionLeft/Right` chained (Wish, Shop, Settings, Travel). `ButtonB` (or `DPadDown` again) clears `SelectedObject` and re-enables combat; activating a button opens its screen as usual (modal screens keep combat disabled until closed). `ButtonB` is `Move2` in combat, but combat contexts are disabled while the strip has focus, so it cannot fire a move. `DPadDown` is not used for anything else (Quick-Chat is a future hook, §6.15).

**Mobile touch cluster** (`HB_Touch`; centre offsets from the **bottom-right** of the canvas in reference units; negative = left/up). Buttons are created at runtime with `Name` = the Part 3 touch id and bound with `InputController.bindTouch(action, button)` (Part 3 §3.10.3, `InputBinding.UIButton` [Docs]). The layout editor and the one-hand preset are deferred (GDD §15.2, §16.3).

| Button (`Name`) | Action | Diameter | Centre | Notes |
|---|---|---|---|---|
| `PunchBtn` | M1 | 168 | (−300, −150) | "Big Punch button" |
| `Move1Btn` | Move1 | 96 | (−465, −150) | Ring radius 165 around Punch at 180° |
| `Move2Btn` | Move2 | 96 | (−417, −267) | 135° |
| `Move3Btn` | Move3 | 96 | (−300, −315) | 90° |
| `Move4Btn` | Move4 | 96 | (−183, −267) | 45° |
| `QuirkBtn` | Quirk | 72 | (−398, −386) | "Small button above the move ring" |
| `AwakenBtn` | Awaken | 104 | (−199, −395) | The Peak button: "glowing meter button (top of cluster)"; Peak Meter ring as on the hotbar |
| `BlockBtn` | Block | 96 | (−600, −90) | Hold; with `holdToToggleBlock` a tap toggles (§6.9.11) |
| `DashBtn` | Dash | 96 | (−600, −220) | Becomes Breakaway in STUN |
| `LauncherBtn` | Launcher | 80 | (−410, −62) | Visible only while the local `St == M1` with `Sub == 3`, or within the chain window after M1-3. A tap toggles `InputController.fire("Launcher", true)`; it resets to false when the Ender starts or the chain window closes (GDD §15.2) |
| `ShiftLockBtn` | UI `ShiftLock` | 64 | (−60, −470) | Touch shift-lock toggle (`AimLock`, Part 3) |
| `CharmBtn` | UI `StyleWheel` | 64 | (−60, −390) | The touch Charm button (GDD §15.2): opens the Charm Ring in `totem` mode, as Tab / View do (§6.9.6) |
| Jump | Jump | — | jump reserve x [−210, 0] × y [−210, 0] | Our `Jump` action's `Touch.UIButton` is bound to the default `JumpButton` found with `PlayerGui:FindFirstChild("JumpButton", true)` once `IsLoaded` (the docs' own pattern [Docs `ui/position-and-size.md`]). Spike S-P6-3; fallback: our own `JumpBtn` (120) at (−105, −105) and hide the default one |

After layout, if the default `JumpButton`'s absolute rectangle intersects any cluster button, the whole cluster shifts left by the overlap + 16. `HB_Touch.Enabled = (UserInputService.PreferredInput == Enum.PreferredInput.Touch)`; the hotbar shows otherwise. Both update on `PreferredInput` changes.

**UI input context.** Part 3 assigns Emote and the Charm Ring (internally the Style Wheel) to "Part 6's `UI` context". InputController (Part 6 section) creates a client-local `InputContext` named `UI` under `Players.LocalPlayer` (`Priority = 1000`, `Sink = false`) with four Bool actions: `StyleWheel` (opens the Charm Ring in `totem` mode, §6.9.6; KBM `Tab`, Pad `ButtonSelect`, GDD §15.2 "View"; usable because Part 3's `InputController.init()` disables the CoreGui PlayerList and UI auto-selection first), `ShiftLock` (KBM `LeftShift`), `Emote` (KBM `B`, Pad `DPadUp`; M1-G only shows the toast `ui.toast.emotes_soon`; the emote wheel is a future hook, §6.15) and `MenuFocus` (Pad `DPadDown`; above). Touch uses the menu-strip buttons, `ShiftLockBtn` and `CharmBtn` directly. It reacts to `InputAction.Pressed`. Whether a client-created context fires under Server Authority is spike S-P6-7; fallback: `UserInputService.InputBegan`.

**Verify:** scenarios `hud.hp_drain` (dummy deals 15 → fill snaps; drain reaches the fill within 0.5 ± 0.05 s), `hud.cooldown_wipe` (use `attendant.1` → its radial fraction is 1.0 ± 0.02 at use and 0 at `Cd1`), `hud.meter_ring` (`Dev.Command setMeter 100` → the gilt gradient and the `ui.hud.go_peak` label present), `hud.breakaway_swap` (hitstun → the Dash label reads `ui.hud.breakaway`), `hud.touch_binding` (touch emulation: `Players.LocalPlayer.Combat.M1.Touch.UIButton == PunchBtn`), `hud.combo_counter` (3-hit M1 string on a dummy → "3" shown), `hud.simple_preset`, `hud.wallet_level` (`Dev.Command grant Treasure 100` (the internal Shells field) → the Shells label rises by 100 after the next delta; `Dev.Command setLevel 2` → the level label reads `Skin.text("ui.hud.level", {level = 2})`), `hud.quests_progress` (fresh profile on a battle map; drive row 1 to its goal with the Part 5 seam that matches its id — `simActivity {sec, dmg}` for `active_minutes`/damage quests, `simKo` for `kos` — then `flushProgress {}` → its bar reads 1.0, the row shows a tick and exactly one `ui.notify.quest_done` toast appears; Simple HUD hides the panel), `hud.move_lock_glyph` (`setConfig Features {"DisabledMoves":["attendant.1"]}` → slot 1 shows the lock glyph within 1 s); `mcp.gamepad_menu_nav` (gamepad emulation: `DPadDown` → `GuiService.SelectedObject` is the Scrapbook button; one DPad-right → the Wish button; `ButtonA` opens Wish; after closing it, `ButtonB` clears focus, and the next `ButtonB` press (Move2) leaves Idle). `hud.tide_chip` (on the hub with `Dev.Command setTidePeriod 20`: the chip reads `ui.hud.tide_low` or `ui.hud.tide_high` matching `Movement.tidePhase`, and `ui.hud.tide_in`/`ui.hud.tide_out` with a whole-second count during the 10 s before each swap; hidden on the Staff Room). This HUD also relies on Part 7's `mcp.tab_no_playerlist` (Part 7 §7.4.10, M0: CoreGui `PlayerList` and `Backpack` disabled and `GuiService.AutoSelectGuiEnabled == false` after `Tab`; once the `StyleWheel` screen exists, `Tab` also opens the Charm Ring). MCP `screen_capture` at the Studio device emulator "iPhone SE" profile (human selects the profile; H-12) for review.

#### 6.9.6 Charm Ring and Scrapbook (`UI/Screens/StyleWheel.luau`)

GDD §15.4: owned Charms by rarity, free-rotation marks, Mastery level, the equipped Finish, a per-style Full Uniform toggle; opens on death with a 3 s auto-confirm of the last Charm. One screen module serves both player-facing views (the screen id stays `StyleWheel`, §0.3):
- the **Charm Ring** (`death`, `totem`, `trial` modes): the quick wheel for swapping Charms, titled `ui.wheel.title_ring`;
- the **Scrapbook** (`menu` mode): the collection book of all 18 Charms, titled `ui.wheel.title_scrapbook`.

| Mode (`args.mode`) | Player-facing view | Opened by | Lists | Confirm action |
|---|---|---|---|---|
| `menu` | Scrapbook | menu strip (Scrapbook button) / [All] in a Charm Ring | All 18 styles in rarity rows | `Style.Equip` → applies at respawn (`NextStyle` shown) |
| `death` | Charm Ring | local `St` enters `KO` | Owned, implemented styles only (compact strip) + [All] button (opens the Scrapbook) | Same. Auto-closes after `Perf.Ui.StyleWheelAutoConfirmSec` (3 s, GDD §15.4) with no request when untouched; any touch cancels the countdown |
| `totem` | Charm Ring | Charm Rack prompt, or anywhere the `StyleWheel` action (Tab / View, GDD §15.2) or the touch Charm button `CharmBtn` (§6.9.5) | Owned, implemented styles + [All] button (opens the Scrapbook) | `Style.Equip`: within range of a Charm Rack the server infers a rack equip and applies it now (§6.3.6); elsewhere it applies at respawn (`NextStyle` shown) |
| `trial` | Charm Ring | Staff Room panel / Staff Room Charm Rack | `Styles.implemented()` with a "Trial" badge | `Style.Equip` then `World.Travel {"training"}` (§6.4.2) |

Scrapbook layout (rows by rarity; 1334 × 750): a centred panel 1180 × 640 drawn as an open scrapbook page. Rows top to bottom: Starter (2), Common (3), Uncommon (3), Rare (3), Epic (3), Legendary (2), Mythic (1), Cosmic (1). Each row has its rarity stamp (§6.9.4) at the left and postcard tiles of 112 × 128. A detail pane (360 wide) on the right. The Charm Ring modes use the same tiles in a single compact ring or strip.

| Tile state | Look | Source |
|---|---|---|
| Owned, implemented | style chip in the palette, name, `M<level>`, Finish glint | `PrivateView.styles[id]`; level from `masteryXp` with `Progression.Mastery` (XpBase + XpPerLevel·m curve, Part 2 §2.6.2), computed by the same formula as `ProgressionService.masteryLevel` |
| Owned, `implemented = false` | chip at 60 % + **In development** badge (`ui.wheel.in_dev`); Equip disabled | §0.10 |
| Trial | chip + `ui.wheel.trial` badge | `MapId == "training"` and `Progression.TrainingTrialAllStyles` |
| Free rotation | `ui.wheel.free` badge | `Progression.FreeRotation.Enabled` (true in M1-G: badge shown for rotation styles; Part 5 §5.1.4, DES-6) |
| Not owned | an empty postcard slot (palette primary at 20 %, "?"), rarity stamp, hint `ui.wheel.get_hint` (Wishes or Sundry's Rack) | |
| Cosmic, not owned | "???" (GDD §8.3: secret) | |
| Equipped / next | thick ink ring / dashed ring | `EquippedStyle` / `NextStyle` |

Detail pane: name and epithet (`Skin.style(id)`), move list (`Skin.move` for 1–4, q; tags as chips), Finish picker (owned Finishes → `Style.SetFlair`), **Off-Duty Look picker**, **Full Uniform quick toggle** (segmented Inherit / On / Off → `Appearance.Set {styleOverride = {styleId, value}}`; current from `settings.characterLookByStyle[id]`), **Mastery and Stamps**, [Equip].
- **Off-Duty Look picker** (owned styles; GDD §8.7, §9.3): label `ui.wheel.variant`; a row of swatch chips: `ui.wheel.variant_base` (the usual look, `variantId = ""`) plus one chip per owned Off-Duty Look in `PrivateView.styles[id].variants` (sorted `v1`, `v2`, …), each chip drawn in `Theme.stylePalette(id, vid)` with its name `Skin.text("variant.<styleId>.<vid>.name")`. Selected = `styles[id].equippedVariant` (thick ink ring). Tapping a chip sends `Style.SetVariant {styleId, variantId}` (Part 2 §2.7.4 row 6); on success the owner's `Variant` attribute changes and the look refreshes (§6.8.3). Hidden when the style owns no Off-Duty Looks.
- **Mastery and Stamps** (owned styles; GDD §9.2): `M<level>` with an XP bar (`Pity.masteryLevel` / `Pity.masteryXpToReach` with `Progression.Mastery`), the Stamps line `ui.wheel.shards {n}` (`currencies.Shards`), and while `level < Progression.Mastery.ShardCapLevel` (20) a stepper (1 … `ceil((Pity.masteryXpToReach(20) − masteryXp) / Progression.Mastery.ShardXp)`, clamped to the balance) with [`ui.wheel.convert`] → `Mastery.SpendShards {styleId, shards}`; the response `MasterySpendRes` shows a reward toast with `xpGained` and the new `level`. At level ≥ 20 the stepper is replaced by the disabled text `ui.wheel.mastery_combat_only` (Mastery 21–30 is earned in combat).

**Verify:** scenarios `ui.wheel_death_autoconfirm` (KO → the Charm Ring opens within 0.2 s; untouched → closes at 3 ± 0.1 s; no `Style.Equip` sent), `ui.wheel_equip_next` (pick `lifeguard` → `NextStyle == "lifeguard"`; after respawn `EquippedStyle == "lifeguard"`), `ui.wheel_in_dev_badge` (grant `skimstone` via `Dev.Command giveStyle` → badge shown, Equip disabled), `ui.wheel_cl_toggle` (Full Uniform Off for `attendant` → `settings.characterLookByStyle.attendant == false` in the next delta), `ui.mastery_level_matches_server` (10 XP values: the client formula equals `ProgressionService.masteryLevel` via `execute_luau`), `ui.wheel_variant_equip` (read cosmetic slot 4's `itemId` `<styleId>/<vid>` from `Shop.Get`; `Dev.Command giveStyle <styleId>`, `grant Treasure` and buy slot 4 as in `ui.shop_buy_variant`; equip `<styleId>` → the picker shows the usual look + `<vid>`; tapping `<vid>` → `styles[<styleId>].equippedVariant == <vid>` in the next delta and the owner's `Variant` attribute equals `<vid>`; tapping the usual look → `""`), `ui.wheel_spend_shards` (`Dev.Command setMastery attendant 0`, `grant Shards 400` → the stepper max is `ceil(masteryXpToReach(20) / 60)`; converting the max → level 20, the remaining Stamps equal 400 minus the Stamps the server reports spent, and the stepper is replaced by `ui.wheel.mastery_combat_only`), `ui.wheel_titles` (open in `menu` mode → the title reads `ui.wheel.title_scrapbook`; on a KO → `ui.wheel.title_ring`; under touch emulation, tapping `CharmBtn` → `ui.wheel.title_ring` in `totem` mode).

#### 6.9.7 Wish screen (`UI/Screens/Wish.luau`)

GDD §8 / §15.4: a Wish (×1) and a Mailbag (×10), the odds button always visible, the Guaranteed Delivery counters, and the reveal theatre. In the fiction a Wish is a "Wish you were here!" postcard posted through Pip-Pip the postbox; a staff member writes back with their Charm. The screen shows postcards, stamps and Pip-Pip, never gambling imagery (GDD §16.1).

Layout (modal panel 1240 × 680): left, the Pip-Pip placeholder (a rounded Frame 420 × 420 in coral-and-cream beach-hut stripes (a `UIGradient` with hard stops), a mouth Frame, and a flag Frame that pops up with a boing on each Wish); bottom centre, [Wish ×1] and [Mailbag ×10] (`ui.wish.make_1`, `ui.wish.make_10`) with costs and the `Wishes` balance; top right, **[Odds]** (always visible, never disabled); right column, the Guaranteed Delivery counters; bottom left, the Lucky Stamp status (`passes.lucky_jar`).

**PRI-restricted wallet** (Part 5 §5.2.10, §5.2.14; GDD §17.1). When `PrivateView.policy.paidRandomRestricted` is true, the balance shown is the **spendable** count `currencies.Wishes − wishStats.paidUnspent` (never below 0), and when `paidUnspent > 0` a second line `Skin.text("ui.wish.locked_paid", {n = paidUnspent})` appears under it. [Wish ×1] / [Mailbag ×10] are disabled while spendable < 1 / < 10. A response `{ok = false, err = "restricted", detail = "paid_wishes_locked"}` is shown as that same locked-paid line (pulsed once), not as the generic `ui.error.restricted` toast. Unrestricted players see the plain `Wishes` balance.

| Guaranteed Delivery row | Value | Max | Note |
|---|---|---|---|
| Epic+ | `pity.sinceEpicPlus` | Epic hard pity (Part 5 `Wish.Pity`) | GDD §8.4: 30 |
| Legendary+ | `pity.sinceLegendaryPlus` | Legendary hard pity | 120; soft-pity marker from 60 with the live rate `OddsView.legendaryPlusPctNow` |
| Mythic+ | `pity.sinceMythicPlus` | Mythic hard pity | 400 |
| Messages in a Bottle | `currencies.CosmicSparks` | `Wish.CosmicSparksNeeded` (10) | [Craft] enabled at 10 → `Wish.CraftCosmic` (10 Messages in a Bottle = a guaranteed Cosmic) |

Part 5 owns the `Wish.Pity` key names; this screen reads them by those names (§6.15).

**Odds panel** (modal over the screen): from `Wish.GetOdds` → `OddsView`, refetched on open and after every Wish (GDD §8.8 "update live"). Sections: per-tier rows (stamp badge, plain rarity name, %), expandable per-style rows (`OddsStyle.pct`, owned tick, **In development** badge when `implemented = false`; the **Cosmic** style row shows `ui.wish.secret` ("???") instead of its name and chip unless the viewer owns it (GDD §8.3: secret until the first global pull), with its % still printed so the odds list every item and sum to 100 %); the Finish table (`OddsView.flairs`); below it the fake-out disclosure `Skin.text("ui.wish.fakeout_note")` shown as-is with no arguments (GDD §8.6: fake-outs are pre-announced; the pack text carries no `{pct}` token); a "Current effective odds" block: Legendary+ now `legendaryPlusPctNow`, Lucky Stamp active, and Guaranteed Delivery. A Studio check logs `[HB][UI] ERROR odds_sum` when the style %s or Finish %s do not sum to 100 ± 0.01.

**Making a Wish.** Capture `seenBefore[r] = flags.seenReveals[r]` from the store **before** sending `Wish.Make {count}`. On the response (`WishBatchResult`):

| Case | Presentation |
|---|---|
| Wish (×1) | Reveal `results[1]` |
| Mailbag (×10) | Reveal `results[best]` first, then a 5 × 2 grid of postcards (best highlighted); duplicates faded (`CanvasGroup.GroupTransparency = 0.4`) with `ui.wish.shards {n}` ("+{n} Stamps"); Finish labels; **In development** badges (GDD §8.6) |
| Error | `ui.error.<err>` toast (`insufficient` → [Shop] button; never a Robux offer right after a lost fight, GDD §8.8: the [Shop] button is omitted while `now − lastLocalKoAt < Economy.NoOfferAfterLossSec`, §6.9.10); `restricted` + `paid_wishes_locked` → the locked-paid line above |

**Stamp-first reveal** (Part 5 §5.2.13 data hook; GDD §8.6). Every reveal is a postcard coming back out of Pip-Pip's mouth, staged from the `WishResult` fields in this order:
1. **The stamp lands** (`rarity`): at `t = 0` the card back drops out of Pip-Pip's mouth and the stamp *thunks* onto it over `Perf.Ui.StampLandSec` (0.25 s) with `sfx.wish.thunk`: the stamp badge of §6.9.4 in the rarity colour, the plain rarity name (`rarity.<id>.name`) and the postal zone (`ui.wish.zone_<rarity>`, flavour only). The rarity is readable from the stamp before the card flips.
2. **Re-stamp** (`fakeOut`; Part 5 sets it on Epic results only): a Rare stamp lands first; at `Perf.Ui.FakeOutUpgradeAt` (50 %) of the duration an Epic stamp is struck over it with a second *thunk*. It only ever upgrades, and the Odds panel discloses it.
3. **The card flips** (`styleId`, `flair`, `isNew`, `shards`, `variantId`) at `Perf.Ui.FlipAt` (60 %) of the duration, so always after the re-stamp: the staff member's postcard with the Finish overlay; "New!" (`ui.wish.new`) or "+{n} Stamps" (`ui.wish.shards`); an Off-Duty Look tag when `variantId` is set.
4. **Length** (`firstReveal`, Quick Post): below.

Reveal theatre per rarity (durations from `Perf.Ui.Reveal`, GDD §8.6 first / repeat; `Theme.revealDuration`):

| Rarity | First / repeat (s) | Placeholder theatre (on top of the stamp-first steps) |
|---|---|---|
| common, uncommon | 1.2 / 0.5 | Plain stamp *thunk*, card flip (UIScale X 1 → 0 → 1), light chime (`sfx.wish.chime`, 2 notes) |
| rare | 2.0 / 0.8 | + a coral shimmer sweeps across the card, chip pose (boing), 3 notes |
| epic | 3.0 / 1.2 | + panel tint (Twilight), a dark-chip tease before the flip, the possible re-stamp; local announcement (Part 5 Notify), 4 notes |
| legendary | 4.5 / 2.0 | + the **gilt stamp**: the stamp lands in `Theme.Gilt` with a glint sweep, and Pip-Pip's flag pops up; bark `ui.vo.legendary`; server announcement, 5 notes |
| mythic | 6.0 / 3.0 | + the panel turns to **night** (the Deep Sea gradient) and, before the flip, a **hump surfaces** across the bottom of the panel and sinks again (three half-ring Frames rise and fall over 1.2 s); cross-server announcement, 6 notes |
| cosmic | 8.0 / 4.0 | + a **ferry horn** (`sfx.wish.horn`), **sea fog** rolls across the panel (three white gradient Frames drifting at transparency 0.3–0.6), and the Music is muted (`duck("cosmic")`); global announcement. In M1-G the first Cosmic on a server also starts the Cosmic Arrival on the hub (Part 5 §5.2.18; the design is the first Cosmic pull anywhere, GDD §8.6; render in §6.5.1) |

Skip: a tap skips **only** when `seenBefore[rarity]` was true; first views play in full (GDD §8.6). `skipSeenReveals` (default true) selects the repeat duration. Photosensitivity: no element flashes more than 3 times per second (GDD §16.3). Reduced Motion: the hump, the fog and the shimmer are static frames; the stamp still lands first.

**Quick Post** (the `quick_wish` pass, `PrivateView.passes.quick_wish`; Part 5 §5.6.5; GDD §17.3 "instant reveals, auto-Mailbag"). While the pass is owned:
- Reveals of a rarity already seen (`seenBefore[r]`) use duration 0: the stamped result card (×1) or the Mailbag grid appears directly, with no theatre. A rarity's **first** view still plays in full once (GDD §8.6 "skip after the first time you've seen a given rarity's reveal"); the pass removes every later delay.
- An **[Auto Mailbag]** toggle (`ui.wish.auto10`) appears next to [Mailbag ×10]. While on, it sends `Wish.Make {count = 10}` repeatedly, each request only after the previous response and ≥ 2.0 s after the previous send (`1 / perSec` of the remote's `{burst = 2, perSec = 0.5}` bucket, Part 2 §2.7.4 row 8, so a long auto run never hits `rate_limited`), while the spendable balance is ≥ 10. It stops on any tap, when the screen closes, on any `legendary` or rarer result, on a style the player did not own before the batch, and on any error response. Each batch is a normal `Wish.Make`; nothing is batched or skipped server-side.
- Without the pass neither behaviour exists (the toggle is not created).

**Contract for Part 5** (§6.13 item 8): WishService sets `flags.seenReveals[rarity] = true` for every rarity in the batch inside the same `DataService.update` as the roll.

**Verify:** scenarios `ui.wish_odds_live` (open Odds; `Dev.Command setPity 0 70 0` → the Legendary+ now value updates on the next fetch; the fake-out note label exists with the text `Skin.text("ui.wish.fakeout_note")`; on a profile without the Cosmic style its row reads `Skin.text("ui.wish.secret")` and no label contains the Cosmic style's `Skin.style` name), `ui.wish_stamp_first` (a ×1 reveal: the stamp Frame is visible in the rarity colour, with `rarity.<id>.name` and `ui.wish.zone_<rarity>`, before the card's flip starts; a Legendary result's stamp uses `Theme.Gilt` and a Rare one does not), `ui.quick_wish_instant` (`Dev.Command setPass quick_wish true`, `grant Wishes 2`; after one ×1 whose rarity was already seen, the second ×1 shows its stamped result card ≤ 0.1 s after the response; [Auto Mailbag] exists only with the pass), `ui.wish_restricted_wallet` (`simulatePurchase wishes_10` while unrestricted, then `setPolicy true` on a profile with 0 free Wishes → balance label 0, the locked line shows 10, ×1 and ×10 are disabled), `ui.wish_reveal_first_unskippable` (fresh profile, `Dev.Command grant Wishes 1`, a tap during the first common reveal does not skip; the second common reveal skips on tap and lasts ≤ 0.5 s), `ui.wish_10x_grid` (10 postcards, best first), `ui.wish_no_gambling_terms` (Part 8 lint: Wish screen Skin values contain none of "slot", "spin", "jackpot", "roulette", "casino", "bet"; Pip-Pip's mouth is never called a "slot" in a player-visible string).

#### 6.9.8 Sundry's Rack and the Store (`UI/Screens/Shop.luau`)

Tabs **[Sundry's Rack]** (`ui.shop.tab_rack`) and **[Store]** (`ui.shop.tab_store`).
- **Header** (both tabs): the Shells balance `Skin.text("ui.shop.balance", {n = currencies.Treasure})`, updated on `currencies` deltas, so players always see what they can spend.
- **Sundry's Rack** (GDD §8.5; Part 5 §5.4): `Shop.Get` → `ShopView`. Sundry is a woken spinning postcard rack under a striped awning, plain-spoken and never pushy. Placeholder: a header with a striped awning (a `UIGradient` with hard stops) over a tall rack Frame; on each refresh the rack's card rows slide one step sideways, as if it had spun (static with Reduced Motion). No luck is involved: the stock is deterministic per player and window.
  - **Style slots 1–3** (`ShopSlot.kind == "style"`): postcard cards 300 × 380: chip, rarity stamp, name, price in Shells, Owned/Bought state, [Buy]. When `Styles.get(itemId).implemented == false` the card carries the **In development** badge (`ui.wheel.in_dev`, §0.10), and [Buy] first opens a confirm modal with `ui.shop.confirm_in_dev` ("This Charm is still being made. Collect it now and play it when it's ready?", Part 5 §5.12 item 14); only its confirm sends `Shop.Buy`.
  - **Cosmetic slots 4–7** (`ShopSlot.kind == "cosmetic"`, `itemId = "<styleId>/<variantId>"`, Part 5 §5.4.3): Off-Duty Looks, shown while `Shop.CosmeticSlotsEnabled` (true in M1-G): cards 220 × 300 with the style chip drawn in `Theme.stylePalette(styleId, variantId)`, `Skin.text("variant.<styleId>.<variantId>.name")`, price in Shells, Owned/Bought state, [Buy] → `Shop.Buy {slot, rotationId, price}`. Empty slots (no `ShopSlot` at that index) are hidden. The "Coming soon" placeholder (`ui.shop.coming_soon`) is shown only while `CosmeticSlotsEnabled == false`. A bought Off-Duty Look is worn from the Scrapbook's Off-Duty Look picker (§6.9.6).
  - A refresh countdown `refreshAt − workspace:GetServerTimeNow()`; Sundry's line `ui.shop.sundry_line_<n>` (n picked uniformly from 1 to the highest n for which `Skin.has` is true; Part 5 ships 1…6). Buy → `Shop.Buy {slot, rotationId, price}`: `stale` → refetch and toast `ui.error.stale`; `insufficient` → toast.
- **Store** (GDD §17.3, §8.8): `Store.Get` → `StoreView`. Wish packs and passes (Lucky Stamp, First Class, Quick Post) with `priceRobux`, a **Mock** badge when `StoreView.mock`; items with `hidden = true` are never shown (PRI items are hidden when restricted, Part 2 §2.11.4). [Buy] → `Store.Prompt {key, entryPoint = "shop_tab"}`: `prompted` → wait for the delta; `needs_confirm` → confirm modal (`ui.store.confirm_4th`) then resend with `confirmed = true`; `mock_granted` → reward toast. A "spent this week" line shows `Skin.text("ui.store.spent_week", {robux = StoreView.spentThisWeekRobux})` (R$, UTC Monday week; GDD §8.8 guardrail). `StoreView.wishBuysLast24h` drives only the 4th-purchase confirm modal.

**Verify:** scenarios `ui.shop_buy_flow` (`grant Treasure 20000`; buy slot 1 → the card shows Bought; the header Shells balance decreases by the price), `ui.shop_in_dev_badge` (a rotation whose style slot holds an unimplemented style, e.g. `skimstone` → the card shows `ui.wheel.in_dev`; [Buy] opens the `ui.shop.confirm_in_dev` modal and sends no `Shop.Buy` until confirmed), `ui.shop_buy_variant` (`grant Treasure`; buy cosmetic slot 4 → the card shows Bought; `styles[<styleId>].variants[<variantId>] == true` in the next delta; empty cosmetic slots are not rendered), `ui.store_restricted_hidden` (`Dev.Command setPolicy true` → no Wish pack or Lucky Stamp card), `ui.store_confirm_4th` (4th mock Wish purchase in 24 h → the confirm modal appears), `ui.store_week_spend` (mock purchases of `wishes_10` then `wishes_55`, as in Part 5 `store.week_spend` → the spent-this-week line reads `Skin.text("ui.store.spent_week", {robux = 548})`).

#### 6.9.9 Settings (`UI/Screens/Settings.luau`)

Left tabs; right `ScrollingFrame` (`AutomaticCanvasSize = Y`) with a `UIListLayout`. Every control writes through `SettingsController.set(key, value)` or `setAppearance(req)` (Part 2 §2.3.7), so the change applies locally at once and is confirmed by the next delta.

| Tab | Controls (setting key) | Applied by |
|---|---|---|
| Appearance (§0.4, GDD §12.3) | **Full Uniform** (`characterLook`, label `ui.settings.character_look`; default Off); per-style overrides list (owned styles: Inherit/On/Off → `styleOverride`); **Your look** (`ownLook`: `ui.settings.own_look_postcard` Postcard Look / `ui.settings.own_look_avatar` Roblox Avatar; "applies on respawn", Part 5 §5.7.2); **Charm Gear** (`showStyleGear`, label `ui.settings.style_gear`); **Show other players as** (`othersAppearance`: Their choice / Always Full Uniform / Always their own looks / Simplified, GDD §12.3); **Postcard Look editor**: body style ◀▶ (4; label `Skin.text("ui.postcard.body." .. Appearance.PostcardStyles[i].id)`), skin tone swatches (`Appearance.PostcardSkins`), colour swatches (12, `Appearance.PostcardColors`, Part 5 §5.10), face 1–6, hat 0–6, outfit 0–4 → `setAppearance({postcard = spec})`, with a live preview in a `ViewportFrame` (a local `LookRig_R15` clone at the body style's proportions, rotating at 30°/s, static with Reduced Motion) | AppearanceController |
| Graphics | `graphicsTier` (Auto/Low/Mid/High); `outlineThickness` (0–2; graybox: nameplate stroke only, hulls are future) | VfxController.setTier, AppearanceController |
| Accessibility (GDD §16.3) | `reducedMotion`, `reducedVfx`, `cameraShake` (0–1, step 0.1), `colorblind` (None/Deuteranopia/Protanopia/Tritanopia), `telegraphColor` (6 swatches), `textScale` (0.8–1.5, step 0.1), `dyslexiaFont`, `simpleHud`, `subtitles`, `misprintCalm` (the Misprint Finish renders static), `holdToToggleBlock`, `holdToToggleRun` (`ui.settings.hold_toggle_run`), `damageNumbers` (`ui.settings.damage_numbers`; GDD §15.3), `skipSeenReveals` | §6.9.11 |
| Audio | `volMaster`, `volMusic`, `volSfx`, `volVo`, `volUi` (0–1); `quietBarks` (`ui.settings.quiet_barks`; GDD §14.5: own barks only when on) | SfxController |
| Controls | Rebind rows for `M1, Move1–4, Quirk, Awaken, Block, Dash, Jump, Run` × KBM/Pad (the `Awaken` row is labelled with the Peak name): [Change] captures the next `UserInputService.InputBegan` key (KBM column: keyboard/mouse inputs only; Pad column: gamepad inputs only) → `SettingsController.set("binds.<Action>.<Device>", KeyCode.Name)` with `<Device>` = `KBM` or `Pad`; [Reset] → `""` (Part 3 default). Duplicate keys within a column are refused with a toast. Mobile: `ui.settings.layout_editor_soon` | **SettingsController** (single owner, Part 2 §2.3.7 rule 6) applies `binds.<Action>.<Device>` to `Player.Combat.<Action>.<Device>.KeyCode`; InputController never writes binding KeyCodes (Part 3 §3.10.1) |
| Safety (`ui.settings.tab_safety`; GDD §16.2) | The list of other players in the server, each with a [`ui.report.button`] button that opens the Report flow (§6.9.12) | UiController |

**Verify:** scenarios `ui.settings_roundtrip` (each key set via the UI → the store shows the value after the next delta; `textScale 1.5` → a 14 px label becomes 21), `ui.rebind_m1` (binding M1 to `E`, the Charm Rack prompt key, is refused with `ui.toast.key_in_use`; binding it to `V` → the store holds `binds["M1.KBM"] == "V"` and `Players.LocalPlayer.Combat.M1.KBM.KeyCode == Enum.KeyCode.V`; then rebinding Pad M1 through a `binds.M1.Pad` set leaves `M1.KBM.KeyCode == V`, proving the two devices are stored separately), `ui.postcard_preview` (body style → `round`, skin swatch 2, colour swatch 1, outfit 0: preview head/arms `Appearance.PostcardSkins[2]`, torso `#F2C14E`, legs `#F2C14E` × 0.65; the torso proxy is `Perf.Looks.BodyScale.round.core` times the classic width; Part 5 §5.7.1), `ui.settings_new_toggles` (`quietBarks`, `damageNumbers`, `holdToToggleRun` each toggle via the UI and read back from the store after the next delta).

#### 6.9.10 Results, notifications, subtitles, FTUE-lite

- **Results card** (`UI/Screens/Results.luau`; not modal): on `Life.Summary`, a 520 × 200 postcard at top-centre for 6 s or until respawn, headed `ui.hud.checked_out` ("Checked out!") when the local player was KO'd. Killer chip and name (or `ui.results.environment`), damage dealt and taken, KOs, assists, life time, Shells / XP / Mastery XP earned. When the killer is a **player** (not a bot, not the environment, not self), the killer chip carries a small [`ui.report.button`] button that opens the Report flow with that player preselected (§6.9.12). It never contains Store entry points (GDD §8.8: no offers right after a lost fight).
- **Notify** (Part 2 §2.7.4 row 22): `toast` / `reward` → toast stack; `error` → red toast; `announce` → banner (Mythic/Cosmic announcements use the rarity gradient). Text = `Skin.text(key, args)`; `ttl` overrides 4 s.
- **Subtitles** (GDD §14.5, §16.3): every bark and cinematic line, when `subtitles` is on.
- **FTUE-lite** (GDD §15.5; the full tutorial path is a future hook): while `account.level == 1` and `account.xpTotal == 0`, one-time contextual hint toasts per session: `ui.tip.punch` at spawn, `ui.tip.block` after the first hit taken, `ui.tip.dash` after 20 s, `ui.tip.awaken` when the Peak Meter is first full (`AwkRdy`), `ui.tip.wish` when `Wishes ≥ 1` and out of combat for 10 s (with a [`ui.wish.title`] button that opens the Wish screen).
- **FTUE steps** (Part 5 §5.8.4 `Ftue.Step`, C2S event, rate 5 / 1). UiController sends `Net.client.fire("Ftue.Step", {step = n, skipped = false})` once per milestone, in increasing order (the server ignores non-increasing steps), only while `flags.ftueDone == false`:

  | Step | First time the local player … | Detected by |
  |---|---|---|
  | 2 | sees `ui.tip.punch` | FTUE-lite toast shown |
  | 3 | lands an M1 contact | local `Cnt` 0→1 in an `M1` state (§6.6.1) |
  | 4 | enters `Block` | local `St` enters `Block` |
  | 5 | dashes | local `St` enters `Dash` |
  | 6 | breaks away | local `St` enters `Breakaway` |
  | 7 | goes Peak | local `St` enters `Awakening` |

  Step 1 (cold open) and steps 8–9 are server-side or not emitted in M1-G. A milestone reached out of order (e.g. a Dash before the first Block) is sent when it happens; the skipped lower steps are then never sent for that profile.
- **No offer after a loss** (GDD §8.8 "Wish offers are never shown right after a lost fight"; Part 5 §5.6.4 `Economy.NoOfferAfterLossSec`, 30 s). UiController records `lastLocalKoAt = os.clock()` when the local `St` enters `KO` (presentation clock; not sim). While `os.clock() − lastLocalKoAt < Runtime.get().Economy.NoOfferAfterLossSec`, `ui.tip.wish` and any other **unsolicited** Wish or Store prompt (tip buttons, the `insufficient` → [Shop] shortcut, any future offer banner) are suppressed; a suppressed `ui.tip.wish` is re-evaluated after the window. Screens the player opens deliberately (menu strip) are never blocked.

**Verify:** scenarios `ui.results_card` (`Dev.Command kill` → card within 0.5 s of `Life.Summary`, gone after respawn; with a bot killer no report button; with a player killer in a two-client run the button exists), `ui.notify_kinds` (server fires each kind → the expected container), `ui.no_offer_after_loss` (`grant Wishes 1`; `Dev.Command kill`; respawn and stay out of combat → no `ui.tip.wish` toast within `NoOfferAfterLossSec` of the KO; it appears once after the window plus 10 s out of combat), `ui.ftue_steps` (fresh profile: punch tip, M1 contact, block, dash → `flags.ftueStep` reads 2, 3, 4, 5 in order in the deltas).

#### 6.9.11 Accessibility matrix (GDD §16.3)

| Feature | Implementation | Verify |
|---|---|---|
| Reduced Motion | Camera shake scale 0; no impact frames (→ outline pulse); no screen flashes; no Peak orbit; no UI boing or slides; the reveal's shimmer, hump and fog are static; no Getaway frame slam or tide-line slide; rotating gradients static | `cam.reduced_motion`, `vfx.impact_frame` |
| Reduced VFX | Budget × 0.4; enemy VFX opacity 60 %; telegraphs untouched | `vfx.budget_tiers`, `vfx.telegraph_never_reduced` |
| Photosensitivity | `CameraController.flash` ≤ 3/s, alpha ≤ 0.5 white; the Misprint Finish's calm variant (`misprintCalm`) | `cam.flash_rate` |
| Colour-blind modes | Rarity CVD palette (§6.9.4) + shapes + borders; telegraph swatch default per mode: None `#FF8A1F`, Deut/Prot `#E69F00`, Trit `#D55E00` (applied only while the user has not picked a swatch) | `ui.cvd_rarity` (capture for review) |
| Telegraph contrast | User colour + adaptive ink/white outline ≥ 4.5:1 to the ground | Lune `theme.contrast` |
| Text scale / dyslexia font | `Create.label` rules; `restyleAll` on change | `ui.settings_roundtrip` |
| Audio buses, subtitles, visual sound indicators | SfxController buses; subtitles; HUD edge indicators | `sfx.bus_volumes`, `hud.edge_indicators` |
| Hold-to-toggle block | With `holdToToggleBlock`, InputController clears the `Block` action's `KBM`/`Pad`/`Touch` bindings on the client's own context and instead toggles the `Script` binding (`InputBinding:Fire`) on each press | `input.block_toggle` |
| Hold-to-toggle run | With `holdToToggleRun` (GDD §16.3 "hold-to-toggle for block and run"), a Run press latches `RunAuto` on (InputController fires the `RunAuto` Script binding) until the next Run press, a stop of movement input for ≥ 0.25 s, or any non-NEUTRAL state | `input.run_toggle` (key on: one Run tap while moving → `RunAuto` stays true for 2 s of held movement; a second tap → false) |
| Simple HUD | §6.9.3 | `hud.simple_preset` |
| Damage numbers | Settings key `damageNumbers` (boolean, default `false`; Part 2 §2.8.6): **off by default everywhere** (GDD §15.3); enabled from the Staff Room panel toggle (`ui.training.damage_numbers`) or Settings ▸ Accessibility (`ui.settings.damage_numbers`), both writing the same key. RenderController shows them only while it is true; `Perf.Ui.DamageNumbersDefault = false` is used before settings load. Simple HUD does not override an explicit `true` | `ui.settings_new_toggles`, `training.damage_numbers_toggle` |
| Haptics | Short `HapticEffect` on own contact and on being hit (gamepad/touch only), off with Reduced Motion (§6.6.1) | `render.haptic_gate` |

#### 6.9.12 Report (panel inside `UI/Screens/Settings.luau`)

GDD §16.2: "In-game report (exploits, teaming, harassment)". The server half is Part 5 §5.9.1 (`Report.Submit`, Part 2 §2.7.4 row 17; ModerationService ring; thank-you Notify). The flow is a sub-panel of the Settings screen, so the §0.3 screen list and `Types.ScreenId` are unchanged: it is opened with `UiController.open("Settings", {tab = "safety", reportUserId = <userId>?})`.

- **Entry points.** (1) The [`ui.report.button`] button on the Results card killer chip (§6.9.10; players only), which calls `UiController.open("Settings", {tab = "safety", reportUserId = killer.UserId})`. (2) Settings ▸ **Safety** tab (`ui.settings.tab_safety`, §6.9.9), which lists every other player in the server (`Players:GetPlayers()` minus the local player; `DisplayName` and a style chip) with a [`ui.report.button`] button each. Bots never appear and have no button.
- **Flow** (modal panel 560 × 420, title `ui.report.title`): the target's `DisplayName` (preselected from the entry point) → a reason `Create.segmented` with four options `ui.report.reason_exploit`, `ui.report.reason_teaming`, `ui.report.reason_harassment`, `ui.report.reason_other` (values `exploit`, `teaming`, `harassment`, `other`; none preselected) → [`ui.report.send`] (enabled once a reason is picked) → `Net.client.fire("Report.Submit", {targetUserId = target.UserId, reason = <value>})` → close. The thank-you toast is the server's `Notify` `ui.notify.report_thanks`; the client shows no toast of its own. No free text is collected.
- **Disabled** (button not created) for self, for bots, and while `Runtime.get().Features.Reports == false`. A target who left the server before [Send] closes the panel without sending.
- Client-side guard: at most one report per target per 60 s (td) from this panel; the server's `2 / 0.02` rate limit still applies.

**Verify:** scenario `ui.report_submit` (level `multi`: `multi(2, Baseline30, ui.report_submit)`; client A opens Settings ▸ Safety, reports client B with reason `teaming` → `ModerationService.reports(1)` has `{reporter = A, target = B, reason = "teaming"}` and A receives a `ui.notify.report_thanks` toast; `Features.Reports = false` via `setConfig` → the Safety list shows no report buttons).

---

### 6.10 Placeholder audio map

All sounds are the 11 built-in files (agent-assets §2.7; §0.5) as `rbxasset://sounds/<file>`, manifest entries `{status = "placeholder", source = "builtin", kind = "audio"}`. Per-move keys (`sfx.<style>.<slot>`) are Part 4 §4.7.3. The generic keys below carry `speed`, `vol`, `prio` (GDD §14.6 order) and `bus` in `Perf.Audio.Map`. Final audio follows GDD §14 (accordion and melodica over a seaside brass band; one short sting per headline face, such as Pip-Pip's *thunk*); briefs name moods and instruments only.

| Key | File | Speed | Vol | Prio | Bus | Event |
|---|---|---|---|---|---|---|
| `sfx.tele.windup` | `volume_slider.ogg` | 0.6 | 0.9 | 1 | Sfx | Telegraph start of moves > 10 damage (GDD §14.4) |
| `sfx.block.perfect` | `volume_slider.ogg` | 2.0 | 1.0 | 2 | Sfx | Perfect Block "ding" (GDD §14.4; played twice, 40 ms apart, as a 2-note ring) |
| `sfx.guard.break` | `impact_explosion_03.mp3` | 1.6 | 0.8 | 2 | Sfx | Guard Break |
| `sfx.hit.light` | `action_jump_land.mp3` | 1.4 | 0.7 | 3/5 | Sfx | M1 contact (3 if local is involved, else 5) |
| `sfx.hit.heavy` | `impact_explosion_03.mp3` | 1.2 | 0.6 | 3/5 | Sfx | Ender/move contact |
| `sfx.hit.block` | `action_footsteps_plastic.mp3` | 1.8 | 0.6 | 3/5 | Sfx | Blocked hit |
| `sfx.block.raise` | `action_footsteps_plastic.mp3` | 1.2 | 0.4 | 5 | Sfx | Block start |
| `sfx.dash.whoosh` | `action_swim.mp3` | 2.0 | 0.5 | 5 | Sfx | Dash, Tech Roll |
| `sfx.breakaway.burst` | `action_get_up.mp3` | 1.5 | 0.8 | 3 | Sfx | Breakaway |
| `sfx.awaken.sting` | `impact_explosion_03.mp3` | 0.7 | 0.9 | 3 | Sfx | Peak start ("Go Peak!") |
| `sfx.ko.poof` | `action_jump_land.mp3` | 0.5 | 0.8 | 3 | Sfx | KO: the soft puff of sand ("Checked out!", GDD §16.1) |
| `sfx.wallsplat.splat` | `impact_water.mp3` | 0.8 | 0.7 | 3 | Sfx | Wall Splat |
| `sfx.ragdoll.land` | `action_jump_land.mp3` | 0.8 | 0.5 | 5 | Sfx | First ragdoll landing |
| `sfx.spawn.ready` | `action_get_up.mp3` | 1.1 | 0.4 | 5 | Sfx | Spawn protection ends |
| `sfx.spring.boing` | `action_jump.mp3` | 0.7 | 0.7 | 5 | Sfx | Trampoline launch (from the local root's vertical velocity jump ≥ 60) |
| `sfx.break.shatter` | `impact_explosion_03.mp3` | 1.8 | 0.7 | 5 | Sfx | Breakable broken (a beach hut or a trapdoor) |
| `sfx.break.redraw` | `volume_slider.ogg` | 0.9 | 0.4 | 7 | Sfx | Hut repaint start |
| `sfx.realm.open` | `impact_explosion_03.mp3` | 0.5 | 1.0 | 2 | Sfx | A Getaway opens (the postcard frame slams down) |
| `sfx.tide.bell` | `volume_slider.ogg` | 0.8 | 0.7 | 5 | Sfx | Tide warning bell, once at the start of each warning (2D, every client on the hub; §6.3.8) |
| `sfx.vo.hurt` | `oof.ogg` | 1.0 | 0.6 | 4 | Vo | Local victim, damage ≥ 8 (bark-throttled) |
| `sfx.vo.ouch` | `ouch.ogg` | 1.0 | 0.6 | 4 | Vo | Local victim KO |
| `sfx.ui.click` | `volume_slider.ogg` | 1.2 | 0.6 | 4 | Ui | Buttons |
| `sfx.ui.open` | `action_jump.mp3` | 2.0 | 0.4 | 4 | Ui | Screen open |
| `sfx.ui.error` | `action_falling.ogg` | 1.6 | 0.5 | 4 | Ui | Error toast |
| `sfx.ui.reward` | `action_get_up.mp3` | 1.8 | 0.6 | 4 | Ui | Reward toast |
| `sfx.wish.thunk` | `action_jump_land.mp3` | 0.6 | 0.9 | 4 | Ui | The stamp lands (Pip-Pip's *thunk*), and the re-stamp of a fake-out |
| `sfx.wish.chime` | `volume_slider.ogg` | 1.0 → 2.0 | 0.8 | 4 | Ui | Wish arpeggio: `n` notes 90 ms apart (n = rarity index + 1), speed rising linearly to 1.0 + 0.15·n, so length and pitch encode rarity (GDD §14.4) |
| `sfx.wish.horn` | `impact_explosion_03.mp3` | 0.3 | 0.9 | 2 | Ui | The ferry horn: a Cosmic reveal (with Music muted), and once at the start of a Cosmic Arrival on the hub (§6.5.1) |

TTS barks (`SfxController.bark`; Studio only; subtitled): `ui.vo.awaken` ("Go Peak!"), `ui.vo.fully_booked` ("Fully booked!", from the caster when a Getaway opens), `ui.vo.ko` ("Checked out!", the only KO bark), `ui.vo.legendary`. Music: no tracks in M1-G; the `Music` bus and ducking exist so stems can drop in (GDD §14.3 is a future hook).

**Verify:** `tools/check_manifest.luau` (Part 7 §7.5.2) finds every key above; scenario `sfx.map_complete` (play each key once; no `[HB][Assets] WARN missing`).

---

### 6.11 `Defaults.Perf`

JSON-safe (Part 2 §2.6.1): numbers, strings, booleans, arrays, string-keyed tables; colours as hex; vectors as arrays. §0.9 gives Part 6 only the `Perf` key, so presentation and world tuning live in `Perf` sub-sections (§6.14 item 1).

```lua
	-- OWNED BY PART 6 (Spec §6.11). Every number cites the GDD or is a tuning default (td).
	Perf = {
		Tiers = {
			Low = { MaxEmitters = 120, MaxParticles = 1500, MaxBeamsTrails = 40,            -- GDD §13.5
				MaxFlipbooks = 40, TargetFps = 30, DrawCalls = 600, TrisOnScreen = 350000, -- GDD §18.7
				MapTrisOnScreen = 220000, TextureMb = 180, MemoryMb = 1100,                -- GDD §12.8, §18.7
				MaxRagdolls = 4, MaxDebris = 24, MaxSounds = 16 },                         -- (td)
			Mid = { MaxEmitters = 220, MaxParticles = 3000, MaxBeamsTrails = 80,
				MaxFlipbooks = 70, TargetFps = 45, DrawCalls = 900, TrisOnScreen = 600000,
				MapTrisOnScreen = 450000, TextureMb = 180, MemoryMb = 1600,
				MaxRagdolls = 8, MaxDebris = 48, MaxSounds = 24 },
			High = { MaxEmitters = 400, MaxParticles = 6000, MaxBeamsTrails = 150,
				MaxFlipbooks = 120, TargetFps = 60, DrawCalls = 1500, TrisOnScreen = 1000000,
				MapTrisOnScreen = 800000, TextureMb = 180, MemoryMb = 2500,
				MaxRagdolls = 12, MaxDebris = 96, MaxSounds = 32 },
		},
		AutoTier = {                                    -- graphicsTier = "Auto" (td)
			ByInput = { Touch = "Low", MicroGamepad = "Low", Gamepad = "Mid", KeyboardAndMouse = "High" },
			QualityLowMax = 3, QualityHighMin = 8,      -- UserGameSettings.SavedQualityLevel 1–3 Low, 8–10 High
		},
		Vfx = {
			PerMoveEmitterCap = { starter = 3, common = 3, uncommon = 5, rare = 5, epic = 6, legendary = 7, mythic = 8, cosmic = 8 }, -- GDD §13.5
			LodSimplifiedStuds = 80, LodTelegraphOnlyStuds = 160,  -- GDD §13.5
			ReducedVfxBudgetMult = 0.4, ReducedVfxEnemyOpacity = 0.6, -- GDD §16.3
			HighlightCap = 12,                                       -- GDD §12.4
			DegradeRateMult = { 1, 0.8, 0.6, 0.45, 0.3, 0.2 },       -- index = level + 1 (td)
			DegradeFrameTimeHigh = 1.15, DegradeFrameTimeLow = 0.85, DegradeUpAfterSec = 2, DegradeDownAfterSec = 5, -- (td)
			FlipbookFps = 12,                                        -- GDD §13.4 "on twos"
			Ink = "#241C33",                                         -- GDD §12.4, §13.4: warm navy-plum ink, never black
			ImpactFrameFrames = 2,                                   -- GDD §6.13
			TelegraphFillTransparency = 0.35, TelegraphOutlineStuds = 0.4, TelegraphContrastMin = 4.5, -- agent-assets §7.4; GDD §16.3
			TelegraphHoldAfterActiveFrames = 2,                      -- (td)
			DebrisLifeSec = { 6, 10 },                               -- GDD §11.1
			RebuildRedrawSec = 3,                                    -- (td) the hut repaint
			Prewarm = { ["vfx.hit.light"] = 16, ["vfx.hit.block"] = 8, ["vfx.dash.puff"] = 8, ["vfx.telegraph.circle"] = 8 }, -- (td)
		},
		Anim = { MaxPlayingTracks = 8 },                             -- GDD §18.2 (unverified; Part 3 spike SA-05)
		Camera = {
			ShiftLockDefault = true,                                 -- GDD §15.2
			ShiftLockOffset = { 1.75, 0, 0 },                        -- (td)
			HitstopFovKick = 2, AwakenCinematicSec = 1.2,            -- (td); GDD §6.9
			MaxFlashesPerSec = 3, FlashMaxWhiteAlpha = 0.5,          -- GDD §16.3; (td)
			OthersShakeFalloffStuds = 30,                            -- (td)
			Shake = {                                                -- GDD §6.13 light/medium/heavy; values (td)
				light = { Amplitude = 1, Frequency = 0.06, FadeInTime = 0, SustainTime = 0.03, FadeOutTime = 0.12, Pos = { 0.3, 0.3, 0 }, Rot = { 0.01, 0.01, 0.012 } },
				medium = { Amplitude = 1, Frequency = 0.07, FadeInTime = 0, SustainTime = 0.06, FadeOutTime = 0.22, Pos = { 0.7, 0.7, 0 }, Rot = { 0.02, 0.02, 0.03 } },
				heavy = { Amplitude = 1, Frequency = 0.08, FadeInTime = 0.04, SustainTime = 0.25, FadeOutTime = 0.45, Pos = { 1.4, 1.4, 0.4 }, Rot = { 0.04, 0.04, 0.06 } },
			},
		},
		Ragdoll = {
			TumbleSwitchFrames = 12,                                 -- GDD §13.3 (10–14f)
			LodStuds = 120, MaxLifeSec = 6, GetUpBlendSec = 0.25,    -- (td)
			FloorOffset = 2.4, AlignForcePerMass = 400, AlignResponsiveness = 25, FrictionTorque = 15, -- (td)
			Joints = {                                               -- degrees (td, "noodle" limits)
				Neck = { 45, -60, 60 }, Waist = { 30, -30, 30 }, Shoulder = { 110, -90, 90 }, Elbow = { 100, -10, 10 },
				Wrist = { 60, -30, 30 }, Hip = { 90, -45, 45 }, Knee = { 110, -5, 5 }, Ankle = { 40, -20, 20 },
			},
		},
		Looks = {
			ProxyDetailStuds = 100, TellOnlyStuds = 160, TellChipStuds = 150, NameRowStuds = 60, -- (td); GDD §11.1
			RingTransparency = 0.35,                                 -- (td)
			StealthRevealStuds = 12,                                 -- (td) deferred stealth render (§6.8.4; first user GDD §7.3.8)
			BodyScale = {                                            -- (td) Postcard Look body styles (§6.8.2): proxy scale only
				classic = { core = { 1, 1, 1 }, limbs = { 1, 1, 1 } },
				round = { core = { 1.25, 1, 1.25 }, limbs = { 1.1, 1, 1.1 } },
				tall = { core = { 0.95, 1.12, 0.95 }, limbs = { 0.95, 1.12, 0.95 } },
				tiny = { core = { 0.85, 0.85, 0.85 }, limbs = { 0.85, 0.85, 0.85 } },
			},
		},
		Ui = {
			RefSize = { 1334, 750 },                                 -- GDD §15.1
			MinTextPx = 14, TextScaleRange = { 0.8, 1.5 },           -- GDD §16.3
			MinTouchTarget = 88,                                     -- (td; 44 pt)
			HpDrainSec = 0.4, HpDrainHoldSec = 0.1,                  -- GDD §15.3; (td)
			ComboMinHits = 3, ComboBigHits = 10,                     -- GDD §15.3; (td)
			KillFeedLines = 4, KillFeedSec = 6,                      -- GDD §15.3; (td)
			NameplateInfoStuds = 25,                                 -- GDD §15.3
			FocusHoldSec = 4,                                        -- (td)
			StyleWheelAutoConfirmSec = 3,                            -- GDD §15.4
			BoingFrames = 2,                                         -- GDD §12.7
			ToastSec = 4, ToastMax = 4, AnnounceSec = 4, ResultsSec = 6, -- (td)
			Reveal = { common = { 1.2, 0.5 }, uncommon = { 1.2, 0.5 }, rare = { 2.0, 0.8 }, epic = { 3.0, 1.2 },
				legendary = { 4.5, 2.0 }, mythic = { 6.0, 3.0 }, cosmic = { 8.0, 4.0 } }, -- GDD §8.6 {first, repeat}
			StampLandSec = 0.25,                                     -- (td) the stamp lands first (§6.9.7)
			FlipAt = 0.6,                                            -- (td) the card flips at this fraction of the reveal
			FakeOutUpgradeAt = 0.5,                                  -- (td) fraction of the reveal (the re-stamp)
			DamageNumbersDefault = false,                            -- GDD §15.3 (off by default; the `damageNumbers` setting enables them)
		},
		Audio = {
			PoolSize = 32, PitchJitter = 0.05, ComboPitchStep = 0.03, ComboPitchMaxSteps = 10, -- (td); GDD §6.13
			BarkMinIntervalSec = 4, BarkRangeStuds = 40,             -- GDD §14.5
			DuckComboDb = -4, DuckRealmDb = -8,                      -- GDD §14.6
			RollOffMinStuds = 8, RollOffMaxStuds = 80,               -- (td)
			TtsInStudioOnly = true,                                  -- (td)
			Map = {                                                  -- §6.10 (speed, vol, prio, bus)
				["sfx.tele.windup"] = { speed = 0.6, vol = 0.9, prio = 1, bus = "Sfx" },
				["sfx.block.perfect"] = { speed = 2.0, vol = 1.0, prio = 2, bus = "Sfx" },
				-- … one row per §6.10 key …
			},
		},
		Lighting = {
			EditPreset = "arena_duel_m0.Day",                        -- switched to "hub.Day" by the Elsewhere-on-Sea task
			TweenSec = 1, CycleEnabled = false, CycleSec = 2700, CycleTweenSec = 10, -- GDD §11.4 (45 min); (td)
			FogTweenSec = 3,                                         -- (td) Cosmic Arrival fog in/out (§6.5.1)
			Presets = { --[[ §6.5.2 table, one record per preset:
				["hub.Day"] = { LightingStyle = "Soft", ClockTime = 14, Brightness = 2.4, Ambient = "#9AA3B8", …,
					Atmosphere = { Density = 0.26, … }, ColorCorrection = { … }, Bloom = { … }, Clouds = { … } }, ]] },
		},
		World = {
			Spring = { LaunchVy = 101, LaunchVyTrapdoor = 130 },     -- (td) trampolines; apex ≈ 26 / 43 studs
			Slow = { Mult = 0.6 },                                   -- (td) GDD §11.2 A: the shallows slow
			Tide = { PeriodSec = 240, WarnSec = 10 },                -- GDD §11.2 A (the tide moves every 4 minutes); WarnSec (td)
			Boundary = { BandStuds = 20, MinMult = 0.2 },            -- (td) GDD §11.1 "gentle push-back"
			Breakable = { HutHp = 2, TrapdoorHp = 1, RebuildSec = 90, -- GDD §11.1 (90 s)
				InflateStuds = 1.5, MinVy = 40, SpawnClearStuds = 60 }, -- (td)
			ScanHz = 10,                                             -- (td)
			Sightline = { MaxStuds = 150, GridStuds = 20, EyeY = 5 }, -- GDD §11.1; (td)
			Hazard = { MaxDamage = 15, MinTelegraphSec = 1.5 },     -- GDD §11.1 (KO credit window: Part 3 Combat.Streak.EnvKoCreditWindowSec)
			Budget = { arena_duel_m0 = 400, training = 400, hub = 2500 }, -- (td) max BaseParts
		},
	},
```

**Appearance:** see Part 5 §5.10 (Part 5 owns the single canonical `Defaults.Appearance` table; decision D-13). This part reads `PostcardStyles` (the four body-style records `classic`, `round`, `tall`, `tiny`, read via `.id`), the skin palette `PostcardSkins` (Part 2 §2.4 `PostcardSpec.skin`), `PostcardColors` (Settings swatches, §6.9.9), `PostcardFaces`, `PostcardHats = 6`, `PostcardOutfits`, `OutfitColors` and `NoOutfitDarken` (three-colour Postcard Look, §6.8.2), `SimplifiedLimbShade` (§6.8.2) and `FaceExpressions` (§6.8.5). The body-style proxy scales are presentation and live in `Perf.Looks.BodyScale`, keyed by the same ids. This part does not write `Appearance`.

`Schema.rules` additions (validated at boot and on overlay, Part 2 §2.6.3): every `Perf.Tiers.<T>.*` int ≥ 0; `Perf.Vfx.LodSimplifiedStuds < LodTelegraphOnlyStuds`; `Perf.Ui.Reveal.<r>[1] ≥ [2]`; `0 < Perf.Ui.FakeOutUpgradeAt < Perf.Ui.FlipAt < 1`; `Perf.World.Hazard.MaxDamage ≤ 15`; `Perf.World.Sightline.MaxStuds ≤ 150`; `Perf.World.Tide.PeriodSec` int ≥ 20 and `Perf.World.Tide.WarnSec` ≥ 1.5 and ≤ `PeriodSec / 2`; every `Perf.Looks.BodyScale` entry has 3 numbers in (0, 2] for `core` and `limbs`; every `Perf.Lighting.Presets` colour matches `^#%x%x%x%x%x%x$`. `Perf.World.Tide.PeriodSec` reaches the simulation only through the builder-written `Zones.TidePeriodSec` attribute, so a live overlay of it changes nothing until the hub is rebuilt (§6.3.8). (`Appearance` rules, incl. `#PostcardStyles ≥ 1`, are Part 5 §5.10's.)

**Verify:** Lune `tests/core/perf.spec.luau`: `perf.defaults_gdd` (the GDD-cited numbers above against a literal table, including `World.Tide.PeriodSec = 240` and `World.Breakable.RebuildSec = 90`), `perf.body_scale_keys` (`Perf.Looks.BodyScale` has exactly the four body-style ids of `Appearance.PostcardStyles`), `perf.json_safe`, `perf.tiers_ordered` (Low ≤ Mid ≤ High for every budget), `perf.audio_map_complete` (every §6.10 key present).

---

### 6.12 Consolidated test catalog (this part)

| Kind | Names |
|---|---|
| Lune | `movement.volume_*` (5: `spring_neutral_only`, `slow_grounded`, `boundary_depth`, `order_deterministic`, `tide_gating`), `movement.tide_phase`, `theme.contrast`, `theme.reveal_durations`, `theme.cooldown_frac`, `theme.touch_no_overlap`, `theme.style_palette_variants`, `theme.rarity_tokens`, `perf.defaults_gdd`, `perf.json_safe`, `perf.tiers_ordered`, `perf.audio_map_complete`, `perf.lighting_presets_complete`, `perf.body_scale_keys`, `anims.base64_roundtrip`, `anims.upload_refuses_without_approve` (`tests/core/upload_anims.spec.luau`, §6.6.9), lints `render.no_attr_signal`, `anim.no_track_cache` |
| Builder | Part 1 §1.6.8 set for all 7 Part 6 builders |
| Scenarios: world | `world.spring_launch`, `world.shallows_slow`, `world.tide_swap`, `world.void_kill`, `world.hut_break_rebuild`, `world.trapdoor_fall_and_trampoline`, `world.rebuild_postponed_when_occupied`, `map.arena_spawn_8`, `map.arena_no_escape`, `map.hub_zone_label`, `map.hub_boundary_soft`, `map.hub_totem_prompt` |
| Scenarios: the Staff Room | `training.dummy_mode_ui`, `training.framedata_m1`, `training.trial_equip`, `training.damage_numbers_toggle`, `training.reset_travel` |
| Scenarios: presentation | `render.hitstop_not_reentry`, `render.no_double_move_vfx`, `render.remote_telegraph_full` (net profile `M0Gate`), `render.focus_target`, `render.haptic_gate`, `anim.cosmetic_budget`, `vfx.budget_tiers`, `vfx.telegraph_never_reduced`, `vfx.highlight_cap`, `vfx.lod_distance`, `vfx.attach_unanchored`, `vfx.impact_frame`, `vfx.templates_all_play`, `sfx.voice_cap`, `sfx.bark_throttle`, `sfx.bus_volumes`, `sfx.map_complete`, `cam.shift_lock_default`, `cam.reduced_motion`, `cam.flash_rate`, `cam.lighting_follows_map`, `cam.arrival_fog`, `ragdoll.clone_lifecycle`, `ragdoll.cap`, `ragdoll.wallsplat_handoff`, `ragdoll.juggle_rebegin`, `ragdoll.no_misprediction`, `look.style_tell_always`, `look.postcard_three_colour`, `look.body_styles`, `look.peak_uniform`, `look.variant_render`, `look.face_expressions` (programmatic + one 3×3 capture) |
| Scenarios: multi (level `multi`; run only via `multi(2, Baseline30, <name>)`) | `look.modes_matrix`, `ui.report_submit` |
| Scenarios: UI | `hud.hp_drain`, `hud.cooldown_wipe`, `hud.meter_ring`, `hud.breakaway_swap`, `hud.touch_binding`, `hud.combo_counter`, `hud.simple_preset`, `hud.edge_indicators`, `hud.wallet_level`, `hud.quests_progress`, `hud.move_lock_glyph`, `hud.tide_chip`, `ui.skin_refresh`, `ui.wheel_death_autoconfirm`, `ui.wheel_equip_next`, `ui.wheel_in_dev_badge`, `ui.wheel_cl_toggle`, `ui.wheel_variant_equip`, `ui.wheel_spend_shards`, `ui.wheel_titles`, `ui.mastery_level_matches_server`, `ui.wish_odds_live`, `ui.wish_stamp_first`, `ui.wish_reveal_first_unskippable`, `ui.wish_10x_grid`, `ui.wish_no_gambling_terms`, `ui.quick_wish_instant`, `ui.wish_restricted_wallet`, `ui.shop_buy_flow`, `ui.shop_in_dev_badge`, `ui.shop_buy_variant`, `ui.store_restricted_hidden`, `ui.store_confirm_4th`, `ui.store_week_spend`, `ui.settings_roundtrip`, `ui.settings_new_toggles`, `ui.rebind_m1`, `ui.postcard_preview`, `ui.results_card`, `ui.notify_kinds`, `ui.no_offer_after_loss`, `ui.ftue_steps`, `ui.cvd_rarity`, `input.block_toggle`, `input.run_toggle` |
| Scenarios: MCP input | `mcp.gamepad_menu_nav` (Part 7 §7.4.10 row, owner P6, M1-G) |
| Defined elsewhere, exercised by this part's code | `combat.env_ko_credit` (Part 3 §3.13; world runtime cause), `tel.perf_sample_sent` (Part 5 §5.8.3; VfxController sender), `wish.cosmic_arrival_once` (Part 5 §5.2.18; this part's render check is `cam.arrival_fog`), `look.gear_silhouette` (Part 4) and `look.clip_stage` (Part 7), both using the §6.8.6 silhouette option, `combat.disabled_move_kill_switch` (Part 3; HUD lock glyph is cosmetic), `mcp.tab_no_playerlist` (Part 7 §7.4.10, M0; Part 3 §3.10.1 CoreGui calls; the Charm Ring half needs §6.9.6) |
| Human (staged by the agent, run and signed by humans) | The blind tests T7–T11 (GDD §5.5): this part supplies the silhouette stills, the T11 masked-silhouette clip staging and three hub captures; the agent appends `pending` rows to `docs/agent/RECOGNIZABILITY.md`, one per subject and preparation run (Part 8 §8.5.3; the three hub captures share one `map:hub` row) (§6.3.7, §6.8.6) |
| Spikes | `spike.S-P6-1` … `spike.S-P6-15` (§6.15) |

Levels and milestones (M0 or M1-G) for these names are in Part 7 §7.4.12, taken from the TASKS card that adds each one.

Log lines introduced: `[HB][World] break|rebuild|void_kill …`, `[HB][World] tide phase=<low|high> f=<f> map=<mapId>`, `[HB][BUILD] reminder Terrain.Decoration is human-set at H-04 (NotScriptable)`, `[builder:BuildTemplates] silhouette templates active (rebuild before commit)` (a `verify()` warning), `[HB][CHECK] upload_anims key=… id=… | FAIL … | SUMMARY ok=<n> fail=<m>` (§6.6.9), `[HB][Anim] clips n=… / WARN track_budget`, `[HB][Vfx] WARN missing_template key=…`, `[HB][Vfx] degrade level=<n>`, `[HB][UI] ERROR odds_sum …`, `[HB][UI] screen open=<id>`, `[HB][Look] mode rid=<n> mode=<m>` (Studio only).

---

### 6.13 New contracts introduced

1. **Map instance layout** (§6.1.1, §6.1.4): map Models with `MapId`, `KillY`, `Origin`, `BuildVersion`; child folders `Geometry` (zone sub-folders plus `Geometry.edge`), `Spawns`, `Totems`, `Breakables`, `DummyPoints`; `Zones` is a **Model** with `ModelStreamingMode = Persistent` holding `Volumes`, `KillVolumes`, `EventZones`, `ZoneBounds`. Map origins: the Bandstand (−2600, 0, 0), Elsewhere-on-Sea (0, 0, 0), the Staff Room (2600, 0, 0). The full attribute table of §6.1.4, including `Tide`, `TidePeriodSec`, `TidePhase` and `GateStub`. *(Parts 2, 3, 5, 7)*
2. **Collision groups `MapBoundary` and `RagdollFx`** with the §6.1.5 matrix, registered by Part 3's `Hurtbox.ensureGroups()` (8 of 32). *(Part 3)*
3. **Predicted volumes:** `Movement.volumeEffect(pos, vel, grounded, neutral, moveDir, vols, tidePhase?)` and `Movement.tidePhase(f, periodF)` (pure, in `Core/Movement.luau`), and their Adapter application (cache `vols` and `tidePeriodF` per map at `track`; apply after `Fsm.step`, before `Loco.apply`). *(Part 3)*
4. **World runtime** block in `CombatService` (`CombatService.world` API, including `tidePhase(mapId)` and the `TidePhase` mirror), which kills with `cause = "environment"`. The 5 s environment KO credit window (GDD §11.1) is **done in Part 3 §3.17** (`Combat.Streak.EnvKoCreditWindowSec = 5`, applied in §3.13.2; scenario `combat.env_ko_credit`); `Perf.World.Hazard` carries no credit key. *(Part 3)*
5. **`Util.luau` MapKit additions:** `Util.KIND`, `mapPart`, `ramp`, `slabRects`, `ensureRootOf`. *(Part 1)*
6. **PlayerService / travel:** (a) set `Player.DevEnableMouseLock = false` at join (server-only property); (b) initial `MapId` = `"hub"` if `Workspace.Map.Hub` exists, else `"arena_duel_m0"`; (c) add `"arena_duel_m0"` to the `World.Travel` enum (valid only when `Workspace.Arena.DuelArena` exists); (d) `canTravel` skips the combat-tag check for `training → training` travel (the Staff Room Reset). *(Parts 2, 3)*
7. **InputController additions:** `setShiftLock(on: boolean)`, `shiftLock(): boolean` (feeds `AimLock`); the client-local `UI` InputContext (`StyleWheel`, `ShiftLock`, `Emote`, `MenuFocus` on Pad `DPadDown`, §6.9.5); hold-to-toggle block via clearing bindings plus the `Script` binding; hold-to-toggle run (`holdToToggleRun` latches `RunAuto`, §6.9.11); InputController never writes binding KeyCodes (SettingsController owns rebinds, Part 2 §2.3.7 rule 6); the CoreGui disabling (`Backpack`, `PlayerList`, `AutoSelectGuiEnabled = false`) is Part 3 §3.10.1's `InputController.init()`; touch `LauncherBtn`, `ShiftLockBtn` and `CharmBtn` (the touch Charm button, GDD §15.2); binding `Jump.Touch.UIButton` to the default `JumpButton`. BotService writes `DummyPoints.Slot_<n>` attribute `Mode` on every `Training.Dummy` change. *(Part 3)*
8. **Part 5 expectations:** WishService sets `flags.seenReveals[rarity] = true` for every rarity of a batch in the roll transaction; StyleService infers a Charm Rack (totem) equip per **Part 5 §5.1.3** (distance from the requester's HRP to a Totem **Model pivot** of its map ≤ `Progression.TotemRangeStuds` (12), plus the swap gate `canTravel`) and applies it now through Part 3's random safe spawn on the same map (DES-3 pending); the builder sets `TotemRadius` from `Progression.TotemRangeStuds` for the ProximityPrompt only; `Wish.Pity` key names are read by the Wish screen; the stamp-first reveal fields (Part 5 §5.2.13) and `CosmicArrivalUntil` (Part 5 §5.2.18) are rendered here (§6.9.7, §6.5.1). *(Part 5)*
9. **DevCmd additions:** `breakAll(mapId)`, `rebuildAll(mapId)`, `setLightingPhase(phase)`, `setTidePeriod(sec)` (Studio only: rewrites the hub's `Zones.TidePeriodSec` on the server for tests; the builder value returns on the next build or `setTidePeriod 240`). *(Parts 2, 7)*
10. **Settings keys** (Part 2 §2.8.6 rows, `Types.Settings`, `SETTING_KEYS`): `quietBarks` (boolean, false, GDD §14.5; SfxController), `damageNumbers` (boolean, false, GDD §15.3; RenderController; replaces `Perf.Ui.DamageNumbersInTraining` with `Perf.Ui.DamageNumbersDefault = false`), `holdToToggleRun` (boolean, false, GDD §16.3; InputController latches `RunAuto`), `misprintCalm` (the Misprint Finish renders static), and `binds.<Action>.<Device>` with `<Device>` ∈ {`KBM`, `Pad`} (SettingsController applies it to `Player.Combat.<Action>.<Device>.KeyCode`; single owner). Title selection: ProgressionService auto-equips the first granted title when `account.title == ""` (Part 5 §5.5.5); there is no `title` settings key or picker in M1-G (§6.15 future hook). *(Parts 2, 3, 5)*
11. **Hurtbox.prepare** additionally sets `Humanoid.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.None`. *(Part 3)*
12. **No hazards in M1-G maps;** the `Hazard*` attribute schema and limits are reserved and verified. The tide is not a hazard: it deals no damage and only slows (§6.3.8). *(Part 7)*
13. **`UiController.open(screen, args?)`** optional args; screen module contract (§6.9.1); lazy screen requires; open screens, the HUD and nameplates re-render on `Skin.onChanged` (Part 2 §2.9.2; future text overrides; scenario `ui.skin_refresh` through the `Skin._notify()` seam). The `StyleWheel` screen serves both the Charm Ring and the Scrapbook (§6.9.6). The Report flow is a Settings sub-panel (`open("Settings", {tab = "safety", reportUserId})`), so `Types.ScreenId` is unchanged. *(Part 2)*
14. **Client-local root `Workspace.HB_Local`** (`Vfx`, `Looks`, `Ragdolls`, `Debris`, `Sfx`, and the `HapticEffect` `HB_Haptic`); `Verify` flags it in Edit. **`ReplicatedStorage.Looks`** contents: `LookRig_R15`, `Face_UI`, `Gear_*`, `Peak_*`, `Summon_*`, `Postcard_Hat_*`, `Postcard_Outfit_*`, and the attribute `Silhouette` (set only by the silhouette build). **`ServerStorage.Templates.Fragments`** cache. **Lighting children** `HB_Atmosphere`, `HB_Sky`, `HB_CC`, `HB_CC_Fx`, `HB_Bloom`, and `Terrain.HB_Clouds`. *(Parts 1, 2, 7)*
15. **Asset keys** (manifest entries in `Shared/Assets.luau`, Part 2 §2.9.1 format, added by the task that builds each key; Part 7's `tools/check_manifest.luau` checks them, including Part 8's rules M09–M13): generic `vfx.*` (§6.6.7, incl. `vfx.realm.frame`, `vfx.realm.ring`, `vfx.tide.foam`), generic `sfx.*` (§6.10, incl. `sfx.wish.thunk`, `sfx.wish.horn`, `sfx.tide.bell`), `anim.common.*` (19, §6.6.8), `model.postcard.hat_<n>`, `model.postcard.outfit_<n>`, and Part 4's six Peak-piece keys `model.<style>.peak_<name>` (built here, §6.8.6); new VFX recipes `VFX_BlockSpark`, `VFX_PerfectStar`, `VFX_Breakaway_Burst`, `VFX_Splat`, `VFX_Tell_Ring`, `VFX_KO_Burst` (sand puff and hankie), `VFX_Getaway_Frame`, `VFX_Tide_Foam`, plus `_Lod1` variants; new poses `hurt_light`, `hurt_heavy`, `tumble`, `splat`, `dizzy`, `ko_lie`, `getup_kneel`. *(Parts 2, 4, 7, 8)*
16. **Skin keys** (Part 8 carries them in the `Base` pack): `map.<mapId>.name` (the Bandstand, Elsewhere-on-Sea, the Staff Room), `map.hub.zone.<zoneId>` (6: `under_pier`, `pavilion`, `promenade`, `beach`, `bandstand_gardens`, `jubilee_clock`), `ui.hud.*` (`breakaway`, `protected`, `combo`, `combo_big`, `go_peak`, `fully_booked`, `checked_out`, `tide_low`, `tide_high`, `tide_in`, `tide_out`), `ui.feed.bot`, `ui.nameplate.rookie`, `ui.totem.action`, `ui.totem.object`, `ui.wheel.*` (`in_dev`, `trial`, `free`, `get_hint`, `title_ring`, `title_scrapbook`), `ui.wish.*` (incl. `make_1`, `make_10`, `new`, `shards`, and Part 5's `zone_<rarity>`), `ui.shop.*` (incl. `tab_rack`, `tab_store`, `sundry_line_<n>`), `ui.store.*` (incl. `confirm_4th`), `ui.settings.*` (incl. `layout_editor_soon`), `ui.results.*` (incl. `environment`), `ui.training.*` (`slot`, `mode_<mode>`, `m1_<sub>`, `ender_<sub>`, `dash_<sub>`), `ui.toast.*` (`emotes_soon`, `key_in_use`), `ui.tip.*` (5), `ui.vo.*` (`awaken` "Go Peak!", `fully_booked` "Fully booked!", `ko` "Checked out!", `legendary`), `ui.postcard.body.<id>`. **Added by decision D-42** (Part 8 rows in `Base`): `ui.hud.level`, `ui.hud.treasure` (Shells), `ui.hud.shards` (Stamps), `ui.hud.sparks` (Messages in a Bottle), `ui.hud.quests`, `ui.quest.<id>` (token `{goal}`), `ui.notify.quest_done`, `ui.report.button`, `ui.report.title`, `ui.report.reason_exploit` / `_teaming` / `_harassment` / `_other`, `ui.report.send`, `ui.settings.tab_safety`, `ui.settings.quiet_barks`, `ui.settings.damage_numbers`, `ui.settings.hold_toggle_run`, `ui.settings.character_look` (Full Uniform), `ui.settings.style_gear` (Charm Gear), `ui.settings.own_look_postcard`, `ui.settings.own_look_avatar`, `ui.training.damage_numbers`, `ui.wheel.shards`, `ui.wheel.convert`, `ui.wheel.mastery_combat_only`, `ui.wheel.variant`, `ui.wheel.variant_base`, `ui.wish.locked_paid`, `ui.wish.auto10`, `ui.wish.secret`, `ui.shop.balance`, `ui.shop.confirm_in_dev`, `ui.store.spent_week`. Existing keys now used: `ui.wish.fakeout_note` (no tokens), `ui.wheel.in_dev` (Sundry's Rack cards), `ui.shop.coming_soon`, `ui.notify.report_thanks`, `ui.notify.cosmic_arrival` (Part 5), `variant.<styleId>.<vid>.name`. *(Part 8)*

    **Base text of this part's keys.** This part owns these lines; Part 8 §8.3.4 and §8.3.7 carry them verbatim, and any other part that quotes one copies it. The Wish, shop, progression and appearance-data lines are Part 5's (§5.12 item 14): `ui.wish.*`, `ui.shop.*` (except the two tab labels below), `ui.hud.treasure` / `shards` / `sparks`, `ui.hud.quests`, `ui.quest.<id>`, `ui.wheel.shards` / `convert`, `ui.notify.*` and `ui.postcard.body.<id>`; the other D-42 keys are Part 8's. Tokens as in Part 8 §8.3.7.

    | Key | Base text |
    |---|---|
    | `map.arena_duel_m0.name` / `map.hub.name` / `map.training.name` | The Bandstand / Elsewhere-on-Sea / The Staff Room |
    | `map.hub.zone.<id>` | `under_pier` Under the Pier · `pavilion` The Pavilion · `promenade` The Promenade · `beach` The Beach · `bandstand_gardens` Bandstand Gardens · `jubilee_clock` The Jubilee Clock |
    | `ui.hud.go_peak` / `ui.hud.fully_booked` / `ui.hud.checked_out` | Go Peak! / Fully booked! / Checked out! |
    | `ui.hud.breakaway` / `ui.hud.protected` | Breakaway! / Protected {sec}s |
    | `ui.hud.combo` / `ui.hud.combo_big` (GDD §15.3) | {n} HIT! / {n} HITS! WHAT A HULLABRAWL! |
    | `ui.hud.tide_low` / `ui.hud.tide_high` / `ui.hud.tide_in` / `ui.hud.tide_out` | Low tide / High tide / Tide coming in: {s}s / Tide going out: {s}s |
    | `ui.feed.bot` / `ui.nameplate.rookie` | Bot / New Arrival |
    | `ui.totem.action` / `ui.totem.object` | Change Charm / Charm Rack |
    | `ui.wheel.title_ring` / `ui.wheel.title_scrapbook` | Charm Ring / Scrapbook |
    | `ui.wheel.in_dev` / `ui.wheel.trial` / `ui.wheel.free` | In development / Trial / Free this week |
    | `ui.wheel.get_hint` | Post a Wish for it, or find it on Sundry's Rack |
    | `ui.shop.tab_rack` / `ui.shop.tab_store` | Sundry's Rack / Store |
    | `ui.store.confirm_4th` | You've bought Wishes several times in the last 24 hours. Buy again? |
    | `ui.settings.layout_editor_soon` | Touch layout editor coming soon |
    | `ui.results.environment` | The seaside |
    | `ui.training.slot` | Dummy {n} |
    | `ui.training.mode_idle` / `_block` / `_perfect_block` / `_m1_spam` / `_dash_spam` | Stand still / Block / Perfect Block / Attack / Dash |
    | `ui.toast.emotes_soon` / `ui.toast.key_in_use` | Emotes are coming soon! / That key is already in use. |
    | `ui.tip.punch` | Attack 4 times in a row for a combo! |
    | `ui.tip.block` | Hold block to guard. Time it right for a Perfect Block! |
    | `ui.tip.dash` | Dash to dodge, or to close the gap. |
    | `ui.tip.awaken` | Your Peak Meter is full. Go Peak! |
    | `ui.tip.wish` | You have a Wish! Post it through Pip-Pip in the Wish menu. |
    | `ui.vo.awaken` / `ui.vo.fully_booked` / `ui.vo.ko` / `ui.vo.legendary` | Go Peak! / Fully booked! / Checked out! / Gilt stamp! |

    The `ui.training.m1_<sub>`, `ender_<sub>` and `dash_<sub>` lines get their text in the task that introduces each `<sub>` (Part 8 §8.3.7).
17. **Config:** complete `Defaults.Perf` (§6.11) plus the `Schema.rules` additions, including `Perf.World.Tide = {PeriodSec = 240, WarnSec = 10}`, `Perf.World.Spring.LaunchVyTrapdoor`, `Perf.World.Breakable.HutHp` / `TrapdoorHp`, `Perf.Looks.BodyScale`, `Perf.Ui.StampLandSec` / `FlipAt`, `Perf.Lighting.FogTweenSec` and `Perf.Vfx.Ink = "#241C33"`; `Perf.Ui.DamageNumbersDefault = false` replaces `DamageNumbersInTraining`; `Perf.World.Hazard` has no `KoCreditSec`. `Defaults.Appearance` is Part 5 §5.10 (merged; D-13). *(Parts 2, 5, 7)*
18. **Log tags** added to Part 2's list: `World`, `Anim`, `Sfx`, `Cam`, `Look` (`Render`, `Vfx`, `UI` already exist). *(Part 2)*
19. **Studio snippet** `tools/studio/map_stats.luau` (§6.1.9). *(Parts 1, 7)*
20. **Theme is load-pure** and Lune-tested (`tests/core/theme.spec.luau`); new Lune specs `movement_volumes.spec.luau`, `perf.spec.luau`, `upload_anims.spec.luau`. *(Part 7)*
21. **Map roots under the `Workspace.Map` container** (decision D-01): `BuildMap_Hub` and `BuildTraining` create their Model roots with `Util.ensureRootOf(Util.ensureContainer(workspace, "Map", c), "<Name>", "Model", NAME, c, opts.reset)`; the container belongs to `BuildAll`'s containers step and is never pruned or reset (§6.1.1). *(Part 1)*
22. **`Terrain.Decoration` is human-set at H-04** (NotScriptable); no builder writes or reads it; `verify()` prints a non-failing reminder (§6.1.6). *(Part 1)*
23. **Animation upload pipeline (H-07 route)** §6.6.9 (decision D-09): snippet `tools/studio/export_anim.luau`, Lune `tools/upload_anims.luau` and `tools/base64.luau`, outputs under `build/anims/`, spike S-P6-14, conditional card T143. Part 1's example permissions file keeps `lune run tools/upload_anims.luau` on **ask** (every run human-approved). *(Parts 1, 7; TASKS)*
24. **Client perf sample sender** (decision D-17): VfxController sends `Telemetry.Perf` while `LocalPlayer:GetAttribute("PerfSampled") == true`, using `Telemetry.PerfWindowSec` / `PerfSendEverySec` (§6.6.3). *(Parts 2, 5)*
25. **Attached VFX are un-anchored and massless** before welding to a character (§6.6.3); the Style Tell ring follows the same rule (§6.8.4). *(Parts 3, 4)*
26. **Render contract** (decision D-29, Part 3 §3.11.8): entry = `St` or `Sub` changed; one-shots keyed by `(actionId, threshold)` with `firedMax`; telegraph only while `Cnt == 0` and before the first active frame; contact undo only with unchanged `actionId` or a rollback to ≤ the contact frame; predicted effects also undone on a STUN/KO entry within 15f; remote views use observation time `eR` (§6.6.1). *(Part 3)*
27. **`Theme.stylePalette(styleId, variantId?)`** (canonical code in §6.9.4): `StyleDef.palette` plus the Off-Duty Look rule (`v1` swaps secondary/accent, `v2` and later darken 25 %); no pack lookups, because the Skin pack holds strings only. `Variant` is an appearance refresh trigger (§6.8.3). *(Parts 4, 8)*
28. **Peak uniform render** (§0.4; §6.8.2): at Peak every viewer except `Simplified` sees the `Character` mode (the Full Uniform) plus Part 4's optional Peak piece (one per slice style) with the gilt trim, whatever the owner's Full Uniform setting; nothing scales; RenderController calls `AppearanceController.setPeak(model, on)` on `Awakening` entry and on `AwkF0` > 0 → 0. *(Parts 3, 4, 5)*
29. **FTUE-lite → `Ftue.Step` mapping** (2–7) and the **no-offer-after-loss** rule (`Economy.NoOfferAfterLossSec`) in UiController (§6.9.10; decision D-40). *(Parts 5, 7)*
30. **Report UI** (§6.9.12): Results-card and Settings ▸ Safety entry points to `Report.Submit`. *(Parts 2, 5)*
31. **Per-move kill switch, HUD side** (decision D-14): the hotbar/cluster lock glyph for move ids in `Runtime.get().Features.DisabledMoves` (cosmetic; the sim refuses via `StyleRuntime.disabled`). *(Part 3)*
32. **HUD meta elements**: Wallet & level (Shells, Stamps, Messages in a Bottle), Quests (on in M1-G, decision D-39), the Shells header in Sundry's Rack, Store weekly spend from `StoreView.spentThisWeekRobux`, Off-Duty Look and Stamp UI in the Scrapbook, Quick Post and the PRI-restricted wallet in the Wish screen (§6.9.5–§6.9.8). *(Part 5)*
33. **The tide** (§6.3.8): two `Slow` volumes tagged `Tide = "low"` / `"high"`; the builder-written `Zones.TidePeriodSec` (240); the phase `Movement.tidePhase(f, TidePeriodSec × 60)` computed by both peers from the simulation frame (swaps on fixed 4-minute boundaries of the server's clock; never client input); the server mirror attribute `TidePhase` and its log line; the 10 s render-side warning (HUD chip, `sfx.tide.bell`, `vfx.tide.foam`). *(Parts 2, 3, 7; §0.4)*
34. **Silhouette template option** (§6.8.6): `BuildTemplates.build({silhouette = true})`, the `ReplicatedStorage.Looks` attribute `Silhouette`, the capture-staging render rules (§6.8.2) and the `verify()` warning; it feeds `look.gear_silhouette` and `look.clip_stage`. Stills, staged clips and hub captures go into `pending` rows in `docs/agent/RECOGNIZABILITY.md`, one row per subject and preparation run (Part 8 §8.5.3); humans run and sign T7–T11. *(Parts 4, 7, 8)*
35. **Cosmic Arrival render** (§6.5.1): from `CosmicArrivalUntil`, the `hub.Fog` overlay, `sfx.wish.horn` once and the Music mute on the hub; message only elsewhere. *(Part 5)*
36. **Hub names** (§6.3): zone IDs `under_pier`, `pavilion`, `promenade`, `beach`, `bandstand_gardens`, `jubilee_clock`; trampolines `Tramp_*` and `PierTramp_W/E` (`Vol = "Spring"`); beach huts `Hut_<i>_<j>` and trapdoors `Trapdoor_W/E`; volumes `Shallows_Low/High`; stubs `TicketBooth_N/S` (`GateStub`) and event zones `gate_n`, `gate_s`, `gull_raid`; the Staff Room trampoline `Tramp_Test`. Scenario names `world.shallows_slow`, `world.hut_break_rebuild`, `world.trapdoor_fall_and_trampoline`, `world.tide_swap`, `map.hub_*`. *(Part 7; TASKS)*
37. **UI tokens** (§6.9.4): Ink `#241C33` (never black), the seaside accents, `Theme.Gilt` (used only for Legendary-and-up stamps and frames, the Gilt Finish, the Peak trim and currency icons), the rarity colours Stripe Red, Seafoam, Sky, Coral, Twilight, Sun Gold, Deep Sea and Fog-and-Starlight, and stamp-shaped rarity badges. *(Parts 4, 5, 8)*

### 6.14 Issues with spine

1. **Config keys.** §0.9 gives Part 6 only `Perf` ("VFX budgets per device tier"). Presentation and world tuning (camera, UI timings, lighting presets, trampoline speeds, the tide period, break HP) also need config homes, because Part 2 forbids magic numbers in code. This part puts them in `Perf.*` sub-sections. Suggested spine fix: add top-level `Presentation` and `World` keys.
2. **No world service.** The fixed service list has no home for the server map runtime (void kills, breakables, the tide mirror). This part specifies it as a block inside Part 3's `CombatService.luau`, and the predicted volume and tide logic inside Part 3's `Movement`/`Adapter`. Suggested spine fix: add `Services/WorldService.luau` (Part 2's ORDER would become 16, after CombatService).
3. **"Style Tell always on" vs stealth** (resolved: §0.4 now names both exceptions, the deferred stealth render, where the shimmer carries the style colour, and Studio-only capture staging, §6.8.2). A stealth status (`iv`, deferred until its first user, GDD §7.3.8, Year 1) hides a player from distant enemies; a permanent nameplate chip would defeat it. When it lands, this part follows Part 4's rule (the chip hides; the shimmer keeps the style colour).
4. **Flat controller files and the 700-line cap.** §0.3 fixes the client file list, so the ragdoll presenter lives in `RenderController.luau`, the face controller and nameplates in `AppearanceController.luau`, telegraphs and the Highlight pool in `VfxController.luau`, and the Staff Room panels in `UI/Screens/Hud.luau`. These will likely exceed Part 2's 700-line cap. Suggested fix: allow `Controllers/<Name>/init.luau` with child modules for these four, or raise their cap to 1,100 lines.
5. **Marker vs data timing** (resolved by decision D-03; kept for the rationale). Part 4 §4.2.0.7 originally fired move VFX/SFX from clip markers with a frame-counter fallback. This part (and the brief) uses data-driven frame timing as the primary path: `GetMarkerReachedSignal` is not Simulation-Access, temporary clip IDs do not exist outside Studio, and markers would fire twice during resimulation. Markers stay in the clips for the timing-lock check.
6. **Kit props in `ServerStorage.Templates`** (resolved: §0.3 now lists Charm Gear props, Peak pieces, summon models, the look rig and the Postcard Look props under `ReplicatedStorage.Looks`, and keeps only rigs and other server-only templates in `ServerStorage.Templates`). Templates there cannot be seen by clients, so this part follows Part 2 §2.13 item 3 (`ReplicatedStorage.Looks`).
7. **Highlight limit** (resolved). GDD §12.4 now says only that a client can render a limited number of Highlights and that we pool and cap them at 12; the docs say 255 and that disabled ones still count [Docs `Highlight.yaml`]. The pooled cap of 12 is kept.
8. **M0 map selection and travel.** The spine's Part 2 defaults assume the hub exists; M0 has only the Bandstand and the Staff Room (§6.13 item 6 asks Parts 2/3 for the fixes).
9. **Touch HUD: HP bar and Breakaway pips top-centre** (deviation from GDD §15.3 "bottom-center, above the hotbar"; GDD §15.1 "mobile defines the layout"). The bottom-centre band on touch is ≈ 150 reference units wide between the thumbstick zone and the cluster (§6.9.5). Needs owner sign-off at H-10; fallback layout in §6.9.5.
10. **Quick Post vs first-view reveals.** GDD §17.3 sells "instant reveals"; GDD §8.6 says a rarity's reveal can be skipped only after its first view. This part keeps the first view of each rarity in full and makes every later reveal instant for pass owners (§6.9.7). Owner to confirm at H-10.
11. **The tide schedule** (resolved: §0.4 now says the swaps "fall on fixed 4-minute boundaries of the server's simulation clock"). Swaps follow the server's clock as seen by the simulation (`time()` under Server Authority, synchronized with clients). A swap driven by `workspace:GetServerTimeNow()` and replicated as an attribute would reach clients late and mispredict the slow. If spike S-P6-15 fails, the fallback pins the tide phase per life (read at spawn, like combat config).

### 6.15 Open questions / spikes

| ID | Question | Method | Pass | Fallback |
|---|---|---|---|---|
| S-P6-1 | Does `Players:CreateHumanoidModelFromDescriptionAsync` run in Edit via `execute_luau` (DummyRig/BotRig/LookRig sizes)? Are the R15 `*RigAttachment` names as listed in §6.7.2 (with `AvatarJointUpgrade` on)? | Run `BuildTemplates`; list attachments | Rig with 15 parts and the 14 joint attachments | Human inserts an R15 rig via Rig Builder into `ServerStorage.Templates.__RigSeed`; read names from it |
| S-P6-2 | Do client-local, massless parts welded to the **local predicted** character (look proxies, Charm Gear, Peak pieces, the Style Tell ring and attached VFX such as `vfx.streak.aura` and `vfx.spawn.shimmer`, all un-anchored per §6.6.3) cause mispredictions or physics changes? | Two clients, 5 min of dashing, Peaks and ragdolls with proxies, ring and auras on; Misprediction log; scenario `vfx.attach_unanchored` | No entries attributable; dash distance error unchanged; `HRP.AssemblyRootPart == HRP` throughout | `BulkMoveTo` follow in the render step for proxies, ring and auras (§6.8.3, §6.8.4) |
| S-P6-3 | Can our `Jump` action's `Touch.UIButton` be bound to the default `JumpButton`, and does it still jump? | Touch emulator | Both the default jump and our `Jump` state fire | Own `JumpBtn`; hide the default button |
| S-P6-4 | Does an `AlwaysOnTop` SurfaceGui outline render above ParticleEmitters and Beams? | Telegraph inside a dense aura | Outline visible in a capture | Raise the fill part 0.3 studs; lower other VFX `ZOffset` |
| S-P6-5 | Is the in-sim trampoline launch (`ChangeState(Freefall)` + `AssemblyLinearVelocity`) predicted without rollback at 150 ms? | `world.spring_launch` under Network Simulator | No `St`/position correction > 1 stud | Launch via `HB_Vel` mover for 6 frames (Part 3 mover path) |
| S-P6-6 | Do the built-in fonts render ○ ◇ △ ☆ ✦ ❂ ✺ ♥ ★ ✦? | Capture a label per glyph per font | All render (no tofu) | Frame-drawn shapes (§6.9.4) |
| S-P6-7 | Does a client-created `InputContext` under `LocalPlayer` fire `Pressed` under Server Authority? | Press Tab | The Charm Ring opens | `UserInputService.InputBegan` for UI actions |
| S-P6-8 | With `DevEnableMouseLock = false`, does setting `MouseBehavior = LockCenter` at `Camera + 1` stick every frame? | KBM playtest | Cursor stays locked; `AimLock` true | Set it in `RenderStepped` at `Last` as well |
| S-P6-9 | Are client-created unanchored parts (ragdoll clones, debris) simulated locally under SA, and do BallSocket limits read as "noodle"? | `ragdoll.clone_lifecycle` + human review (H-10) | Clones fall and settle; limbs flop without breaking | Pose-only ragdoll (anchored, tumble hold) |
| S-P6-10 | GUI units on an SE-class device (points vs pixels) | H-12 device: log `Camera.ViewportSize` | Scale formula keeps 14 ref px ≥ 14 physical px | Adjust `RefSize` clamp |
| S-P6-11 | Server `CanCollide` toggles on breakable chunks near predicted characters: misprediction rate | Break beach huts next to a moving player | ≤ 1 correction per break | Delay collision removal by 6 frames after the visual break |
| S-P6-12 | Does `Font.fromName("ComicNeueAngular")` resolve? | Label test | Renders | `Enum.Font.Arimo` |
| S-P6-13 | Do pooled Highlights toggled via `Adornee`/`Enabled` avoid the add/remove hitch the GDD mentions? | Harness `PerfProbe` samples `Stats.RenderCPUFrameTime` [Docs `Stats.yaml`] every frame for 10 s idle (baseline) and 10 s while toggling 12 Highlights/s (agent-run; MicroProfiler is optional, for a human) | No frame more than 2 ms above the baseline's P95 | Keep Highlights adorned and change transparency only |
| S-P6-14 | Does the §6.6.9 export fit through `execute_luau`? (`SerializeInstancesAsync` from the plugin context; base64 size of the shortest and the longest existing clip vs the tool's result limit and the 2 KB payload cap) | In T143: `tools/studio/export_anim.luau` on the shortest and the longest existing clip (`anim.common.*`; `anim.attendant.m1_1` does not exist before T174, which repeats the check on it); decode on the host; compare byte length with `size` | One chunk (or a fixed chunk length) round-trips byte-exactly | Human H-08 (Save to Roblox); T143 `skipped` |
| S-P6-15 | Does the tide swap predict cleanly: do the owner client and the server compute the same `Movement.tidePhase` at every frame, including the swap frame and during resimulation? | `world.tide_swap` (period 20 s) under the `M0Gate` net profile; log `f` and phase on both peers around two swaps | Same phase on both peers at every logged frame; no position or `St` correction > 1 stud at a swap | Pin the phase per life: read it at spawn (`Adapter.track`) and keep it until the next respawn; the HUD then shows the character's own pinned phase (§6.14 item 11) |

Design questions:
- **Q1:** Trampoline speeds (101 / 130 studs/s), the shallows multiplier (0.6), the tide warning (10 s) and the boundary band (20 studs, ×0.2 at the wall) are tuning defaults for H-10. The 4-minute tide period is the GDD's.
- **Q2:** Beach-hut HP 2 (the first slam cracks, the second breaks) keeps wall combos usable. Confirm with design.
- **Q3:** Under the Pier's only exits are the trampolines under the trapdoors (they break a rebuilt trapdoor from below). Confirm this matches "high-risk shortcut".
- **Q4:** (resolved) Damage numbers use the Part 2 settings key `damageNumbers` (off by default; the Staff Room panel and Accessibility toggles, §6.4.2, §6.9.11).
- **Q5:** The client mastery-level formula duplicates `ProgressionService.masteryLevel`. A pure shared home (e.g. a function in `Config/Schema.luau`) would remove the duplication; Part 5 decides.
- **Q6:** The rarity CVD palette (one Okabe–Ito mapping for all three modes) needs human review with simulators. The Starter colour (Stripe Red) and the exact rarity hex values are tuning defaults for the art pass.
- **Q7:** Touch layout positions are designed for 1334 × 750; the layout editor and the one-hand preset (GDD §15.2, §16.3) are deferred past M1-G.
- **Q8:** The Postcard Look body-style scales (§6.8.2) are tuning defaults; the art pass replaces the proxies with the shared soft-toy body (GDD §12.3) and may retune them.

**Future hooks** (recorded decisions; not built in M1-G; owner sign-off at H-10):
- **Quick-Chat wheel** (GDD §16.2: non-free-text phrases such as "GG!", "Rematch?" and "Same time tomorrow?", plus style barks). No key binding is reserved in M1-G (`DPadDown` is the menu focus, §6.9.5); a later card adds a UI action, phrases as Skin keys and `TextChannel:SendAsync` on the default channel with a client-side rate limit.
- **Emote wheel** (the `Emote` action only shows `ui.toast.emotes_soon`).
- **Title picker** (GDD titles): M1-G auto-equips the first granted title (Part 5 §5.5.5) and has no `title` settings key; a picker in Settings ▸ Appearance is a later card.
- **Charm Ring move previews and tooltips** (GDD §16.3 learning aids): the Scrapbook detail pane lists moves by name and tags only.
- **Weekly quests and rerolls** (Part 5 §5.5.9): the HUD shows daily quests only.
- **Elsewhere-on-Sea landmarks and moving parts** (GDD §11.2 A): the funicular car on its timetable (it needs a deterministic moving platform; a spike first), the pier deck's breakable railings and ring-outs, Pip-Pip's sorting office, Sundry's Rack beside the Pavilion, the Summer Board, the picnic hampers on rebuild sites (a small speed buff), the lifeguards calling world events from their towers, sandcastles that the tide washes away and that rebuild, the world events themselves (GDD §10.4: the Gull Raid, with its stub zone `gull_raid`; `Features.WorldEvent`), and a use for the reserved gate markers (`TicketBooth_N/S`, `gate_n`, `gate_s`; no GDD event uses them yet).
- **Postcard toon pass over real avatars** (§6.8.2) and the touch layout editor / one-hand preset (Q7).
