[← Build spec index](../STUDIO_BUILD_SPEC.md) · **Part 3 of 8** · sections §3.x · Part 0 contracts (§0.x) are in the index file

> ⚠️ **DRAFT — pre-review snapshot.** An editorial/technical review of this spec found ~200 issues and the fixed
> v1.0 is still being prepared. **Agents: do not execute any task card beyond the P0 human-setup steps (H-01…H-12)
> until this banner is gone.** Humans can start the one-time setup in Part 1 §1.2 now.

## 3. Combat core and Server Authority integration

This part specifies the shared combat chassis (GDD §6) and how it runs under Roblox Server Authority (GDD §18.2). It owns `src/Shared/Core/*` (except per-Realm rule tables, §3.9.3), `src/Shared/Sim/*`, `src/Shared/Combat/*`, `Services/CombatService.luau`, `Services/BotService.luau` (with Part 7), `Builders/BuildInputs.luau`, the `Defaults.Combat` config section (§3.17) and the RenderController *contract* (§3.11.8; Part 6 implements it).

Reading order for the agent before any P2/P3 task: Part 0 → §2.1, §2.5, §2.6.6 → this part's §3.1, §3.3, §3.11 → the official SA pages (`projects/server-authority/index.md`, `techniques.md`) → `input/input-action-system.md`.

Conventions in this part:
- **Frames** are 60 Hz frames. `f` is the absolute simulation frame `Frames.fromTime(time())` (§3.1.3). `e` is the frame *elapsed in the current state*: `e = f - StF`. `e = 0` is the first frame of a state.
- **Frame-data convention.** For a move with `startup S`, `active A`, `recovery R`: startup is `e ∈ [0, S-1]`, active is `e ∈ [S, S+A-1]`, recovery is `e ∈ [S+A, S+A+R-1]`, and the state ends when `e ≥ S+A+R`. So "startup 6f" means the first frame that can hit is the 7th frame of the move. 1-indexed frame windows in the GDD (e.g. side-dash i-frames "frames 1–7") map to `e + 1 ∈ [1, 7]`.
- `(td)` = tuning default chosen by this part where the GDD gives no number; it satisfies the GDD rules cited next to it. Every number lives in `Defaults.Combat` (§3.17); prose cites the config key.
- "Verify" lines name a Lune spec (`tests/core/<file>.spec.luau`, test `<name>`), an in-Studio scenario (Part 7 harness, printed `[ALG][TEST] <name> PASS|FAIL`), an MCP call against `Roblox_Studio`, or an `[ALG]` log line.
- Every engine API named here was checked in the offline API reference (snapshot 2026-10-10) for existence, deprecation and the `simulationAccess` flag. Anything the docs cannot settle is marked **(verify in spike SA-nn)** and has a spike in §3.16.4.

---

### 3.1 Principles and hard rules

#### 3.1.1 Layering: one pure Core, two adapters

| Layer | Files | Runs | Rule |
|---|---|---|---|
| **Core** (pure) | `Shared/Core/Fsm, Frames, M1, Defense, Movement, Stun, Meter, Breakaway, Moves, Damage, Rng, Realm` | both peers + Lune | No `game`, no services, no Instances, no `time()`, no `os.*`, no `math.random`. Inputs: plain tables, numbers, strings, `Vector3`, `CFrame`. Config is always a parameter (§2.6.5). Deterministic (§3.4.3). |
| **Sim schema** (pure) | `Shared/Sim/Attr.luau` | both + Lune | Attribute names, types, defaults, writer classes (§3.3). |
| **Adapter** | `Shared/Sim/Adapter.luau`, `Shared/Sim/InputMap.luau`, `Shared/Combat/Query, Rewind, Hurtbox` | both | Translates Instances ↔ Core. Holds the roster (filled outside the sim). Only Simulation-Access APIs inside the step. |
| **Loop** | `Shared/Sim/Simulation.luau` | both | The one `RunService:BindToSimulation(fn, Enum.StepFrequency.Hz60, priority)` binding. |
| **Authority host** | `Services/CombatService.luau` | server | Spawning, config pinning, meta flush (KO, signals), respawn. |
| **Presentation** | `Controllers/RenderController` (+ Vfx/Sfx/Camera/Appearance); combat animation driver `Adapter.animate` (both peers, `BindToAnimation`) | client (render) / both (animation) | Reads attributes in `RenderStepped`; effects keyed on transitions; undoable (§3.11.8). |

`Config.NetModel` selects the adapter mode: `"ServerAuthority"` (default, this part) or `"Classic"` (§3.14). Core is byte-identical in both.

#### 3.1.2 "Movement is predicted; consequences are authoritative"

Every simulation attribute has a declared **writer class** (column `W` in §3.3.1). Composite classes such as `S·B` list every writer, the dominant one first:

| Class | Who writes | When | Misprediction profile |
|---|---|---|---|
| `B` (both) | The character's own FSM on **both** peers (owner client predicts; server is authoritative) | Every step, from inputs + attributes only | Only when an input arrives late or another class changed an input to the FSM |
| `BQ` (both, query-dependent) | Attacker's own FSM on both peers, from **its own** hit query (client: live, against replicated hurtboxes; server: rewound, §3.12) | On contact | When the client's view of the target differs from the server's rewound view (rare at ≤ 200 ms) |
| `S` (server) | Server sim step only | On consequences (hits, KO, realm membership, meter-full) | Always mispredicts once on the affected character's owner client (expected: SA docs "actions by other players will always trigger some amount of misprediction") |
| `S·B` / `B·S` / `B·BQ` / `BQ·S` | Combinations: e.g. the server sets on an event and the owner FSM advances or resets it deterministically on both peers | `StunF0` set by a hit (S), reset at stun exit (B) | Once per server event |

Rule: **the server writes a cause once; both peers derive per-frame effects from it.** Example: a hit writes `St=Ragdoll`, `StF`, `KbV` on the victim (S). Every frame after, the victim's FSM (B) computes the knockback velocity from `KbV` and `e`, on the victim's client and on the server. After the one rollback, the victim's client predicts the rest of the knockback correctly.

The client never writes another character's attributes. Its sim steps only the local character (§3.11.3).

#### 3.1.3 Hz60 binding and the frame clock

- `Simulation.start()` MUST call `RunService:BindToSimulation(fn, Enum.StepFrequency.Hz60, Combat.Chassis.Sim.Priority)` on both peers. The API default is `Hz30` [Docs `RunService.yaml`], which would halve all frame data.
- `f = Frames.fromTime(time())` with `Frames.fromTime(t) = math.floor(t * 60 + 0.5)`. Under `AuthorityMode.Server`, `time()` "is synchronized between client and server" and rewinds during resimulation [Docs `RobloxGlobals.yaml`, `server-authority/index.md`].
- `dt` must equal 1/60. If `math.abs(dt - 1/60) > 1e-4`, the step still runs; the server logs (outside the sim, once per session) `[ALG][Sim] WARN dt=<dt>`.
- **No exact-frame equality tests in Core.** Never branch on `f == X`. One-shot "entry" effects run in the step that performs the transition (§3.2.4), and window tests use `≥`/`<` on absolute frames. If the engine ever skips a frame, logic degrades gracefully. This also covers spike SA-01 failing on frame exactness.

**Verify:** scenario `sim.hz60_clock` (server and client each record 600 consecutive `f` values via a test probe; every delta = 1; the client's `f` for a given input-tagged step equals the server's). Log on bind: `[ALG][Sim] bound hz=60 priority=2000 side=<server|client>`.

#### 3.1.4 Attribute rules

1. Simulation state lives in attributes on each character's `HumanoidRootPart` (HRP), listed in `Shared/Sim/Attr.luau` (§3.3). No other HRP attributes are allowed.
2. HRP attributes are written **only inside the BindToSimulation callback** (`Instance:SetAttribute` is Simulation-Access [Docs `Instance.yaml`]). This includes the server-only `S` writes and the BotService intent attributes (§3.15.1).
3. Budget: ≤ 56 per HRP (Part 2 §2.5.1; SA replicates only the first 64 [Docs `server-authority/index.md#attribute-limits`]). Current use: 49 on players, 52 on bots. Names ≤ 12 chars (limit 50), ASCII alphanumerics only (no `.`). String values ≤ 50 chars.
4. Write only on change: `Adapter.writeDiff` compares the old and new `CharState` field by field and calls `SetAttribute` only for changed fields.
5. **Quantization (Q-rule).** Values written to attributes are integers, or are rounded: HP/guard/meter-like values to 0.1 (`math.floor(x * 10 + 0.5) / 10`), Vector3 components to 0.01, yaw to 0.001 rad. Trigonometric results never reach an attribute unrounded, because `math.sin`/`math.atan2` may differ in the last bit between the server CPU and an ARM phone, and SA compares attributes exactly.
6. The sim never reads `Player` attributes (Part 2 §2.5.1). Player-level data that the sim needs (style, pinned config) is cached in the roster entry outside the sim.

#### 3.1.5 Banned inside the simulation step (complete list)

"Inside the sim" means any code that runs during the BindToSimulation callback, including functions it calls, `task.spawn` from it (which keeps sim restrictions [Docs `server-authority/index.md`]) and Core.

| Banned | Why | Use instead |
|---|---|---|
| `os.clock`, `os.time`, `tick`, `DateTime.now`, `workspace.DistributedGameTime`, `workspace:GetServerTimeNow()` | Not synchronized, not rolled back | `f` from `time()`; `dt` |
| `math.random`, `Random.new()` without a sim-derived seed | Diverges between peers | `Core/Rng` seeded from `(f, rid, salt)` (§3.4.3) |
| `task.wait`, `task.delay`, `WaitForChild`, any yield, `pcall` around yielding calls | The step is synchronous | Countdown in attributes; `task.defer` to leave the sim |
| Reading `workspace.CurrentCamera`, `Camera.CFrame` | No camera on the server; replay needs the old value | `Aim` Direction3D input (§3.10) |
| `UserInputService`, `ContextActionService`, `InputAction.Pressed/Released/StateChanged`, `GuiButton.Activated` | Not replayed on resim | `InputAction:GetState()` + `PrevIn` bitmask |
| Any RemoteEvent/RemoteFunction/UnreliableRemoteEvent fire, `BindableEvent:Fire`, `Signal:Fire`, `Log.*`, `print`/`warn` | Effects double-fire on resim; remotes are unordered vs attributes | Server: append to the step's event queue, flush with one `task.defer` (§3.11.6). Client: nothing; render reacts to attributes. |
| Sounds, particles, `Highlight`, camera shake, `HapticEffect`, UI writes, `BasePart.Color/Transparency/LocalTransparencyModifier` | Effects; not Sim-Access | RenderController (§3.11.8) |
| `Instance:Destroy()`, `ClearAllChildren`, writing `Name`/`Parent` of an instance already in the DataModel, `AddTag`/`RemoveTag` | No Simulation Access [Docs `Instance.yaml`] | Flag attribute `Dead=true`; server destroys in a deferred pass (§3.11.5) |
| `Instance.new`/`Clone` **except** stitched spawns through `Adapter.spawn` (§3.11.5) | Non-idempotent across replays | Pre-wire at spawn (§3.11.2); stitched spawn helper |
| `Players:GetPlayers()`, `Players.LocalPlayer`, `Player.Character`, `Player.UserId`, `Player:GetAttribute` | No Simulation Access [Docs `Players.yaml`, `Player.yaml`] | Roster cache (§3.11.1) |
| `Humanoid.Health/MaxHealth/TakeDamage/PlatformStand`, `JointInstance.Enabled`, `Motor6D.C0/C1`, `AnimationTrack.Length`, `GetMarkerReachedSignal`, `GetTimeOfKeyframe` | No Simulation Access [api-sheet §1.5] | Attribute `HP`; frame data in config/MoveDefs |
| `RunService:IsServer()`/`IsClient()` called inside the step | Not flagged Simulation-Access | Capture `IS_SERVER` once at require time (§3.11.3) |
| Branching on the peer to compute a **different outcome for the same writer class** | The peers compute different worlds | Only the documented `S` / `BQ` split (§3.1.2) |
| Iterating dictionaries whose iteration order can matter (`pairs` over keyed tables) | Order is not guaranteed | Arrays sorted by `rid` |
| Module-level mutable Lua state on the **client** that influences outcomes | Does not roll back | Attributes or constraint properties |
| `AlignOrientation.PrimaryAxisOnly`, `Enum.HumanoidStateType.Ragdoll`, `Player:LoadCharacter`, `InputAction:Fire`, `PhysicsService` collision APIs, BodyMovers | Deprecated [Docs] | `AlignType`, our Ragdoll state, `LoadCharacterAsync`, `InputBinding:Fire`, `workspace:RegisterCollisionGroup`, `LinearVelocity`/`AlignPosition` |

**Verify:** Lune `standards.sim_banned_tokens` (Part 7 lint): it scans `src/Shared/Core/**`, `src/Shared/Sim/**`, `src/Shared/Combat/**` and `src/Shared/Behaviors/**` for the banned tokens above, with an allow-list for `Simulation.luau`'s single `IsServer()` at load and `Adapter.spawn`'s `Instance.new`.

#### 3.1.6 Simulation-Access APIs this part uses (whitelist)

All flagged `simulationAccess: true` in the snapshot: `Instance:GetAttribute/SetAttribute/GetAttributes/FindFirstChild/FindFirstChildWhichIsA/IsDescendantOf/HasTag/Clone`; `BasePart.CFrame/Position/Size/AssemblyLinearVelocity/AssemblyMass/CanCollide/CanQuery/CanTouch/CollisionGroup/Anchored`; `PVInstance:GetPivot/PivotTo`; `Humanoid.WalkSpeed/JumpHeight/UseJumpPower/AutoRotate/Jump/MoveDirection/FloorMaterial/Move/SetStateEnabled/GetStateEnabled/ChangeState/GetState`; `Constraint.Enabled/Attachment0/Attachment1`; every `LinearVelocity`, `AlignOrientation` (except the deprecated `PrimaryAxisOnly`) and `AlignPosition` property; `Attachment.CFrame/WorldCFrame`; `WorldRoot:GetPartBoundsInBox/GetPartBoundsInRadius/GetPartsInPart/Raycast/Blockcast/Spherecast`; `InputAction:GetState`; `InputBinding:Fire` (tests only); `RunService:IsResimulating/GetPredictionStatus`; `Players:GetPlayerFromCharacter`; `Workspace.Gravity`; `TweenService:SmoothDamp` (render only). `WorldRoot:Shapecast` is Sim-Access but Thread-Unsafe; we do not use it.

#### 3.1.7 Effects only on the render side

