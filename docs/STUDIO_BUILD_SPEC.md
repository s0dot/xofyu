# ALGEBRAIC! — Studio Build Spec

**How an AI coding agent builds the M0 feel prototype and the M1-G graybox vertical slice in Roblox Studio**

> ⚠️ **DRAFT — pre-review snapshot.** An editorial/technical review of this spec found ~200 issues and the fixed
> v1.0 is still being prepared. **Agents: do not execute any task card beyond the P0 human-setup steps (H-01…H-12)
> until this banner is gone.** Humans can start the one-time setup in Part 1 §1.2 now.

| | |
|---|---|
| **Spec version** | 0.9 draft |
| **Date** | 15 October 2026 |
| **Status** | Draft (pre-review snapshot) |
| **Design source** | [`docs/GDD.md`](GDD.md) v1.2 (all tuning numbers; this spec never changes them) |
| **Audience** | A coding agent (Claude Code) driving Roblox Studio through the built-in `Roblox_Studio` MCP server, plus the human operator for steps H-01…H-12 |

> **Start here.** Humans: do the one-time setup in §1.2 (Part 1) and the human steps in §0.7 below. Agents: read `AGENTS.md` at the repo root, then Part 0 (below) and Part 1, then take the next card in `docs/agent/TASKS.md`. A reference like "§3.4" means Part 3, section 3.4; "GDD §6.3" points into the GDD.

## Where each part lives

| Part | Title | File | Sections |
|---|---|---|---|
| 0 | Shared contracts (names, IDs, paths, phases, human steps) | this file, below | §0.x |
| 1 | Operating the agentic session | [`docs/build-spec/part-1-operating.md`](build-spec/part-1-operating.md) | §1.x |
| 2 | Architecture and contracts | [`docs/build-spec/part-2-architecture.md`](build-spec/part-2-architecture.md) | §2.x |
| 3 | Combat core and Server Authority integration | [`docs/build-spec/part-3-combat.md`](build-spec/part-3-combat.md) | §3.x |
| 4 | Style kit framework and vertical-slice kits | [`docs/build-spec/part-4-styles.md`](build-spec/part-4-styles.md) | §4.x |
| 5 | Meta systems | [`docs/build-spec/part-5-meta.md`](build-spec/part-5-meta.md) | §5.x |
| 6 | World, UI and presentation | [`docs/build-spec/part-6-world-ui.md`](build-spec/part-6-world-ui.md) | §6.x |
| 7 | Testing, verification and performance | [`docs/build-spec/part-7-testing.md`](build-spec/part-7-testing.md) | §7.x |
| 8 | IP and naming layer | [`docs/build-spec/part-8-ip.md`](build-spec/part-8-ip.md) | §8.x |

Companion files: [`docs/agent/TASKS.md`](agent/TASKS.md) (ordered task cards), [`AGENTS.md`](../AGENTS.md) (agent operating manual), [`docs/GDD.md`](GDD.md) (design source of truth).

---

## 0. Shared contracts

Every part uses the names, IDs, paths and decisions in this part **verbatim**. If another part appears to contradict Part 0, Part 0 wins: follow it and record a `spec-gap` blocker (§1.8.1).

### 0.0 Purpose and scope of the build spec

**Audience.** An AI coding agent (Claude Code or similar) driving **Roblox Studio** through Studio's **built-in MCP server**. Code lives on disk and is synced into Studio by **Script Sync**, following Roblox's official "coding harness". A human operator handles the steps the agent cannot do.

**Build target.** Two milestones, in order:
1. **M0 "Feel prototype"** (graybox):
   - 2 styles: `hero` and `stretch`
   - full shared combat chassis: M1 chain with Enders, block / Perfect Block / guard, 3 dashes, hitstun / Stun Decay, ragdoll state, Breakaway, Algebraic Meter plus Awakening for both styles
   - 1 graybox duel arena and the Training Grounds dummy
   - Server Authority netcode
   - PC + mobile touch + gamepad input
   - ≥ 30 fps on the low-end reference with 8 clients plus server bots
2. **M1-G "Graybox Vertical Slice"**:
   - 6 styles: `hero`, `stretch`, `monarch`, `rocker`, `frost`, `decay`
   - Candy Kingdom graybox map (20 players)
   - Style Wheel and Style Totem
   - Wish system end-to-end: odds, pity, Flairs, ledger, PolicyService gating, mock purchases
   - Choose Goose shop, Account Level and Style Mastery (data only plus simple UI), HUD
   - Character Look toggle and appearance system
   - Persistence via ProfileStore (mock in Studio)
   - Telemetry wrapper
   - One Realm (`decay` A4 "The Well of Power"), without Realm Clash

