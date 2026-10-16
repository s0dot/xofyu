# HULLABRAWL — Studio Build Spec

**How an AI coding agent builds the M0 feel prototype and the M1-G graybox vertical slice in Roblox Studio**

| | |
|---|---|
| **Spec version** | 2.0 |
| **Date** | 16 October 2026 |
| **Status** | v2.0: rewritten for the original world; builds on the reviewed v1.0 engine spec |
| **Game** | *Hullabrawl*, an original battlegrounds brawler set in the Wishaway Isles |
| **Design source** | [`docs/GDD.md`](GDD.md) v2.0 (all tuning numbers; this spec never changes them) |
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
| 8 | Originality and naming layer | [`docs/build-spec/part-8-originality.md`](build-spec/part-8-originality.md) | §8.x |

Companion files: [`docs/agent/TASKS.md`](agent/TASKS.md) (ordered task cards), [`AGENTS.md`](../AGENTS.md) (agent operating manual), [`docs/GDD.md`](GDD.md) (design source of truth).

---

## 0. Shared contracts

Every part uses the names, IDs, paths and decisions in this part **verbatim**. If another part appears to contradict Part 0, Part 0 wins: follow it and record a `spec-gap` blocker (§1.8.1).

### 0.0 Purpose and scope of the build spec

**Game.** *Hullabrawl*, a Roblox battlegrounds fighter set in **the Wishaway Isles**, an original world of seaside holiday islands. Players are Guests; they collect **Charms** (Fighting Styles) by posting Wishes through Pip-Pip the postbox. Player-facing names are fixed in §12.

**Audience.** An AI coding agent (Claude Code or similar) driving **Roblox Studio** through Studio's **built-in MCP server**. Code lives on disk and is synced into Studio by **Script Sync**, following Roblox's official "coding harness". A human operator handles the steps the agent cannot do.

**Build target.** Two milestones, in order:
1. **M0 "Feel prototype"** (graybox):
   - 2 styles: `attendant` and `lifeguard`
   - full shared combat chassis: M1 chain with Enders, block / Perfect Block / guard, 3 dashes, hitstun / Stun Decay, ragdoll state, Breakaway, Peak Meter plus Peak (the awakened state, `AwakeningDef`) for both styles
   - 1 graybox duel arena (the Bandstand, `arena_duel_m0`) and the training dummy in the Staff Room (`training`)
   - Server Authority netcode
   - PC + mobile touch + gamepad input
   - ≥ 30 fps on the low-end reference with 8 clients plus server bots
   - Engine item **E2** (`chase` targets the enemy nearest the carrier, so a speed-0 summon works as a trap or turret). It lands with the Summon behavior, before the M0 kits, because Sonny's Quirk *Reserved* (`attendant.q`) is an M0 move (Part 4 §4.2.7, §4.9 item 24)
2. **M1-G "Graybox Vertical Slice"**:
   - 6 styles: `attendant`, `lifeguard`, `sandcastle`, `scoop`, `busker`, `dustsheet`
   - Elsewhere-on-Sea graybox map (`hub`, 20 players), including the tide: a deterministic, server-scheduled volume swap every 4 minutes (Part 6 §6.3.8)
   - Charm Ring and Charm Rack (internally the Style Wheel and the Style Totem, §12)
   - Wish system end-to-end: odds, pity (Guaranteed Delivery), Finishes (the `Flair` field), ledger, PolicyService gating, mock purchases
   - Sundry's Rack (the shop), Account Level and Style Mastery (data only plus simple UI), HUD
   - Full Uniform toggle (`CharacterLook`, default Off) and the appearance system (Charm Gear, Postcard Look)
   - Persistence via ProfileStore (mock in Studio)
   - Telemetry wrapper
   - One Getaway (Realm): `dustsheet` A4 "Greetings from the Closed Season", rules keyed `Realm.rules["dustsheet.a4"]`, without Double Booking (Realm Clash)
   - Engine item **E1** (a Realm `tick` rule can read the member's position; if it slips, the Getaway's Lights Out rule ships as darkness only; Part 3 §3.9.3, Part 4 §4.9 item 23)

