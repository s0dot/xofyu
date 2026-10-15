# ARCHITECTURE.md — source of truth for what exists

**Rule:** this file lists every module, script folder and instance folder of *Algebraic!*. Read it instead of
querying Studio. **Update it in the same turn (and the same commit) whenever anything is added, moved, renamed,
deleted, or changes status.** `tools/lint_rules.luau` (rule `architecture`) fails the host gate if a file under
`src/` has no row, or a `wip`/`done` row has no file.

**Row format (machine-checked):** six cells — disk path (in backticks, e.g. `src/Shared/Log.luau`),
Studio path, kind, status, owner part, notes. `lint_rules` reads every table row whose first cell starts with a
backticked `src/` path.
- Status: `planned` → `wip` → `done` (or `deprecated`). A folder row (path ending in `/`) covers every file under
  it that has no row of its own.
- Owner part = the build-spec part that defines the module (§2 = Part 2, etc.). Owners marked `*` are
  provisional; correct them to match the assembled spec the first time you touch the row.
- Kinds: `Module` (ModuleScript), `Script/Server`, `Script/Client`, `Folder`, `Builder`, `Runtime`, `Setting`, `Host`.

## 1. Synced roots (disk ↔ Studio via Script Sync; disk is the source of truth)

### 1.1 `ReplicatedStorage.Shared` ↔ `src/Shared/`
| Disk path | Studio path | Kind | Status | Owner | Notes |
|---|---|---|---|---|---|
| `src/Shared/Types.luau` | ReplicatedStorage.Shared.Types | Module | planned | Part 2 | Canonical types (§0.8); pure |
| `src/Shared/Log.luau` | ReplicatedStorage.Shared.Log | Module | planned | Part 2 | `[ALG][<tag>] msg` |
| `src/Shared/Assets.luau` | ReplicatedStorage.Shared.Assets | Module | planned | Part 2 / Part 6* | Logical key → asset entry; `Assets.products[gameId]` |
| `src/Shared/Config/Defaults.luau` | ReplicatedStorage.Shared.Config.Defaults | Module | planned | Part 2 (sections: 3, 5, 6, 8) | Every GDD tunable; pure |
| `src/Shared/Config/Schema.luau` | ReplicatedStorage.Shared.Config.Schema | Module | planned | Part 2 | Config validation; pure |
| `src/Shared/Config/Runtime.luau` | ReplicatedStorage.Shared.Config.Runtime | Module | planned | Part 2 | `Runtime.get()`, `Runtime.version()` |
| `src/Shared/Core/Fsm.luau` | ReplicatedStorage.Shared.Core.Fsm | Module | planned | Part 3 | Pure; Lune-tested |
| `src/Shared/Core/Frames.luau` | ReplicatedStorage.Shared.Core.Frames | Module | planned | Part 3 | `toSeconds(f) = f / 60` |
| `src/Shared/Core/M1.luau` | ReplicatedStorage.Shared.Core.M1 | Module | planned | Part 3 | Pure |
| `src/Shared/Core/Defense.luau` | ReplicatedStorage.Shared.Core.Defense | Module | planned | Part 3 | Pure |
| `src/Shared/Core/Movement.luau` | ReplicatedStorage.Shared.Core.Movement | Module | planned | Part 3 | Pure |
| `src/Shared/Core/Stun.luau` | ReplicatedStorage.Shared.Core.Stun | Module | planned | Part 3 | Pure |
| `src/Shared/Core/Meter.luau` | ReplicatedStorage.Shared.Core.Meter | Module | planned | Part 3 | Pure |
| `src/Shared/Core/Breakaway.luau` | ReplicatedStorage.Shared.Core.Breakaway | Module | planned | Part 3 | Pure |
| `src/Shared/Core/Moves.luau` | ReplicatedStorage.Shared.Core.Moves | Module | planned | Part 3 | Pure |
| `src/Shared/Core/Damage.luau` | ReplicatedStorage.Shared.Core.Damage | Module | planned | Part 3 | Pure |
| `src/Shared/Core/Rng.luau` | ReplicatedStorage.Shared.Core.Rng | Module | planned | Part 3* | Pure, seeded; also used by Wish |
| `src/Shared/Core/Realm.luau` | ReplicatedStorage.Shared.Core.Realm | Module | planned | Part 3* / Part 4 | Pure |
| `src/Shared/Sim/Simulation.luau` | ReplicatedStorage.Shared.Sim.Simulation | Module | planned | Part 3 | BindToSimulation @ Hz60 |
| `src/Shared/Sim/Attr.luau` | ReplicatedStorage.Shared.Sim.Attr | Module | planned | Part 3 | Canonical attribute list |
| `src/Shared/Sim/InputMap.luau` | ReplicatedStorage.Shared.Sim.InputMap | Module | planned | Part 3 | IAS actions and default bindings |
| `src/Shared/Sim/Adapter.luau` | ReplicatedStorage.Shared.Sim.Adapter | Module | planned | Part 3 | `NetModel` adapter (ServerAuthority/Classic) |
| `src/Shared/Combat/Query.luau` | ReplicatedStorage.Shared.Combat.Query | Module | planned | Part 3 | Spatial queries |
| `src/Shared/Combat/Rewind.luau` | ReplicatedStorage.Shared.Combat.Rewind | Module | planned | Part 3 | Hurtbox history ≤ 200 ms rewind |
| `src/Shared/Combat/Hurtbox.luau` | ReplicatedStorage.Shared.Combat.Hurtbox | Module | planned | Part 3 | Fixed capsule |
| `src/Shared/Styles/init.luau` | ReplicatedStorage.Shared.Styles | Module | planned | Part 4 | Style registry |
| `src/Shared/Styles/hero.luau` | ReplicatedStorage.Shared.Styles.hero | Module | planned | Part 4 | M0 |
| `src/Shared/Styles/stretch.luau` | ReplicatedStorage.Shared.Styles.stretch | Module | planned | Part 4 | M0 |
| `src/Shared/Styles/monarch.luau` | ReplicatedStorage.Shared.Styles.monarch | Module | planned | Part 4 | M1-G |
| `src/Shared/Styles/rocker.luau` | ReplicatedStorage.Shared.Styles.rocker | Module | planned | Part 4 | M1-G |
| `src/Shared/Styles/frost.luau` | ReplicatedStorage.Shared.Styles.frost | Module | planned | Part 4 | M1-G |
| `src/Shared/Styles/decay.luau` | ReplicatedStorage.Shared.Styles.decay | Module | planned | Part 4 | M1-G; Realm A4 |
| `src/Shared/Behaviors/Projectile.luau` | ReplicatedStorage.Shared.Behaviors.Projectile | Module | planned | Part 4 | |
| `src/Shared/Behaviors/Rush.luau` | ReplicatedStorage.Shared.Behaviors.Rush | Module | planned | Part 4 | |
| `src/Shared/Behaviors/Grab.luau` | ReplicatedStorage.Shared.Behaviors.Grab | Module | planned | Part 4 | |
| `src/Shared/Behaviors/Counter.luau` | ReplicatedStorage.Shared.Behaviors.Counter | Module | planned | Part 4 | |
| `src/Shared/Behaviors/Zone.luau` | ReplicatedStorage.Shared.Behaviors.Zone | Module | planned | Part 4 | |
| `src/Shared/Behaviors/Summon.luau` | ReplicatedStorage.Shared.Behaviors.Summon | Module | planned | Part 4 | |
| `src/Shared/Behaviors/Buff.luau` | ReplicatedStorage.Shared.Behaviors.Buff | Module | planned | Part 4 | |
| `src/Shared/Behaviors/Mobility.luau` | ReplicatedStorage.Shared.Behaviors.Mobility | Module | planned | Part 4 | |
| `src/Shared/Behaviors/RealmCast.luau` | ReplicatedStorage.Shared.Behaviors.RealmCast | Module | planned | Part 4 | |
| `src/Shared/Skin/init.luau` | ReplicatedStorage.Shared.Skin | Module | planned | Part 8 | `Skin.text`, `Skin.style` |
| `src/Shared/Skin/Packs/Codename.luau` | ReplicatedStorage.Shared.Skin.Packs.Codename | Module | planned | Part 8 | Dev default pack |
| `src/Shared/Skin/Packs/Original.luau` | ReplicatedStorage.Shared.Skin.Packs.Original | Module | planned | Part 8 | IP-neutral |
| `src/Shared/Skin/Packs/Canon.luau` | ReplicatedStorage.Shared.Skin.Packs.Canon | Module | planned | Part 8 | Locked (`LicenseConfirmed`) |
| `src/Shared/Wish/Odds.luau` | ReplicatedStorage.Shared.Wish.Odds | Module | planned | Part 5 | Pure; UI shows identical odds |
| `src/Shared/Wish/Pity.luau` | ReplicatedStorage.Shared.Wish.Pity | Module | planned | Part 5 | Pure |
| `src/Shared/Net/init.luau` | ReplicatedStorage.Shared.Net | Module | planned | Part 2 | Typed RemoteEvent wrapper |
| `src/Shared/Net/Schemas.luau` | ReplicatedStorage.Shared.Net.Schemas | Module | planned | Part 2 | Remote schemas + rate limits |
| `src/Shared/Vendor/Trove.luau` | ReplicatedStorage.Shared.Vendor.Trove | Module | planned | Part 2 | Vendored (MIT); see THIRD_PARTY.md |
| `src/Shared/Vendor/Signal.luau` | ReplicatedStorage.Shared.Vendor.Signal | Module | planned | Part 2 | Vendored (MIT) |
| `src/Shared/Vendor/Shake.luau` | ReplicatedStorage.Shared.Vendor.Shake | Module | planned | Part 2 | Vendored (MIT) |