**Out of scope for this spec:** final art, audio and animation (placeholders only), ranked/matchmaking, raids, seasons, Kingdoms, Adventure Pass, live-ops tooling, Realm Clash, Bond Moves, world events other than one stub. Each part may list "future hooks".

**Design source of truth.** `docs/GDD.md` (v1.1+) holds all tuning numbers (frame data, odds, prices). The spec **must not change GDD numbers**. Encode them in `Shared/Config/Defaults.luau` and cite them as "GDD §x.y".

### 0.1 Deliverable files

**Spec parts.** All parts are assembled into this file (`docs/STUDIO_BUILD_SPEC.md`).

| Part | Title | Source file | Section numbers |
|---|---|---|---|
| 1 | Operating the agentic session | `P1_operating.md` | §1.x |
| 2 | Architecture and contracts | `P2_architecture.md` | §2.x |
| 3 | Combat core and Server Authority integration | `P3_combat.md` | §3.x |
| 4 | Style kit framework and vertical-slice kits | `P4_styles.md` | §4.x |
| 5 | Meta systems | `P5_meta.md` | §5.x |
| 6 | World, UI and presentation | `P6_world_ui.md` | §6.x |
| 7 | Testing, verification and performance | `P7_testing.md` | §7.x |
| 8 | IP and naming layer | `P8_ip.md` | §8.x |

- **Task plan:** `TASKS.md`, which ships as `docs/agent/TASKS.md`.
- **Root agent files** (they ship at repo root): `AGENTS.md`, `CLAUDE.md`, `ARCHITECTURE.md`.
- **Skills:**
  - `skills/algebraic-conventions.md` and `skills/algebraic-style-kit.md`, in Roblox Assistant skill format (frontmatter `name` / `description` / `enabled`). They ship to `docs/agent/skills/`.
  - A Claude Code mirror at `.claude/skills/algebraic-studio/SKILL.md`.
- **MCP config example:** `mcp.example.json`, shipping to `docs/agent/mcp.example.json`.

The assembled spec ships as `docs/STUDIO_BUILD_SPEC.md`. References into the GDD look like "GDD §6.3". References into the spec look like "§3.4".

### 0.2 Toolchain decision (fixed)

- **Primary harness: Roblox's official coding harness.**
  - Script Sync (disk ↔ Studio, scripts only), an external agent (Claude Code), and the built-in Studio MCP server (instances, playtests, console, screenshots, input simulation).
  - The MCP server is named **`Roblox_Studio`**.
  - The agent discovers tools via `tools/list` at session start and **does not hard-code tool names without a fallback**.
  - Known names (Aug 2026 docs): `list_roblox_studios`, `execute_luau` (with `datamodel_type` Edit/Server/Client), `start_stop_play`, `get_studio_state`, `get_console_output`, `screen_capture`, `user_keyboard_input`, `user_mouse_input`, `character_navigation`, `search_game_tree`, `inspect_instance`, `script_read`, `multi_edit`, `script_search`, `script_grep`, `insert_asset`, `search_asset`, `generate_mesh`, `generate_material`, `generate_procedural_model`, `wait_job_finished`, `upload_image`, `store_image`, `subagent`, `http_get`, `skill`. Every call takes a `studio_id`.
- **Rojo** is optional and used only for CI `rojo build`. The disk layout below is compatible with both Script Sync and Rojo.
- **Lune** runs the pure-core unit tests and lints on the host, with no Roblox needed. Other tools: **StyLua**, **Selene** and **luau-lsp**, managed via **Rokit** (`rokit.toml`).
- **No frameworks.** Plain typed ModuleScript services with `init()` / `start()`. **No Wally in M0/M1-G.** Third-party MIT code is **vendored** into `Vendor/` folders with an attribution header and recorded in `docs/THIRD_PARTY.md`. Vendored files: ProfileStore (MadStudio, MIT), Trove, Signal and Shake (Sleitnick RbxUtil, MIT).
- **No Blink/Zap in M0/M1-G.** Use a typed RemoteEvent wrapper (`Shared/Net`).