- VFX, SFX, camera shake, haptics, hitstop *visuals* and UI run in `RenderController` (`RunService.RenderStepped`), keyed on **attribute transitions** (§3.11.8).
- Combat **animations** are the one exception: following the SA guidance to mirror animation logic, they are a deterministic function of sim state, played from a `RunService:BindToAnimation(..., Hz60)` binding on both peers (§3.11.8).
- Any effect started for a *predicted* transition (the local character's own state, or `Cnt` contact) MUST be undoable within `Render.UndoWindowFrames` = 15 frames (constant in RenderController, not gameplay config).
- The render side never plays a one-shot while `RunService:IsResimulating()` is true. In practice RenderStepped does not run during resim; the guard is for property-changed handlers, which *do* re-fire during resimulation [Docs `RunService.yaml`]. So RenderController MUST NOT use `GetAttributeChangedSignal` for one-shots. It polls in RenderStepped and diffs against its own last-seen snapshot.

#### 3.1.8 Server-only Lua state

The server never rolls back. So server-only Lua tables (rewind rings, meter, damage ledgers, hit registries, realm list, roster latency) are legal inside the server branch of the step, **provided that no value derived from them is written to a `B`-class attribute** (that would make the server compute something the client cannot). They may feed `S`-class writes only.

---

### 3.2 Character combat state machine

#### 3.2.1 States (`Fsm.S`, attribute `St`)

| Id | Name | Kind | Duration (frames) | I-frames | `Sub` meaning | GDD |
|---|---|---|---|---|---|---|
| 1 | `Idle` | neutral, grounded | untimed | — | 0 | — |
| 2 | `Air` | neutral, airborne | untimed | — | 0 | — |
| 3 | `M1` | attack | `S+A+R` from §3.5 tables | — | 1–3 ground hit; 11–13 air hit; 20 Dash M1 | §6.3, §6.5 |
| 4 | `Ender` | attack | 10+4+22 | — | 1 Normal, 2 Uppercut, 3 Downslam, 4 Side-L, 5 Side-R | §6.3 |
| 5 | `Move` | attack | from `MoveDef` | per MoveDef (behavior) | 1–4 base slot, 5 Quirk, 11–14 awakened slot | §6.7 |
| 6 | `Block` | defense | untimed (held) | — | 0 | §6.4 |
| 7 | `BlockStun` | defense | `StDur` (§3.6.4) | — | 0 | §6.4 |
| 8 | `Dash` | mobility | `Dash.<Kind>.Frames` | frames `IFrom..ITo` | 1 Front, 2 Side-L, 3 Side-R, 4 Back, 5 Air | §6.5 |
| 9 | `TechRoll` | mobility | `Dash.TechRoll.IFrames` = 18 | all 18 | 0 | §6.5 #4 |
| 10 | `Hitstun` | stun | `StDur` | — | 0 grounded, 1 airborne | §6.6 |
| 11 | `Stagger` | stun | `StDur` | — | 1 Perfect-Block victim, 2 push (Breakaway/Awakening/shockwave), 3 melee bounce (kit) | §6.4, §6.8, §6.9 |
| 12 | `Launched` | stun, airborne | rise + hang (§3.5.3) | — | 0 | §6.3 |
| 13 | `Ragdoll` | stun | `StDur` (60 Ender / 54 Downslam / move) | — | 0 = not yet landed; `n` = `e+1` at first landing | §6.3, §6.6 |
| 14 | `WallSplat` | stun | 30 | — | 0 | §6.3 |
| 15 | `GuardBroken` | stun | 84 | — | 0 | §6.4 |
| 16 | `Grabbed` | stun | set by the grab behavior (Part 4) | — | grab kind (Part 4) | §6.4, §6.7 |
| 17 | `Breakaway` | escape | 20 | all 20 | 0 | §6.8 |
| 18 | `ShakeOff` | escape | 18 | all 18 | 0 | §6.6 |
| 19 | `Awakening` | special | 72 | all 72 | 0 | §6.9 |
| 20 | `RealmCast` | special | 90 (+ queue, §3.9.3) | — | awakened slot (14) | §6.10 |
| 21 | `KO` | terminal | until respawn | all | 0 | §6.2, §6.11 |

Sets used by the rules (`Fsm.SET`):
- `NEUTRAL = {Idle, Air}`.
- `STUN = {Hitstun, Stagger, Launched, Ragdoll, WallSplat, GuardBroken, Grabbed}`: Breakaway-eligible states and the states that keep a combo "continuous" (§3.8.1).
- `ATTACK = {M1, Ender, Move, RealmCast}`: starting one breaks spawn protection and wake-up i-frames.
- "Respawning" is not an HRP state. After `KO`, the body stays (invisible after `Respawn.BodyHideSec`) until `LoadCharacterAsync` replaces it (§3.13.4).
- Run is a locomotion modifier inside `Idle` (walk speed 26), not a state. Render reads `Humanoid.MoveDirection` and speed.

**Invulnerability** (`Fsm.isInvulnerable(s, f, ctx, cfg)`) is true when any of these hold: `St ∈ {TechRoll, Breakaway, ShakeOff, Awakening, KO}`; `St = Dash` and `IFrom ≤ e+1 ≤ ITo`; `f < InvF` (wake-up, §3.8.6); `f < ProtF` (spawn protection, §3.13.5); a Move's behavior declares i-frames for `e` (Part 4 `params.iFrames = {from, to}`).

#### 3.2.2 Inputs accepted per state

`✓` = acts now. `B` = buffered (§3.2.5). `✗` = ignored. `*` = special rule in the referenced section. Hold-type inputs (Block, Jump-hold, Run, AimLock, Launcher, Aim) are read as levels and never buffered.

| State | M1 | Move 1–4 / Quirk | Block (hold) | Dash | Jump | Awaken |
|---|---|---|---|---|---|---|
| Idle | ✓ | ✓ | ✓ | ✓ | ✓ (jump) | ✓ |
| Air | ✓ (air rules §3.5.3) | ✓ if `MoveDef.params.air` | ✗ | ✓ Air dash if allowed (§3.7.2) | ✗ | ✓ |
| M1 | B (whole recovery, §3.5.1) | B | ✗ | `*` Hop Cancel (§3.2.6) else B | ✗ | ✗ |
| Ender | B (only during Uppercut hang) | B | ✗ | B | `*` Uppercut select at entry | ✗ |
| Move | `*` `params.cancelInto` | B | ✗ | B | ✗ | ✗ |
| Block | ✗ | ✗ | (release → Idle) | ✓ (dash out of block) | ✗ | ✓ |
| BlockStun | B | B | (re-press sets `BlkF`, §3.6.3) | B | ✗ | ✗ |
| Dash | `*` Dash M1 (front only) else B | B | ✗ | ✗ | ✗ | ✗ |
| TechRoll | B | B | ✗ | B | ✗ | ✗ |
| Hitstun / Stagger / Launched / WallSplat | ✗ | ✗ | ✗ | `*` Breakaway (§3.9.1) | ✗ | ✗ |
| Ragdoll | ✗ | ✗ | ✗ | `*` Breakaway | `*` Tech Roll (§3.8.6) | ✗ |
| GuardBroken | ✗ | ✗ | ✗ | `*` Breakaway after 30f | ✗ | ✗ |
| Grabbed | ✗ | ✗ | ✗ | `*` Breakaway | ✗ | ✗ |
| Breakaway / ShakeOff | B | B | ✗ | B | ✗ | ✗ |
| Awakening / RealmCast / KO | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |

Status effects (§3.4.2, Moves status) further gate: `rooted` blocks Dash and walking, `silenced` blocks Move/Quirk, `blockDelayF` delays Block activation.

#### 3.2.3 Transition table

Evaluated by `Fsm.step` in this priority order (lower number wins). Within one step at most **one** input-driven transition fires. Timer and forced transitions are evaluated first. A state entered this step is not re-evaluated until the next step.

| P | From | Trigger and guard | To | Entry effects (same step) |
|---|---|---|---|---|
| 0 | any ≠ KO | `HP ≤ 0` (server consequence, §3.13.1) | `KO` | movers off, grab detached; event `ko` |
| 1 | any | Hit consequence (server, §3.12.6) | per `Damage.resolve` | per §3.6–§3.8 |
| 2 | any `STUN` | `StunF0 > 0` and `f - StunF0 ≥ Stun.ShakeOffFrames` (210) | `ShakeOff` | push (§3.8.3); reset combo (§3.8.1) |
| 3 | timed state | `e ≥ StDur` (or the frame-data total) | successor (below) | successor entry effects |
| 4 | `Ragdoll` (Ender KB) | wall probe hits (§3.5.5), `e < M1.WallSplat.WindowFrames`, `CFlags` bit WallSplat clear | `WallSplat` | `CFlags \|= WALLSPLAT`; motion freeze |
| 5 | `STUN` (GuardBroken only if `f ≥ BkLockF`) | `Dash` pressed; `Breakaway.canUse` | `Breakaway` | `BkP -= 1`; detach grab; event `breakaway` (§3.9.1) |
| 6 | `Ragdoll` | `Jump` pressed; `Sub > 0`; `(e+1) - Sub < Dash.TechRoll.WindowFrames` (12) | `TechRoll` | reset combo |
| 7 | `NEUTRAL`, `Block` | `Awaken` pressed; `AwkRdy`; `AwkF0 == 0` | `Awakening` | §3.9.2 activation effects |
| 8 | `M1` recovery, `Sub ∈ {2,3}`, `Cnt ≠ 0` | `Dash` pressed with Side direction; `f ≥ CdDS` | `Dash` (Side) | `M1EndF = f` (chain stays alive) |
| 9 | `Dash` Front | `M1` pressed | `M1` Sub 20 (Dash M1) | `ProtF = 0`; lunge |
| 9b | `NEUTRAL` | `M1` pressed; `f ≤ frontDashStart + Front.Frames + M1.DashM1.WindowFrames`, where `frontDashStart = CdDF − toFrames(Dash.Front.CooldownSec)` (derived; no attribute) | `M1` Sub 20 | as above |
| 10 | `NEUTRAL`, `Block` (not `rooted`) | `Dash` pressed; kind per §3.7.1; that kind's cooldown ready | `Dash` | set `Cd<kind>`; `Flg \|= AIRDASH` for Air |
| 11 | `NEUTRAL` | Move/Quirk pressed; `Moves.canUse` | `Move` or `RealmCast` | set `Cd<slot>`; `ProtF = 0`; `InvF = 0` (§3.8.6); behavior `onStart` |
| 12 | `NEUTRAL` | `M1` pressed; `f ≥ M1LockF` | `M1` or `Ender` (§3.5.1) | `M1Idx`/`AirN` update; `ProtF = 0`; `InvF = 0` |
| 13 | `Idle` | `Block` held; `f ≥ BlkLockF`; grounded | `Block` | `BlkF = f` if pressed this step; materialize guard (§3.6.2) |
| 14 | `Block` | `Block` released | `Idle` | anti-mash check (§3.6.3) |
| 15 | `Idle` ↔ `Air` | grounded sensor changes | `Air` / `Idle` | landing: `Flg &= ~AIRDASH`; deferred end lag (§3.5.3) |

Successors at timer expiry (P3):

| State | Successor |
|---|---|
| M1, Ender, Move, Dash, TechRoll, Breakaway, ShakeOff, Awakening, Stagger, Hitstun | `Idle` if grounded else `Air`; M1/Ender also run the chain-exit bookkeeping (§3.5.1) |
| BlockStun | `Block` if `Block` held else `Idle` |
| Launched | `Air` (gravity resumes; victim stays in combo only if hit again) |
| Ragdoll | `Idle` with `InvF = f + Ragdoll.WakeUpIFrames` (30) |
| WallSplat | `Ragdoll` with `StDur = M1.WallSplat.AfterRagdollFrames` (30), `Sub = 1` (already grounded or sliding) |
| GuardBroken | `Idle` |
| Grabbed | set by the grab behavior's release (Part 4); fallback after `StDur`: `Idle` |
| RealmCast | `Idle` (server opens the Realm at `e = 90`, §3.9.3) |

P0 (KO) and P1 (hit consequences) are applied by the server adapter (phases 3–4, §3.11.3), never by `Fsm.step`; the FSM sees their results in the attributes on its next step. P2 onward are computed by `Fsm.step` on both peers.

#### 3.2.4 Step order inside `Fsm.step`

```
1. e := f - StF. If e < 0 (pending after hitstop, §3.5.6): no transitions; motion = freeze; go to 7.
2. Forced/timer transitions (P2, P3, P4).
3. Read input edges: pressed = bits & ~PrevIn. Merge the buffered input (QIn, QInF) if still valid.
4. Input-driven transition by priority (P5–P14). On transition: run entry effects; clear QIn if consumed.
5. If no transition consumed a pressed buffered-eligible action, buffer the highest-priority pressed action (§3.2.5).
6. State tick: compute MotionCmd, FacingCmd, LocoCmd, HitRequests (active frames), behavior hook calls.
7. Periodic ticks (§3.8.7 regen, §3.9.1 Breakaway refill, §3.6.2 guard is lazy). Status expiry.
8. PrevIn := bits.
```

#### 3.2.5 Input buffering

- Buffered actions and codes (`QIn`): 1 M1, 2–5 Move1–4, 6 Quirk, 7 Awaken, 8 Dash (with direction resolved at execution).
- A press that cannot act in the current state is stored: `QIn = code`, `QInF = f`. One slot only: the latest press wins (priority order Awaken > Dash > Move/Quirk > M1 when several are pressed in one frame).
- At step 3, if `QIn ≠ 0` and `f - QInF ≤ Combat.Chassis.Input.BufferFrames` (6, td), the buffered action is treated as pressed this frame. Otherwise `QIn = 0`.
- Exception (§3.5.1): an M1 pressed during M1 recovery stays buffered until the recovery ends, regardless of age (`M1.BufferWholeRecovery = true`, td). This keeps mashing reliable while respecting the 36f chain window.
- Buffered presses are discarded on entering any `STUN` state.

**Verify:** Lune `fsm.buffer_6f` (Dash pressed 6f before Dash-able → executes; 7f → dropped), `fsm.buffer_latest_wins`, `fsm.buffer_cleared_on_stun`.

#### 3.2.6 Cancels

| Cancel | Rule | GDD |
|---|---|---|
| **Hop Cancel** | During `M1` recovery with `Sub ∈ Dash.HopCancelFromHits` ({2,3}) and `Cnt ∈ {1 hit, 2 blocked}`, a Side dash press (direction classified Side, §3.7.1) cancels into `Dash` Side if `f ≥ CdDS`. Front/Back dashes cannot cancel. | §6.5 |
| **Dash-Ender** | Front dash → `M1` pressed during the dash or ≤ `M1.DashM1.WindowFrames` (8, td) after it ends → Dash M1 (Sub 20). On exit, `M1Idx = 1`, so the next chain hit is M1-2. | §6.5 |
| **Tech Roll** | `Jump` pressed within 12f of the first landing during `Ragdoll` → `TechRoll` (18f i-frames, 6-stud roll in the input direction, td). Ends the combo. | §6.5, §6.6 |
| **Back dash** | Cancels nothing; usable only from `NEUTRAL`/`Block`. | §6.5 |
| **Uppercut recovery** | On Uppercut contact, the Ender's recovery is skipped (`M1.Uppercut.HitCancelsRecovery = true`, td) so the 48f hang fits 3 air M1s and a Downslam. | §6.3 |
| **Move → M1** | Only if the MoveDef has `params.cancelInto = {"m1"}` and `params.cancelWindow = {a, b}` (move-relative `e`); the M1 continues the chain at `params.cancelChainIndex` (default 2). Default: no cancels. | App. A |
| **Breakaway** | Cancels any `STUN` state (§3.9.1). | §6.8 |

#### 3.2.7 Locomotion and facing per state

`LocoCmd = {walkSpeed, canJump, jump}` and `FacingCmd` (which also decides `Humanoid.AutoRotate`) are produced by `Movement.loco` and `Movement.facing` and applied by the adapter every step (writes skipped when unchanged).

| State | WalkSpeed | Jump | Facing |
|---|---|---|---|
| Idle | 18 (`Chassis.WalkSpeed`), 26 when `Run`/`RunAuto` held and moving (`Chassis.RunSpeed`) | yes, height 7.5 (`Chassis.JumpHeight`) | `AimLock` held → face aim yaw; else `auto` (Humanoid.AutoRotate) |
| Air | same as Idle (air control) | no | same as Idle |
| M1 / Ender / Move (startup) | `Chassis.StateWalkSpeed.M1` 6 / `.Ender` 0 / `.Move` 0 (td; a behavior may override) | no | track aim yaw |
| M1 / Ender / Move (active, recovery) | same | no | `hold` (locked at the last startup yaw) |
| Block | 8 (`Block.WalkSpeed`) | no | track aim yaw (the front 180° follows the camera) |
| BlockStun | 0 | no | hold |
| Dash / TechRoll | 0 (mover drives) | no | hold at the start yaw (front dash: start yaw = aim yaw) |
| STUN states, ShakeOff, Breakaway | 0 | no | hold (Grabbed: face the grabber, set by Part 4) |
| Awakening / RealmCast / KO | 0 | no | hold |

Status multipliers apply on top: `walkSpeed *= Π statusSpeedMult`, `rooted → 0`, Kaiju-style `ChassisMods.walkSpeedMult` (§3.4.2).

Facing implementation (§3.11.2): `auto` → `Humanoid.AutoRotate = true`, `ALG_Face.Enabled = false`. `set yaw`/`hold` → `AutoRotate = false`, `ALG_Face.Enabled = true`, `ALG_Face.CFrame = CFrame.Angles(0, yaw, 0)` (unchanged for `hold`). Yaw from a direction `d`: `yaw = round3(math.atan2(-d.X, -d.Z))`. The stored `ALG_Face.CFrame` *is* the facing state: a predicted, Sim-Access constraint property that rolls back, so no attribute is needed.

---

### 3.3 Attribute schema (`Shared/Sim/Attr.luau`)

#### 3.3.1 HRP simulation attributes (canonical, complete)

`W` = writer class (§3.1.2); `B·S` means the owner FSM writes it on both peers *and* the server writes it as a consequence. Frame values are absolute frames `f`. `0` in a `…F` field means "never/none".

| # | Name | Type | Default (init step) | W | Purpose | Spec |
|---|---|---|---|---|---|---|
| 1 | `St` | number | 1 (`Idle`) | B·S | State id (§3.2.1) | §3.2 |
| 2 | `StF` | number | `f` at init | B·S | Frame the state started (shifted forward by hitstop) | §3.5.6 |
| 3 | `StDur` | number | 0 | B·S | Duration of timed states whose length is set by a hit/kit (0 = from frame data) | §3.2.3 |
| 4 | `Sub` | number | 0 | B·S | Per-state sub-code (§3.2.1) | §3.2.1 |
| 5 | `PrevIn` | number | 0 | B | Previous-frame input bitmask (`Fsm.IN`) | §3.10.4 |
| 6 | `QIn` | number | 0 | B | Buffered action code | §3.2.5 |
| 7 | `QInF` | number | 0 | B | Frame of the buffered press | §3.2.5 |
| 8 | `M1Idx` | number | 0 | B | Last ground chain index started (0–4) | §3.5.1 |
| 9 | `M1EndF` | number | 0 | B | Frame the last M1/Dash M1 state exited (chain window base) | §3.5.1 |
| 10 | `M1LockF` | number | 0 | B·BQ | M1 locked until (Ender end lag) | §3.5.2 |
| 11 | `AirN` | number | 0 | B | Air M1s used in the current hang | §3.5.3 |
| 12 | `HangF` | number | 0 | BQ·S | Hang window end (attacker: BQ; launched victim: S) | §3.5.3 |
| 13 | `Cnt` | number | 0 | BQ·S | Contact on the current action: 0 none, 1 hit, 2 blocked, 3 perfect-blocked (S) | §3.12.6 |
| 14 | `HsF` | number | 0 | BQ·S | Hitstop: frozen while `f ≤ HsF` | §3.5.6 |
| 15 | `CombatF` | number | 0 | BQ·S | Last frame this character dealt (BQ) or took (S) damage | §3.8.7 |
| 16 | `HP` | number | style `hp` (100) | S·B | Hit points (damage S; regen/heal B) | §3.13 |
| 17 | `Grd` | number | 100 | S·B | Guard value at `GrdF` | §3.6.2 |
| 18 | `GrdF` | number | 0 | S·B | Guard reference frame (last block hold or guard change) | §3.6.2 |
| 19 | `BlkF` | number | −1000 | B | Frame Block was last pressed (Perfect Block window) | §3.6.3 |
| 20 | `BlkLockF` | number | 0 | B | Block locked until (anti-mash) | §3.6.3 |
| 21 | `CdDF` | number | 0 | B | Front dash ready at | §3.7.2 |
| 22 | `CdDS` | number | 0 | B | Side dash ready at | §3.7.2 |
| 23 | `CdDB` | number | 0 | B | Back dash ready at | §3.7.2 |
| 24–27 | `Cd1` `Cd2` `Cd3` `Cd4` | number | 0 | B·S | Move slot 1–4 ready at (base or awakened set; reset at Awakening start/end) | §3.9.2 |
| 28 | `CdQ` | number | 0 | B·S | Quirk ready at | §3.2.3 |
| 29 | `MvA` | number | 0 | B | Behavior scratch A (Part 4) | §3.11.4 |
| 30 | `MvB` | number | 0 | B | Behavior scratch B (Part 4) | §3.11.4 |
| 31 | `Kit` | number | 0 | B | Style-persistent state (e.g. a Quirk toggle, hover frames; Part 4) | §3.11.4 |
| 32 | `StunF0` | number | 0 | S·B | Frame the current continuous stun began (0 = not stunned) | §3.8.1 |
| 33 | `CHits` | number | 0 | S·B | Hits taken in the current combo | §3.8.4 |
| 34 | `CFlags` | number | 0 | S·B | Combo flags (table below) | §3.8 |
| 35 | `KbV` | Vector3 | (0,0,0) | S | Knockback initial velocity (studs/s), also the push direction | §3.8.5 |
| 36 | `InvF` | number | 0 | B | Generic i-frames until (wake-up) | §3.8.6 |
| 37 | `PileF` | number | 0 | S | Pile-on protection until | §3.8.4 |
| 38 | `ProtF` | number | `f + 480` | S·B | Spawn protection until (S at spawn; B clears on attack) | §3.13.5 |
| 39 | `BkP` | number | 2 | B·S | Breakaway pips (0–2) | §3.9.1 |
| 40 | `BkProg` | number | 0 | B·S | Refill progress toward the next pip, in frames (0–1500) | §3.9.1 |
| 41 | `BkLockF` | number | 0 | S | Breakaway unavailable until | §3.9.1 |
| 42 | `AwkRdy` | boolean | false | S·B | Algebraic Meter is full (S sets; B clears on activation) | §3.9.2 |
| 43 | `AwkF0` | number | 0 | B | Awakening start frame (0 = not awakened) | §3.9.2 |
| 44 | `AwkEndF` | number | 0 | B·S | Awakening end frame (S extends on KO) | §3.9.2 |
| 45 | `Buffs` | string ≤ 50 | "" | B·S | Encoded self-status list (§3.4.2 status codec) | §3.4.2 |
| 46 | `Debuffs` | string ≤ 50 | "" | S·B | Encoded debuff list (S applies; B expires) | §3.4.2 |
| 47 | `RlmId` | number | 0 | S | Realm membership id (0 = none) | §3.9.3 |
| 48 | `RlmWardF` | number | 0 | B·S | Stubborn Heart ward until; > 0 also marks "used in this Realm" | §3.9.3 |
| 49 | `Flg` | number | 0 | B·S | Self flags (table below) | — |
| 50 | `BIn` | number | 0 | bot (S) | **Bots only.** Intent input bits (same layout as `PrevIn`) | §3.15.1 |
| 51 | `BAim` | Vector3 | (0,0,−1) | bot (S) | **Bots only.** Aim direction (unit) | §3.15.1 |
| 52 | `BMv` | Vector3 | (0,0,0) | bot (S) | **Bots only.** World move direction (magnitude ≤ 1) | §3.15.1 |

Totals: 49 on player characters, 52 on bots; headroom 4 (≤ 56, Part 2 §2.5.1). Part 4 may claim headroom only through its "New contracts" and by updating `Attr.SPECS` (the Lune budget test fails above 56).

`CFlags` bits (victim combo; reset with the combo):

| Bit | Name | Meaning |
|---|---|---|
| 1 | `OTG` | The one OTG hit of this combo was used (GDD §6.6) |
| 2 | `DOWNSLAM` | A Downslam already landed in this combo (GDD §6.3) |
| 4 | `WALLSPLAT` | A Wall Splat already happened in this combo (GDD §6.3) |
| 8 | `ENDERKB` | The current Ragdoll came from an Ender (wall-splat eligible) |
| 16 | `SPIKE` | The current Ragdoll is a Downslam spike (bounce on landing) |

`Flg` bits (self):

| Bit | Name | W | Meaning |
|---|---|---|---|
| 1 | `AIRDASH` | B | Air dash used this airtime (cleared on landing) |
| 2 | `REALMQ` | S | Realm cast queued (§3.9.3) |
| 4 | `PBHIT` | S·B | A Perfect Block happened since the last Block press (S sets; B clears on press) |
| 8 | `FOLLOW` | BQ | Uppercut follow rise active (attacker) |
| 16 | `ENDLAG` | BQ | Ender end lag deferred until landing/Downslam end (Uppercut) |

#### 3.3.2 `Attr.luau` skeleton

```lua
--!strict
-- src/Shared/Sim/Attr.luau — canonical HRP simulation attributes. Pure (no Roblox globals at load). Spec §3.3.
export type AttrType = "number" | "boolean" | "string" | "Vector3"
export type Writer = "B" | "BQ" | "S" | "B·S" | "S·B" | "B·BQ" | "BQ·S" | "bot"
export type Spec = { name: string, type: AttrType, default: any, writer: Writer, botOnly: boolean? }

local MAX = 56
local SPECS: { Spec } = {
	{ name = "St", type = "number", default = 1, writer = "B·S" },
	{ name = "StF", type = "number", default = 0, writer = "B·S" },
	-- ... one row per §3.3.1 entry, in table order ...
	{ name = "KbV", type = "Vector3", default = { 0, 0, 0 }, writer = "S" }, -- arrays: no Vector3 at load (Lune)
	{ name = "BMv", type = "Vector3", default = { 0, 0, 0 }, writer = "bot", botOnly = true },
}

local N: { [string]: string } = {}
for _, s in SPECS do
	N[s.name] = s.name
end

local Attr = {}
Attr.MAX = MAX
Attr.SPECS = table.freeze(SPECS)
Attr.N = table.freeze(N) -- N.St == "St": typo-safe constants for adapters and render

function Attr.validate(): (boolean, { string })
	local problems: { string } = {}
	local seen: { [string]: boolean } = {}
	for _, s in SPECS do
		if #s.name > 50 or string.find(s.name, "[^%w]") then table.insert(problems, "bad name " .. s.name) end
		if seen[s.name] then table.insert(problems, "dup " .. s.name) end
		seen[s.name] = true
	end
	if #SPECS > MAX then table.insert(problems, `count {#SPECS} > {MAX}`) end
	return #problems == 0, problems
end

return table.freeze(Attr)
```

`Adapter` owns the Instance side: `Adapter.readState(root: BasePart): Fsm.CharState` (one `GetAttributes()` call, Sim-Access), `Adapter.writeDiff(root, old, new)` (Q-rule rounding, then `SetAttribute` only for changed fields), and `Adapter.initState(entry, f)` (server only, §3.11.3).

#### 3.3.3 Player attributes added by Part 3

Written by CombatService on the server **outside** the sim (deferred flush, §3.11.6). Part 2 §2.5.2 allows combat mirrors; the Player attribute total becomes 19 (limit 40).

| Attribute | Type | Default | Writer / when | Purpose |
|---|---|---|---|---|
| `Meter` | number (int 0–100) | 0 | CombatService flush, after any step that changed the meter | HUD meter ring (GDD §15.3); Analyze-style reveals (Part 4) |
| `Streak` | number (int ≥ 0) | 0 | CombatService flush on KO/death | Streak aura (GDD §6.11), nameplate |

#### 3.3.4 Server memory (never replicated)

Per roster entry, inside `Adapter` (server branch) and `CombatService`: `meter: number` (float), `rttEma: number`, `rewindFrames: number`, `ring: Rewind.Ring`, `hitReg: {[key: string]: {[rid: number]: boolean}}` (cleared on state entry), `ledger: { {rid: number, amount: number, f: number} }` (damage taken, last 600 frames), `lastHit: {rid: number, f: number}?`, `pileHits: { {rid: number, f: number} }`, `life: LifeStats`, `spawnCf: CFrame?`, `queuedPatches: {{[string]: any}}` (dev/test seam). Per player across lives: `streak: number`. Global: `realms: {RealmInstance}`, `realmQueue: {number}` (rids), `metaQueue: {SimEvent}`.

#### 3.3.5 Placement decisions (where each value lives and why)

| Value | Home | Justification |
|---|---|---|
| HP | HRP `HP` | The sim needs it (KO, Finn's below-30 passive, Awakening heal, regen). Damage is a victim-side consequence; the victim mispredicts once per hit anyway because its state changes. Nameplates read it from streamed characters, so no Player HP mirror is needed. |
| Algebraic Meter | server memory + Player `Meter`; HRP `AwkRdy` flag | Meter gains depend on server-only damage numbers (scaling, pile-on, Realm ×0.5). On the HRP, every hit would also mispredict the **attacker**. The sim only needs "full or not", so the server writes `AwkRdy=true` once per fill: one misprediction per fill instead of one per hit. (GDD §18.2 lists `Meter` among attributes; this is a deliberate refinement, see §3.19.) |
| Breakaway pips and refill | HRP `BkP`, `BkProg`, `BkLockF` | The owner's own input (Dash while stunned) gates on them, so they must be predicted. Refill ticks are deterministic (B). Damage refunds and lockout are victim consequences (S). Pips appear on enemy nameplates (GDD §6.8) through replication. |
| Guard | HRP `Grd`, `GrdF` (lazy regen) | The block decision and the guard bar need it. It changes only on hits (S) and block holds (B). Lazy evaluation avoids per-frame writes. |
| Cooldowns | HRP `Cd*` | Owner input gating; deterministic (B). Realm cooldown multipliers are S (§3.9.3). |
| Combo counters, pile-on result | HRP `CHits`, `CFlags`, `StunF0`, `PileF` | Victim FSM reads them for Shake-Off, wake-up and OTG rules; the server writes them on hits. |
| Pile-on attacker list, KO credit, assists, streak | server memory | Meta-only; needs identities. |
| Facing | `ALG_Face.CFrame` (constraint property) | Predicted, Sim-Access, rolls back; no attribute needed. |
| Grab attachment | `ALG_Hold.Attachment1/Enabled` (constraint properties) | Same. |
| Hurtbox pose | Hurtbox part `CFrame`/`Size`, server-written | Clients read replicated poses; the server keeps the 60-frame ring in memory. |

**Verify:** Lune `attr.budget` (`Attr.validate()` ok; ≤ 56; names ≤ 12 chars alphanumeric); Lune `attr.defaults_match_fsm` (`Fsm.initial(100, 0, cfg)` keys == player SPECS names); scenario `sim.attr_budget` (on a live player HRP and a bot HRP after spawn: `#GetAttributes()` == 49 / 52, every SPEC present with the right `typeof`); scenario `security.no_private_attrs` (Part 2) still passes.

---

### 3.4 Pure Core API

#### 3.4.1 Shared Core types (exported from `Core/Fsm.luau`)

```lua
--!strict
-- src/Shared/Core/Fsm.luau (types excerpt). Pure. Spec §3.4.1.
local Types = require("../Types")

export type CharState = {
	St: number, StF: number, StDur: number, Sub: number,
	PrevIn: number, QIn: number, QInF: number,
	M1Idx: number, M1EndF: number, M1LockF: number, AirN: number, HangF: number,
	Cnt: number, HsF: number, CombatF: number,
	HP: number, Grd: number, GrdF: number, BlkF: number, BlkLockF: number,
	CdDF: number, CdDS: number, CdDB: number,
	Cd1: number, Cd2: number, Cd3: number, Cd4: number, CdQ: number,
	MvA: number, MvB: number, Kit: number,
	StunF0: number, CHits: number, CFlags: number, KbV: Vector3, InvF: number, PileF: number, ProtF: number,
	BkP: number, BkProg: number, BkLockF: number,
	AwkRdy: boolean, AwkF0: number, AwkEndF: number,
	Buffs: string, Debuffs: string,
	RlmId: number, RlmWardF: number,
	Flg: number,
}

export type InputFrame = {
	bits: number,       -- level of every Bool action this frame (Fsm.IN layout)
	pressed: number,    -- bits & ~PrevIn (rising edges)
	aim: Vector3,       -- unit aim direction (camera look, assisted) — Aim action or BAim
	moveDir: Vector3,   -- world move direction, |v| <= 1 (Humanoid.MoveDirection or BMv)
}

export type ChassisMods = {           -- resolved per step by Moves.chassisMods (§3.4.2)
	breakawayRefillMult: number,      -- 1 default; Finn below 30 HP: 1.5 (GDD §7.3.1)
	enderDamageAdd: number,           -- 0 default; Finn below 30 HP: 2
	ragdollFramesAdd: number,         -- 0 default; Jake: -10 (GDD §7.3.2)
	m1RangeAdd: number,               -- 0 default; Finn awakened: 3 (GDD §6.9)
	hurtboxScale: number,             -- 1 default; >= 1 only (GDD §6.2)
	walkSpeedMult: number,            -- 1 default; Kaiju: 0.85 (GDD §7.3.2)
	airDash: boolean,                 -- style has the Aerial capability (GDD §6.5)
	cooldownMult: number,             -- 1 default; Realm rule may set 0.7 (GDD §7.3.15)
}

export type StyleRuntime = {          -- built once per life by Adapter.styleRuntime(def) outside the sim
	id: Types.StyleId, hp: number,
	base: { Types.MoveDef },          -- index 1..4 = slots "1".."4"
	awakened: { Types.MoveDef },      -- index 1..4 = slots "a1".."a4"
	quirk: Types.MoveDef,
	passiveParams: { [string]: any }, -- StyleDef.passive.params
	awakenParams: { [string]: any },  -- AwakeningDef.modifiers + hurtboxScale
}

export type HitDef = {
	moveId: string,                   -- "<style>.m1" for chassis M1s, MoveDef.id otherwise
	kind: "m1" | "ender" | "airm1" | "dashm1" | "move" | "projectile" | "zone" | "grab",
	damage: number, guardDamage: number, chip: number,
	hitstun: number, blockstun: number, hitstop: number,
	knockback: Types.KnockbackDef?,
	enderVariant: number,             -- 0 unless kind == "ender" (Sub code)
	parryable: boolean, unblockable: boolean, melee: boolean,
	groundOnly: boolean,              -- ground grabs miss airborne victims (GDD §6.4)
	tick: boolean,                    -- zone/DoT tick: no hitstun, no hitstop, no combo count
	realm: boolean,                   -- Realm damage (meter ×0.5, GDD §6.9)
	meterGain: number,                -- MoveDef.meterGain or 1
	signature: boolean,
	cinematicLockF: number,           -- Breakaway lock for awakened ultimates (<= 120, GDD §6.8)
}

export type HitboxReq = { shape: "Box" | "Sphere" | "Cast", cf: CFrame, size: Vector3, prevCf: CFrame? }
export type HitRequest = { key: string, box: HitboxReq, hit: HitDef, maxTargets: number }
export type SpawnRequest = {
	kind: "projectile" | "zone" | "summon", moveId: string, origin: CFrame,
	params: { [string]: number | string | boolean },   -- JSON-safe; written as carrier attributes
}

export type MotionCmd =
	{ mode: "none" }                              -- LinearVelocity off; Humanoid locomotion
	| { mode: "planar", vx: number, vz: number }  -- Plane mode (world X/Z), gravity acts on Y
	| { mode: "vector", v: Vector3 }              -- Vector mode: full velocity control
	| { mode: "freeze" }                          -- Vector mode, v = 0 (hitstop, Wall Splat)
export type FacingCmd = { mode: "auto" } | { mode: "set", yaw: number } | { mode: "hold" }
export type LocoCmd = { walkSpeed: number, canJump: boolean, jump: boolean }

export type SimEvent = { k: string, rid: number, f: number, a: number?, b: number?, s: string? }
-- k ∈ "hit" | "block" | "pb" | "guardbreak" | "ko" | "breakaway" | "awaken" | "awaken_end" | "shakeoff"
--     | "techroll" | "wallsplat" | "realm_open" | "realm_close" | "spawn" | "style_moment"

export type StepCtx = {
	f: number, rid: number,
	grounded: boolean,                -- Humanoid.FloorMaterial ~= Enum.Material.Air (adapter)
	rootCf: CFrame, vel: Vector3,     -- HRP CFrame / AssemblyLinearVelocity at step start
	style: StyleRuntime, mods: ChassisMods,
	faceYaw: number,                  -- yaw currently stored in ALG_Face.CFrame (the facing state, §3.2.7)
	wallHit: boolean,                 -- adapter wall probe result (§3.5.5); false when not probed
	inEnemyRealm: boolean,            -- RlmId ~= 0 and the Realm's caster is not this rid
	realmHealMult: number,            -- 1 unless a Realm rule changes heals for this member (§3.9.3)
	maxHp: number,                    -- style.hp (100 at launch)
}

export type StepOut = {
	s: CharState, motion: MotionCmd, facing: FacingCmd, loco: LocoCmd,
	hits: { HitRequest }, spawns: { SpawnRequest }, events: { SimEvent },
	entered: boolean,                 -- a new state was entered this step (clears the hit registry, §3.12.5)
	behaviorPhase: ("startup" | "active" | "recovery")?, -- Move state only: adapter calls the Part 4 hook
}
```

`Fsm.step` never mutates its input `s`: it starts with `local n = table.clone(s)` and returns `n` in `StepOut.s`.

#### 3.4.2 Module APIs

Every function is pure. "cfg" is the pinned `Combat` section (`Runtime.at(CombatCfg).Combat`, §2.6.6). Mutating helpers take a `CharState` that the caller owns (a clone).

| Module | Exports (signature) | Notes |
|---|---|---|
| `Frames` | `HZ: number` (60); `toSeconds(fr: number): number` (= fr/60); `toFrames(sec: number): number` (= floor(sec·60+0.5)); `fromTime(t: number): number`; `phase(e: number, startup: number, active: number, recovery: number): "startup"\|"active"\|"recovery"\|"done"`; `inWindow1(e: number, from1: number, to1: number): boolean` (tests `from1 ≤ e+1 ≤ to1`) | §0.4 |
| `Rng` | `new(seed: number): number`; `next(state: number): (number, number)` (value ∈ [0,1), new state); `nextInt(state: number, lo: number, hi: number): (number, number)`; `hash01(a: number, b: number, c: number): number` | xorshift32 on `bit32` (§3.4.3) |
| `Fsm` | `S`, `SET`, `IN`, `CF`, `FLG` constant tables; `initial(hp: number, f: number, cfg): CharState`; `isStun(st: number): boolean`; `isNeutral(st: number): boolean`; `isInvulnerable(s: CharState, f: number, ctx: StepCtx?, cfg): boolean`; `enter(s: CharState, st: number, f: number, dur: number, sub: number): ()`; `step(s: CharState, inp: InputFrame, ctx: StepCtx, cfg): StepOut` | §3.2 |
| `M1` | `frameData(sub: number, cfg, mods: ChassisMods): {startup: number, active: number, recovery: number, hit: HitDef}`; `onPress(s: CharState, inp: InputFrame, ctx: StepCtx, cfg): (number?, number?)` (returns `St`, `Sub` or nil if locked); `enderVariant(inp: InputFrame, ctx: StepCtx, cfg): number`; `exitBookkeeping(s: CharState, f: number, cfg): ()`; `hitbox(sub: number, ctx: StepCtx, cfg, mods: ChassisMods): HitboxReq`; `canHopCancel(s: CharState, e: number, f: number, cfg): boolean` | §3.5 |
| `Defense` | `guardAt(s: CharState, f: number, cfg): number`; `materializeGuard(s: CharState, f: number, cfg): ()`; `isFront(defCf: CFrame, srcPos: Vector3): boolean`; `isPerfect(s: CharState, f: number, cfg, blockDelayF: number): boolean`; `onPress(s: CharState, f: number): ()`; `onRelease(s: CharState, f: number, cfg): ()` (anti-mash); `blockstun(hit: HitDef, adv: number?, cfg): number`; `applyGuardDamage(s: CharState, f: number, amount: number, cfg): boolean` (true = broken) | §3.6 |
| `Movement` | `yawOf(d: Vector3): number`; `classify(moveDir: Vector3, aimYaw: number, cfg): "front"\|"left"\|"right"\|"back"\|"none"`; `dashSub(s: CharState, inp: InputFrame, ctx: StepCtx, cfg): number?`; `dashCooldownKey(sub: number): string?`; `dashMotion(sub: number, e: number, yaw: number, cfg): MotionCmd`; `isDashInvulnerable(sub: number, e: number, cfg): boolean`; `knockbackMotion(s: CharState, e: number, ctx: StepCtx, cfg): MotionCmd`; `launchMotion(s: CharState, f: number, e: number, cfg): MotionCmd`; `hangMotion(inp: InputFrame, cfg): MotionCmd`; `loco(s: CharState, inp: InputFrame, ctx: StepCtx, cfg): LocoCmd`; `facing(s: CharState, inp: InputFrame, ctx: StepCtx, cfg): FacingCmd` | §3.7 |
| `Stun` | `enterStun(v: CharState, st: number, f: number, frames: number, sub: number, cfg): ()` (sets `StunF0` if 0, applies decay); `decayed(frames: number, v: CharState, f: number, cfg): number`; `comboScale(hitIndex: number, cfg): number`; `isDowned(v: CharState): boolean`; `resetCombo(v: CharState): ()`; `shouldShakeOff(v: CharState, f: number, cfg): boolean`; `ragdollFrames(base: number, vicMods: ChassisMods): number` | §3.8 |
| `Breakaway` | `tick(s: CharState, f: number, mods: ChassisMods, cfg): ()`; `refund(s: CharState, damage: number, cfg): ()`; `canUse(s: CharState, f: number, cfg): boolean`; `use(s: CharState, f: number, cfg): ()` | §3.9.1 |
| `Meter` | `gainDealt(m: number, dmg: number, hit: HitDef, cfg): number`; `gainTaken(m: number, dmg: number, cfg): number`; `gainEvent(m: number, kind: "pb"\|"style"\|"ko"\|"assist", cfg): number`; `isFull(m: number, cfg): boolean`; `activate(s: CharState, f: number, style: StyleRuntime, cfg): ()`; `extendOnKo(s: CharState, cfg): ()`; `tickAwakening(s: CharState, f: number): boolean` (true when it ended this step) | §3.9.2 |
| `Moves` | `slotKey(sub: number): string` ("Cd1".."Cd4", "CdQ"); `defFor(style: StyleRuntime, sub: number): Types.MoveDef?`; `canUse(s: CharState, sub: number, f: number, ctx: StepCtx, cfg): boolean`; `cooldownFrames(def: Types.MoveDef, mods: ChassisMods): number`; `hitDef(def: Types.MoveDef, cfg): HitDef`; `hitbox(def: Types.MoveDef, ctx: StepCtx, prevCf: CFrame?): HitboxReq?`; `validate(def: Types.MoveDef, slot: string, cfg): { string }`; `chassisMods(style: StyleRuntime, s: CharState, cfg): ChassisMods`; status codec below | §3.4.2, §6.7 |
| `Damage` | `resolve(att: AttackerView, vic: VictimView, hit: HitDef, hc: HitCtx, cfg): HitResult`; `scaled(base: number, comboIndex: number, pileOn: boolean, guardBroken: boolean, cfg): number`; `quantize(x: number): number` | §3.12.6 |
| `Realm` | `castPhase(e: number, cfg): "interruptible"\|"armored"\|"opening"`; `interrupts(e: number, hit: HitDef, guardBreak: boolean, cfg): boolean`; `contains(center: Vector3, radius: number, pos: Vector3): boolean`; `canStubbornHeart(s: CharState, blockE: number, f: number, cfg): boolean`; `slotFree(active: number, cfg): boolean`; `rules: { [string]: RealmRules }` (Part 4 fills) | §3.9.3 |

**Status codec** (in `Core/Moves.luau`; buffs and debuffs from kits, GDD §6.7 tags `Buff`/`Debuff`):

```lua
export type Status = { kind: string, untilF: number, n: number } -- kind: 2-char id from Moves.STATUS (Part 4 registry)
export type StatusMods = {
	speedMult: number, rooted: boolean, silenced: boolean, blockDelayF: number,
	regenBlocked: boolean, nextHitAdd: number, shield: number, -- shield HP (cap Chassis.ShieldCap = 20)
}
-- Moves.statusDecode(s: string): { Status }        "<kind>:<untilF base36>:<n>;..." ; "" -> {}
-- Moves.statusEncode(list: { Status }): string      sorted by kind; <= 4 entries; asserts #result <= 50
-- Moves.statusAdd(list, st: Status): { Status }     same kind: refresh untilF = max, n = min(n + st.n, maxStacks(kind));
--                                                   over capacity: evict the entry with the smallest untilF
-- Moves.statusExpire(list, f: number): ({ Status }, boolean)   removes untilF <= f; boolean = changed
-- Moves.statusMods(buffs: { Status }, debuffs: { Status }): StatusMods   via Moves.STATUS[kind].mods (Part 4)
```

Part 4 owns the `Moves.STATUS` registry: `{[kind: string]: {name: string, maxStacks: number, mods: (n: number) -> partial StatusMods}}`, e.g. Decay (`"dc"`, max 5, regenBlocked, nextHitAdd = n), Whispered (`"wh"`, blockDelayF = 6), Root (`"rt"`), Slow (`"sl"`).

**Behavior hook contract** (called by the adapter, implemented by Part 4 in `Shared/Behaviors/<Behavior>.luau`; `Strike` needs no module because Core handles plain melee):

```lua
export type BehaviorCtx = {
	f: number, e: number, phase: "startup" | "active" | "recovery",
	def: Types.MoveDef, s: Fsm.CharState, ctx: Fsm.StepCtx, cfg: any,
	isServer: boolean,                 -- read-only fact; behaviors may NOT compute different B-class outcomes from it
}
export type BehaviorOut = {
	motion: Fsm.MotionCmd?, hits: { Fsm.HitRequest }?, spawns: { Fsm.SpawnRequest }?,
	patch: { [string]: any }?,         -- B-class attribute changes on the owner (validated against Attr.SPECS)
	events: { Fsm.SimEvent }?,
}
export type BehaviorModule = {
	step: (bc: BehaviorCtx) -> BehaviorOut?,                                   -- every Move-state frame, both peers (owner)
	onContact: ((bc: BehaviorCtx, result: Damage.HitResult) -> BehaviorOut?)?, -- server always; owner client for BQ predictions
	onEnd: ((bc: BehaviorCtx) -> BehaviorOut?)?,                               -- state exit (also on interrupt)
}
```

#### 3.4.3 Determinism rules and the seeded RNG

1. Core functions depend only on their arguments. No upvalue mutation, no globals other than `math`, `bit32`, `string`, `table`, `Vector3`, `CFrame`.
2. Iterate arrays with numeric `for`/`ipairs`-style generalized iteration over arrays only. Keyed tables are iterated only after sorting keys.
3. Integer math for frames. Floating values that reach attributes follow the Q-rule (§3.1.4).
4. `Rng`: xorshift32. `new(seed)` returns `bit32.bor(seed % 4294967296, 1)` (never 0). `next(x)`: `x ^= x << 13; x ^= x >> 17; x ^= x << 5` via `bit32.bxor/lshift/rshift`, value = `x / 4294967296`. `hash01(a, b, c)` = `next(new(a * 73856093 + b * 19349663 + c * 83492791))`. Seeds come from sim data only, e.g. `(f, rid, saltForThisDecision)`. Rng state is never stored across frames on the client.
5. Golden traces: `tests/core/fsm.spec.luau` feeds scripted `InputFrame` sequences into `Fsm.step` and compares the resulting `St/StF/Sub/HP` per frame against checked-in traces (`tests/core/golden/*.json`). Any change to tuning requires regenerating traces with `lune run tools/export_framedata.luau --golden`.

#### 3.4.4 Lune test catalog (Part 7 runs; names are binding)

`tests/core/frames.spec.luau`: `frames.to_seconds`, `frames.phase_boundaries`, `frames.window_1indexed`.
`tests/core/rng.spec.luau`: `rng.deterministic`, `rng.range_uniform_chi2`, `rng.hash_stable`.
`tests/core/attr.spec.luau`: `attr.budget`, `attr.defaults_match_fsm`.
`tests/core/fsm.spec.luau`: `fsm.buffer_6f`, `fsm.buffer_latest_wins`, `fsm.buffer_cleared_on_stun`, `fsm.priority_awaken_over_dash`, `fsm.no_input_in_stun`, `fsm.golden_m1_string`, `fsm.golden_uppercut_string`, `fsm.golden_hop_cancel`, `fsm.pure_no_mutation`.
`tests/core/m1.spec.luau`: `m1.frame_table_gdd`, `m1.chain_window_36f`, `m1.chain_reset_37f`, `m1.ender_endlag_60f`, `m1.variant_select`, `m1.air_string_fits_hang`, `m1.dashm1_continues_at_2`, `m1.downslam_once_per_combo`.
`tests/core/defense.spec.luau`: `defense.front_180`, `defense.behind_unblocked`, `defense.pb_window_9f`, `defense.antimash_20f`, `defense.guard_regen_lazy`, `defense.guardbreak_84f`, `defense.blockstun_from_adv`.
`tests/core/movement.spec.luau`: `movement.classify_dirs`, `movement.dash_distances`, `movement.side_iframes_1_7`, `movement.back_iframes_1_5`, `movement.cooldowns`, `movement.airdash_once`.
`tests/core/stun.spec.luau`: `stun.decay_after_150f`, `stun.shakeoff_210f`, `stun.otg_once`, `stun.combo_scale_tiers`, `stun.scale_floor`, `stun.jake_ragdoll_minus10`.
`tests/core/breakaway.spec.luau`: `breakaway.lockout_12f`, `breakaway.refill_25s`, `breakaway.damage_refund`, `breakaway.pileon_2x`, `breakaway.guardbreak_30f`.
`tests/core/meter.spec.luau`: `meter.gains_table`, `meter.realm_half`, `meter.awaken_effects`, `meter.awaken_60s_plus5_cap90`.
`tests/core/damage.spec.luau`: `damage.m1_blocked_zero`, `damage.move_chip_20`, `damage.pileon_25`, `damage.guardbroken_plus10`, `damage.iframes_whiff`, `damage.grab_vs_block`, `damage.grab_vs_airborne`, `damage.pb_parryable_only`, `damage.design_target_no_meter` (the §3.5.7 string totals 28–36).
`tests/core/moves.spec.luau`: `moves.status_codec_roundtrip`, `moves.status_capacity_evicts`, `moves.validate_bands`, `moves.validate_tell_8f`.
`tests/core/realm.spec.luau`: `realm.interrupt_window_45f`, `realm.contains`, `realm.max2`, `realm.stubborn_heart_once`.

---

### 3.5 M1 chain, Enders, air M1s, Dash M1, Wall Splat

#### 3.5.1 Chain data and rules (GDD §6.3)

Frame data and damage are identical for every style (GDD §6.3). Styles only re-skin (`StyleDef.m1`). Blockstun is derived so that the GDD "on block" advantage holds for a hit on the first active frame: `blockstun = (active − 1) + recovery + blockAdv`.

| Sub | Hit | Startup | Active | Recovery | Damage | Hitstun | On block (adv → blockstun) | Guard | Hitstop | Source |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | M1-1 | 6 | 3 | 12 | 3.5 | 27 | −2 → 12 | 12 | 3 | GDD §6.3, §6.13 |
| 2 | M1-2 | 6 | 3 | 12 | 3.5 | 27 | −2 → 12 | 12 | 3 | GDD §6.3 |
| 3 | M1-3 | 7 | 3 | 13 | 3.5 | 27 | −3 → 12 | 12 | 3 | GDD §6.3 |
| Ender | M1-4 | 10 | 4 | 22 | 5.0 | KB 38 → Ragdoll 60 | −10 → 15 | 20 | 6 | GDD §6.3, §6.13 |
| 11–13 | Air M1 | 5 (td) | 3 (td) | 10 (td) | 2.5 | 27 | −2 (td) → 10 | 8 (td) | 3 | GDD §6.3 dmg |
| 20 | Dash M1 | 6 (td) | 4 (td) | 16 (td) | 4.0 | 30 | −4 (td) → 15 | 12 (td) | 3 | GDD §6.5 dmg/hitstun |
| Ender 3 | Downslam | 8 (td) | 4 (td) | 22 (td) | 5.0 | knockdown 54 | −10 (td) → 15 | 20 | 6 | GDD §6.3 |

Rules:
1. **Index selection** (`M1.onPress`, called at P12 or from the buffer):
```lua
-- inside Core/M1.luau
function M1.onPress(s: Fsm.CharState, inp: Fsm.InputFrame, ctx: Fsm.StepCtx, cfg: any): (number?, number?)
	local f, m = ctx.f, cfg.M1
	if f < s.M1LockF then
		return nil, nil -- Ender end lag (GDD §6.3: 60f)
	end
	if f < s.HangF and not ctx.grounded then -- Uppercut hang (§3.5.3)
		if s.AirN < m.Air.MaxHits then
			return Fsm.S.M1, 11 + s.AirN -- 11, 12, 13
		end
		return Fsm.S.Ender, 3 -- 4th air press = Downslam
	end
	local idx = 1
	if s.M1Idx >= 1 and s.M1Idx <= 3 and f <= s.M1EndF + m.ChainWindowFrames then
		idx = s.M1Idx + 1 -- GDD §6.3 chain window 36f after the previous hit's recovery
	end
	if idx == 4 then
		return Fsm.S.Ender, M1.enderVariant(inp, ctx, cfg)
	end
	return Fsm.S.M1, idx
end
```
2. **Entry effects.** `M1Idx = idx` (ground hits and Ender = 4); `AirN += 1` for air hits; `Cnt = 0`; `ProtF = 0`; `InvF = 0`; facing tracks aim during startup.
3. **Chain progress on whiff.** A whiffed hit still advances the chain (`M1.ChainOnWhiff = true`, td; genre convention).
4. **Buffering.** An M1 pressed at any time during M1 recovery is buffered until the recovery ends (`M1.BufferWholeRecovery`), then starts the next index on the exit frame. This guarantees the 36f window is never missed by mashing.
5. **Exit bookkeeping** (`M1.exitBookkeeping`, on timer exit or cancel): `M1` exit → `M1EndF = f`. Dash M1 exit → `M1Idx = 1`, `M1EndF = f` (Dash-Ender: the chain continues at M1-2, GDD §6.5). `Ender` exit → `M1Idx = 0`, then end lag (§3.5.2).
6. **Hitbox** (`M1.hitbox`): Box `Combat.M1.Hitbox.Size` = (5, 5, 6) at `Offset` = (0, 0, −3.5) in the attacker's yaw-only root frame (td). `ChassisMods.m1RangeAdd` (Finn awakened +3, GDD §6.9) adds to the depth and moves the centre forward by half of it. Queried on every active frame; each target at most once per action instance (§3.12.5).
7. **Lunge and push** (td): the attacker moves `M1.LungeStuds` = 1.5 forward during startup (planar). A ground-M1 hit pushes the victim `M1.PushbackStuds` = 1.5 along the hit direction during the first `Stun.HitPushFrames` = 6 frames of hitstun. Equal lunge and push keep the chain in range.

#### 3.5.2 Enders (GDD §6.3)

Variant chosen once, on the frame M1-4 starts (`M1.enderVariant`), in this order:

| Order | Condition | Variant (`Sub`) | On hit |
|---|---|---|---|
| 1 | Attacker not grounded | 3 Downslam | §3.5.4 |
| 2 | `Jump` or `Launcher` held | 2 Uppercut | §3.5.3 |
| 3 | lateral input `\|x\| ≥ M1.SideEnder.SideInputMin` (0.5, td), where `x` is `moveDir` projected on the aim-right axis | 4 Side-L / 5 Side-R | knockback rotated ±45° toward the held side (GDD §6.3), else as Normal |
| 4 | otherwise | 1 Normal | knockback 38 studs along the attacker's facing, angle `Ragdoll.KbAngleDeg` = 10° (td), Ragdoll 60f; `CFlags \|= ENDERKB` |

**End lag:** on Ender exit, `M1LockF = f + M1.EndLagFrames` (60, GDD §6.3). Exception: an Uppercut that **connected** sets `Flg |= ENDLAG` instead, and the lock is applied when the air string ends: on Downslam exit, on landing, or when `f ≥ HangF` (whichever comes first). At that point `AirN = 0` and `Flg &= ~(ENDLAG|FOLLOW)`.

Finn's passive (GDD §7.3.1) adds `ChassisMods.enderDamageAdd` (+2 below 30 HP) to every Ender variant's base damage.

#### 3.5.3 Uppercut, hang time and air M1s (GDD §6.3)

Timeline (frames after the hit frame `h0`, hitstop `hs = 6`; rise starts at `h0 + hs + 1`):

| Who | Frames | Motion (Vector mode) | State |
|---|---|---|---|
| Victim | rise `R = M1.Uppercut.RiseFrames` = 18 (td) | `vy(e) = v0·(1 − e/R)`, `v0 = 2·22·60/(R+1)` = 138.9 st/s (exactly 22 studs, GDD) | `Launched`, `StDur = R + 48` |
| Victim | hang 48 (GDD) | `vy = −M1.Uppercut.HangFallSpeed` = −3 st/s (td), horizontal 0 | `Launched`; air-M1 hits switch it to `Hitstun` Sub 1 but it keeps floating while `f < HangF` |
| Attacker (BQ on contact) | rise 18 | `v0 = 2·20·60/19` = 126.3 st/s (`AttackerRiseStuds` = 20, td: ends 2 studs below the victim) | `Ender` ends at the end of hitstop (`Uppercut.HitCancelsRecovery`), then `Air`; `Flg \|= FOLLOW\|ENDLAG` |
| Attacker | hang 48 | `vy = −3`, horizontal `moveDir · HangDriftSpeed` (6 st/s, td) | `Air` |

Both get `HangF = h0 + hs + 1 + 18 + 48`. While `f < HangF` and airborne: M1 presses produce Air M1 #1–#3 (2.5 dmg each, GDD), and a 4th press produces the Downslam. Budget check: 3 air M1s × 18f + Downslam startup 8 = 62 ≤ 66 frames of rise+hang. Lune `m1.air_string_fits_hang` asserts this from config.

If the Uppercut whiffs: no follow, no hang; normal end lag.

#### 3.5.4 Downslam (GDD §6.3)

- On hit, if the victim's `CFlags` lacks `DOWNSLAM`: victim → `Ragdoll`, `StDur = M1.Downslam.KnockdownFrames` (54, GDD) + victim `ragdollFramesAdd`; `CFlags |= DOWNSLAM|SPIKE`; motion `vector (0, −M1.Downslam.SpikeSpeed, 0)` (120 st/s, td) until grounded; on first landing, bounce `vy = sqrt(2·g·BounceStuds)` (2 studs, td → 28.0 st/s at g = 196.2), integrated as `vy − g·e'/60` until grounded again; then no mover.
- If `DOWNSLAM` is already set ("only once per combo"), the hit resolves as a **Normal Ender** (horizontal knockback, Ragdoll 60f).
- Attacker on Downslam entry: `HangF = f` (the attacker falls), then end lag on exit.

#### 3.5.5 Wall Splat (GDD §6.3)

- Eligible victim: `St = Ragdoll`, `CFlags` has `ENDERKB` (set by Normal/Side Ender hits and by moves with `MoveDef.params.wallSplat = true`, e.g. GDD §7.3.1 move 4), lacks `WALLSPLAT`, and `e < M1.WallSplat.WindowFrames` (18f = 0.3 s, GDD).
- Probe (adapter, both peers for the victim's owner; before `Fsm.step`): `workspace:Raycast(rootPos, hdir * (vh(e)/60 + M1.WallSplat.ProbeStuds), params)` where `hdir` is the horizontal unit of `KbV`, `vh(e)` is this frame's knockback speed, `ProbeStuds` = 1.6 (capsule radius 1.1 + 0.5, td), `params.CollisionGroup = "WorldQuery"` (§3.12.1), `params.ExcludeInstances = {workspace.Sim}`. It is a wall if the result exists and `math.abs(result.Normal.Y) < 0.5`. The result goes in `ctx.wallHit`.
- Effect: `WallSplat` for 30f (GDD), motion `freeze` (held on the wall against gravity), `CFlags |= WALLSPLAT`, event `wallsplat`. A hit on a wall-splatted victim resolves normally (usually → `Hitstun` at the wall: "the attacker can extend the combo"). After 30f without a hit → `Ragdoll` (`AfterRagdollFrames` = 30, td) → wake-up.
- Maps must keep walls and terrain in the `Default` collision group (Part 6 contract, §3.18).

#### 3.5.6 Hitstop mechanics (GDD §6.13)

Durations: `Damage.Hitstop.M1` 3 (also Air M1 and Dash M1), `.Ender` 6 (all variants), `.Move` 4 (td; non-signature moves), `.Signature` 9 (GDD 8–10f), `.PerfectBlock` 6 (both players). A block uses the same hitstop as the hit (`OnBlockSame = true`). Realm-opening hitstop (12f) is render-only.

On contact at frame `f` with hitstop `hs`:
- Attacker (BQ): `HsF = f + hs`; `StF += hs` (the action's timeline pauses). Hit-registry keys never contain `StF`, so the shift does not reopen targets (§3.12.5).
- Victim (S): enters its new state with `StF = f + hs + 1` and `HsF = f + hs`.
- While `f ≤ HsF`, `Fsm.step` is frozen: no timer or input transitions (inputs are buffered), no hit queries, motion `freeze`, `LocoCmd.walkSpeed = 0`. Render freezes animations for the same frames (§3.11.8).

#### 3.5.7 Damage design check (GDD §6.6)

The longest no-meter, no-wall chassis string: M1-1, M1-2, M1-3 (3 × 3.5), Uppercut (5), 3 air M1s (3 × 2.5), Downslam (5) = **28.0** in 8 hits (all at scale ×1.0). That is inside the 28–36 band (GDD §6.6) before kit extenders. Lune `damage.design_target_no_meter` computes it from config and asserts `28 ≤ total ≤ 36`. Part 4 extends the test with each kit's recorded routes (`Damage.Targets`).

**Verify (§3.5):** Lune `m1.*` (§3.4.4). Scenarios: `combat.m1_chain_frames` (idle dummy; four buffered presses; server event log shows hits at `e = 6, 6, 7, 10` of each state; total damage 15.5; the dummy ends in `Ragdoll` with `StDur = 60`; `M1LockF = EnderExit + 60`); `combat.chain_window_reset` (M1-2 pressed 37f after M1-1 exit → `Sub = 1`); `combat.uppercut_air_string` (scripted bot attacker vs idle dummy → 8 hits, 28.0 damage, dummy `CFlags` has DOWNSLAM); `combat.wall_splat` (Ender toward a wall 10 studs away → `WallSplat` within 18f; a second Ender in the same combo → no splat).

---

### 3.6 Block, guard, Perfect Block, Guard Break (GDD §6.4)

#### 3.6.1 Block state and the front 180°

- `Block` is entered from `Idle` on `Block` held when `f ≥ BlkLockF` (P13). Block startup is 0 (`Block.StartupFrames`, td): it is active on the press frame unless a status sets `blockDelayF` (Whispered, GDD §7.3.15: +6f). Walk speed is 8 (GDD). Facing tracks aim.
- Block covers the front 180°. Source point for the test: melee/grab = the attacker's present root position on the server; projectile = projectile position at the hit; zone/AoE = zone centre.

```lua
-- Core/Defense.luau
function Defense.isFront(defCf: CFrame, srcPos: Vector3): boolean
	local look = defCf.LookVector
	local dx, dz = srcPos.X - defCf.Position.X, srcPos.Z - defCf.Position.Z
	if dx * dx + dz * dz < 0.25 then
		return true -- overlapping centres: treat as front (deterministic tie-break)
	end
	return look.X * dx + look.Z * dz >= 0 -- half-plane in front of the defender (GDD §6.4: front 180°)
end
```

Hits from behind ignore Block and Perfect Block entirely (GDD §6.4).

#### 3.6.2 Guard meter (lazy regeneration)

- `Defense.guardAt(s, f)`: if `St ∈ {Block, BlockStun}` → `Grd` (no regen while blocking). Otherwise `min(GuardMax, Grd + max(0, f − GrdF − GuardRegenDelayFrames) · GuardRegenPerSec / 60)` with GuardMax 100, delay 60f (1.0 s), 25/s (GDD).
- Writes: on Block exit `Grd = guardAt(f)`, `GrdF = f`; on Block entry `Grd = guardAt(f)` (materialize); on guard damage (S) `Grd = guardAt(f) − dmg`, `GrdF = f`. No per-frame writes.
- Guard damage: M1-1..3 12, Ender 20 (GDD), Air M1 8 (td), moves `MoveDef.guardDamage` (validated to 20–60, GDD §6.4).

#### 3.6.3 Perfect Block and anti-mash

- `BlkF = f` on every Block press (rising edge), including a re-press during `BlockStun`; `Flg &= ~PBHIT` on press.
- **Perfect** (`Defense.isPerfect`): `St ∈ {Block, BlockStun}` and `0 ≤ f − (BlkF + blockDelayF) ≤ Block.PerfectWindowFrames` (9, GDD), the hit is from the front, and the hit is blockable or a grab with `parryable = true`.
- **PB effects** (server): the hit is negated completely (no damage, no guard damage, no chip). Defender: `Flg |= PBHIT`, `HsF = f + 6`; meter +6 (GDD); event `pb`. Attacker (melee only): `Stagger` Sub 1 for 36f if the hit is M1-family (`m1`, `ender`, `airm1`, `dashm1`), else 18f for a melee move; `HsF = f + 6`; `Cnt = 3`. Projectiles and zones: negated, no stagger.
- **Anti-mash** (`Defense.onRelease`, B): on Block release, if `f − BlkF ≤ Block.TapMaxFrames` (15, td: "a tap") and `PBHIT` is not set → `BlkLockF = f + Block.AntiMashLockFrames` (20, GDD). Holding block normally (> 15f) never locks.

#### 3.6.4 Blocked hits, chip, blockstun

| Hit kind | Blocked from front | Damage | Guard | Defender state |
|---|---|---|---|---|
| M1 family | yes | 0 (GDD: negates 100%) | per §3.6.2 | `BlockStun` for the §3.5.1 blockstun |
| Move (blockable) | yes | `damage · chip` (default `Block.MoveChipDefault` 0.2, GDD); chip cannot reduce HP below 1 (`Block.ChipCanKo = false`, td) | `MoveDef.guardDamage` | `BlockStun` for `MoveDef.params.blockstun` or `max(Block.MoveBlockstunMin, floor(hitstun · Block.MoveBlockstunDefaultMult))` = max(8, ⌊0.5·hitstun⌋) (td) |
| Projectile | yes | as Move | as Move | as Move |
| Grab (`unblockable`) | no (grabs ignore block, GDD) | full | — | grabbed (Part 4) |
| Zone tick (`tick`) | no (td: ticks ignore block) | full tick | — | unchanged |

Pushback on block: `Block.PushbackStuds` = 1.0 (td), planar during the first 6 frames of `BlockStun`. Attacker gets `Cnt = 2` (enables Hop Cancel, GDD §6.5).

#### 3.6.5 Guard Break

When a blocked hit brings guard to ≤ 0: the hit itself stays blocked (chip only). Then the defender → `GuardBroken` for 84f (GDD), `BkLockF = StF + 30` (no Breakaway for the first 30f, GDD), `StunF0` starts if 0, and damage taken while `St = GuardBroken` is ×1.10 (GDD). On exit, `Grd = GuardMax` (`Block.GuardRefillOnBreakEnd = true`, td). Events: `guardbreak` (render plays the glass-candy shatter, GDD §6.4).

**Verify (§3.6):** Lune `defense.*`. Scenarios: `combat.block_front_back` (blocking dummy: front M1 → 0 damage, `Grd` 88; from behind → 3.5 damage); `combat.perfect_block` (dummy mode `perfect_block` vs player M1 → attacker `St = Stagger`, `StDur = 36`; defender `Meter` Player attribute +6); `combat.antimash` (bot taps Block for 5f with no hit → `BlkLockF = release + 20`; Block press during the lock does nothing); `combat.guard_break` (9 M1s into a holding-block dummy → `GuardBroken` on the 9th, `StDur = 84`; Breakaway attempt at e = 20 refused, at e = 31 accepted).

---

### 3.7 Movement, dashes and the physics implementation (GDD §6.2, §6.5)

#### 3.7.1 Direction classification

`Movement.classify(moveDir, aimYaw)`: the aim frame is `fwd = (−sin yaw, 0, −cos yaw)` and `right = (cos yaw, 0, −sin yaw)`. With `x = moveDir·right` and `z = moveDir·fwd`: if `x² + z² < Dash.DirDeadzone²` (0.2, td) → `none`. Otherwise `a = deg(atan2(x, z))`: `|a| ≤ 45` → `front`; `|a| ≥ 135` → `back`; `a > 0` → `right`; else `left` (thresholds `Dash.DirFrontMaxDeg`/`DirBackMinDeg`, td). Classification is relative to the **aim** (camera) yaw, so W/S/A/D map to front/back/side dashes with or without Shift-Lock.

`Movement.dashSub`: grounded `none|front` → 1 Front (`Dash.NoDirection = "Front"`, td), `left` → 2, `right` → 3, `back` → 4. Airborne → 5 Air, only if `ChassisMods.airDash` (the style's Aerial capability, GDD §6.5) and `Flg` lacks `AIRDASH`. `rooted` status blocks all dashes.

#### 3.7.2 Dash table

Speeds are constant: `v = Studs · 60 / Frames`. Cooldowns start at dash start: `Cd<kind> = f + toFrames(CooldownSec)`.

| Sub | Dash | Studs | Frames | Speed (st/s) | I-frames (`e+1`) | Cooldown | Path | Source |
|---|---|---|---|---|---|---|---|---|
| 1 | Front | 30 | 18 | 100 | none | 2.0 s → `CdDF` | straight along the start yaw | GDD §6.5 |
| 2/3 | Side L/R | 20 | 15 | 80 | 1–7 | 1.6 s → `CdDS` | curved: `dir(e) = lat·cos θ + fwd·sin θ`, `θ = rad(CurveDeg)·e/(Frames−1)`, `CurveDeg` = 45 (td) | GDD §6.5 |
| 4 | Back | 16 | 14 | 68.57 | 1–5 | 2.0 s → `CdDB` | straight, −fwd | GDD §6.5 |
| 5 | Air | 14 | 12 | 70 | none | once per airtime (`Flg AIRDASH`) | straight, horizontal `moveDir` (else fwd), `vy = 0` | GDD §6.5 |

Facing holds the start yaw for the whole dash. Front dash enables Dash M1 (§3.5.1, P9/P9b). Side dash enables Hop Cancel only as a *target* (P8).

Tech Roll (GDD §6.5 #4): 18 frames, all invulnerable, `Dash.TechRoll.Studs` = 6 (td) along horizontal `moveDir` (else along `KbV`'s horizontal direction), planar 20 st/s.

#### 3.7.3 Physics implementation (movers inside the sim)

Each character carries pre-wired actuators (created outside the sim, §3.11.2). The sim only writes their targets, never creates them (Cakeman rule 6, SA techniques). All properties used are Simulation-Access [Docs `LinearVelocity.yaml`, `AlignOrientation.yaml`, `AlignPosition.yaml`].

| `MotionCmd` | `ALG_Vel` (LinearVelocity, `RelativeTo = World`, `ForceLimitsEnabled = false`) |
|---|---|
| `none` | `Enabled = false` |
| `planar {vx, vz}` | `VelocityConstraintMode = Plane`, `PrimaryTangentAxis = (1,0,0)`, `SecondaryTangentAxis = (0,0,1)`, `PlaneVelocity = Vector2.new(vx, vz)`, `Enabled = true`. Gravity still acts on Y. |
| `vector {v}` | `VelocityConstraintMode = Vector`, `VectorVelocity = v`, `Enabled = true` |
| `freeze` | `vector` with `v = Vector3.zero` |

- `ForceLimitsEnabled = false` lets the solver apply "a force that is large enough to achieve the target velocity" [Docs], which overrides Humanoid friction. Spike SA-08 measures distance error and jitter. **Fallback A** (if LinearVelocity jitters or rubber-bands under 150 ms): write `HRP.AssemblyLinearVelocity` every frame instead (Sim-Access, rolls back; Cakeman notes it is legal but should be used deliberately). **Fallback B**: `ForceLimitsEnabled = true`, `ForceLimitMode = Magnitude`, `MaxForce = Combat.Chassis.MoverMaxForce · AssemblyMass`.
- `ApplyImpulse` is not used in M0/M1-G. Vertical arcs are prescribed velocities (Q-rule friendly and gravity-independent).
- `Chassis.Gravity` (196.2) is used for prescribed arcs. `CombatService.init()` asserts `workspace.Gravity == Chassis.Gravity` and logs `[ALG][Combat] ERROR gravity mismatch` otherwise.
- Humanoid locomotion stays on the default controller: the sim writes `Humanoid.WalkSpeed`, `Humanoid.JumpHeight` (with `UseJumpPower = false`), `Humanoid:SetStateEnabled(Enum.HumanoidStateType.Jumping, canJump)`, `Humanoid.Jump = true` for an accepted Jump press, and `Humanoid.AutoRotate`. All are Simulation-Access; each is written only when its value changes.

#### 3.7.4 Character Controller Library: decision

**Decision: custom FSM on the default Humanoid controller; CCL not adopted for M0/M1-G.** Reasons:
1. Core must be Lune-testable and identical under the Classic fallback (§3.14). CCL ability state lives inside the engine (`abilityCtx.State`) and cannot run in Lune or in Classic mode.
2. CCL is beta [Docs `character-controller-library/index.md`], gated by a Studio beta flag and an Avatar Settings toggle (H-05).
3. CCL action slots give at most 7 touch buttons [Docs `includes/action-slots.md`]. We need 12+ actions, and remapping slots to custom InputActions is "future" per the docs.
4. Everything CCL would give us (timed labels, exclusive groups, rollback-safe state) is already covered by attributes and the FSM.

Spike SA-06 confirms that the dev place does **not** spawn characters with a CCL `ControllerManager`. If a future Avatar Settings change makes characters spawn with a `ControllerManager`, the adapter switches to `ControllerLoco`: `walkSpeed → ControllerManager.BaseMoveSpeed`, facing → `ControllerManager.FacingDirection` (both Sim-Access). The locomotion interface in `Adapter` (`Loco.apply(entry, cmd)`) has exactly these two implementations. Revisit CCL after M1-G (§3.20).

**Verify (§3.7):** Lune `movement.*`. Scenarios: `combat.dash_distances` (each dash on flat ground: horizontal displacement within ±10% of the table; cooldown attributes `= start + 120/96/120`); `combat.side_dash_iframes` (dummy `m1_spam` hits during side-dash frames 1–7 → `whiff` events; frame 8+ → hit); `combat.hop_cancel` (M1-2 hits a blocking dummy → Side dash accepted during recovery; on whiff → refused).

---

### 3.8 Hitstun, Stun Decay, combo limits, knockback and ragdoll (GDD §6.6, §13.3)

#### 3.8.1 Combos and continuous stun

- A **combo** on a victim is the sequence of hits while the victim stays in `STUN` states without a break. Entering any non-`STUN` state (`Idle`, `Air`, `Block`, `Breakaway`, `ShakeOff`, `TechRoll`, …) ends it: `Stun.resetCombo` sets `StunF0 = 0`, `CHits = 0`, `CFlags = 0`. The owner FSM runs it on both peers (B).
- `StunF0` is set (S) on the first stun entry of a combo: `StunF0 = f` if it was 0. Continuous stun time is `f − StunF0`.

#### 3.8.2 Hitstun and Stun Decay

- `Stun.enterStun(v, st, f, frames, sub, cfg)`: if `f − StunF0 ≥ Stun.DecayStartFrames` (150 = 2.5 s, GDD) then `frames = floor(frames · Stun.DecayMult)` (×0.6, GDD). Decay applies to `Hitstun` and `Ragdoll` durations and to the `Launched` hang; it does not apply to Guard Break, Perfect-Block stagger or pushes.
- Hitstun is refreshed per hit (GDD §6.6): a new hit sets `StF`/`StDur` anew. It is not added to the remaining time.
- Airborne hitstun (`Sub = 1`): the victim keeps the hang motion while `f < HangF`; otherwise gravity applies.

#### 3.8.3 Shake-Off (GDD §6.6)

At `f − StunF0 ≥ Stun.ShakeOffFrames` (210 = 3.5 s), P2 forces `ShakeOff` for 18 frames, all invulnerable, with a 6-stud planar push (20 st/s) along the horizontal direction of `KbV` (fallback: −facing). It detaches any grab (`ALG_Hold.Enabled = false`), resets the combo and emits event `shakeoff`. B-class: the victim's client predicts it once it knows `StunF0`.

#### 3.8.4 Damage scaling, OTG, pile-on

- **Scaling** (GDD §6.6): for the hit with index `n = CHits + 1`: tier ×1.0 for n ≤ 8, ×0.85 for 9–14, ×0.70 for 15+ (`Damage.ComboTiers`). `scale = max(Damage.ScaleFloor (0.5), tier · (pileOn and PileOn.DamageMult or 1))`. Then `final = quantize((base + flatAdds) · scale · (St == GuardBroken and 1.10 or 1))`, where `flatAdds` are kit adds (Finn Ender +2, Decay stacks, next-hit buffs) and `quantize` rounds to 0.1. Ticks (`hit.tick`) and chip ignore combo scaling (td) and do not increment `CHits`.
- **OTG** (GDD §6.6): the victim is **downed** when `St = Ragdoll` and `Sub > 0` (it has landed). A hit on a downed victim is allowed only if `CFlags` lacks `OTG`; it sets `OTG`, applies damage, and sets `StDur` so the remaining ragdoll is `max(remaining, hit.hitstun)` (no restart). A second OTG hit resolves as `otg_denied` (no effect, event for render). Airborne ragdolled victims (juggles) are hittable without OTG cost.
- **Pile-on** (GDD §6.6): server memory per victim keeps `{rid, f}` for each damaging hit, pruned to the last `toFrames(PileOn.WindowSec)` = 240 frames. When the number of distinct attacker rids ≥ 3, the server sets `PileF = f + 240` (S). While `f < PileF`: damage ×0.75 (including the triggering hit) and Breakaway refill ×2.

#### 3.8.5 Knockback profiles (prescribed velocities)

`KbV` stores the initial velocity; Core recomputes each frame from `KbV` and `e`. All profiles use the exact discrete form `v0 = 2·D·60/(N+1)`, so the summed per-frame displacement equals `D`.

| Case | State | Profile (Vector mode unless noted) | Defaults |
|---|---|---|---|
| Ragdoll knockback (Ender, move `knockback.ragdoll`) | `Ragdoll`, `StDur` = 60 (Ender) or `params.ragdollFrames` + victim `ragdollFramesAdd` | horizontal `vh(e) = v0·(1 − e/N)` along `hdir`; vertical `vy(e) = v0·tan(angle) − g·e/60` until grounded after `e ≥ 3`, then 0; after `N` frames → mover off | D = 38 (GDD), N = `Ragdoll.KbTravelFrames` 30 (td), angle 10° (td) → v0 = 147.1 st/s, apex 1.7 studs |
| Launch (Uppercut, move `knockback.launch`) | `Launched` | §3.5.3 rise + hang; for moves `D` = `knockback.dist` (height), hang = `params.hangFrames` (default 0) | — |
| Push (non-ragdoll move knockback) | `Hitstun` | planar, constant `D·60/N`, `N = min(StDur, Stun.MoveKbFramesMax)` (12, td) | — |
| M1 push | `Hitstun` | planar `1.5·60/6` = 15 st/s for 6 frames | td |
| Spike (Downslam) | `Ragdoll` + `SPIKE` | §3.5.4 | — |
| Wall Splat | `WallSplat` | `freeze` | GDD 30f |
| Stagger push (Breakaway/Awakening shockwave) | `Stagger` Sub 2 | planar constant `D·60/12` for 12 frames | D = 8 (Breakaway) / 10 (Awakening), GDD |

Direction `hdir`: Enders use the attacker's facing (Side Enders rotate ±45°). Moves use the horizontal unit vector from the attacker's root to the victim's root (fallback: attacker facing if closer than 0.5 studs). Projectiles use their velocity direction. `KbV` is written with the Q-rule.

#### 3.8.6 Ragdoll lifecycle, wake-up and Tech Roll

- **Authoritative capsule (kinematic rules):** the physical character never tips over. `ALG_Face` holds the yaw (AllAxes alignment keeps it upright), the hurtbox stays an upright box (§3.12.2), and Humanoid walk speed is 0 with jumping disabled. The tumble animation (10–14f), the noodle physics ragdoll and the get-up are **client-side cosmetic** (GDD §13.3), built by Part 6 on the `Ragdoll` transition, with `CanQuery = false` on every visual part.
- **Landing:** while `St = Ragdoll` and `Sub = 0`, the first step with `ctx.grounded` and `e ≥ 3` sets `Sub = e + 1` (landed; now "downed").
- **Wake-up:** on Ragdoll expiry → `Idle` with `InvF = f + Ragdoll.WakeUpIFrames` (30, GDD). Entering any `ATTACK` state sets `InvF = 0` (`Ragdoll.WakeUpIFramesEndOnAttack = true`, td: no attacking with i-frames).
- **Tech Roll** (GDD §6.5): `Jump` pressed while `Sub > 0` and `(e + 1) − Sub < 12` → `TechRoll` (§3.7.2). It ends the combo.

#### 3.8.7 Out-of-combat regeneration (GDD §6.2)

B-class tick on frames where `f % Chassis.RegenTickFrames == 0` (60): `+RegenPerSec` (2 HP) if `f − CombatF ≥ toFrames(RegenDelaySec)` (480), `HP < maxHp`, `RlmId == 0` (disabled inside Realms, GDD §6.2), status `regenBlocked` false (e.g. Decay stacks, GDD §7.3.15) and `St ≠ KO`. (Periodic ticks are the one allowed use of `f % N == 0`; a skipped frame only skips a tick, identically on both peers.)

**Verify (§3.8):** Lune `stun.*`, `damage.*`. Scenarios: `combat.stun_decay` (bot juggle keeps a dummy stunned: hitstun applied at continuous stun ≥ 150 is 16 instead of 27; `ShakeOff` at 210); `combat.otg_once` (two OTG hits after a Downslam → the second logs event `otg_denied`); `combat.pileon` (three bots hit a dummy within 4 s → `PileF` set; the next 3.5 hit deals 2.6, i.e. 3.5 × 0.75 rounded to 0.1); `combat.regen` (the player character set to 50 HP via `Dev.Command combatPatch`, untouched for 8 s → then +2 HP per 60 frames; dummies are excluded because their HP resets, §3.15.2).

---

### 3.9 Breakaway, Algebraic Meter and Awakening, Realm framework

#### 3.9.1 Breakaway (GDD §6.8)

| Rule | Implementation |
|---|---|
| 2 pips | `BkP ∈ {0,1,2}`, start full (`Breakaway.StartPips` = 2, td) |
| Use | In any `STUN` state, `Dash` pressed, `Breakaway.canUse`: `BkP ≥ 1`, `f ≥ BkLockF`. → `Breakaway` (20f, all invulnerable, GDD), `BkP −= 1`, combo reset, grab detach (the victim's own `ALG_Hold.Enabled = false`, `Attachment1 = nil`). Event `breakaway`. |
| Burst | Server, on event: every non-allied character whose root is within `Breakaway.PushRadius` (8 studs, td) of the user → `Stagger` Sub 2 for `PushStaggerFrames` (12, td), push 8 studs (GDD) directly away. If that character was grabbing the user, its grab behavior's `onEnd` runs (Part 4). |
| Refill | B-class tick every `Breakaway.TickFrames` (30, td) while `BkP < 2`: `BkProg += 30 · mult`, `mult = ChassisMods.breakawayRefillMult · (f < PileF and PileOn.BreakawayRefillMult or 1)`. When `BkProg ≥ 1500` (25 s, GDD): `BkP += 1`, `BkProg −= 1500`; at `BkP = 2`, `BkProg = 0`. |
| Damage refill | S-class on each damaging hit: `BkProg += (damage / Breakaway.DamagePerSecondRefund) · 60` (−1 s per 2 damage, GDD), with the same carry rule. |
| Lockout | S-class on every hit: `BkLockF = max(BkLockF, f + hs + Breakaway.LockoutFrames)` (12f after hitstop, GDD). Guard Break: `StF + 30`. An awakened ultimate with `params.cinematicFrames`: `max(BkLockF, f + min(cinematicFrames, toFrames(CinematicMaxSec)))` (≤ 2.0 s, GDD). |
| Visibility | `BkP` replicates; nameplates and HUD show pips (Part 6). The HUD Dash button shows the Breakaway glyph while `St ∈ STUN` (§3.10.5). |
| Finn passive | `breakawayRefillMult = 1.5` below 30 HP (GDD §7.3.1) via `ChassisMods`. |

#### 3.9.2 Algebraic Meter and Awakening (GDD §6.9)

Meter gains (server memory `meter`, clamped 0–100; ignored while awakened, `Meter.GainWhileAwakened = false`, td, because the meter "starts again from 0" at the end):

| Event | Gain | Who |
|---|---|---|
| Damage dealt | +1.0 per HP × `hit.meterGain`; Realm damage ×0.5 | attacker |
| Damage taken | +0.5 per HP | victim |
| Perfect Block | +6 | defender |
| Style Moment (kit-detected, Part 4 emits `style_moment`) | +10 | user |
| KO / assist | +15 / +7 | killer / assisters |

- **Full:** when `meter ≥ 100`, `AwkF0 == 0` and `AwkRdy == false`, the server writes `AwkRdy = true` (S). The flush writes Player `Meter`.
- **Activation** (B, P7): `Awaken` pressed in `NEUTRAL` or `Block` with `AwkRdy` and `AwkF0 == 0` (not from `STUN`: `Awakening.AllowFromStun = false`, td, because Breakaway is the stun escape). Entry effects in `Meter.activate`: `St = Awakening` (72f, invulnerable, GDD); `AwkRdy = false`; `AwkF0 = f`; `AwkEndF = f + 72 + toFrames(DurationSec)` (60 s counted after activation, td reading); `HP = min(maxHp, HP + 10 · realmHealMult)`; `BkP = min(2, BkP + 1)` (if it becomes 2, `BkProg = 0`); `Cd1..Cd4 = 0` (the awakened set starts ready); event `awaken`.
- **Server on `awaken`:** `meter = 0`; shockwave: every non-allied character within 10 studs (GDD) → `Stagger` Sub 2, 12f (td), pushed 10 studs away; life stat `awakened = true`. Render: local 1.2 s camera mini-cinematic; observers see pose + burst (GDD).
- **Awakened mode:** `AwkF0 > 0`. Slots 1–4 map to `StyleRuntime.awakened`. `ChassisMods` adds awakening modifiers (e.g. `m1RangeAdd`, `hurtboxScale ≥ 1`, `walkSpeedMult`). The hurtbox size is multiplied by `hurtboxScale` (§3.12.2).
- **KO extension** (S): each KO while awakened → `AwkEndF = min(AwkEndF + toFrames(KoExtendSec), AwkF0 + 72 + toFrames(MaxSec))` (+5 s, max 90 s, GDD).
- **End** (B): `f ≥ AwkEndF` → `AwkF0 = 0`, `AwkEndF = 0`, `Cd1..Cd4 = 0` (the base set is ready), hurtbox back to scale 1, event `awaken_end`. A 1 s power-down animation plays with no vulnerability penalty (GDD).

#### 3.9.3 Realm framework (GDD §6.10; enough for `decay` A4 "The Well of Power")

Lifecycle:

| Phase | Rule | Writer |
|---|---|---|
| Cast start | Awakened slot 4 whose MoveDef has `behavior = "RealmCast"`, `Cd4` ready, `NEUTRAL` → `RealmCast` (90f, GDD). Queue: if the server sees `#activeRealms ≥ Realm.MaxActive` (2, GDD) on the cast's first step, it reverts the caster to `Idle`, sets `Flg \|= REALMQ` and appends the rid to the FIFO `realmQueue`. When a slot frees, the first queued rid that is alive, still awakened and in `NEUTRAL` is put into `RealmCast` with `StF = f` (S). The entry is dropped when its Awakening ends. | B (start), S (queue) |
| Cast e 0–44 | **Interruptible** (`Realm.InterruptibleFrames` = 45, GDD) by: any grab hit, any hit with `parryable = true`, or a hit whose `guardDamage ≥ guardAt` ("Guard Break"). An interrupt cancels the cast (normal hit consequences apply) and sets `Cd4 = f + toFrames(Realm.InterruptRefundCooldownSec)` (10 s, td; the Realm is not consumed). Other hits deal damage without stun (cast armor). | S |
| Cast e 45–89 | Hyper armor: damage applies, no stun, no interrupt | S |
| Open (server, first step with e ≥ 90) | `id = nextRealmId`; `center` = caster root position; members = every alive character (caster included, allies excluded) whose root is within `Realm.RadiusStuds` (40, GDD) at this moment ("anyone outside the ring when it closes is safe"). Members: `RlmId = id`, `RlmWardF = 0` (S). Caster: `Cd4 = AwkEndF` (one Realm per Awakening, GDD), state → `Idle`. Deferred: create marker Model `Realm_<id>` under `Workspace.Sim` with attributes `Center` (Vector3), `Radius`, `OpenF`, `EndF = OpenF + 720` (12 s, GDD), `RuleId` (= MoveDef id, e.g. `"decay.a4"`), `Caster` (rid). Event `realm_open` (render: ring close, 12f server-wide time-dilation VFX, GDD §6.13). | S |
| Active | Each server step: (1) members outside `radius − Realm.BoundaryMarginStuds` (1, td) are moved back inside with `PivotTo` along the radial direction (S); (2) `Realm.rules[RuleId].tick` runs (Part 4); (3) out-of-combat regen is disabled for members (§3.8.7). | S |
| Close | At `f ≥ EndF`, on caster KO, or on caster despawn: members `RlmId = 0`; marker attribute `Dead = true` (destroyed in the deferred pass); free the slot; dequeue. Event `realm_close`. | S |

Stubborn Heart (B, in the member's own FSM): `St = Block`, `ctx.inEnemyRealm`, block elapsed `e ≥ Realm.StubbornHeart.HoldFrames` (60 = 1 s, GDD), `RlmWardF == 0`, `BkP ≥ PipCost` (1) → `BkP −= 1`, `RlmWardF = f + toFrames(WardSec)` (5 s, GDD), event `stubborn_heart`. `RlmWardF > 0` stays as the "used in this Realm" marker until the server assigns a new `RlmId` (which resets it to 0): "once per Realm per player" (GDD).

Rules hook (Part 4 fills `Realm.rules[ruleId]` in `Core/Realm.luau`; framework functions above are Part 3's):

```lua
export type RealmView = { id: number, center: Vector3, radius: number, openF: number, endF: number, casterRid: number, ruleId: string }
export type MemberView = { rid: number, isCaster: boolean, warded: boolean, s: Fsm.CharState }
export type RealmRules = {
	healMult: ((m: MemberView) -> number)?,         -- multiplier applied to every heal on m (Well of Power: -1 for enemies)
	cooldownMult: ((casterRid: number) -> number)?, -- applied when the caster starts a move (Well of Power: 0.7)
	tickFrames: number?,                            -- e.g. 120 (every 2 s)
	tick: ((r: RealmView, m: MemberView, f: number) -> { statusAdd: Moves.Status?, hit: Fsm.HitDef? }?)?, -- server only
}
```

Heals route through one function, `Adapter.heal(entry, amount, source)` (server) or `HP += amount · ctx.realmHealMult` (B-class heals such as the Awakening heal). The adapter resolves `ctx.realmHealMult` and `ctx.mods.cooldownMult` from the member's `RlmId` → `workspace.Sim:FindFirstChild("Realm_" .. id)` → `RuleId` attribute → `Realm.rules` (FindFirstChild and GetAttribute are Simulation-Access), so both peers agree. Warded members (`f < RlmWardF`) get `healMult = 1` and no rule ticks. Realm damage is tagged `hit.realm = true` (meter ×0.5, GDD §6.9).

**Verify (§3.9):** Lune `breakaway.*`, `meter.*`, `realm.*`. Scenarios: `combat.breakaway` (dummy hits the player into hitstun; Dash at hit+5f is refused (lockout), at hit+16f accepted; `BkP` 2→1; the attacker within 8 studs ends in `Stagger`); `combat.breakaway_refill` (after one use, wait 25 s untouched in a play session; `BkP` returns to 2 within ±30 frames; then take 10 damage at `BkP = 1` and check `BkProg` jumped by 300); `combat.awakening` (`Dev.Command setMeter 100` → `AwkRdy=true`; press G → `St = Awakening` for 72f, invulnerable to dummy hits, `HP +10`, `BkP +1`, `AwkEndF − AwkF0 = 3672`); `combat.awaken_ko_extend` (3 KOs while awakened → `AwkEndF` +900, capped at `AwkF0 + 5472`); M1-G: `combat.realm_cast_interrupt` (parryable hit at cast e = 30 → cancelled; at e = 60 → not cancelled), `combat.realm_open_members` (3 bots at 10/35/45 studs → members 2), `combat.realm_cap_queue` (third cast queued, opens when one closes), `combat.stubborn_heart` (member holds Block 1 s → `RlmWardF` set, `BkP −1`; a second attempt in the same Realm does nothing).

---

### 3.10 Input layer (Input Action System)

#### 3.10.1 The `Combat` InputContext template (built by `BuildInputs`)

`BuildInputs.build()` (Edit mode, Part 0 builder pattern) creates `ReplicatedStorage.Inputs` (Folder) → `Combat` (InputContext, `Priority = 2000`, `Sink = false`, `Enabled = true`) → one InputAction per row below. Each action gets child InputBindings named `KBM`, `Pad`, `Touch`, `Script`, each only where the table has a value. The single source of truth is `InputMap.ACTIONS` (the builder iterates it). `Touch` bindings are created with `UIButton = nil`; buttons are assigned per client at runtime (§3.10.3). `Script` bindings have `Type = Enum.InputBindingType.Scriptable`.

| Action | `Type` | Bit (`Fsm.IN`) | `KBM` (`KeyCode`) | `Pad` (`KeyCode`) | `Touch` button id | `Script` (who fires it) | GDD |
|---|---|---|---|---|---|---|---|
| `M1` | Bool | 1 | `MouseLeftButton` | `ButtonX` | `PunchBtn` | tests | §15.2 |
| `Move1` | Bool | 2 | `One` | `ButtonY` | `Move1Btn` | tests | §15.2 |
| `Move2` | Bool | 4 | `Two` | `ButtonB` | `Move2Btn` | tests | §15.2 |
| `Move3` | Bool | 8 | `Three` | `ButtonR1` | `Move3Btn` | tests | §15.2 |
| `Move4` | Bool | 16 | `Four` | `ButtonL1` | `Move4Btn` | tests | §15.2 |
| `Quirk` | Bool | 32 | `R` | `DPadRight` | `QuirkBtn` | tests | §15.2 |
| `Awaken` | Bool | 64 | `G` | `ButtonR3` with `PrimaryModifier = ButtonL3` (LS+RS; verify SA-14, fallback `ButtonL3` alone) | `AwakenBtn` | tests | §15.2 |
| `Block` | Bool | 128 | `F` | `ButtonL2` (`PressedThreshold` 0.5, `ReleasedThreshold` 0.4, td) | `BlockBtn` | tests; hold-to-toggle setting (GDD §16.3) | §15.2 |
| `Dash` | Bool | 256 | `Q` | `ButtonR2` (0.5 / 0.4) | `DashBtn` | tests | §15.2 |
| `Jump` | Bool | 512 | `Space` | `ButtonA` | `JumpBtn` | tests | §15.2 |
| `Run` | Bool | 1024 | `LeftControl` | — | — | tests | §6.2 |
| `RunAuto` | Bool | 2048 | — | — | — | InputController (double-tap W; full stick push) | §6.2 |
| `Launcher` | Bool | 4096 | — | — | — | HudController (mobile Launcher toggle) | §15.2 |
| `AimLock` | Bool | 8192 | — | — | — | InputController (Shift-Lock active) | §15.2 |
| `Aim` | Direction3D | — | — | — | — | InputController (camera look, assisted) | §15.2 |

- `Sink = false` (deliberate): our context must not block keys from the default PlayerScripts contexts (Space/ButtonA jump, movement). None of our keys collide with default gameplay bindings. The guide's `Sink = true` recipe is optional [Docs `input-action-system.md`].
- Emote (B / D-pad up) and Style Wheel (Tab / View) are UI actions in Part 6's `UI` context, not simulation inputs.
- Remapping (GDD §15.2): SettingsController rewrites `KBM`/`Pad` `KeyCode`s on the **client's own** context from `settings.binds` (outside the sim). The server never needs the key, only the action state.
- `InputAction:Fire` is deprecated and banned; tests and synthesized inputs use `InputBinding:Fire(state)` on the `Script` binding [Docs `InputBinding.yaml`].

**Verify:** `BuildInputs.verify()` → `{ok = true}` when the tree matches `InputMap.ACTIONS` exactly (15 actions, binding names and KeyCodes, Scriptable types); scenario `input.attach_contexts` (each joined player has `Player.Combat` with 15 actions).

#### 3.10.2 Per-player cloning (server)

`InputMap.attach(player: Player): InputContext` is called by PlayerService at join step 2 (Part 2 §2.1.3 D), outside the sim. It clones `ReplicatedStorage.Inputs.Combat` and parents the clone **directly under the Player** (an InputContext must be a descendant of the Player [Docs `server-authority/index.md`]). It is idempotent (an existing `Player.Combat` is kept) and survives respawns, because `LoadCharacterAsync` clears `PlayerGui`, not Player children. Log: `[ALG][Input] attached user=<id> actions=15`.

The server caches `InputMap.actionsOf(ctx)` (name → InputAction) in the player's roster entry at `CharacterAdded`.

#### 3.10.3 Client duties (`Controllers/InputController.luau`, outside the sim)

```lua
export type InputControllerApi = {
	context: () -> InputContext?,                        -- Player.Combat once present
	bindTouch: (action: string, button: GuiButton) -> (), -- sets <action>.Touch.UIButton; Part 6 HUD calls it per button
	touchRoot: () -> ScreenGui,                          -- PlayerGui.ALG_Touch (ResetOnSpawn=false), created here; Part 6 fills it
	setCombatEnabled: (on: boolean) -> (),               -- Combat.Enabled (menus); not Sim-Access, client-only
	fire: (action: string, state: boolean | Vector3) -> (), -- <action>.Script:Fire(state); tests and synthesized inputs
}
```

`InputController.start()` also starts the client simulation: `Adapter.startClient()` (roster tracking, §3.11.1), then `Simulation.start(Combat.Chassis.Sim.Priority)` and `Simulation.startAnimation()`.

Synthesized inputs, computed every `RenderStepped` and fired through the `Script` binding only when they change:

| Action | Rule |
|---|---|
| `Aim` | `dir = workspace.CurrentCamera.CFrame.LookVector`. On touch/gamepad, when the assist setting is on and not in Ranked, rotate `dir`'s yaw toward the nearest enemy root within a 12° cone and 40 studs (GDD §15.2: "rotates the character only"; td range). Fire when the angle to the last sent value exceeds `Chassis.Input.AimMinDeltaRad` (0.02), or as a keep-alive every `AimKeepAliveSec` (0.5 s; IAS deduplicates identical states, so the keep-alive only matters after a context re-enable). |
| `AimLock` | `UserInputService.MouseBehavior == Enum.MouseBehavior.LockCenter` (Shift-Lock, default ON per GDD §15.2) or the touch Shift-Lock toggle (Part 6). |
| `RunAuto` | Keyboard: two `W` presses within `Chassis.Input.RunDoubleTapFrames` (15f; detected with `UserInputService.InputBegan` here, *outside* the sim) latch true until `Humanoid.MoveDirection` is zero for 6 frames. Gamepad/touch: `Humanoid.MoveDirection.Magnitude ≥ RunStickThreshold` (0.95; analog magnitude verify SA-11). |
| `Launcher` | HudController toggles it on tap (GDD §15.2: "a Launcher toggle appears beside Punch on the 3rd hit"). |

Touch buttons: Part 6 builds them inside `touchRoot()` with `Name` = the touch id in §3.10.1 and calls `bindTouch(action, button)`. The HUD re-binds after any rebuild. A button whose `UIButton` binding is set works for Bool actions [Docs `InputBinding.yaml`].

#### 3.10.4 Reading inputs inside the sim (`InputMap.read`)

```lua
--!strict
-- src/Shared/Sim/InputMap.luau (excerpt). read() is SIM-SAFE: only InputAction:GetState (Simulation-Access).
local Fsm = require("../Core/Fsm")
export type Actions = { [string]: InputAction }
local InputMap = {}

local BOOL_ORDER = { "M1", "Move1", "Move2", "Move3", "Move4", "Quirk", "Awaken", "Block", "Dash", "Jump",
	"Run", "RunAuto", "Launcher", "AimLock" }

local function flat(v: Vector3): Vector3
	local h = Vector3.new(v.X, 0, v.Z)
	if h.Magnitude > 1 then
		return h.Unit
	end
	return h
end

function InputMap.read(a: Actions, prevBits: number, moveDir: Vector3, rootLook: Vector3): Fsm.InputFrame
	local bits = 0
	for _, name in BOOL_ORDER do
		local act = a[name]
		if act ~= nil and act:GetState() == true then
			bits = bit32.bor(bits, Fsm.IN[name])
		end
	end
	local raw = if a.Aim then a.Aim:GetState() else nil
	local aim = rootLook
	if typeof(raw) == "Vector3" and raw == raw and raw.Magnitude > 0.5 and raw.Magnitude < 1.5 then
		aim = raw.Unit -- sanity: finite, near-unit (SA docs: inputs "should be checked for sanity")
	end
	return {
		bits = bits,
		pressed = bit32.band(bits, bit32.bnot(prevBits)),
		aim = aim,
		moveDir = flat(moveDir),
	}
end
```

- `moveDir` comes from `Humanoid.MoveDirection` (Sim-Access, driven by the default IAS-based controls on both peers [Docs `Workspace.yaml` `PlayerScriptsUseInputActionSystem`]).
- `PrevIn` is written each step only when it changes, so edges survive resimulation (SA docs pattern: "record the previous frame's input in an attribute").
- Input cadence signal (server, outside the sim via the event queue): more than 20 rising edges per second on any action, sustained for 3 s → `ModerationService.signal(player, "sim_input", {reason = "cadence"})` (td; a signal only, Part 2 §2.11.3).

#### 3.10.5 Breakaway contextual remap

There is no separate Breakaway action. `Dash` pressed in a `STUN` state *is* Breakaway (P5, §3.2.3), which matches the GDD control table (Q / Dash button / RT "when stunned"). The HUD swaps the Dash button glyph and label to `Skin.text("ui.hud.breakaway")` while the local `St ∈ STUN` and `BkP ≥ 1` (Part 6). The IAS binding never changes.

#### 3.10.6 Bots

Bots are not Players, so they cannot own InputContexts [api-sheet §3]. They use the intent-attribute protocol (§3.15.1), read by `InputMap.readBot(root, prevBits): InputFrame` (GetAttribute only).

**Verify (§3.10):** scenario `input.mcp_m1_reaches_server` (MCP `user_mouse_input` click on the viewport during a play session; within 10 frames `execute_luau` on Server reads the local character's `St == 3`). This depends on P1 spike S9; if S9 failed, the scenario uses `InputController.fire("M1", true)` then `false`. Scenario `input.scriptable_aim_replicates` (client fires `Aim = (1,0,0)`; the server's `Aim:GetState()` equals it within 0.001 after ≤ 10 frames). Scenario `input.default_jump_works` (Space press via MCP → Humanoid leaves the ground; proves `Sink = false`).

---

### 3.11 Simulation adapter (`Sim/Simulation.luau`, `Sim/Adapter.luau`)

#### 3.11.1 Roster (filled outside the sim)

```lua
--!strict
-- src/Shared/Sim/Adapter.luau (types excerpt). Spec §3.11.
export type Movers = {
	vel: LinearVelocity, face: AlignOrientation, hold: AlignPosition, grab: Attachment, rootAtt: Attachment,
}
export type ServerData = {
	ring: Rewind.Ring, rttEma: number, rewindFrames: number, meter: number,
	hitReg: { [string]: { [number]: boolean } },       -- key -> victim rid -> true (cleared on state entry)
	ledger: { { rid: number, amount: number, f: number } }, lastHit: { rid: number, f: number }?,
	pileHits: { { rid: number, f: number } }, life: { [string]: number | boolean },
	spawnCf: CFrame?, queuedPatches: { { [string]: any } },
}
export type RosterEntry = {
	rid: number,                 -- players: UserId; bots: -1000000 - n (BotId "b<n>")
	player: Player?, botId: string?, isLocal: boolean,
	model: Model, root: BasePart, humanoid: Humanoid, hurtbox: BasePart, movers: Movers, proto: BasePart,
	actions: { [string]: InputAction }?,                -- players only
	style: Fsm.StyleRuntime, cfgVersion: number, cfg: any, -- pinned Combat section
	anims: { [string]: Animation },                     -- Animation instances by logical key (never tracks)
	simulate: boolean, ready: boolean,
	srv: ServerData?,                                   -- server only
}
export type AdapterApi = {
	track: (model: Model, opts: { player: Player?, botId: string?, dummy: boolean?, styleId: string, cfgVersion: number, spawnCf: CFrame? }) -> RosterEntry,
	setIntentProvider: (fn: (f: number, bots: { RosterEntry }) -> ()) -> (), -- server; BotService.init (§3.15.1)
	untrack: (model: Model) -> (),
	list: () -> { RosterEntry },                        -- sorted by rid ascending; never mutated inside the sim
	entryOfModel: (model: Model) -> RosterEntry?,
	entryOfHurtbox: (part: BasePart) -> RosterEntry?,
	entryOfPlayer: (player: Player) -> RosterEntry?,
	startClient: () -> (),                              -- client: tracks every character and bot (local one simulated)
	step: (f: number, dt: number, isServer: boolean) -> (), -- called by Simulation only
	animate: (f: number, isServer: boolean) -> (),      -- called by Simulation.startAnimation's binding only
	heal: (e: RosterEntry, amount: number, source: string) -> (), -- server; applies realm heal multipliers
}
```

Lifecycle:
- **Server:** CombatService connects `Player.CharacterAdded` (and BotService calls directly for bots). Handler: `model:WaitForChild("HumanoidRootPart", 10)` → `Hurtbox.prepare(model)` (§3.11.2) → `Adapter.track(model, pending[player])` with `simulate = true`, `ready = true`, `srv` filled. `Player.CharacterRemoving` → `Adapter.untrack`, `ConfigReplicator.unpin(v)`.
- **Client (`Adapter.startClient`):** for every Player's `CharacterAdded` and every child of `Workspace.Bots`, it waits (outside the sim) for `Hurtbox`, `ALG_Vel`, `ALG_Face`, `ALG_Hold`, `ALG_Grab`, `ALG_Proto` and the HRP attribute `St`, then tracks with `simulate = (player == LocalPlayer)`. For the local entry it also needs `Player.Combat` (actions), `EquippedStyle` and `CombatCfg` resolved to `Runtime.at(v).Combat` (Part 2 §2.6.6). Only then `ready = true`. Remote entries are tracked only so that hurtbox parts map to entries for predicted contact (§3.12.7) and for render.
- Roster arrays and maps are mutated only in these handlers, never inside the sim.

#### 3.11.2 Per-character preparation (`Combat/Hurtbox.prepare`, server, outside the sim)

Created on the server at `CharacterAdded` (bots: before parenting). Names prefixed `ALG_` so they never collide with rig parts. All replicate to clients.

| Instance | Class | Parent | Properties |
|---|---|---|---|
| `Hurtbox` | Part | character Model | `Anchored = true`, `CanCollide = false`, `CanTouch = false`, `CanQuery = true`, `CastShadow = false`, `Transparency = 1`, `Size = (2.2, 5.2, 2.2)` (GDD §6.2), `CollisionGroup = "Hurtbox"` |
| `ALG_Proto` | Part | character Model | `Anchored = true`, `CanCollide/CanQuery/CanTouch = false`, `Transparency = 1`, `Size = (1,1,1)`: clone source for stitched carriers (§3.11.5) |
| `ALG_Vel` | LinearVelocity | HRP | `Attachment0 = RootAttachment`, `RelativeTo = World`, `ForceLimitsEnabled = false`, `VelocityConstraintMode = Vector`, `PrimaryTangentAxis = (1,0,0)`, `SecondaryTangentAxis = (0,0,1)`, `Enabled = false` |
| `ALG_Face` | AlignOrientation | HRP | `Mode = OneAttachment`, `Attachment0 = RootAttachment`, `AlignType = AllAxes`, `RigidityEnabled = true`, `CFrame = CFrame.Angles(0, yaw0, 0)`, `Enabled = false` |
| `ALG_Hold` | AlignPosition | HRP | `Mode = TwoAttachment`, `Attachment0 = RootAttachment`, `Attachment1 = nil`, `Responsiveness = 100`, `MaxForce = 1e7`, `MaxVelocity = 200` (td), `Enabled = false` (being grabbed; Part 4 sets `Attachment1` to the grabber's `ALG_Grab`) |
| `ALG_Grab` | Attachment | HRP | `CFrame = CFrame.new(0, 0, -2.5)` (td; where this character holds a victim) |

Also: every `BasePart` in the model (now and via `DescendantAdded`) gets `CollisionGroup = "Characters"` except `Hurtbox`/`ALG_Proto`. Humanoid: `BreakJointsOnDeath = false`, `RequiresNeck = false`, `UseJumpPower = false`, `JumpHeight = 7.5`, `WalkSpeed = 18`. `RootAttachment` is the rig's HRP attachment; if it is missing, create `ALG_RootAtt` at the origin and use it (verify on the published rig in SA-11). Each constraint is asserted to have non-nil attachments after creation; a missing attachment logs `[ALG][Combat] ERROR mover_unwired` (a nil attachment is a silent failure per Cakeman).

#### 3.11.3 The step (`Simulation` + `Adapter.step`)

```lua
--!strict
-- src/Shared/Sim/Simulation.luau — the single simulation binding on both peers. Spec §3.11.3.
local RunService = game:GetService("RunService")
local Adapter = require("./Adapter")
local Frames = require("../Core/Frames")
local Log = require("../Log")

local IS_SERVER = RunService:IsServer() -- captured at require time, outside the sim (§3.1.5)
local Simulation = {}
local simConn: RBXScriptConnection? = nil
local animConn: RBXScriptConnection? = nil

function Simulation.start(priority: number)
	assert(simConn == nil, "[ALG][Sim] already bound")
	simConn = RunService:BindToSimulation(function(dt: number)
		Adapter.step(Frames.fromTime(time()), dt, IS_SERVER)
	end, Enum.StepFrequency.Hz60, priority) -- MUST pass Hz60: the default is Hz30
	Log.info("Sim", `bound hz=60 priority={priority} side={if IS_SERVER then "server" else "client"}`)
end

function Simulation.startAnimation()
	assert(animConn == nil, "[ALG][Sim] animation already bound")
	animConn = RunService:BindToAnimation(function(_dt: number)
		Adapter.animate(Frames.fromTime(time()), IS_SERVER)
	end, Enum.StepFrequency.Hz60)
end

function Simulation.stop()
	if simConn then simConn:Disconnect() simConn = nil end
	if animConn then animConn:Disconnect() animConn = nil end
end

return Simulation
```

Who starts it: on the server, `CombatService.init()` runs `Hurtbox.ensureGroups()` and `CombatService.start()` calls `Simulation.start(cfg.Combat.Chassis.Sim.Priority)` then `Simulation.startAnimation()` (config read outside the sim). On the client, `InputController.start()` does the same after `Adapter.startClient()` (§3.10.3).

`Adapter.step(f, dt, isServer)` runs these phases in order. `scratch` tables are module-level but cleared at the start of every step, so no state survives between steps on the client.

| Phase | Server | Client (local entry only) |
|---|---|---|
| 0 Pose | For each ready entry: `Hurtbox.place(e)` (CFrame = root position + `(0, OffsetY·scale, 0)`, yaw-only rotation from `ALG_Face`/root; `Size` = base × `hurtboxScale`); `Rewind.record(e.srv.ring, f, cf, size)` | — |
| 0b Init | Entries without `St`: `Adapter.initState(e, f)` writes every SPEC default (`HP = style.hp`, `ProtF = f + 480`, `StF = f`), `root:PivotTo(spawnCf)`; log `[ALG][Sim] ready rid=<n> cfg=<v>` via queue; apply any `queuedPatches` (test seam) | — |
| 0c Intents | Call the BotService intent provider for bot entries (§3.15.1) | — |
| 1 Step | For each ready entry (rid ascending): `readState` → `InputMap.read`/`readBot` → wall probe if eligible (§3.5.5) → `Fsm.step` → behavior hook (§3.11.4) → `writeDiff` → `applyMotion`/`applyFacing`/`Loco.apply` → `spawnAll` → if `out.entered` clear `srv.hitReg` → collect hits and events | same, for the local entry only (`assert(e.isLocal)`) |
| 1b Carriers | Step every live carrier in `workspace.Sim` (§3.11.5); collect their hit requests | Step carriers with `Own == localRid` only |
| 2 Query | Rewound queries (§3.12.5) → contact list | Live queries for local requests (§3.12.7) |
| 3 Apply | `Damage.resolve` per contact; S + BQ patches; meta updates (§3.12.6) | Attacker-self BQ patch on first contact per action |
| 4 Post | Realm phases (§3.9.3), pile-on expiry, `AwkRdy` writes, KO detection (§3.13.1), `scheduleFlush()` | — |

The client never writes attributes or constraint properties of a non-local entry. `Adapter.step` asserts this in Studio (`RunService:IsStudio()` captured at load).

#### 3.11.4 Movers, locomotion and behavior hooks

```lua
-- Adapter.applyMotion (SIM-SAFE). Reads the constraint's own (predicted) properties; keeps no Lua cache.
local function applyMotion(e: RosterEntry, m: Fsm.MotionCmd)
	local vel = e.movers.vel
	if m.mode == "none" then
		if vel.Enabled then vel.Enabled = false end
	elseif m.mode == "planar" then
		vel.VelocityConstraintMode = Enum.VelocityConstraintMode.Plane
		vel.PlaneVelocity = Vector2.new(m.vx, m.vz)
		if not vel.Enabled then vel.Enabled = true end
	else
		local v = if m.mode == "vector" then m.v else Vector3.zero
		vel.VelocityConstraintMode = Enum.VelocityConstraintMode.Vector
		vel.VectorVelocity = v
		if not vel.Enabled then vel.Enabled = true end
	end
end
```

(Assigning an identical value to a property is harmless; the "on change" rule is an optimisation, not a correctness requirement.)

`Loco.apply` (Humanoid implementation): `WalkSpeed`, `JumpHeight`, `SetStateEnabled(Enum.HumanoidStateType.Jumping, canJump)`, `Jump = true` when `cmd.jump`, and `AutoRotate` from the facing mode. For bots it also calls `humanoid:Move(inp.moveDir)`. Players' movement comes from the default controls.

`applyFacing`: `auto` → `ALG_Face.Enabled = false`, `AutoRotate = true`; `set` → `AutoRotate = false`, `ALG_Face.CFrame = CFrame.Angles(0, yaw, 0)`, `Enabled = true`; `hold` → `AutoRotate = false`, `Enabled = true`, CFrame unchanged. When leaving `auto` for `hold` (a stun begins in neutral), the adapter first writes the root's current yaw (rounded, Q-rule) so the hold has a defined target.

Behavior hooks (Part 4 contract, §3.4.2): while `St = Move`, the adapter calls `Behaviors[def.behavior].step(bc)` after `Fsm.step` on both peers for the owner. `onContact` runs on the server for every contact of that move, and on the owner client only for its predicted first contact. `onEnd` runs on state exit, including interrupts. A behavior's `patch` may write only B-class fields from `Attr.SPECS`; the adapter rejects others and logs `[ALG][Combat] ERROR behavior_patch field=<name>` (Studio).

#### 3.11.5 Stitched carriers (projectiles, zones, summons) and despawn

- **Creation (inside the sim, both peers for the owner):** `Adapter.spawn(e, req, f)` does `local p = e.proto:Clone()` (stitching: `Clone` uses the source's GUID as the context seed and works when the source replicated to both peers [Docs `techniques.md`]), sets `Name = "C_<rid>_<f>_<n>"`, `CFrame = req.origin`, attributes `Own` (rid), `Kind`, `MoveId`, `F0` (= f), `Dead` (= false), `Hits` (= 0) plus up to 10 `req.params` keys, and finally `p.Parent = workspace.Sim` (parent last, in the same frame, as the docs require).
- **Server-only creation** (zones whose position depends on other characters; Realm markers): same helper, server branch only. These are not predicted.
- **Prediction:** carriers are anchored. Anchored parts are not predicted under `PredictionMode.Automatic` (Cakeman rule 7), so the owner client calls `RunService:SetPredictionMode(p, Enum.PredictionMode.On)` in a `workspace.Sim.ChildAdded` handler (outside the sim; client-only API) when `p:GetAttribute("Own") == localRid`.
- **Stepping (phase 1b):** `Carriers.step` calls Part 4's `Behaviors.Projectile.stepCarrier(view, f)` / `Behaviors.Zone.stepCarrier(view, f)`. These return the new CFrame (written to `p.CFrame`, Sim-Access), hit requests (projectiles: `Cast` from the previous to the current position, so nothing tunnels), and `dead` (lifetime ≥ `params.lifeF`, capped by `Chassis.Sim.MaxCarrierLifeFrames` = 600, or the hit limit reached on the server). Zones produce `tick` hits every `params.tickFrames` (server only).
- **Despawn (outside the sim):** the sim only sets `Dead = true`. In the deferred flush (§3.11.6) the server destroys carriers whose `Dead` has been true for ≥ 2 frames (so clients see the flag first) and any carrier older than the cap. Clients never destroy carriers; render hides `Dead` carriers at once.
- **Spike SA-09** checks GUID stitching when two characters spawn carriers in the same frame (the docs' per-script call counter could make server and client counters differ). **Fallback:** server-only carriers. The owner client then renders a cosmetic "ghost" projectile from its predicted move timeline (RenderController, `CanQuery = false`, group `CosmeticNoQuery`) and hides it when the authoritative carrier appears or the state is undone.

#### 3.11.6 Server event queue and deferred flush

Inside the server step, all meta work is appended to `metaQueue` as `SimEvent`s plus typed records (`{k = "damage", ...}`, `{k = "destroy", inst = ...}`, `{k = "marker", ...}`). The first append in a step calls `task.defer(Adapter.flush)` once; `task.defer` runs outside simulation restrictions [Docs `server-authority/index.md`]. `flush` runs in queue order:
1. Player attribute writes (`Meter`, `Streak`).
2. `CombatService.damage:Fire(...)`, `.ko`, `.streak`, `.lifeEnded`; then `Net.server.fire("Life.Summary", ...)` and `Net.server.fireAll("Combat.Feed", ...)`.
3. Respawn scheduling (`task.delay(Respawn.DelaySec, CombatService.spawnAsync, player)`).
4. Carrier/marker creation and destruction; Realm marker attributes.
5. `ModerationService.signal`, `Telemetry` calls, `Log` lines (`[ALG][Combat] ko …`, `[ALG][Combat] respawn …`).

Clients have no queue: client-side consequences are expressed only as attributes, and render reacts to them.

#### 3.11.7 Rollback handling

- No client Lua state influences the step (scratch tables are cleared per step; roster data is static per life). So nothing in the sim needs restoring on rollback.
- `RunService.Rollback(time)` "fires after state is rolled back, before resim begins" [Docs `RunService.yaml`]. RenderController connects it and calls `onRollback(Frames.fromTime(time))`, which marks predicted effects started after that frame as *unverified*. They are re-checked on the next `RenderStepped` (§3.11.8).
- Constraint targets (`ALG_Vel`, `ALG_Face`, `ALG_Hold`) are predicted properties and are restored by the engine. The adapter keeps no "last written" Lua caches; every "write on change" test compares against the instance's current property or attribute value.
- Debug aid (Studio, plugin capability): Part 7's AlgebraicTestHarness plugin may connect `RunService.Misprediction` and log per-attribute misprediction counts (`[ALG][Sim] mispredict attr=<name> n=<count>` every 10 s) to find noisy writers.

#### 3.11.8 Animation and render contract

**Animation (sim-mirrored).** Following the SA guidance to "mirror animation logic" and avoid track caching [Docs `techniques.md`], `Adapter.animate(f, isServer)` runs in `RunService:BindToAnimation(..., Enum.StepFrequency.Hz60)` on both peers for simulated entries. Server: every character. Client: the local one. It maps `(St, Sub, style)` to an animation key via `AnimKeys` (table below), gets the track with `animator:GetTrackByAnimationId(anim.AnimationId) or animator:LoadAnimation(anim)` (never cached), plays it with priority `Action2` (`Action4` for Awakening/RealmCast), stops tracks of the previous state, and sets `track.TimePosition = e / 60` when `|TimePosition − e/60| > 2/60`. During hitstop it calls `AdjustSpeed(0)`, then `AdjustSpeed(1)`. Locomotion stays with the default Animate script. At most `Render.MaxCombatTracks` = 2 combat tracks play per character (one action + one reaction) until spike SA-05 settles the per-Animator limit.

| State / Sub | Anim key (`Assets` key; Part 6 publishes placeholders) |
|---|---|
| M1 1–3, 11–13, 20 | `StyleDef.m1.anim` .. `_<sub>` (e.g. `anim.hero.m1_1`) |
| Ender 1–5 | `anim.<style>.ender_<sub>` |
| Move | `MoveDef.anim` |
| Block / BlockStun | `anim.common.block` / `anim.common.block_hit` |
| Dash 1–5, TechRoll | `anim.common.dash_<sub>`, `anim.common.techroll` |
| Hitstun, Stagger, Launched, WallSplat, GuardBroken, Grabbed | `anim.common.<state lowercased>` |
| Ragdoll | `anim.common.tumble` (first 12f), then the cosmetic ragdoll (render) |
| Breakaway, ShakeOff, Awakening, RealmCast, KO | `anim.common.breakaway`, `.shakeoff`, `anim.<style>.awaken`, `anim.<style>.realm`, `anim.common.ko` |

**Render (client, `RenderController`, RenderStepped).** Per streamed character it keeps `last = {St, StF, Sub, Cnt, HsF, BkP, AwkF0, RlmId, HP, Flg, CFlags}` and dispatches these transitions to Vfx/Sfx/Camera/Appearance controllers (Part 6 implements):

| Transition (detected by diff) | Effect | Predicted for local? | Undo rule |
|---|---|---|---|
| `St/StF` enter (new `StF` > old `StF`) | state VFX/SFX (dash puffs, block shield, breakaway sparkles, awakening aura + local 1.2 s camera cinematic + music sting) | yes | If within 15f the identity `(St, StF)` is replaced by one with `StF ≤` the effect's `StF` (rollback), stop the effect |
| `Cnt` 0→1 / 0→2 on the local attacker | hit spark at the hitbox centre, hit SFX (rising pitch per combo), light/medium camera shake (GDD §6.13) | yes | `Cnt` back to 0 within 15f → fade the spark |
| Victim `HP` decrease or `STUN` entry | hit reaction flash, damage number (Training/accessibility), combo counter from victim `CHits` ≥ 3 (GDD §15.3) | no (authoritative) | — |
| `Flg PBHIT` set / attacker `Stagger` Sub 1 | "ding!" + star flash; 6f freeze both | no | — |
| `St = GuardBroken` | glass-candy shatter | no | — |
| `St = Ragdoll` | hide the real rig locally (`LocalTransparencyModifier = 1`), spawn a cosmetic ragdoll clone (`CanQuery = false`, group `CosmeticNoQuery`) for tumble → noodle → get-up (GDD §13.3) | no | restore on exit |
| `St = KO` | candy-sparkle poof (GDD §6.11); hide the body after `Respawn.BodyHideSec` | no | — |
| Realm marker appears / `Dead` | ring close, 12f time-dilation VFX / ring fade | no | — |

Rules: never use `GetAttributeChangedSignal` for one-shots (it re-fires during resim); reduced-motion and reduced-VFX settings (GDD §16.3) scale effects but never hide telegraphs or hurtbox debug.

**Verify (§3.11):** scenario `sim.client_writes_local_only` (instrumented Studio build: the client adapter's assert never fires during a 2-client brawl); scenario `sim.carrier_stitch` (SA-09); scenario `render.undo_spark` (network sim 150 ms; local attacker whiffs a dummy that the client wrongly predicts as hit by teleporting the dummy server-side at the active frame; the spark is removed within 15f).

---

### 3.12 Hit resolution

#### 3.12.1 Collision groups

Registered with `workspace:RegisterCollisionGroup` (Sim-Access; `PhysicsService` collision APIs are deprecated; max `workspace:GetMaxCollisionGroups()` = 32 [Docs `WorldRoot.yaml`]). `Combat/Hurtbox.ensureGroups()` is idempotent: register each missing group (`IsCollisionGroupRegistered`), then set the full matrix with `CollisionGroupSetCollidable`. It runs in `BuildInputs.build()` (Edit mode, so the groups persist in the place file; registering is costly at runtime) and again in `CombatService.init()`, which only registers what is missing and logs `[ALG][Combat] WARN groups_missing <names>`.

| Group | Members | Collidable with |
|---|---|---|
| `Default` | map geometry, terrain | `Default`, `Characters` |
| `Characters` | every character/bot body part | `Default` only (no body-to-body contact: "PvP contact always mispredicts", Cakeman; td) |
| `Hurtbox` | hurtbox parts, summon hurtboxes (Part 4) | `HitQuery` only |
| `HitQuery` | none (query-only group) | `Hurtbox` only |
| `WorldQuery` | none (query-only group) | `Default` only |
| `CosmeticNoQuery` | client visual parts (ragdoll clones, VFX meshes, ghosts) | nothing |

6 of 32 groups. Spike SA-10 confirms that Humanoid floor detection ignores `Hurtbox` parts and that groups registered in Edit mode exist on clients. **Verify:** `BuildInputs.verify()` checks the matrix with `workspace:CollisionGroupsAreCollidable`.

#### 3.12.2 Hurtbox

- One anchored box part per character (§3.11.2). `Enum.PartType` has no capsule (Ball, Block, Cylinder, Wedge, CornerWedge [Docs]), so the GDD capsule 2.2 × 5.2 is approximated by its bounding box. `GetPartBoundsInBox` tests bounding boxes anyway.
- Pose (server, phase 0): position = root position + `(0, Chassis.Hurtbox.OffsetY · scale, 0)` (−0.3, td; calibrated in SA-11); rotation = yaw only; size = `(2.2, 5.2, 2.2) · scale` with `scale = ChassisMods.hurtboxScale` (≥ 1, GDD §6.2). It stays upright in every state, including Ragdoll (the authoritative capsule; GDD §13.3).
- Clients receive hurtbox poses by replication. Spike SA-07 confirms that an anchored part inside a predicted character model does not disturb the character's prediction. Fallback: parent hurtboxes to `workspace.Sim` as `HB_<rid>`.

#### 3.12.3 `Combat/Query.luau`

```lua
--!strict
-- src/Shared/Combat/Query.luau — spatial queries against the Hurtbox group. SIM-SAFE. Spec §3.12.3.
local Query = {}

function Query.overlapParams(selfHurtbox: BasePart): OverlapParams
	local p = OverlapParams.new()
	p.CollisionGroup = "HitQuery" -- collides only with Hurtbox (§3.12.1)
	p.ExcludeInstances = { selfHurtbox }
	p.MaxParts = 0
	return p
end

function Query.box(cf: CFrame, size: Vector3, p: OverlapParams): { BasePart }
	return workspace:GetPartBoundsInBox(cf, size, p)
end

function Query.sphere(pos: Vector3, radius: number, p: OverlapParams): { BasePart }
	return workspace:GetPartBoundsInRadius(pos, radius, p)
end

-- Swept box: casts skip parts that start inside the shape and return only the first hit, so:
-- (1) overlap at the start pose, (2) Blockcast loop with growing ExcludeInstances. Max iterations from config.
function Query.sweep(prevCf: CFrame, cf: CFrame, size: Vector3, selfHurtbox: BasePart, maxIter: number): { BasePart }
	local hits = Query.box(prevCf, size, Query.overlapParams(selfHurtbox))
	local dir = cf.Position - prevCf.Position
	if dir.Magnitude < 0.01 then
		return hits
	end
	local exclude: { Instance } = { selfHurtbox }
	for _, h in hits do
		table.insert(exclude, h)
	end
	for _ = 1, maxIter do
		local rp = RaycastParams.new()
		rp.CollisionGroup = "HitQuery"
		rp.ExcludeInstances = exclude
		local r = workspace:Blockcast(prevCf, size, dir, rp)
		if r == nil then
			break
		end
		table.insert(hits, r.Instance)
		table.insert(exclude, r.Instance)
	end
	return hits
end

return Query
```

`Sphere` hitboxes use `size.X / 2` as the radius. `Cast` hitboxes use `Query.sweep` from the previous frame's hitbox CFrame (the request's `prevCf`). Projectiles use the same loop with `workspace:Spherecast(position, radius, direction, params)` (Thread-Safe, unlike `Shapecast`). `Blockcast` limits: size ≤ 512, distance ≤ 1024 [Docs]. `CastMaxIterations` = 8 (td).

#### 3.12.4 `Combat/Rewind.luau` (lag compensation; GDD §18.2)

- **Ring:** per character, `Damage.Rewind.BufferFrames` = 60 poses (1 s at 60 Hz, GDD), each `{f, cf, size}`, recorded in phase 0 on the server (server memory, never replicated). `Rewind.at(ring, f)` returns the pose for frame `f`, or the nearest older one, or the oldest available.
- **Rewind amount** for attacker `a`: `k = clamp(round((rttEma + InterpDelaySec) · 60), 0, round(MaxSec · 60))`, with `MaxSec = 0.2` (GDD: capped at 200 ms, so k ≤ 12) and `InterpDelaySec = 0.05` (td; calibrated by spike SA-07b). Bots and carriers without a player owner: `k = 0`.
- **Latency source:** `Player:GetNetworkPing()` returns the round-trip latency in seconds and is Thread-Safe [Docs `Player.yaml`]. CombatService samples it every `PingSampleSec` (0.5 s) on `Heartbeat` (outside the sim) into `srv.rttEma` (EMA, α = `PingEmaAlpha` 0.3) and recomputes `srv.rewindFrames`. It is engine-measured, not client-reported, unlike `central`'s client-reported measure [reference-impl §2.2]. The RTT approximates the GDD's "client-server step delta"; the interpolation term covers render buffering.
- **Clamp and anti-cheat logging:** when `rttEma + InterpDelaySec > MaxSec`, `k = 12` and the flush logs `[ALG][Combat] rewind clamp user=<id> estMs=<n>` (throttled to once per 30 s per player). When a ping sample deviates from the EMA by more than `SpikeSec` (0.3 s) more than `SpikesPerMinSignal` (5) times in a rolling minute → `ModerationService.signal(player, "sim_input", {reason = "ping_spike"})`. The rewind never exceeds 0.2 s: a high-ping attacker gets no extra reach (GDD §18.2).
- **Rewound query:** victims' hurtbox parts are moved to `Rewind.at(ring, f − k)`, the query runs, and the present pose is restored, all inside the same server step (CFrame/Size are Sim-Access; the parts are anchored and not welded, so nothing else moves):

```lua
-- Rewind.withPoses (server, inside the sim). Restores present poses even if fn errors.
function Rewind.withPoses(victims: { Adapter.RosterEntry }, f: number, k: number, fn: () -> ())
	if k <= 0 then
		fn()
		return
	end
	for _, v in victims do
		local p = Rewind.at((v.srv :: Adapter.ServerData).ring, f - k)
		v.hurtbox.CFrame = p.cf
		v.hurtbox.Size = p.size
	end
	local ok, err = pcall(fn) -- fn never yields
	for _, v in victims do
		local p = Rewind.at((v.srv :: Adapter.ServerData).ring, f)
		v.hurtbox.CFrame = p.cf
		v.hurtbox.Size = p.size
	end
	if not ok then
		error(err, 0)
	end
end
```

Damage, stun and knockback still apply in present time (GDD §18.2). Fallback if SA-07 shows that queries do not see the moved CFrames within the same step: `Query.obbOverlap(boxCf, size, pose)` (pure OBB separating-axis test in Lua) against ring poses, with no part moves.

#### 3.12.5 Server query pipeline (phase 2)

1. Group pending hit requests by attacker `k` (`srv.rewindFrames`; ≤ 13 groups, usually 1–3).
2. For each group: victims = ready entries (not the attacker, not `KO`) whose present root is within `Damage.QueryBroadphaseStuds` (30, td) of any attacker in the group. Call `Rewind.withPoses(victims, f, k, run)`.
3. `run` executes each request's query (§3.12.3), maps parts to entries (`Adapter.entryOfHurtbox`), skips the attacker itself and victims already in `attacker.srv.hitReg[req.key]`, adds them to the registry, and appends `{att, vic, req}`. At most `req.maxTargets` (default `Damage.QueryMaxTargets` = 8) per request per frame.
4. Registry keys: chassis M1/Ender use `"m1"`. Behaviors choose keys (e.g. `"<moveId>#<hitIndex>"` for multi-hit moves). Carriers use their instance name. The registry is cleared when the owner enters a new state (`StepOut.entered`). Carriers' registries live with the carrier.

#### 3.12.6 `Damage.resolve` and consequence application (phase 3, server)

```lua
export type Outcome = "ignore" | "whiff" | "otg_denied" | "hit" | "block" | "perfect" | "guardbreak" | "armor" | "interrupt"
export type AttackerView = { rid: number, s: Fsm.CharState, rootCf: CFrame, mods: Fsm.ChassisMods }
export type VictimView = {
	rid: number, s: Fsm.CharState, rootCf: CFrame, grounded: boolean, maxHp: number,
	mods: Fsm.ChassisMods, status: Moves.StatusMods, castE: number?, -- castE: RealmCast elapsed if casting
}
export type HitCtx = { f: number, srcPos: Vector3, hdir: Vector3, ally: boolean, pileOn: boolean, projectile: boolean }
export type HitResult = {
	outcome: Outcome, damage: number,
	vicPatch: { [string]: any },   -- S-class writes on the victim
	attPatch: { [string]: any },   -- attacker writes; fields in Damage.BQ_FIELDS are the client-predictable subset
	meterAtt: number, meterVic: number, events: { Fsm.SimEvent },
}
```

Resolution order (first match wins):

| # | Condition | Outcome | Effects |
|---|---|---|---|
| 1 | victim `KO`, or `ally` (party, no friendly fire, GDD §6.12) | `ignore` | none |
| 2 | `Fsm.isInvulnerable(victim)` (i-frames, wake-up, spawn protection) | `whiff` | event (render "dodge") |
| 3 | `hit.groundOnly` and victim airborne (ground grabs lose to jumps, GDD §6.4) | `whiff` | — |
| 4 | victim downed (`Ragdoll`, `Sub > 0`) and `CFlags` has `OTG` | `otg_denied` | event |
| 5 | victim in `Block`/`BlockStun`, `Defense.isFront`, and (`not hit.unblockable` or (`hit.kind == "grab"` and `hit.parryable`)) and `Defense.isPerfect` | `perfect` | §3.6.3 |
| 6 | victim in `Block`/`BlockStun`, front, `not hit.unblockable` | `block` or `guardbreak` | §3.6.4–§3.6.5 |
| 7 | victim `RealmCast` with `Realm.interrupts(castE, hit, guardBreak)` | `interrupt` | cast cancelled + continue as `hit` |
| 8 | victim `RealmCast` (not interruptible), or in a Move with `superArmor` during `params.superArmorFrames` (default: startup + active), or `projectileArmor` vs `projectile` during startup | `armor` | damage + meter only; no stun, no knockback |
| 9 | otherwise | `hit` | below |

`hit` effects. Victim (S): `damage = Damage.scaled(...)` (§3.8.4) minus status shield; `HP −= damage` (floor 0); `CombatF = f`; `CHits += 1` (not for ticks); `StunF0` set if 0; OTG flag if downed; the state from the hit's knockback (§3.8.5) or `Hitstun` with `hit.hitstun` (decayed); `KbV`; `HsF = f + hitstop`; `StF = f + hitstop + 1`; `BkLockF` (§3.9.1); `BkProg` refund; debuffs from `onContact`; grabs set `Grabbed` via the Grab behavior. Attacker: `Cnt = 1`; `HsF`/`StF += hitstop`; `CombatF = f`; Uppercut follow (`HangF`, `Flg FOLLOW|ENDLAG`, Ender ends after hitstop). Meta: meter dealt/taken, ledger, `lastHit`, pile-on list, life stats, `damage` event.

Application: contacts are processed grouped by victim rid ascending, and within a victim by attacker rid then key. Each resolve sees the victim's state as already modified by earlier contacts in this step (sequential), and the attacker's state after its own phase-1 step. Two characters that hit each other in the same frame both get hit (trade), because phase 1 already advanced both before any consequence applies. Patches go through `Adapter.writeDiff`.

`Damage.BQ_FIELDS = {"Cnt", "HsF", "StF", "CombatF", "HangF", "Flg", "StDur"}`. `Damage.predictAttacker(att, vic, hit, hc, cfg): {[string]: any}` returns exactly the attacker patch that `resolve` would produce for outcomes `hit`/`block`, from the victim's replicated state. It cannot predict `perfect`; the server's Stagger arrives by rollback.

#### 3.12.7 Client predicted feedback (undoable)

On the client, phase 2 runs `Query` *live* (no rewind) for the local entry's requests, against hurtboxes as replicated. Remote characters are rendered "slightly in the past" [Docs `techniques.md`], which is exactly what the server's rewind reconstructs. For the first contact of the current action (`s.Cnt == 0`), the client writes `Damage.predictAttacker(...)` (BQ fields). Render plays the spark/SFX/hitstop on the `Cnt` transition (§3.11.8). If the server disagrees, the rollback restores its values and the render undo removes the spark. The client never predicts victim HP, victim state, meter or carrier hits.

**Verify (§3.12):** Lune `damage.*`, `rewind.ring_wraps`, `rewind.frames_for_rtt` (rtt 0.30 s → k = 12; rtt 0.05 → k = 6), `query.sweep_excludes` (shim). Scenarios: `combat.rewind_hit_150ms` (Studio Network Simulator 150 ms ±30 ms jitter, 1% loss; a bot strafes at walk speed across the player's front; a player M1 timed visually on the client hits on ≥ 9 of 10 attempts, versus ≤ 4 of 10 with `MaxSec = 0` in a control run); `combat.rewind_cap` (rtt forced to 0.5 s via the test seam `CombatService._setRtt` → `[ALG][Combat] rewind clamp` logged, k = 12); `combat.trade_same_frame` (two bots with mirrored scripted M1s → both `Hitstun`).

---

### 3.13 Damage, KO, assists, respawn, spawn protection, streaks

#### 3.13.1 KO

Phase 4 (server): any entry with `HP ≤ 0` and `St ≠ KO` → `St = KO`, `StF = f`, movers off, grab detached; event `ko`. `KO` is invulnerable (§3.2.1) and accepts no input. The Humanoid never dies through `Humanoid.Health` (not Sim-Access); CombatService still connects `Humanoid.Died` (outside the sim) as a safety net (fall below `FallenPartsDestroyHeight`, broken rig): it logs `[ALG][Combat] WARN humanoid_died user=<id>` and runs the KO flow with no killer. Kill planes (Part 6 zones) set `HP = 0` through `CombatService.killAsync(entry, "environment")`, which queues a patch for the next step.

#### 3.13.2 KO credit and assists (GDD §6.11)

- Killer = `srv.lastHit.rid` if `f − lastHit.f ≤ toFrames(Streak.KoCreditWindowSec)` (10 s, td), else none (environmental KO).
- Assists = distinct attackers other than the killer whose summed damage in the victim's ledger within the last `toFrames(Streak.AssistWindowSec)` (600 frames) is ≥ `Streak.AssistMinDamage` (15) (GDD).
- Meter (server memory): killer +15, each assist +7 (GDD §6.9). Killer awakened → `AwkEndF` extension (§3.9.2).

#### 3.13.3 Streaks and bounties (GDD §6.11)

- `streak` per Player (server memory, Player attribute `Streak`): +1 per KO of a **player** (bots count only when `Streak.CountBots = true`, td false; training dummies never). It resets to 0 on the player's own KO.
- Reaching 3, 5 or 10 (`Streak.Tiers`) fires `CombatService.streak` with `kind = "tier"`. When a player whose streak is ≥ 3 is KO'd, it fires `kind = "ended"` with `bounty` = the bounty for the highest tier reached (25 / 60 / 150 Treasure, `Streak.Bounty`, GDD) and `endedBy` = the killer (nil if environmental).
- Part 5 (ProgressionService) subscribes: it pays the bounty (Economy reason `streak_bounty`) and broadcasts `Notify` (`ui.notify.streak_<tier>`, scope `server`). Part 6 renders the aura from the `Streak` attribute (≥ 3).

#### 3.13.4 Respawn flow (GDD §6.2: 4 s, random safe spawn)

```lua
-- CombatService.spawnAsync(player) — outside the sim; yields. Spec §3.13.4.
-- 1. if player:GetAttribute("DataLoaded") ~= true then return end   -- not PlayerService: init order §2.3.3
-- 2. local styleId = StyleService.resolveForSpawn(player)              -- writes EquippedStyle/Flair/Variant
-- 3. local v = Runtime.version(); ConfigReplicator.pin(v); player:SetAttribute("CombatCfg", v)
-- 4. local spawnCf = pickSpawn(player:GetAttribute("MapId"))          -- random Part with SpawnFor == MapId, no
--                                                                      -- enemy within Respawn.SafeSpawnClearStuds (20, td);
--                                                                      -- up to 8 tries, else any random spawn
-- 5. pending[player] = { styleId = styleId, cfgVersion = v, spawnCf = spawnCf + Vector3.new(0, 3, 0) }
-- 6. player:LoadCharacterAsync()                                       -- never the deprecated LoadCharacter
-- 7. CharacterAdded handler: Hurtbox.prepare -> Adapter.track(model, pending[player]);
--    the first server step initialises attributes and PivotTo(spawnCf) (§3.11.3 phase 0b)
-- 8. log "[ALG][Combat] respawn user=<id> map=<mapId> cfg=<v>"; fire CombatService.spawned(player, model)
```

- On KO the flush schedules `task.delay(Respawn.DelaySec, CombatService.spawnAsync, player)` (4 s, GDD). The body stays until `LoadCharacterAsync` replaces it; render hides it after `Respawn.BodyHideSec` (1.2 s, the Finale length, GDD §6.11). No `Destroy` is needed.
- `PlayerService.travelAsync` (Part 2) sets `MapId`, then calls `CombatService.spawnAsync`.
- Combat config: the new life uses `CombatCfg = v` for its whole duration (Part 2 §2.6.6). `CharacterRemoving` unpins the old version.

#### 3.13.5 Spawn protection (GDD §6.2, §15.5)

`ProtF = f + toFrames(Respawn.ProtectionSec)` (480) at init (S). While `f < ProtF` every hit is a `whiff` ("no damage for 8 s"). Entering any `ATTACK` state, or a Dash M1, sets `ProtF = 0` (B; "broken by attacking"). Render shows a shimmer while protected (Part 6).

#### 3.13.6 Rookie hooks

Rookie rules are economic (GDD §15.5: no Treasure for a veteran killing the same Rookie more than twice in 10 minutes). Part 3 provides `KoEvent` with Player references; Part 5 reads the victim's `Rookie` Player attribute and applies `Economy.RookieKill`. Spawn protection applies to every player, so there is no Rookie-specific combat rule.

#### 3.13.7 Signals, payloads and remotes

`CombatServiceApi` (Part 2 §2.3.5) is implemented exactly, plus these members:

```lua
export type StreakEvent = {
	player: Player, streak: number, kind: "tier" | "ended", tier: number, -- tier: 3 | 5 | 10
	bounty: number, endedBy: Player?,                                    -- bounty only for kind == "ended"
}
export type CombatServiceExtra = {
	streak: Signal.Signal<StreakEvent>,
	styleMoment: Signal.Signal<Player, string>,          -- Part 4 kits report Style Moments (GDD §7); +10 meter applied here
	killAsync: (target: Player | Model, cause: "environment" | "dev") -> (),
	isInCombat: (player: Player) -> boolean,             -- f - CombatF < CombatTagSec
	-- Studio-only test seams (no-ops outside Studio):
	_queuePatch: (target: Player | Model, patch: { [string]: any }) -> (), -- applied at the next server step
	_setMeter: (player: Player, value: number) -> (),
	_setRtt: (player: Player, seconds: number?) -> (),   -- overrides the ping sample (nil = live)
}
```

Payloads (Part 2 `Types`): `DamageEvent` per damaging contact (`amount` after scaling, `moveId`, `styleId` = the attacker's, `t = workspace:GetServerTimeNow()` taken in the flush). `KoEvent` (`streak` = the killer's streak after this KO, 0 if none). `LifeSummary`: CombatService fills the combat fields (`damageDealt`, `damageTaken`, `kos`, `assists`, `lifeSec`, `awakened`, `realmCast`, `breakawaysUsed`, `perfectBlocks`, `killerStyle`). Then it fires `lifeEnded(player, summary)`; handlers MUST NOT yield and MAY fill `treasureEarned`, `xpEarned`, `masteryXpEarned` (Part 5). Then it sends `Life.Summary` to the owner.

New remote `Combat.Feed` (S2C, event, `fireAll`, ≤ 1 per KO; Part 6 kill feed, GDD §15.3):

```lua
export type CombatFeed = {
	killer: string, victim: string,       -- UserId as a string, bot id "b<n>", or "" (none)
	killerStyle: string, victimStyle: string, assists: number, streak: number,
}
```

`canTravel(player)` = entry exists, `St ∉ STUN ∪ {KO}`, and `f − CombatF ≥ toFrames(Chassis.CombatTagSec)` (8 s, td).

**Verify (§3.13):** scenarios `combat.ko_respawn` (`Dev.Command kill` → `St = KO`; a `ko` log line; after 4 ± 0.1 s a new character, `ProtF − StF = 480`; `Life.Summary` received by the client); `combat.spawn_protection` (dummy hits during protection → `whiff`, HP unchanged; the player presses M1 → `ProtF = 0`; the next dummy hit lands); `combat.assist_credit` (bot A deals 16, bot B kills → KoEvent `assists = [A]`, meter +7 for A); `combat.streak_bounty` (scripted 3 KOs by player P → `streak` tier 3 event; P KO'd → `ended` with bounty 25); `combat.feed_fired` (client receives `Combat.Feed` once per KO).

---

### 3.14 Classic fallback adapter (`Config.NetModel = "Classic"`)

Activated only by a human decision at the M0 gate (H-10) if spikes SA-01, SA-03, SA-08 or SA-13 fail (§3.16.4). Core, MoveDefs, attributes and render are unchanged. Only the adapter changes:

| Aspect | ServerAuthority (default) | Classic |
|---|---|---|
| Place | `AuthorityMode = Server` (H-04) | human sets `AuthorityMode = Automatic` |
| Loop | `BindToSimulation` Hz60 on both peers | Server: fixed-step accumulator on `RunService.Heartbeat` (step 1/60, ≤ 4 catch-up steps, frame counter in server memory). Client: the same accumulator drives a **local shadow** `CharState` (Lua table) for the local character only. |
| Input | `InputAction:GetState()` in the sim | If spike SA-16 shows the server can read IAS state in Automatic mode (with `PlayerScriptsUseInputActionSystem` enabled), unchanged. Otherwise the new C2S remote `Combat.Input` (UnreliableRemoteEvent; 30 Hz; payload `{seq, f, frames = {{bits, aim (3 × int16 quantized), mv (2 × int8)}} × 4}` redundancy, ≤ 120 B). Registered only when `NetModel == "Classic"`. |
| State | HRP attributes written by both peers | HRP attributes written by the server only (normal replication). The client shadow FSM gives instant startup animation/VFX and is re-seeded from server attributes whenever they change. |
| Movement | predicted, server-owned characters | client network ownership (engine default); the client applies dash movers locally; the server checks displacement per dash (≤ `Studs × 1.25`) and cooldowns, and logs `sim_reject` on violations |
| Hit detection | server rewound queries | the client runs `Query` for its own attacks and sends `Combat.HitClaim {seq, key, victims: {rid}, attackerPos}` (C2S event, rate 30/10) |
| Validation (Laser Tag 1A structure [reference-impl §1.3]) | — | (1) schema; (2) server FSM: attacker in the claimed action within active frames ± `Classic.ActiveSlackFrames` (6, td); (3) claimed origin within `Classic.OriginToleranceStuds` (6, td) of the rewound server root; (4) each victim's rewound hurtbox overlaps the hitbox inflated by `Classic.RangeToleranceStuds` (2, td); (5) victim in the hitbox's forward half-space (dot ≥ −0.2); (6) `workspace:Raycast` chest-to-chest in `WorldQuery` unobstructed; (7) one claim per key per victim. Failure → claim ignored + `ModerationService.signal(player, "sim_reject", {reason})`. |
| Consequences | server, inside the sim step | server, in the Heartbeat step (same `Damage.resolve`) |
| Knockback | prescribed movers on both peers | server `victimRoot:SetNetworkOwner(nil)` for the knockback/ragdoll window, movers applied server-side, then `SetNetworkOwnershipAuto()` |

Config keys for Classic live in `Combat.Chassis.Classic` (§3.17). Lune tests for the validators: `classic.origin_tolerance`, `classic.los_blocked`, `classic.active_window_slack`.

---

### 3.15 Bots and Training dummies

#### 3.15.1 Intent-attribute protocol

- There is exactly one simulation binding (Part 2 §2.3.2: "the single BindToSimulation loop"). BotService registers an intent provider in `BotService.init()`: `Adapter.setIntentProvider(function(f: number, bots: { Adapter.RosterEntry }) ... end)`. The server's `Adapter.step` calls it in phase 0c (after init, before phase 1), inside the same binding. The provider writes each bot HRP's intent attributes `BIn` (bitmask, `Fsm.IN` layout), `BAim` (unit Vector3), `BMv` (Vector3, |v| ≤ 1) with the Q-rule. These are written inside the sim callback, so §3.1.4 rule 2 holds. The provider must not yield, must use only Simulation-Access APIs, and is not called on clients.
- The main loop reads them with `InputMap.readBot(root, prevBits)` and drives `humanoid:Move(BMv)`, the jump, and every combat input through the same `Fsm.step` as players. One code path.
- Bot brains are server Lua (legal on the server, §3.1.8), deterministic with `Rng.hash01(f, rid, salt)`. They may read any sim attribute.
- Spawning: `BotService.spawnAsync(spec: {kind: "dummy" | "brawl", styleId: StyleId, mapId: MapId, slot: number?}): Model`. It clones `ServerStorage.Templates.DummyRig` or `BotRig`, sets the Model attributes `IsBot`, `BotId` (`"b<n>"`), `EquippedStyle`, `Flair`, `Level`, `CombatCfg` **before** parenting (Part 2 §2.5.4), calls `Hurtbox.prepare(model)`, parents to `Workspace.Bots`, then `Adapter.track(model, {botId, styleId, cfgVersion, spawnCf})`. rid = −1000000 − n.
- Client side: bots are tracked with `simulate = false`. Spike SA-12 checks whether clients auto-predict nearby NPC humanoids. If they do and mispredictions are high, the client calls `RunService:SetPredictionMode(botModel, Enum.PredictionMode.Off)` from `Workspace.Bots.ChildAdded`.

#### 3.15.2 Dummy modes (`Training.Dummy` behavior enum)

`TrainingDummyReq.behavior ∈ {"idle", "block", "perfect_block", "m1_spam", "dash_spam"}`. The request applies to the dummy whose `DummyPoints` slot is nearest the requesting player (≤ 40 studs, td) and is stored in BotService memory. Log `[ALG][Bot] dummy mode=<m> slot=<n>`.

| Mode | Intents per step |
|---|---|
| `idle` | none; faces its spawn yaw |
| `block` | `Block` held; `BAim` toward the nearest player |
| `perfect_block` | Block released by default. When the nearest attacker within 12 studs is in an `ATTACK` state whose first active frame (from its `St/Sub/StF` and frame data) is exactly `BotService.PbLeadFrames` (3, td) frames away, press `Block` and hold 20 frames |
| `m1_spam` | faces the nearest player within 15 studs; `M1` pressed on every other frame (one rising edge per 2 frames) |
| `dash_spam` | faces the nearest player; when any of the Side/Back cooldowns is ready, dash in a random allowed direction (seeded) |
| `brawl` (load test only; not in the remote enum) | approach the nearest target, M1 chain, block on an incoming startup with probability 0.3, dash when pressured, use moves off cooldown |

Dummy lifecycle (`Respawn.Dummy`): KO plays the poof and the dummy respawns at its slot after `RespawnSec` (2, td). Phase 4 (server) restores `HP = maxHp` for entries tracked with `opts.dummy = true` once `f − CombatF ≥ toFrames(HpResetSec)` (2 s, td), for practice.

Load test: `Dev.Command spawnBots {count, mapId}` (≤ 20) and `clearBots` (new DevCmd entries, §3.18) spawn `brawl` bots for the GDD §18.2 gate (20 bots + 4 humans).

**Verify (§3.15):** scenario `bot.dummy_modes` (each mode for 10 s against a scripted player: `block` → every front M1 is `block`; `perfect_block` → ≥ 80% of M1s are `perfect`; `m1_spam` → hits land; `dash_spam` → ≥ 3 dashes); scenario `bot.intent_path` (a bot with `BIn = Dash` and `BMv = right` performs a Side dash, identical in outcome to a player).

---

### 3.16 Determinism and rollback checklist; Server Authority spikes

#### 3.16.1 Checklist (every P2/P3 PR; Part 7 automates items marked L)

1. (L) Every `BindToSimulation(`/`BindToAnimation(` call passes `Enum.StepFrequency.Hz60`, and `BindToSimulation(` appears exactly once in `src/` (in `Simulation.luau`).
2. (L) No banned token from §3.1.5 inside Core/Sim/Combat/Behaviors (allow-list: the single `IsServer()` at load in `Simulation.luau`, `Adapter.spawn`'s `Clone`).
3. (L) Every HRP attribute write goes through `Adapter.writeDiff`/`Adapter.patch` with `Attr.N` names (no string literals in `SetAttribute` calls in Shared/).
4. (L) No generalized iteration over dictionaries in Core without a `-- det:sorted` marker comment on a preceding sorted-keys line.
5. (L) No `== f`/`f ==` comparisons in Core (entry effects run on transition).
6. The client adapter writes only the local entry (Studio assert, §3.11.3).
7. Server-only Lua state never feeds B-class attributes (review).
8. Render never uses `GetAttributeChangedSignal` for one-shots (L in `RenderController.luau`).
9. Carriers are created only via `Adapter.spawn` and destroyed only in the flush.
10. The sim reads config only from the roster's pinned table; never `Runtime.get()` inside the sim.
11. Golden traces (`fsm.golden_*`) pass and are regenerated with `export_framedata --golden` on any tuning change.
12. Values written to attributes obey the Q-rule (Lune `fsm.q_rule` scans `StepOut.s` over a 2,000-frame random-input fuzz: every number is an integer or a multiple of 0.1/0.01 as specified).
13. The M0 gate run (Part 7): prediction success ≥ 97% in the SA visualizer, no visible rubber-banding on dashes, Perfect Block success within 5 points of the 30 ms baseline (GDD §18.2).

#### 3.16.2 Misprediction budget (expected sources)

| Source | Frequency | Accepted? |
|---|---|---|
| Victim state on every hit (S) | per hit | yes (inherent) |
| Attacker BQ contact disagreement | rare (client view ≠ rewound view) | yes |
| `AwkRdy` set | once per meter fill | yes |
| Realm membership / boundary / queue | rare | yes |
| Remote-character input not forwarded | none (others render in the past, not predicted) | — |
| Body contact | none (`Characters` group non-collidable) | — |

#### 3.16.3 Spike procedure

Each spike is a task in the phase listed in §3.20 (most are P0). It has a scenario module `src/DevTools/Tests/Spike_SAxx.luau` that prints `[ALG][TEST] spike.SA-xx PASS|FAIL <detail>`, and its result is recorded in `docs/agent/PROGRESS.md` → *Spike results* (Part 1 §1.2.8). Spikes need H-04 (AuthorityMode) done; SA-05 and SA-11 also need published placeholder animations (H-08).

#### 3.16.4 Spike catalog

| ID | Question | Method | Pass | Fallback if fail |
|---|---|---|---|---|
| SA-01 | Does `BindToSimulation(fn, Hz60)` give `dt = 1/60`, and does `Frames.fromTime(time())` advance by exactly 1 per call on both peers, including during resim? | Probe binding records `f` and `dt` for 600 calls on Server and Client (`execute_luau` reads the probe buffer) | every Δf = 1; `\|dt − 1/60\| < 1e-4`; client `f` at an input-tagged step equals the server's | Robust anyway (no `f == X` logic). If Δf ≠ 1 regularly: escalate; Classic at the M0 gate |
| SA-02 | Attribute replication limits | Set 52 SPEC attributes + 12 junk ones on a test HRP (64) and a 65th; one 51-char name; one 51-char string | the first 64 replicate; the 65th and the over-long ones do not (matches docs); all 52 SPECs replicate | Trim SPECs |
| SA-03 | Do IAS inputs reach the server sim under SA, including MCP virtual input (P1 S9) and `InputBinding:Fire` on Scriptable bindings? | MCP `user_mouse_input` click; client `InputController.fire("Aim", v)` | server `M1:GetState()` true within 10 frames; `Aim` equal within 0.001 | MCP fails → tests use Scriptable bindings; Scriptable fails → Classic |
| SA-04 | Can `Players:GetPlayers()` / `player.Character` be read inside `BindToSimulation`? (YAML says no; official samples do it) | `pcall` both inside a probe binding | record the result only | None needed: the roster cache is mandatory either way |
| SA-05 | Per-Animator playing-track limit under SA; `BindToAnimation` playback mirrored on both peers | Play 10 published tracks on one Animator inside `BindToAnimation`; count `IsPlaying` | record N; our design needs ≤ 4 | Lower `Render.MaxCombatTracks`; merge reaction anims |
| SA-06 | Is the CCL enabled on the dev place (Avatar Settings), and does the default Humanoid controller work under SA? | Inspect a spawned character for `ControllerManager`; `require("@rbx/AvatarAbilities")` in `pcall` | no `ControllerManager`; WalkSpeed changes from the sim are predicted | `ControllerLoco` path (§3.7.4) |
| SA-07 | (a) Do spatial queries see anchored hurtbox CFrames moved earlier in the same server step? (b) Does an anchored hurtbox inside a predicted character leave `GetPredictionStatus(root) = Predicted`? (b2) Calibrate `InterpDelaySec` | (a) Move a hurtbox, query, restore inside one step; (b) client reads the prediction status; (b2) at 150 ms netsim, measure the server-pose vs rendered-pose delay of a moving bot | (a) hit found; (b) Predicted; (b2) value recorded | (a) Lua OBB fallback (§3.12.4); (b) parent hurtboxes to `workspace.Sim` |
| SA-08 | `LinearVelocity` movers under SA at 150 ms ±30 ms, 1% loss | 50 dashes of each kind and 20 Enders on a dummy, with the SA visualizer open | distance error ≤ 10%; no visible rubber-banding (H-10 eyeball + correction magnitude < 1 stud) | Fallback A (`AssemblyLinearVelocity` writes) → B (force-limited) → Classic |
| SA-09 | Instance stitching of carriers cloned from per-character `ALG_Proto` when two characters spawn in the same frame; `SetPredictionMode(On)` on anchored carriers | Two bots + local player fire projectiles on the same frame (scripted) | exactly one instance per carrier on the client (no duplicates); the owner sees it immediately | Server-only carriers + client ghost (§3.11.5) |
| SA-10 | Collision groups: groups registered in Edit exist on clients; Humanoid floor detection ignores `Hurtbox` | Stand a character on another's head; check `workspace:GetRegisteredCollisionGroups()` on a client | groups present; no standing on hurtboxes | Register at runtime on both peers; shrink the hurtbox Y offset |
| SA-11 | Humanoid property writes from the sim (`WalkSpeed`, `SetStateEnabled(Jumping)`, `Jump`, `AutoRotate`) are predicted; `RootAttachment` exists; `Humanoid.MoveDirection` magnitude is analog for sticks; hurtbox `OffsetY` | Scripted state changes; compare client/server; inspect the rig | no persistent mispredictions; analog magnitude recorded | `ControllerLoco`; `RunAuto` = any stick push |
| SA-12 | Prediction status of remote players' and bots' roots on a client | `RunService:GetPredictionStatus(remoteRoot)` | not `Predicted` (or mispredictions negligible) | `SetPredictionMode(model, Off)` on the client |
| SA-13 | End-to-end prediction success under load | GDD §18.2 gate: 20 brawl bots + 4 clients, 150 ms ±30 ms, 1% loss, 5 minutes | ≥ 97% prediction success; server heartbeat ≥ 59 FPS | Reduce S-class writes; then Classic |
| SA-14 | Gamepad chord binding (`PrimaryModifier` with gamepad buttons) and multiple bindings on one Bool action | Press LS+RS via the Controller Emulator | `Awaken` true only with both | Bind `ButtonL3` alone |
| SA-15 | Bot Model attributes set before parenting cause no rollback (Part 2 §2.14 Q6) | Spawn 10 bots near a client; watch Misprediction counts | none attributable | Move bot meta to server memory |
| SA-16 | (Classic only) Can the server read IAS state with `AuthorityMode = Automatic` and `PlayerScriptsUseInputActionSystem` enabled? | Scratch place | server `GetState` mirrors the client | `Combat.Input` remote (§3.14) |

**Fallback decision at the M0 gate (H-10):** ServerAuthority stays if SA-01, SA-03, SA-08 and SA-13 pass (possibly using their in-SA fallbacks). Otherwise the human sets `Config.NetModel = "Classic"` (a code-level constant, not live-overridable, Part 2 §2.6.4) and `AuthorityMode = Automatic`, and the agent runs the Classic tasks.

---

### 3.17 `Defaults.Combat` (complete; replaces Part 2's placeholder `{}`s)

JSON-safe (Part 2 §2.6.1): numbers, strings, booleans, arrays, string-keyed tables; vectors are `{x, y, z}` arrays. Every value cites the GDD or is a tuning default.

```lua
	Combat = {
		Chassis = {
			MaxHp = 100,                              -- GDD §6.2
			HpBand = { 85, 115 },                     -- GDD §6.2 reserved live-balance band (validation only)
			ShieldCap = 20,                           -- GDD §6.2
			WalkSpeed = 18, RunSpeed = 26,            -- GDD §6.2
			JumpHeight = 7.5,                         -- GDD §6.2
			Hurtbox = { Diameter = 2.2, Height = 5.2, OffsetY = -0.3 }, -- GDD §6.2; OffsetY tuning default (SA-11)
			HurtboxScaleMin = 1,                      -- GDD §6.2 (awakenings scale up, never down)
			RegenPerSec = 2, RegenDelaySec = 8,       -- GDD §6.2
			RegenTickFrames = 60,                     -- tuning default
			Gravity = 196.2,                          -- tuning default; must equal workspace.Gravity
			StateWalkSpeed = { M1 = 6, Ender = 0, Move = 0, Stun = 0 }, -- tuning default
			CombatTagSec = 8,                         -- tuning default (canTravel; matches the regen delay)
			FriendlyFire = false, PartyMax = 2,       -- GDD §6.12
			MoverMaxForce = 50000,                    -- tuning default (fallback B only, × AssemblyMass)
			Input = {
				BufferFrames = 6,                     -- tuning default
				AimMinDeltaRad = 0.02, AimKeepAliveSec = 0.5, -- tuning default (Cakeman camera-action pattern)
				RunStickThreshold = 0.95, RunDoubleTapFrames = 15, -- tuning default (GDD §6.2 "full stick push", "double-tap W")
				AssistConeDeg = 12, AssistRangeStuds = 40, -- GDD §15.2 (12°); range tuning default
				CadenceSignalPerSec = 20, CadenceSignalSec = 3, -- tuning default (exploit signal only)
			},
			Sim = { Priority = 2000, MaxCarrierLifeFrames = 600 }, -- tuning default
			Classic = { ActiveSlackFrames = 6, OriginToleranceStuds = 6, RangeToleranceStuds = 2, DashSlack = 1.25 }, -- tuning default
		},
		M1 = {
			Hits = {                                  -- GDD §6.3 (M1-1..M1-3)
				{ Startup = 6, Active = 3, Recovery = 12, Damage = 3.5, Hitstun = 27, BlockAdv = -2, Guard = 12 },
				{ Startup = 6, Active = 3, Recovery = 12, Damage = 3.5, Hitstun = 27, BlockAdv = -2, Guard = 12 },
				{ Startup = 7, Active = 3, Recovery = 13, Damage = 3.5, Hitstun = 27, BlockAdv = -3, Guard = 12 },
			},
			Ender = { Startup = 10, Active = 4, Recovery = 22, Damage = 5.0, KbStuds = 38, RagdollFrames = 60, BlockAdv = -10, Guard = 20 }, -- GDD §6.3
			ChainWindowFrames = 36, EndLagFrames = 60, -- GDD §6.3
			ChainOnWhiff = true, BufferWholeRecovery = true, -- tuning default
			Uppercut = {
				LaunchStuds = 22, HangFrames = 48,    -- GDD §6.3
				RiseFrames = 18, AttackerRiseStuds = 20, HangFallSpeed = 3, HangDriftSpeed = 6, -- tuning default
				HitCancelsRecovery = true,            -- tuning default (3 air M1s + Downslam fit the hang)
			},
			Air = { MaxHits = 3, Damage = 2.5,        -- GDD §6.3
				Startup = 5, Active = 3, Recovery = 10, Hitstun = 27, BlockAdv = -2, Guard = 8 }, -- tuning default (hitstun = GDD M1 27)
			Downslam = { Damage = 5.0, KnockdownFrames = 54, -- GDD §6.3
				Startup = 8, Active = 4, Recovery = 22, BlockAdv = -10, Guard = 20, SpikeSpeed = 120, BounceStuds = 2 }, -- tuning default
			SideEnder = { AngleDeg = 45,              -- GDD §6.3
				SideInputMin = 0.5 },                 -- tuning default
			WallSplat = { WindowFrames = 18, StickFrames = 30, -- GDD §6.3 (0.3 s, 30f)
				AfterRagdollFrames = 30, ProbeStuds = 1.6 }, -- tuning default
			DashM1 = { Damage = 4, Hitstun = 30,      -- GDD §6.5
				Startup = 6, Active = 4, Recovery = 16, BlockAdv = -4, Guard = 12, LungeStuds = 6, WindowFrames = 8 }, -- tuning default
			Hitbox = { Size = { 5, 5, 6 }, Offset = { 0, 0, -3.5 } }, -- tuning default
			LungeStuds = 1.5, PushbackStuds = 1.5,    -- tuning default
		},
		Block = {
			ArcDeg = 180, M1Negate = 1.0, MoveChipDefault = 0.2, WalkSpeed = 8, -- GDD §6.4
			GuardMax = 100, GuardRegenDelayFrames = 60, GuardRegenPerSec = 25,  -- GDD §6.4
			MoveGuardMin = 20, MoveGuardMax = 60,     -- GDD §6.4 (MoveDef validation)
			GuardBreakFrames = 84, GuardBreakDamageMult = 1.10, GuardBreakNoBreakawayFrames = 30, -- GDD §6.4
			PerfectWindowFrames = 9, PbStaggerM1Frames = 36, PbStaggerMeleeFrames = 18, PbMeter = 6, -- GDD §6.4
			AntiMashLockFrames = 20,                  -- GDD §6.4
			TapMaxFrames = 15, StartupFrames = 0, PushbackStuds = 1.0, -- tuning default
			MoveBlockstunDefaultMult = 0.5, MoveBlockstunMin = 8, -- tuning default
			ChipCanKo = false, GuardRefillOnBreakEnd = true, -- tuning default
		},
		Dash = {
			Front = { Studs = 30, Frames = 18, IFrom = 0, ITo = 0, CooldownSec = 2.0 }, -- GDD §6.5
			Side = { Studs = 20, Frames = 15, IFrom = 1, ITo = 7, CooldownSec = 1.6, CurveDeg = 45 }, -- GDD §6.5; CurveDeg tuning default
			Back = { Studs = 16, Frames = 14, IFrom = 1, ITo = 5, CooldownSec = 2.0 }, -- GDD §6.5
			Air = { Studs = 14, Frames = 12, IFrom = 0, ITo = 0, OncePerAirtime = true }, -- GDD §6.5
			HopCancelFromHits = { 2, 3 },             -- GDD §6.5
			TechRoll = { WindowFrames = 12, IFrames = 18, -- GDD §6.5
				Studs = 6 },                          -- tuning default
			NoDirection = "Front", DirDeadzone = 0.2, DirFrontMaxDeg = 45, DirBackMinDeg = 135, -- tuning default
		},
		Stun = {
			M1Hitstun = 27,                           -- GDD §6.6
			DecayStartFrames = 150, DecayMult = 0.6,  -- GDD §6.6 (2.5 s)
			ShakeOffFrames = 210, ShakeOffIFrames = 18, ShakeOffStuds = 6, -- GDD §6.6 (3.5 s)
			OtgMaxPerCombo = 1,                       -- GDD §6.6
			HitPushFrames = 6, MoveKbFramesMax = 12,  -- tuning default
		},
		Ragdoll = {
			EnderFrames = 60, WakeUpIFrames = 30,     -- GDD §6.6
			TumbleAnimFrames = { 10, 14 },            -- GDD §13.3 (render)
			KbTravelFrames = 30, KbAngleDeg = 10, WakeUpIFramesEndOnAttack = true, -- tuning default
		},
		Breakaway = {
			Pips = 2, RefillSec = 25, DamagePerSecondRefund = 2, -- GDD §6.8 (-1 s per 2 damage)
			IFrames = 20, PushStuds = 8, LockoutFrames = 12, CinematicMaxSec = 2.0, -- GDD §6.8
			StartPips = 2, TickFrames = 30, PushRadius = 8, PushStaggerFrames = 12, -- tuning default
		},
		Meter = {
			Max = 100, DealtPerHp = 1.0, RealmDealtMult = 0.5, TakenPerHp = 0.5, -- GDD §6.9
			PerfectBlock = 6, StyleMoment = 10, Ko = 15, Assist = 7,             -- GDD §6.9
			GainWhileAwakened = false,                -- tuning default (meter restarts from 0, GDD §6.9)
		},
		Awakening = {
			ActivationFrames = 72, ShockwaveStuds = 10, CinematicSec = 1.2, Heal = 10, BreakawayPips = 1, -- GDD §6.9
			DurationSec = 60, KoExtendSec = 5, MaxSec = 90, PowerDownSec = 1.0, -- GDD §6.9
			AllowFromStun = false, ShockwaveStaggerFrames = 12, -- tuning default
		},
		Realm = {
			CastFrames = 90, InterruptibleFrames = 45, RadiusStuds = 40, DurationSec = 12, MaxActive = 2, -- GDD §6.10
			OnePerAwakening = true, RegenDisabled = true, OpenHitstopFrames = 12, -- GDD §6.10, §6.2, §6.13
			StubbornHeart = { HoldFrames = 60, PipCost = 1, WardSec = 5 }, -- GDD §6.10
			InterruptRefundCooldownSec = 10, BoundaryMarginStuds = 1, -- tuning default
		},
		Damage = {
			ComboTiers = { { MaxHit = 8, Mult = 1.0 }, { MaxHit = 14, Mult = 0.85 }, { MaxHit = 1000000, Mult = 0.70 } }, -- GDD §6.6
			ScaleFloor = 0.5,                         -- GDD §6.6
			Quantum = 0.1,                            -- tuning default (Q-rule)
			MaxBaseMove = 18, MaxQuirk = 6, MaxUltimate = 30, -- GDD §6.7
			TellMinFrames = 8, TellDamageAbove = 10,  -- GDD §6.14
			CooldownBands = { Basic = { 6, 10 }, Utility = { 10, 16 }, Signature = { 14, 22 }, Quirk = { 3, 8 }, QuirkRandomMax = 12 }, -- GDD §6.7
			Summon = { MaxBodies = 3, Hp = { 20, 60 }, LifeSec = { 6, 12 }, UltimateLifeMaxSec = 6, M1DamageMult = 2 }, -- GDD §6.7
			Targets = { NoMeter = { 28, 36 }, AwakenedWall = { 40, 50 }, Cap = 55 }, -- GDD §6.6
			Hitstop = { M1 = 3, Ender = 6, Signature = 9, PerfectBlock = 6, RealmOpen = 12, -- GDD §6.13 (signature 8–10)
				Move = 4, OnBlockSame = true },       -- tuning default
			Rewind = { MaxSec = 0.2, BufferFrames = 60, -- GDD §18.2
				InterpDelaySec = 0.05, PingSampleSec = 0.5, PingEmaAlpha = 0.3, SpikeSec = 0.3, SpikesPerMinSignal = 5 }, -- tuning default (SA-07b)
			QueryMaxTargets = 8, CastMaxIterations = 8, QueryBroadphaseStuds = 30, -- tuning default
		},
		Respawn = {
			DelaySec = 4, ProtectionSec = 8, ProtectionBrokenByAttack = true, -- GDD §6.2, §15.5
			BodyHideSec = 1.2,                        -- GDD §6.11 (Finale length)
			SafeSpawnClearStuds = 20, SafeSpawnTries = 8, -- tuning default
			Dummy = { RespawnSec = 2, HpResetSec = 2 }, -- tuning default
		},
		PileOn = { DistinctAttackers = 3, WindowSec = 4, DamageMult = 0.75, BreakawayRefillMult = 2, DurationSec = 4 }, -- GDD §6.6
		Streak = {
			Tiers = { 3, 5, 10 }, Bounty = { 25, 60, 150 }, -- GDD §6.11
			AssistWindowSec = 10, AssistMinDamage = 15, -- GDD §6.11
			KoCreditWindowSec = 10, CountBots = false, -- tuning default
		},
	},
```

`Schema.rules` added by Part 3 (validated at boot and on every ConfigService overlay, Part 2 §2.6.3): every `*Frames` int 0–3600; every `*Sec` num 0–600; `Damage.Rewind.MaxSec` num 0–0.2 (never above the GDD cap); `Chassis.MaxHp` int within `HpBand`; `Block.PerfectWindowFrames` int 1–20; `Damage.ScaleFloor` num 0.1–1; `Realm.MaxActive` int 0–4; `Chassis.Sim.Priority` int 1–10000. Cross-field checks: `M1.Air.MaxHits · (Startup+Active+Recovery) + Downslam.Startup ≤ Uppercut.RiseFrames + Uppercut.HangFrames` (Lune `m1.air_string_fits_hang`); `Chassis.Gravity` equals 196.2 unless the map builder also changes `workspace.Gravity` (asserted at boot).

**Verify:** Lune `schema.defaults_valid` and `schema.defaults_json_safe` (Part 2) pass with this section; Lune `config.combat_gdd_values` asserts the GDD-cited numbers above against a literal table in the test (guards against accidental edits).

---

### 3.18 New contracts introduced

| # | Contract | Consumers |
|---|---|---|
| 1 | **HRP attribute schema** (§3.3.1): 49 player attributes + 3 bot intents, names, types, defaults, writer classes `B`/`BQ`/`S`/`S·B`/`B·S`; `Attr.SPECS`, `Attr.N`, `Attr.MAX = 56`; `CFlags` and `Flg` bit tables | Parts 4, 6, 7 |
| 2 | **Player attributes** `Meter` (int 0–100) and `Streak` (int), written by CombatService's flush (Player attribute total becomes 19) | Parts 5, 6 |
| 3 | **State enum** `Fsm.S` ids 1–21, `Sub` codes per state, sets `NEUTRAL`/`STUN`/`ATTACK`; "Respawning" is not an HRP state | Parts 4, 6, 7 |
| 4 | **Input**: `ReplicatedStorage.Inputs.Combat` (Priority 2000, `Sink = false`); 15 actions (`M1, Move1–4, Quirk, Awaken, Block, Dash, Jump, Run, RunAuto, Launcher, AimLock, Aim`); binding names `KBM`/`Pad`/`Touch`/`Script`; touch button ids (`PunchBtn`, `Move1Btn`…`Move4Btn`, `QuirkBtn`, `AwakenBtn`, `BlockBtn`, `DashBtn`, `JumpBtn`); `Fsm.IN` bit layout; `InputMap.ACTIONS/attach/actionsOf/read/readBot`; context cloned directly under `Player` as `Combat` | Parts 1, 2, 6, 7 |
| 5 | **InputController API** `context/bindTouch/touchRoot/setCombatEnabled/fire`; `InputController.start()` starts the client simulation and animation bindings; Part 6's HUD builds buttons in `touchRoot()` and calls `bindTouch` | Part 6 |
| 6 | **Per-character instances** created by `Hurtbox.prepare`: `Hurtbox`, `ALG_Proto` (Model children); `ALG_Vel`, `ALG_Face`, `ALG_Hold`, `ALG_Grab` (HRP children); character parts in group `Characters`; Humanoid settings | Parts 4, 6 |
| 7 | **rid convention**: players = `UserId`; bots = −1000000 − n with `BotId = "b<n>"` | Parts 5, 6, 7 |
| 8 | **Collision groups** `Characters`, `Hurtbox`, `HitQuery`, `WorldQuery`, `CosmeticNoQuery` + matrix; registered by `BuildInputs.build()`; map geometry stays in `Default`; every client visual uses `CanQuery = false` + `CosmeticNoQuery` | Parts 6, 7 |
| 9 | **Core types** in `Core/Fsm.luau`: `CharState, InputFrame, StepCtx, StepOut, HitDef, HitboxReq, HitRequest, SpawnRequest, MotionCmd, FacingCmd, LocoCmd, SimEvent, ChassisMods, StyleRuntime`; `Damage.Outcome/HitResult/AttackerView/VictimView/HitCtx`, `Damage.BQ_FIELDS`, `Damage.predictAttacker` | Part 4 |
| 10 | **Behavior hook contract** `BehaviorModule {step, onContact?, onEnd?}` with `BehaviorCtx`/`BehaviorOut`; carrier stepping `Behaviors.Projectile.stepCarrier(view, f)`, `Behaviors.Zone.stepCarrier(view, f)`; carrier attributes `Own, Kind, MoveId, F0, Dead, Hits` + ≤ 10 params; names `C_<rid>_<f>_<n>` under `workspace.Sim` | Part 4 |
| 11 | **Status codec** in `Core/Moves.luau` (`Buffs`/`Debuffs` strings, ≤ 4 entries each) and the `Moves.STATUS` registry (Part 4 fills kinds and `mods`); `StatusMods` fields | Part 4 |
| 12 | **Recognized `MoveDef.params` keys**: `air`, `iFrames {from,to}`, `cancelInto`, `cancelWindow {a,b}`, `cancelChainIndex`, `blockstun`, `ragdollFrames`, `hangFrames`, `wallSplat`, `cinematicFrames`, `groundOnly`, `hitstop`, `superArmorFrames {from,to}`. **Recognized passive/awakening keys** (`StyleDef.passive.params`, `AwakeningDef.modifiers`): `breakawayRefillMult`, `enderDamageAdd`, `lowHpThreshold` + `lowHp` (sub-table of the same keys), `ragdollFramesAdd`, `m1RangeAdd`, `walkSpeedMult`, `airDash`; `AwakeningDef.hurtboxScale` | Part 4 |
| 13 | **Realm framework** in `Core/Realm.luau`: Part 3 lifecycle functions; Part 4 fills `Realm.rules[<moveId>]` (`RealmRules`); marker Model `Realm_<id>` in `workspace.Sim` with attributes `Center, Radius, OpenF, EndF, RuleId, Caster, Dead`; HRP `RlmId`/`RlmWardF` | Parts 4, 6 |
| 14 | **Animation keys** (§3.11.8) and the sim-mirrored animation driver in `BindToAnimation`; Part 6 publishes placeholder animations under these `Assets` keys; `Render.MaxCombatTracks = 2`, `Render.UndoWindowFrames = 15` | Parts 6, 4 |
| 15 | **RenderController contract**: transition table, undo rule, `RenderController.onRollback(f)` connected to `RunService.Rollback`; no `GetAttributeChangedSignal` for one-shots | Part 6 |
| 16 | **CombatService extras**: `streak: Signal<StreakEvent>`, `styleMoment: Signal<Player, string>`, `killAsync`, `isInCombat`, Studio seams `_queuePatch/_setMeter/_setRtt`; `StreakEvent` and `CombatFeed` types (Part 2 adds them to `Types.luau`); `lifeEnded` handlers must not yield and may fill `treasureEarned/xpEarned/masteryXpEarned` before `Life.Summary` is sent | Parts 2, 5, 6 |
| 17 | **Remotes**: `Combat.Feed` (S2C event, fireAll, payload `CombatFeed`). Classic-only (registered only when `NetModel == "Classic"`): `Combat.Input` (C2S unreliable, rate 40/30) and `Combat.HitClaim` (C2S event, rate 30/10) | Parts 2, 6 |
| 18 | **`TrainingDummyReq.behavior` enum**: `"idle" \| "block" \| "perfect_block" \| "m1_spam" \| "dash_spam"` | Parts 2, 6 |
| 19 | **DevCmd additions** (Part 2 §2.11.5 enum): `setMeter(value)`, `spawnBots(count, mapId)`, `clearBots()`, `combatPatch(json)` (→ `CombatService._queuePatch` on the caller's character) | Parts 2, 7 |
| 20 | **Adapter API**: `track` (opts incl. `dummy`), `untrack`, `list`, `entryOf*`, `setIntentProvider` (BotService), `startClient`, `step`, `animate`, `heal`; exactly one `BindToSimulation` in `src/` | Parts 4, 7 |
| 21 | **Moderation signal reasons**: `sim_input {reason = "cadence" \| "ping_spike"}`, `sim_reject {reason = <validator name>}` (Classic) | Part 2 |
| 22 | **Studio snippets** (Part 1 catalog): `tools/studio/combat_attrs.luau` (args `{target}`: player name or bot id → JSON of every HRP SPEC attribute + `St` name) and `tools/studio/combat_force.luau` (args `{target, patch}` → `CombatService._queuePatch`) | Parts 1, 7 |
| 23 | **Config**: complete `Defaults.Combat` (§3.17) incl. sub-tables `Chassis.Input`, `Chassis.Sim`, `Chassis.Classic`, `Damage.Hitstop`, `Damage.Rewind`, `Respawn.Dummy`; Part 3 `Schema.rules` | Parts 2, 4, 7 |
| 24 | **Test names**: the Lune specs in §3.4.4 (+ `rewind.*`, `query.sweep_excludes`, `classic.*`, `config.combat_gdd_values`); scenarios `sim.*`, `input.*`, `combat.*`, `bot.*`, `render.undo_spark`, `spike.SA-01…SA-16` | Part 7 |

### 3.19 Issues with spine

1. **Meter placement vs GDD §18.2.** The GDD lists `Meter` among simulation attributes. This part keeps the meter in server memory plus the Player attribute `Meter`, and exposes only `AwkRdy` to the sim, to avoid one attacker misprediction per hit (§3.3.5). This is consistent with §0.4, which places Meter on the Player or in server memory, but it differs from the GDD's wording. No GDD number changes.
2. **"Fixed capsule" hurtbox.** `Enum.PartType` has no capsule [Docs]. The hurtbox is a box with the capsule's extents (§3.12.2). Queries via `GetPartBoundsInBox` are bounding-box tests anyway.
3. **"Simulation input never uses remotes"** (§0.4) holds under ServerAuthority. The Classic fallback may need `Combat.Input` if the server cannot read IAS in Automatic mode (spike SA-16). Suggested spine wording: "under ServerAuthority".
4. **File layout has no home** for the status codec, the animation driver or the Realm framework split. To avoid adding files to the "exact" layout, they live in `Core/Moves.luau`, `Sim/Adapter.luau` and `Core/Realm.luau` (framework by Part 3, rule tables by Part 4; Part 2 lists `Core/Realm` as Part 4-owned).
5. **Deprecated list (§0.4) is incomplete.** Add `AlignOrientation.PrimaryAxisOnly`, `Enum.HumanoidStateType.Ragdoll` (both deprecated in the snapshot) and `Player:LoadCharacter` (also raised by Part 2).
6. **Attribute budget.** Part 0 says ≤ 64; Part 2 caps Part 3 at ≤ 56. This part uses 49/52 and follows the stricter cap.
7. **"Simulation state lives in attributes … only"** (§0.4) is slightly narrower than reality: facing, grab holds and hurtbox poses are predicted *constraint/part properties*, not attributes. They follow the same "write only inside the sim" rule.

### 3.20 Open questions / spikes

Spikes: SA-01 … SA-16 (§3.16.4). P0: SA-01, 02, 03, 04, 06, 10, 11, 12, 14, 15 (16 only if Classic is considered). P3: SA-05 (needs H-08 published animations), SA-07, SA-08, SA-09. P8: SA-13 (the GDD §18.2 network gate).

Design interpretations needing owner sign-off (feel test H-10):
1. **Uppercut follow is automatic** on hit (attacker rises 20 studs with the target). The GDD only says the attacker "gets 48f of reduced-gravity hang time".
2. **Uppercut skips its recovery on hit**, and its 60f end lag is deferred until the air string ends. Without this, 3 air M1s cannot fit the 48f hang.
3. **Stagger (Perfect Block victim, pushes) counts as stun**, so Breakaway is allowed after its lockout.
4. **Awakening cannot be activated while stunned** (Breakaway is the stun escape).
5. **Realm "trapping":** members are kept inside by moving them back across the boundary. A third cast is queued at cast start (not after the wind-up), so every opening keeps its full 90f telegraph.
6. **Chip cannot KO**; **character bodies do not collide** with each other; **zone ticks ignore block**; **bots don't count for streaks**.
7. Tuning defaults that need feel validation: M1 hitbox (5×5×6 at 3.5 studs), lunge/push 1.5 studs, air M1 and Dash M1 frame data, knockback travel 30f, Downslam spike/bounce, `InterpDelaySec` 0.05 (SA-07b), input buffer 6f, anti-mash tap 15f.
8. **CCL** is not adopted for M0/M1-G (§3.7.4). Revisit after M1-G if the beta graduates and touch slots become remappable.
9. **Shift-Lock under SA:** whether the default PlayerModule's mouse-lock rotation conflicts with `ALG_Face` (`AutoRotate = false` while `AimLock`). Part of SA-11; if it fights, CameraController disables the PlayerModule's character rotation and relies on `AimLock` facing.
10. **Remote-character interpolation** is engine-controlled; the rewind assumes `RTT + InterpDelaySec`. If SA-07b measures a large variance, evaluate `central`'s client-measured delay (clamped to 0.2 s) as an *additional* signal, never as the sole source.