### 1.2 `ServerScriptService` ↔ `src/ServerScriptService/`
| Disk path | Studio path | Kind | Status | Owner | Notes |
|---|---|---|---|---|---|
| `src/ServerScriptService/Main.server.luau` | ServerScriptService.Main | Script/Server | planned | Part 2 | Fixed init/start order; prints `[ALG][BOOT] server ready` |
| `src/ServerScriptService/Services/DataService.luau` | ServerScriptService.Services.DataService | Module | planned | Part 5* | ProfileStore (mock in Studio) |
| `src/ServerScriptService/Services/PlayerService.luau` | ServerScriptService.Services.PlayerService | Module | planned | Part 2* | |
| `src/ServerScriptService/Services/StyleService.luau` | ServerScriptService.Services.StyleService | Module | planned | Part 4* / Part 5 | Ownership + equip |
| `src/ServerScriptService/Services/CombatService.luau` | ServerScriptService.Services.CombatService | Module | planned | Part 3 | Hit resolution, KO, respawn |
| `src/ServerScriptService/Services/ConfigReplicator.luau` | ServerScriptService.Services.ConfigReplicator | Module | planned | Part 2 | ConfigService overlay → `Runtime.Config` |
| `src/ServerScriptService/Services/WishService.luau` | ServerScriptService.Services.WishService | Module | planned | Part 5 | |
| `src/ServerScriptService/Services/ShopService.luau` | ServerScriptService.Services.ShopService | Module | planned | Part 5 | |
| `src/ServerScriptService/Services/ProgressionService.luau` | ServerScriptService.Services.ProgressionService | Module | planned | Part 5 | |
| `src/ServerScriptService/Services/MonetizationService.luau` | ServerScriptService.Services.MonetizationService | Module | planned | Part 5 | Mock purchases |
| `src/ServerScriptService/Services/PolicyGate.luau` | ServerScriptService.Services.PolicyGate | Module | planned | Part 5 | PolicyService gating |
| `src/ServerScriptService/Services/Telemetry.luau` | ServerScriptService.Services.Telemetry | Module | planned | Part 5 | |
| `src/ServerScriptService/Services/ModerationService.luau` | ServerScriptService.Services.ModerationService | Module | planned | Part 5* | |
| `src/ServerScriptService/Services/BotService.luau` | ServerScriptService.Services.BotService | Module | planned | Part 7* | Server bots for load tests |
| `src/ServerScriptService/Services/AppearanceService.luau` | ServerScriptService.Services.AppearanceService | Module | planned | Part 5 | Appearance data + replication |
| `src/ServerScriptService/Services/TestHarness.luau` | ServerScriptService.Services.TestHarness | Module | planned | Part 7 | Studio-only scenario runner |
| `src/ServerScriptService/Vendor/ProfileStore.luau` | ServerScriptService.Vendor.ProfileStore | Module | planned | Part 5* | Vendored (MIT) |