### 0.3 Script Sync roots and disk layout (fixed)

**Sync rules.**
- Only folders that contain **scripts only** are synced. Script Sync ignores other instances and warns against mixing them.
- Non-script instances live in **unsynced sibling folders** created by idempotent **builder modules**.
- Scripts must not carry attributes or tags; they are not synced.

| Studio path (synced) | Disk folder |
|---|---|
| `ReplicatedStorage.Shared` | `src/Shared/` |
| `ServerScriptService` | `src/ServerScriptService/` |
| `ServerStorage.DevTools` | `src/DevTools/` |
| `StarterPlayer.StarterPlayerScripts` | `src/StarterPlayerScripts/` |

**Unsynced, builder-created instance folders.** These are never edited by hand.

| Folder | Contents |
|---|---|
| `ReplicatedStorage.Inputs` | InputContext templates |
| `ReplicatedStorage.VFX` | VFX templates |
| `ReplicatedStorage.Anim` | Animation instances and placeholder KeyframeSequences |
| `ReplicatedStorage.Remotes` | Created at runtime by the server |
| `ReplicatedStorage.Runtime` | Runtime config StringValue, created at runtime |
| `ServerStorage.Templates` | Dummy rig, kit props |
| `Workspace.Map` | Built map |
| `Workspace.Arena` | M0 duel arena |
| `Lighting` | Children set by the lighting builder |

**File naming.** Script Sync and Rojo share these conventions:
- `Name.luau` → ModuleScript
- `Name.server.luau` → Script (RunContext Server)
- `Name.client.luau` → Script (RunContext Client)
- `Folder/init.luau` → ModuleScript with children

**Disk layout** (exact):

```
src/
  Shared/
    Types.luau            Log.luau            Assets.luau
    Config/   Defaults.luau  Schema.luau  Runtime.luau
    Core/     (PURE: no Instance/Roblox-service access; only plain tables, numbers, Vector3/CFrame datatypes allowed; Lune-testable)
              Fsm.luau  Frames.luau  M1.luau  Defense.luau  Movement.luau  Stun.luau  Meter.luau  Breakaway.luau
              Moves.luau  Damage.luau  Rng.luau  Realm.luau
    Sim/      Simulation.luau  Attr.luau  InputMap.luau  Adapter.luau
    Combat/   Query.luau  Rewind.luau  Hurtbox.luau
    Styles/   init.luau  hero.luau  stretch.luau  monarch.luau  rocker.luau  frost.luau  decay.luau
    Behaviors/ Projectile.luau  Rush.luau  Grab.luau  Counter.luau  Zone.luau  Summon.luau  Buff.luau  Mobility.luau  RealmCast.luau
    Skin/     init.luau  Packs/Codename.luau  Packs/Original.luau  Packs/Canon.luau
    Wish/     Odds.luau  Pity.luau   (PURE; shared so the UI shows server-identical odds)
    Net/      init.luau  Schemas.luau
    Vendor/   Trove.luau  Signal.luau  Shake.luau
  ServerScriptService/
    Main.server.luau
    Services/ DataService.luau  PlayerService.luau  StyleService.luau  CombatService.luau  ConfigReplicator.luau
              WishService.luau  ShopService.luau  ProgressionService.luau  MonetizationService.luau  PolicyGate.luau
              Telemetry.luau  ModerationService.luau  BotService.luau  AppearanceService.luau  TestHarness.luau
    Vendor/   ProfileStore.luau
  DevTools/
    Builders/ BuildAll.luau  BuildInputs.luau  BuildArena.luau  BuildMap_CandyKingdom.luau  BuildTraining.luau
              BuildLighting.luau  BuildVfxTemplates.luau  BuildPlaceholderAnims.luau  BuildTemplates.luau  Verify.luau
    Tests/    (in-Studio scenario modules; see Part 7)
  StarterPlayerScripts/
    Main.client.luau
    Controllers/ InputController.luau  CameraController.luau  RenderController.luau  AnimController.luau  VfxController.luau
                 SfxController.luau  HudController.luau  UiController.luau  AppearanceController.luau  SettingsController.luau
    UI/ Create.luau  Theme.luau  Screens/ Hud.luau  StyleWheel.luau  Wish.luau  Shop.luau  Settings.luau  Results.luau
tests/   (Lune) run.luau  core/*.spec.luau  wish/*.spec.luau  shim/ (Roblox datatype shims if needed)
tools/   lint_strings.luau  check_manifest.luau  export_framedata.luau  (Lune)
         studio/ (tiny execute_luau snippets, ≤ 2 KB each)
         plugin/AlgebraicTestHarness/ (local Studio plugin source for multi-client tests)
docs/    GDD.md  STUDIO_BUILD_SPEC.md  THIRD_PARTY.md  agent/TASKS.md  agent/skills/  agent/mcp.example.json  reference/
AGENTS.md  CLAUDE.md  ARCHITECTURE.md  README.md  rokit.toml  selene.toml  stylua.toml  .luaurc  default.project.json
```

