[← Build spec index](../STUDIO_BUILD_SPEC.md) · **Part 6 of 8** · sections §6.x · Part 0 contracts (§0.x) are in the index file

## 6. World, UI and presentation

This part specifies everything the player sees and hears, and the places they fight in:
- the map builder framework and the three maps (`arena_duel_m0`, `candy_kingdom`, `training`);
- lighting presets;
- the client presentation layer: RenderController, AnimController, VfxController, SfxController, CameraController, the cosmetic ragdoll, and appearance rendering (Character Look modes, the Style Tell, UI-drawn faces);
- the UI system (`UI/Create.luau`, `UI/Theme.luau`, every M1-G screen);
- the placeholder audio map;
- the `Defaults.Perf` config section (VFX and presentation budgets); `Defaults.Appearance` is owned by Part 5 §5.10 and includes the keys this part reads.

**Relationship to other parts (binding).**
- Part 3 owns the simulation and the RenderController *contract* (Part 3 §3.11.8): the transition table, the 15-frame undo rule, `RenderController.onRollback(f)`, and "no `GetAttributeChangedSignal` for one-shots". Part 3 also owns combat animation playback (`Adapter.animate` inside `BindToAnimation`). This part implements the render side and never writes HRP attributes.
- Part 4 owns kit presentation data: palettes, per-move VFX recipe and SFX maps, Style Gear and summon recipes, the placeholder-animation algorithm and pose library (Part 4 §4.6–§4.7). This part builds and plays them.
- Part 2 owns the DataModel tree, client init order, PlayerGui root names, `UiController`/`SettingsController` APIs, `Types`, the remote catalog and Player attributes. This part fills the screens and the roots Part 2 assigns to it.
- Part 5 owns appearance data and replication (`AppearanceService`), Wish/Shop/Store semantics and payloads. This part renders them.

**Conventions.**
- Frames are 60 Hz. `fNow = Frames.fromTime(time())` sampled once per render step. `e = fNow − StF` (Part 3) for the **local** character's view only; remote views, bots and carriers not owned by the local player use observation time `eR`/`ageR` (§6.6.1 "Timing per view").
- Distances are studs, times seconds (`Sec` suffix). Map coordinates are **map-local**: relative to the map Model's origin (§6.1.1), with **+Y up, North = −Z, East = +X**. A centre `(x, y, z)` is a Part's `Position`.
- UI sizes are **reference units** on the 1334 × 750 canvas (§6.9.3).
- `(td)` marks a tuning default chosen here because the GDD gives no number. GDD numbers are cited "GDD §x.y" and never changed.
- Every engine API named here was checked in the offline creator-docs snapshot (2026-10-10). Items that could not be verified carry "(verify in spike S-P6-n)"; the spikes are listed in §6.15.
- Every requirement has a **Verify** line: a Lune test (`tests/...`), an in-Studio scenario (Part 7 harness, printed `[ALG][TEST] <name> PASS|FAIL`), an MCP call against `Roblox_Studio`, or an `[ALG]` log line.

**Files owned by this part.**

| Path | Contents |
|---|---|
| `src/DevTools/Builders/BuildArena.luau`, `BuildMap_CandyKingdom.luau`, `BuildTraining.luau`, `BuildLighting.luau` | World builders (§6.1–§6.5) |
| `src/DevTools/Builders/BuildTemplates.luau`, `BuildVfxTemplates.luau`, `BuildPlaceholderAnims.luau` | Presentation templates (§6.6.7, §6.6.8, §6.8.6) |
| Additive "MapKit" functions in `src/DevTools/Builders/Util.luau` | Shared map helpers (§6.1.2); Part 1 owns the file |
| `src/StarterPlayerScripts/Controllers/RenderController, AnimController, VfxController, SfxController, CameraController, AppearanceController, HudController` | Presentation (§6.6–§6.8) |
| Part 6 sections of `UiController.luau`, `SettingsController.luau`, `InputController.luau` | Screen router, toasts, settings application, touch layout (§6.9) |
| `src/StarterPlayerScripts/UI/Create.luau`, `Theme.luau`, `Screens/*.luau` | UI system (§6.9) |
| `Defaults.Perf` (complete) in `src/Shared/Config/Defaults.luau` (`Defaults.Appearance` is Part 5 §5.10's, including this part's keys) | §6.11 |
| `tools/studio/map_stats.luau`, `tools/studio/export_anim.luau`, `tools/upload_anims.luau`, `tools/base64.luau`, `tests/core/upload_anims.spec.luau` | Map stats snippet (§6.1.9); animation upload pipeline, H-07 route (§6.6.9; built by conditional card T143) |
| Specified here, implemented inside Part 3 files: `Movement.volumeEffect` (`Core/Movement.luau`), volume application (`Sim/Adapter.luau`), the world runtime block (`Services/CombatService.luau`) | §6.1.7, §6.1.8 |

---

### 6.1 Map builder framework

#### 6.1.1 Instance layout and ownership

Every map is a `Model` with fixed child folders. Ownership follows Part 1 §1.6.1 and Part 2 §2.2. Owned roots carry `AlgBuilder = "<Builder>"`.

| Map ID | Root instance | Owner | Map origin (world) | Part 2 tree |
|---|---|---|---|---|
| `arena_duel_m0` | `Workspace.Arena` (Folder) › `DuelArena` (Model) | `BuildArena` | `(−2600, 0, 0)` | `Workspace.Arena.DuelArena` |
| `candy_kingdom` | `Workspace.Map.CandyKingdom` (Model) | `BuildMap_CandyKingdom` | `(0, 0, 0)` | `Workspace.Map.CandyKingdom` |
| `training` | `Workspace.Map.Training` (Model) | `BuildTraining` | `(2600, 0, 0)` | `Workspace.Map.Training` |

The origins keep every pair of map footprints more than 2,000 studs apart (Part 2 §2.2), so StreamingEnabled never streams two maps to one client.

**Root creation.** `Workspace.Map` is a **container** owned by `BuildAll`'s containers step (Part 1 §1.6.1, §1.6.6 step 0): a Folder stamped `AlgBuilder = "BuildAll"`, never pruned and never destroyed by `reset`. The map builders own only their Model under it and never prune or stamp the container:
- `BuildMap_CandyKingdom`: `local root = Util.ensureRootOf(Util.ensureContainer(workspace, "Map", c), "CandyKingdom", "Model", NAME, c, opts.reset)`.
- `BuildTraining`: `local root = Util.ensureRootOf(Util.ensureContainer(workspace, "Map", c), "Training", "Model", NAME, c, opts.reset)`.
- `BuildArena`: `Workspace.Arena` is its own Folder root (`Util.ensureRoot`), and `DuelArena` is an owned child Model inside it.

`Util.ensureContainer(parent, name, c, quiet?)` (Part 1 §1.6.4) finds or creates the Folder (stamped `AlgBuilder = "BuildAll"`, ignoring `reset`). The map builders call it with three arguments: if `Workspace.Map` was missing, `ensureContainer` itself adds the warning `container Workspace.Map missing; run BuildAll` to the builder's counter (only `BuildAll`'s containers step passes `quiet = true`); the map builder adds no second warning. Idempotency rule 2 (prune unproduced children) applies to the owned Model roots only, so `BuildMap_CandyKingdom` never deletes `Workspace.Map.Training` and vice versa.

Children of each map Model (every name is fixed):

| Child | Class | Contents |
|---|---|---|
| `Geometry` | Folder | Every static gameplay part (kinds `Floor`, `Wall`, `Cover`, `Decor`, `Boundary`, visual spring pads), grouped in sub-Folders by zone (`Geometry.<ZoneId>`) |
| `Spawns` | Folder | Spawn marker Parts, attribute `SpawnFor = <MapId>` (Part 2 §2.5.1, Part 3 §3.13.4) |
| `Totems` | Folder | Style Totem Models (§6.1.4) |
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
	-- (west, east, north, south strips). Used for floor slabs with hatches.
	slabRects: (rect: { number }, holes: { { number } }) -> { { number } },
	-- Find-or-create an owned root of any class (Model or Folder) stamped with AlgBuilder (generalises ensureRoot).
	ensureRootOf: (parent: Instance, name: string, className: string, owner: string, c: Counter, reset: boolean?) -> Instance,
}
```

Rules for every map builder:
1. Layout lives in **data tables at the top of the builder** (one row per part, as in §6.2–§6.4). Logic only iterates tables. Gameplay tunables (spring speed, slow multiplier, break HP, rebuild time) come from `Defaults.Perf.World` (§6.11) and are written as **attributes** on the volume or breakable instances, so both peers read identical values from the place.
2. Parts are created in table order. Names are unique within their folder (`<Zone>_<Thing>_<n>`).
3. Map geometry is in collision group `Default` (Part 3 §3.5.5, §3.12.1). Only `Boundary` parts use `MapBoundary` (§6.1.5).
4. `Anchored = true` on every part. `CanTouch = false` everywhere (the world runtime uses AABB tests, not `Touched`).
5. No scripts, no asset IDs. AI-generated props from `ServerStorage.AIProps` (Part 1 C10) may replace `Decor` parts only, with a primitive fallback when the folder is missing.

#### 6.1.3 Zone palettes (Candy Kingdom)

Gameplay-relevant kinds (`Breakable`, `Hazard`, `Spawn`, `Event`, `KillVolume`, `Volume`, `Totem`) always use the semantic colour code. `Floor`, `Wall`, `Cover` and `Decor` take the zone palette below, so a player knows the zone at a glance (GDD §11.1) while blockout meaning is preserved. Arena and Training use their own palettes (§6.2, §6.4).

| Zone (`ZoneId`) | Floor | Wall | Cover | Accent (Decor) | Source |
|---|---|---|---|---|---|
| `castle_plaza` | `#F8BBD0` | `#FFF3D6` | `#F48FB1` | `#B39DDB` | GDD §12.6 (bubblegum, cream, mint, lavender) |
| `candy_streets` | terrain Pavement | `#FFE0B2` | `#7FD8BE` | `#F48FB1` | GDD §12.6 |
| `barracks` | terrain Ground | `#FFF176` | `#C8B560` | `#FFD54F` | GDD §11.2 A (Banana Guard yellow) (td) |
| `royal_gardens` | terrain LeafyGrass | `#3E2723` (licorice hedge) | `#CE93D8` | `#F8BBD0` | GDD §11.2 A (td) |
| `dungeon` | `#3A2F3F` | `#2E2433` | `#4A3D52` | `#FFE082` (grate glow) | GDD §11.2 A "dark, lit by grates" (td) |
| `earldom_border` | terrain Sand | `#F0E442` | `#C0CA33` | `#FFF176` | GDD §11.2 A "sour yellow corner" (td) |

**Wall-splat marking** (GDD §11.1): every part with `WallSplat = true` gets a `Trim` part: a dark band `#2B2B2B`, 0.6 studs tall, 0.2 studs proud of each face at its base, non-colliding. Houses use one trim block per house (§6.3.4).

#### 6.1.4 Attribute tagging contract