### 1.3 `ServerStorage.DevTools` ↔ `src/DevTools/`
| Disk path | Studio path | Kind | Status | Owner | Notes |
|---|---|---|---|---|---|
| `src/DevTools/Builders/Util.luau` | ServerStorage.DevTools.Builders.Util | Module | planned | Part 1 | Leaf; builder helpers (§1.6.4) |
| `src/DevTools/Builders/BuildAll.luau` | ServerStorage.DevTools.Builders.BuildAll | Builder | planned | Part 1 | Fixed order (§1.6.6) |
| `src/DevTools/Builders/Verify.luau` | ServerStorage.DevTools.Builders.Verify | Builder | planned | Part 1 | Read-only aggregator (§1.6.7) |
| `src/DevTools/Builders/BuildInputs.luau` | ServerStorage.DevTools.Builders.BuildInputs | Builder | planned | Part 3 | → `ReplicatedStorage.Inputs` |
| `src/DevTools/Builders/BuildArena.luau` | ServerStorage.DevTools.Builders.BuildArena | Builder | planned | Part 6 | → `Workspace.Arena` |
| `src/DevTools/Builders/BuildMap_CandyKingdom.luau` | ServerStorage.DevTools.Builders.BuildMap_CandyKingdom | Builder | planned | Part 6 | → `Workspace.Map` |
| `src/DevTools/Builders/BuildTraining.luau` | ServerStorage.DevTools.Builders.BuildTraining | Builder | planned | Part 6 | Training Grounds |
| `src/DevTools/Builders/BuildLighting.luau` | ServerStorage.DevTools.Builders.BuildLighting | Builder | planned | Part 6 | → `Lighting` children |
| `src/DevTools/Builders/BuildVfxTemplates.luau` | ServerStorage.DevTools.Builders.BuildVfxTemplates | Builder | planned | Part 6 | → `ReplicatedStorage.VFX` |
| `src/DevTools/Builders/BuildPlaceholderAnims.luau` | ServerStorage.DevTools.Builders.BuildPlaceholderAnims | Builder | planned | Part 6 | → `ReplicatedStorage.Anim` |
| `src/DevTools/Builders/BuildTemplates.luau` | ServerStorage.DevTools.Builders.BuildTemplates | Builder | planned | Part 6 | → `ServerStorage.Templates` |
| `src/DevTools/Tests/` | ServerStorage.DevTools.Tests | Folder | planned | Part 7 | In-Studio scenario modules (add rows as created) |