**Builder invocation pattern** (fixed; keeps MCP payloads tiny and bypasses the require cache):

```lua
-- run via execute_luau with datamodel_type = "Edit"
local m = game.ServerStorage.DevTools.Builders.BuildArena:Clone()
local ok, res = pcall(function() return require(m).build({ reset = true }) end)
m:Destroy()
return game:GetService("HttpService"):JSONEncode({ ok = ok, res = res })
```

Every builder exposes these and returns a JSON-able summary table:
- `build(opts) -> {created = n, updated = n, warnings = {...}}`
- `verify() -> {ok = bool, problems = {...}}`

Builders are **idempotent**: re-running produces the same tree.

### 0.4 Runtime conventions (fixed)

- **`--!strict` in every file.**
  - No `_G` and no `shared`.
  - No `wait` / `spawn` / `delay`; use `task.*`.
  - No deprecated APIs: `PhysicsService` collision APIs (use `workspace:RegisterCollisionGroup`), `KeyframeSequenceProvider` (use `AnimationClipProvider`), `MarketplaceService:GetProductInfo` (use `GetProductInfoAsync`), `InputAction:Fire` (use `InputBinding:Fire` with a Scriptable binding), BodyMovers.
- **Bootstrap.**
  - `Main.server.luau` requires every module in `Services/`, calls `init()` on each in the fixed order listed in Part 2, then `start()`.
  - `Main.client.luau` does the same for `Controllers/`.
- **Logging.**
  - `Log.info/warn/error(tag, msg, data?)` prints `[ALG][<tag>] msg`.
  - Test lines are exactly `[ALG][TEST] <name> PASS` or `[ALG][TEST] <name> FAIL <detail>`, plus a final `[ALG][TEST] SUMMARY pass=<n> fail=<m>`.
- **Config.**
  - `Shared/Config/Defaults.luau` holds every tunable from the GDD.
  - `ConfigReplicator` (server) overlays Roblox `ConfigService` values (server-only API) and publishes a frozen, versioned JSON snapshot to `ReplicatedStorage.Runtime.Config` (StringValue + attribute `Version`).
  - `Config/Runtime.luau` gives both peers `Runtime.get(): ConfigTable` and `Runtime.version()`.
  - Combat config changes take effect only at respawn/round boundaries.
  - Pure Core functions take config as a parameter. They never read globals.
- **Time.**
  - Inside the simulation, only `time()` and the step `dt`.
  - Outside the simulation, `os.clock()` is allowed for non-outcome timing.
  - `workspace:GetServerTimeNow()` is used for meta systems (shop rotation, daily resets) on the server.
- **Frame units.**
  - Frame data is authored in **frames at 60 Hz** (GDD §6.1).
  - Core converts with `Frames.toSeconds(f) = f / 60`.
  - The simulation step **must** be bound with `Enum.StepFrequency.Hz60`; the API default is Hz30.
- **Net.**
  - The server creates RemoteEvents/RemoteFunctions under `ReplicatedStorage.Remotes` at boot, from `Net/Schemas.luau`.
  - Every client→server message is schema-validated and rate-limited (leaky bucket) server-side.
  - **Simulation input never uses remotes.** It goes through the InputAction System.
- **Server Authority.**
  - `Workspace.AuthorityMode = Server` is set **by the human in the Properties window**; it is not script-settable. That also turns on NextGenerationReplication, IAS for PlayerScripts, Deferred signals, UseFixedSimulation and StreamingEnabled.
  - `Config.NetModel ∈ {"ServerAuthority", "Classic"}` selects the adapter. Classic is the fallback if M0 spikes fail.
  - The pure Core is identical in both modes.