| Attribute | On | Type | Writer | Meaning / rule |
|---|---|---|---|---|
| `Kind` | every map BasePart | string (`Util.Kind`) | builder | Semantic kind; `verify()` checks flags against `Util.KIND` |
| `ZoneId` | `ZoneBounds` parts; `Geometry.<ZoneId>` folders | string | builder | One of the IDs in §6.3.1; HUD zone label (§6.9.5) |
| `WallSplat` | Wall/Cover/Breakable parts | boolean | builder | Marks a deliberate splat surface (trim added). Informational: Part 3's wall probe hits any steep `Default` surface |
| `Breakable` | Breakable **Model** | boolean | builder | §6.1.8 |
| `BreakHp`, `BreakFromAir`, `RebuildSec` | Breakable Model | number / boolean / number | builder (from `Perf.World.Breakable`) | §6.1.8 |
| `State`, `Hp`, `BrokenAt`, `RebuildAt` | Breakable Model | string / number / number / number | server world runtime | §6.1.8; `BrokenAt`/`RebuildAt` are `workspace:GetServerTimeNow()` seconds |
| `T0` | Breakable chunk parts | number | builder | Original transparency, restored at rebuild |
| `Hazard`, `HazardDamage`, `HazardTelegraphSec`, `HazardPeriodSec` | Hazard parts | boolean / number / number / number | builder | GDD §11.1: damage ≤ 15, telegraph ≥ 1.5 s. **No hazard instances exist in M1-G maps** (§6.13 item 12) |
| `SpawnFor` | spawn Parts | string (MapId) | builder | Part 3 `pickSpawn`; the part's CFrame faces the map centre |
| `HubId` | spawn Parts, Totem Models | string `"N"`/`"S"`/`"E"`/`"W"`/`"T"` | builder | Groups spawns with their hub |
| `StyleTotem` | Totem Model and its `Pillar` part | boolean | builder | §6.3.6 |
| `TotemRadius` | Totem Model | number | builder (from `Progression.TotemRangeStuds`, 12, Part 5 §5.10) | Used **only** for the ProximityPrompt `MaxActivationDistance`. The server range rule is Part 5 §5.1.3 (distance from the requester's HRP to the Totem **Model pivot** ≤ `Progression.TotemRangeStuds`); the server never reads `TotemRadius` (§6.3.6) |
| `Vol` | `Zones.Volumes` parts | string `"Spring"`/`"Slow"`/`"Boundary"` | builder | §6.1.7 |
| `LaunchVy` / `SlowMult` / `Inward`, `BandStuds`, `MinMult` | volume parts | number / number / Vector3, number, number | builder | §6.1.7 |
| `KillVolume` | `Zones.KillVolumes` parts | boolean | builder | §6.1.8 |
| `EventZone` | `Zones.EventZones` parts | string | builder | World-event stub zones (§6.3.7) |
| `GuardianStub`, `PatrolStub` | stub Models | boolean | builder | Future-hook markers (§6.3.7) |
| `DummySlot` | `DummyPoints` parts | number | builder | Part 3 §3.15.2 |
| `Mode` | `DummyPoints` parts | string | builder (`"idle"`), then BotService | Current dummy mode, replicated for the Training UI (§6.4, §6.13 item 7) |

#### 6.1.5 Collision groups

Part 3 §3.12.1 lists all eight groups (8 of 32), and `Combat/Hurtbox.ensureGroups()` (Part 3's single registration point) registers them and sets the full matrix, so the matrix lives in one place and persists in the place file (§6.13 item 2). Two of the eight exist for this part; Part 6 adds no second registration point:

| Group | Members | Collidable with | Why |
|---|---|---|---|
| `MapBoundary` | invisible boundary walls and caps (`Kind = "Boundary"`) | `Characters`, `RagdollFx` | Keeps characters in the map without being a wall-splat surface: `WorldQuery` (the wall probe) does not collide with it |
| `RagdollFx` | client-only ragdoll clones and breakable debris (§6.7, §6.1.8) | `Default`, `MapBoundary` | Debris and ragdolls must land on floors. They must never touch `Characters` (pushing a predicted character mispredicts) |

Client visuals that must not collide at all (proxy look parts, gear, rings, VFX parts) use Part 3's `CosmeticNoQuery` with `CanQuery = false`, `CanCollide = false`, `CanTouch = false`.

**Verify:** `BuildArena.verify()` and `BuildMap_CandyKingdom.verify()` report `[builder:<Name>] group MapBoundary missing (run BuildInputs)` when `workspace:IsCollisionGroupRegistered("MapBoundary")` is false, and check the matrix with `workspace:CollisionGroupsAreCollidable` (`MapBoundary`×`Characters` true, `MapBoundary`×`WorldQuery` false, `RagdollFx`×`Characters` false, `RagdollFx`×`Default` true).

#### 6.1.6 Terrain usage

Only `candy_kingdom` uses Terrain. The arena and Training are part-only.
- **Materials and colours** are set once per build with `Terrain:SetMaterialColor(material, color)`. This recolours the material everywhere, so no other map may use these terrain materials.

  | Material | Colour | Use |
  |---|---|---|
  | `Grass` | `#F8BBD0` | Outer ring outside the boundary |
  | `LeafyGrass` | `#B2DFDB` | Royal Gardens |
  | `Pavement` | `#FFE0B2` | Candy Streets |
  | `Ground` | `#FFF59D` | Barracks yard |
  | `Sand` | `#E6EE9C` | Earldom border, hills |
  | `Rock` | `#CE93D8` | Decorative hills outside the boundary |

- **`Terrain.Decoration` is human-set at H-04** (unchecked; GDD §12.5: no realistic 3D grass). The property is tagged `NotScriptable` (`Terrain.yaml`), so no Luau (builder, command bar or `execute_luau`) can read or write it; the builder never touches it.
- **No Terrain water.** The chocolate river is a part channel plus a `Slow` volume (§6.3.5). Swimming would change Humanoid state and physics under prediction.
- Fill order (deterministic; a `reset` build first clears the full region with `Terrain:FillBlock(cf, size, Enum.Material.Air)`):
  1. `Grass` slab: `FillBlock` region x[−360, 360] × z[−360, 360], y[−4, 0].
  2. Zone materials over the same y range: Streets x[−300,−100] z[−300,100] `Pavement`; Barracks x[−100,300] z[−300,−100] `Ground`; Gardens x[100,300] z[−100,300] and x[−100,100] z[100,300] `LeafyGrass`; Earldom x[−300,−100] z[100,300] `Sand`.
  3. Clear (`Air`) the part-floored regions: Plaza x[−100,100] z[−100,100]; dungeon roof x[−200,−100] z[−70,70]; river channel x[100,300] z[52,68].
  4. Hills: `FillBall` with `Sand`: centres `(−250,−6,150)` r 16, `(−180,−8,230)` r 18, `(−130,−6,160)` r 14, `(−240,−8,270)` r 16. Outside the boundary, `Rock` hills at the four corners `(±340, −10, ±340)` r 30.
- Terrain voxels are not instances, so terrain writes do not increment `created`/`updated`. The idempotency test (Part 1 §1.6.8) still passes.

**Verify:** `verify()` checks `Terrain:GetMaterialColor(m)` for the six materials (tolerance 1/255) and reads 6 sample voxels with `Terrain:ReadVoxels` (one per zone plus the plaza hole, which must be `Air`). It does **not** read `Terrain.Decoration` (NotScriptable); it adds the non-failing reminder line `[ALG][BUILD] reminder Terrain.Decoration is human-set at H-04 (NotScriptable)` to its output (not a problem string).

#### 6.1.7 Predicted volumes (springs, chocolate river, soft boundary)

Movement effects must be **predicted** (computed identically on the owner client and the server inside the sim). A server-only launch would rubber-band. Volumes are therefore static data read by both peers.

**Data.** Each volume is an invisible Part under `<Map>.Zones.Volumes` whose world AABB (from `CFrame`/`Size`; volumes are never rotated) defines the region:

| `Vol` | Attributes | Effect (applied by the Adapter, inside the sim) |
|---|---|---|
| `Spring` | `LaunchVy` (studs/s) | If the root is inside, `St ∈ NEUTRAL` and `AssemblyLinearVelocity.Y ≤ 1`: `Humanoid:ChangeState(Enum.HumanoidStateType.Freefall)` then `root.AssemblyLinearVelocity = Vector3.new(v.X, LaunchVy, v.Z)` |
| `Slow` | `SlowMult` | While inside and grounded: walk speed × `SlowMult` |
| `Boundary` | `Inward` (unit Vector3), `BandStuds`, `MinMult` | While inside the band and moving outward (`moveDir · Inward < 0`): walk speed × `1 − (1 − MinMult) · clamp(depth / BandStuds, 0, 1)` |

**Pure function** (added to Part 3's `Core/Movement.luau`; Lune-testable):

```lua
--!strict
-- Addition to src/Shared/Core/Movement.luau (pure; no Instances). Spec §6.1.7.
export type VolumeKind = "Spring" | "Slow" | "Boundary"
export type Volume = {
	kind: VolumeKind, min: Vector3, max: Vector3,   -- world AABB
	launchVy: number?, slowMult: number?,
	inward: Vector3?, bandStuds: number?, minMult: number?,
}
export type VolumeEffect = { speedMult: number, launchVy: number? }

local function inside(p: Vector3, v: Volume): boolean
	return p.X >= v.min.X and p.X <= v.max.X and p.Y >= v.min.Y and p.Y <= v.max.Y
		and p.Z >= v.min.Z and p.Z <= v.max.Z
end

-- Returns the walk-speed multiplier (rounded to 0.01, Q-rule) and an optional spring launch speed.
function Movement.volumeEffect(pos: Vector3, vel: Vector3, grounded: boolean, neutral: boolean,
	moveDir: Vector3, vols: { Volume }): VolumeEffect
	local mult = 1
	local launch: number? = nil
	for _, v in vols do -- array order is deterministic (builder order)
		if inside(pos, v) then
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
1. Outside the sim, at `track` time, the roster entry caches `entry.vols: { Movement.Volume }` for the entry's map. The list is built once per map from `<Map>.Zones.Volumes` children sorted by `Name`, cached per `MapId`, and rebuilt on `Zones.Volumes.ChildAdded` (client streaming safety; `Zones` is Persistent).
2. Inside phase 1, after `Fsm.step` and before `Loco.apply`: `local ve = Movement.volumeEffect(root.Position, root.AssemblyLinearVelocity, ctx.grounded, inNeutral, inp.moveDir, entry.vols)`; `loco.walkSpeed *= ve.speedMult`; if `ve.launchVy` then change state and write the velocity as in the table. `BasePart.AssemblyLinearVelocity` and `Humanoid:ChangeState` are Simulation-Access [Docs `BasePart.yaml`, api-sheet §1.5]. Whether the launch predicts without rollback is spike S-P6-5.

**Verify:** Lune `tests/core/movement_volumes.spec.luau`: `movement.volume_spring_neutral_only` (Idle → launch; Hitstun → none; vy = 5 → none), `movement.volume_slow_grounded` (0.6 grounded, 1.0 airborne), `movement.volume_boundary_depth` (depth 0 → 1.0, 10/20 → 0.6, 20/20 → 0.2, moving inward → 1.0), `movement.volume_order_deterministic`. Scenario `world.spring_launch` (character walks onto Training spring `Spring_Test`; within 30 frames the server reads root Y ≥ pad top + 20; the client `RunService:GetPredictionStatus(root)` stays `Predicted` and the Part 7 misprediction counter for `St` stays 0). Scenario `world.river_slow` (WalkSpeed reads 18 × 0.6 = 10.8 on both peers while in the channel).

#### 6.1.8 Server world runtime (void kills, breakables, rebuild)

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
}
```

**Void and kill volumes.** On each scan tick, for every ready roster entry (`Adapter.list()`) whose `St ≠ KO` and that has not already been killed this life:
- if `root.Position.Y − mapOrigin.Y < map:GetAttribute("KillY")`, or the root is inside any `KillVolume` AABB of its map → `CombatService.killAsync(target, "environment")` once, and log `[ALG][World] void_kill rid=<n> map=<mapId>`.
- KO credit: GDD §11.1 says hazards and environment KOs credit "the last attacker within 5 s". Part 3 owns the rule: `Combat.Streak.EnvKoCreditWindowSec = 5` (Part 3 §3.17) and §3.13.2 credit `lastHit.rid` for `killAsync(…, "environment")` only if `f − lastHit.f ≤ toFrames(EnvKoCreditWindowSec)` (the general 10 s `KoCreditWindowSec` applies to all other KOs). The world runtime only passes `cause = "environment"`; `Perf.World.Hazard` holds no credit window (§6.11, §6.13 item 4). Scenario `combat.env_ko_credit` (Part 3 §3.13 Verify).

**Breakables.** A Breakable is a `Model` with `Breakable = true` holding pre-split chunk Parts (kind `Breakable`).

| Step | Rule |
|---|---|
| Contact (server) | Every scan tick, for each entry whose root lies inside the model's bounding box inflated by `BreakInflateStuds` (1.5, td), the entry is a *breaker* when `St ∈ {Ragdoll, Launched, WallSplat}` (the character was slammed into it), or when the model has `BreakFromAir = true` and `St == Air` with `|AssemblyLinearVelocity.Y| ≥ BreakMinVy` (40, td; hatches only). Dedup key `"<rid>:<StF>"` per model, so one slam deals one point |
| Damage | `Hp −= 1`. At `Hp > 0`: `State = "cracked"`. At `Hp ≤ 0`: break |
| Break | Each chunk: `CanCollide = false`, `CanQuery = false`, `Transparency = 1`. Model: `State = "broken"`, `BrokenAt = now`, `RebuildAt = now + RebuildSec` (90 s, GDD §11.1). Log `[ALG][World] break id=<name> cause=<slam\|air\|dev>` |
| Rebuild | At `now ≥ RebuildAt`: if any root is inside the inflated box, postpone by 1 s; else restore chunk `CanCollide = true`, `CanQuery = true`, `Transparency = T0`, `State = "intact"`, `Hp = BreakHp`. Log `[ALG][World] rebuild id=<name>` |
| Client | VfxController watches `State` through `GetAttributeChangedSignal` (allowed: this is not a predicted one-shot). On `cracked`: darken chunks 15 % locally plus a puff. On `broken`: spawn pooled debris (clones of the chunks, unanchored, group `RagdollFx`, outward impulse, lifetime random in `Perf.Vfx.DebrisLifeSec` = 6–10 s, GDD §11.1). During the last `Perf.Vfx.RebuildRedrawSec` (3 s, td) before `RebuildAt`, the "storybook redraw": local ghost copies of the chunks fade from transparency 1 to 0.5, each with an ink outline made from a second copy 0.1 stud larger in `#1A1A1A` at transparency 0.7 |

`now` is `workspace:GetServerTimeNow()` (§0.4 meta clock).

Placement rules (`verify()` enforces them):
- No breakable within `Perf.World.Breakable.SpawnClearStuds` (60) of any spawn hub centre. GDD §11.1: destruction never opens sightlines into spawns.
- A hatch has an open shaft under it and a `Spring` volume on the floor below, so nobody is ever trapped.

**Optional fracture (GDD §18.8).** By default, chunks are deterministic quarters (§6.3.4). With `build({fragment = true})` the builder runs `GeometryService:GenerateFragmentSites(part, {SiteSpacing = 6})` and `GeometryService:FragmentAsync(part, sites, {SplitApart = true})` once per **template** (one house, one hatch) and stores the MeshParts under `ServerStorage.Templates.Fragments.<Template>`. Later builds reuse the cache unless `opts.refragment = true`. Site generation is not repeatable [api-sheet §18.6], so idempotency holds only because the cache is reused.

**Verify:** scenarios `world.void_kill` (character teleported to `KillY − 5` → `St = KO` within 0.2 s; log line present), `world.house_break_rebuild` (bot Ender into `House_2_0` twice → `State` `cracked` then `broken`; debris count > 0 on the client; `Dev.Command setTimeOffset 90` → `State = "intact"`), `world.hatch_fall_and_spring` (break `Hatch_E`; a character on it falls to y ≈ −33; standing on `DungeonSpring_E` launches it above y = 0 and breaks the rebuilt hatch from below), `world.rebuild_postponed_when_occupied`.

#### 6.1.9 Budgets and common `verify()` checks

Budgets (from `Perf.World.Budget`, §6.11):

| Map | Max BaseParts | Max breakables | Spawns | Totems | Source |
|---|---|---|---|---|---|
| `arena_duel_m0` | 400 | 0 | 8 | 0 | (td) |
| `training` | 400 | 0 | 2 | 1 | (td) |
| `candy_kingdom` | 2,500 | 42 | 16 | 4 | (td); GDD §11.1 (4 hubs) |

On-screen triangles and draw calls (GDD §12.8 map ≤ 220k / 450k / 800k triangles; GDD §18.7 draw calls ≤ 600 / 900 / 1,500) are measured by Part 7's perf pass with the Render Stats overlay. Graybox parts are cheap, so part count is the builder-side proxy.

Every map builder's `verify()` runs these checks; each problem string starts with `[builder:<Name>]`:
1. Root exists, is owned, has `MapId`, `KillY`, `Origin`.
2. Every BasePart has `Kind`; its flags (`CanCollide`, `CanQuery`, `CanTouch = false`, `CollisionGroup`, `Anchored = true`) match `Util.KIND`.
3. Counts match the budget table (spawns and totems exactly; parts and breakables at most).
4. Spawn parts face the map centre within 5°.
5. Breakable placement rules (§6.1.8).
6. Hazards (if any): `HazardDamage ≤ 15`, `HazardTelegraphSec ≥ 1.5` (GDD §11.1).
7. `Zones` is a Model with `ModelStreamingMode == Enum.ModelStreamingMode.Persistent`; every part under `Zones.Volumes` has a valid `Vol` and that kind's attributes; every `ZoneBounds` part has a `ZoneId`.
8. **Wall splat and verticality per zone** (GDD §11.1; Candy Kingdom only): each zone has ≥ 4 parts with `WallSplat = true`, and ≥ 1 walkable top surface at local Y ≥ 8 (layer 2).
9. **Sightline audit** (GDD §11.1, ≤ 150 studs; Candy Kingdom only): from every point of a 20-stud grid inside the playable square, at local Y = 5, cast `workspace:Raycast` in 8 compass directions with length 160, using `RaycastParams` with `CollisionGroup = "WorldQuery"` and `ExcludeInstances = {Zones}`. A ray that travels more than `Perf.World.Sightline.MaxStuds` (150) without a hit and whose end stays inside the playable square is a problem: `sightline <len> from (x,z) dir <d>`. **Fix rule for the agent:** add a `Cover` block (6 × 6 × 6, zone colour) to the layout table at the ray midpoint, snapped to the nearest street centre or open area, then re-run. The fix is a data change, so the build stays deterministic.

Studio snippet `tools/studio/map_stats.luau` (Edit; ARGS `{mapId}`; ≤ 1.5 KB): returns `{parts, byKind, byZone, breakables, spawns, totems, worstSightline}` as JSON.

**Verify (all builders):** Part 1 §1.6.8 tests (`build_fresh`, `build_idempotent`, `build_reset_equal`, `verify_detects`, `no_sync_side_effects`) for each of `BuildArena`, `BuildTraining`, `BuildMap_CandyKingdom`, `BuildLighting`, `BuildTemplates`, `BuildVfxTemplates`, `BuildPlaceholderAnims`.

---

### 6.2 M0 duel arena `arena_duel_m0`

An IP-neutral take on GDD §11.2 D's competitive standard: a pale "cube atrium" of 120 × 120 studs with Escher-like stairs and cubes and **no hazards**. Display name: `Skin.text("map.arena_duel_m0.name")`.

#### 6.2.1 Layout (map-local; origin `(−2600, 0, 0)`)

| Name | Kind | Shape / size | Centre | Notes |
|---|---|---|---|---|
| `Floor_<i>_<j>` (36) | Floor | Block 20 × 4 × 20 | x, z ∈ {−50, −30, −10, 10, 30, 50}, y −2 | Colour `#ECEAF8` when `(i+j)` is even, else `#F7F5FF`. The tiles are the scale reference (agent-assets §7.3) |
| `Wall_N` / `Wall_S` | Wall, `WallSplat` | 128 × 30 × 4 | (0, 15, ∓62) | Colour `#DCD6F2` |
| `Wall_W` / `Wall_E` | Wall, `WallSplat` | 4 × 30 × 120 | (∓62, 15, 0) | |
| `Trim_N/S/W/E` | Trim | 120 × 0.6 × 0.4 (N/S); 0.4 × 0.6 × 120 (W/E) | inner face base: (0, 0.3, ∓59.8), (∓59.8, 0.3, 0) | GDD §11.1 ink border |
| `Cap_N/S/W/E` | Boundary | 128 × 60 × 4 / 4 × 60 × 120 | (0, 60, ∓62) / (∓62, 60, 0) | Invisible caps from y 30 to 90: uppercut follow-ups (20 studs + hang) cannot land on wall tops |
| `Cube_C` | Cover, `WallSplat` | 16 × 12 × 16 | (0, 6, 0) | Central cube; top = layer 2 at y 12. Colour `#B39DDB` |
| `Cube_NE/NW/SE/SW` | Cover, `WallSplat` | 12 × 12 × 12 | (±36, 6, ±36) | Colour `#CFC6EE` |
| `Ramp_<corner>` (4) | Floor (WedgePart via `Util.ramp`) | width 8, rise 12 | low (±12, ·, ±36) → high (±30, ·, ±36), same sign as the cube | Climbs along X to each corner cube's inner face |
| `Bridge_<corner>` (4) | Floor | 4 × 1 × 31.1 | (±19, 11.5, ±19), yaw ±45° along the diagonal | From each corner cube's inner corner (±30, ±30) to the central cube's corner (±8, ±8); layer 2 is connected |
| `EscherCube_<n>` (4) | Decor `#F48FB1` | 6 × 6 × 6 | (±20, 24, 0), (0, 28, ±20) | Floating, non-colliding |
| `Spawn_<n>` (8) | Spawn, `SpawnFor = "arena_duel_m0"` | 4 × 1 × 4 | (−24, 0.5, −50), (24, 0.5, −50), (−24, 0.5, 50), (24, 0.5, 50), (−50, 0.5, −24), (−50, 0.5, 24), (50, 0.5, −24), (50, 0.5, 24) | `CFrame.lookAt(pos, origin-height centre)`. 8 spawns serve 8 clients + bots (§0.0 M0) |
| `Zones.KillVolumes.Kill_Floor` | KillVolume | 200 × 10 × 200 | (0, −50, 0) | Safety net |

Map attributes: `MapId = "arena_duel_m0"`, `KillY = −40`. Lighting preset `arena_duel_m0.Day` (§6.5).

**Out of bounds.** Walls plus caps make leaving impossible. Falling through geometry hits `KillY`. The arena has no volumes.

**Sightlines.** The duel arena is exempt from the 150-stud rule (GDD §11.1 applies to Battlegrounds maps), but the central and corner cubes still break every centre-to-corner line.

#### 6.2.2 M0 spawn selection

Part 2's default `MapId` is `"candy_kingdom"`, which does not exist in M0. PlayerService must set the initial `MapId` to `"candy_kingdom"` when `Workspace.Map.CandyKingdom` exists, else `"arena_duel_m0"` (§6.13 item 6). The HUD travel button sends `World.Travel {mapId = "training"}` and back. The return target is `"candy_kingdom"`, or `"arena_duel_m0"` when CandyKingdom is absent, which needs `"arena_duel_m0"` added to the `World.Travel` enum (§6.13 item 6).

**Verify:** `BuildArena.verify()` `{ok = true}` with 36 + 4 + 4 + 4 + 5 + 4 + 4 + 4 + 8 + 1 = 74 BaseParts. Scenario `map.arena_spawn_8` (8 bots + 1 player spawn; every spawn CFrame faces the centre; no two characters spawn within 4 studs). Scenario `map.arena_no_escape` (scripted uppercut follow at a wall: the attacker's root Y never exceeds 31 while |x| or |z| > 58). MCP `screen_capture` from `(0, 40, 90)` looking at the centre, for human review of layer 2 readability.

---

### 6.3 Candy Kingdom graybox `candy_kingdom`

Playable 600 × 600 studs (x, z ∈ [−300, 300]) with a soft boundary, 6 zones, 4 spawn hubs with Style Totems, and graybox interactables (GDD §11.1, §11.2 A). Display names: `map.candy_kingdom.name` and `map.candy_kingdom.zone.<ZoneId>`.

#### 6.3.1 Zones

`ZoneBounds` parts (invisible, `Kind = "Volume"`, no `Vol` attribute) carry `ZoneId`. A point belongs to the first matching box in this order (dungeon first, because it lies under other zones):

| Order | `ZoneId` | Boxes: x range, z range, y range | Silhouette / verticality |
|---|---|---|---|
| 1 | `dungeon` | [−190, 70], [−60, 60], [−40, −4] | Low pillared hall under the plaza and streets |
| 2 | `castle_plaza` | [−100, 100], [−100, 100], [−4, 90] | Tiered cake castle; balconies at y 20 / 36 / 50 |
| 3 | `candy_streets` | [−300, −100], [−300, 100], [−4, 40] | Dense gumdrop-house grid; rooftops at y 14 |
| 4 | `barracks` | [−100, 300], [−300, −100], [−4, 40] | Open yard, two halls (roofs y 16), two towers (y 24) |
| 5 | `royal_gardens` | [100, 300], [−100, 300] and [−100, 100], [100, 300], y [−4, 40] | Cotton-candy canopies (y 20), licorice maze, gazebo roof (y 13) |
| 6 | `earldom_border` | [−300, −100], [100, 300], [−4, 40] | Sour hills (up to y ≈ 10), lookout block (y 10) |

The six zones match GDD §11.2 A. "Earldom border" sits in the south-west corner. The GDD's castle view is a decorative silhouette outside the boundary (§6.3.7).

#### 6.3.2 Castle Plaza (`Geometry.castle_plaza`)

| Name | Kind | Shape / size | Centre / ends | Notes |
|---|---|---|---|---|
| `Plaza_Floor_<k>` | Floor `#F8BBD0` | 4 blocks from `Util.slabRects({−100,100,−100,100}, {{35,45,35,45}})`, thickness 4 | top at y 0 | Leaves the `Hatch_E` hole at (40, 40) |
| `Castle_T1` | Wall `#FFF3D6`, `WallSplat` | 64 × 20 × 64 | (0, 10, 0) | Top = balcony 1 (y 20) |
| `Castle_T2` | Wall, `WallSplat` | 44 × 16 × 44 | (0, 28, 0) | Top y 36 |
| `Castle_T3` | Wall, `WallSplat` | 28 × 14 × 28 | (0, 43, 0) | Top y 50 |
| `Castle_Spire` | Wall | Cylinder Ø10 × 24 | (0, 62, 0) | Landmark, top y 74. The castle Model sets `LevelOfDetail = Enum.ModelLevelOfDetail.StreamingMesh` so it reads from afar |
| `Ramp_N`, `Ramp_S` | Floor | width 12, rise 20 | low (0, ·, ∓72) → high (0, ·, ∓32) | Ground to balcony 1 |
| `Ramp_T12` | Floor | width 8, rise 16 | low (27, ·, 14) → high (27, ·, −14), y0 = 20 | On the east ring of balcony 1, beside tier 2 |
| `Ramp_T23` | Floor | width 6, rise 14 | low (−18, ·, −12) → high (−18, ·, 12), y0 = 36 | On the west ring of tier 2 |
| `Spring_PlazaE`, `Spring_PlazaW` pads | Floor `#7FD8BE` | Cylinder Ø8 × 1 | (±44, 0.5, 0) | Visual pads; volumes in §6.3.6 |
| `Planter_<n>` (8) | Cover `#F48FB1`, `WallSplat` | 4 × 6 × 12 at x = ±70; 12 × 6 × 4 at z = ±70 | (±70, 3, ±30), (±30, 3, ±70) | Line-of-sight breakers |
| `AmberNiche` | Decor `#FFB74D`, transparency 0.4 | 4 × 7 × 4 | (0, 3.5, 36) | Easter-egg niche on the south face (GDD §11.2 A). Named neutrally |
| `Gate_N_L/R`, `Gate_N_Lintel` | Wall | 8 × 24 × 8 pillars at (∓16, 12, −100); lintel 40 × 4 × 8 at (0, 26, −100) | | North gate |
| `Gate_S_L/R`, `Gate_S_Lintel` | Wall | mirror at z = +100 | | South gate |
| `Lamp_<n>` (8) | Decor `#B39DDB` | Cylinder Ø1 × 10 | 6 studs outward from each planter | Non-colliding |

#### 6.3.3 The Dungeon (`Geometry.dungeon`)

| Name | Kind | Shape / size | Centre | Notes |
|---|---|---|---|---|
| `Dungeon_Floor` | Floor `#3A2F3F` | 260 × 4 × 120 | (−60, −38, 0) | Top y −36 |
| `Dungeon_Wall_N/S` | Wall `#2E2433`, `WallSplat` | 268 × 32 × 4 | (−60, −20, ∓62) | |
| `Dungeon_Wall_W/E` | Wall, `WallSplat` | 4 × 32 × 120 | (−192, −20, 0), (72, −20, 0) | |
| `Streets_Roof_<k>` | Floor (zone `candy_streets` colour) | `Util.slabRects({−200,−100,−70,70}, {{−155,−145,−5,5}})`, thickness 4, top y 0 | | Dungeon roof under the streets with the `Hatch_W` hole |
| `Pillar_<n>` (10) | Cover `#4A3D52`, `WallSplat` | 6 × 32 × 6 | x ∈ {−160, −110, −60, −10, 40}, z = ±25, y −20 | |
| `Grate_<n>` (6) | Decor `#FFE082`, Neon, transparency 0.4 | 8 × 0.2 × 8 | ceiling underside y −4.2 at x ∈ {−150, −100, −50, 0, 40, 60}, z = 0 (40 for x = 40) | "Lit by grates"; Neon emulates light (GDD §12.4: no dynamic lights) |
| `DungeonSpring_W/E` pads | Floor `#7FD8BE` | Cylinder Ø8 × 1 | (−150, −35.5, 0), (40, −35.5, 40) | Exits under the hatches |

Hatches (`Breakables`):

| Model | Chunks | Centre | Attributes |
|---|---|---|---|
| `Hatch_W` | 4 × Block 5 × 1 × 5 | (−150, −0.5, 0) | `Breakable = true`, `BreakHp = 1`, `BreakFromAir = true`, `RebuildSec = 90` |
| `Hatch_E` | 4 × Block 5 × 1 × 5 | (40, −0.5, 40) | same |

A hatch breaks when a character is slammed into it (Ender, Downslam spike), lands on it from height (|vy| ≥ 40), or is launched into it from the dungeon spring below. This makes the dungeon the GDD's "high-risk shortcut" between the streets and the plaza.

**Dungeon lighting overlay.** While the local root is inside the `dungeon` ZoneBounds box, CameraController blends in the `candy_kingdom.Dungeon` ColorCorrection overlay (§6.5) over 0.5 s.

#### 6.3.4 Candy Streets (`Geometry.candy_streets`, `Breakables`)

**House generator** (deterministic; implemented in the builder as data plus one loop):
- Rows `j = 0..11`: `z = −282 + 34·j`. Columns `i = 0..4`: `x = −272 + 34·i + (17 if j is odd else 0)` (brick offset, so north–south streets dog-leg every row).
- Skip a cell when (a) its centre is within 68 studs of hub W `(−270, 0)`; (b) it is within 24 studs of `Hatch_W (−150, 0)`; (c) `(i + 2·j) % 5 == 0` (small squares).
- Result: **exactly 40 houses**: `(i,j)` = (1,0) (2,0) (3,0) (4,0) (0,1) (1,1) (2,1) (4,1) (0,2) (2,2) (3,2) (4,2) (0,3) (1,3) (2,3) (3,3) (0,4) (1,4) (3,4) (4,4) (1,5) (2,5) (3,5) (4,5) (0,6) (1,6) (2,6) (4,6) (2,7) (3,7) (4,7) (3,9) (4,9) (2,10) (3,10) (4,10) (0,11) (1,11) (2,11) (4,11).

Each house `House_<i>_<j>` is a Breakable Model (`BreakHp = 2`, `BreakFromAir = false`, `RebuildSec = 90`):
- 4 chunks `Q1..Q4`: Block 8 × 14 × 8 at `(x ± 4, 7, z ± 4)`, kind `Breakable`, `WallSplat = true`. Flat roofs at y 14 are layer 2.
- `Dome`: Decor Ball Ø10 at `(x, 14, z)`, colour cycling `#F48FB1` / `#7FD8BE` / `#B39DDB` by `(i + j) % 3`. It hides with the house.
- A `Trim` block (16.6 × 0.6 × 16.6 at `(x, 0.3, z)`) sits in `Geometry`, outside the Model, so the foundation border stays when the house breaks.

Two breaks are needed (first slam cracks, second breaks). Wall combos into houses therefore still work at least once (GDD §11.2 A: "good for wall combos; the houses are destructible").

Other street parts:

| Name | Kind | Size | Centre | Notes |
|---|---|---|---|---|
| `Stand_<n>` | Cover `#7FD8BE`, `WallSplat` | 6 × 6 × 6 | (−204, 3, zs) for each east–west street centre `zs = row centre + 17` ∈ {−265, −231, −197, −163, −129, −95, −61, −27, 41, 75} | "Gumball stands" that break the 200-stud east–west street lines. The street at z = 7 is skipped (inside the hub clearance) |
| `CandyWall_<k>` (3) | Wall `#FFE0B2`, `WallSplat` | 2 × 12 × L | x = −101; z spans [−100, −50], [−38, 52], [64, 100] | Streets–plaza border with gates at z ≈ −44 and z ≈ 58. Houses `(4,7)` and `(4,10)` stand behind the gates, so no straight line runs from the streets into the plaza |
| `Spring_S1`, `Spring_S2` pads | Floor `#7FD8BE` | Cylinder Ø8 × 1 | (−153, 0.5, −163), (−255, 0.5, −95) | Rooftop access |
| `StreetLamp_<n>` (10) | Decor | Cylinder Ø1 × 9 | street intersections at x = −187, z ∈ {−265, −197, −129, −61, 41} and x = −255, same z | |

#### 6.3.5 Barracks, Royal Gardens and the chocolate river

**Banana Guard Barracks** (`Geometry.barracks`):

| Name | Kind | Size | Centre / ends | Notes |
|---|---|---|---|---|
| `Hall_A` | Wall `#FFF176`, `WallSplat` | 80 × 16 × 24 | (150, 8, −270) | Roof y 16 |
| `Ramp_HallA` | Floor | width 10, rise 16 | low (230, ·, −270) → high (190, ·, −270) | |
| `Hall_B` | Wall, `WallSplat` | 24 × 16 × 60 | (260, 8, −180) | Roof reached by `Spring_B` |
| `Tower_1`, `Tower_2` | Wall, `WallSplat` | 10 × 24 × 10 | (40, 12, −150), (120, 12, −230) | Tops y 24 |
| `Crate_<n>` (8) | Cover `#C8B560`, `WallSplat` | 8 × 8 × 8 | (−60, 4, −250), (0, 4, −200), (70, 4, −180), (100, 4, −280), (180, 4, −130), (200, 4, −210), (250, 4, −120), (280, 4, −260) | Sightline breakers in the open yard |
| `YardDummy_<n>` (6) | Cover `#FFD54F` | Cylinder Ø2 × 5 | x ∈ {60, 80, …, 160}, z = −250, y 2.5 | Static props (not bots) |
| `PatrolStub` | Decor `#FFEB3B` | Cylinder Ø2 × 5 | (100, 2.5, −200) | `PatrolStub = true`; the NPC patrol is a future hook |
| `Spring_B`, `Spring_T1`, `Spring_T2` pads | Floor `#7FD8BE` | Cylinder Ø8 × 1 | (230, 0.5, −180), (40, 0.5, −132), (120, 0.5, −212) | |

**Royal Gardens** (`Geometry.royal_gardens`):

| Name | Kind | Size | Centre | Notes |
|---|---|---|---|---|
| `Tree_<n>_Trunk` (11) | Cover `#CE93D8`, `WallSplat` | Cylinder Ø2 × 14 | y 7 at (130,−70) (190,−60) (250,−80) (130,−20) (220,−20) (130,30) (200,35) (−70,140) (70,140) (−40,200) (40,200) | |
| `Tree_<n>_Canopy` (11) | Floor `#F8BBD0` (odd n `#B39DDB`) | Cylinder Ø14 × 6 | same x, z; y 17 | Walkable tops at y 20 ("branches", layer 2) |
| `Spring_G1`, `Spring_P` pads | Floor `#7FD8BE` | Cylinder Ø8 × 1 | (160, 0.5, −45), (0, 0.5, 225) | |
| `Hedge_<n>` (15) | Wall `#3E2723`, `WallSplat` | height 8, thickness 2 | see the maze table | Licorice hedge maze in x [150, 290] × z [100, 260] |
| `Gazebo_Pillar_<n>` (4) | Wall | 2 × 12 × 2 | (±9, 6, 191), (±9, 6, 209) | |
| `Gazebo_Roof` | Floor | 22 × 1 × 22 | (0, 12.5, 200) | Layer 2 |
| `Bed_<n>` (4) | Cover `#CE93D8` | 4 × 6 × 20 | (±50, 3, 180), (±50, 3, 240) | Promenade hedges |

Maze segments (centre; size):

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

The entrances are the gaps in segments 1/2 (x 210–230), 3/4 (x 250–270) and 5/6 (z 180–200).

**Chocolate river** (GDD §11.2 A "chocolate river (slows)"):

| Name | Kind | Size | Centre | Notes |
|---|---|---|---|---|
| `River_Bed` | Floor `#6D4C41` | 200 × 4 × 16 | (200, −4, 60) | Top y −2: a channel 2 studs deep (terrain cleared, §6.1.6) |
| `Bridge_1`, `Bridge_2` | Floor `#FFE0B2` | 10 × 1 × 22 | (160, −0.5, 60), (250, −0.5, 60) | |
| `Zones.Volumes.River_Slow` | Volume `Vol = "Slow"`, `SlowMult = Perf.World.Slow.Mult` (0.6, td) | AABB x [100, 300], y [−3, 2], z [52, 68] | | The root is at y ≈ 1 in the channel and y ≈ 3 on a bridge, so bridges are not slowed |

#### 6.3.6 Spawn hubs, Style Totems and volumes

**Hubs.** Four hubs at the edge midpoints (GDD §11.1): `N (0, −270)`, `S (0, 270)`, `W (−270, 0)`, `E (270, 0)`. Each hub has:
- a pad `Hub_<id>_Pad`: Spawn-kind visual, 36 × 0.4 × 36 at `(hx, 0.2, hz)`, colour `#43A047`, transparency 0.6, non-colliding;
- 4 spawn Parts `Spawn_<id>_<k>` (4 × 1 × 4) at `(hx ± 8, 0.5, hz ± 8)` with `SpawnFor = "candy_kingdom"` and `HubId = <id>`, each `CFrame.lookAt(pos, Vector3.new(0, pos.Y, 0) + origin)`;
- a Style Totem Model `Totem_<id>` at the hub centre.

There are no cover or breakables within 30 studs of a hub centre, and no breakables within 60.

**Style Totem** (`Totems.Totem_<id>`; also used in Training):
- `Base`: Totem kind, Cylinder Ø4 × 1 at y 0.5.
- `Pillar`: Totem kind, 2 × 8 × 2 at y 5. Attributes `StyleTotem = true`, `HubId`.
- `Orb`: Decor Ball Ø3 at y 10.5. The client cycles its colour through the palettes of the viewer's owned styles at 1 Hz.
- `ALG_TotemPrompt` (ProximityPrompt) under `Pillar`: `KeyboardKeyCode = Enum.KeyCode.E`, `GamepadKeyCode = Enum.KeyCode.DPadLeft` (free in Part 3's map), `HoldDuration = 0`, `MaxActivationDistance = Progression.TotemRangeStuds` (12, Part 5 §5.10; read from `Defaults` the same way as `Perf.World`, rule 1 of §6.1.2), `RequiresLineOfSight = false`, `ActionText = ""`, `ObjectText = ""`. The client fills the texts from `Skin.text("ui.totem.action")` / `Skin.text("ui.totem.object")`, because builders never write player-visible literals.
- Model attributes `StyleTotem = true`, `HubId`, `TotemRadius = Progression.TotemRangeStuds` (12). `TotemRadius` exists only to size the prompt (§6.1.4).
- Client: `ProximityPromptService.PromptTriggered` with a prompt named `ALG_TotemPrompt` → `UiController.open("StyleWheel", {mode = "totem", hubId = …})`.
- Server (Part 5 §5.1.3 is canonical): StyleService treats an equip as a totem equip when the distance from the requester's HRP to a Totem **Model pivot** (`Totem_<id>:GetPivot().Position`) of its current map is ≤ `Progression.TotemRangeStuds` (12) and the swap gate passes (`CombatService.canTravel(player)`). A totem equip applies **now**: the swap respawns the player through Part 3's random safe spawn on the same map (not necessarily that totem's hub; Part 5 DES-3 pending) (§6.13 item 8).

**Volumes** (`Zones.Volumes`; sorted by `Name` when cached):

| Name | `Vol` | AABB (x, y, z ranges) | Attributes |
|---|---|---|---|
| `Spring_PlazaE`, `Spring_PlazaW` | Spring | [±40, ±48], [0, 5], [−4, 4] | `LaunchVy = Perf.World.Spring.LaunchVy` (101) |
| `Spring_S1`, `Spring_S2`, `Spring_B`, `Spring_T1`, `Spring_T2`, `Spring_G1`, `Spring_P` | Spring | pad centre ± 4 in x/z; y [0, 5] | `LaunchVy = 101` |
| `DungeonSpring_W`, `DungeonSpring_E` | Spring | pad centre ± 4; y [−36, −31] | `LaunchVy = Perf.World.Spring.LaunchVyHatch` (130) |
| `River_Slow` | Slow | §6.3.5 | `SlowMult = 0.6` |
| `Band_N` | Boundary | [−300, 300], [−50, 200], [−300, −280] | `Inward = (0, 0, 1)`, `BandStuds = 20`, `MinMult = 0.2` |
| `Band_S` | Boundary | [−300, 300], [−50, 200], [280, 300] | `Inward = (0, 0, −1)` |
| `Band_W` | Boundary | [−300, −280], [−50, 200], [−300, 300] | `Inward = (1, 0, 0)` |
| `Band_E` | Boundary | [280, 300], [−50, 200], [−300, 300] | `Inward = (−1, 0, 0)` |

Spring speeds (td) come from apex = v² / (2g) with g = 196.2 (Part 3 `Chassis.Gravity`): 101 studs/s gives an apex of ≈ 26 studs (balcony 1, rooftops, canopies, tower tops); 130 studs/s gives ≈ 43 studs (out of the dungeon through a hatch).

#### 6.3.7 Soft boundary, kill volume and stubs

| Name | Kind | Size | Centre | Notes |
|---|---|---|---|---|
| `Boundary_N/S` | Boundary | 620 × 200 × 4 | (0, 100, ∓306) | Invisible, group `MapBoundary` |
| `Boundary_W/E` | Boundary | 4 × 200 × 620 | (∓306, 100, 0) | |
| `FogWall_N/S/W/E` | Decor `#D1C4E9`, transparency 0.55 | 660 × 60 × 2 / 2 × 60 × 660 | (0, 30, ∓318), (∓318, 30, 0) | "Painted fog wall" (GDD §11.1) |
| `Zones.KillVolumes.Kill_Under` | KillVolume | 800 × 10 × 800 | (0, −70, 0) | Map `KillY = −60` |
| `Guardian_N`, `Guardian_S` (Models) | Cover `#F48FB1` | Base 12 × 6 × 12 at y 3; Ball Ø16 at y 14 | (−40, ·, −108), (40, ·, 108) | `GuardianStub = true`. GDD §11.2 A Gumball Guardians at two gates; the world event is a future hook (`Features.WorldEvent = false`). **Original look set:** see the variant row below |
| `Guardian_N`, `Guardian_S` (Models), **Original look set** | Cover, default `Util.KIND.Cover` colour `#3A7BD5` (no pink) | Generic gate tower: `Tower` Block 12 × 22 × 12 at y 11, plus 4 `Merlon_<k>` Blocks 3 × 3 × 3 on the top corners (y 23.5, offsets ±4.5 in x and z): a crenellated top. **No head, no eye, no Ball** | same | Built instead of the row above when the look set is Original. `BuildMap_CandyKingdom.build(opts)` reads `opts.lookSet`, which defaults to `"Original"` when `Defaults.SkinPack == "Original"`, else `"Codename"` (same rule as `BuildTemplates`, Part 8 §8.5.6). Same Model names, `GuardianStub = true`, 5 parts per Model. The function (gate marker, event stub) is kept; the canon landmark is not |
| `Zones.EventZones.Guardian_N/S` | Event | 30 × 1 × 30 | (0, 0.05, ∓120) | `EventZone = "guardian_n" / "guardian_s"` |
| `Zones.EventZones.Inspection` | Event | 100 × 1 × 100 | (−210, 0.05, 230) | `EventZone = "inspection"` (GDD §11.2 A event trigger stub) |
| `Earldom_Stone_<n>` (3) | Cover `#C0CA33`, `WallSplat` | 10 × 10 × 10 | (−200, 5, 140), (−150, 5, 280), (−270, 5, 210) | |
| `Earldom_Lookout` | Cover, `WallSplat` | 16 × 10 × 16 | (−130, 5, 250) | Top y 10; `Ramp_Lookout` low (−130, ·, 282) → high (−130, ·, 258), width 8, rise 10 |
| `SourCastle_<n>` (4) | Decor `#FFF176`, transparency 0 | main block 60 × 80 × 40 at (−340, 40, 340) plus 3 towers 12 × 100 × 12 around it | outside the boundary | Neutral "sour castle" silhouette in view of the corner |

"Sugar-rush candies" (GDD §11.2 A) are a future hook. They need a pickup system that M1-G does not have.

**Verify (§6.3):** `BuildMap_CandyKingdom.verify()` `{ok = true}` with: 16 spawns, 4 totems, 42 breakables (40 houses + 2 hatches), 11 springs, 1 slow volume, 4 boundary bands, 6 zones, the sightline audit clean, per-zone wall-splat and layer-2 checks passing, and ≤ 2,500 BaseParts; built with the Original look set, `Guardian_N/S` each hold the gate-tower parts (`Tower` + 4 `Merlon_<k>`) and no `Ball` part (Part 8 §8.5.6). Scenarios `map.ck_zone_label` (teleport to one point per zone → HUD zone label shows `map.candy_kingdom.zone.<id>`), `map.ck_boundary_soft` (walk outward from (0, 0, 270): WalkSpeed falls to ≤ 3.6 at the wall, root never beyond z = 304), `map.ck_totem_prompt` (prompt within 12 studs; triggering it opens the Style Wheel in totem mode). MCP `screen_capture` ×3 for human review (H-10): plaza from the south ramp, the streets at head height, the gardens from a canopy.

---

### 6.4 Training Grounds `training`

#### 6.4.1 Layout (map-local; origin `(2600, 0, 0)`)

| Name | Kind | Size | Centre | Notes |
|---|---|---|---|---|
| `Floor_<i>_<j>` (100) | Floor | 20 × 4 × 20 | x, z ∈ {−90, −70, …, 90}, y −2 | `#D7E3D0` / `#E3ECDD` alternating |
| `Wall_N/S` | Wall `#B9C8B0`, `WallSplat` | 208 × 20 × 4 | (0, 10, ∓102) | |
| `Wall_W/E` | Wall, `WallSplat` | 4 × 20 × 200 | (∓102, 10, 0) | |
| `Cap_N/S/W/E` | Boundary | as the walls, 60 tall | y 50 | |
| `SpawnPad` | Spawn (visual) | 24 × 0.2 × 12 | (0, 0.1, 70) | |
| `Spawn_1`, `Spawn_2` | Spawn, `SpawnFor = "training"`, `HubId = "T"` | 4 × 1 × 4 | (±6, 0.5, 70), facing −Z | |
| `Totem_T` | Totem (§6.3.6) | | (0, ·, 84) | Opens the Style Wheel in `trial` mode |
| `DummyPoints.Slot_<n>` (5) | Volume (invisible marker) | 2 × 1 × 2 | x ∈ {−40, −20, 0, 20, 40}, y 0.5, z −20, facing +Z | `DummySlot = n`, `Mode = "idle"` (Part 3 §3.15.2) |
| `Placard_<n>` (5) | Decor `#FFFFFF`, transparency 0 | 8 × 0.2 × 3 | (x, 0.1, −12) | Flat floor placards. `SurfaceGui` (`Face = Top`, `SizingMode = PixelsPerStud`, `PixelsPerStud = 40`, `LightInfluence = 0`) showing slot number and mode; the client fills the text |
| `SplatWall` | Wall `#9FB3A0`, `WallSplat` | 100 × 20 × 4 | (0, 10, −44) | Face at z = −42: 22 studs behind the dummies, inside the 38-stud Ender knockback, reached at about e = 10, well inside the 18f splat window |
| `AirDeck` | Cover, `WallSplat` | 30 × 8 × 20 | (70, 4, 30) | Air-combo practice platform |
| `Ramp_AirDeck` | Floor | width 8, rise 8 | low (70, ·, 58) → high (70, ·, 40) | |
| `Spring_Test` pad and volume | Floor / Volume `Spring` | Ø8 pad at (−70, 0.5, 30); AABB ± 4, y [0, 5] | | `LaunchVy = 101` |
| `Slow_Test` pad and volume | Floor `#6D4C41` / Volume `Slow` | 20 × 0.2 × 20 at (−70, 0.1, −60); AABB y [0, 4] | | `SlowMult = 0.6` |
| `Zones.KillVolumes.Kill_Floor` | KillVolume | 220 × 10 × 220 | (0, −50, 0) | Map `KillY = −40` |

Lighting preset `training.Day`.

#### 6.4.2 Training runtime and UI

All Training UI lives in `UI/Screens/Hud.luau`, shown only while `Player.MapId == "training"` (no extra files; §0.3).

| Panel | Anchor / position (ref units) | Contents | Actions |
|---|---|---|---|
| `TrainingPanel` | left-middle `(0, 0.5)`, offset (16, 0), 300 × 380 | Nearest dummy slot (by distance to `DummyPoints`, ≤ 40 studs) with its current `Mode`; 5 mode buttons (`idle`, `block`, `perfect_block`, `m1_spam`, `dash_spam`; labels `ui.training.mode_<mode>`); a **Damage numbers** toggle (`ui.training.damage_numbers`); [Try styles]; [Reset]; [Back to battle] | Mode → `Net.client.fire("Training.Dummy", {behavior = <mode>})` (Part 3 enum). Damage numbers → `SettingsController.set("damageNumbers", not current)`. [Try styles] → `UiController.open("StyleWheel", {mode = "trial"})`. [Reset] → `World.Travel {mapId = "training"}`. [Back] → `World.Travel {mapId = <battle map>}` (§6.2.2) |
| `FrameDataPanel` | right-middle `(1, 0.5)`, offset (−16, 0), 340 × 320 | Last action of the local player (below) | [Pin] keeps the panel open in Candy Kingdom only while in Studio (dev aid) |
| Damage numbers | world BillboardGuis | Damage per contact (HP delta of the victim) | **Off by default** (GDD §15.3: "off by default; can be enabled in Training Grounds and accessibility settings"). Shown when the settings key `damageNumbers` (boolean, default `false`, Part 2 §2.8.6) is true, in any map. Two toggles write that one key through `Settings.Set`: the `TrainingPanel` row `ui.training.damage_numbers` and Settings ▸ Accessibility `ui.settings.damage_numbers` (§6.9.9). RenderController reads it (§6.9.11); `Perf.Ui.DamageNumbersDefault = false` is the fallback before settings load |
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

**Style trial selector.** The Style Wheel in `trial` mode (§6.9.6) lists every `Styles.implemented()` style with a "Trial" badge. Selecting one sends `Style.Equip {styleId}` (accepted in Training when `Progression.TrainingTrialAllStyles` is true, Part 2 §2.7.4 row 4), then `World.Travel {mapId = "training"}` so the trial applies immediately.

**Reset.** `World.Travel {mapId = "training"}` respawns at a Training spawn (HP, cooldowns and Breakaway reset by a new life). Part 3's `canTravel` blocks travel for 8 s after combat (`Chassis.CombatTagSec`). This part asks that travel **from `training` to `training`** skip the combat-tag check, keeping only the STUN/KO check (§6.13 item 6). Dummies reset themselves (Part 3 `Respawn.Dummy`).

**Verify:** `BuildTraining.verify()` `{ok = true}` (100 tiles, 5 dummy points with `DummySlot` 1–5, 2 spawns, 1 totem). Scenarios `training.dummy_mode_ui` (pressing [Block] near slot 3 → slot 3 `Mode` attribute `"block"` within 0.5 s; placard text updates), `training.framedata_m1` (four M1 presses on an idle dummy → panel rows 6/3/12, 6/3/12, 7/3/13, 10/4/22 and observed contacts at e = 6, 6, 7, 10), `training.trial_equip` (select `monarch` → respawn with `EquippedStyle = "monarch"` in ≤ 6 s), `training.damage_numbers_toggle` (fresh profile: hitting a dummy shows no damage BillboardGui; the panel toggle → `settings.damageNumbers == true` in the next delta and the next hit shows one; Settings ▸ Accessibility shows the same value), `training.reset_travel` (Reset 2 s after hitting a dummy succeeds once §6.13 item 6 is implemented).

---

### 6.5 Lighting presets as data

GDD §11.4 / §12.4: `LightingStyle = Soft`, boosted ambient, `EnvironmentSpecularScale ≈ 0`, mild shadows, per-map `ColorCorrectionEffect`, mild `BloomEffect` (it only affects Neon "magic" parts because its threshold is high), `Atmosphere` for painted haze. No depth of field. Painted skies need uploads, so graybox uses the default `Sky` (agent-assets §2.4).

#### 6.5.1 Ownership and application

- `BuildLighting` (Edit) owns `Lighting` properties plus children `ALG_Atmosphere` (Atmosphere), `ALG_Sky` (Sky, defaults), `ALG_CC` (ColorCorrectionEffect, base), `ALG_CC_Fx` (ColorCorrectionEffect, neutral; runtime effects tween it) and `ALG_Bloom` (BloomEffect). It also owns `Workspace.Terrain.ALG_Clouds` (Clouds must be under Terrain). Each child carries `AlgBuilder = "BuildLighting"`. Other Lighting children are left alone; `verify()` warns about them.
- The Edit-time preset is `Perf.Lighting.EditPreset` (`"arena_duel_m0.Day"` until the Candy Kingdom task switches it to `"candy_kingdom.Day"`).
- **Runtime (client):** `Lighting` is global, and maps share one place, so CameraController applies the preset for the local player's `MapId` on every `MapId` change. These are local writes. It tweens numeric and colour values over `Perf.Lighting.TweenSec` (1 s on map change). The day/golden/night cycle (GDD §11.4: 45 min) is data-ready but off in M1-G (`CycleEnabled = false`). When on, phase = `floor((GetServerTimeNow() % CycleSec) / (CycleSec / 3))` → Day / Golden / Night, with transitions tweened over 10 s.
- Zone overlays (the Dungeon) only change `ALG_CC_Fx`.

#### 6.5.2 Presets (`Defaults.Perf.Lighting.Presets`; all `LightingStyle = "Soft"`, `EnvironmentSpecularScale = 0`, `GlobalShadows = true`, `PrioritizeLightingQuality = false`; values td unless cited)

| Preset | ClockTime | Brightness | Ambient | OutdoorAmbient | EnvDiffuse | ShadowSoftness | Exposure | Atmosphere (Density, Offset, Color, Decay, Glare, Haze) | ColorCorrection (Brightness, Contrast, Saturation, Tint) | Bloom (Intensity, Size, Threshold) | Clouds (Cover, Density, Color) |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `candy_kingdom.Day` | 14 | 2.2 | `#A88A9C` | `#E6C3D6` | 0.25 | 0.75 | 0.15 | 0.26, 0.15, `#F7D6E6`, `#C7A6D8`, 0, 1.0 | 0.02, 0.06, 0.18, `#FFF4F8` | 0.35, 24, 1.7 | 0.55, 0.55, `#FFFFFF` |
| `candy_kingdom.Golden` | 17.6 | 1.8 | `#B08A7C` | `#F0C3A0` | 0.25 | 0.75 | 0.10 | 0.30, 0.15, `#FFD8B0`, `#D99A8A`, 0, 1.6 | 0.02, 0.06, 0.22, `#FFE9D6` | 0.35, 24, 1.7 | 0.5, 0.55, `#FFE0C8` |
| `candy_kingdom.Night` | 21.5 | 1.2 | `#6A5A8A` | `#8C7AB8` | 0.2 | 0.8 | 0.25 | 0.32, 0.1, `#9C8AC8`, `#4A3A6A`, 0, 0.8 | 0.03, 0.05, 0.10, `#E4DEFF` | 0.45, 24, 1.6 | 0.4, 0.5, `#B8A8D8` |
| `candy_kingdom.Dungeon` (overlay, `ALG_CC_Fx` only) | — | — | — | — | — | — | — | — | −0.12, 0.10, −0.20, `#C8C0E0` | — | — |
| `arena_duel_m0.Day` | 12 | 2.6 | `#C4C0DE` | `#ECE8FA` | 0.3 | 1.0 | 0.25 | 0.40, 0, `#EEE8FF`, `#D6CCF5`, 0, 0.4 | 0.03, 0, −0.05, `#F6F4FF` | 0.25, 24, 1.8 | 0, 0, `#FFFFFF` |
| `training.Day` | 13 | 2.2 | `#9AA494` | `#D4DECC` | 0.25 | 0.75 | 0.10 | 0.25, 0.1, `#E3ECDD`, `#B9C8B0`, 0, 1.0 | 0, 0.05, 0.10, `#FAFFF6` | 0.3, 24, 1.7 | 0.4, 0.5, `#FFFFFF` |

All properties above were verified on `Lighting`, `Atmosphere`, `ColorCorrectionEffect`, `BloomEffect` and `Clouds` [Docs class YAMLs]. `Lighting.LightingStyle` is script-writable (security None) [Docs `Lighting.yaml`].

**Verify:** `BuildLighting.verify()` (`LightingStyle == Enum.LightingStyle.Soft`, `EnvironmentSpecularScale == 0`, the five owned children plus `Terrain.ALG_Clouds` present). Lune `perf.lighting_presets_complete` (every preset has every key; colours match `^#%x%x%x%x%x%x$`). Scenario `cam.lighting_follows_map` (travel to Training → `Lighting.ClockTime == 13` on the client within 1.5 s; back → 14).

---

### 6.6 Presentation architecture

#### 6.6.0 Overview and rules

```
RunService (client)
  BindToSimulation / BindToAnimation   ── Part 3: sim + combat animation (Adapter.animate)
  RenderStepped (BindToRenderStep)
    "ALG_Render"   Character+1  RenderController: snapshot diff → transitions → Vfx/Sfx/Camera/Appearance/Hud
    "ALG_Camera"   Camera+1     CameraController: shift-lock offset, shake, kick, cinematic
    "ALG_Hud"      Last         HudController: bars, cooldowns, meters
```

Rules (all controllers in this section):
1. Never write HRP attributes, constraint properties or Humanoid properties of any character. Presentation writes only client-local instances, `LocalTransparencyModifier`, `Lighting`/`SoundService` locally, and the local `Animator.PreferLodEnabled` override on remote characters (§6.6.1 "Animator LOD").
2. Never use `GetAttributeChangedSignal` for combat one-shots (Part 3 §3.11.8). Poll snapshots once per frame.
3. Guard every render step with `if RunService:IsResimulating() then return end` (the resim replays sim steps only, but the guard is cheap and protects shared handlers).
4. Effects started for **predicted** transitions (the local character's own state, or `Cnt` contact) are undoable within `UNDO_WINDOW = 15` frames (Part 3 `Render.UndoWindowFrames`; a constant, not config).
5. No yields in render handlers (Part 2 §2.10 item 6). Asset loading (`ContentProvider:PreloadAsync`) runs in `task.spawn` at `start()`.
6. All client-created instances live under `Workspace.ALG_Local` (Folder, created by RenderController `init()`; sub-folders `Vfx`, `Looks`, `Ragdolls`, `Debris`, `Sfx`). It never exists in Edit (`Verify` reports it as a leftover).
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
	RunService:BindToRenderStep("ALG_Render", Enum.RenderPriority.Character.Value + 1, step)
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

**Haptics** (GDD §18.4 "`HapticEffect` waveforms for hits on phones and pads"). RenderController `init()` creates one client-local `HapticEffect` `ALG_Haptic` under `Workspace.ALG_Local` with `Type = Enum.HapticEffectType.GameplayCollision` [Docs `HapticEffect.yaml`, `input/gamepad.md`]. It calls `ALG_Haptic:Play()` on the local attacker's `Cnt` 0→1 and on a local-victim `HP` decrease, at most once per 6 frames, only when `UserInputService.GamepadEnabled or UserInputService.TouchEnabled`, and never with Reduced Motion on. It is cosmetic and skipped while `IsResimulating()` like every render handler.

**Animator LOD** (GDD §18.4; Part 3 §3.11.2 contract). The server sets `Animator.PreferLodEnabled = false` on every player and bot character at `Hurtbox.prepare`, so attacker startups (the telegraphs) are never LOD-throttled. Every 0.5 s, from the `ALG_Render` step (outside the sim; the property is not Sim-Access), RenderController overrides it locally on each **remote** view's Animator (other players and bots; never the local character): `false` when the view's root is within 60 studs of the current `Camera.CameraSubject`'s position, `true` beyond. It writes only when the value changes, and skips the pass while `IsResimulating()`.

**Transition dispatch** (implements Part 3 §3.11.8; P = predicted for the local character):

| Detect | P | Vfx | Sfx (§6.10) | Camera | Appearance / face | Notes |
|---|---|---|---|---|---|---|
| enter `M1`/`Ender` | yes | `vfx.<style>.m1` swing (Slash_Smear on the right hand) at e = startup | `sfx.<style>.m1` at e = startup | — | `grit` → `shout` at active | |
| enter `Move` (Sub → slot) | yes | telegraph (§6.6.3) from `e = S − leadFrames` to `S − 1`; `def.vfx` at hitbox centre at `e ≥ S` (and at each `S + hitFrames[i] − 1` for barrages); `vfx.dash.puff` for Rush/Mobility | `sfx.tele.windup` at telegraph start if `damage > 10` (GDD §14.4); `def.sfx` at `e ≥ S` | — | `face.windup/active/recovery` from `Styles.anim(style)[slot].face` (Part 4) | One-shots keyed on `(actionId, threshold)`, fired on the first frame where `e ≥ threshold` (remote: `eR`) and `threshold > firedMax`; telegraph only while `Cnt == 0` and not `reachedActive` (see "Action identity") |
| enter `Dash` / `TechRoll` | yes | `vfx.dash.puff` | `sfx.dash.whoosh` | — | — | |
| enter `Block` | yes | `vfx.guard.bubble` (owner + everyone ≤ 40 studs) | `sfx.block.raise` | — | `block` | Bubble alpha follows `Grd` (0.2 → 0.6) |
| `Cnt` 0→1 (local attacker) | yes | `vfx.hit.light` (M1) or `vfx.hit.heavy` (Ender/move) at hitbox centre; impact frame if `def.signature` (§6.6.6) | `sfx.hit.light`/`heavy`, pitch rising with the local combo count | `light` (M1) / `medium` (Ender/move); hitstop kick (§6.6.6) | — | Undo: `Cnt` back to 0 within 15f **with `actionId` unchanged**, or `Rollback` to a frame ≤ `contactF` → fade (a `Cnt` reset by a new action is not an undo). Haptic pulse (see "Haptics") |
| `Cnt` 0→2 (blocked) | yes | `vfx.hit.block` | `sfx.hit.block` | `light` | — | |
| victim `HP` decrease | no | Highlight hit flash (3f, §6.6.3); damage number when the `damageNumbers` setting is true (§6.4.2, §6.9.11); local victim: haptic pulse | local victim: `sfx.vo.hurt` (throttled as a bark) | local victim: `light` if < 8 damage else `medium` | `hurt_light` / `hurt_heavy` | Combo counter update if this view is the focus target |
| enter `Hitstun`/`Stagger`/`Launched`/`Grabbed` | no | — | — | — | `hurt_*` | |
| `Flg` gains `PBHIT` (defender) | no | `vfx.pb.star` + gold outline flash (6f) | `sfx.block.perfect` (top priority after telegraphs, GDD §14.6) | `light` both | `perfect_block` | |
| enter `GuardBroken` | no | `vfx.guard.break` | `sfx.guard.break` | `medium` if local | `dizzy` | |
| enter `WallSplat` | no | `vfx.wallsplat.splat` at the wall contact (root + facing × −1.1) | `sfx.wallsplat.splat` | `medium` if local attacker/victim | `hurt_heavy` | |
| enter `Ragdoll` | no | — | `sfx.ragdoll.land` on first landing (`Sub` 0→n) | — | — | Ragdoll presenter `begin` at e ≥ 12 (§6.7) |
| leave `Ragdoll` → `Idle` | no | — | — | — | — | Presenter `finish(blend)` (wake-up, §6.7.1) |
| leave `Ragdoll` → any other state except `KO`; or Ragdoll re-entry (`Sub` reset to 0, or `StF` increased) | no | — | — | — | — | Presenter `finish(instant)`, then the normal `begin` rule for a new Ragdoll (§6.7.1) |
| enter `Breakaway` | yes | `vfx.breakaway.burst` | `sfx.breakaway.burst` | — | `determined` | Presenter `finish(instant)` |
| enter `Awakening` | yes | `vfx.awaken.burst` + persistent `vfx.awaken.aura` (palette) | `sfx.awaken.sting` + bark `ui.vo.awaken` | local: 1.2 s cinematic (GDD §6.9) + `heavy`; others ≤ 30 studs: `medium` | `awaken` | Aura lives while `AwkF0 > 0` |
| `AwkF0` > 0 → 0 | no | power-down puff | — | — | `neutral` | |
| enter `KO` | no | `vfx.ko.poof` at `BodyHideSec` (1.2 s, Part 3); hide the look | `sfx.ko.poof` | — | `ko` | Local KO opens the Style Wheel (§6.9.6) |
| `ProtF > fNow` | no | `vfx.spawn.shimmer` while true | `sfx.spawn.ready` when it ends | — | — | Part 3 §3.13.5 |
| `Buffs`/`Debuffs` change | no | status visuals (Part 4 §4.7.6) | — | `wh` sway (Part 4) | — | Decoded with `Moves.statusDecode` |
| Realm marker `Realm_<id>` appears / `Dead` | no | `vfx.realm.dome` + ring; 12f time dilation (`ALG_CC_Fx` saturation −0.4 and VFX `TimeScale` 0.2) | `sfx.realm.open`; Music duck −8 dB while the local player is a member (GDD §14.6) | `heavy` if a member | — | Reduced Motion: no colour-correction pulse |
| `Workspace.Sim` carrier added / `Dead` | own: yes | attach `vfx.<style>.<slot>` (or a `model.*` summon) by `MoveId`; hide at once on `Dead` | `def.sfx` | — | — | Part 3 §3.11.5 |
| Player `Streak ≥ 3` | no | `vfx.streak.aura` (gold, low rate) | — | — | — | Part 3 §3.13.3 |

**Focus target** (GDD §15.3 "current lock/last-hit target"): the view whose `HP` or `CHits` changed within 6 frames after the local attacker's `Cnt` 0→1, picked as the nearest such view in front of the attacker (dot > 0) within 15 studs. It is held for `Perf.Ui.FocusHoldSec` (4 s) after the last contact. HUD, nameplates and VFX LOD use it.

**Verify:** Lune lint `render.no_attr_signal` (on `RenderController.luau`: no `GetAttributeChangedSignal(` text; a Lune check, not a Studio scenario); scenarios `render.undo_spark` (Part 3), `render.hitstop_not_reentry` (local M1 hits a dummy → exactly one swing effect per M1 state despite the `StF` shift), `render.no_double_move_vfx` (`hero.3` hits a dummy → exactly one `vfx.hero.3` and one `sfx.hero.3` play, and no telegraph is drawn after the contact frame), `render.remote_telegraph_full` (under the Part 7 `M0Gate` net profile, ≈ 150 ms RTT: a bot uses `hero.2`; the observer client draws the Arc telegraph for 8 ± 1 render frames before the move's VFX), `render.focus_target` (hit a dummy → `focusTarget()` is that dummy for 4 s), `render.haptic_gate` (a test seam counts `ALG_Haptic:Play()` calls: gamepad emulation + one M1 contact → 1; with `reducedMotion = true` → 0; KBM-only → 0).

#### 6.6.2 AnimController

Part 3 plays every combat track from `Adapter.animate` (`BindToAnimation`, both peers for simulated entries; server-played tracks replicate to observers). AnimController handles everything else:
1. **Studio clip registration check.** `Main.client` already calls `Assets.registerStudioClips(ReplicatedStorage.Anim)` (Part 2 §2.9.1). AnimController's `start()` logs `[ALG][Anim] clips n=<count> studio=<bool>` and warns when the count is 0 in Studio.
2. **Cosmetic tracks** (render-only, never gameplay): `anim.common.getup` after a ragdoll, Style Wheel / Settings preview rigs, Wish reveal poses, the Training mirror.
3. **The track budget.** Playing tracks per Animator ≤ `Perf.Anim.MaxPlayingTracks` (8, pending Part 3 spike SA-05; GDD §18.2's figure is unverified [api-sheet §1 #17]). A cosmetic play that would exceed it is skipped, with one `[ALG][Anim] WARN track_budget` per model.

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

**Templates** come from `ReplicatedStorage.VFX` (built by `BuildVfxTemplates`, §6.6.7). They are cloned into per-key pools under `Workspace.ALG_Local.Vfx`.

```lua
--!strict
export type Layer = "fx" | "telegraph"
export type Tint = { core: Color3, mid: Color3, ink: Color3 }
export type VfxSpec = {
	key: string,                          -- Assets key, e.g. "vfx.hit.light" → "tpl:VFX/VFX_HitSpark_Light"
	at: CFrame?, attach: BasePart?,       -- world placement (stays anchored), or weld to a part (auras): un-anchored first (below)
	owner: Model?,                        -- whose effect (LOD, tint, Flair, Reduced VFX opacity)
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

**Attached effects (`spec.attach`).** Every VFX template part is built anchored (§6.6.7), and welding an anchored part into a character makes it the assembly root and implicitly anchors the whole character [Docs `physics/assemblies.md`: "When one of an assembly's parts is anchored … all of the other parts become implicitly anchored with it"]. The predicted local character would then stop moving client-side and rubber-band. So when `spec.attach` is set, VfxController takes the pooled clone and, **before** parenting or welding it, sets on every `BasePart` descendant: `Anchored = false`, `Massless = true`, `CanCollide = false`, `CanQuery = false`, `CanTouch = false`, `CollisionGroup = "CosmeticNoQuery"`. It then positions the root at `spec.attach.CFrame` and welds it with one `WeldConstraint` (`Part0 = spec.attach`, `Part1 = <template root>`) parented under the clone. World-placed effects (`spec.at`) stay anchored. A pooled clone returned to the pool keeps its un-anchored flags and is re-anchored only if it is reused with `at` (pools are keyed per key and placement mode). This rule covers `vfx.awaken.aura`, `vfx.streak.aura`, `vfx.spawn.shimmer`, status visuals and the Style Tell ring (§6.8.4).

**Perf sample sender** (GDD §20.4 `perf_sample`; Part 5 §5.8.3; Part 2 §2.7.4 row 18). While `Players.LocalPlayer:GetAttribute("PerfSampled") == true` (written by Telemetry at join; Part 2 §2.5.2), VfxController records `RenderStepped` dt into a ring covering `Runtime.get().Telemetry.PerfWindowSec` (120 s). Once per `Telemetry.PerfSendEverySec` (300 s), the first send only after the first full window, it computes fps P50 and P5 by nearest rank over `1/dt`, reads `Stats:GetTotalMemoryUsageMb()` [Docs `Stats.yaml`], and fires `Net.client.fire("Telemetry.Perf", {deviceTier = <current tier>, fpsP50 = …, fpsP5 = …, memoryMb = …, vfxDegradeLevel = <degrade level 0–5>})` (`TelPerfSample`). The work runs in the existing render step (no extra binding, no yields). When the attribute is false or absent, nothing is recorded or sent.

**Palette and Flair.** The tint is `core = palette.accent`, `mid = palette.primary`, `ink = #1A1A1A` (Part 4 §4.7.1; GDD §13.4 three tones plus an ink rim). Emitters get `ColorSequence(core → mid)`; ink parts keep ink. The Flair overlay is applied to the owner's effects only and never changes shape (GDD §8.7, §13.4):

| Flair | Overlay |
|---|---|
| `standard` | none |
| `shiny` | extra sparkle emitter (default sparkles texture, Rate 4) + brighter outline |
| `golden` | tint lerp 50 % toward `#FFD54F`; ink → `#8D6E00` |
| `crystal` | tint lerp 40 % toward `#B3E5FC`; +0.15 transparency |
| `rainbow` | tint hue cycles at 0.5 Hz (`Color3.fromHSV`) |
| `nightosphere` | core `#B71C1C`, mid `#212121` |
| `glitched` | core alternates `#00E5FF`/`#FF00E5` at 6 Hz; with `glitchedCalm` (GDD §16.3) static `#00E5FF`. Never faster than 3 full-screen flashes per second (it is not full-screen anyway) |

**Budgets** (`Perf.Tiers[tier]`; GDD §13.5). Every template root has attributes `Emitters`, `ParticlesEst` (max simultaneous particles: Rate × Lifetime.Max + Emit bursts), `BeamsTrails`, `Layer`, `Lod1` (variant name or `""`). The controller keeps running totals of active `fx`-layer templates:
1. When `play` would exceed `MaxEmitters`, `MaxParticles` or `MaxBeamsTrails` for the tier (× `ReducedVfxBudgetMult` 0.4 when Reduced VFX is on, GDD §16.3): effects of the local player and the focus target still play; others play their `Lod1` variant if it fits, else are skipped.
2. **LOD by distance** (GDD §13.5): the local player's and focus target's effects are always full quality. Others beyond `LodSimplifiedStuds` (80) use `Lod1`; beyond `LodTelegraphOnlyStuds` (160) only telegraphs render.
3. **Degrade scheduler:** a frame-time EMA (α = 0.1) compared with `1 / TargetFps`. Above `DegradeFrameTimeHigh` (×1.15) for `DegradeUpAfterSec` (2 s) → level + 1 (max 5); below ×0.85 for 5 s → level − 1. Level `n` multiplies every emitter `Rate` and `Emit` count by `DegradeRateMult[n + 1]` = {1, 0.8, 0.6, 0.45, 0.3, 0.2}. It lowers emitter rates first and **never** touches telegraphs (GDD §13.5). The level is reported in `Telemetry.Perf` (`vfxDegradeLevel`, Part 2 §2.7.4 row 18).
4. Reduced VFX also sets enemy-owned effects to 60 % opacity (`ReducedVfxEnemyOpacity`, GDD §16.3): transparency `t → 1 − 0.6·(1 − t)`.
5. **Per-move emitter caps** by rarity (GDD §13.5: Starter–Common 3, Uncommon–Rare 5, Epic 6, Legendary 7, Mythic–Cosmic 8) are enforced at build time by `BuildVfxTemplates.verify()` and asserted in Studio at `play`.

**Telegraph layer** (GDD §13.4, §16.3). `telegraph()` builds from `VFX_Telegraph_<Shape>`:
- a flat Neon fill (thickness 0.05, `Transparency = TelegraphFillTransparency` 0.35) in the user's `telegraphColor`;
- an outline copy `TelegraphOutlineStuds` (0.4) larger underneath. Its colour is chosen at spawn: raycast down from the telegraph centre (`WorldQuery`), take the ground colour (`RaycastResult.Instance.Color`, or `Terrain:GetMaterialColor(result.Material)` for Terrain), then pick ink `#1A1A1A` if `Theme.contrast(ink, ground) ≥ 4.5`, else `#FFFFFF`;
- an **always-on-top overlay**: a `SurfaceGui` (`Face = Top`, `AlwaysOnTop = true`, `LightInfluence = 0`) on the fill part, holding the shape's outline drawn with Frames (`UIStroke` 3 px, outline colour). Circle: one frame with `UICorner` 0.5. Line: a rectangle. Cone and Arc: the two edges plus the far arc as 8 rotated thin frames. This keeps the hit area's edge visible above particles and beams (spike S-P6-4).

Telegraphs are never pooled out, never LOD'd, never reduced, never tinted by Flair. Telegraph colour swatches and colour-blind variants are in §6.9.4.

**Highlight pool** (GDD §12.4: gameplay feedback only, pooled, cap 12). The docs allow 255 per client and count disabled ones [Docs `Highlight.yaml`]; 12 is the GDD's design cap. The pool creates 12 Highlights at `start()` under `ALG_Local.Vfx` and toggles `Adornee`/`Enabled` (no add/remove hitches). Priority when full: `impact` > `pb` > `crack` > `hit` > `lock` (evict the lowest-priority, oldest one).

| Kind | Fill / outline | Frames |
|---|---|---|
| `hit` | Fill `#FFFFFF` at 0.5, outline off | 3 |
| `pb` | Fill off, outline `#FFD54F` | 6 |
| `crack` | Fill off, outline lerps `#FFFFFF` → `#E53935` as `Grd` falls from 40 to 0 | while `Grd < 40` and blocking |
| `lock` | Fill off, outline `Theme.Color.Focus` at 0.5 | while focus target |
| `impact` | Fill ink `#1A1A1A` at 0, outline `#FFFFFF`, `DepthMode = Occluded` | 2 (§6.6.6) |
| `pulse` | Fill off, outline `#FFFFFF`, outline transparency 0 → 1 | 15 (Reduced Motion replacement) |

**Verify:** scenario `vfx.budget_tiers` (spawn 30 hit sparks per frame from bots for 3 s at tier Low → `stats().emitters ≤ 120` and `particles ≤ 1500`; with Reduced VFX ≤ 48 / 600); `vfx.telegraph_never_reduced` (Reduced VFX + degrade level 5 → a move telegraph still renders with fill transparency 0.35); `vfx.highlight_cap` (20 simultaneous hits → never more than 12 Highlights with `Enabled = true`); `vfx.lod_distance` (an enemy effect at 120 studs uses `Lod1`; at 170 studs only its telegraph exists); `vfx.attach_unanchored` (attach `vfx.awaken.aura` and the Style Tell ring to the local character → every attached `BasePart` has `Anchored == false` and `Massless == true`, `HRP.AssemblyRootPart == HRP`, `HRP.Anchored == false`, and a 2 s scripted walk moves the root ≥ 10 studs with `RunService:GetPredictionStatus(root)` still `Predicted`); `tel.perf_sample_sent` (Part 5 §5.8.3: seam `Telemetry._setPerfSampled(P, true)` with `setConfig Telemetry {"PerfWindowSec":5,"PerfSendEverySec":10}` → a `perf_sample` record within 15 s); Lune `theme.contrast` (ink vs `#F8BBD0` ≥ 4.5; ink vs `#3A2F3F` < 4.5 → white chosen).

#### 6.6.4 SfxController

- **Buses:** `SoundService.Music`, `Sfx`, `Vo`, `Ui` (SoundGroups, client-created at `init()`; Part 2 §2.2). `Volume = settings.vol<Bus> × settings.volMaster`.
- **Pool:** `Perf.Audio.PoolSize` (32) `Sound` instances. Spatial sounds are parented to pooled `Attachment`s on one anchored, invisible part `ALG_Local.Sfx.Anchor`; 2D sounds are parented to `SoundService`. `Sound.SoundId` takes the manifest ID (`rbxasset://sounds/<file>` placeholders, §0.5). `RollOffMaxDistance = 80`, `RollOffMinDistance = 8` (td).
- **Voice cap** `Perf.Tiers[tier].MaxSounds` (16 / 24 / 32). When full, the lowest-priority, oldest sound is stopped. Priorities follow GDD §14.6: 1 telegraph cue, 2 Perfect Block / Guard Break, 3 hits on or from you, 4 your VO, 5 nearby hits, 6 music, 7 ambience.
- **Speed:** `Styles.SFX_SPEED[key]` for per-move keys (Part 4 §4.7.3), else `Perf.Audio.Map[key].speed`, times a uniform jitter of ±`PitchJitter` (5 %, td) from a module `Random.new()` (presentation randomness is allowed; it never reaches the sim), times the combo rise `1 + ComboPitchStep × min(hits − 1, ComboPitchMaxSteps)` (0.03 × ≤ 10, td; GDD §6.13 "rising pitch on combo counters").
- **Barks** (GDD §14.5): ≤ 1 per `BarkMinIntervalSec` (4 s) per player, only within `BarkRangeStuds` (40) of the listener, own barks only when `quietBarks` is on (§6.13 item 10), always subtitled when `subtitles` is on (§6.9.10). Placeholder voice is `AudioTextToSpeech` wired to an `AudioDeviceOutput` with a `Wire` (`SourceInstance`/`TargetInstance`) [Docs `audio/objects.md`], using `Skin.text("ui.vo.<line>")`. TTS is throttled by the engine (1 + 6 × CCU per minute per experience; 4 per second per client [Docs `AudioTextToSpeech.yaml`]); a refused TTS falls back to the subtitle only. TTS runs only in Studio (`Perf.Audio.TtsInStudioOnly = true`). Part 8's release lint blocks it anyway.
- **Ducking** (GDD §14.6): Music bus −4 dB (×0.631) while the local player's combo `CHits ≥ 3` on the focus target; −8 dB (×0.398) while a member of an active Realm; Cosmic Wish reveal: Music muted (GDD §14.4). Tweened over 0.2 s.

API: `play(key, {at, owner, speed, volume, prio, comboStep}?) -> Sound?`, `bark(owner: Player | Model, lineKey: string) -> boolean`, `duck(reason: "combo" | "realm" | "cosmic", on: boolean)`, `applySettings(s: Types.Settings)`.

**Verify:** scenario `sfx.voice_cap` (40 hit sounds in one frame at tier Low → ≤ 16 playing; the PB ding survives); `sfx.bark_throttle` (two barks 1 s apart from one player → the second returns false); `sfx.bus_volumes` (settings volMusic 0.5 → `SoundService.Music.Volume == 0.5 × volMaster`).

#### 6.6.5 CameraController

**Shift-lock** (GDD §15.2: "Mouse (Shift-Lock default ON)"). The default PlayerModule mouse lock is disabled: `Player.DevEnableMouseLock = false` is set **by the server** at join (it cannot be set client-side [Docs `Player.yaml`]; §6.13 item 6). CameraController implements its own:
- State is owned by InputController (`setShiftLock(on)` / `shiftLock()`, §6.13 item 7), so InputController can synthesize `AimLock` (Part 3 §3.10.3) without requiring a later controller. The default is `Perf.Camera.ShiftLockDefault` (true). The `ShiftLock` UI action (LeftShift) and the touch `ShiftLockBtn` toggle it.
- `BindToRenderStep("ALG_Camera", Enum.RenderPriority.Camera.Value + 1, fn)`. When on and `UserInputService.PreferredInput == Enum.PreferredInput.KeyboardAndMouse`: `UserInputService.MouseBehavior = Enum.MouseBehavior.LockCenter`. On any input: `camera.CFrame = camera.CFrame * CFrame.new(ShiftLockOffset)` ({1.75, 0, 0}, td). Character facing comes from the sim (`AimLock` + `Aim`, Part 3), never from the camera script. Whether the PlayerModule overrides `MouseBehavior` after Camera priority is spike S-P6-8.

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

-- Called from the ALG_Camera render step after the shift-lock offset:
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
| `heavy` | Awakening, Realm open | 1 | 0.08 | 0.04 / 0.25 / 0.45 | {1.4, 1.4, 0.4} | {0.04, 0.04, 0.06} |

**Other camera duties.**
- `kick(frames)`: hitstop FOV punch `−Perf.Camera.HitstopFovKick` (2°) for `frames` frames, then restore. Skipped with Reduced Motion.
- `cinematic("awaken")`: GDD §6.9 local 1.2 s mini-cinematic. `CameraType = Scriptable`; orbit 150° around the root at distance 10 and height 3 (`TweenService` on a NumberValue driving the CFrame), then restore `Custom`. With Reduced Motion: a static three-quarter framing for 0.6 s, no orbit.
- `flash(color, alpha, frames)`: local screen-space flash (ScreenGui frame). Rate-limited to ≤ `MaxFlashesPerSec` (3, GDD §16.3) and never `#FFFFFF` above alpha 0.5. Disabled with Reduced Motion (replaced by `VfxController.flash(model, "pulse")`).
- `applyLighting(mapId, phase?)` (§6.5).

**Verify:** scenario `cam.shift_lock_default` (KBM client: `UserInputService.MouseBehavior == LockCenter` after spawn; LeftShift toggles; the `AimLock` action reads true); `cam.reduced_motion` (`reducedMotion = true` → `shake()` creates no instance; the awaken cinematic does not orbit); `cam.flash_rate` (10 flash requests in 1 s → ≤ 3 applied).

#### 6.6.6 Hitstop presentation and impact frames

- **Hitstop** (GDD §6.13: M1 3f, Ender 6f, signature 8–10f, Perfect Block 6f; Realm 12f render-only). The sim freezes the state, and `Adapter.animate` calls `AdjustSpeed(0)` (Part 3 §3.5.6). Render adds: while `fNow ≤ HsF` for a view, `VfxController.freeze(model, true)` (effects attached to that model get `ParticleEmitter.TimeScale = 0.05`); `CameraController.kick(HsF − fNow)` for the local attacker or victim.
- **Impact frames** (GDD §6.13, §13.4): on contact by a move with `signature = true`, for 2 frames, "local to the impact, never full-screen", off with Reduced Motion. Implementation: Highlight `impact` on the victim (fill ink, outline white) plus `VFX_ImpactStar` with the ink starburst variant behind the victim (BillboardGui, 8 studs). Shown to the local attacker (predicted), the local victim, and observers ≤ 40 studs. With Reduced Motion: Highlight `pulse` (outline only, 15f).

**Verify:** scenario `vfx.impact_frame` (`hero.4` signature hit on a dummy → a Highlight with `FillTransparency = 0` exists for 2 ± 1 render frames; with Reduced Motion, none with fill; a pulse instead).

#### 6.6.7 `BuildVfxTemplates` (`ReplicatedStorage.VFX`)

Every template is a Model named after its recipe. Rules:
- An invisible root part `Anchor` (0.2 stud cube, `Transparency = 1`, `Anchored = true`, all collision/query/touch off, `CastShadow = false`, `CollisionGroup = "CosmeticNoQuery"`) with Attachments `A0`/`A1`, plus visual parts with the same flags. Templates stay anchored for world-placed (`at`) effects; VfxController un-anchors and sets `Massless = true` on every part of a clone **before** welding it to a character (`attach`, §6.6.3). Templates never contain welds or constraints to a character.
- `LightInfluence = 0` on every ParticleEmitter, Beam and Trail. `LightEmission` only for magic (Beam_Magic, Awaken_Aura). Default particle texture only (agent-assets §2.6).
- Root attributes `Recipe`, `Layer`, `Emitters`, `ParticlesEst`, `BeamsTrails`, `Lod1`, `Tintable`.
- CSG at **Edit time only** (`GeometryService:SubtractAsync`, `UnionAsync`, `IntersectAsync`, `FragmentAsync`; security None, yields [Docs `GeometryService.yaml`]). Results are cached in the template and never computed at runtime.

| Recipe | Source | Spec (beyond agent-assets §7.4 / Part 4 §4.7.2) |
|---|---|---|
| `VFX_HitSpark_Light`, `VFX_Impact_Heavy`, `VFX_ImpactStar`, `VFX_Slash_Smear`, `VFX_Beam_Magic`, `VFX_Shockwave_Ring`, `VFX_Telegraph_Circle`, `_Line`, `_Cone`, `VFX_Guard_Bubble`, `VFX_Guard_Break`, `VFX_Dash_Puff`, `VFX_Awaken_Aura`, `VFX_KO_Burst`, `VFX_Realm_Dome` | agent-assets §7.4 | As listed there. `VFX_Guard_Break` = the bubble Ball fragmented at build time (`GenerateFragmentSites` + `FragmentAsync`, cached). The telegraph templates add the `SurfaceGui` overlay of §6.6.3. `VFX_KO_Burst` uses `Explosion` with `BlastPressure = 0`, `DestroyJointRadiusPercent = 0` created at play time (an Explosion cannot sit in a template) plus 3 × ImpactStar |
| `VFX_Telegraph_Arc`, `VFX_Projectile_Orb`, `VFX_Zone_Disc`, `VFX_Wave_Wall`, `VFX_Encase_Block`, `VFX_Stealth_Shimmer`, `VFX_Status_Pips` | Part 4 §4.7.2 | As listed there |
| `VFX_BlockSpark` | new | ParticleEmitter, default texture, `Emit(6)`, Lifetime 0.1–0.15, Speed 12–18, white→`#B3E5FC` |
| `VFX_PerfectStar` | new | BillboardGui (`AlwaysOnTop = false`, 6 × 6 studs) with a `★` TextLabel in LuckiestGuy, gold `#FFD54F`, ink UIStroke; plus a ring (Shockwave_Ring at 0.5 scale, gold) |
| `VFX_Breakaway_Burst` | new | Shockwave_Ring (white, Size ×4 over 0.2 s) + sparkles `Emit(24)`, Lifetime 0.3 |
| `VFX_Splat` | new | Flat Neon disc Ø4 (0.05 thick) oriented to the wall normal, palette core, fading over 0.6 s, + ImpactStar |
| `VFX_Tell_Ring` | new | CSG ring (Cylinder Ø4.5 minus Cylinder Ø3.7, 0.1 thick), SmoothPlastic, transparency 0.35; the Style Tell ground ring (§6.8.4) |
| `VFX_Telegraph_*_Lod1`, `VFX_HitSpark_Light_Lod1`, … | derived | Every fx recipe with emitters gets a `_Lod1` copy: emitter Rate and Emit halved, Trails disabled |

Asset keys (Part 2 §2.9.1 format) for generic effects, all `{id = "tpl:VFX/<Recipe>", status = "placeholder", source = "agent", kind = "vfx"}`:

| Key | Recipe | Key | Recipe |
|---|---|---|---|
| `vfx.hit.light` | VFX_HitSpark_Light | `vfx.awaken.aura` | VFX_Awaken_Aura |
| `vfx.hit.heavy` | VFX_Impact_Heavy | `vfx.awaken.burst` | VFX_Shockwave_Ring |
| `vfx.hit.star` | VFX_ImpactStar | `vfx.ko.poof` | VFX_KO_Burst |
| `vfx.hit.block` | VFX_BlockSpark | `vfx.wallsplat.splat` | VFX_Splat |
| `vfx.pb.star` | VFX_PerfectStar | `vfx.realm.dome` | VFX_Realm_Dome |
| `vfx.guard.bubble` | VFX_Guard_Bubble | `vfx.realm.ring` | VFX_Shockwave_Ring |
| `vfx.guard.break` | VFX_Guard_Break | `vfx.spawn.shimmer` | VFX_Stealth_Shimmer (white) |
| `vfx.dash.puff` | VFX_Dash_Puff | `vfx.tell.ring` | VFX_Tell_Ring |
| `vfx.breakaway.burst` | VFX_Breakaway_Burst | `vfx.streak.aura` | VFX_Awaken_Aura (gold, Rate × 0.4) |
| `vfx.telegraph.circle` / `.line` / `.cone` / `.arc` | VFX_Telegraph_* | `vfx.spring.boing` | VFX_Shockwave_Ring (mint) |

**Verify:** `BuildVfxTemplates.verify()`: every recipe above plus every recipe referenced by `vfx.<style>.<slot>` manifest entries exists; `Emitters` on each template ≤ the per-move cap for the lowest rarity using it; no part has `CanCollide`/`CanQuery`/`CanTouch` true; every emitter has `LightInfluence == 0`. Scenario `vfx.templates_all_play` (play each key once at a fixed point; `[ALG][Vfx]` logs no missing template).

#### 6.6.8 `BuildPlaceholderAnims` (`ReplicatedStorage.Anim`)

Implements Part 4 §4.6 exactly (22 sequences per implemented style; naming, attributes `AssetKey`, `MoveId`, `Frames`, `Gen`; markers; `PoseBase.EasingStyle` `CubicV2`/`Constant`/`Linear`; `AnimationClip.Loop`/`Priority`). Pose hierarchy: `HumanoidRootPart › LowerTorso › UpperTorso › Head`, `UpperTorso › <Side>UpperArm › <Side>LowerArm › <Side>Hand`, `LowerTorso › <Side>UpperLeg › <Side>LowerLeg › <Side>Foot`, built with `Keyframe:AddPose` / `Pose:AddSubPose`. Joint names are read from `ServerStorage.Templates.DummyRig` (§6.8.6), hence the builder order (Part 1 §1.6.6). This part adds the common clips of Part 3 §3.11.8 and the render-only get-up.

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
| 3. Upload | Host: Lune `tools/upload_anims.luau [--only <key>]` | Reads `ALG_OPEN_CLOUD_API_KEY` from `process.env` and **never prints, logs or writes it** (missing → `[ALG][CHECK] upload_anims FAIL missing ALG_OPEN_CLOUD_API_KEY`, exit 1). Reads `groupId` from `docs/agent/dev-place.json`. For each `build/anims/*.b64` without a line in `build/anims/ids.txt`: decode base64 (pure Luau) → `build/anims/<key>.rbxm` (byte length must equal the recorded `size`), then `POST https://apis.roblox.com/assets/v1/assets` with header `x-api-key` and a `multipart/form-data` body: part `request` = `{"assetType":"Animation","displayName":"<key>","description":"Algebraic placeholder","creationContext":{"creator":{"groupId":"<groupId>"}}}`, part `fileContent` = the `.rbxm` bytes with content type `model/x-rbxm` [Docs `cloud/guides/usage-assets.md`: Animation accepts `.rbxm`, `model/x-rbxm`]. The response `{"path":"operations/<id>"}` is polled with `GET https://apis.roblox.com/assets/v1/operations/<id>` every 2 s up to 60 s (td) until `done == true`; then `response.assetId` is appended to `build/anims/ids.txt` as `key=id`. One upload at a time, ≥ 1 s apart (td). Prints `[ALG][CHECK] upload_anims key=<key> id=<id>` or `… FAIL key=<key> reason=<status message>`; finally `[ALG][CHECK] upload_anims SUMMARY ok=<n> fail=<m>` |
| 4. Patch | Host (agent) | For each `key=id` line, the agent edits `src/Shared/Assets.luau`: `id = "rbxassetid://<id>"`, `status = "placeholder"` (content is still placeholder), `source = "agent"`, `kind = "anim"`. Then the host gate and the sync gate |

Rules:
- **Every run of step 3 is human-approved.** The tool refuses to upload unless invoked with `--approve <n>` where `n` equals the number of pending keys it prints in a dry pass without the flag (so the human sees exactly what will be uploaded). Part 1's example permissions file keeps `lune run tools/upload_anims.luau` on **ask** (§6.13 item 23).
- The uploaded clips belong to the **dev group** only (Part 8 §8.2.6: production assets are re-uploaded at release).
- `lint_rules` `secrets` covers `tools/upload_anims.luau`; the env var name is the only key-related literal in it.
- The tool writes `build/anims/<key>.rbxm` and `ids.txt` only through `tools/lib/SafeWrite.luau` (Part 8 §8.4.5; Lune `ip.no_tool_writes_signed_files` scans `tools/**`).
- Re-running is idempotent: keys already in `ids.txt` are skipped. A failed key is retried only on the next approved run.

**Verify:** spike **S-P6-14** (export size vs the `execute_luau` result limit and the 2 KB payload cap: in T143 export the shortest and the longest existing clip (`anim.common.*`; `anim.hero.m1_1` does not exist before T174, which repeats the check on it); pass when one chunk round-trips byte-exactly, i.e. the decoded `.rbxm` length equals `size`; otherwise set the chunk length so every call returns ≤ the measured limit; if neither works, fall back to H-08 and mark T143 `skipped`). Lune `anims.base64_roundtrip` in `tests/core/upload_anims.spec.luau` (pure helper `tools/base64.luau`, loaded with `@tools/`, used by `upload_anims.luau`; encoding then decoding a 3-byte, a 4-byte and a 1 KB seeded-random buffer is lossless); Lune `anims.upload_refuses_without_approve` (a dry pass with a fake key prints the pending count and makes no request). After a real run (T143): `[ALG][CHECK] upload_anims SUMMARY ok=<n> fail=0`, `Assets.isPlaceholder` unchanged, and a two-client playtest plays an uploaded clip on the observer (`AnimationTrack.IsPlaying` true on the remote rig; in T143 an `anim.common.*` clip, from T174 also `anim.hero.m1_1`).

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

- The originals are hidden with `AppearanceController.setHidden(model, true)` (sets `LocalTransparencyModifier = 1` on the visual set and gear). Gear follows the clone's parts while ragdolled.
- The face `SurfaceGui` travels with the cloned head, so expressions keep working (`ko`, `hurt_heavy`).
- Under Server Authority, client-created unanchored parts are simulated locally; this is spike S-P6-9.

**Verify:** scenario `ragdoll.clone_lifecycle` (bot Ender on a dummy within 30 studs: a clone Model appears under `ALG_Local.Ragdolls` 12 ± 2 frames after `Ragdoll` entry; every clone part has `CanQuery == false` and `CollisionGroup == "RagdollFx"`; it is gone ≤ 0.3 s after the dummy's `Ragdoll` ends; the real rig's parts have `LocalTransparencyModifier == 0` again); `ragdoll.cap` (6 dummies ragdolled at once at tier Low → ≤ 4 clones); `ragdoll.wallsplat_handoff` (Training: an Ender toward `SplatWall` from 18 studs, splat at e ≈ 14 → on the first render frame where the dummy's `St == WallSplat`, or the next one, no clone exists under `ALG_Local.Ragdolls` for it and its real rig parts have `LocalTransparencyModifier == 0`); `ragdoll.juggle_rebegin` (a Ragdoll refresh by an air hit → the old clone is destroyed at once and at most one clone exists per view at any time); `ragdoll.no_misprediction` (the local player is ragdolled 10 times; the Part 7 misprediction log shows no entries attributable to `RagdollFx` parts).

---

### 6.8 Appearance rendering (Character Look modes)

§0.4 fixes the settings (`CharacterLook`, `OwnLook`, `ShowStyleGear`, viewer-local `OthersAppearance`), the always-on Style Tell, and the fixed hurtbox. Part 5 owns data and replication (Player attributes `CharacterLook`, `OwnLook`, `ShowStyleGear`, `OosonaCode`, `EquippedStyle`, `Flair`, Part 2 §2.5.2). This section is the graybox renderer, implemented in `AppearanceController.luau`.

#### 6.8.1 Effective mode per viewer

```lua
--!strict
export type LookMode = "Character" | "Oosona" | "Avatar" | "Simplified"
-- owner attributes: CharacterLook (bool), OwnLook ("Oosona"|"Avatar"); bots have neither → Character.
local function effectiveMode(characterLook: boolean?, ownLook: string?, viewer: string): LookMode
	if viewer == "Simplified" then
		return "Simplified"
	elseif viewer == "AlwaysCharacter" then
		return "Character"
	end
	local own: LookMode = if ownLook == "Avatar" then "Avatar" else "Oosona"
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

The local player's own character uses `AsChosen` regardless of `OthersAppearance`. Missing attributes (M0, before Part 5 lands) mean `Character`.

#### 6.8.2 What each mode renders

| Mode | Body | Props | Face | Source |
|---|---|---|---|---|
| `Character` | Proxy rig (§6.8.3) in the palette `Theme.stylePalette(styleId, variantId)` (§6.9.4; `variantId` = the owner's `Variant` attribute, `""` = base): torso `primary`, limbs `secondary`, head `secondary` | **Always** the style's Style Gear (`StyleDef.kitProps` → `model.<style>.gear_*`, Part 4 §4.7.4): the "signature read" (GDD §12.3). Gear colours also come from `Theme.stylePalette(styleId, variantId)` | UI face (§6.8.5), expressions on | GDD §12.3 "Character Look On" |
| `Oosona` | Proxy rig in the Part 5 §5.7.1 three-colour rule: head and arms `Appearance.OosonaSpecies[spec.species].skin`; torso `spec.color`; legs `Appearance.OutfitColors[spec.outfit]`, or `spec.color` darkened by `Appearance.NoOutfitDarken` (0.65) when `outfit = 0`. Species props (by `OosonaSpecies[spec.species].id`); hat and outfit props | Style Gear if `ShowStyleGear` | UI face, variant `OosonaSpec.face` | GDD §12.3 "Ooo-sona"; Part 5 §5.7.1 |
| `Avatar` | The real Roblox avatar as loaded (no proxies) | Style Gear if `ShowStyleGear`, attached to the real parts | The avatar's own face (UI face hidden) | GDD §12.3 "Roblox Avatar". The toon post-process is a future hook |
| `Simplified` | Proxy rig: torso `primary`, limbs `primary` × `Appearance.SimplifiedLimbShade` (0.75, Part 5 §5.10), head `secondary` (palette from `Theme.stylePalette(styleId, variantId)`) | None | UI face, `neutral` only | §0.4 viewer override; Part 4 §4.7.6 |

All modes keep the **Style Tell** (§6.8.4). The hurtbox is never touched (§0.4, GDD §6.2).

**Variants (graybox render).** A Variant is a palette change only; shapes never change (GDD §8.7). `Theme.stylePalette(styleId, variantId)` returns the base palette for `""` (the active pack's `Skin.look` override, else `StyleDef.palette`; Part 8 §8.2.7); for `v1` it swaps `secondary` and `accent`; for `v2` it darkens every colour by 25 % (`Color3:Lerp(black, 0.25)`, applied to `primary`, `secondary` and `accent`). Proxy body, Style Gear, the Style Tell ring and chip, and the owner's VFX tint (§6.6.3) all read it, so a variant is visible in every mode where the style palette shows. Variant ids beyond `v2` (future content) fall back to `v2`'s rule until Part 4 defines per-variant palettes.

**Codename looks kill switch** (`Features.CodenameLooks`, Part 2 §2.6.2; Part 8 §8.2.6 / §8.6.3; overridable live through `ALG_Features`). AppearanceController reads `Runtime.get().Features.CodenameLooks` and re-applies on `Runtime.changed`. When it is **false**: every `Gear_*` and `Summon_*` clone whose template has `LookSet == "Codename"` (Part 8 §8.5.6) is hidden (`LocalTransparencyModifier = 1`, effects disabled); summons render as a neutral Ball (Ø3, `primary` of the **Original** palette); `Character` mode renders as `Simplified` for every viewer; and `Theme.stylePalette` returns the Original pack's palette whatever pack is active (§6.9.4), so proxies, chips, rings and VFX tints all switch to Original colours. It takes effect within 1 s with no respawn (Part 8 scenario `ip.pack_switch_live` asserts no Codename-look part is visible within 1 s).

#### 6.8.3 Proxy rig and attachment strategy

- `ReplicatedStorage.Looks.LookRig_R15` (built by `BuildTemplates`): 15 primitive Parts named like the R15 body parts. Sizes are taken from `DummyRig` at build time; the head is a Ball, the rest Blocks. All `CosmeticNoQuery`, `Massless`, no collision/query/touch, `CastShadow = true`. Each proxy part carries copies of its source part's `*RigAttachment`s (same names and CFrames), so the ragdoll (§6.7) and gear attach the same way.
- For a non-`Avatar` mode, the controller clones the proxy rig into `ALG_Local.Looks.<rid>`, sizes each proxy to the real part's `Size`, welds it to the matching real body part with a client-local `WeldConstraint` (`Part0 = proxy`, `Part1 = real`), and hides the real body parts and accessories (`LocalTransparencyModifier = 1`). Clothing and avatar heads are hidden with the parts they render on.
- Gear Accessories (Part 4 §4.7.4) are cloned per character. Each `Handle` is welded so that its named attachment matches the same-named attachment on the target part (proxy or real). This is the `Humanoid:AddAccessory` alignment rule done by hand, because the proxy is not in the Humanoid's model. Handles are `Massless`, `CosmeticNoQuery`.
- The welds make the engine handle interpolation of remote characters. Whether a client-local, massless part welded into the **local predicted** character causes mispredictions is spike S-P6-2. The fallback is no welds: a single `workspace:BulkMoveTo(parts, cframes, Enum.BulkMoveMode.FireCFrameChanged)` per frame in the `ALG_Render` step.
- **LOD** (`Perf.Looks`): beyond `ProxyDetailStuds` (100) the face SurfaceGui is disabled and gear hidden. Beyond `TellOnlyStuds` (160) proxies are hidden too (ring and chip remain).
- **Refresh triggers:** the owner's `EquippedStyle`, `CharacterLook`, `OwnLook`, `ShowStyleGear`, `OosonaCode`, `Flair`, `Variant` attribute changes; the viewer's `othersAppearance` setting; `CharacterAdded`; `Runtime.changed` (`Features.CodenameLooks`, §6.8.2); `Skin.onChanged` (pack switch: palettes and nameplate text, §6.8.4). These are not combat one-shots, so attribute-changed signals are fine here.
- Part 3's `Hurtbox.prepare` is asked to set `Humanoid.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.None` server-side (§6.13 item 11), so the default name/health display never competes with the nameplate.

#### 6.8.4 Style Tell (always on)

Rendered for every character in every mode and for every viewer (§0.4; GDD §12.3 readability guarantee):
1. **Nameplate chip.** BillboardGui `ALG_Nameplate` on the head (proxy or real): `AlwaysOnTop = false`, `MaxDistance = Perf.Looks.TellChipStuds` (150, matching the GDD §11.1 sightline cap), `LightInfluence = 0`, `StudsOffset = (0, 2.6, 0)`, `Size = UDim2.fromOffset(220, 72)`. Rows:
   - **Chip row (always):** a circle in the style `primary` with the rarity shape glyph (§6.9.4) and `Skin.style(id).name` in Body font 14; a Flair glint (✦ in the Flair colour) when `Flair ≠ "standard"`.
   - **Name row (≤ 60 studs):** `Player.DisplayName`, title (`Skin.text("title." .. Title)` **only when `Title ~= ""`**; otherwise the title element is omitted, so no `title.` key is ever resolved), Rookie tag (`ui.nameplate.rookie` when `Rookie`), prestige stars, VIP stroke (gold `UIStroke` when `Vip`).
   - **Info row (GDD §15.3: focus target, or ≤ `NameplateInfoStuds` 25 studs):** thin HP bar, 2 Breakaway pips (`BkP`, refill sweep from `BkProg / 1500`), block-crack icon when `Grd < 40` while blocking, broken icon in `GuardBroken`.
2. **Ground ring.** `vfx.tell.ring` in `primary`, placed at foot level (`−(Humanoid.HipHeight + root.Size.Y / 2) + 0.05` below the HRP) through `VfxController.play{attach = HRP}`, so the §6.6.3 attach rule applies: every ring part is set `Anchored = false`, `Massless = true`, `CanCollide/CanQuery/CanTouch = false`, `CollisionGroup = "CosmeticNoQuery"` **before** the `WeldConstraint` to the HRP is created (an anchored ring would anchor the whole character). If spike S-P6-2 shows welds into the local predicted character mispredict, the ring uses the same fallback as the proxies: no weld, moved once per frame with `workspace:BulkMoveTo` in the `ALG_Render` step. Hidden beyond `TellOnlyStuds` 160.
3. **VFX palette.** All of the owner's effects are tinted from `Theme.stylePalette(styleId, variantId)` (§6.6.3, §6.8.2).
4. **Text refresh.** Chip and name rows re-render on `Skin.onChanged` (live pack switch; Part 2 §2.9.2), on the owner's `Title`, `Rookie`, `Prestige`, `Vip` attribute changes, and on the refresh triggers of §6.8.3.

**Stealth** (Part 4 §4.7.6 / Part 4 Q9): for enemy viewers beyond 12 studs of an owner with status `iv`, the nameplate **name and info rows** and the ring are hidden; the chip is hidden too and the Tell is carried by `VFX_Stealth_Shimmer` in the owner's style colour. This follows Part 4's proposal and is an exception to "always on" (§6.14 item 3).

#### 6.8.5 UI-drawn faces and FaceController

- `ReplicatedStorage.Looks.Face_UI`: a `SurfaceGui` (`Face = Front`, `SizingMode = PixelsPerStud`, `PixelsPerStud = 100`, `LightInfluence = 0`, `AlwaysOnTop = false`) holding `Eyes.L`, `Eyes.R` (Frames with `UICorner` 0.5, ink) and `Mouth` (a Frame inside a clipping Frame with `ClipsDescendants = true`, so a stroked rounded frame shows only its upper or lower half). It is parented to the proxy head (or nothing in Avatar mode).
- Final art later drops into the same SurfaceGui as an `ImageLabel` atlas (`ImageRectOffset`/`ImageRectSize`, agent-assets §7.2): same container, same API.

Expression table (24 keys, GDD §12.3 "24 expressions"). Eye shapes: `dot` (round), `wide` (larger dot), `line` (flat bar), `happy` (upper half-ring), `sad` (lower half-ring), `x` (two crossed bars), `spiral` (ring with stroke), `star` (★ TextLabel). Mouth shapes: `line`, `smile` (lower half-ring), `frown` (upper half-ring), `open` (filled ellipse), `o` (small ring), `grit` (bar with 3 tick marks), `wave` (3 rotated bars), `flat_wide`.

| Expr | Eyes (shape, scaleY, offsetY px) | Mouth (shape, w, h) | Expr | Eyes | Mouth |
|---|---|---|---|---|---|
| `neutral` | dot, 1, 0 | line, 16, 3 | `hurt_light` | line, 1, 0 | frown, 14, 6 |
| `smile` | dot, 1, 0 | smile, 18, 8 | `hurt_heavy` | x, 1, 0 | open, 14, 12 |
| `grin` | happy, 1, 0 | smile, 24, 10 | `dizzy` | spiral, 1, 0 | wave, 18, 6 |
| `grit` | line, 0.6, 2 | grit, 20, 6 | `ko` | x, 1, 0 | o, 8, 8 |
| `shout` | wide, 1.2, −2 | open, 18, 16 | `block` | line, 0.7, 2 | flat_wide, 20, 3 |
| `focus` | dot, 0.8, 1 | line, 12, 3 | `perfect_block` | star, 1, 0 | smile, 20, 8 |
| `smug` | happy, 0.6, 2 | smile, 14, 5 | `awaken` | star, 1.2, −2 | open, 20, 14 |
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

The Oosona face variants 1–6 change the `neutral`/`blank` eye shapes only: 1 dot, 2 wide, 3 oval (dot with scaleY 1.4), 4 happy, 5 line (sleepy), 6 dot with a white highlight. RenderController drives expressions (§6.6.1 table). Expression changes for characters beyond `ProxyDetailStuds` are skipped.

#### 6.8.6 `BuildTemplates`

| Output | Contents |
|---|---|
| `ServerStorage.Templates.DummyRig`, `BotRig` | `Players:CreateHumanoidModelFromDescriptionAsync(desc, Enum.HumanoidRigType.R15)` with a default `HumanoidDescription` (body colours `#9E9E9E` dummy / `#BDBDBD` bot). Edit mode via `execute_luau` is spike S-P6-1 (the docs show it in a Studio script); fallback: the human inserts one R15 rig via Avatar ▸ Rig Builder into `ServerStorage.Templates.__RigSeed`, and the builder clones it. Then `PrimaryPart = HumanoidRootPart`, Humanoid `DisplayDistanceType = None`, every part `Anchored = false` |
| `ServerStorage.Templates.Fragments.*` | Optional fracture cache (§6.1.8) |
| `ReplicatedStorage.Looks.LookRig_R15` | §6.8.3 |
| `ReplicatedStorage.Looks.Face_UI` | §6.8.5 |
| `ReplicatedStorage.Looks.Gear_<style>_<name>`, `Summon_*` | Part 4 §4.7.4 / §4.7.5 recipes (Edit-time `GeometryService:UnionAsync`/`SubtractAsync`). One look set per build (`opts.lookSet`, attribute `LookSet`, Part 8 §8.5.6); both look sets stay in the builder as plain data (per key, a list of primitives `{ shape, size, offset, role }`) so the Lune test `ip.gear_not_codename_clone` compares them without Studio |
| `ReplicatedStorage.Looks.Oosona_Species_<id>` (10), `Oosona_Hat_<n>` (6), `Oosona_Outfit_<n>` (4) | Primitive Accessories (table below) |

Oosona catalog (GDD §12.3 species templates; graybox primitives; keys `model.oosona.species_<id>`, `model.oosona.hat_<n>`, `model.oosona.outfit_<n>`):

| Species (index into `Appearance.OosonaSpecies`, Part 5 §5.10; the builder and UI read `OosonaSpecies[i].id`; skin colour from `.skin`) | Props | Hats (`Appearance.OosonaHats = 6`) | Outfits (`OosonaOutfits = 4`) |
|---|---|---|---|
| 1 `human` | none | 1 cap (Cylinder + Wedge brim) | 1 vest (Block front/back) |
| 2 `candy` | gumdrop cap (Ball 1.4 × 0.9 × 1.4) | 2 beanie (Ball half) | 2 scarf (Cylinder ring) |
| 3 `dog` | 2 floppy ears (Ball 0.4 × 0.9 × 0.25) + tail | 3 neutral crown (Cylinder + 3 Wedges) | 3 short cape (Block) |
| 4 `cat` | 2 ear Wedges + tail | 4 top hat (Cylinder 1.0 × 1.2) | 4 belt (Block band) |
| 5 `vampire` | high collar (Block) | 5 flower (Ball + 4 Balls) | |
| 6 `flame` | 3 Neon orange Wedges (crest) | 6 horn band (Block + 2 Wedges) | |
| 7 `lumpy` | 3 Balls on the torso | | |
| 8 `wizard` | beard (white Wedge) | | |
| 9 `ghost` | proxy transparency 0.35 locally; leg proxies hidden | | |
| 10 `penguin` | white belly plate + orange beak Wedge (head/arms take the species `skin` `#2E3440`; torso and legs follow the §5.7.1 rule like every species) | | |

**Verify:** `BuildTemplates.verify()` (both rigs have 15 body parts and a Humanoid; `LookRig_R15` has 15 parts with RigAttachments; every Part 4 gear key and every Oosona key resolves through `Assets.template`); scenarios `look.modes_matrix` (level **`multi`**: runs only through `multi(2, Baseline30, look.modes_matrix)`, never in solo `studio(…)`; owner sets each of Character/Oosona/Avatar × gear on/off; viewer sets each `OthersAppearance` → the expected mode per §6.8.1; the hurtbox `Size` is unchanged in all 16 cases), `look.style_tell_always` (in every mode, the viewer sees the chip and ring for an enemy at 100 studs), `look.oosona_three_colour` (spec `{species = 2, color = "#F48FB1", outfit = 0}` → head/arm proxies `#F7A8C8`, torso `#F48FB1`, legs `#F48FB1` × 0.65 within 1/255; outfit 3 → legs `#8B3A3A`), `look.variant_render` (owner on `hero` in Character mode; the server-side harness step sets the owner's Player attribute `Variant = "v1"` (render-only check; ownership is covered by `ui.wheel_variant_equip`) → the torso proxy stays `primary` and limb proxies show the base `accent`; `v2` → the torso is `primary` darkened 25 %; switching back to base restores the colours within 1 s), Part 4 `look.gear_silhouette` and `look.stealth_ranges` (level `multi`: `multi(2, Baseline30, look.stealth_ranges)` only), `look.face_expressions` (each of the 24 `set` calls is asserted **programmatically**: eye/mouth shape, size and offset of the `Face_UI` frames match the table; then **one** `screen_capture` of a 3×3 grid of sample expressions goes to human review, within the 3-captures-per-task cap).

---

### 6.9 UI system

All UI is built at runtime into `PlayerGui` (StarterGui stays empty, Part 2 §2.2). Every player-visible string comes from `Skin.text`/`Skin.style`/`Skin.move` (§0.4). No image assets: shapes are Frames, glyphs come from built-in fonts.

#### 6.9.1 Roots, router and screen contract

| Root (`ScreenGui`) | Creator | `DisplayOrder` | `ScreenInsets` | Contents |
|---|---|---|---|---|
| `ALG_Hud` | HudController | 10 | `CoreUISafeInsets` | HUD, nameplate-independent overlays, Training panels |
| `ALG_Touch` | InputController (Part 3) | 11 | `DeviceSafeInsets` | Touch cluster (§6.9.5) |
| `ALG_Screens` | UiController | 20 | `CoreUISafeInsets` | Modal screens |
| `ALG_Toasts` | UiController | 30 | `CoreUISafeInsets` | Toasts, banners, subtitles |

All roots use `ResetOnSpawn = false`, `ZIndexBehavior = Sibling`, `IgnoreGuiInset = false`.

Router (Part 6 section of `UiController`): `open(screen: Types.ScreenId, args: { [string]: any }?)` (the optional `args` is added to Part 2's signature; backward compatible), `close(screen)`, `top() -> ScreenId?`. Modal screens (StyleWheel, Wish, Shop, Settings) call `InputController.setCombatEnabled(false)` while open, except the Style Wheel in `death` mode (the character is KO'd anyway).

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

**Live pack switch** (Part 2 §2.9.2 `Skin.activePack()` / `Skin.onChanged`). UiController registers one `Skin.onChanged` callback at `start()`. On a pack change it unmounts and re-mounts the open screens with the same `args` (screens read every string at mount time), re-renders toasts' pending text, and HudController rebuilds its labels; nameplates refresh themselves (§6.8.4). No screen caches `Skin.text` results across mounts. **Verify:** Part 8 scenario `ip.pack_switch_live` (open Style Wheel and HUD, switch the pack → visible labels change within 1 s and `Skin.missing()` stays empty).

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
	panel: (props: Props?, children: { Instance }?) -> Frame,          -- paper fill, 3 px ink UIStroke, UICorner 14, UIPadding 12
	label: (text: string, style: TextStyle, size: TextSize, props: Props?) -> TextLabel,
	button: (spec: { text: string?, style: TextStyle?, size: Vector2?, touch: boolean?, sfx: string?,
		onActivated: () -> () }) -> TextButton,                       -- touch=true → UISizeConstraint MinSize 88×88
	toggle: (value: boolean, onChange: (boolean) -> ()) -> Frame,
	segmented: (options: { string }, index: number, onChange: (number) -> ()) -> Frame,
	slider: (min: number, max: number, step: number, value: number, onChange: (number) -> ()) -> Frame,
	bar: (spec: { width: number, height: number, fill: Color3 }) -> BarHandle,   -- HP bar with white drain
	radial: (diameter: number) -> RadialHandle,                    -- radial cooldown wipe
	rarity: (rarity: string, size: number) -> Frame,               -- shape glyph + colour + border (§6.9.4)
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
- **Simple HUD** (GDD §16.3 "Cognitive"): hides kill feed, combo counter, zone label, the Quests panel, nameplate name rows and edge indicators; keeps HP, pips, meter, hotbar/cluster, Wallet & level and toasts.

#### 6.9.4 `UI/Theme.luau` (tokens and pure helpers)

Theme is **load-pure** (hex strings and numbers; `Color3`/`Font` values are built lazily inside functions), so Lune can test its helpers.

| Token | Value | Source |
|---|---|---|
| Fonts | Header `LuckiestGuy`; Impact `Bangers`; Body `FredokaOne`; Number `BuilderSansBold`; Hand `PatrickHand`; Dyslexia `Font.fromName("ComicNeueAngular")`, fallback `Enum.Font.Arimo` | GDD §12.7 (rounded hand-lettered headers + legible sans for body and numbers); agent-assets §2.5. All enum members verified in `Font.yaml`; `ComicNeueAngular` is not in the enum, so it is loaded by family name (spike S-P6-12) |
| Ink | `#1A1A1A` | GDD §12.7 3 px ink borders; §13.4 ink rim |
| Paper / PaperShade | `#FFF8EC` / `#F3E6CF` | GDD §12.7 paper |
| Candy accents | Bubblegum `#F48FB1`, Mint `#7FD8BE`, Lavender `#B39DDB`, Cream `#FFF3D6`, Mauve `#9C6B8A` | GDD §12.6 |
| HP | Fill `#6BD06B`; low (≤ 30 HP) `#FF6F61`; drain `#FFFFFF`; guard `#4FC3F7`; pip `#FFD54F`; focus `#FFFFFF` | (td); drain white per GDD §15.3 |
| Telegraph swatches | `#FF8A1F` (default "warm orange"), `#D55E00`, `#F0E442`, `#56B4E9`, `#CC79A7`, `#FFFFFF` | GDD §16.3; settings default `#FF8A1F` (Part 2 §2.8.6) |
| Text sizes | S 14, M 18, L 24, XL 36, Huge 64 | GDD §16.3 min 14 |
| Stroke / corner / pad / gap | 3 / 14 / 12 / 8 | GDD §12.7 3 px |

**Rarity encoding** (GDD §8.3 shapes and colour names; GDD §16.3: colour is never the only cue). Each badge is the shape glyph in the rarity colour inside a border whose style also varies, so three cues differ:

| Rarity | Glyph | Default colour | CVD colour (all three modes; Okabe–Ito) | Border |
|---|---|---|---|---|
| `starter` | ♥ | `#E57373` | `#E69F00` | solid 2 px |
| `common` | ○ | `#4CAF50` (grass green) | `#009E73` | solid 2 px |
| `uncommon` | ◇ | `#42A5F5` (candy blue) | `#56B4E9` | solid 3 px |
| `rare` | △ | `#F06292` (bubblegum pink) | `#CC79A7` | double (2 strokes) |
| `epic` | ☆ | `#8E44AD` (nightosphere purple) | `#0072B2` | double, thick |
| `legendary` | ✦ | `#FFB300` (flame gold) | `#E69F00` | dashed (4 segments) |
| `mythic` | ❂ | prismatic `UIGradient` (`#FF5252` → `#FFEB3B` → `#69F0AE` → `#448AFF` → `#E040FB`) | `#F0E442` | gradient stroke |
| `cosmic` | ✺ | `#1B0B2E` with an animated gradient (Rotation +90°/s; static with Reduced Motion) | `#D55E00` | animated stroke |

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

**Style palette helper** (canonical code in Part 8 §8.2.7, reproduced here: `Theme.stylePalette(styleId, variantId?)` with this part's variant rule and the `CodenameLooks` switch, §6.8.2; under the switch it reads `Skin.look(styleId, "Original")`). Every presentation colour that depends on a style (proxy rig, Style Gear, nameplate chip and ring, Style Wheel chips, VFX tints, Shop cards) goes through it; nothing reads `StyleDef.palette` directly.

```lua
--!strict
-- Theme.stylePalette — style colours for presentation. Spec §6.9.4 (canonical code: Part 8 §8.2.7).
-- Excerpt of UI/Theme.luau: the requires go at the top of the module; `Theme` is the module table above.
-- Variant rule: §6.8.2 (v1 swaps secondary/accent; v2 and later darken 25 %). Kill switch: Part 8 §8.6.3.
local Skin = require("@game/ReplicatedStorage/Shared/Skin")
local Styles = require("@game/ReplicatedStorage/Shared/Styles")
local Runtime = require("@game/ReplicatedStorage/Shared/Config/Runtime")

export type StylePalette = { primary: Color3, secondary: Color3, accent: Color3 }
local BLACK = Color3.new(0, 0, 0)

function Theme.stylePalette(styleId: string, variantId: string?): StylePalette
	-- Features.CodenameLooks == false: Original palettes whatever pack is active.
	local pack: Skin.PackId? = nil
	if Runtime.get().Features.CodenameLooks == false then -- Runtime is applied before any presentation runs
		pack = "Original"
	end
	local o = Skin.look(styleId, pack) -- one pack only, no fallback (Part 8 §8.2.7)
	local base: StylePalette = Styles.get(styleId).palette
	if o ~= nil and o.palette ~= nil then
		base = {
			primary = Color3.fromHex(o.palette.primary),
			secondary = Color3.fromHex(o.palette.secondary),
			accent = Color3.fromHex(o.palette.accent),
		}
	end
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

`Color3.fromHex` and `Color3:Lerp` are verified in the offline docs [Docs `ui/styling/index.md` (`Color3.fromHex("FF0099")`), `ui/3D-drag-detectors.md` (`COLOR2:Lerp(COLOR1, t)` on `Color3` values)]. The requires are pure modules, so Theme stays load-pure.

**Verify:** Lune `tests/core/theme.spec.luau`: `theme.contrast` (ink/paper ≥ 15; the telegraph default vs ink ≥ 4.5), `theme.reveal_durations` (all 7 rarities from `Defaults.Perf.Ui.Reveal` match GDD §8.6), `theme.cooldown_frac`, `theme.touch_no_overlap` (§6.9.5 table: no two circles overlap; all inside 1334 × 750; none in the jump reserve), `theme.style_palette_variants` (base = `Styles.get("hero").palette` under Codename; `v1` swaps secondary/accent; `v2` = each channel × 0.75 within 1e-6; with `Features.CodenameLooks = false` the base equals the Original pack's `looks.hero.palette` even while Codename is active).

#### 6.9.5 HUD (`UI/Screens/Hud.luau`, mounted by `HudController.start()`)

Data: the local view snapshot (`RenderController.localView()`), Player attributes (`Meter`, `Streak`, `MapId`, `EquippedStyle`), the pinned combat config (`Runtime.at(CombatCfg)`), the store (`UiController.state()`). Updated in the `ALG_Hud` render step (priority `Last`).

| Element | KBM / Gamepad placement (anchor; offset; size) | Touch placement | Data and behaviour | GDD |
|---|---|---|---|---|
| Kill feed | (1, 0); (−16, 16); 420 × 140 | same | `Combat.Feed`: 4 lines, newest on top, 6 s each; killer and victim names (`Players:GetPlayerByUserId` → `DisplayName`; bots `ui.feed.bot`), style chips, Flair glint from the killer's `Flair`, assists `+n`, streak tag | §15.3 |
| Event banner | (0.5, 0); (0, 16); 640 × 56 | same | `Notify kind = "announce"`: 4 s each, queued | §15.3 |
| Menu strip | (0, 0); (16, 16); 5 × 64 + gaps | same | Buttons: Style (Tab / View), Wish, Shop, Settings, Travel (Training ↔ battle). Wish shows a badge with the `Wishes` count. **Gamepad access:** see "Gamepad menu focus" below | §15.4 |
| Wallet & level | (0, 0); (16, 96); 360 × 56 | same | Row 1: `Skin.text("ui.hud.level", {level})` plus a thin XP bar (`Create.bar`, 200 × 8) filled `xpInto / toNext` from `Pity.accountLevel(account.xpTotal, Runtime.get().Progression.Account)` (full bar at level 100, `toNext = 0`). Row 2: `ui.hud.treasure {n}`, `ui.hud.shards {n}`, `ui.hud.sparks {n}` from `PrivateView.currencies` (`Treasure`, `Shards`, `CosmicSparks`), Number font. Updated on `UiController.changed` for `currencies`/`account`; a `+n` float (boing) on increase. Kept in Simple HUD | §0.0 (Account Level "simple UI"); GDD §17.2 |
| Quests | (0, 0); (16, 160); 360 × 132 | same | Header `ui.hud.quests`; 3 rows from `PrivateView.quests.daily` (in order): `Skin.text("ui.quest." .. q.id, {goal = q.goal})`, a progress bar `q.progress / q.goal`, a tick when `q.claimed`. Rewards are auto-claimed by the server (Part 5 §5.5.9), which also sends the `Notify` reward toast `ui.notify.quest_done`; the HUD only animates the row's tick (no second toast). Hidden when `Progression.Quests.Enabled == false`, when the list is empty, and in Simple HUD | GDD §17.5; Part 5 §5.5.9 (`Quests.Enabled = true` in M1-G) |
| Zone label | (0.5, 0); (0, 80); 400 × 28 | (0.5, 0); (0, 150) | `map.<id>.zone.<ZoneId>` from the local root position; fades after 2 s; shown on zone change | §11.1 |
| HP bar | (0.5, 1); (0, −120); 480 × 28 | (0.5, 0); (0, 88); 420 × 24 (**deviation**, see below) | `HP / maxHp`; white drain over 0.4 s; low colour ≤ 30; shield overlay when a shield buff is active | §15.3 |
| Breakaway pips | (0.5, 1); (0, −96); 2 × (40 × 14) | (0.5, 0); (0, 118) (**deviation**) | `BkP`; a full pip sparkles (boing at 1 Hz); an empty pip shows a refill sweep `BkProg / 1500`; pips are tinted grey while `fNow < BkLockF` (Breakaway lockout, HRP attribute) | §15.3, §6.8 |
| Protection tag | (0.5, 1); (0, −150); 220 × 24 | (0.5, 0); (0, 140) | `ProtF > fNow` → `ui.hud.protected` with whole seconds | §6.2 |
| Hotbar | (0.5, 1); (0, −16); 6 × 72 + 5 × 8 | hidden | Slots M1, 1, 2, 3, 4, Quirk: move name tooltip on hover, key label from the binding (`settings.binds["<Action>.KBM"]` / `["<Action>.Pad"]` override for the current device, else Part 3 defaults; gamepad glyph text `X Y B RB LB →`), radial wipe `Theme.cooldownFrac(Cd<n>, toFrames(def.cooldown), fNow)` from the awakened set when `AwkF0 > 0`. **Kill-switched move:** when the slot's move id is in `Runtime.get().Features.DisabledMoves`, the slot (and its touch-cluster button) is greyed (60 % transparency) with a lock glyph (Frame-drawn padlock: a rounded body plus a `UICorner` shackle; no image, no text); cosmetic only (the sim refuses the move through `StyleRuntime.disabled`, Part 3 §3.4.2) | §15.3; GDD §18.6 |
| Awaken button | (0.5, 1); (300, −16); 88 × 88 | in the cluster | The ring fills with `Meter / 100` (Player attribute, Part 3); at `AwkRdy` a rainbow `UIGradient` rotates (static with Reduced Motion). While awakened, the ring shows the remaining time `(AwkEndF − fNow) / (AwkEndF − AwkF0 − 72)` | §15.3, §6.9 |
| Combo counter | (1, 0.5); (−260, −120); 320 × 96 | (1, 0); (−16, 170) | Shown when the focus target's `CHits ≥ 3` (GDD §15.3): `ui.hud.combo {n}`, `ui.hud.combo_big {n}` at ≥ 10 ("ALGEBRAIC 12 HIT!"), Impact font growing 48 → 72; hides 1 s after the combo ends | §15.3 |
| Toasts | `ALG_Toasts` (0, 1); (16, −220); 360 wide | (0, 0.5); (16, 0) | `Notify` toast/reward/error: auto-stacking, max 4, 4 s each | §15.3 |
| Subtitles | `ALG_Toasts` (0.5, 1); (0, −170); 800 × 32 | (0.5, 0.5); (0, 120) | Bark and cinematic lines | §16.3 |
| Edge indicators | canvas edges, 48 × 48 arrows, max 4 | same | Off-screen Realm markers; enemies ≤ 30 studs in an `ATTACK` startup outside the camera's ±100° cone ("attacks from behind"); active `EventZones` | §16.3 |
| Training panels | §6.4.2 | same | Only when `MapId == "training"` | §4.5 |

**Dash/Breakaway swap** (Part 3 §3.10.5): while the local `St ∈ STUN` and `BkP ≥ 1`, the Dash slot (cluster button and hotbar hint) shows `ui.hud.breakaway` with a sparkle border.

**Touch HP and pips deviation (owner sign-off).** GDD §15.3 places HP and Breakaway pips bottom-centre above the hotbar, and GDD §15.1 says mobile defines the layout. On touch, the bottom-centre band between the thumbstick zone (left 40 %) and the cluster (`BlockBtn` left edge at x ≈ 686 of 1334) is only ≈ 150 reference units wide, too narrow for a 420-wide bar. The touch HUD therefore puts them top-centre. This is a recorded deviation from GDD §15.3 that the owner signs off at the H-10 feel gate (§6.14 item 9); if rejected, the fallback is a 140-wide HP bar at (0.5, 1); (0, −40) with pips above it, re-checked by `theme.touch_no_overlap`.

**Gamepad menu focus** (Part 3 §3.10.1: `InputController.init()` disables CoreGui `Backpack` and `PlayerList` and sets `GuiService.AutoSelectGuiEnabled = false`, so `Tab`/`ButtonSelect` no longer toggle the player list or UI-selection mode). The `UI` context gains a Bool action `MenuFocus` (Pad `DPadDown`, free in Part 3's map). Pressing it while no modal screen is open calls `InputController.setCombatEnabled(false)` and sets `GuiService.SelectedObject` to the menu strip's Style button; the strip buttons are `Selectable` with `NextSelectionLeft/Right` chained (Wish, Shop, Settings, Travel). `ButtonB` (or `DPadDown` again) clears `SelectedObject` and re-enables combat; activating a button opens its screen as usual (modal screens keep combat disabled until closed). `ButtonB` is `Move2` in combat, but combat contexts are disabled while the strip has focus, so it cannot fire a move. `DPadDown` is not used for anything else (Quick-Chat is a future hook, §6.15).

**Mobile touch cluster** (`ALG_Touch`; centre offsets from the **bottom-right** of the canvas in reference units; negative = left/up). Buttons are created at runtime with `Name` = the Part 3 touch id and bound with `InputController.bindTouch(action, button)` (Part 3 §3.10.3, `InputBinding.UIButton` [Docs]). The layout editor and the one-hand preset are deferred (GDD §15.2, §16.3).

| Button (`Name`) | Action | Diameter | Centre | Notes |
|---|---|---|---|---|
| `PunchBtn` | M1 | 168 | (−300, −150) | "Big Punch button" |
| `Move1Btn` | Move1 | 96 | (−465, −150) | Ring radius 165 around Punch at 180° |
| `Move2Btn` | Move2 | 96 | (−417, −267) | 135° |
| `Move3Btn` | Move3 | 96 | (−300, −315) | 90° |
| `Move4Btn` | Move4 | 96 | (−183, −267) | 45° |
| `QuirkBtn` | Quirk | 72 | (−398, −386) | "Small button above the move ring" |
| `AwakenBtn` | Awaken | 104 | (−199, −395) | "Glowing meter button (top of cluster)"; meter ring as on the hotbar |
| `BlockBtn` | Block | 96 | (−600, −90) | Hold; with `holdToToggleBlock` a tap toggles (§6.9.11) |
| `DashBtn` | Dash | 96 | (−600, −220) | Becomes Breakaway in STUN |
| `LauncherBtn` | Launcher | 80 | (−410, −62) | Visible only while the local `St == M1` with `Sub == 3`, or within the chain window after M1-3. A tap toggles `InputController.fire("Launcher", true)`; it resets to false when the Ender starts or the chain window closes (GDD §15.2) |
| `ShiftLockBtn` | UI `ShiftLock` | 64 | (−60, −470) | Touch shift-lock toggle (`AimLock`, Part 3) |
| Jump | Jump | — | jump reserve x [−210, 0] × y [−210, 0] | Our `Jump` action's `Touch.UIButton` is bound to the default `JumpButton` found with `PlayerGui:FindFirstChild("JumpButton", true)` once `IsLoaded` (the docs' own pattern [Docs `ui/position-and-size.md`]). Spike S-P6-3; fallback: our own `JumpBtn` (120) at (−105, −105) and hide the default one |

After layout, if the default `JumpButton`'s absolute rectangle intersects any cluster button, the whole cluster shifts left by the overlap + 16. `ALG_Touch.Enabled = (UserInputService.PreferredInput == Enum.PreferredInput.Touch)`; the hotbar shows otherwise. Both update on `PreferredInput` changes.

**UI input context.** Part 3 assigns Emote and Style Wheel to "Part 6's `UI` context". InputController (Part 6 section) creates a client-local `InputContext` named `UI` under `Players.LocalPlayer` (`Priority = 1000`, `Sink = false`) with four Bool actions: `StyleWheel` (KBM `Tab`, Pad `ButtonSelect`, GDD §15.2 "View"; usable because Part 3's `InputController.init()` disables the CoreGui PlayerList and UI auto-selection first), `ShiftLock` (KBM `LeftShift`), `Emote` (KBM `B`, Pad `DPadUp`; M1-G only shows the toast `ui.toast.emotes_soon`; the emote wheel is a future hook, §6.15) and `MenuFocus` (Pad `DPadDown`; above). Touch uses the menu-strip buttons and `ShiftLockBtn` directly. It reacts to `InputAction.Pressed`. Whether a client-created context fires under Server Authority is spike S-P6-7; fallback: `UserInputService.InputBegan`.

**Verify:** scenarios `hud.hp_drain` (dummy deals 15 → fill snaps; drain reaches the fill within 0.5 ± 0.05 s), `hud.cooldown_wipe` (use `hero.1` → its radial fraction is 1.0 ± 0.02 at use and 0 at `Cd1`), `hud.meter_ring` (`Dev.Command setMeter 100` → rainbow gradient present), `hud.breakaway_swap` (hitstun → the Dash label reads `ui.hud.breakaway`), `hud.touch_binding` (touch emulation: `Players.LocalPlayer.Combat.M1.Touch.UIButton == PunchBtn`), `hud.combo_counter` (3-hit M1 string on a dummy → "3" shown), `hud.simple_preset`, `hud.wallet_level` (`Dev.Command grant Treasure 100` → the Treasure label rises by 100 after the next delta; `Dev.Command setLevel 2` → the level label reads `Skin.text("ui.hud.level", {level = 2})`), `hud.quests_progress` (fresh profile on a battle map; drive row 1 to its goal with the Part 5 seam that matches its id — `simActivity {sec, dmg}` for `active_minutes`/damage quests, `simKo` for `kos` — then `flushProgress {}` → its bar reads 1.0, the row shows a tick and exactly one `ui.notify.quest_done` toast appears; Simple HUD hides the panel), `hud.move_lock_glyph` (`setConfig Features {"DisabledMoves":["hero.1"]}` → slot 1 shows the lock glyph within 1 s); `mcp.gamepad_menu_nav` (gamepad emulation: `DPadDown` → `GuiService.SelectedObject` is the Style button; one DPad-right → the Wish button; `ButtonA` opens Wish; after closing it, `ButtonB` clears focus, and the next `ButtonB` press (Move2) leaves Idle). This HUD also relies on Part 7's `mcp.tab_no_playerlist` (Part 7 §7.4.10, M0: CoreGui `PlayerList` and `Backpack` disabled and `GuiService.AutoSelectGuiEnabled == false` after `Tab`; once the Style Wheel exists, `Tab` also opens it). MCP `screen_capture` at the Studio device emulator "iPhone SE" profile (human selects the profile; H-12) for review.

#### 6.9.6 Style Wheel (`UI/Screens/StyleWheel.luau`)

GDD §15.4: owned styles by rarity ring, free-rotation marks, Mastery level, equipped Flair, a per-style Character Look toggle; opens on death with a 3 s auto-confirm of the last style.

| Mode (`args.mode`) | Opened by | Lists | Confirm action |
|---|---|---|---|
| `menu` | menu strip / `StyleWheel` action | All 18 styles in rarity rows | `Style.Equip` → applies at respawn (`NextStyle` shown) |
| `death` | local `St` enters `KO` | Owned, implemented styles only (compact strip) + [All] button | Same. Auto-closes after `Perf.Ui.StyleWheelAutoConfirmSec` (3 s, GDD §15.4) with no request when untouched; any touch cancels the countdown |
| `totem` | Totem prompt | Owned, implemented styles | `Style.Equip` (server infers totem, applies now, §6.3.6) |
| `trial` | Training panel / Training totem | `Styles.implemented()` with a "Trial" badge | `Style.Equip` then `World.Travel {"training"}` (§6.4.2) |

Layout ("rings" are rows; 1334 × 750): a centred panel 1180 × 640. Rows top to bottom: Starter (2), Common (3), Uncommon (3), Rare (3), Epic (3), Legendary (2), Mythic (1), Cosmic (1). Each row has its rarity badge at the left and tiles of 112 × 128. A detail pane (360 wide) on the right.

| Tile state | Look | Source |
|---|---|---|
| Owned, implemented | style chip in the palette, name, `M<level>`, Flair glint | `PrivateView.styles[id]`; level from `masteryXp` with `Progression.Mastery` (XpBase + XpPerLevel·m curve, Part 2 §2.6.2), computed by the same formula as `ProgressionService.masteryLevel` |
| Owned, `implemented = false` | chip at 60 % + **In development** badge (`ui.wheel.in_dev`); Equip disabled | §0.10 |
| Trial | chip + `ui.wheel.trial` badge | `MapId == "training"` and `Progression.TrainingTrialAllStyles` |
| Free rotation | `ui.wheel.free` badge | `Progression.FreeRotation.Enabled` (true in M1-G: badge shown for rotation styles; Part 5 §5.1.4, DES-6) |
| Not owned | silhouette (palette primary at 20 %, "?"), rarity badge, hint `ui.wheel.get_hint` (Wishes or Choose Goose) | |
| Cosmic, not owned | "???" (GDD §8.3: secret) | |
| Equipped / next | thick ink ring / dashed ring | `EquippedStyle` / `NextStyle` |

Detail pane: name and epithet (`Skin.style(id)`), move list (`Skin.move` for 1–4, q; tags as chips), Flair picker (owned Flairs → `Style.SetFlair`), **Variant picker**, **Character Look quick toggle** (segmented Inherit / On / Off → `Appearance.Set {styleOverride = {styleId, value}}`; current from `settings.characterLookByStyle[id]`), **Mastery and Shards**, [Equip].
- **Variant picker** (owned styles; GDD §8.7, §9.3): label `ui.wheel.variant`; a row of swatch chips: `ui.wheel.variant_base` (Base, `variantId = ""`) plus one chip per owned variant in `PrivateView.styles[id].variants` (sorted `v1`, `v2`, …), each chip drawn in `Theme.stylePalette(id, vid)` with its name `Skin.text("variant.<styleId>.<vid>.name")`. Selected = `styles[id].equippedVariant` (thick ink ring). Tapping a chip sends `Style.SetVariant {styleId, variantId}` (Part 2 §2.7.4 row 6); on success the owner's `Variant` attribute changes and the look refreshes (§6.8.3). Hidden when the style owns no variants.
- **Mastery and Shards** (owned styles; GDD §9.2): `M<level>` with an XP bar (`Pity.masteryLevel` / `Pity.masteryXpToReach` with `Progression.Mastery`), the line `ui.wheel.shards {n}` (`currencies.Shards`), and while `level < Progression.Mastery.ShardCapLevel` (20) a stepper (1 … `ceil((Pity.masteryXpToReach(20) − masteryXp) / Progression.Mastery.ShardXp)`, clamped to the balance) with [`ui.wheel.convert`] → `Mastery.SpendShards {styleId, shards}`; the response `MasterySpendRes` shows a reward toast with `xpGained` and the new `level`. At level ≥ 20 the stepper is replaced by the disabled text `ui.wheel.mastery_combat_only` (Mastery 21–30 is earned in combat).

**Verify:** scenarios `ui.wheel_death_autoconfirm` (KO → wheel opens within 0.2 s; untouched → closes at 3 ± 0.1 s; no `Style.Equip` sent), `ui.wheel_equip_next` (pick `stretch` → `NextStyle == "stretch"`; after respawn `EquippedStyle == "stretch"`), `ui.wheel_in_dev_badge` (grant `sentry` via `Dev.Command giveStyle` → badge shown, Equip disabled), `ui.wheel_cl_toggle` (Off for `hero` → `settings.characterLookByStyle.hero == false` in the next delta), `ui.mastery_level_matches_server` (10 XP values: the client formula equals `ProgressionService.masteryLevel` via `execute_luau`), `ui.wheel_variant_equip` (read cosmetic slot 4's `itemId` `<styleId>/<vid>` from `Shop.Get`; `Dev.Command giveStyle <styleId>`, `grant Treasure` and buy slot 4 as in `ui.shop_buy_variant`; equip `<styleId>` → the picker shows Base + `<vid>`; tapping `<vid>` → `styles[<styleId>].equippedVariant == <vid>` in the next delta and the owner's `Variant` attribute equals `<vid>`; tapping Base → `""`), `ui.wheel_spend_shards` (`Dev.Command setMastery hero 0`, `grant Shards 400` → the stepper max is `ceil(masteryXpToReach(20) / 60)`; converting the max → level 20, the remaining Shards equal 400 minus the shards the server reports spent, and the stepper is replaced by `ui.wheel.mastery_combat_only`).

#### 6.9.7 Wish screen (`UI/Screens/Wish.luau`)

GDD §8 / §15.4: 1× / 10×, the odds button always visible, pity counters, reveal theatre; a wish jar and shooting stars, never gambling imagery (GDD §16.1).

Layout (modal panel 1240 × 680): left, the wish jar (rounded Frame 420 × 420, glass `UIStroke`, floating ★ particles drawn as TextLabels); bottom centre, [Wish ×1] and [Wish ×10] with costs and the `Wishes` balance; top right, **[Odds]** (always visible, never disabled); right column, pity counters; bottom left, Lucky Jar status (`passes.lucky_jar`).

**PRI-restricted wallet** (Part 5 §5.2.10, §5.2.14; GDD §17.1). When `PrivateView.policy.paidRandomRestricted` is true, the balance shown is the **spendable** count `currencies.Wishes − wishStats.paidUnspent` (never below 0), and when `paidUnspent > 0` a second line `Skin.text("ui.wish.locked_paid", {n = paidUnspent})` appears under it. [Wish ×1] / [Wish ×10] are disabled while spendable < 1 / < 10. A response `{ok = false, err = "restricted", detail = "paid_wishes_locked"}` is shown as that same locked-paid line (pulsed once), not as the generic `ui.error.restricted` toast. Unrestricted players see the plain `Wishes` balance.

| Pity row | Value | Max | Note |
|---|---|---|---|
| Epic+ | `pity.sinceEpicPlus` | Epic hard pity (Part 5 `Wish.Pity`) | GDD §8.4: 30 |
| Legendary+ | `pity.sinceLegendaryPlus` | Legendary hard pity | 120; soft-pity marker from 60 with the live rate `OddsView.legendaryPlusPctNow` |
| Mythic+ | `pity.sinceMythicPlus` | Mythic hard pity | 400 |
| Cosmic Sparks | `currencies.CosmicSparks` | `Wish.CosmicSparksNeeded` (10) | [Craft] enabled at 10 → `Wish.CraftCosmic` |

Part 5 owns the `Wish.Pity` key names; this screen reads them by those names (§6.15).

**Odds panel** (modal over the screen): from `Wish.GetOdds` → `OddsView`, refetched on open and after every Wish (GDD §8.8 "update live"). Sections: per-tier rows (badge, name, %), expandable per-style rows (`OddsStyle.pct`, owned tick, **In development** badge when `implemented = false`; the **Cosmic** style row shows `ui.wish.secret` ("???") instead of its name and chip unless the viewer owns it (GDD §8.3: secret until the first global pull), with its % still printed so the odds list every item and sum to 100 %); the Flair table (`OddsView.flairs`); below it the fake-out disclosure `Skin.text("ui.wish.fakeout_note")` shown as-is with no arguments (GDD §8.6: fake-outs are pre-announced; the pack text carries no `{pct}` token); a "Current effective odds" block: Legendary+ now `legendaryPlusPctNow`, Lucky Jar active, and pity guarantees. A Studio check logs `[ALG][UI] ERROR odds_sum` when the style %s or Flair %s do not sum to 100 ± 0.01.

**Making a Wish.** Capture `seenBefore[r] = flags.seenReveals[r]` from the store **before** sending `Wish.Make {count}`. On the response (`WishBatchResult`):

| Case | Presentation |
|---|---|
| 1× | Reveal `results[1]` |
| 10× | Reveal `results[best]` first, then a 5 × 2 summary grid (best highlighted); duplicates faded (`CanvasGroup.GroupTransparency = 0.4`) with `ui.wish.shards {n}`; Flair labels; **In development** badges (GDD §8.6) |
| Error | `ui.error.<err>` toast (`insufficient` → [Shop] button; never a Robux offer right after a lost fight, GDD §8.8: the [Shop] button is omitted while `now − lastLocalKoAt < Economy.NoOfferAfterLossSec`, §6.9.10); `restricted` + `paid_wishes_locked` → the locked-paid line above |

Reveal theatre per rarity (durations from `Perf.Ui.Reveal`, GDD §8.6 first / repeat; `Theme.revealDuration`):

| Rarity | First / repeat (s) | Placeholder theatre |
|---|---|---|
| common, uncommon | 1.2 / 0.5 | Card flip (UIScale X 1 → 0 → 1), light chime (`sfx.wish.chime`, 2 notes) |
| rare | 2.0 / 0.8 | + pink shimmer gradient sweep, style chip pose (boing), 3 notes |
| epic | 3.0 / 1.2 | + panel tint, silhouette tease (dark chip) before the flip; local announcement (Part 5 Notify), 4 notes |
| legendary | 4.5 / 2.0 | + the jar "cracks" (3 ink crack lines tween in), full chip pose, bark `ui.vo.legendary`; server announcement, 5 notes |
| mythic | 6.0 / 3.0 | + cosmic backdrop gradient (within the panel), shooting stars; cross-server announcement, 6 notes |
| cosmic | 8.0 / 4.0 | + glitch: UI elements jitter ±6 ref px at 12 Hz (off with Reduced Motion), Music muted, `sfx.wish.glitch`; global announcement |

Skip: a tap skips **only** when `seenBefore[rarity]` was true; first views play in full (GDD §8.6). `skipSeenReveals` (default true) selects the repeat duration. Fake-outs (`WishResult.fakeOut`): the card first shows the next lower rarity, then glitches up to the real one at 50 % of the duration; it only ever upgrades. Photosensitivity: no element flashes more than 3 times per second (GDD §16.3).

**Quick Wish pass** (`PrivateView.passes.quick_wish`; Part 5 §5.6.5; GDD §17.3 "instant reveals, auto-10× wishing"). While the pass is owned:
- Reveals of a rarity already seen (`seenBefore[r]`) use duration 0: the result card (1×) or the 10× summary grid appears directly, with no theatre. A rarity's **first** view still plays in full once (GDD §8.6 "skip after the first time you've seen a given rarity's reveal"); the pass removes every later delay.
- An **[Auto ×10]** toggle (`ui.wish.auto10`) appears next to [Wish ×10]. While on, it sends `Wish.Make {count = 10}` repeatedly, each request only after the previous response and ≥ 2.0 s after the previous send (`1 / perSec` of the remote's `{burst = 2, perSec = 0.5}` bucket, Part 2 §2.7.4 row 8, so a long auto run never hits `rate_limited`), while the spendable balance is ≥ 10. It stops on any tap, when the screen closes, on any `legendary` or rarer result, on a style the player did not own before the batch, and on any error response. Each batch is a normal `Wish.Make`; nothing is batched or skipped server-side.
- Without the pass neither behaviour exists (the toggle is not created).

**Contract for Part 5** (§6.13 item 8): WishService sets `flags.seenReveals[rarity] = true` for every rarity in the batch inside the same `DataService.update` as the roll.

**Verify:** scenarios `ui.wish_odds_live` (open Odds; `Dev.Command setPity 0 70 0` → the Legendary+ now value updates on the next fetch; the fake-out note label exists with the text `Skin.text("ui.wish.fakeout_note")`; on a profile without the Cosmic style its row reads `Skin.text("ui.wish.secret")` and no label contains the Cosmic style's `Skin.style` name), `ui.quick_wish_instant` (`Dev.Command setPass quick_wish true`, `grant Wishes 2`; after one 1× whose rarity was already seen, the second 1× shows its result card ≤ 0.1 s after the response; [Auto ×10] exists only with the pass), `ui.wish_restricted_wallet` (`simulatePurchase wishes_10` while unrestricted, then `setPolicy true` on a profile with 0 free Wishes → balance label 0, the locked line shows 10, ×1 and ×10 are disabled), `ui.wish_reveal_first_unskippable` (fresh profile, `Dev.Command grant Wishes 1`, a tap during the first common reveal does not skip; the second common reveal skips on tap and lasts ≤ 0.5 s), `ui.wish_10x_grid` (10 cards, best first), `ui.wish_no_gambling_terms` (Part 8 lint: Wish screen Skin values contain none of "slot", "spin", "jackpot", "roulette", "casino", "bet").

#### 6.9.8 Shop (`UI/Screens/Shop.luau`)

Tabs **[Choose Goose]** and **[Store]**.
- **Header** (both tabs): the Treasure balance `Skin.text("ui.shop.balance", {n = currencies.Treasure})`, updated on `currencies` deltas, so players always see what they can spend.
- **Choose Goose** (GDD §8.5): `Shop.Get` → `ShopView`.
  - **Style slots 1–3** (`ShopSlot.kind == "style"`): cards 300 × 380: chip, rarity badge, name, price in Treasure, Owned/Bought state, [Buy]. When `Styles.get(itemId).implemented == false` the card carries the **In development** badge (`ui.wheel.in_dev`, §0.10), and [Buy] first opens a confirm modal with `ui.shop.confirm_in_dev` ("This style is still in development. Collect it now and play it when it ships?"); only its confirm sends `Shop.Buy`.
  - **Cosmetic slots 4–7** (`ShopSlot.kind == "cosmetic"`, `itemId = "<styleId>/<variantId>"`, Part 5 §5.4.3), shown while `Shop.CosmeticSlotsEnabled` (true in M1-G): cards 220 × 300 with the style chip drawn in `Theme.stylePalette(styleId, variantId)`, `Skin.text("variant.<styleId>.<variantId>.name")`, price in Treasure, Owned/Bought state, [Buy] → `Shop.Buy {slot, rotationId, price}`. Empty slots (no `ShopSlot` at that index) are hidden. The "Coming soon" placeholder (`ui.shop.coming_soon`) is shown only while `CosmeticSlotsEnabled == false`. A bought variant is equipped from the Style Wheel Variant picker (§6.9.6).
  - A refresh countdown `refreshAt − workspace:GetServerTimeNow()`; the merchant line `ui.shop.goose_line_<n>` (n picked uniformly from 1 to the highest n for which `Skin.has` is true, Part 8 §8.3.7). Buy → `Shop.Buy {slot, rotationId, price}`: `stale` → refetch and toast `ui.error.stale`; `insufficient` → toast.
- **Store** (GDD §17.3, §8.8): `Store.Get` → `StoreView`. Wish packs and passes with `priceRobux`, a **Mock** badge when `StoreView.mock`; items with `hidden = true` are never shown (PRI items are hidden when restricted, Part 2 §2.11.4). [Buy] → `Store.Prompt {key, entryPoint = "shop_tab"}`: `prompted` → wait for the delta; `needs_confirm` → confirm modal (`ui.store.confirm_4th`) then resend with `confirmed = true`; `mock_granted` → reward toast. A "spent this week" line shows `Skin.text("ui.store.spent_week", {robux = StoreView.spentThisWeekRobux})` (R$, UTC Monday week; GDD §8.8 guardrail). `StoreView.wishBuysLast24h` drives only the 4th-purchase confirm modal.

**Verify:** scenarios `ui.shop_buy_flow` (grant 20,000 Treasure; buy slot 1 → the card shows Bought; the header balance and Treasure decrease by the price), `ui.shop_in_dev_badge` (a rotation whose style slot holds an unimplemented style, e.g. `sentry` → the card shows `ui.wheel.in_dev`; [Buy] opens the `ui.shop.confirm_in_dev` modal and sends no `Shop.Buy` until confirmed), `ui.shop_buy_variant` (grant Treasure; buy cosmetic slot 4 → the card shows Bought; `styles[<styleId>].variants[<variantId>] == true` in the next delta; empty cosmetic slots are not rendered), `ui.store_restricted_hidden` (`Dev.Command setPolicy true` → no Wish pack or Lucky Jar card), `ui.store_confirm_4th` (4th mock Wish purchase in 24 h → the confirm modal appears), `ui.store_week_spend` (mock purchases of `wishes_10` then `wishes_55`, as in Part 5 `store.week_spend` → the spent-this-week line reads `Skin.text("ui.store.spent_week", {robux = 548})`).

#### 6.9.9 Settings (`UI/Screens/Settings.luau`)

Left tabs; right `ScrollingFrame` (`AutomaticCanvasSize = Y`) with a `UIListLayout`. Every control writes through `SettingsController.set(key, value)` or `setAppearance(req)` (Part 2 §2.3.7), so the change applies locally at once and is confirmed by the next delta.

| Tab | Controls (setting key) | Applied by |
|---|---|---|
| Appearance (§0.4, GDD §12.3) | Character Look global (`characterLook`); per-style overrides list (owned styles: Inherit/On/Off → `styleOverride`); Your look (`ownLook`: Ooo-sona / Roblox Avatar); Style Gear (`showStyleGear`); Show other players as (`othersAppearance`: As chosen / Always characters / Always their own looks / Simplified); Ooo-sona editor: species ◀▶ (10; label `Skin.text("ui.oosona.species." .. Appearance.OosonaSpecies[i].id)`), colour swatches (12, `Appearance.OosonaColors`, Part 5 §5.10), face 1–6, hat 0–6, outfit 0–4 → `setAppearance({oosona = spec})`, with a live preview in a `ViewportFrame` (a local `LookRig_R15` clone rotating at 30°/s, static with Reduced Motion) | AppearanceController |
| Graphics | `graphicsTier` (Auto/Low/Mid/High); `outlineThickness` (0–2; graybox: nameplate stroke only, hulls are future) | VfxController.setTier, AppearanceController |
| Accessibility (GDD §16.3) | `reducedMotion`, `reducedVfx`, `cameraShake` (0–1, step 0.1), `colorblind` (None/Deuteranopia/Protanopia/Tritanopia), `telegraphColor` (6 swatches), `textScale` (0.8–1.5, step 0.1), `dyslexiaFont`, `simpleHud`, `subtitles`, `glitchedCalm`, `holdToToggleBlock`, `holdToToggleRun` (`ui.settings.hold_toggle_run`), `damageNumbers` (`ui.settings.damage_numbers`; GDD §15.3), `skipSeenReveals` | §6.9.11 |
| Audio | `volMaster`, `volMusic`, `volSfx`, `volVo`, `volUi` (0–1); `quietBarks` (`ui.settings.quiet_barks`; GDD §14.5: own barks only when on) | SfxController |
| Controls | Rebind rows for `M1, Move1–4, Quirk, Awaken, Block, Dash, Jump, Run` × KBM/Pad: [Change] captures the next `UserInputService.InputBegan` key (KBM column: keyboard/mouse inputs only; Pad column: gamepad inputs only) → `SettingsController.set("binds.<Action>.<Device>", KeyCode.Name)` with `<Device>` = `KBM` or `Pad`; [Reset] → `""` (Part 3 default). Duplicate keys within a column are refused with a toast. Mobile: `ui.settings.layout_editor_soon` | **SettingsController** (single owner, Part 2 §2.3.7 rule 6) applies `binds.<Action>.<Device>` to `Player.Combat.<Action>.<Device>.KeyCode`; InputController never writes binding KeyCodes (Part 3 §3.10.1) |
| Safety (`ui.settings.tab_safety`; GDD §16.2) | The list of other players in the server, each with a [`ui.report.button`] button that opens the Report flow (§6.9.12) | UiController |

**Verify:** scenarios `ui.settings_roundtrip` (each key set via the UI → the store shows the value after the next delta; `textScale 1.5` → a 14 px label becomes 21), `ui.rebind_m1` (binding M1 to `E`, the Totem prompt key, is refused with `ui.toast.key_in_use`; binding it to `V` → the store holds `binds["M1.KBM"] == "V"` and `Players.LocalPlayer.Combat.M1.KBM.KeyCode == Enum.KeyCode.V`; then rebinding Pad M1 through a `binds.M1.Pad` set leaves `M1.KBM.KeyCode == V`, proving the two devices are stored separately), `ui.oosona_preview` (species → penguin, colour swatch 1, outfit 0: preview head/arms `#2E3440`, torso `#F2C14E`, legs `#F2C14E` × 0.65; Part 5 §5.7.1), `ui.settings_new_toggles` (`quietBarks`, `damageNumbers`, `holdToToggleRun` each toggle via the UI and read back from the store after the next delta).

#### 6.9.10 Results, notifications, subtitles, FTUE-lite

- **Results card** (`UI/Screens/Results.luau`; not modal): on `Life.Summary`, a 520 × 200 card at top-centre for 6 s or until respawn. Killer chip and name (or `ui.results.environment`), damage dealt and taken, KOs, assists, life time, Treasure / XP / Mastery XP earned. When the killer is a **player** (not a bot, not the environment, not self), the killer chip carries a small [`ui.report.button`] button that opens the Report flow with that player preselected (§6.9.12). It never contains Store entry points (GDD §8.8: no offers right after a lost fight).
- **Notify** (Part 2 §2.7.4 row 22): `toast` / `reward` → toast stack; `error` → red toast; `announce` → banner (Mythic/Cosmic announcements use the rarity gradient). Text = `Skin.text(key, args)`; `ttl` overrides 4 s.
- **Subtitles** (GDD §14.5, §16.3): every bark and cinematic line, when `subtitles` is on.
- **FTUE-lite** (GDD §15.5; the full tutorial path is a future hook): while `account.level == 1` and `account.xpTotal == 0`, one-time contextual hint toasts per session: `ui.tip.punch` at spawn, `ui.tip.block` after the first hit taken, `ui.tip.dash` after 20 s, `ui.tip.awaken` on the first `AwkRdy`, `ui.tip.wish` when `Wishes ≥ 1` and out of combat for 10 s (with a [Make a Wish] button).
- **FTUE steps** (Part 5 §5.8.4 `Ftue.Step`, C2S event, rate 5 / 1). UiController sends `Net.client.fire("Ftue.Step", {step = n, skipped = false})` once per milestone, in increasing order (the server ignores non-increasing steps), only while `flags.ftueDone == false`:

  | Step | First time the local player … | Detected by |
  |---|---|---|
  | 2 | sees `ui.tip.punch` | FTUE-lite toast shown |
  | 3 | lands an M1 contact | local `Cnt` 0→1 in an `M1` state (§6.6.1) |
  | 4 | enters `Block` | local `St` enters `Block` |
  | 5 | dashes | local `St` enters `Dash` |
  | 6 | breaks away | local `St` enters `Breakaway` |
  | 7 | awakens | local `St` enters `Awakening` |

  Step 1 (cold open) and steps 8–9 are server-side or not emitted in M1-G. A milestone reached out of order (e.g. a Dash before the first Block) is sent when it happens; the skipped lower steps are then never sent for that profile.
- **No offer after a loss** (GDD §8.8 "Wish offers are never shown right after a lost fight"; Part 5 §5.6.4 `Economy.NoOfferAfterLossSec`, 30 s). UiController records `lastLocalKoAt = os.clock()` when the local `St` enters `KO` (presentation clock; not sim). While `os.clock() − lastLocalKoAt < Runtime.get().Economy.NoOfferAfterLossSec`, `ui.tip.wish` and any other **unsolicited** Wish or Store prompt (tip buttons, the `insufficient` → [Shop] shortcut, any future offer banner) are suppressed; a suppressed `ui.tip.wish` is re-evaluated after the window. Screens the player opens deliberately (menu strip) are never blocked.

**Verify:** scenarios `ui.results_card` (`Dev.Command kill` → card within 0.5 s of `Life.Summary`, gone after respawn; with a bot killer no report button; with a player killer in a two-client run the button exists), `ui.notify_kinds` (server fires each kind → the expected container), `ui.no_offer_after_loss` (`grant Wishes 1`; `Dev.Command kill`; respawn and stay out of combat → no `ui.tip.wish` toast within `NoOfferAfterLossSec` of the KO; it appears once after the window plus 10 s out of combat), `ui.ftue_steps` (fresh profile: punch tip, M1 contact, block, dash → `flags.ftueStep` reads 2, 3, 4, 5 in order in the deltas).

#### 6.9.11 Accessibility matrix (GDD §16.3)

| Feature | Implementation | Verify |
|---|---|---|
| Reduced Motion | Camera shake scale 0; no impact frames (→ outline pulse); no screen flashes; no awaken orbit; no UI boing, slides, glitch jitter; rotating gradients static | `cam.reduced_motion`, `vfx.impact_frame` |
| Reduced VFX | Budget × 0.4; enemy VFX opacity 60 %; telegraphs untouched | `vfx.budget_tiers`, `vfx.telegraph_never_reduced` |
| Photosensitivity | `CameraController.flash` ≤ 3/s, alpha ≤ 0.5 white; Glitched calm variant | `cam.flash_rate` |
| Colour-blind modes | Rarity CVD palette (§6.9.4) + shapes + borders; telegraph swatch default per mode: None `#FF8A1F`, Deut/Prot `#E69F00`, Trit `#D55E00` (applied only while the user has not picked a swatch) | `ui.cvd_rarity` (capture for review) |
| Telegraph contrast | User colour + adaptive ink/white outline ≥ 4.5:1 to the ground | Lune `theme.contrast` |
| Text scale / dyslexia font | `Create.label` rules; `restyleAll` on change | `ui.settings_roundtrip` |
| Audio buses, subtitles, visual sound indicators | SfxController buses; subtitles; HUD edge indicators | `sfx.bus_volumes`, `hud.edge_indicators` |
| Hold-to-toggle block | With `holdToToggleBlock`, InputController clears the `Block` action's `KBM`/`Pad`/`Touch` bindings on the client's own context and instead toggles the `Script` binding (`InputBinding:Fire`) on each press | `input.block_toggle` |
| Hold-to-toggle run | With `holdToToggleRun` (GDD §16.3 "hold-to-toggle for block and run"), a Run press latches `RunAuto` on (InputController fires the `RunAuto` Script binding) until the next Run press, a stop of movement input for ≥ 0.25 s, or any non-NEUTRAL state | `input.run_toggle` (key on: one Run tap while moving → `RunAuto` stays true for 2 s of held movement; a second tap → false) |
| Simple HUD | §6.9.3 | `hud.simple_preset` |
| Damage numbers | Settings key `damageNumbers` (boolean, default `false`; Part 2 §2.8.6): **off by default everywhere** (GDD §15.3); enabled from the Training panel toggle (`ui.training.damage_numbers`) or Settings ▸ Accessibility (`ui.settings.damage_numbers`), both writing the same key. RenderController shows them only while it is true; `Perf.Ui.DamageNumbersDefault = false` is used before settings load. Simple HUD does not override an explicit `true` | `ui.settings_new_toggles`, `training.damage_numbers_toggle` |
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

All sounds are the 11 built-in files (agent-assets §2.7; §0.5) as `rbxasset://sounds/<file>`, manifest entries `{status = "placeholder", source = "builtin", kind = "audio"}`. Per-move keys (`sfx.<style>.<slot>`) are Part 4 §4.7.3. The generic keys below carry `speed`, `vol`, `prio` (GDD §14.6 order) and `bus` in `Perf.Audio.Map`.

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
| `sfx.awaken.sting` | `impact_explosion_03.mp3` | 0.7 | 0.9 | 3 | Sfx | Awakening start |
| `sfx.ko.poof` | `impact_explosion_03.mp3` | 1.0 | 0.8 | 3 | Sfx | KO poof (GDD §16.1) |
| `sfx.wallsplat.splat` | `impact_water.mp3` | 0.8 | 0.7 | 3 | Sfx | Wall Splat |
| `sfx.ragdoll.land` | `action_jump_land.mp3` | 0.8 | 0.5 | 5 | Sfx | First ragdoll landing |
| `sfx.spawn.ready` | `action_get_up.mp3` | 1.1 | 0.4 | 5 | Sfx | Spawn protection ends |
| `sfx.spring.boing` | `action_jump.mp3` | 0.7 | 0.7 | 5 | Sfx | Spring launch (from the local root's vertical velocity jump ≥ 60) |
| `sfx.break.shatter` | `impact_explosion_03.mp3` | 1.8 | 0.7 | 5 | Sfx | Breakable broken |
| `sfx.break.redraw` | `volume_slider.ogg` | 0.9 | 0.4 | 7 | Sfx | Storybook redraw start |
| `sfx.realm.open` | `impact_explosion_03.mp3` | 0.5 | 1.0 | 2 | Sfx | Realm opens |
| `sfx.vo.hurt` | `oof.ogg` | 1.0 | 0.6 | 4 | Vo | Local victim, damage ≥ 8 (bark-throttled) |
| `sfx.vo.ouch` | `ouch.ogg` | 1.0 | 0.6 | 4 | Vo | Local victim KO |
| `sfx.ui.click` | `volume_slider.ogg` | 1.2 | 0.6 | 4 | Ui | Buttons |
| `sfx.ui.open` | `action_jump.mp3` | 2.0 | 0.4 | 4 | Ui | Screen open |
| `sfx.ui.error` | `action_falling.ogg` | 1.6 | 0.5 | 4 | Ui | Error toast |
| `sfx.ui.reward` | `action_get_up.mp3` | 1.8 | 0.6 | 4 | Ui | Reward toast |
| `sfx.wish.chime` | `volume_slider.ogg` | 1.0 → 2.0 | 0.8 | 4 | Ui | Wish arpeggio: `n` notes 90 ms apart (n = rarity index + 1), speed rising linearly to 1.0 + 0.15·n, so length and pitch encode rarity (GDD §14.4) |
| `sfx.wish.glitch` | `impact_water.mp3` | 0.3 | 0.9 | 4 | Ui | Cosmic reveal (with Music muted) |

TTS barks (`SfxController.bark`; Studio only; subtitled): `ui.vo.awaken` ("Algebraic!"-style line per the pack), `ui.vo.ko`, `ui.vo.legendary`. Music: no tracks in M1-G; the `Music` bus and ducking exist so stems can drop in (GDD §14.3 is a future hook).

**Verify:** `tools/check_manifest.luau` (Part 8) finds every key above; scenario `sfx.map_complete` (play each key once; no `[ALG][Assets] WARN missing`).

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
			Ink = "#1A1A1A",                                         -- GDD §13.4
			ImpactFrameFrames = 2,                                   -- GDD §6.13
			TelegraphFillTransparency = 0.35, TelegraphOutlineStuds = 0.4, TelegraphContrastMin = 4.5, -- agent-assets §7.4; GDD §16.3
			TelegraphHoldAfterActiveFrames = 2,                      -- (td)
			DebrisLifeSec = { 6, 10 },                               -- GDD §11.1
			RebuildRedrawSec = 3,                                    -- (td)
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
			RingTransparency = 0.35, StealthRevealStuds = 12,        -- (td); GDD §7.3.12 via Part 4
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
			FakeOutUpgradeAt = 0.5,                                  -- (td) fraction of the reveal
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
			EditPreset = "arena_duel_m0.Day",                        -- switched to "candy_kingdom.Day" by the CK task
			TweenSec = 1, CycleEnabled = false, CycleSec = 2700, CycleTweenSec = 10, -- GDD §11.4 (45 min); (td)
			Presets = { --[[ §6.5.2 table, one record per preset:
				["candy_kingdom.Day"] = { LightingStyle = "Soft", ClockTime = 14, Brightness = 2.2, Ambient = "#A88A9C", …,
					Atmosphere = { Density = 0.26, … }, ColorCorrection = { … }, Bloom = { … }, Clouds = { … } }, ]] },
		},
		World = {
			Spring = { LaunchVy = 101, LaunchVyHatch = 130 },        -- (td) apex ≈ 26 / 43 studs
			Slow = { Mult = 0.6 },                                   -- (td) GDD §11.2 A "slows"
			Boundary = { BandStuds = 20, MinMult = 0.2 },            -- (td) GDD §11.1 "gentle push-back"
			Breakable = { HouseHp = 2, HatchHp = 1, RebuildSec = 90, -- GDD §11.1 (90 s)
				InflateStuds = 1.5, MinVy = 40, SpawnClearStuds = 60 }, -- (td)
			ScanHz = 10,                                             -- (td)
			Sightline = { MaxStuds = 150, GridStuds = 20, EyeY = 5 }, -- GDD §11.1; (td)
			Hazard = { MaxDamage = 15, MinTelegraphSec = 1.5 },     -- GDD §11.1 (KO credit window: Part 3 Combat.Streak.EnvKoCreditWindowSec)
			Budget = { arena_duel_m0 = 400, training = 400, candy_kingdom = 2500 }, -- (td) max BaseParts
		},
	},
```

**Appearance:** see Part 5 §5.10 (Part 5 owns the single canonical `Defaults.Appearance` table; decision D-13). It already includes this part's keys `OosonaColors` (Settings swatches, §6.9.9), `SimplifiedLimbShade` (§6.8.2) and `FaceExpressions` (§6.8.5), with `OosonaSpecies` as `{id, skin}` records (read via `.id`), `OosonaHats = 6`, `OutfitColors` and `NoOutfitDarken` (three-colour Oosona, §6.8.2). This part does not write `Appearance`.

`Schema.rules` additions (validated at boot and on overlay, Part 2 §2.6.3): every `Perf.Tiers.<T>.*` int ≥ 0; `Perf.Vfx.LodSimplifiedStuds < LodTelegraphOnlyStuds`; `Perf.Ui.Reveal.<r>[1] ≥ [2]`; `Perf.World.Hazard.MaxDamage ≤ 15`; `Perf.World.Sightline.MaxStuds ≤ 150`; every `Perf.Lighting.Presets` colour matches `^#%x%x%x%x%x%x$`. (`Appearance` rules, incl. `#OosonaSpecies ≥ 1`, are Part 5 §5.10's.)

**Verify:** Lune `tests/core/perf.spec.luau`: `perf.defaults_gdd` (the GDD-cited numbers above against a literal table), `perf.json_safe`, `perf.tiers_ordered` (Low ≤ Mid ≤ High for every budget), `perf.audio_map_complete` (every §6.10 key present).

---

### 6.12 Consolidated test catalog (this part)

| Kind | Names |
|---|---|
| Lune | `movement.volume_*` (4), `theme.contrast`, `theme.reveal_durations`, `theme.cooldown_frac`, `theme.touch_no_overlap`, `theme.style_palette_variants`, `perf.defaults_gdd`, `perf.json_safe`, `perf.tiers_ordered`, `perf.audio_map_complete`, `perf.lighting_presets_complete`, `anims.base64_roundtrip`, `anims.upload_refuses_without_approve` (`tests/core/upload_anims.spec.luau`, §6.6.9), lints `render.no_attr_signal`, `anim.no_track_cache` |
| Builder | Part 1 §1.6.8 set for all 7 Part 6 builders |
| Scenarios: world | `world.spring_launch`, `world.river_slow`, `world.void_kill`, `world.house_break_rebuild`, `world.hatch_fall_and_spring`, `world.rebuild_postponed_when_occupied`, `map.arena_spawn_8`, `map.arena_no_escape`, `map.ck_zone_label`, `map.ck_boundary_soft`, `map.ck_totem_prompt` |
| Scenarios: training | `training.dummy_mode_ui`, `training.framedata_m1`, `training.trial_equip`, `training.damage_numbers_toggle`, `training.reset_travel` |
| Scenarios: presentation | `render.hitstop_not_reentry`, `render.no_double_move_vfx`, `render.remote_telegraph_full` (net profile `M0Gate`), `render.focus_target`, `render.haptic_gate`, `anim.cosmetic_budget`, `vfx.budget_tiers`, `vfx.telegraph_never_reduced`, `vfx.highlight_cap`, `vfx.lod_distance`, `vfx.attach_unanchored`, `vfx.impact_frame`, `vfx.templates_all_play`, `sfx.voice_cap`, `sfx.bark_throttle`, `sfx.bus_volumes`, `sfx.map_complete`, `cam.shift_lock_default`, `cam.reduced_motion`, `cam.flash_rate`, `cam.lighting_follows_map`, `ragdoll.clone_lifecycle`, `ragdoll.cap`, `ragdoll.wallsplat_handoff`, `ragdoll.juggle_rebegin`, `ragdoll.no_misprediction`, `look.style_tell_always`, `look.oosona_three_colour`, `look.variant_render`, `look.face_expressions` (programmatic + one 3×3 capture) |
| Scenarios: multi (level `multi`; run only via `multi(2, Baseline30, <name>)`) | `look.modes_matrix`, `ui.report_submit` (and Part 4's `look.stealth_ranges`) |
| Scenarios: UI | `hud.hp_drain`, `hud.cooldown_wipe`, `hud.meter_ring`, `hud.breakaway_swap`, `hud.touch_binding`, `hud.combo_counter`, `hud.simple_preset`, `hud.edge_indicators`, `hud.wallet_level`, `hud.quests_progress`, `hud.move_lock_glyph`, `ui.wheel_death_autoconfirm`, `ui.wheel_equip_next`, `ui.wheel_in_dev_badge`, `ui.wheel_cl_toggle`, `ui.wheel_variant_equip`, `ui.wheel_spend_shards`, `ui.mastery_level_matches_server`, `ui.wish_odds_live`, `ui.wish_reveal_first_unskippable`, `ui.wish_10x_grid`, `ui.wish_no_gambling_terms`, `ui.quick_wish_instant`, `ui.wish_restricted_wallet`, `ui.shop_buy_flow`, `ui.shop_in_dev_badge`, `ui.shop_buy_variant`, `ui.store_restricted_hidden`, `ui.store_confirm_4th`, `ui.store_week_spend`, `ui.settings_roundtrip`, `ui.settings_new_toggles`, `ui.rebind_m1`, `ui.oosona_preview`, `ui.results_card`, `ui.notify_kinds`, `ui.no_offer_after_loss`, `ui.ftue_steps`, `ui.cvd_rarity`, `input.block_toggle`, `input.run_toggle` |
| Scenarios: MCP input | `mcp.gamepad_menu_nav` (Part 7 §7.4.10 row, owner P6, M1-G) |
| Defined elsewhere, exercised by this part's code | `combat.env_ko_credit` (Part 3 §3.13; world runtime cause), `tel.perf_sample_sent` (Part 5 §5.8.3; VfxController sender), `ip.pack_switch_live` (Part 8; `Skin.onChanged` re-render and `CodenameLooks`), `combat.disabled_move_kill_switch` (Part 3; HUD lock glyph is cosmetic), `mcp.tab_no_playerlist` (Part 7 §7.4.10, M0; Part 3 §3.10.1 CoreGui calls; the Style Wheel half needs §6.9.6) |
| Spikes | `spike.S-P6-1` … `spike.S-P6-14` (§6.15) |

Log lines introduced: `[ALG][World] break|rebuild|void_kill …`, `[ALG][BUILD] reminder Terrain.Decoration is human-set at H-04 (NotScriptable)`, `[ALG][CHECK] upload_anims key=… id=… | FAIL … | SUMMARY ok=<n> fail=<m>` (§6.6.9), `[ALG][Anim] clips n=… / WARN track_budget`, `[ALG][Vfx] WARN missing_template key=…`, `[ALG][Vfx] degrade level=<n>`, `[ALG][UI] ERROR odds_sum …`, `[ALG][UI] screen open=<id>`, `[ALG][Look] mode rid=<n> mode=<m>` (Studio only).

---

### 6.13 New contracts introduced

1. **Map instance layout** (§6.1.1, §6.1.4): map Models with `MapId`, `KillY`, `Origin`, `BuildVersion`; child folders `Geometry`, `Spawns`, `Totems`, `Breakables`, `DummyPoints`; `Zones` is a **Model** with `ModelStreamingMode = Persistent` holding `Volumes`, `KillVolumes`, `EventZones`, `ZoneBounds`. Map origins: arena (−2600, 0, 0), Candy Kingdom (0, 0, 0), Training (2600, 0, 0). The full attribute table of §6.1.4. *(Parts 2, 3, 5, 7)*
2. **Collision groups `MapBoundary` and `RagdollFx`** with the §6.1.5 matrix, registered by Part 3's `Hurtbox.ensureGroups()` (8 of 32). *(Part 3)*
3. **Predicted volumes:** `Movement.volumeEffect` (pure, in `Core/Movement.luau`) and its Adapter application (cache per map at `track`; apply after `Fsm.step`, before `Loco.apply`). *(Part 3)*
4. **World runtime** block in `CombatService` (`CombatService.world` API), which kills with `cause = "environment"`. The 5 s environment KO credit window (GDD §11.1) is **done in Part 3 §3.17** (`Combat.Streak.EnvKoCreditWindowSec = 5`, applied in §3.13.2; scenario `combat.env_ko_credit`); `Perf.World.Hazard` carries no credit key. *(Part 3)*
5. **`Util.luau` MapKit additions:** `Util.KIND`, `mapPart`, `ramp`, `slabRects`, `ensureRootOf`. *(Part 1)*
6. **PlayerService / travel:** (a) set `Player.DevEnableMouseLock = false` at join (server-only property); (b) initial `MapId` = `"candy_kingdom"` if `Workspace.Map.CandyKingdom` exists, else `"arena_duel_m0"`; (c) add `"arena_duel_m0"` to the `World.Travel` enum (valid only when `Workspace.Arena.DuelArena` exists); (d) `canTravel` skips the combat-tag check for `training → training` travel (Training Reset). *(Parts 2, 3)*
7. **InputController additions:** `setShiftLock(on: boolean)`, `shiftLock(): boolean` (feeds `AimLock`); the client-local `UI` InputContext (`StyleWheel`, `ShiftLock`, `Emote`, `MenuFocus` on Pad `DPadDown`, §6.9.5); hold-to-toggle block via clearing bindings plus the `Script` binding; hold-to-toggle run (`holdToToggleRun` latches `RunAuto`, §6.9.11); InputController never writes binding KeyCodes (SettingsController owns rebinds, Part 2 §2.3.7 rule 6); the CoreGui disabling (`Backpack`, `PlayerList`, `AutoSelectGuiEnabled = false`) is Part 3 §3.10.1's `InputController.init()`; touch `LauncherBtn` and `ShiftLockBtn`; binding `Jump.Touch.UIButton` to the default `JumpButton`. BotService writes `DummyPoints.Slot_<n>` attribute `Mode` on every `Training.Dummy` change. *(Part 3)*
8. **Part 5 expectations:** WishService sets `flags.seenReveals[rarity] = true` for every rarity of a batch in the roll transaction; StyleService infers a totem equip per **Part 5 §5.1.3** (distance from the requester's HRP to a Totem **Model pivot** of its map ≤ `Progression.TotemRangeStuds` (12), plus the swap gate `canTravel`) and applies it now through Part 3's random safe spawn on the same map (DES-3 pending); the builder sets `TotemRadius` from `Progression.TotemRangeStuds` for the ProximityPrompt only; `Wish.Pity` key names are read by the Wish screen. *(Part 5)*
9. **DevCmd additions:** `breakAll(mapId)`, `rebuildAll(mapId)`, `setLightingPhase(phase)`. *(Parts 2, 7)*
10. **Settings keys** (Part 2 §2.8.6 rows, `Types.Settings`, `SETTING_KEYS`): `quietBarks` (boolean, false, GDD §14.5; SfxController), `damageNumbers` (boolean, false, GDD §15.3; RenderController; replaces `Perf.Ui.DamageNumbersInTraining` with `Perf.Ui.DamageNumbersDefault = false`), `holdToToggleRun` (boolean, false, GDD §16.3; InputController latches `RunAuto`), and `binds.<Action>.<Device>` with `<Device>` ∈ {`KBM`, `Pad`} (SettingsController applies it to `Player.Combat.<Action>.<Device>.KeyCode`; single owner). Title selection: ProgressionService auto-equips the first granted title when `account.title == ""` (Part 5 §5.5.5); there is no `title` settings key or picker in M1-G (§6.15 future hook). *(Parts 2, 3, 5)*
11. **Hurtbox.prepare** additionally sets `Humanoid.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.None`. *(Part 3)*
12. **No hazards in M1-G maps;** the `Hazard*` attribute schema and limits are reserved and verified. *(Part 7)*
13. **`UiController.open(screen, args?)`** optional args; screen module contract (§6.9.1); lazy screen requires; open screens, the HUD and nameplates re-render on `Skin.onChanged` (Part 2 §2.9.2; `Skin.activePack()`). The Report flow is a Settings sub-panel (`open("Settings", {tab = "safety", reportUserId})`), so `Types.ScreenId` is unchanged. *(Part 2)*
14. **Client-local root `Workspace.ALG_Local`** (`Vfx`, `Looks`, `Ragdolls`, `Debris`, `Sfx`, and the `HapticEffect` `ALG_Haptic`); `Verify` flags it in Edit. **`ReplicatedStorage.Looks`** contents: `LookRig_R15`, `Face_UI`, `Gear_*`, `Summon_*`, `Oosona_*`. **`ServerStorage.Templates.Fragments`** cache. **Lighting children** `ALG_Atmosphere`, `ALG_Sky`, `ALG_CC`, `ALG_CC_Fx`, `ALG_Bloom`, and `Terrain.ALG_Clouds`. *(Parts 1, 2, 7)*
15. **Asset keys** (Part 8 adds manifest entries): generic `vfx.*` (§6.6.7), generic `sfx.*` (§6.10), `anim.common.*` (19, §6.6.8), `model.oosona.species_<id>`, `model.oosona.hat_<n>`, `model.oosona.outfit_<n>`; new VFX recipes `VFX_BlockSpark`, `VFX_PerfectStar`, `VFX_Breakaway_Burst`, `VFX_Splat`, `VFX_Tell_Ring`, plus `_Lod1` variants; new poses `hurt_light`, `hurt_heavy`, `tumble`, `splat`, `dizzy`, `ko_lie`, `getup_kneel`. *(Parts 4, 8)*
16. **Skin keys** (Part 8 fills `Codename` and `Original`): `map.<mapId>.name`, `map.candy_kingdom.zone.<zoneId>` (6), `ui.hud.*` (`breakaway`, `protected`, `combo`, `combo_big`), `ui.feed.bot`, `ui.nameplate.rookie`, `ui.totem.action`, `ui.totem.object`, `ui.wheel.*` (`in_dev`, `trial`, `free`, `get_hint`), `ui.wish.*`, `ui.shop.*` (incl. `goose_line_<n>`), `ui.store.*` (incl. `confirm_4th`), `ui.settings.*` (incl. `layout_editor_soon`), `ui.results.*` (incl. `environment`), `ui.training.*` (`slot`, `mode_<mode>`, `m1_<sub>`, `ender_<sub>`, `dash_<sub>`), `ui.toast.*` (`emotes_soon`, `key_in_use`), `ui.tip.*` (5), `ui.vo.*` (`awaken`, `ko`, `legendary`), `ui.oosona.species.<id>`. **Added by decision D-42** (Part 8 §8.3.7 rows; Codename and Canon inherit): `ui.hud.level`, `ui.hud.treasure`, `ui.hud.shards`, `ui.hud.sparks`, `ui.hud.quests`, `ui.quest.<id>` (token `{goal}`), `ui.notify.quest_done`, `ui.report.button`, `ui.report.title`, `ui.report.reason_exploit` / `_teaming` / `_harassment` / `_other`, `ui.report.send`, `ui.settings.tab_safety`, `ui.settings.quiet_barks`, `ui.settings.damage_numbers`, `ui.settings.hold_toggle_run`, `ui.training.damage_numbers`, `ui.wheel.shards`, `ui.wheel.convert`, `ui.wheel.mastery_combat_only`, `ui.wheel.variant`, `ui.wheel.variant_base`, `ui.wish.locked_paid`, `ui.wish.auto10`, `ui.wish.secret`, `ui.shop.balance`, `ui.shop.confirm_in_dev`, `ui.store.spent_week`. Existing keys now used: `ui.wish.fakeout_note` (no tokens), `ui.wheel.in_dev` (Shop cards), `ui.shop.coming_soon`, `ui.notify.report_thanks`, `variant.<styleId>.<vid>.name`. *(Part 8)*
17. **Config:** complete `Defaults.Perf` (§6.11) plus the `Schema.rules` additions; `Perf.Ui.DamageNumbersDefault = false` replaces `DamageNumbersInTraining`; `Perf.World.Hazard` has no `KoCreditSec`. `Defaults.Appearance` is Part 5 §5.10 (merged; D-13). *(Parts 2, 5, 7)*
18. **Log tags** added to Part 2's list: `World`, `Anim`, `Sfx`, `Cam`, `Look` (`Render`, `Vfx`, `UI` already exist). *(Part 2)*
19. **Studio snippet** `tools/studio/map_stats.luau` (§6.1.9). *(Parts 1, 7)*
20. **Theme is load-pure** and Lune-tested (`tests/core/theme.spec.luau`); new Lune specs `movement_volumes.spec.luau`, `perf.spec.luau`, `upload_anims.spec.luau`. *(Part 7)*
21. **Map roots under the `Workspace.Map` container** (decision D-01): `BuildMap_CandyKingdom` and `BuildTraining` create their Model roots with `Util.ensureRootOf(Util.ensureContainer(workspace, "Map", c), "<Name>", "Model", NAME, c, opts.reset)`; the container belongs to `BuildAll`'s containers step and is never pruned or reset (§6.1.1). *(Part 1)*
22. **`Terrain.Decoration` is human-set at H-04** (NotScriptable); no builder writes or reads it; `verify()` prints a non-failing reminder (§6.1.6). *(Part 1)*
23. **Animation upload pipeline (H-07 route)** §6.6.9 (decision D-09): snippet `tools/studio/export_anim.luau`, Lune `tools/upload_anims.luau` and `tools/base64.luau`, outputs under `build/anims/`, spike S-P6-14, conditional card T143. Part 1's example permissions file keeps `lune run tools/upload_anims.luau` on **ask** (every run human-approved). *(Parts 1, 7; TASKS)*
24. **Client perf sample sender** (decision D-17): VfxController sends `Telemetry.Perf` while `LocalPlayer:GetAttribute("PerfSampled") == true`, using `Telemetry.PerfWindowSec` / `PerfSendEverySec` (§6.6.3). *(Parts 2, 5)*
25. **Attached VFX are un-anchored and massless** before welding to a character (§6.6.3); the Style Tell ring follows the same rule (§6.8.4). *(Parts 3, 4)*
26. **Render contract** (decision D-29, Part 3 §3.11.8): entry = `St` or `Sub` changed; one-shots keyed by `(actionId, threshold)` with `firedMax`; telegraph only while `Cnt == 0` and before the first active frame; contact undo only with unchanged `actionId` or a rollback to ≤ the contact frame; predicted effects also undone on a STUN/KO entry within 15f; remote views use observation time `eR` (§6.6.1). *(Part 3)*
27. **`Theme.stylePalette(styleId, variantId?)`** (canonical code in Part 8 §8.2.7, reproduced in §6.9.4): variants `v1` (swap secondary/accent) and `v2` (darken 25 %), and `Features.CodenameLooks == false` forces the Original pack's palette (`Skin.look(styleId, "Original")`). `Variant` is an appearance refresh trigger (§6.8.3). *(Parts 4, 8)*
28. **`Features.CodenameLooks` behaviour** (decision D-43): hide Codename-look gear and summons, neutral Ball summons, Character → Simplified, Original palettes (§6.8.2). *(Parts 2, 8)*
29. **FTUE-lite → `Ftue.Step` mapping** (2–7) and the **no-offer-after-loss** rule (`Economy.NoOfferAfterLossSec`) in UiController (§6.9.10; decision D-40). *(Parts 5, 7)*
30. **Report UI** (§6.9.12): Results-card and Settings ▸ Safety entry points to `Report.Submit`. *(Parts 2, 5)*
31. **Per-move kill switch, HUD side** (decision D-14): the hotbar/cluster lock glyph for move ids in `Runtime.get().Features.DisabledMoves` (cosmetic; the sim refuses via `StyleRuntime.disabled`). *(Part 3)*
32. **HUD meta elements**: Wallet & level, Quests (on in M1-G, decision D-39), Shop Treasure header, Store weekly spend from `StoreView.spentThisWeekRobux`, Variant/Shard UI in the Style Wheel, Quick Wish and PRI-restricted wallet in the Wish screen (§6.9.5–§6.9.8). *(Part 5)*

### 6.14 Issues with spine

1. **Config keys.** §0.9 gives Part 6 only `Perf` ("VFX budgets per device tier"). Presentation and world tuning (camera, UI timings, lighting presets, spring speeds, break HP) also need config homes, because Part 2 forbids magic numbers in code. This part puts them in `Perf.*` sub-sections. Suggested spine fix: add top-level `Presentation` and `World` keys.
2. **No world service.** The fixed service list has no home for the server map runtime (void kills, breakables). This part specifies it as a block inside Part 3's `CombatService.luau`, and the predicted volume logic inside Part 3's `Movement`/`Adapter`. Suggested spine fix: add `Services/WorldService.luau` (Part 2's ORDER would become 16, after CombatService).
3. **"Style Tell always on" vs stealth.** GDD §7.3.12 stealth (and Part 4 §4.7.6) hides an invisible player from distant enemies; a permanent nameplate chip would defeat it. This part follows Part 4's proposal (the chip hides; the shimmer keeps the style colour). The spine wording should say "always on except during stealth, where the shimmer carries the style colour".
4. **Flat controller files and the 700-line cap.** §0.3 fixes the client file list, so the ragdoll presenter lives in `RenderController.luau`, the face controller and nameplates in `AppearanceController.luau`, telegraphs and the Highlight pool in `VfxController.luau`, and the Training panels in `UI/Screens/Hud.luau`. These will likely exceed Part 2's 700-line cap. Suggested fix: allow `Controllers/<Name>/init.luau` with child modules for these four, or raise their cap to 1,100 lines.
5. **Marker vs data timing** (resolved by decision D-03; kept for the rationale). Part 4 §4.2.0.7 originally fired move VFX/SFX from clip markers with a frame-counter fallback. This part (and the brief) uses data-driven frame timing as the primary path: `GetMarkerReachedSignal` is not Simulation-Access, temporary clip IDs do not exist outside Studio, and markers would fire twice during resimulation. Markers stay in the clips for the timing-lock check.
6. **Kit props in `ServerStorage.Templates`** (§0.3 table) cannot be seen by clients. This part follows Part 2 §2.13 item 3 (`ReplicatedStorage.Looks`).
7. **Highlight limit.** GDD §12.4 says the client renders at most 31 Highlights; the docs say 255 and that disabled ones still count [Docs `Highlight.yaml`]. The pooled cap of 12 (GDD) is kept; the GDD's 31 should be corrected.
8. **M0 map selection and travel.** The spine's Part 2 defaults assume Candy Kingdom exists; M0 has only the arena and Training (§6.13 item 6 asks Parts 2/3 for the fixes).
9. **Touch HUD: HP bar and Breakaway pips top-centre** (deviation from GDD §15.3 "bottom-center, above the hotbar"; GDD §15.1 "mobile defines the layout"). The bottom-centre band on touch is ≈ 150 reference units wide between the thumbstick zone and the cluster (§6.9.5). Needs owner sign-off at H-10; fallback layout in §6.9.5.
10. **Quick Wish vs first-view reveals.** GDD §17.3 sells "instant reveals"; GDD §8.6 says a rarity's reveal can be skipped only after its first view. This part keeps the first view of each rarity in full and makes every later reveal instant for pass owners (§6.9.7). Owner to confirm at H-10.

### 6.15 Open questions / spikes

| ID | Question | Method | Pass | Fallback |
|---|---|---|---|---|
| S-P6-1 | Does `Players:CreateHumanoidModelFromDescriptionAsync` run in Edit via `execute_luau` (DummyRig/BotRig/LookRig sizes)? Are the R15 `*RigAttachment` names as listed in §6.7.2 (with `AvatarJointUpgrade` on)? | Run `BuildTemplates`; list attachments | Rig with 15 parts and the 14 joint attachments | Human inserts an R15 rig via Rig Builder into `ServerStorage.Templates.__RigSeed`; read names from it |
| S-P6-2 | Do client-local, massless parts welded to the **local predicted** character (look proxies, gear, the Style Tell ring and attached VFX such as `vfx.awaken.aura`, `vfx.spawn.shimmer`, all un-anchored per §6.6.3) cause mispredictions or physics changes? | Two clients, 5 min of dashing, awakenings and ragdolls with proxies, ring and auras on; Misprediction log; scenario `vfx.attach_unanchored` | No entries attributable; dash distance error unchanged; `HRP.AssemblyRootPart == HRP` throughout | `BulkMoveTo` follow in the render step for proxies, ring and auras (§6.8.3, §6.8.4) |
| S-P6-3 | Can our `Jump` action's `Touch.UIButton` be bound to the default `JumpButton`, and does it still jump? | Touch emulator | Both the default jump and our `Jump` state fire | Own `JumpBtn`; hide the default button |
| S-P6-4 | Does an `AlwaysOnTop` SurfaceGui outline render above ParticleEmitters and Beams? | Telegraph inside a dense aura | Outline visible in a capture | Raise the fill part 0.3 studs; lower other VFX `ZOffset` |
| S-P6-5 | Is the in-sim spring launch (`ChangeState(Freefall)` + `AssemblyLinearVelocity`) predicted without rollback at 150 ms? | `world.spring_launch` under Network Simulator | No `St`/position correction > 1 stud | Launch via `ALG_Vel` mover for 6 frames (Part 3 mover path) |
| S-P6-6 | Do the built-in fonts render ○ ◇ △ ☆ ✦ ❂ ✺ ♥ ★ ✦? | Capture a label per glyph per font | All render (no tofu) | Frame-drawn shapes (§6.9.4) |
| S-P6-7 | Does a client-created `InputContext` under `LocalPlayer` fire `Pressed` under Server Authority? | Press Tab | Style Wheel opens | `UserInputService.InputBegan` for UI actions |
| S-P6-8 | With `DevEnableMouseLock = false`, does setting `MouseBehavior = LockCenter` at `Camera + 1` stick every frame? | KBM playtest | Cursor stays locked; `AimLock` true | Set it in `RenderStepped` at `Last` as well |
| S-P6-9 | Are client-created unanchored parts (ragdoll clones, debris) simulated locally under SA, and do BallSocket limits read as "noodle"? | `ragdoll.clone_lifecycle` + human review (H-10) | Clones fall and settle; limbs flop without breaking | Pose-only ragdoll (anchored, tumble hold) |
| S-P6-10 | GUI units on an SE-class device (points vs pixels) | H-12 device: log `Camera.ViewportSize` | Scale formula keeps 14 ref px ≥ 14 physical px | Adjust `RefSize` clamp |
| S-P6-11 | Server `CanCollide` toggles on breakable chunks near predicted characters: misprediction rate | Break houses next to a moving player | ≤ 1 correction per break | Delay collision removal by 6 frames after the visual break |
| S-P6-12 | Does `Font.fromName("ComicNeueAngular")` resolve? | Label test | Renders | `Enum.Font.Arimo` |
| S-P6-13 | Do pooled Highlights toggled via `Adornee`/`Enabled` avoid the add/remove hitch the GDD mentions? | Harness `PerfProbe` samples `Stats.RenderCPUFrameTime` [Docs `Stats.yaml`] every frame for 10 s idle (baseline) and 10 s while toggling 12 Highlights/s (agent-run; MicroProfiler is optional, for a human) | No frame more than 2 ms above the baseline's P95 | Keep Highlights adorned and change transparency only |
| S-P6-14 | Does the §6.6.9 export fit through `execute_luau`? (`SerializeInstancesAsync` from the plugin context; base64 size of the shortest and the longest existing clip vs the tool's result limit and the 2 KB payload cap) | In T143: `tools/studio/export_anim.luau` on the shortest and the longest existing clip (`anim.common.*`; `anim.hero.m1_1` does not exist before T174, which repeats the check on it); decode on the host; compare byte length with `size` | One chunk (or a fixed chunk length) round-trips byte-exactly | Human H-08 (Save to Roblox); T143 `skipped` |

Design questions:
- **Q1:** Spring speeds (101 / 130 studs/s), the river multiplier (0.6) and the boundary band (20 studs, ×0.2 at the wall) are tuning defaults for H-10.
- **Q2:** House HP 2 (first slam cracks, second breaks) keeps wall combos usable. Confirm with design.
- **Q3:** The Dungeon's only exits are the hatch springs (they break a rebuilt hatch from below). Confirm this matches "high-risk shortcut".
- **Q4:** (resolved) Damage numbers use the Part 2 settings key `damageNumbers` (off by default; Training panel and Accessibility toggles, §6.4.2, §6.9.11).
- **Q5:** The client mastery-level formula duplicates `ProgressionService.masteryLevel`. A pure shared home (e.g. a function in `Config/Schema.luau`) would remove the duplication; Part 5 decides.
- **Q6:** The rarity CVD palette (one Okabe–Ito mapping for all three modes) needs human review with simulators.
- **Q7:** Touch layout positions are designed for 1334 × 750; the layout editor and the one-hand preset (GDD §15.2, §16.3) are deferred past M1-G.

**Future hooks** (recorded decisions; not built in M1-G; owner sign-off at H-10):
- **Quick-Chat wheel** (GDD §16.2 "fully social without chat": non-free-text phrases such as "GG!"). No key binding is reserved in M1-G (`DPadDown` is the menu focus, §6.9.5); a later card adds a UI action, phrases as Skin keys and `TextChannel:SendAsync` on the default channel with a client-side rate limit.
- **Emote wheel** (the `Emote` action only shows `ui.toast.emotes_soon`).
- **Title picker** (GDD titles): M1-G auto-equips the first granted title (Part 5 §5.5.5) and has no `title` settings key; a picker in Settings ▸ Appearance is a later card.
- **Style Wheel move previews and tooltips** (GDD §16.3 learning aids): the detail pane lists moves by name and tags only.
- **Weekly quests and rerolls** (Part 5 §5.5.9): the HUD shows daily quests only.
- **Avatar toon post-process** (§6.8.2) and the touch layout editor / one-hand preset (Q7).