**Out of scope for this spec:** final art, audio and animation (placeholders only), ranked/matchmaking, Excursions (raids), seasons, Beach Huts (guilds), the Holiday Pass (battle pass), live-ops tooling, Double Booking (Realm Clash), Double Acts (Bond Moves), world events other than one stub, and the Year-1 engine items E3–E7 (Part 4 §4.9 item 25, future hooks). Each part may list "future hooks".

**Design source of truth.** `docs/GDD.md` (v2.0+) holds all tuning numbers (frame data, odds, prices). The spec **must not change GDD numbers**. Encode them in `Shared/Config/Defaults.luau` and cite them as "GDD §x.y". Kits are cited as "GDD §7.3.N" with the numbers in §4.

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
| 8 | Originality and naming layer | `P8_ip.md` | §8.x |

- **Task plan:** `TASKS.md`, which ships as `docs/agent/TASKS.md`.
- **Root agent files** (they ship at repo root): `AGENTS.md`, `CLAUDE.md`, `ARCHITECTURE.md`.
- **Skills:**
  - `skills/hullabrawl-conventions.md` and `skills/hullabrawl-style-kit.md`, in Roblox Assistant skill format (frontmatter `name` / `description` / `enabled`). They ship to `docs/agent/skills/`.
  - A Claude Code mirror at `.claude/skills/hullabrawl-studio/SKILL.md` (frontmatter `name: hullabrawl-studio`).
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
| `ReplicatedStorage.Looks` | Client-visible cosmetic templates: Charm Gear props, the optional Peak pieces, summon models, the proxy look rig and the Postcard Look hats and outfits (Part 2 §2.13 item 3; Part 6 §6.8.6) |
| `ReplicatedStorage.Remotes` | Created at runtime by the server |
| `ReplicatedStorage.Runtime` | Runtime config StringValue, created at runtime |
| `ServerStorage.Templates` | Dummy and bot rigs and other server-only templates |
| `Workspace.Map` | Container for built maps: `Workspace.Map.Hub` (Elsewhere-on-Sea, `BuildMap_Hub`) and `Workspace.Map.Training` (the Staff Room, `BuildTraining`) |
| `Workspace.Arena` | M0 duel arena (the Bandstand) |
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
    Styles/   init.luau  attendant.luau  lifeguard.luau  sandcastle.luau  scoop.luau  busker.luau  dustsheet.luau
    Behaviors/ Projectile.luau  Rush.luau  Grab.luau  Counter.luau  Zone.luau  Summon.luau  Buff.luau  Mobility.luau  RealmCast.luau
    Skin/     init.luau  Packs/Base.luau
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
    Builders/ BuildAll.luau  BuildInputs.luau  BuildArena.luau  BuildMap_Hub.luau  BuildTraining.luau
              BuildLighting.luau  BuildVfxTemplates.luau  BuildPlaceholderAnims.luau  BuildTemplates.luau  Verify.luau
    Tests/    (in-Studio scenario modules; see Part 7)
  StarterPlayerScripts/
    Main.client.luau
    Controllers/ InputController.luau  CameraController.luau  RenderController.luau  AnimController.luau  VfxController.luau
                 SfxController.luau  HudController.luau  UiController.luau  AppearanceController.luau  SettingsController.luau
    UI/ Create.luau  Theme.luau  Screens/ Hud.luau  StyleWheel.luau  Wish.luau  Shop.luau  Settings.luau  Results.luau