- **Attributes.**
  - Simulation state lives in attributes on each character's `HumanoidRootPart`, written **only** inside BindToSimulation callbacks.
  - Limits: names ≤ 50 chars, string values ≤ 50 chars, ≤ 64 replicated attributes per instance.
  - The canonical list lives in `Shared/Sim/Attr.luau` (Part 3 defines it).
  - Non-simulation, server-owned stats such as HP-for-UI mirror, Meter, Wishes and currencies live as attributes on the `Player` instance or in server memory (Part 2 and Part 3 define which).
- **IDs.**
  - **Style IDs** (IP-neutral, fixed):

    | ID | Style | ID | Style | ID | Style |
    |---|---|---|---|---|---|
    | `hero` | Finn | `stretch` | Jake | `sentry` | Banana Guard |
    | `granny` | Tree Trunks | `diva` | Lumpy Space Princess | `tyrant` | Earl of Lemongrab |
    | `console` | BMO | `grappler` | Susan Strong | `monarch` | Princess Bubblegum |
    | `flame` | Flame Princess | `butler` | Peppermint Butler | `rocker` | Marceline |
    | `frost` | Ice King | `huntress` | Huntress Wizard | `decay` | The Lich |
    | `demon` | Hunson Abadeer | `wish` | Prismo | `chaos` | GOLB |

  - **Move IDs:** `<style>.m1`, `<style>.1` … `<style>.4`, `<style>.q` (Quirk), `<style>.a1` … `<style>.a4`, `<style>.passive`, `<style>.awaken`.
  - **Map IDs:** `arena_duel_m0`, `candy_kingdom`, `training`.
  - **Flair IDs:** `standard`, `shiny`, `golden`, `crystal`, `rainbow`, `nightosphere`, `glitched`.
  - **Rarity IDs:** `starter`, `common`, `uncommon`, `rare`, `epic`, `legendary`, `mythic`, `cosmic`.
- **Skin packs (naming layer).**
  - All player-visible strings come from `Skin.text(key)` / `Skin.style(id)`.
  - Packs:
    - `Codename`: JJS-style epithets from GDD §7.2 plus GDD move names. **This is the dev default**, per the owner's chosen approach.
    - `Original`: IP-neutral archetype names.
    - `Canon`: canonical names. **Locked**: only allowed when `Config.LicenseConfirmed = true`.
  - Key format: `style.<id>.name`, `style.<id>.epithet`, `move.<moveId>.name`, `map.<mapId>.name`, `ui.<screen>.<key>`, `flair.<id>.name`, `rarity.<id>.name`.
  - `Config.SkinPack` selects the active pack. Part 8 defines the publish gate.
- **Asset manifest.**
  - `Shared/Assets.luau` maps logical keys to `{id: string, status: "placeholder" | "final", source: "builtin" | "agent" | "ai" | "creator-store" | "artist", kind: "image" | "audio" | "anim" | "mesh" | "model" | "vfx", note: string?}`.
  - Code calls `Assets.get(key)`, never a raw asset ID.
  - Per-universe product IDs live under `Assets.products[gameId]`.