### 1.4 `StarterPlayer.StarterPlayerScripts` ↔ `src/StarterPlayerScripts/`
| Disk path | Studio path | Kind | Status | Owner | Notes |
|---|---|---|---|---|---|
| `src/StarterPlayerScripts/Main.client.luau` | StarterPlayer.StarterPlayerScripts.Main | Script/Client | planned | Part 2 | Prints `[ALG][BOOT] client ready` |
| `src/StarterPlayerScripts/Controllers/InputController.luau` | StarterPlayer.StarterPlayerScripts.Controllers.InputController | Module | planned | Part 3 | IAS contexts |
| `src/StarterPlayerScripts/Controllers/CameraController.luau` | StarterPlayer.StarterPlayerScripts.Controllers.CameraController | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/Controllers/RenderController.luau` | StarterPlayer.StarterPlayerScripts.Controllers.RenderController | Module | planned | Part 6* | Reacts to sim state transitions |
| `src/StarterPlayerScripts/Controllers/AnimController.luau` | StarterPlayer.StarterPlayerScripts.Controllers.AnimController | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/Controllers/VfxController.luau` | StarterPlayer.StarterPlayerScripts.Controllers.VfxController | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/Controllers/SfxController.luau` | StarterPlayer.StarterPlayerScripts.Controllers.SfxController | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/Controllers/HudController.luau` | StarterPlayer.StarterPlayerScripts.Controllers.HudController | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/Controllers/UiController.luau` | StarterPlayer.StarterPlayerScripts.Controllers.UiController | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/Controllers/AppearanceController.luau` | StarterPlayer.StarterPlayerScripts.Controllers.AppearanceController | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/Controllers/SettingsController.luau` | StarterPlayer.StarterPlayerScripts.Controllers.SettingsController | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/UI/Create.luau` | StarterPlayer.StarterPlayerScripts.UI.Create | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/UI/Theme.luau` | StarterPlayer.StarterPlayerScripts.UI.Theme | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/UI/Screens/Hud.luau` | StarterPlayer.StarterPlayerScripts.UI.Screens.Hud | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/UI/Screens/StyleWheel.luau` | StarterPlayer.StarterPlayerScripts.UI.Screens.StyleWheel | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/UI/Screens/Wish.luau` | StarterPlayer.StarterPlayerScripts.UI.Screens.Wish | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/UI/Screens/Shop.luau` | StarterPlayer.StarterPlayerScripts.UI.Screens.Shop | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/UI/Screens/Settings.luau` | StarterPlayer.StarterPlayerScripts.UI.Screens.Settings | Module | planned | Part 6 | |
| `src/StarterPlayerScripts/UI/Screens/Results.luau` | StarterPlayer.StarterPlayerScripts.UI.Screens.Results | Module | planned | Part 6 | |

## 2. Unsynced instance folders (never hand-edited)