tests/   (Lune) run.luau  core/*.spec.luau  wish/*.spec.luau  shim/ (Roblox datatype shims if needed)
tools/   lint_strings.luau  lint_retired_keys.luau  check_manifest.luau  export_framedata.luau  (Lune)
         lint_strings.rules.json (rules data, generic terms only; Part 8)
         lib/SafeWrite.luau (the only file writer for Lune host tools; Part 8)
         denylist.local.txt (gitignored; owner- or counsel-supplied; never committed; Part 8)
         retired_keys.txt (gitignored; owner-supplied; never committed, written, printed or quoted by the agent; Part 8)
         studio/ (tiny execute_luau snippets, ≤ 2 KB each)
         plugin/HullabrawlTestHarness/ (local Studio plugin source for multi-client tests)
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
  - No deprecated APIs: `PhysicsService` collision APIs (use `workspace:RegisterCollisionGroup`), `KeyframeSequenceProvider` (use `AnimationClipProvider`), `MarketplaceService:GetProductInfo` (use `GetProductInfoAsync`), `InputAction:Fire` (use `InputBinding:Fire` with a Scriptable binding), `Player:LoadCharacter` (use `LoadCharacterAsync`), `AlignOrientation.PrimaryAxisOnly` (use `AlignType`), `Enum.HumanoidStateType.Ragdoll` (use our own Ragdoll state), BodyMovers.
- **Bootstrap.**
  - `Main.server.luau` requires every module in `Services/`, calls `init()` on each in the fixed order listed in Part 2, then `start()`.
  - `Main.client.luau` does the same for `Controllers/`.
- **Logging.**
  - `Log.info/warn/error(tag, msg, data?)` prints `[HB][<tag>] msg`. The tag list is in Part 2 §2.3.6.
  - Test lines are exactly `[HB][TEST] <name> PASS` or `[HB][TEST] <name> FAIL <detail>`, plus a final `[HB][TEST] SUMMARY pass=<n> fail=<m>`.
  - Boot lines: `[HB][BOOT] server ready …` and `[HB][BOOT] client ready …`.
- **Prefixes.**
  - Log prefix `[HB]`.
  - Instance, live-config override and environment names use `HB_` (e.g. `HB_Proto`, `HB_Features`, `HB_Kits`, `HB_OPEN_CLOUD_API_KEY`).
  - Attributes and identifiers use `Hb` (e.g. `HbClientReady`, `HbBuilder`); runtime scratch folders are `__HbSpike` and `__HbScratch`.
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
  - The hub's tide (a volume swap every 4 minutes) is deterministic and server-owned: swaps fall on fixed 4-minute boundaries of the server's simulation clock, so both peers swap on the same frame, and the tide never depends on client input (Part 6 §6.3.8).
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
  - **Style IDs** (fixed). They are internal keys and never shown to players; the name and epithet come from `Skin`. `Types.STYLE_IDS` lists exactly these 18 IDs in this order. GDD §7.3.N follows the same order.

    | GDD § | Rarity | ID | Name | Epithet | Archetype (Part 4) |
    |---|---|---|---|---|---|
    | 7.3.1 | `starter` | `attendant` | Sonny | Take a Seat | `DeckchairAttendant` |
    | 7.3.2 | `starter` | `lifeguard` | Paddles | The Lifeguard Who Can't Swim | `HippoLifeguard` |
    | 7.3.3 | `common` | `skimstone` | Skimmer | Never Sinks | `SkimmingStone` |
    | 7.3.4 | `common` | `masseur` | Kneady | The Mandatory Massage | `BottleMasseur` |
    | 7.3.5 | `common` | `bellhop` | Pocket | The Express Bellhop | `HamsterBellhop` |
    | 7.3.6 | `uncommon` | `angler` | Stilts | The Patient Hook | `HeronAngler` |
    | 7.3.7 | `uncommon` | `kite` | Fret | The Kite Afraid of Heights | `NervousKite` |
    | 7.3.8 | `uncommon` | `boxcamera` | Shutter | Say Cheese | `BoxCamera` |
    | 7.3.9 | `rare` | `sandcastle` | Burrows | The Tide-Proof Architect | `MoleBuilder` |
    | 7.3.10 | `rare` | `scoop` | Dollop | Every Flavour Ever | `SealScooper` |
    | 7.3.11 | `rare` | `diver` | Plunge | The Perfect Ten | `DolphinDiver` |
    | 7.3.12 | `epic` | `busker` | Bellows | The Pier Busker | `GullBusker` |
    | 7.3.13 | `epic` | `lamplighter` | Flicker | The Light That Never Went Out | `PuffinLamplighter` |
    | 7.3.14 | `epic` | `cablecar` | Yodel | The Summit Echo | `CuckooClock` |
    | 7.3.15 | `legendary` | `dustsheet` | Mothball | Under Wraps | `DustSheetSpirit` |
    | 7.3.16 | `legendary` | `raincloud` | Bluster | Rain Stopped Play | `StormCloud` |
    | 7.3.17 | `mythic` | `serpent` | Seldom | The Seldom Seen | `SeaSerpent` |
    | 7.3.18 | `cosmic` (secret) | `ferry` | Homeward | The Last Ferry | `LastFerry` |

  - **Starter and implemented sets.**
    - `Types.STARTER_STYLES` = `attendant`, `lifeguard`.
    - `Types.IMPLEMENTED_M0` = `attendant`, `lifeguard`.
    - `Types.IMPLEMENTED_M1G` = `attendant`, `lifeguard`, `sandcastle`, `scoop`, `busker`, `dustsheet`. Both are test aids; `IMPLEMENTED_M1G` grows one ID per kit card, and each `StyleDef.implemented` is the authority.
    - The other 12 styles are stubs (`implemented = false`).
  - **Move IDs:** `<style>.m1`, `<style>.1` … `<style>.4`, `<style>.q` (Quirk), `<style>.a1` … `<style>.a4`, `<style>.passive`, `<style>.awaken` (e.g. `attendant.4`, `dustsheet.a4`).
  - **Map IDs:** `arena_duel_m0` (the Bandstand, M0 duel arena), `hub` (Elsewhere-on-Sea, the M1-G battleground; builder `BuildMap_Hub`, instance `Workspace.Map.Hub`), `training` (the Staff Room; `Workspace.Map.Training`). `Types.MAP_IDS` holds these three. Reserved for Year 1 and not built in M1-G: `gullcliff` (Gullcliff), `mountcocoa` (Mount Cocoa).
  - **Flair IDs** (the `Flair` data field; players see **Finishes**): `plain`, `glossy`, `gilt`, `seaglass`, `pearl`, `glowtide`, `misprint`, in ascending rarity (`Types.FLAIR_ORDER`). Display names: Plain, Glossy, Gilt, Sea-Glass, Pearl, Glowtide, Misprint.
  - **Rarity IDs:** `starter`, `common`, `uncommon`, `rare`, `epic`, `legendary`, `mythic`, `cosmic`.
- **Skin (naming layer).**
  - All player-visible strings come from `Skin.text(key)` / `Skin.style(id)`.
  - One pack ships: **`Base`** (`src/Shared/Skin/Packs/Base.luau`). It holds the roster names and epithets above, the player-facing names in §12, and every UI string.
  - The module keeps its full API (`Skin.text`, `Skin.style`, `Skin.activePack()`, `Skin.onChanged(fn)`) so locale or seasonal text overrides can be added later without touching callers.
  - Key format: `style.<id>.name`, `style.<id>.epithet`, `move.<moveId>.name`, `map.<mapId>.name`, `map.hub.zone.<zoneId>`, `ui.<screen>.<key>`, `flair.<id>.name`, `rarity.<id>.name`, `variant.<styleId>.<vid>.name`, `title.level_<n>`, `title.mastery_<styleId>_5`, `title.master_<styleId>`, `product.<key>.name`, `pass.<key>.name`, `badge.<key>.name`. These are the only namespaces (Part 8 §8.2.2). Fixed keys: `ui.game.title` = "Hullabrawl", `ui.world.name` = "The Wishaway Isles".
  - Each key's `Base` text is owned by the part that defines the key: Part 4 the style and move keys, Part 5 the Wish, shop, progression and appearance-data keys, Part 6 the HUD, UI and world keys, Part 8 the remaining core keys (including the two fixed keys). Every other file that quotes a key copies the owner's text verbatim (Part 8 §8.3).
  - The repository never contains third-party names. `tools/lint_strings.luau` checks player-visible strings and metadata against the optional local denylist `tools/denylist.local.txt` (required for a public publish, optional in dev). `tools/lint_retired_keys.luau` fails on retired keys and prefixes in `src/`, `tests/`, `tools/` and in anything that replicates to clients (retired style IDs only as style-ID literals or move-ID prefixes; never plain English words or the Wish system). It reads the owner's local list `tools/retired_keys.txt` (gitignored; the agent never writes, prints or quotes it) and prints findings only as `<path>:<line> retired-key`. Without the list it prints `SKIP no retired_keys list` and dev work proceeds; a public publish needs the list present and the lint passing.
  - Part 8 ("Originality and naming layer") defines the pack rules, the lint rules and lint policy, and the publish gate; Part 7 §7.5 builds both lint tools.
- **Asset manifest.**
  - `Shared/Assets.luau` maps logical keys to `{id: string, status: "placeholder" | "final", source: "builtin" | "agent" | "ai" | "creator-store" | "artist", kind: "image" | "audio" | "anim" | "mesh" | "model" | "vfx", note: string?}`.
  - Code calls `Assets.get(key)`, never a raw asset ID.
  - Per-universe product IDs live under `Assets.products[gameId]`.
- **Appearance (owner's rule: the full character look is optional).**
  - Per-player settings, saved in profile:
    - `CharacterLook: boolean` (default **false**): the **Full Uniform** toggle. On, the player shows the equipped style's full uniform; off, their own look plus Charm Gear.
    - `OwnLook: "Postcard" | "Avatar"` (default "Postcard"). `"Postcard"` is the **Postcard Look**: the player's own avatar drawn in the postcard toon style, with one of four neutral body styles (`classic`, `round`, `tall`, `tiny`; `Appearance.PostcardStyles`) and a skin tone chosen separately from the skin palette `Appearance.PostcardSkins`. The look is stored as `PostcardSpec = { v, body, skin, color, face, hat, outfit }` (Part 2 §2.4; values in Part 5 §5.10); body-style proportions are render-only (`Perf.Looks.BodyScale[<id>]`, Part 6 §6.8.2). `"Avatar"` is the player's Roblox avatar as it is.
    - `ShowStyleGear: boolean` (default true): **Charm Gear**, one oversized held or back prop plus a head piece. The prop is the style's silhouette read.
  - Viewer-local setting `OthersAppearance: "AsChosen" | "AlwaysCharacter" | "AlwaysOwn" | "Simplified"`.
  - At Peak the style's uniform appears for 60 s (GDD §6.9) whatever the Full Uniform setting, for every viewer except one who chose `Simplified` (render only, Part 6 §6.8.2). Nobody grows.
  - The **Style Tell** (nameplate chip in the style colour, style VFX palette) is always on. The only exceptions are the stealth render (deferred until its first user; the shimmer then carries the style colour) and Studio-only capture staging for the blind tests (Part 6 §6.8.2, §6.8.4).
  - The hurtbox is a fixed capsule in every mode (GDD §6.2). No Peak scales it: `hurtboxScale = 1` for every kit.
  - Graybox rendering of each mode is defined in Part 6; data and replication in Part 5.

### 0.5 Placeholder policy (fixed)

- **The agent builds everything as graybox.** Parts, CSG, Terrain, built-in fonts, built-in particle textures, untextured Beams/Trails, and the 11 built-in sounds.
- **AI generation tools** (`generate_mesh` / `generate_procedural_model` / `generate_material`) are used **only** for generic props. Never for characters, the mascot, Charm Gear, summon models, or anything prompted with a character, place or third-party name. Tag results `source = "ai"`.
- **Final character art** (Pip-Pip, the staff and the core faces) is drawn by people under written assignment, never generated by AI (GDD §5.4, §12.8).
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
| P5 | Style framework and kits (M0: `attendant`, `lifeguard`; M1-G: + `sandcastle`, `scoop`, `busker`, `dustsheet` incl. its Getaway) | T160–T199 |
| P6 | World (the Bandstand arena, Elsewhere-on-Sea graybox, the Staff Room, Charm Racks, lighting) and UI screens | T200–T229 |
| P7 | Meta systems (style ownership, Wish, Sundry's Rack, progression, policy, mock monetization, telemetry) | T230–T269 |
| P8 | Hardening (bots and load test, perf pass, anti-exploit, originality publish gate, docs/ARCHITECTURE update) | T270–T299 |

**Milestone gates:**
- **M0 gate:** after P5's `attendant`/`lifeguard` tasks plus P6's arena tasks.
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
| H-01 | Create the dev universe and place "Hullabrawl DEV" (group-owned); save the place to a file inside the repo folder (gitignored) |
| H-02 | Enable Studio MCP server (Assistant → ⋯ → Manage MCP Servers) and connect Claude Code |
| H-03 | Enable Script Sync and sync the four roots |
| H-04 | Set `Workspace.AuthorityMode = Server` and `StreamingEnabled`; set `StarterPlayer.AvatarJointUpgrade = Enabled` in Properties |
| H-05 | Avatar Settings: Consistent Gameplay preset + Single Collider; enable the CCL beta, if adopted after spike |
| H-06 | Game Settings → Security: Enable Studio Access to API Services (dev universe only) |
| H-07 | Create a group-owned Open Cloud API key (asset:read/write) for placeholder animation uploads; store it outside the repo |
| H-08 | Publish placeholder animations (or approve the agent upload pipeline) |
| H-09 | Create dev-universe developer products (not for sale) or approve the Open Cloud script; prices need human sign-off |
| H-10 | Feel playtests at each combat gate |
| H-11 | Publish (human only; the agent never publishes). To the dev place: after the agent reports the Part 8 dev-publish checks green (`[HB][GATE] publish-dev PASS`, Part 8 §8.4.7). Any public build also needs the **originality publish gate** (Part 8 §8.4.4, procedure §8.4.7): blind tests T7–T11 signed off in `docs/agent/RECOGNIZABILITY.md` (T11, the gameplay-clip test, runs after M0 and before any public footage), `docs/agent/TITLE_CLEARANCE.md` signed (title knockout and counsel clearance), the local denylist `tools/denylist.local.txt` present so `lint_strings` checks against it, the owner's `tools/retired_keys.txt` present and `lint_retired_keys` passing, asset provenance complete, and no third-party name in the title, description, tags, thumbnails, update names, ads or search keywords |
| H-12 | Mobile device test (low-end Android reference) |

### 0.8 Canonical data types (fixed; Part 2 places them in `Shared/Types.luau`, Part 4 uses them)

```lua
export type Rarity = "starter" | "common" | "uncommon" | "rare" | "epic" | "legendary" | "mythic" | "cosmic"
export type Tag = "Projectile" | "Rush" | "Grab" | "Counter" | "Zone" | "Summon" | "Buff" | "Debuff" | "Mobility"
	| "AoE" | "Utility" | "Stealth" | "Ultimate" | "Realm" | "Music" | "Opener" | "Extender"
export type Behavior = "Strike" | "Projectile" | "Rush" | "Grab" | "Counter" | "Zone" | "Summon" | "Buff" | "Mobility" | "RealmCast"
export type HitboxDef = { shape: "Box" | "Sphere" | "Cast", size: Vector3, offset: CFrame, castDistance: number? }
export type KnockbackDef = { dist: number, angleDeg: number, ragdoll: boolean, launch: boolean? }
export type MoveDef = {
	id: string,                       -- e.g. "attendant.3"
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
export type AwakeningDef = {          -- the Peak (player-facing name, §12)
	id: string, durationSec: number,  -- 60 (GDD §6.9)
	hurtboxScale: number?,            -- 1 for every kit: no Peak scales the hurtbox (GDD §6.2)
	modifiers: { [string]: any }?,    -- Peak passive modifiers (only where the kit says so)
	moves: { MoveDef },               -- a1..a4 (a4 may be behavior "RealmCast" for legendary+)
}
export type StyleDef = {
	id: string, rarity: Rarity, implemented: boolean,
	archetype: string,                -- §4 archetype string, e.g. "DeckchairAttendant"
	hp: number,                       -- 100 at launch (GDD §6.2)
	palette: { primary: Color3, secondary: Color3, accent: Color3 },
	passive: { id: string, params: { [string]: any } },
	quirk: MoveDef,
	moves: { MoveDef },               -- slots "1".."4"
	awakening: AwakeningDef,
	m1: { anim: string, vfx: string?, sfx: string? },  -- M1 frame data is shared (GDD §6.3); only presentation differs
	kitProps: { string }?,            -- logical template keys for Charm Gear props and summon models
}
```

- `kitProps` order: the two Charm Gear keys (`model.<style>.gear_<name>`: the oversized prop, then the head piece), then every summon model key (`model.<style>.summon_<name>`; Part 4 rule K23). Part 6 attaches only the `gear_` entries as Charm Gear. The optional Peak piece (`model.<style>.peak_<name>`, Part 4 §4.7.4) is not listed in `kitProps`.

### 0.9 Config top-level structure (fixed keys; each owning part fills its section)

```
Defaults = {
  Version: number,
  NetModel: "ServerAuthority" | "Classic",
  MockPurchases: boolean,                -- Part 5
  Combat = { Chassis, M1, Block, Dash, Stun, Ragdoll, Breakaway, Meter, Awakening, Realm, Damage, Respawn, PileOn, Streak },  -- Part 3
  Wish = { TierOdds, Pity, Flair, LuckyJarFlairMultiplier, StorageCap, OverflowTreasure, PoolMode, DuplicateShards, VariantChance, CosmicSparksNeeded, FirstWishGuarantee },  -- Part 5
  Shop = {...}, Economy = {...}, Progression = {...}, Appearance = {...}, Telemetry = {...},  -- Part 5
  Perf = {...},                           -- Part 6 (VFX budgets per device tier)
}
```

- Config keys are internal identifiers; players see the §12 names (for example `OverflowTreasure` pays **Shells**, `DuplicateShards` pays **Stamps**, `CosmicSparksNeeded` counts **Messages in a Bottle**, and `LuckyJarFlairMultiplier` belongs to the **Lucky Stamp** pass). The internal pass keys `vip`, `quick_wish` and `lucky_jar` stay as they are; prose and Skin text say First Class, Quick Post and Lucky Stamp.
- One skin pack ships (`Base`, §4), so Config has no pack selector. The Part 8 publish gate is human-run and has no Config flag.
- The full character look is a per-player setting (`CharacterLook`, §4), never a Config switch, and there is no look-set switch in `Features`.

### 0.10 Wish pool rule for M1-G (fixed)

- The Wish pool uses the **full 16-style rollable roster and the odds from GDD §8.3–§8.4**: `Wish.PoolMode = "Full"`.
- Pool membership by rarity (GDD §7.2; `Types.STYLE_IDS` order inside each tier):

  | Rarity | Styles | Count |
  |---|---|---|
  | `common` | `skimstone`, `masseur`, `bellhop` | 3 |
  | `uncommon` | `angler`, `kite`, `boxcamera` | 3 |
  | `rare` | `sandcastle`, `scoop`, `diver` | 3 |
  | `epic` | `busker`, `lamplighter`, `cablecar` | 3 |
  | `legendary` | `dustsheet`, `raincloud` | 2 |
  | `mythic` | `serpent` | 1 |
  | `cosmic` | `ferry` | 1 |

- Styles with `implemented = false` are **collectible but not equippable** ("In development" badge). This keeps the GDD odds and pity (Guaranteed Delivery) math intact while only 6 kits exist. In M1-G the implemented rollable styles are `sandcastle` and `scoop` (Rare), `busker` (Epic) and `dustsheet` (Legendary).
- The two **Starter** styles (`attendant`, `lifeguard`) are granted to every player and never appear in the pool.
- The first Wish is a guaranteed Rare: `Wish.FirstWishGuarantee.Styles = { "sandcastle", "scoop" }`, `Weights = { 1, 1 }` (Burrows or Dollop, 50/50; GDD §15.5, Part 5 §5.10). Both are implemented in M1-G.

### 0.11 Simulation-verified economy numbers (fixed; from a 4,000-run Monte Carlo of the GDD §8.4 pity rules at 14 Wishes/day)

| Milestone | Mean day | Median day | Worst / P95 |
|---|---|---|---|
| First Epic+ | 1.4 | 1 | worst 3 |
| First Legendary+ | 3.2 | 3 | worst 9 (hard pity 120) |
| First Mythic+ | 12.6 | 10 | worst 29 (hard pity 400) |
| Full non-Cosmic roster | 16.4 | 14 | P95 29 |

Part 7's Lune distribution test must reproduce these within tolerance: ±10% on means, using a fixed seed and ≥ 2,000 runs.

### 0.12 Player-facing names

Internal identifiers stay as they are in code, data, config and Skin keys. Prose about the player experience and every `Skin` string uses the player-facing name. Every part cites this table as §0.12.

| Internal identifier (unchanged) | Player-facing name |
|---|---|
| Realm (`RealmCast`, `Realm.rules`, `RlmId`) / Realm Clash / the universal Realm counter | **Getaway** / **Double Booking** / **Do Not Disturb** |
| Awakening / Meter (`Meter` attribute) | **Peak** ("**Go Peak!**", the uniform appears for 60 s; nobody grows) / **Peak Meter** |
| Wish / 10× roll / pity | **Wish** (a postcard posted through Pip-Pip) / **Mailbag** / **Guaranteed Delivery** |
| Flair (data field) | **Finish**: Plain, Glossy, Gilt, Sea-Glass, Pearl, Glowtide, Misprint |
| duplicate-refund currency | **Stamps** |
| Cosmic tokens (10 = guaranteed Cosmic) | **Messages in a Bottle** |
| `Treasure` (earn-only currency field) | **Shells** |
| the deterministic shop | **Sundry's Rack** |
| luck pass / VIP pass / quick-roll pass | **Lucky Stamp** / **First Class** / **Quick Post** |
| battle pass / subscription | **Holiday Pass** / **Deckchair Club** |
| Variants (alternate looks) | **Off-Duty Looks** (no themed variants of anyone else's characters) |
| `CharacterLook` (bool) / `ShowStyleGear` / own-look `Postcard` | **Full Uniform** (default **Off**, player toggle) / **Charm Gear** (one oversized prop + head piece; always the silhouette read) / **Postcard Look** |
| Style Totem / Style Wheel / Collection | **Charm Rack** / **Charm Ring** / **Scrapbook** |
| Fighting Style | **Charm** (in fiction a souvenir; its power is a **knack**). "Fighting Style" stays fine in UI help text |
| style moment / Bond Move / Finale | **Snapshot** / **Double Act** / **Send-off** |
| guilds / guild wars / guild hall / Cosmic hall of fame | **Beach Huts** / **Bunting Wars** / **The Winter Gardens** / **The Bottle Wall** |
| training area / creator workshop / raid | **The Staff Room** / **The Sandpit** / **Excursion** |
| party modes | **Evening Entertainments**: Lucky Dip; The Tide's Coming In; Freeze Frame; King of the Pool (team hill); Pick a Postcard; Bouncy Castle Brawl (GDD §10.5) |
| ranked tiers | Day Tripper → Weekender → Holidaymaker → Regular → Old Hand → Local Legend → Grand Tourer; top 100 = **Headliners**; monthly **Headliner Cup** |
| KO call-out | "**Checked out!**" (a puff of sand and a waving hankie) |

- Generic terms stay as they are: Breakaway, Perfect Block, Ender, Hop Cancel, Stun Decay, Quirk, M1.
- Status and move tags are internal; the `Tag` values are exactly those in §8 (`Music` stays).
- Style names and epithets: §4 Style IDs table. Map names: §4 Map IDs.