- **Appearance (owner's change).**
  - Per-player settings, saved in profile:
    - `CharacterLook: boolean` (default true)
    - `OwnLook: "Oosona" | "Avatar"` (default "Oosona")
    - `ShowStyleGear: boolean` (default true)
  - Viewer-local setting `OthersAppearance: "AsChosen" | "AlwaysCharacter" | "AlwaysOwn" | "Simplified"`.
  - The **Style Tell** (nameplate chip in the style colour, style VFX palette) is always on.
  - The hurtbox is a fixed capsule in every mode (GDD §6.2).
  - Graybox rendering of each mode is defined in Part 6; data and replication in Part 5.

### 0.5 Placeholder policy (fixed)

- **The agent builds everything as graybox.** Parts, CSG, Terrain, built-in fonts, built-in particle textures, untextured Beams/Trails, and the 11 built-in sounds.
- **AI generation tools** (`generate_mesh` / `generate_procedural_model` / `generate_material`) are used **only** for generic props. Never for characters, signature props, or anything prompted with show names. Tag results `source = "ai"`.
- **Creator Store insertion** is limited to non-script assets (meshes, audio, decals) that pass the vetting checklist in Part 1. Models containing scripts are never inserted.
- **Placeholder animations:**
  - Authored as KeyframeSequences by a builder from the frame data.
  - For Studio-only iteration, registered via `AnimationClipProvider:RegisterAnimationClip`. Temporary IDs work only in the DataModel that registered them, and only in Studio.
  - For multi-peer or Server Authority playtests, they must be **published by the human** (or via Open Cloud with a human-created group API key). Their IDs go into `Assets.luau`.
- **Releases:** a release build fails if any `placeholder` asset is referenced (Part 7 lint).

### 0.6 Phases and task-ID ranges for TASKS.md (fixed)

| Phase | Name | Task IDs |
|---|---|---|
| P0 | Session setup and platform spikes | T000–T019 |
| P1 | Foundations (skeleton, log, config, net, data mock, assets, skin, bootstrap, test harness) | T020–T049 |
| P2 | Pure combat core + Lune tests | T050–T089 |
| P3 | Server Authority integration (inputs, sim adapter, hit resolution, rewind, damage/KO/respawn) | T090–T129 |
| P4 | Presentation (placeholder anims, render/VFX/SFX/camera, HUD, appearance) | T130–T159 |
| P5 | Style framework and kits (M0: hero, stretch; M1-G: + monarch, rocker, frost, decay incl. Realm) | T160–T199 |
| P6 | World (arena, Candy Kingdom graybox, training, totems, lighting) and UI screens | T200–T229 |
| P7 | Meta systems (styles ownership, Wish, shop, progression, policy, mock monetization, telemetry) | T230–T269 |
| P8 | Hardening (bots and load test, perf pass, anti-exploit, publish gate, docs/ARCHITECTURE update) | T270–T299 |

**Milestone gates:**
- **M0 gate:** after P5's `hero`/`stretch` tasks plus P6's arena tasks.
- **M1-G gate:** at the end of P8.

**Task card format** (TASKS.md):

```
### T0xx — <imperative title>
- Phase / milestone:
- Depends on:
- Spec refs: §… / GDD §…
- Files & instances: (create/modify)
- Steps: (numbered, concrete)
- Acceptance criteria: (observable, testable)
- Verify: (exact commands / MCP calls / test names; expected log lines)
- Human checkpoint: (none | what the human must do/approve)
```

### 0.7 Human-only steps (fixed list)

Part 1 expands these into a checklist; tasks reference them as `H-xx`.

| ID | Step |
|---|---|
| H-01 | Create the dev universe and place (group-owned); save the place to a file inside the repo folder (gitignored) |
| H-02 | Enable Studio MCP server (Assistant → ⋯ → Manage MCP Servers) and connect Claude Code |
| H-03 | Enable Script Sync and sync the four roots |
| H-04 | Set `Workspace.AuthorityMode = Server` and `StreamingEnabled`; set `StarterPlayer.AvatarJointUpgrade = Enabled` in Properties |
| H-05 | Avatar Settings: Consistent Gameplay preset + Single Collider; enable the CCL beta, if adopted after spike |
| H-06 | Game Settings → Security: Enable Studio Access to API Services (dev universe only) |
| H-07 | Create a group-owned Open Cloud API key (asset:read/write) for placeholder animation uploads; store it outside the repo |
| H-08 | Publish placeholder animations (or approve the agent upload pipeline) |
| H-09 | Create dev-universe developer products (not for sale) or approve the Open Cloud script; prices need human sign-off |
| H-10 | Feel playtests at each combat gate |
| H-11 | Publish to the dev place / approve any public publish (the publish gate, Part 8) |
| H-12 | Mobile device test (low-end Android reference) |

### 0.8 Canonical data types (fixed; Part 2 places them in `Shared/Types.luau`, Part 4 uses them)

```lua
export type Rarity = "starter" | "common" | "uncommon" | "rare" | "epic" | "legendary" | "mythic" | "cosmic"
export type Tag = "Projectile" | "Rush" | "Grab" | "Counter" | "Zone" | "Summon" | "Buff" | "Debuff" | "Mobility"
	| "AoE" | "Utility" | "Stealth" | "Ultimate" | "Realm" | "Music" | "Demon" | "Opener" | "Extender"
export type Behavior = "Strike" | "Projectile" | "Rush" | "Grab" | "Counter" | "Zone" | "Summon" | "Buff" | "Mobility" | "RealmCast"
export type HitboxDef = { shape: "Box" | "Sphere" | "Cast", size: Vector3, offset: CFrame, castDistance: number? }
export type KnockbackDef = { dist: number, angleDeg: number, ragdoll: boolean, launch: boolean? }
export type MoveDef = {
	id: string,                       -- e.g. "hero.3"
	slot: "m1" | "1" | "2" | "3" | "4" | "q" | "a1" | "a2" | "a3" | "a4",
	behavior: Behavior,
	tags: { Tag },
	signature: boolean?,              -- exactly one base move per style has signature = true (GDD §6.7)
	startup: number, active: number, recovery: number,   -- frames @ 60 Hz
	damage: number, guardDamage: number, chip: number,   -- chip as fraction (0.2 = 20%)
	hitstun: number,                  -- frames
	knockback: KnockbackDef?,
	hitbox: HitboxDef?,
	cooldown: number,                 -- seconds
	parryable: boolean,
	superArmor: boolean?, projectileArmor: boolean?,
	telegraph: { shape: "Circle" | "Line" | "Cone" | "Arc", leadFrames: number, size: Vector3 }?,
	params: { [string]: any }?,       -- behavior-specific params; typed per behavior in Part 4
	meterGain: number?,               -- multiplier on damage→meter (default 1.0)
	anim: string,                     -- logical asset key (Assets.get)
	vfx: string?, sfx: string?,       -- logical asset keys
}
export type AwakeningDef = {
	id: string, durationSec: number,  -- 60 (GDD §6.9)
	hurtboxScale: number?,            -- ≥ 1 only (GDD §6.2)
	modifiers: { [string]: any }?,    -- awakened passive modifiers (only where the kit says so)
	moves: { MoveDef },               -- a1..a4 (a4 may be behavior "RealmCast" for legendary+)
}
export type StyleDef = {
	id: string, rarity: Rarity, archetype: string, implemented: boolean,
	hp: number,                       -- 100 at launch (GDD §6.2)
	palette: { primary: Color3, secondary: Color3, accent: Color3 },
	passive: { id: string, params: { [string]: any } },
	quirk: MoveDef,
	moves: { MoveDef },               -- slots "1".."4"
	awakening: AwakeningDef,
	m1: { anim: string, vfx: string?, sfx: string? },  -- M1 frame data is shared (GDD §6.3); only presentation differs
	kitProps: { string }?,            -- logical template keys for Style Gear props
}
```

### 0.9 Config top-level structure (fixed keys; each owning part fills its section)

```
Defaults = {
  Version: number,
  NetModel: "ServerAuthority" | "Classic",
  SkinPack: "Codename" | "Original" | "Canon",
  LicenseConfirmed: boolean,             -- Part 8
  PublicCodenameAcknowledged: boolean,   -- Part 8
  MockPurchases: boolean,                -- Part 5
  Combat = { Chassis, M1, Block, Dash, Stun, Ragdoll, Breakaway, Meter, Awakening, Realm, Damage, Respawn, PileOn, Streak },  -- Part 3
  Wish = { TierOdds, Pity, Flair, LuckyJarFlairMultiplier, StorageCap, OverflowTreasure, PoolMode, DuplicateShards, VariantChance, CosmicSparksNeeded, FirstWishGuarantee },  -- Part 5
  Shop = {...}, Economy = {...}, Progression = {...}, Appearance = {...}, Telemetry = {...},  -- Part 5
  Perf = {...},                           -- Part 6 (VFX budgets per device tier)
}
```

### 0.10 Wish pool rule for M1-G (fixed)

- The Wish pool uses the **full 16-style rollable roster and the odds from GDD §8.3–§8.4**: `Wish.PoolMode = "Full"`.
- Styles with `implemented = false` are **collectible but not equippable** ("In development" badge). This keeps the GDD odds and pity math intact while only 6 kits exist.
- The two **Starter** styles (`hero`, `stretch`) are granted to every player and never appear in the pool.

### 0.11 Simulation-verified economy numbers (fixed; from a 4,000-run Monte Carlo of the GDD §8.4 pity rules at 14 Wishes/day)

| Milestone | Mean day | Median day | Worst / P95 |
|---|---|---|---|
| First Epic+ | 1.4 | 1 | worst 3 |
| First Legendary+ | 3.2 | 3 | worst 9 (hard pity 120) |
| First Mythic+ | 12.6 | 10 | worst 29 (hard pity 400) |
| Full non-Cosmic roster | 16.4 | 14 | P95 29 |

Part 7's Lune distribution test must reproduce these within tolerance: ±10% on means, using a fixed seed and ≥ 2,000 runs.