| Studio path | Created by | Kind | Status | Owner | Notes |
|---|---|---|---|---|---|
| `ReplicatedStorage.Inputs` | BuildInputs | Folder | planned | Part 3 | InputContext templates; `AlgBuilder` attribute |
| `ReplicatedStorage.VFX` | BuildVfxTemplates | Folder | planned | Part 6 | VFX templates |
| `ReplicatedStorage.Anim` | BuildPlaceholderAnims | Folder | planned | Part 6 | Animation instances + placeholder KeyframeSequences |
| `ServerStorage.Templates` | BuildTemplates | Folder | planned | Part 6 | Dummy rig, kit props |
| `Workspace.Map` | BuildMap_CandyKingdom | Folder | planned | Part 6 | Built map |
| `Workspace.Arena` | BuildArena | Folder | planned | Part 6 | M0 duel arena |
| `Lighting` (children) | BuildLighting | — | planned | Part 6 | Owned children carry `AlgBuilder` |
| `ReplicatedStorage.Remotes` | Server at runtime | Runtime | planned | Part 2 | Must not exist in Edit |
| `ReplicatedStorage.Runtime` | Server at runtime | Runtime | planned | Part 2 | `Config` StringValue + `Version`; attribute `ServerReady` |
| `ServerStorage.AIProps` | Agent via MCP `generate_*` | Folder | planned | Part 1 | AI generic props only; provenance rows required |
| `ServerStorage.__AlgScratch` | `Util.freshTree` (transient) | Folder | n/a | Part 1 | Must never persist; `Verify` flags it |
| `ServerStorage.Quarantine` | Agent (scratch place only) | Folder | n/a | Part 1 | Never in the dev place |

## 3. Human-owned settings (not scriptable; verified by probes)

| Setting | Set in | H-step | Status | Probe |
|---|---|---|---|---|
| `Workspace.AuthorityMode = Server` (+ StreamingEnabled etc.) | Properties | H-04 | planned | `ping.luau` streamingEnabled; `client_status.luau` prediction |
| `StarterPlayer.AvatarJointUpgrade = Enabled` | Properties | H-04 | planned | `client_status.luau` animationConstraints > 0 |
| Avatar Settings: Consistent Gameplay + Single Collider (+ CCL if adopted) | Avatar Settings | H-05 | planned | Human attestation |
| Enable Studio Access to API Services (dev only) | Experience Settings ⟩ Security | H-06 | planned | DataStore probe |

## 4. Host-side files (not synced)

| Path | Kind | Status | Owner | Notes |
|---|---|---|---|---|
| `tests/run.luau` | Host | planned | Part 7 | Lune test runner; `[ALG][TEST]` lines |
| `tests/core/` | Host | planned | Part 3 / Part 7 | `*.spec.luau` |
| `tests/wish/` | Host | planned | Part 5 / Part 7 | `*.spec.luau`; distribution test (§0.11) |
| `tests/shim/` | Host | planned | Part 7 | Roblox datatype shims |
| `tools/check.luau` | Host | planned | Part 1 | Single host gate |
| `tools/lint_rules.luau` | Host | planned | Part 1 | Project rules incl. this file's coverage |
| `tools/sync_manifest.luau` | Host | planned | Part 1 | Disk side of the sync gate |
| `tools/lint_strings.luau` | Host | planned | Part 8 | IP/string lint |
| `tools/check_manifest.luau` | Host | planned | Part 7* | Asset manifest checks |
| `tools/export_framedata.luau` | Host | planned | Part 3* / Part 7 | Frame-data export |
| `tools/studio/` | Host | planned | Part 1 | `execute_luau` snippets ≤ 1,536 bytes |
| `tools/types/` | Host | planned | Part 1 | Vendored luau-lsp Roblox type definitions |
| `tools/plugin/AlgebraicTestHarness/` | Host | planned | Part 7 | Local Studio plugin for multi-client tests |
| `default.project.json` | Host | planned | Part 1 | Rojo project (sourcemap, CI build) |
| `rokit.toml`, `.luaurc`, `selene.toml`, `stylua.toml`, `.styluaignore`, `.gitignore`, `.gitattributes` | Host | planned | Part 1 | Toolchain config |
| `.claude/settings.json`, `.claude/skills/algebraic-studio/SKILL.md` | Host | planned | Part 1 | Claude Code config |
| `docs/agent/` | Host | planned | Part 1 | TASKS, PROGRESS, dev-place.json, MCP snapshot, provenance, skills, MCP example |
| `docs/THIRD_PARTY.md` | Host | planned | Part 2 | Vendored code + type definitions |
