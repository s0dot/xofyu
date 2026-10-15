[← Build spec index](../STUDIO_BUILD_SPEC.md) · **Part 8 of 8** · sections §8.x · Part 0 contracts (§0.x) are in the index file

## 8. IP and naming layer

This part defines:
- the content of the three skin packs;
- the rules that keep IP strings out of code, instance names and public metadata;
- the publish gate;
- asset provenance and the recognizability review;
- the response playbook for enforcement events.

Ownership:
- Part 2 §2.9.2 owns the `Skin` API.
- Part 7 §7.5 owns the lint *mechanism* (`tools/lint_strings.luau`, `tools/check_manifest.luau`).
- This part owns the rules *data*, the extra rules L07–L22 and M09–M12, and the gate procedure.

Names, IDs and paths follow Part 0 verbatim.

---

### 8.1 Summary for the agent

#### 8.1.1 What the naming layer is for

*Algebraic!* is designed around *Adventure Time* (AT), and the team holds no AT license. The naming layer keeps everything IP-bearing in swappable data:
- every player-visible name and string;
- every likeness-bearing look (style palettes, Style Gear and summon models);
- every piece of public metadata.

One code base therefore ships in three IP postures selected by `Config.SkinPack`, and a public build can be checked mechanically before the human publishes it. Combat, Wish, progression and UI code never know which pack is active. They see IP-neutral IDs (§0.4) and Skin keys.

#### 8.1.2 The three packs

| Pack | What players see | Where it may run | Status |
|---|---|---|---|
| `Codename` | Style names are the GDD §7.2 epithets ("The Last Human"). Move names are GDD §7.3 with canonical proper nouns and show slang removed (§8.3.1). Places get codename-safe names ("The Candy Capital"). Graybox Style Gear reproduces each style's signature read (Part 4 §4.7.4). | Dev and private test places: always. Public places: only through the owner's informed-consent gate (§8.4.4). | **Dev default** (`Defaults.SkinPack = "Codename"`), per the owner's chosen approach |
| `Original` | IP-neutral original names written from the GDD archetypes, plus the Original look set (palettes, gear and summon recipes, §8.5.6) | Anywhere | The fallback for every key. **Recommended for every public build** |
| `Canon` | Canonical AT names and GDD §7.3 move names verbatim | Only with `Config.LicenseConfirmed = true` and a recorded license (§8.4.4) | **Locked** |

The owner's position is the Jujutsu Shenanigans model: change the names and avoid official art. Codename implements that model. It is not the lowest-risk version of it. Each of these would reduce detection or evidence further:
- an avatar-first public default (`CharacterLook = false`, `OwnLook = "Avatar"`, as in JJS, which puts movesets on the player's own avatar);
- dropping the "Algebraic!" callouts from Codename (`ui.hud.meter_name`, `ui.hud.awaken_ready`, `ui.hud.combo_big`, `ui.tip.awaken`, `ui.vo.awaken`);
- replacing the Q-P8-4 names.

These are owner options, recorded in PUBLISH_ACK at signing (§8.4.5, OPT-1…OPT-3). The research recommendation is unchanged:
- keep the build license-ready;
- use the Original pack for public builds unless the owner explicitly accepts the risk through the gate.

The agent implements both postures and never chooses between them.

#### 8.1.3 Risk in one paragraph

*This is not legal advice.* Counsel review is planned before M0 (GDD §21.2).

- **Renaming reduces two exposures but does not remove them.** It weakens trademark-confusion claims (in *Characters for Hire* the infringement claim failed while the dilution and copyright claims survived) and lowers keyword matching. Canon epithets, the catchphrase title and in-game callouts can still be registered as secondary keywords: the Codename pack still shows canon epithets and "Algebraic!" and ships IDs such as `CandyKingdom`, and a rights holder can register any of them [inference].
- **Renaming does not change copyright in recognizable characters, places and scenes.** In *Disney v. Characters for Hire* (S.D.N.Y. 2018) the trademark-infringement claim against renamed costumes failed, while the dilution and copyright claims survived. In *DC Comics v. Towle* (9th Cir. 2015) self-built replicas with no official art still infringed. Game mechanics are free (*Capcom v. Data East*, *DaVinci v. ZiKo*).
- **The anime battlegrounds games appear to survive mainly because of who owns their IP [inference].** Their owners are Japanese publishers with a history of tolerating fan works and no competing Roblox product.
- **AT's owner is in the opposite category.** Since **6 Oct 2026** it is **Skydance** (the closed Paramount–WBD merger). Its merged games unit (WB Games and Paramount Games Studio under Tony Driscoll) will very likely also own AT's Roblox decisions [inference]; it already oversees the unit behind Paramount's Roblox program, which is:
  - the most documented Western fan-game enforcer on Roblox (*The Spongy Construction Project* was removed in 2022, June 2023 and October 2023);
  - a successful licensor (Gamefam and Wonder Works partner games).
- **Roblox now makes enforcement cheap** [Docs `ip-licensing/*.md`, `production/publishing/rights-manager.md`]:
  - License Manager matches public, actively played games **daily by keywords and images**.
  - A standard license offer **auto-activates after 14 days** unless disputed, at a revenue share the rights holder sets (**0–95 %**).
  - Rights Manager accepts removal requests of **up to 250 links**.

**Likelihood of at least one adverse rights-holder action within 12 months of public launch.** These are judgement estimates from precedent (GDD §5.7; they assume a monetized game with paid Wishes), not statistics; read them as a gradient.

| Scenario | Small (< ~500 CCU) | Breakout (~5k–50k CCU) | Hit + trigger event | Most likely form |
|---|---|---|---|---|
| (a) **Codename build.** Original models that are still recognizable AT characters under epithets. No AT words in metadata. No official assets. | **3–10 %** | **20–40 %** | **50–75 %** | DMCA or Rights Manager removal if Skydance has a partner AT game. A license or conditional offer if AT is on License Manager. |
| (b) As (a), but "Adventure Time" (or Finn, Jake, Ooo) in the title, thumbnail, tags, group or Discord | **10–20 %** | **40–60 %** | **70–90 %** | Trademark plus copyright; found by keyword search |
| (c) Ripped official art or audio | **20–40 %** | **60–85 %** | **85–95 %+** | Fastest and least defensible; account strikes |
| Reference: **Original build** (no AT likeness) | < 1 % | ~1–3 % | ~2–5 % | Only on design drift or a nuisance/competitor claim |

The biggest trigger is an official or licensed AT Roblox game (§8.6).

**What a removal costs:**
- the game;
- Robux-backed player progress, with refund backlash (*Pokémon Brick Bronze*);
- the publishing group's standing under the repeat-infringer policy;
- the chance of a license.

#### 8.1.4 Standing rules for the agent

| ID | Rule |
|---|---|
| R-IP-01 | Every player-visible string comes from `Skin.text` / `Skin.style` / `Skin.move` (§0.4). Surfaces are listed in §8.2.1. |
| R-IP-02 | Canonical names (rules list A, §8.4.2) appear only in four places: `src/Shared/Skin/Packs/Canon.luau`, `tools/lint_strings.rules.json`, `tests/ip/fixtures/**` and `docs/**`. They never appear elsewhere in code, comments, identifiers, file names, asset keys, instance names, test names, commit messages, log lines, AI prompts or asset searches. |
| R-IP-03 | `Original.luau` is always complete and clean. A task that adds a Skin key adds the Original value first. A task that adds an identity key (`style.*`, `move.*`, `map.*`, mastery titles) also adds the Codename and Canon values (§8.2.3). |
| R-IP-04 | The agent never sets `PublicCodenameAcknowledged`, `LicenseConfirmed`, `Ip.LicenseRef` or `Ip.PublicAckExpiresUnix`. It never writes `docs/agent/PUBLISH_ACK.md`, `LICENSE_CONFIRMATION.md` or `TITLE_CLEARANCE.md`, by any tool (`lune run` scripts included; every `tools/*.luau` writer refuses those paths, §8.4.5). It never fills a sign-off cell in `RECOGNIZABILITY.md` or `ASSET_PROVENANCE.md`. If the lint finds a flag set without a valid, owner-committed file (L09/L10 provenance, §8.4.5), the agent opens a blocker of type `safety`. |
| R-IP-05 | The agent never publishes (Part 1 §1.7.1). It runs the publish gate (§8.4.7) before requesting H-11. |
| R-IP-06 | Graybox only. AI generation is for generic props only, with linted prompts (§8.5.4). No Creator Store asset may depict or be named after the show (§8.5.5). |
| R-IP-07 | No voice cloning, no sound-alike direction and no show music (§8.5.7). |
| R-IP-08 | The agent never decides IP questions. Any IP signal opens a blocker of the new type `ip` with the facts and the matching playbook row. IP signals are: a §8.6 trigger, a lint finding that renaming cannot fix, and a recognizability fail. |
| R-IP-09 | Original content is written from the archetype descriptions (GDD §7.2 archetype column, §8.3), never by find-and-replace on canon text. Records are never deleted. |
| R-IP-10 | The repository stays private, because it holds the canon GDD and the Canon pack. |

#### 8.1.5 Status in M0 and M1-G

All milestone work runs in the private dev universe with the Codename pack. A public publish is impossible until assets are final: §0.5, and Part 7 item R2 (`check_manifest --release`) fails on placeholders. This part's gate is therefore built and **dry-run** in phase P8 (the agent procedure `ip.gate_dry_run`, §8.4.7), so it is ready when the owner decides.

---

### 8.2 Skin system rules

#### 8.2.1 Player-visible surfaces

| Surface | Members | Rule |
|---|---|---|
| GUI text | `TextLabel.Text`, `TextButton.Text`, `TextBox.Text`, `TextBox.PlaceholderText`; Part 6 `UI/Create` helpers | Skin key only |
| World text | BillboardGui and SurfaceGui labels (nameplate, placards); `ProximityPrompt.ActionText` / `.ObjectText` | Skin key only, set at runtime on the client (Part 6 §6.3: builders write `""`) |
| Server text | `Notify` payloads (keys only, Part 2 §2.7.4 row 22); `Player:Kick` reasons (`ui.error.*`) | Key only |
| Voice and subtitles | `AudioTextToSpeech.Text` (Studio placeholder barks, Part 6); subtitles | `ui.vo.*` |
| Names drawn by the core UI | `Tool.Name` / `Tool.ToolTip`, `Team.Name`, `leaderstats` child names, NPC `Humanoid.DisplayName` | None in M1-G. If one is added, set it from Skin at runtime. |
| Platform metadata | Experience title, description, tags, icon, thumbnails, update names; product, pass and badge names and descriptions; group, Discord and social names | `docs/agent/PUBLISH_METADATA.md`, linted (§8.4.8) |

All members above were checked in the API docs (`TextLabel.yaml`, `TextButton.yaml`, `TextBox.yaml`, `ProximityPrompt.yaml`, `AudioTextToSpeech.yaml`, `Tool.yaml`, `Humanoid.yaml`).

#### 8.2.2 Keys and namespaces

**Grammar:** `^[a-z][a-z0-9_]*(%.[a-z0-9_]+)+$` (lowercase and dotted, Part 2 §2.10 item 3).

**Interpolation:** `{token}`. Tokens whose names end in `Id` are resolved by Part 2. Codename and Canon override values must use exactly the same token set as the Original value; rule L03, extended by this part, enforces it.

| Namespace | Pattern | Defined in | Key list owner |
|---|---|---|---|
| style | `style.<id>.name`, `style.<id>.epithet` (all 18 IDs) | Original, Codename, Canon | §0.4 / §8.3.2 |
| move | `move.<moveId>.name` for `m1, passive, q, 1–4, awaken, a1–a4` of every **implemented** style | Original, Codename, Canon | §0.4 / §8.3.3 |
| map | `map.<mapId>.name`, `map.<mapId>.zone.<zoneId>` | Original, Codename, Canon | Part 6 / §8.3.4 |
| flair, rarity | `flair.<id>.name`, `rarity.<id>.name` | Original (Canon overrides `flair.nightosphere.name`) | §0.4 / §8.3.5 |
| variant | `variant.<styleId>.<vid>.name` (`v1`, `v2`) | Original (generated) | Part 5 / §8.3.6 |
| title | `title.level_<n>`, `title.mastery_<styleId>_5`, `title.master_<styleId>` | Original. Mastery titles are generated in every pack from that pack's style names. | Part 5 / §8.3.6 |
| product, pass | `product.<key>.name`, `pass.<key>.name` | **Original only** (rule L08) | Part 5 / §8.3.6 |
| ui | `ui.<screen>.<key>` | Original. Codename and Canon may override only the keys marked in §8.3.7. | Parts 2, 5, 6, 8 |

#### 8.2.3 Pack files: shape, purity, inheritance

Shape. Part 2 defines `{ id, strings }`; this part adds the optional field `looks`:

```lua
--!strict
-- Shape of a pack module's return value (Part 2 Pack type plus Part 8's optional `looks`).
export type LookOverride = {
	palette: { primary: string, secondary: string, accent: string }?, -- "#RRGGBB"
}
export type Pack = {
	id: "Codename" | "Original" | "Canon",
	strings: { [string]: string },
	looks: { [string]: LookOverride }?, -- keyed by styleId; only Original defines entries in M1-G
}
```

Rules:
1. **Pure and require-free.** A pack file contains literal tables and loops only. It is loaded three ways:
   - by Studio `require`;
   - by Part 7's Lune loader;
   - from a `git show HEAD:` copy by the lint (§8.4.3).
   
   It starts with `--!strict` and builds `strings` through a local `put()` that asserts no duplicate key.
2. **Original is the complete base.** Every key referenced by `src/` resolves in Original (rule L02a).
3. **Codename and Canon contain overrides only.** Resolution falls back to Original (Part 2 fallback order). One exception: **identity namespaces** must be complete in every pack, so that a screen never mixes two packs' names (rule L02b). The identity namespaces are:
   - `style.*`;
   - `move.<implemented style>.*`;
   - `map.*`;
   - `title.mastery_*` and `title.master_*`.
4. **Placement of `looks`.** `looks` is empty in Codename and Canon, because the Part 4 `StyleDef.palette` values *are* the Codename look. Original defines a palette for each implemented style (§8.5.6).
5. **Coverage semantics for Part 7's L02 and `skin.pack_key_coverage`.** "Exists in Codename" means `key ∈ Codename ∪ Original` for non-identity namespaces and `key ∈ Codename` for identity namespaces.

#### 8.2.4 Missing-key behaviour

- **Runtime** (Part 2, unchanged). Lookup order is active pack → Original → literal `"[" .. key .. "]"`. Each (pack, key) miss logs once: `[ALG][Skin] WARN missing key=<key> pack=<pack>`. The bracket marker is deliberate: it is visible in dev and easy to spot in screenshots.
- **Host.** L02 fails for any key referenced in `src/` that is missing from Original, and for any identity key missing from a non-stub Codename or Canon pack (strip stubs, §8.3.8, are skipped; L14 checks which packs are stubs).
- **Studio.** Scenario `ip.no_missing_keys_tour`, run inside the agent procedure `ip.gate_dry_run` while the packs are stripped (§8.4.7):
  1. Open every screen (Hud with the Wallet & level and Quests elements, StyleWheel with the Shards and Variant controls, Wish with odds and a 10× reveal, Shop, Store, Settings including the Safety tab, Report, Results, Training panel).
  2. Trigger each Notify kind.
  3. Assert that `Skin.missing()` is empty on server and client and that no `[ALG][Skin] WARN missing` line was printed.

#### 8.2.5 Where IP strings may never appear

| Place | Rule | Enforced by |
|---|---|---|
| `src/**`, `tests/**` (except `tests/ip/fixtures/**`) and `tools/**` (except the rules file and the lint itself): code, comments, identifiers, string literals, file and folder names | No list A term and no Canon-only phrase | L04 (`src` scope, extended to `tests` and `tools`), L13 |
| Asset keys and `Assets` `note` fields | IP-neutral only (keys already use style IDs) | L04 (`Assets.luau` is in `src`) |
| Instance names created by builders or at runtime | IP-neutral | L04 on literals, plus the Studio tree scan L15 |
| Badge, pass and product names and descriptions | Original strings only, never Codename or Canon, on every track except a licensed Canon build | L08, L11 |
| Telemetry event, funnel and step names; custom-field values | Lowercase IDs and enums only. Never pass `Skin.*` output to `Telemetry`. | L04, plus review |
| Log lines | Keys and IDs only, never resolved Skin text | Review |
| Commit messages, PR titles, branch names | IDs only | `lune run tools/lint_strings.luau --text "<msg>" --scope src` (optional `commit-msg` hook) |
| AI prompts and Creator Store search queries | Lists A, B, C and E | L12; `--text … --scope prompt` before every `generate_*` / `search_asset` call |

**Exempt identifiers.** Two groups of fixed internal IDs are exempt.

Part 0 fixed IDs that embed AT place names (§8.9 item 1):
- `candy_kingdom`;
- `CandyKingdom`;
- `BuildMap_CandyKingdom`;
- `nightosphere` (the lowercase Flair ID).

Part 6 zone IDs, which appear as instance names (`Geometry.<ZoneId>`) and `ZoneId` attributes and would otherwise match the dynamic Codename-only phrases of L13 (§8.9 item 10):
- `castle_plaza`;
- `candy_streets`;
- `barracks`;
- `royal_gardens`;
- `dungeon`;
- `earldom_border`.

Exempt identifiers are matched as whole identifiers, with case sensitivity, in the `src` and `tree` scopes, and replaced by `<id>` before scanning. Each run reports them once as the warning `W-EXEMPT-ID count=<n>`. A capitalized `Nightosphere` or the phrases `Candy Kingdom` and `Castle Plaza` are **not** exempt.

The exemption covers the dev lint only. In a public gate, rule **L22** (§8.4.3) treats every exempt identifier found in the **tree** dump as an error for an Original target, and requires the PUBLISH_ACK box S4 for a Codename target, because instance names and attributes ship to every client. L22 clears once the §8.9 items 1 and 10 renames land.

#### 8.2.6 Effective pack at runtime (runtime guard)

`Config.SkinPack` can change at runtime through ConfigService `ALG_SkinPack` (Part 2 §2.6.4). A public build therefore needs a runtime guard as well as the publish gate. Part 8 adds one pure function to `Shared/Skin`. ConfigReplicator calls it and publishes the **resolved** pack in the snapshot, so clients keep calling `Skin.setPack(cfg.SkinPack, cfg.LicenseConfirmed)` unchanged.

```lua
--!strict
-- Addition to src/Shared/Skin/init.luau (Part 2 owns the module). Pure. Spec §8.2.6.
export type PackGate = {
	licenseConfirmed: boolean,
	publicCodenameAcknowledged: boolean,
	ackValid: boolean, -- now <= Config.Ip.PublicAckExpiresUnix (computed by the server)
	devContext: boolean, -- Studio, or game.GameId listed in Config.Ip.DevUniverseIds (computed by the server)
}
export type PackDowngrade = "canon_locked" | "codename_public_unacked" | "codename_ack_expired"

function Skin.resolvePack(requested: PackId, gate: PackGate): (PackId, PackDowngrade?)
	if requested == "Canon" and not gate.licenseConfirmed then
		return "Original", "canon_locked"
	end
	if requested == "Codename" and not gate.devContext and not (gate.publicCodenameAcknowledged and gate.ackValid) then
		if gate.publicCodenameAcknowledged then
			return "Original", "codename_ack_expired"
		end
		return "Original", "codename_public_unacked"
	end
	return requested, nil
end
```

```lua
--!strict
-- In ConfigReplicator (server), applied to every candidate snapshot before it is published (v1 and every overlay).
local RunService = game:GetService("RunService")

local function isDevContext(cfg: Config): boolean
	return RunService:IsStudio() or table.find(cfg.Ip.DevUniverseIds, game.GameId) ~= nil
end

local function applyPackGuard(cfg: Config)
	local devContext = if ConfigReplicator._devContextOverride ~= nil
		then ConfigReplicator._devContextOverride -- Studio test seam (TestHarness only)
		else isDevContext(cfg)
	local effective, reason = Skin.resolvePack(cfg.SkinPack, {
		licenseConfirmed = cfg.LicenseConfirmed,
		publicCodenameAcknowledged = cfg.PublicCodenameAcknowledged,
		ackValid = DateTime.now().UnixTimestamp <= cfg.Ip.PublicAckExpiresUnix,
		devContext = devContext,
	})
	if reason ~= nil then
		Log.error("Skin", "pack_downgraded", { requested = cfg.SkinPack, effective = effective, reason = reason })
		if not devContext then
			cfg.Features.CodenameLooks = false -- §8.6.3: also hide the Codename look set (Part 6 AppearanceController)
		end
	end
	cfg.SkinPack = effective
end
```

Outside a dev context, a downgrade therefore switches the names (resolved pack), the palettes (`Skin.look` of the resolved pack) and the Codename Style Gear and summons (`Features.CodenameLooks = false`) together.

**Expiry on long-running servers.** `ackValid` depends on the clock, so a server that started before `valid_until` must not keep Codename after it. ConfigReplicator keeps the last pre-guard candidate. Outside Studio, while that candidate requests `Codename` and `devContext` is false, a `task.delay` loop re-runs `applyPackGuard` on a copy every `GUARD_RECHECK_SEC = 3600` s (tuning default, a ConfigReplicator constant) and publishes a new version when the effective pack changes. The ACK lapse therefore takes effect within an hour on every live server, with the `reason=codename_ack_expired` log line.

Truth table (Lune `skin.resolve_pack_matrix`, seven rows):

| Requested | `licenseConfirmed` | `devContext` | `publicCodenameAcknowledged` | `ackValid` | Effective | Reason |
|---|---|---|---|---|---|---|
| Original | any | any | any | any | Original | none |
| Codename | any | true | any | any | Codename | none |
| Codename | any | false | true | true | Codename | none |
| Codename | any | false | true | false | **Original** | `codename_ack_expired` |
| Codename | any | false | false | any | **Original** | `codename_public_unacked` |
| Canon | true | any | any | any | Canon | none |
| Canon | false | any | any | any | **Original** | `canon_locked` |

Expected log lines: `[ALG][Skin] ERROR pack_downgraded effective=Original reason=codename_public_unacked requested=Codename`, and on a lapsed acknowledgement `… reason=codename_ack_expired …`.

APIs used:
- `RunService:IsStudio()`, security None [Docs `RunService.yaml`].
- `DataModel.GameId` (int64; read None; ReadOnly, NotReplicated) [Docs `DataModel.yaml`].
- `DateTime.now()` and `DateTime.UnixTimestamp` (UTC seconds; the `UnixTimestamp` property is used in [Docs `production/monetization/roblox-plus.md`]).
- `Player:IsInGroupAsync(groupId)` (Yields; security None; not deprecated, unlike `Player:IsInGroup`) and `Player:Kick(message)` [Docs `Player.yaml`].

**New config section** (Part 8; code-only, never ConfigService-overridable):

```lua
--!strict
-- Added to src/Shared/Config/Defaults.luau. JSON-safe. Spec §8.2.6.
Ip = {
	DevUniverseIds = {} :: { number }, -- the dev universe only: the agent copies `universeId` from docs/agent/dev-place.json (L16)
	DevGroupId = 0,                     -- the dedicated dev group: the agent copies `groupId` from docs/agent/dev-place.json (L16)
	LicenseRef = "",                    -- human-set; must equal LICENSE_CONFIRMATION.md `license_ref` when LicenseConfirmed = true (L10)
	PublicAckExpiresUnix = 0,           -- human-set with PublicCodenameAcknowledged: Unix seconds of PUBLISH_ACK `valid_until` 23:59:59 UTC (L09)
},
```

- The agent may write only the dev universe ID and the dev group ID from `docs/agent/dev-place.json` (human-confirmed at H-01) into `DevUniverseIds` and `DevGroupId`. L16 enforces both.
- **Groups.** The Codename dev universe is owned by a **dedicated dev group** that owns no public experience (Part 1 H-01). The production group owns only universes that run the Original pack (or a licensed Canon pack) and never hosts Codename-look uploads. A public Codename build, if the owner chooses one, lives in a third group that is neither (PUBLISH_ACK `target_group_id`, §8.4.4). Assets needed in production (for example placeholder animations uploaded at H-07/H-08) are re-uploaded to the production group at release. Removals and repeat-infringer strikes land on the owning group, so this keeps them away from the group that publishes the game.
- **The dev universe must stay private.** Making it public is a public publish and requires the gate. The human re-checks it weekly (TW-11, §8.6.1).
- **Dev universe members only** (defence in depth, in case the dev universe is made public by mistake). PlayerService runs the check (Part 2 §2.3.5 hosts the code; this part owns the rule). On join, before `DataService.loadAsync`, outside Studio, when `table.find(cfg.Ip.DevUniverseIds, game.GameId) ~= nil`: `local ok, member = pcall(player.IsInGroupAsync, player, cfg.Ip.DevGroupId)`. Unless `ok and member`, the player is kicked with `player:Kick(Skin.text("ui.error.restricted"))` and the server logs `[ALG][Skin] ERROR dev_universe_non_member userId=<id>`. A failed call also kicks (fail closed).

**Studio-only additions:**
- `DevCmd setSkinPack(pack)` runs the same guard path as an `ALG_SkinPack` overlay.
- The seam `ConfigReplicator._devContextOverride: boolean?` exists only for TestHarness.
- The seam `PlayerService._devAccessCheck(player: Player, opts: { forceDevUniverse: boolean?, member: boolean?, dryRun: boolean? }?): boolean` runs the members-only rule with the universe test and the group answer forced by `opts`. With `dryRun = true` it returns the decision and logs (adding `dry_run=true`) instead of kicking. `opts` is ignored outside Studio, like every seam.

#### 8.2.7 Look overrides

`Skin.look(styleId, pack?): LookOverride?` reads **one pack only** (the active pack unless `pack` is given), with no fallback. If it fell back to Original, Codename would inherit Original palettes. The optional `pack` argument exists for the Codename-looks kill switch (§8.6.3), which needs Original palettes while Codename names stay active.

```lua
--!strict
-- Addition to src/Shared/Skin/init.luau. Pure. Spec §8.2.7.
function Skin.look(styleId: string, pack: PackId?): LookOverride?
	local looks = PACKS[pack or activeId].looks
	return if looks ~= nil then looks[styleId] else nil
end
```

Presentation code never reads `StyleDef.palette` directly for colours. Part 6 adds one helper to `UI/Theme.luau` (this code is canonical; Part 6 §6.9.4 reproduces it) and uses it for:
- proxy-rig colours;
- the nameplate chip and ring;
- Style Wheel chips (including the Variant picker);
- VFX palette tints;
- Style Gear colours;
- Shop cosmetic cards.

```lua
--!strict
-- src/StarterPlayerScripts/UI/Theme.luau (Part 6 file) — Part 8 helper. Spec §8.2.7.
-- Variant rule: Part 6 §6.8.2 (v1 swaps secondary/accent; v2 and later darken 25 %). Kill switch: §8.6.3.
export type StylePalette = { primary: Color3, secondary: Color3, accent: Color3 }
local BLACK = Color3.new(0, 0, 0)
function Theme.stylePalette(styleId: string, variantId: string?): StylePalette
	-- Features.CodenameLooks == false: Original palettes whatever pack is active.
	local pack: Skin.PackId? = nil
	if Runtime.get().Features.CodenameLooks == false then
		pack = "Original"
	end
	local o = Skin.look(styleId, pack)
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
		return { primary = base.primary, secondary = base.accent, accent = base.secondary }
	end
	return {
		primary = base.primary:Lerp(BLACK, 0.25),
		secondary = base.secondary:Lerp(BLACK, 0.25),
		accent = base.accent:Lerp(BLACK, 0.25),
	}
end
```

`Color3.fromHex`, `Color3.new` and `Color3:Lerp` are verified [Docs `datatypes/Color3.yaml`]. Style Gear **shapes** are a build-time choice (look set, §8.5.6), not a runtime one; the kill switch can only hide Codename-look gear, not reshape it (§8.6.3).

#### 8.2.8 Verify

**Lune tests:**
- `skin.resolve_pack_matrix`: all seven rows above.
- `skin.look_active_only`: Codename active gives `nil`; Original active gives the palette; `Skin.look("hero", "Original")` gives the Original palette while Codename is active.
- Part 2's `skin.fallback_order` and `skin.canon_locked`.
- Part 7's `skin.pack_key_coverage`, with the §8.2.3 coverage semantics (strip stubs skipped, §8.4.3 L02).

**Studio scenarios:**
- `ip.runtime_guard_public_sim`. Set the seam `_devContextOverride = false`, then `Dev.Command setSkinPack Codename` with the acknowledgement flag false. Expect `Runtime.get().SkinPack == "Original"`, `Runtime.get().Features.CodenameLooks == false` and the `pack_downgraded` log line (run with `allowErrors {"pack_downgraded"}`). Clear the seam and the overlay at the end.
- `ip.canon_locked_runtime`. `setSkinPack Canon` gives Original and `reason=canon_locked` (run with `allowErrors {"pack_downgraded"}`).
- `ip.pack_switch_live` (emergency switch, §8.6.3 step 1). In dev, with the Codename look set built: `Dev.Command setConfig Features {"CodenameLooks": false, "PaidRandomItems": false}`, then `setSkinPack Original`. Within 1 s: the local nameplate chip text equals `Original` `style.hero.name` and its colour equals `#C8553D` (Original hero primary, §8.5.6); every `Gear_*`/`Summon_*` clone whose template has `LookSet == "Codename"` is hidden (`LocalTransparencyModifier == 1` on all its parts). Restore both values at the end.
- `ip.dev_universe_members_only` (§8.2.6 seam). `PlayerService._devAccessCheck(P, {forceDevUniverse = true, member = false, dryRun = true})` returns false and logs `[ALG][Skin] ERROR dev_universe_non_member dry_run=true userId=<id>` (run with `allowErrors {"dev_universe_non_member"}`); `member = true` returns true with no log; with `opts = nil` in Studio it returns true (Studio skip).
- `ip.no_missing_keys_tour` (§8.2.4).

---

### 8.3 Pack content tables

The tables are ready to paste into `Packs/Original.luau`, `Packs/Codename.luau` and `Packs/Canon.luau`. "inherits" means the key is **not** written in that pack.

#### 8.3.1 Rules per pack, and codename-safe substitutions

- **Original.**
  - Names are written from the GDD §7.2 archetypes, not derived from canon.
  - No term from lists A, B or C. No near-homophones. No "<Name> the <Noun>" pattern.
  - Epithets never quote canon titles.
  - Every Original name passes the line test (T4, §8.5.3) by construction.
- **Codename.**
  - `style.<id>.name` is the **GDD §7.2 epithet verbatim** (§0.4).
  - `style.<id>.epithet` is the GDD §7.2 archetype. `decay` is the one exception: it uses "Decay Sorcerer", because "Lich" is a list A term.
  - Move names are GDD §7.3 verbatim, except the 12 codename-safe substitutions below. Each of those GDD names contains a canonical proper noun, an episode title or show slang, which is exactly what a codename pack exists to remove.
  - Place names are codename-safe variants.
- **Canon.** Canonical names and GDD names verbatim. Locked behind the license (§8.2.6, §8.4.4).

| Move ID | GDD §7.3 (= Canon) | Codename | Reason |
|---|---|---|---|
| `hero.4` | Shmowzow! Flying Kick | Battle-Cry Flying Kick | Show slang |
| `stretch.awaken` | Kaiju Jake | Kaiju Form | Character name |
| `monarch.2` | Banana Guard Squad | Royal Guard Squad | Character name |
| `monarch.4` | Gumball Guardian Beam | Candy Colossus Beam | Character name |
| `monarch.awaken` | Gum War Commander | Candy War Commander | Canon event name |
| `monarch.a2` | Rattleballs Strike | Swordbot Strike | Character name |
| `monarch.a3` | Goliad Link | Psychic Link | Character name |
| `monarch.a4` | Royal Gumball Guardian | Royal Candy Colossus | Character name |
| `rocker.a1` | Empress Eyes | Hypno Gaze | Episode title |
| `frost.q` | GUNTER! | PENGUIN! | Character name |
| `frost.a4` | Ice Kingdom Rising | Frost Spire Rising | Place name |
| `decay.awaken` | Giant Lich | Towering Form | Character name |

Some kept Codename names are descriptive but canon-adjacent: "Vampire Queen Ascendant", "Moon Beast Pounce", "Fool's Frenzy", "The Wishing Eye", "Well of Power", "Starchild Shapeshift". They are in list C, which means they are allowed in Codename and forbidden in Original and metadata. Open question Q-P8-4.

**Later kits.** The table above covers the six slice kits only. For every later kit (Part 4 §4.8), a GDD §7.3 move name gets a Codename substitution, added to this table in the same task, if it contains a list A or B term (§8.4.2), a GDD Appendix D episode title or a quoted line from the show (for example "ONE MILLION YEARS DUNGEON!", "Ate Your Fries", "Maloso Vobiscum"). L13 alone cannot catch such a name once it is copied into Codename, because the phrase would then exist in Codename too.

#### 8.3.2 Styles (all 18; `style.<id>.name` / `style.<id>.epithet`)

| ID | Original name | Original epithet | Codename name (GDD §7.2 epithet) | Codename epithet | Canon name | Canon epithet |
|---|---|---|---|---|---|---|
| `hero` | Pell | The Seed Knight | The Last Human | Sword Hero | Finn | The Last Human |
| `stretch` | Mochi | The Rubber Capybara | The Magic Dog | Stretchy Companion | Jake | The Magic Dog |
| `sentry` | Corporal Cog | The Clockwork Spearman | The Royal Banana | Comic Spearman | Banana Guard | The Royal Banana |
| `granny` | Old Mossy | The Tortoise Baker | The Crystal Granny | Baker Granny | Tree Trunks | The Crystal Granny |
| `diva` | Jellabelle | The Jellyfish Diva | The Lumpy Royal | Floaty Diva | Lumpy Space Princess | The Lumpy Royal |
| `tyrant` | Duke Squall | The Shrieking Duke | The Sour Earl | Shrieking Tyrant | Earl of Lemongrab | The Sour Earl |
| `console` | Dial | The Living Radio | The Living Console | Gadget Console | BMO | The Living Console |
| `grappler` | Granite | The Cave Wrestler | The Hyooman | Super-strong Grappler | Susan Strong | The Hyooman |
| `monarch` | Regent Fizz | The Alchemist Regent | The Candy Monarch | Candy-Science Monarch | Princess Bubblegum | The Candy Monarch |
| `flame` | Vela | The Ember Heir | The Flame Queen | Fire Elemental | Flame Princess | The Flame Queen |
| `butler` | Hollins | The Candlewick Valet | The Dark Butler | Necromancer Butler | Peppermint Butler | The Dark Butler |
| `rocker` | Vesper | The Night Rocker | The Vampire Queen | Vampire Rocker | Marceline | The Vampire Queen |
| `frost` | Sleet | The Frost Hermit | The Crowned Hermit | Ice Wizard | Ice King | The Crowned Hermit |
| `huntress` | Fernwyn | The Thorn Archer | The Forest Huntress | Forest Archer-Mage | Huntress Wizard | The Forest Huntress |
| `decay` | Gloam | The Rot Sovereign | The Undying End | Decay Sorcerer | The Lich | The Undying End |
| `demon` | Malgrave | The Abyss King | The Lord of Evil | Demon Lord | Hunson Abadeer | The Lord of Evil |
| `wish` | Tallis | The Wish Weaver | The Wishmaster | Cosmic Wish-Granter | Prismo | The Wishmaster |
| `chaos` | Riftmaw | The Unmaking | Discord Incarnate | God of Chaos | GOLB | Discord Incarnate |

#### 8.3.3 Move names for the six vertical-slice kits (`move.<moveId>.name`)

The `m1` names are new (tuning default); the GDD does not name M1 chains. The Canon `m1` names equal Codename's.

Original move names were drafted from the archetype and the move's effect, not from the canon name (R-IP-09). The tables put the three packs side by side only so the agent can paste them; the Original column is not a list of replacements. T6 reviewers (§8.5.3) check every slot whose Original name is a synonym of its Canon name (for example `frost.2` Ice Cage, `frost.4` Snow Golem, `frost.a2` Seal Stampede) and may send it back for renaming.

**`hero`** (GDD §7.3.1)

| Slot | Original | Codename | Canon |
|---|---|---|---|
| m1 | Seed-Blade Combo | Sword Combo | Sword Combo |
| passive | Never Quit | Hero's Heart | Hero's Heart |
| q | Blade Swap | Sword Swap | Sword Swap |
| 1 | Leaping Slash | Hero Leap | Hero Leap |
| 2 | Rapid Jabs | Flurry of Fists | Flurry of Fists |
| 3 | Hook Throw | Grass Sword Lash | Grass Sword Lash |
| 4 | Flying Dropkick | Battle-Cry Flying Kick | Shmowzow! Flying Kick |
| awaken | Second Wind | Grass Arm | Grass Arm |
| a1 | Fan of Cuts | Grass Whip Barrage | Grass Whip Barrage |
| a2 | Rubble Grip | Tower Arm | Tower Arm |
| a3 | Echo Slash | Paradox Slash | Paradox Slash |
| a4 | Rain of Blades | Every Sword I've Ever Swung | Every Sword I've Ever Swung |

**`stretch`** (GDD §7.3.2)

| Slot | Original | Codename | Canon |
|---|---|---|---|
| m1 | Rubber Combo | Stretch Combo | Stretch Combo |
| passive | Bouncy Hide | Rubber Body | Rubber Body |
| q | Stretch Grapple | Stretch Grapple | Stretch Grapple |
| 1 | Long Arm Jab | Long Arm Jab | Long Arm Jab |
| 2 | Giant Fist | Giant Fist | Giant Fist |
| 3 | Rubber Wall | Stretchy Shield | Stretchy Shield |
| 4 | Wrap-Up | Wrap-Up | Wrap-Up |
| awaken | Titan Mode | Kaiju Form | Kaiju Jake |
| a1 | Stomp | Stomp | Stomp |
| a2 | Pick-Up | Pick-Up | Pick-Up |
| a3 | Belly Bounce | Belly Bounce | Belly Bounce |
| a4 | Wild Shapeshift | Starchild Shapeshift | Starchild Shapeshift |

**`monarch`** (GDD §7.3.9)

| Slot | Original | Codename | Canon |
|---|---|---|---|
| m1 | Gadget Combo | Science Combo | Science Combo |
| passive | Experiment Chain | Scientific Method | Scientific Method |
| q | Analyze | Analyze | Analyze |
| 1 | Sticky Shot | Gum Blaster | Gum Blaster |
| 2 | Clockwork Guards | Royal Guard Squad | Banana Guard Squad |
| 3 | Fume Flask | Chemistry Flask | Chemistry Flask |
| 4 | Colossus Eye Beam | Candy Colossus Beam | Gumball Guardian Beam |
| awaken | War Regent | Candy War Commander | Gum War Commander |
| a1 | Mortar Volley | Candy Mortar | Candy Mortar |
| a2 | Duelist Drone | Swordbot Strike | Rattleballs Strike |
| a3 | Levitation Field | Psychic Link | Goliad Link |
| a4 | Brass Colossus | Royal Candy Colossus | Royal Gumball Guardian |

**`rocker`** (GDD §7.3.12)

| Slot | Original | Codename | Canon |
|---|---|---|---|
| m1 | Riff Combo | Bass Combo | Bass Combo |
| passive | Vital Drain | Color Drain | Color Drain |
| q | Moth Shift | Bat Shift | Bat Shift |
| 1 | Keytar Slam | Axe Bass Swing | Axe Bass Swing |
| 2 | Shadow Ambush | Invisible Ambush | Invisible Ambush |
| 3 | Life Siphon | Red Drain | Red Drain |
| 4 | Fright Mask | Monster Scare | Monster Scare |
| awaken | Night Ascendant | Vampire Queen Ascendant | Vampire Queen Ascendant |
| a1 | Mesmer Gaze | Hypno Gaze | Empress Eyes |
| a2 | Beast Pounce | Moon Beast Pounce | Moon Beast Pounce |
| a3 | Encore Frenzy | Fool's Frenzy | Fool's Frenzy |
| a4 | Night Swoop | Giant Bat | Giant Bat |

**`frost`** (GDD §7.3.13)

| Slot | Original | Codename | Canon |
|---|---|---|---|
| m1 | Frost Combo | Ice Combo | Ice Combo |
| passive | Cloak Glide | Beard Flight | Beard Flight |
| q | Seal Pup! | PENGUIN! | GUNTER! |
| 1 | Frost Bolt | Ice Lightning | Ice Lightning |
| 2 | Ice Cage | Ice Prison | Ice Prison |
| 3 | Ice Skate | Ice Slide | Ice Slide |
| 4 | Snow Golem | Ice Golem | Ice Golem |
| awaken | Deep Winter | The Wishing Eye | The Wishing Eye |
| a1 | Whiteout | Blizzard | Blizzard |
| a2 | Seal Stampede | Penguin Avalanche | Penguin Avalanche |
| a3 | Icicle Ring | Frozen Blades | Frozen Blades |
| a4 | Glacier Spire | Frost Spire Rising | Ice Kingdom Rising |

**`decay`** (GDD §7.3.15)

| Slot | Original | Codename | Canon |
|---|---|---|---|
| m1 | Rot Combo | Decay Combo | Decay Combo |
| passive | Blight | Extinction | Extinction |
| q | Dread Whisper | Hypnotic Whisper | Hypnotic Whisper |
| 1 | Rot Bolt | Decay Bolt | Decay Bolt |
| 2 | Grave Grasp | Thrall Grasp | Thrall Grasp |
| 3 | Rot Pool | Well of Power | Well of Power |
| 4 | Soul Hook | Possession | Possession |
| awaken | Colossal Rot | Towering Form | Giant Lich |
| a1 | Blight Wave | Extinction Wave | Extinction Wave |
| a2 | Mire | Grey Slime | Grey Slime |
| a3 | Skull Swarm | Decay Swarm | Decay Swarm |
| a4 | Realm: The Rotting Grove | Realm: The Well of Power | REALM: The Well of Power |

When a new kit is implemented (Part 4 §4.8), its 12 rows are added to all three packs in the same task. Codename rows follow §8.3.1. L02b fails until all three packs have them.

#### 8.3.4 Maps and zones

| Key | Original | Codename | Canon |
|---|---|---|---|
| `map.arena_duel_m0.name` | The Cube Atrium | The Timeless Room | Prismo's Time Room |
| `map.candy_kingdom.name` | Sugarcrest | The Candy Capital | Candy Kingdom |
| `map.training.name` | Training Grounds | Training Grounds | Training Grounds |
| `map.candy_kingdom.zone.castle_plaza` | Keep Plaza | Castle Plaza | Castle Plaza |
| `map.candy_kingdom.zone.candy_streets` | Gumdrop Lanes | Gumdrop Streets | Candy Streets |
| `map.candy_kingdom.zone.barracks` | Guard Yard | Guard Barracks | Banana Guard Barracks |
| `map.candy_kingdom.zone.royal_gardens` | Floss Gardens | Royal Gardens | Royal Gardens |
| `map.candy_kingdom.zone.dungeon` | The Undercroft | The Dungeon | The Dungeon |
| `map.candy_kingdom.zone.earldom_border` | Sour Hills | Earldom Border | Earldom Border |

#### 8.3.5 Rarity and Flair names

- **Rarity** (Original only; the other packs inherit): `starter` = Starter, `common` = Common, `uncommon` = Uncommon, `rare` = Rare, `epic` = Epic, `legendary` = Legendary, `mythic` = Mythic, `cosmic` = Cosmic.
- **Flair** (Original; the other packs inherit unless noted): `standard` = Standard, `shiny` = Shiny, `golden` = Golden, `crystal` = Crystal, `rainbow` = Rainbow, `nightosphere` = **Infernal** (Canon overrides it with "Nightosphere"), `glitched` = Glitched.

#### 8.3.6 Generated families

These are built by loops inside each pack file.

| Family | Original | Codename | Canon |
|---|---|---|---|
| `variant.<styleId>.v1.name` / `.v2.name` (18 × 2) | "Dusk Variant" / "Dawn Variant" | inherits | inherits. Canon Variants are added only under license (GDD §9.3; Variants are a Critical-risk element without one). |
| `title.level_<n>`, n = 10…100 step 10 | 10 Scrapper, 20 Brawler, 30 Duelist, 40 Veteran, 50 Champion, 60 Vanguard, 70 Paragon, 80 Mythmaker, 90 Legend, 100 Grand Legend | inherits | inherits |
| `title.mastery_<styleId>_5` | `"<pack style name> Adept"`, e.g. "Pell Adept" | "The Last Human Adept" | "Finn Adept" |
| `title.master_<styleId>` | `"<pack style name> Master"` | Codename names | Canon names |
| `product.<key>.name` | `wishes_10` "10 Wishes", `wishes_55` "55 Wishes", `wishes_120` "120 Wishes", `wishes_300` "300 Wishes" | **never** (L08) | **never** (L08) |
| `pass.<key>.name` | `vip` "Royal VIP", `quick_wish` "Quick Wish", `lucky_jar` "Lucky Jar" | **never** | **never** |

Product and pass names are also the **Creator Hub** names. The purchase prompt shows the platform name, not Skin, so they must stay IP-neutral on every non-licensed track (§8.4.8).

#### 8.3.7 Core UI strings

- The Original column is the base.
- "inherits" means the key is not written in that pack.
- `{tokens}` must match across packs.
- Keys marked † are new keys introduced here; Parts 5 and 6 use them instead of inventing their own.

| Key | Original | Codename | Canon |
|---|---|---|---|
| `ui.game.title` † | Algebraic! *(title pending clearance, §8.7)* | inherits | Adventure Time: Algebraic! |
| `ui.world.name` † | The Patchwork Wilds | inherits | The Land of Ooo |
| `ui.hud.meter_name` † | Awaken Meter | Algebraic Meter | Algebraic Meter |
| `ui.hud.awaken_ready` † | Awaken! | Go Algebraic! | Go Algebraic! |
| `ui.hud.breakaway` | Breakaway! | inherits | inherits |
| `ui.hud.protected` | Protected {sec}s | inherits | inherits |
| `ui.hud.combo` | {n} HIT | inherits | inherits |
| `ui.hud.combo_big` | WILD {n} HIT! | ALGEBRAIC {n} HIT! | ALGEBRAIC {n} HIT! |
| `ui.hud.level` † | Lv {level} | inherits | inherits |
| `ui.hud.treasure` † | {n} Treasure | inherits | inherits |
| `ui.hud.shards` † | {n} Shards | inherits | inherits |
| `ui.hud.sparks` † | {n} Cosmic Sparks | inherits | inherits |
| `ui.hud.quests` † | Daily Quests | inherits | inherits |
| `ui.quest.deal_damage` † | Deal {goal} damage | inherits | inherits |
| `ui.quest.perfect_blocks` † | Land {goal} Perfect Blocks | inherits | inherits |
| `ui.quest.kos` † | Get {goal} KOs | inherits | inherits |
| `ui.quest.active_minutes` † | Play for {goal} minutes | inherits | inherits |
| `ui.quest.awaken` † | Use Awaken ×{goal} | inherits | inherits |
| `ui.feed.bot` | Bot | inherits | inherits |
| `ui.nameplate.rookie` | Rookie | inherits | inherits |
| `ui.totem.action` | Change Style | inherits | inherits |
| `ui.totem.object` | Style Totem | inherits | inherits |
| `ui.wheel.in_dev` | In development | inherits | inherits |
| `ui.wheel.trial` | Trial | inherits | inherits |
| `ui.wheel.free` | Free this week | inherits | inherits |
| `ui.wheel.get_hint` | Unlock it with Wishes or at the Trading Post | Unlock it with Wishes or at the Rhyming Goose's shop | Unlock it with Wishes or at Choose Goose's shop |
| `ui.wheel.shards` † | Shards: {n} | inherits | inherits |
| `ui.wheel.convert` † | Convert Shards | inherits | inherits |
| `ui.wheel.mastery_combat_only` † | Mastery 21–30 is earned in combat | inherits | inherits |
| `ui.wheel.variant` † | Variant | inherits | inherits |
| `ui.wheel.variant_base` † | Base | inherits | inherits |
| `ui.wish.title` † | Make a Wish | inherits | inherits |
| `ui.wish.make_1` † | Wish ×1 | inherits | inherits |
| `ui.wish.make_10` † | Wish ×10 | inherits | inherits |
| `ui.wish.odds` † | Odds | inherits | inherits |
| `ui.wish.pity_title` † | Wishmaker's Promise | inherits | inherits |
| `ui.wish.new` † | New! | inherits | inherits |
| `ui.wish.shards` | +{n} Shards | inherits | inherits |
| `ui.wish.fakeout_note` | Fake-outs only ever upgrade: some Epic pulls start as a Rare card that turns into the Epic. | inherits | inherits |
| `ui.wish.locked_paid` † | Locked paid Wishes: {n} | inherits | inherits |
| `ui.wish.auto10` † | Auto ×10 | inherits | inherits |
| `ui.wish.secret` † | ??? | inherits | inherits |
| `ui.shop.title` † | The Trading Post | The Rhyming Goose | Choose Goose's Shop |
| `ui.shop.refresh` † | New stock in {time} | inherits | inherits |
| `ui.shop.coming_soon` † | Coming soon | inherits | inherits |
| `ui.shop.goose_line_1` | New stock arrives when the countdown ends. | Honk and haggle, look and see, the finest styles from me to thee. | = Codename |
| `ui.shop.goose_line_2` | Every style here has a fixed price. | Feathers ruffled, prices fair; pick a style and wear it with flair. | = Codename |
| `ui.shop.goose_line_3` | Save your Treasure for the one you want. | No luck, no tricks, the price is plain; save your treasure, come again. | = Codename |
| `ui.shop.goose_line_4` | No luck needed: what you see is what you get. | Twelve hours pass, my wares will change; come waddle back, it isn't strange. | = Codename |
| `ui.shop.goose_line_5` | Check back after the next restock. | A goose who rhymes has much to sell; pick what you like and fight it well. | = Codename |
| `ui.shop.goose_line_6` | Try a style in Training before you buy it. | Shake my wing and seal the deal; your treasure buys a style that's real. | = Codename |
| `ui.shop.balance` † | Your Treasure: {n} | inherits | inherits |
| `ui.shop.confirm_in_dev` † | This style is still in development. Collect it now and play it when it ships? | inherits | inherits |
| `ui.store.title` † | Store | inherits | inherits |
| `ui.store.mock` † | Test mode | inherits | inherits |
| `ui.store.confirm_4th` | You've bought Wishes several times in the last 24 hours. Buy again? | inherits | inherits |
| `ui.store.spent_week` † | Spent this week: R$ {robux} | inherits | inherits |
| `ui.settings.title` † | Settings | inherits | inherits |
| `ui.settings.tab_safety` † | Safety | inherits | inherits |
| `ui.settings.quiet_barks` † | Quiet barks | inherits | inherits |
| `ui.settings.damage_numbers` † | Damage numbers | inherits | inherits |
| `ui.settings.hold_toggle_run` † | Hold-to-toggle run | inherits | inherits |
| `ui.settings.character_look` † | Character Look | inherits | inherits |
| `ui.settings.style_gear` † | Style Gear | inherits | inherits |
| `ui.settings.own_look_oosona` † | My Toon | inherits | Ooo-sona |
| `ui.settings.own_look_avatar` † | My Roblox Avatar | inherits | inherits |
| `ui.settings.layout_editor_soon` | Touch layout editor coming soon | inherits | inherits |
| `ui.results.environment` | The arena | inherits | inherits |
| `ui.training.slot` | Dummy {n} | inherits | inherits |
| `ui.training.damage_numbers` † | Damage numbers | inherits | inherits |
| `ui.report.button` † | Report | inherits | inherits |
| `ui.report.title` † | Report a player | inherits | inherits |
| `ui.report.reason_exploit` / `_teaming` / `_harassment` / `_other` † | Cheating or exploits / Teaming / Harassment / Something else | inherits | inherits |
| `ui.report.send` † | Send report | inherits | inherits |
| `ui.training.mode_idle` / `_block` / `_perfect_block` / `_m1_spam` / `_dash_spam` | Stand still / Block / Perfect Block / Attack / Dash | inherits | inherits |
| `ui.toast.emotes_soon` | Emotes are coming soon! | inherits | inherits |
| `ui.toast.key_in_use` | That key is already in use. | inherits | inherits |
| `ui.tip.punch` | Attack 4 times in a row for a combo! | inherits | inherits |
| `ui.tip.block` | Hold block to guard. Time it right for a Perfect Block! | inherits | inherits |
| `ui.tip.dash` | Dash to dodge, or to close the gap. | inherits | inherits |
| `ui.tip.awaken` | Your meter is full. Awaken! | Your meter is full. Go Algebraic! | Your meter is full. Go Algebraic! |
| `ui.tip.wish` | You have a Wish! Make it in the Wish menu. | inherits | inherits |
| `ui.vo.awaken` | Full power! | Algebraic! | Mathematical! |
| `ui.vo.ko` | Down you go! | inherits | inherits |
| `ui.vo.legendary` | Legendary! | inherits | inherits |
| `ui.oosona.species.<id>` | human Human · candy Sweet Folk · dog Dog · cat Cat · vampire Vampire · flame Flame Folk · lumpy Puffball · wizard Wizard · ghost Ghost · penguin Penguin | candy Candy Person · flame Flame Person · lumpy Lumpy (others inherit) | = Codename |
| `ui.notify.wish_epic` | You wished for {styleId}! ({rarityId}) | inherits | inherits |
| `ui.notify.wish_legendary` | {who} wished for {styleId}! | inherits | inherits |
| `ui.notify.wish_mythic_global` | Someone just wished for {styleId}! | inherits | inherits |
| `ui.notify.wish_cosmic_global` | A Cosmic Wish just came true somewhere! | inherits | inherits |
| `ui.notify.level_up` | Level {level}! | inherits | inherits |
| `ui.notify.mastery_up` | {styleId} Mastery {level}! | inherits | inherits |
| `ui.notify.daily_login` | Daily reward collected! | inherits | inherits |
| `ui.notify.streak_3` / `_5` / `_10` | {who} is on a 3-KO streak! / {who} is on a 5-KO streak! / {who} is unstoppable: 10 KOs! | inherits | inherits |
| `ui.notify.streak_ended` | Streak ended! | inherits | inherits |
| `ui.notify.rookie_ended` | Rookie protection has ended. Good luck! | inherits | inherits |
| `ui.notify.shop_bought` | Bought! | inherits | inherits |
| `ui.notify.purchase_granted` | Purchase complete. Thank you! | inherits | inherits |
| `ui.notify.report_thanks` | Thanks. Your report was sent. | inherits | inherits |
| `ui.notify.quest_done` † | Daily quest complete! | inherits | inherits |
| `ui.error.data_load` | We couldn't load your data. Please rejoin. | inherits | inherits |
| `ui.error.session_ended` | Your game was opened somewhere else. | inherits | inherits |
| `ui.error.not_ready` | Still loading. Try again in a moment. | inherits | inherits |
| `ui.error.rate_limited` | Slow down a little! | inherits | inherits |
| `ui.error.restricted` | That isn't available for your account. | inherits | inherits |
| `ui.error.disabled` | That feature is turned off right now. | inherits | inherits |
| `ui.error.insufficient` | You don't have enough for that yet. | inherits | inherits |
| `ui.error.stale` | That changed. Refreshing… | inherits | inherits |
| `ui.error.cap_reached` | You've reached the limit. | inherits | inherits |
| `ui.error.timeout` | That took too long. Try again. | inherits | inherits |
| `ui.error.internal` | Something went wrong. | inherits | inherits |

Notes:
- Token names fixed here for Parts 5 and 6: `ui.hud.protected {sec}`, `ui.shop.refresh {time}`, `ui.training.slot {n}`, `ui.hud.level {level}`, `ui.hud.treasure|shards|sparks {n}`, `ui.quest.<id> {goal}`, `ui.wheel.shards {n}`, `ui.wish.locked_paid {n}`, `ui.shop.balance {n}`, `ui.store.spent_week {robux}`. `ui.wish.fakeout_note` takes no token.
- `ui.quest.<id>` has one key per `Progression.Quests.DailyPool[*].id` (Part 5 §5.5.9: `deal_damage`, `perfect_blocks`, `kos`, `active_minutes`, `awaken`). A task that adds a pool entry adds its Original line in the same task (L02a, registry below).
- Parts 3 and 6 own the `<sub>` IDs of `ui.training.m1_<sub>`, `ender_<sub>` and `dash_<sub>`. Each one gets an Original value in the same task that introduces it. L02a enforces this.
- The shop's merchant lines use the P6 key form `ui.shop.goose_line_<n>`, with n = 1…6. They rhyme in Codename and Canon; the Original lines are plain prose with no travel motif, so the Original merchant does not keep the canon merchant's defining traits (R-IP-09). Part 6 picks n uniformly from 1 to the highest n for which `Skin.has` is true.

#### 8.3.8 Pack file skeletons

```lua
--!strict
-- src/Shared/Skin/Packs/Original.luau — IP-neutral base pack and fallback for every key.
-- Realm: shared. Pure: yes (require-free). Spec §8.3.
local S: { [string]: string } = {}
local function put(t: { [string]: string })
	for k, v in t do
		assert(S[k] == nil, "duplicate skin key " .. k)
		S[k] = v
	end
end

put({
	["style.hero.name"] = "Pell", ["style.hero.epithet"] = "The Seed Knight",
	-- … every row of §8.3.2 (Original columns), §8.3.3, §8.3.4, §8.3.5, §8.3.7 …
	["ui.game.title"] = "Algebraic!",
})

local STYLE_IDS = { "hero", "stretch", "sentry", "granny", "diva", "tyrant", "console", "grappler", "monarch",
	"flame", "butler", "rocker", "frost", "huntress", "decay", "demon", "wish", "chaos" }
local LEVEL_TITLES = { "Scrapper", "Brawler", "Duelist", "Veteran", "Champion", "Vanguard", "Paragon", "Mythmaker",
	"Legend", "Grand Legend" }
for i, title in LEVEL_TITLES do
	S["title.level_" .. tostring(i * 10)] = title
end
for _, id in STYLE_IDS do
	local name = S["style." .. id .. ".name"]
	S["variant." .. id .. ".v1.name"] = "Dusk Variant"
	S["variant." .. id .. ".v2.name"] = "Dawn Variant"
	S["title.mastery_" .. id .. "_5"] = name .. " Adept"
	S["title.master_" .. id] = name .. " Master"
end

local LOOKS = { -- §8.5.6; hex strings, so the pack stays JSON-safe and pure
	hero = { palette = { primary = "#C8553D", secondary = "#8B5A2B", accent = "#E07A2E" } },
	stretch = { palette = { primary = "#7D5A4F", secondary = "#6E4B30", accent = "#7FB069" } },
	monarch = { palette = { primary = "#2F8F8A", secondary = "#B87333", accent = "#F4D35E" } },
	rocker = { palette = { primary = "#5B4B8A", secondary = "#3D5A40", accent = "#E9D985" } },
	frost = { palette = { primary = "#A3B18A", secondary = "#3E5C76", accent = "#E07A5F" } },
	decay = { palette = { primary = "#7A8B3C", secondary = "#5A3E5D", accent = "#D96C3B" } },
}
return { id = "Original", strings = S, looks = LOOKS }
```

```lua
--!strict
-- src/Shared/Skin/Packs/Codename.luau — dev default (owner's approach). Overrides only; identity namespaces complete.
-- Realm: shared. Pure: yes (require-free). Spec §8.3.
local S: { [string]: string } = {}
local function put(t: { [string]: string })
	for k, v in t do
		assert(S[k] == nil, "duplicate skin key " .. k)
		S[k] = v
	end
end
put({
	["style.hero.name"] = "The Last Human", ["style.hero.epithet"] = "Sword Hero",
	-- … every Codename cell of §8.3.2–§8.3.4 and the non-"inherits" Codename cells of §8.3.7 …
})
local STYLE_IDS = { "hero", "stretch", "sentry", "granny", "diva", "tyrant", "console", "grappler", "monarch",
	"flame", "butler", "rocker", "frost", "huntress", "decay", "demon", "wish", "chaos" }
for _, id in STYLE_IDS do
	local name = S["style." .. id .. ".name"]
	S["title.mastery_" .. id .. "_5"] = name .. " Adept"
	S["title.master_" .. id] = name .. " Master"
end
return { id = "Codename", strings = S, looks = {} }
```

`Canon.luau` has the same shape as `Codename.luau`, filled from the Canon columns, with `id = "Canon"` and `looks = {}`. Its header line reads: `-- LOCKED: active only when Config.LicenseConfirmed = true (Spec §8.2.6). Stripped from every public build without a license (§8.4.4).` Without a license, `Skin.resolvePack` never activates it in any context, Studio included, so nobody sees Canon names in the build. A private pitch build uses the Codename pack; canonical names appear only in the pitch deck (GDD §5.3 item 2).

The **stub** written by `--strip` (§8.4.3) is a different file from these skeletons; it replaces a pack's contents temporarily:
```lua
--!strict
-- STRIPPED for public build target=<Pack> commit=<sha7>. Restore: lune run tools/lint_strings.luau --unstrip
return { id = "<Pack>", strings = {}, looks = {} }
```

**Verify:**
- Lune `ip.pack_shapes`: each pack loads through the Part 7 loader, has a valid `id` and no duplicate keys, and every key matches the grammar.
- Lune `ip.original_clean` and `ip.codename_clean`: rules L04 and L13 on pack values.
- Part 7 `skin.pack_key_coverage`.

---

### 8.4 Publish gate

#### 8.4.1 Profiles

| Profile | When | Requirements | Result line |
|---|---|---|---|
| **Dev publish** | H-11 to the private dev universe (`docs/agent/dev-place.json`) | Host gate green, including `lint_strings` in dev mode (all "always" rules). `Verify.verify()` ok. `Defaults.SkinPack ∈ {Codename, Original}`, or Canon with a valid license (L05, L10). The human confirms the universe is **private**. Part 7 R2 is not required. | `[ALG][GATE] publish-dev PASS commit=<sha7>` |
| **Public publish** | Any universe not listed in `Ip.DevUniverseIds`, or making the dev universe public | Everything in §8.4.4 for the target pack, plus Part 7 §7.7.6 R1–R4 | `[ALG][GATE] publish PASS target=<Pack> commit=<sha7> items=<n>/<n>` |

The final verdict combines Part 7's `[ALG][GATE] publish-hook …` with this part's `[ALG][GATE] publish-ip …`. Both must PASS.

#### 8.4.2 Rules data: banned-string lists

**Rules file.** `tools/lint_strings.rules.json` (Part 7's proposed path, adopted). It is never under `src/`, so the lists never ship. Schema **version 2** extends Part 7's v1 additively:

```json
{
  "version": 2,
  "exemptIdentifiers": ["BuildMap_CandyKingdom", "CandyKingdom", "candy_kingdom", "nightosphere",
                        "castle_plaza", "candy_streets", "barracks", "royal_gardens", "dungeon", "earldom_border"],
  "algebraicAllowKeys": {
    "Original": ["ui.game.title"],
    "Codename": ["ui.game.title", "ui.hud.meter_name", "ui.hud.awaken_ready", "ui.hud.combo_big", "ui.vo.awaken", "ui.tip.awaken"]
  },
  "dynamic": { "minWords": 2, "codenameOnlyInto": ["Original", "meta", "tree"], "canonOnlyInto": ["Original", "Codename", "src", "meta", "prompt", "tree"] },
  "terms": [
    { "term": "adventure time", "list": "A", "packs": ["Original", "Codename"], "scopes": ["pack", "src", "meta", "prompt", "tree"], "mode": "always" },
    { "term": "lsp", "list": "A", "packs": ["Original", "Codename"], "scopes": ["pack", "meta", "prompt", "tree"], "mode": "always", "note": "not src: collides with luau-lsp" },
    { "term": "mathematical", "list": "B", "packs": ["Original", "Codename"], "scopes": ["pack", "meta", "prompt", "tree"], "mode": "always" },
    { "term": "UNACCEPTABLE", "list": "B", "packs": ["Original", "Codename"], "scopes": ["pack", "meta", "prompt", "tree"], "mode": "always", "caseSensitive": true },
    { "term": "fin", "list": "H", "packs": ["Original", "Codename"], "scopes": ["pack"], "mode": "always", "keyPattern": "^style%." },
    { "term": "spin", "list": "D", "packs": ["Original", "Codename", "Canon"], "scopes": ["pack", "meta"], "mode": "always", "keyPattern": "^ui%.wish%.|^ui%.store%.|^product%.|^pass%." }
  ]
}
```

The JSON shows examples. The **Lists** table below is authoritative for every term's `packs` and `scopes`; `ip.rules_schema` checks that each term's `packs` and `scopes` equal its list's row (the per-list values are embedded in the test, no terms), except that a term with a `note` may drop the `src` scope. `keyPattern` applies to the `pack` scope only.

**Field semantics** (Part 7's mechanism implements them):
- `term`: a lowercase phrase, unless `caseSensitive`.
- `packs`: the packs whose **values** the term is checked against.
- `scopes`: which texts are scanned.
  - `pack`: pack values.
  - `src`: files under `src/**`, `tests/**` and `tools/**`, minus the exemptions in §8.2.5.
  - `meta`: `PUBLISH_METADATA.md`.
  - `prompt`: the prompt and query columns of `ASSET_PROVENANCE.md`, and `--text … --scope prompt`.
  - `tree`: the Studio dump (§8.4.6).
- `mode`: `always`, or `public` (checked only under `--public`).
- `keyPattern`: optional; restricts the term to pack keys that match this Lua pattern. Alternatives are written with `|`; the mechanism splits on `|`.
- **Canon** values are checked only against list D. Every other list is skipped for Canon. The `meta` and `tree` scopes are relaxed for a Canon target only as far as LICENSE_CONFIRMATION.md grants (§8.4.4).

**Matching.** Both haystack and term are normalized:
- CamelCase is split (`(%l)(%u)` becomes `%1 %2`);
- apostrophes (`'` and `’`) are removed, so "Founders' Island" becomes "founders island" and "Finn's" becomes "finns";
- `_ - . / \` become spaces;
- text is lowercased (except for `caseSensitive` terms);
- whitespace runs collapse to one space.

A term matches between **frontier patterns** (`%f[%w]` before it if it starts with an alphanumeric, `%f[%W]` after it if it ends with one). A list A or C term also matches when followed by `s` or `es` (plurals, and possessives after apostrophe removal): the matcher tries `term`, `term .. "s"` and `term .. "es"` with the same frontiers. Lists B, D, E and H match exactly, so "globes" or "spines" do not hit "glob" or "spin". So `IceKingBoss` hits "ice king", "Banana Guards" hits "banana guard", "Gumball Guardians" hits "gumball guardian" and "Gunters" hits "gunter", while "Finnish", "lichen" and "Jakeson" do not hit.

**Lists:**

| List | Meaning | Terms (normalized) | Packs | Scopes |
|---|---|---|---|---|
| **A** | Canonical proper nouns, brands, creators | adventure time, cartoon network, ooo, finn, jake, bmo, marceline, princess bubblegum, bonnibel, ice king, simon petrikov, simon, gunter, gunther, lich, prismo, golb, lemongrab, peppermint butler, banana guard, tree trunks, lumpy space, susan strong, flame princess, huntress wizard, hunson, abadeer, choose goose, gumball guardian, rattleballs, goliad, rainicorn, fionna, tree fort, candy kingdom, ice kingdom, fire kingdom, nightosphere, time room, gumbaldia, mushroom war, card wars, cosmic owl, patience st pim, neptr, hambo, grob gob glob grod, quartzion, distant lands, skydance, paramount, nickelodeon, warner bros, frederator, pendleton ward, rebecca sugar, ghostshrimp, betty, kara, golbetty, gum war, founders island, founders fury, incendium, winter king, ice thing, sweet p, magic man, ricardio, farmworld, fern, marshall lee, prince gumball, flame prince; and, outside `src` only: lsp, marcy | Original, Codename | all |
| **B** | Catchphrases, show-invented slang and quoted lines (line test T4, §8.5.3) | mathematical, algebraic *(allowed only in the `algebraicAllowKeys` keys, in code, and in the metadata `experience_title`, which gets warning W-TITLE)*, math!, what time is it, oh my glob, glob, glob it, shmowzow, schmowzow, lumping, lumpin, what the lump, oh my lump, everything burrito, bacon pancakes, one million years dungeon, ate your fries, maloso vobiscum, UNACCEPTABLE (case-sensitive) | Original, Codename | pack, meta, prompt, tree |
| **C** | Canon-quoting descriptors: the Codename vocabulary. Allowed in Codename, forbidden in Original and metadata. | Static: last human, magic dog, vampire queen, vampire king, candy monarch, crowned hermit, wishing eye, well of power, crystal dimension, starchild, candy capital, candy person, candy people, bubblegum, lumpy, hyooman, earldom, royal banana, sour earl, living console, crystal granny, dark butler, flame queen, forest huntress, undying end, lord of evil, wishmaster, discord incarnate, rhyming goose. Dynamic: see L13. | Original | pack, meta, prompt, tree |
| **H** | Near-homophones of canon names (§8.3.1 Original rules; line test T4, §8.5.3) | fin, jayk, jak, beemo, bemo, marcelyn, marcy, bonnie, bonny, prizmo, golbe, lemonhead, gunthar | Original, Codename | pack (`keyPattern` `^style%.`) |
| **D** | Gambling vocabulary (GDD §8.8; Part 6 `ui.wish_no_gambling_terms`) | slot, slots, spin, spins, jackpot, roulette, casino, bet, bets, gamble, gambling | all three | pack (`keyPattern` `^ui%.wish%.\|^ui%.store%.\|^product%.\|^pass%.`), meta |
| **E** | Signature-design descriptors (palette + prop test T2, §8.5.3) | bear hat, bear ear, bear ears, bear eared, axe bass, bass axe, yellow dog, stretchy dog, magic dog, vampire girl, candy princess, pink princess, crown and beard, blue skinned wizard, gumball, living game console, boy and his dog | — | prompt, meta |

**The "Algebraic" / "Mathematical" stylization caveat** (§8.7). Both are show catchphrases.
- "Mathematical" is banned in Original and Codename.
- "Algebraic" is the working title, and it is allowed only:
  - in code and internal names (project codename);
  - in `ui.game.title` (Original) and the Codename allow-keys;
  - in the metadata `experience_title`, which raises warning **W-TITLE** until counsel clears the title (§8.7).
- In Original builds the word appears only as the title ("Algebraic!" in `ui.game.title` and the metadata `experience_title`). It is never used as a bark, combo callout, achievement, button label or any other exclamation.
- It is never stylized like the show logo (font, colour, layout) and never paired with "Mathematical", "Adventure" or "Time".

Counsel may add terms; the agent adds them only on a human instruction recorded in `PROGRESS.md`.

#### 8.4.3 `tools/lint_strings.luau` rules, flags and output

Part 7 owns the mechanism and rules L01–L06. This part supplies the data and the rule semantics for L02 and L04, and adds rules L07–L22 to the same tool.

**Flags:**
- Part 7: `--public`, `--pack <P>` (default `Defaults.SkinPack`), `--rules <path>`.
- Part 8 additions:
  - `--tree <json>`: post-strip phase.
  - `--meta <path>`: default `docs/agent/PUBLISH_METADATA.md`.
  - `--strip <P>` and `--unstrip`.
  - `--text "<s>" --scope <src|prompt|meta>`: ad-hoc check of commit messages, prompts and search queries.
  - `--dry-run`: used only by the agent procedure `ip.gate_dry_run` (§8.4.7). Three findings are reported as the warning `W-DRYRUN` instead of errors: L11 missing or placeholder metadata, L17 missing or `pending` recognizability rows, and L22 exempt identifiers in the tree. All other rules are unchanged; a term hit in a real metadata field, or a `fail` row, stays an error. A run that printed any `W-DRYRUN` is never valid for H-11.

**Output.** Lines follow Part 7: `[ALG][LINT] <path>:<line> <rule> <detail>`, and the summary `[ALG][LINT] lint_strings SUMMARY errors=<n> warnings=<m>`. The `--public --tree` run also prints `[ALG][GATE] publish-ip PASS|FAIL target=<P> commit=<sha7> errors=<n> warnings=<m>` (`warnings` counts W-TITLE, W-EXEMPT-ID and W-DRYRUN). Exit codes follow Part 7 (0 clean, 1 errors, 2 usage).

**Strip stub.** A pack file is a strip stub when its second line starts with `-- STRIPPED for public build` and its `strings` table is empty (the file `--strip` writes, §8.3.8). L02 skips stubs; L14 checks that exactly the required packs are stubs.

| Rule | Mode | Check |
|---|---|---|
| L02 (data) | always | (a) Every referenced key exists in Original. (b) Identity namespaces (§8.2.3) are complete in every pack that is not a strip stub. (c) Override token parity with Original. Families come from the registry below. |
| L04 (data) | always / `--public` | Terms per §8.4.2. Pack values are checked per `term.packs` for **every** listed pack, Codename included. Before `src` and `tree` scanning (and before L13 runs on them), `exemptIdentifiers` are replaced by `<id>`, and the warning `W-EXEMPT-ID` is printed once. |
| L07 | always | List D on `ui.wish.*`, `ui.store.*`, `product.*` and `pass.*` values in all packs |
| L08 | always | `product.*`, `pass.*` and `badge.*` keys are defined **only** in Original |
| L09 | always if the flag is true; required under `--public --pack Codename` | `Defaults.PublicCodenameAcknowledged == true` ⇔ `docs/agent/PUBLISH_ACK.md` is valid (§8.4.5). The validity date is checked against today. `Defaults.Ip.PublicAckExpiresUnix` equals the Unix time of `valid_until` 23:59:59 UTC. M5 is checked whenever L20 raises W-TITLE; S4 is checked whenever L22 finds exempt identifiers. Owner provenance of the file and of the flag lines (§8.4.5). |
| L10 | always if the flag is true; required under `--public --pack Canon` | `Defaults.LicenseConfirmed == true` ⇔ `docs/agent/LICENSE_CONFIRMATION.md` is valid, and `Defaults.Ip.LicenseRef` equals its `license_ref`. Owner provenance of the file and of the flag lines (§8.4.5). |
| L11 | `--public` | PUBLISH_METADATA.md exists and parses, with no empty field and no `<…>` placeholder value. `target_pack == --pack`. `target_universe_id ∉ Ip.DevUniverseIds`. Every text field passes the `meta` scope (lists A, B, C, D, E); a Canon target is relaxed per its license. Under `--dry-run`, a missing file and empty or placeholder values are `W-DRYRUN`. |
| L12 | always | Every prompt and query cell in `docs/agent/ASSET_PROVENANCE.md` passes the `prompt` scope |
| L13 | always | **Cross-pack leakage.** Codename-only phrases are Codename `style.*`, `move.*` and `map.*` values with ≥ 2 words that are absent from Original values; they must not appear in Original values, metadata or the tree. Canon-only phrases (the same rule against Original ∪ Codename) must not appear in Original, Codename, `src`, metadata, prompts or the tree. With `--tree`, the lists are computed from `git show HEAD:` copies, because the files on disk are stubs. |
| L14 | `--public --tree` | Strip state: exactly the required packs (§8.4.4) are strip stubs, and no other pack is |
| L15 | `--public --tree` | Every string in the Verify dump passes the `tree` scope for the target, and the dump's `problems` list is empty |
| L16 | always | `Defaults.Ip.DevUniverseIds` ⊆ { `dev-place.json.universeId` }, and `Defaults.Ip.DevGroupId == dev-place.json.groupId` |
| L17 | `--public` (non-Canon) | Every required recognizability subject has a `pass` row in `RECOGNIZABILITY.md` (§8.5.3). Under `--dry-run`, missing or `pending` rows are `W-DRYRUN`; a `fail` row stays an error. |
| L18 | `--public` | `Defaults.SkinPack == --pack` |
| L19 | `--public` | `Defaults.Perf.Audio.TtsInStudioOnly == true` (Part 6) |
| L20 | `--public` | Warning `W-TITLE` if the title contains "algebraic" and `TITLE_CLEARANCE.md` is missing or has `status ≠ cleared` |
| L21 | `--public --pack Codename` | If PUBLISH_ACK.md checks `MON-A`, then `Defaults.Features.PaidRandomItems == false` |
| L22 | `--public --tree`, non-Canon targets | Exempt identifiers (§8.2.5) found in the tree dump (instance names and string attributes ship to every client). Original target: an error per identifier. Codename target: an error unless PUBLISH_ACK.md checks S4. Under `--dry-run`: `W-DRYRUN`. The rule clears once the §8.9 items 1 and 10 renames land. |

**Key-family registry** (inside the lint; parts register new dynamic families in the same task):

| Family | Expansion |
|---|---|
| `style.<id>.name`, `style.<id>.epithet` | 18 style IDs |
| `move.<id>.<slot>.name` | implemented style IDs (from `Styles`, through the Part 7 loader) × `m1, passive, q, 1, 2, 3, 4, awaken, a1, a2, a3, a4` |
| `map.<mapId>.name`, `map.candy_kingdom.zone.<zoneId>` | 3 maps; 6 zones (Part 6 §6.3.1) |
| `flair.<id>.name`, `rarity.<id>.name` | §0.4 IDs |
| `variant.<id>.v1.name`, `variant.<id>.v2.name` | 18 IDs |
| `title.level_<n>`, `title.mastery_<id>_5`, `title.master_<id>` | n = 10…100; 18 IDs |
| `product.<key>.name`, `pass.<key>.name` | Part 5 `ProductKey`, `PassKey` |
| `ui.error.<ErrCode>` | Part 2 list |
| `ui.notify.wish_<rarity>_global`, `ui.notify.streak_<tier>` | rarity ∈ {mythic, cosmic}; tier ∈ {3, 5, 10} |
| `ui.training.mode_<mode>` | idle, block, perfect_block, m1_spam, dash_spam |
| `ui.oosona.species.<id>` | `Appearance.OosonaSpecies[*].id` |
| `ui.shop.goose_line_<n>` | n = 1…6 |
| `ui.quest.<id>` | `Progression.Quests.DailyPool[*].id` (Part 5 §5.5.9) |
| `ui.report.reason_<reason>` | exploit, teaming, harassment, other (Part 6 §6.9.12) |

Any string literal in `src/` that ends with `"."` and is followed by `..` must match a registered family prefix; otherwise the error is `L02 unregistered-family`.

**`--strip <P>` / `--unstrip`:**

```lua
--!strict
-- tools/lint_strings.luau — Part 8 strip/unstrip (Lune). Spec §8.4.3.
local process = require("@lune/process")
local SafeWrite = require("./lib/SafeWrite") -- §8.4.5: the only file writer for host tools

local PACK_DIR = "src/Shared/Skin/Packs"
local STRIP: { [string]: { string } } = {
	Original = { "Codename", "Canon" },
	Codename = { "Canon" },
	Canon = {},
}

local function git(args: { string }): (boolean, string)
	local r = process.exec("git", args)
	return r.ok, r.stdout
end

local function strip(target: string): number
	local list = STRIP[target]
	if list == nil then
		print("[ALG][LINT] strip usage: --strip Original|Codename|Canon")
		return 2
	end
	local clean = git({ "diff", "--quiet", "HEAD", "--", PACK_DIR })
	if not clean then
		print(`[ALG][LINT] {PACK_DIR} strip refused: packs differ from HEAD (commit or stash first)`)
		return 1
	end
	local _, sha = git({ "rev-parse", "--short=7", "HEAD" })
	sha = (sha:gsub("%s+", ""))
	for _, id in list do
		local stub = string.format(
			'--!strict\n-- STRIPPED for public build target=%s commit=%s. Restore: lune run tools/lint_strings.luau --unstrip\n'
				.. 'return { id = "%s", strings = {}, looks = {} }\n',
			target,
			sha,
			id
		)
		SafeWrite.writeFile(`{PACK_DIR}/{id}.luau`, stub)
		print(`[ALG][LINT] {PACK_DIR}/{id}.luau stripped target={target}`)
	end
	return 0
end

local function unstrip(): number
	for _, id in { "Codename", "Canon" } do
		local ok, src = git({ "show", `HEAD:{PACK_DIR}/{id}.luau` })
		if not ok then
			print(`[ALG][LINT] {PACK_DIR}/{id}.luau unstrip failed`)
			return 1
		end
		SafeWrite.writeFile(`{PACK_DIR}/{id}.luau`, src)
	end
	print("[ALG][LINT] packs restored from HEAD")
	return 0
end
```

- Both commands use only allowlisted Bash (`lune run`); git is called from inside Lune.
- `process.exec` follows Part 1 §1.4.2.
- Files are written through `tools/lib/SafeWrite.luau` (§8.4.5), which refuses the human-signed files.
- Script Sync pushes the rewritten files into Studio. The agent then runs the sync gate (Part 1 §1.4.3) before the Studio step.

#### 8.4.4 Requirements per target pack

| Requirement | Original | Codename | Canon |
|---|---|---|---|
| `Defaults.SkinPack == target` (L18) | ✔ | ✔ | ✔ |
| Host gate green in dev mode (all "always" rules) | ✔ | ✔ | ✔ |
| Original pack clean (A, B, C, H, L13) | ✔ | ✔ (it is the fallback) | ✔ |
| Codename pack | **stripped** | clean (A, B, H, L13) | kept (licensed) |
| Canon pack | **stripped** | **stripped** | present; L02b complete |
| Flags | none | `PublicCodenameAcknowledged = true` and `Ip.PublicAckExpiresUnix` **set by the owner** + valid, owner-committed `PUBLISH_ACK.md` (L09) | `LicenseConfirmed = true` + `Ip.LicenseRef` + valid, owner-committed `LICENSE_CONFIRMATION.md` (L10), all set by the owner |
| Exempt identifiers in the tree (L22) | **error** until the §8.9 renames land | allowed only with PUBLISH_ACK box S4 | not checked |
| Owning group of the target universe (HP-01, §8.2.6 Groups) | production group | a separate group that is neither the dev group nor the production group (`target_group_id` in PUBLISH_ACK), so removals and strikes never land on the production group | production group |
| Look set of `ReplicatedStorage.Looks` `Gear_*` / `Summon_*` (§8.5.6) | `Original` | `Codename` | `Codename` until licensed art exists |
| Recognizability rows, `pass` required (L17) | `style:<id>` for every implemented style, `pair:hero+stretch`, `map:<mapId>` for every built map, `meta:icon`, `meta:thumbnail_<n>` | `meta:icon`, `meta:thumbnail_<n>` | licensor approvals instead (GDD §5.4) |
| Asset provenance complete; no ripped or fan-made assets (Part 7 M03 + M09–M12) | ✔ | ✔ | ✔ |
| Metadata lint (L11) | lists A, B, C, D, E; W-TITLE | lists A, B, C, D, E; W-TITLE | D; marks only as far as the license grants |
| Studio tree scan (L15) | A, B, C, L13 | A, B, L13 | D |
| Paid random items | allowed | per PUBLISH_ACK monetization choice (L21) | per license |
| TTS Studio-only (L19) | ✔ | ✔ | ✔ |
| Part 7 R1–R4 | ✔ | ✔ | ✔ |
| Human checklist HP-01…HP-08 (§8.4.7) | ✔ | ✔ | ✔ |

#### 8.4.5 Human-signed files

The templates live in `docs/agent/templates/`: `PUBLISH_ACK.template.md`, `LICENSE_CONFIRMATION.template.md`, `TITLE_CLEARANCE.template.md`, `PUBLISH_METADATA.template.md`, `RECOGNIZABILITY.template.md` and `TRIGGER_WATCH.template.md`. The agent writes these. The human copies a signed-file template (the first three) to `docs/agent/` and fills it in. The agent creates `RECOGNIZABILITY.md` and `TRIGGER_WATCH.md` from their templates and may draft `PUBLISH_METADATA.md` (§8.5.3, §8.6.1).

**Who can write the signed files.** The example permissions file (`docs/agent/claude-settings.example.json`, Part 1 §1.2.4) already denies Edit/Write on these files (these six entries of its `permissions.deny` array):

```json
[
  "Edit(docs/agent/PUBLISH_ACK.md)", "Write(docs/agent/PUBLISH_ACK.md)",
  "Edit(docs/agent/LICENSE_CONFIRMATION.md)", "Write(docs/agent/LICENSE_CONFIRMATION.md)",
  "Edit(docs/agent/TITLE_CLEARANCE.md)", "Write(docs/agent/TITLE_CLEARANCE.md)"
]
```

The deny entries do not cover `lune run`, which can write any file. They are a guard against accidents, not a mechanism the agent cannot bypass. **The binding control is HP-08 plus the provenance check below.** As a second guard, every Lune host tool writes files only through one helper that refuses the signed paths:

```lua
--!strict
-- tools/lib/SafeWrite.luau (Lune) — the only file writer for host tools. Spec §8.4.5.
-- Refuses the human-signed files by base name, whatever the directory or case. A guard, not a boundary (HP-08 is).
local fs = require("@lune/fs")

local SIGNED: { [string]: boolean } = { publish_ack = true, license_confirmation = true, title_clearance = true }

local SafeWrite = {}

function SafeWrite.isSigned(path: string): boolean
	local unix = (string.gsub(path, "\\", "/"))
	local base = string.lower(string.match(unix, "([^/]*)$") or "")
	local stem = string.match(base, "^(.-)%.md$")
	return stem ~= nil and SIGNED[stem] == true
end

function SafeWrite.writeFile(path: string, contents: string)
	if SafeWrite.isSigned(path) then
		error(`[ALG][LINT] {path} write refused: human-signed file (Spec §8.4.5)`)
	end
	fs.writeFile(path, contents)
end

return SafeWrite
```

Lune test `ip.no_tool_writes_signed_files`: (a) no Lune script under `tools/**` (excluding the Studio snippets `tools/studio/**` and the plugin source `tools/plugin/AlgebraicTestHarness/**`) calls `fs.writeFile`, `fs.copy` or `fs.move` except `tools/lib/SafeWrite.luau`; (b) `SafeWrite.writeFile` raises for `docs/agent/PUBLISH_ACK.md`, `./docs/agent/publish_ack.md`, `docs\agent\LICENSE_CONFIRMATION.md` and `x/../TITLE_CLEARANCE.md`, and writes `docs/agent/templates/PUBLISH_ACK.template.md` in a temp directory.

**Provenance check (L09 and L10).** It runs whenever the rule runs, and a failure is an error `[ALG][LINT] <path> L09|L10 provenance <detail>`; the agent then opens a `safety` blocker (R-IP-04).
1. The signed file is committed and unchanged: `git status --porcelain -- <file>` prints nothing.
2. `git log -1 --format=%ae%n%B -- <file>`: the author email equals `ownerGitEmail` in `docs/agent/dev-place.json` (recorded at H-01 and confirmed by the human; case-insensitive), and no message line starts with `Co-Authored-By: Claude` (case-insensitive).
3. The same two checks cover the `src/Shared/Config/Defaults.luau` lines the rule depends on (L09: `PublicCodenameAcknowledged`, `PublicAckExpiresUnix`; L10: `LicenseConfirmed`, `LicenseRef`). For each, the newest commit that touched the line is read with `git log -1 -L '/<Name> *=/,+1:src/Shared/Config/Defaults.luau' --format=%ae%n%B` (diff output ignored), and `git diff --quiet HEAD -- src/Shared/Config/Defaults.luau` must succeed.

Lune test `ip.signed_file_provenance` (temporary git repository fixture): an owner-authored commit passes; an agent-authored commit, a commit with the `Co-Authored-By: Claude` trailer, an uncommitted signed file and an agent-authored flag line each give the provenance error.

**Parse rules** (all files):
- `key: value` lines (`^([a-z_0-9]+):%s*(.*)$`).
- `|` starts a block whose lines are indented by two spaces.
- Checkboxes are `- [x] <ID> <text>` or `- [ ] <ID> <text>`.
- Dates are `YYYY-MM-DD`.
- An unknown key is an error, so typos fail loudly.

**`docs/agent/PUBLISH_ACK.md`** (required for a public Codename build; template `PUBLISH_ACK.template.md`):

```markdown
# PUBLISH_ACK — public Codename build (owner's informed consent)
pack: Codename
target_universe_id: <number>
target_group_id: <number; neither the dev group nor the production group, §8.4.4>
signed_by: <full name>
roblox_account: <username>
role: <Owner | delegate with authority>
date: <YYYY-MM-DD>
valid_until: <YYYY-MM-DD, at most 90 days after date; also set Ip.PublicAckExpiresUnix>
counsel_consulted: <yes | no>
counsel_ref: <required when counsel_consulted is yes>

## Risk acknowledgement (all required)
- [ ] A1 I read Build Spec §8.1.3 and GDD §5.7. Renaming reduces, but does not remove, trademark and keyword-match exposure (this build still shows canon epithets and 'Algebraic!'), and it does nothing for copyright in recognizable characters, places and scenes.
- [ ] A2 License Manager matches images as well as keywords. A rights holder can remove the game (DMCA, or Rights Manager up to 250 links), or send a license offer that auto-activates after 14 days at a revenue share it sets (0–95 %).
- [ ] A3 Players can lose Robux-backed progress, and the publishing group's standing can suffer under the repeat-infringer policy.
- [ ] A4 Judgement estimates, not statistics (GDD §5.7; they assume paid Wishes): at least one adverse rights-holder action within 12 months of public launch is estimated at 3–10 % for a small game, 20–40 % for a breakout (~5k–50k CCU) and 50–75 % for a breakout after a trigger event.
## Metadata (M1–M4 required; M5 required when L20 raises W-TITLE)
- [ ] M1 Description, tags, update names and product/pass/badge names contain no AT words (PUBLISH_METADATA.md passes lint L11). The title contains no AT words except 'Algebraic'.
- [ ] M2 The icon and thumbnails show no AT character, likeness, palette-plus-prop combination, logo, logo font or show art, and have recognizability rows.
- [ ] M3 Group name and description, Discord server name and social handles contain no AT words.
- [ ] M4 No ads, sponsored listings or influencer briefs reference AT.
- [ ] M5 The title 'Algebraic!' is itself an AT catchphrase and is not cleared (TITLE_CLEARANCE.md status: <cleared | not-cleared | pending | missing>). Next to Codename content it adds evidence of intent. I accept warning W-TITLE.
## Assets (S1–S3 required; S4 required when L22 finds exempt identifiers)
- [ ] S1 No ripped or official art, audio, fonts or logos, and no fan-made AT Creator Store assets. Provenance is complete.
- [ ] S2 No AI-cloned or sound-alike voices. All music is original.
- [ ] S3 The Canon pack is stripped from this build.
- [ ] S4 I accept that internal IDs `candy_kingdom`/`nightosphere` (and the other exempt identifiers of Build Spec §8.2.5) ship in this build as instance names and attributes that clients can read.
## Owner options (optional; check each one chosen; the agent applies it on the release branch before §8.4.7 step 2)
- [ ] OPT-1 Avatar-first public default: `CharacterLook = false`, `OwnLook = "Avatar"` (as in JJS).
- [ ] OPT-2 Drop the 'Algebraic!' callouts from Codename: its overrides of `ui.hud.meter_name`, `ui.hud.awaken_ready`, `ui.hud.combo_big`, `ui.tip.awaken` and `ui.vo.awaken` are removed, so the Original values show.
- [ ] OPT-3 Replace the Q-P8-4 names in Codename with new codename-safe names (Build Spec §8.10).
## Monetization (exactly one of MON-A / MON-B; MON-C required)
- [ ] MON-A Paid random items (Robux Wish packs, Lucky Jar) are OFF for this build (recommended; sets Features.PaidRandomItems = false).
- [ ] MON-B Paid random items are ON. I accept that paid rolls on kids' characters are a top enforcement trigger.
- [ ] MON-C Product and pass names are the Original strings.
## Operations (all required)
- [ ] O1 Trigger-watch owner: <name>; weekly cadence (§8.6).
- [ ] O2 I read the response playbook (§8.6). We will not counter-notice AT content.
- [ ] O3 An Original build of this commit passes the gate and is ready as the switch target.
- [ ] O4 No §8.6 trigger has occurred since `date`.
```

**Validation (L09):**
- every required box is checked (M5 whenever L20 raises W-TITLE; S4 whenever L22 finds exempt identifiers in the `--tree` run);
- exactly one of MON-A / MON-B is checked;
- every field except `counsel_ref` is non-empty; `counsel_ref` is required when `counsel_consulted: yes`;
- `date ≤ today ≤ valid_until ≤ date + 90 days` (validity period is a tuning default);
- `Defaults.Ip.PublicAckExpiresUnix` equals the Unix time of `valid_until` 23:59:59 UTC, so the runtime guard downgrades to Original when the acknowledgement lapses (§8.2.6);
- `target_universe_id ∉ Ip.DevUniverseIds`, and `target_group_id ≠ dev-place.json.groupId`;
- the provenance check above passes.

**`docs/agent/LICENSE_CONFIRMATION.md`** (Canon):
- **Fields:** `pack: Canon`, `license_type: <LicenseManager | Direct>`, `license_ref`, `licensor` (legal entity), `effective_date`, `expiry_date` (a date or `perpetual`), `territory`, `revenue_share_pct`, `content_standards_ref`, `approved_assets_ref`, `signed_by`, `date`.
- **Scope fields:** `scope_names`, `scope_likeness`, `scope_vo`, `scope_music`, `scope_marks_in_metadata`, each `<yes | no>`.
- **Boxes:** L1 counsel reviewed the agreement. L2 a licensor approval pipeline is in place (GDD §5.4). L3 metadata follows the licensor's brand guidelines. L4 the GDD Variants and VO in this build are inside the license scope.
- **Lint L10:**
  - all fields are present;
  - `effective_date ≤ today`;
  - `expiry_date` is `perpetual` or later than today;
  - `license_ref == Defaults.Ip.LicenseRef`;
  - with `scope_marks_in_metadata: no`, list A applies to the metadata;
  - with `yes`, list A hits are warnings;
  - the provenance check above passes.

**`docs/agent/TITLE_CLEARANCE.md`:** fields `title`, `status` (`cleared | not-cleared | pending`), `counsel`, `date`, `classes` (for example `9, 28, 41`), and an optional `notes` block.

**`docs/agent/PUBLISH_METADATA.md`** (every public publish; the agent may draft it, and the human owns the final text):
- `target_pack`, `target_universe_id`.
- `experience_title`, `experience_description` (block), `genre`, `tags`.
- `update_name`.
- `icon_description`, and `thumbnail_<n>_description` for every thumbnail.
- `group_name`, `group_description` (block), `discord_server_name`, `social_handles`.
- `ad_copy` (block, or `none`).
- `product.<key>.name` and `product.<key>.description` for every live product; the same for `pass.<key>.*` and `badge.<key>.*`.

#### 8.4.6 Studio step: `Verify.publishGate`

This is an addition to Part 1's `Verify.luau`. It is invoked through `run_builder.luau` with `{"builder":"Verify","fn":"publishGate","opts":{"target":"Original"}}`. Part 1's snippet gains one branch:

```lua
elseif A.fn == "publishGate" then
	return mod.publishGate(A.opts or {})
```

```lua
--!strict
-- src/DevTools/Builders/Verify.luau — Part 8 addition. Edit only. Spec §8.4.6.
-- Returns the Studio half of the IP gate. Banned-term matching happens on the host (L15), so no IP lists live in the place.
type IpString = { path: string, kind: string, value: string }
type IpGateResult = { ok: boolean, problems: { string }, target: string, packs: { [string]: number }, strings: { IpString } }

local SCAN = { "Workspace", "ReplicatedStorage", "ReplicatedFirst", "StarterGui", "StarterPack", "StarterPlayer",
	"Lighting", "SoundService", "Teams" }
local MAX_STRINGS = 4000 -- (tuning default) keeps the MCP response small; see spike S-P8-3

local function at(root: Instance, path: { string }): Instance
	local node: Instance = root
	for _, name in path do
		local nextNode = node:FindFirstChild(name)
		assert(nextNode ~= nil, "missing " .. root:GetFullName() .. "." .. table.concat(path, "."))
		node = nextNode
	end
	return node
end

local function fresh(m: Instance): any
	local c = m:Clone() :: ModuleScript
	local ok, res = pcall(require, c)
	c:Destroy()
	if not ok then
		error(res)
	end
	return res
end

-- `M` is Verify.luau's module table (Part 1 §1.6.7).
function M.publishGate(opts: { target: string? }): IpGateResult
	local target = opts.target or "Original"
	local problems: { string } = {}
	local RS = game:GetService("ReplicatedStorage")
	local defaults = fresh(at(RS, { "Shared", "Config", "Defaults" }))
	if defaults.SkinPack ~= target then
		table.insert(problems, `[ip] Defaults.SkinPack={defaults.SkinPack} want={target}`)
	end
	-- Pack strip state (L14 re-checks on the host)
	local packs: { [string]: number } = {}
	for _, id in { "Original", "Codename", "Canon" } do
		local p = fresh(at(RS, { "Shared", "Skin", "Packs", id }))
		local n = 0
		for _ in p.strings do
			n += 1
		end
		packs[id] = n
	end
	local mustStrip: { string } = if target == "Original"
		then { "Codename", "Canon" }
		elseif target == "Codename" then { "Canon" }
		else {}
	for _, id in mustStrip do
		if packs[id] ~= 0 then
			table.insert(problems, `[ip] pack {id} not stripped ({packs[id]} keys)`)
		end
	end
	-- Look set of gear and summon templates (§8.5.6)
	local want = if target == "Original" then "Original" else "Codename"
	local looks = RS:FindFirstChild("Looks")
	if looks ~= nil then
		for _, c in looks:GetChildren() do
			if string.match(c.Name, "^Gear_") or string.match(c.Name, "^Summon_") then
				local got = c:GetAttribute("LookSet")
				if got ~= want then
					table.insert(problems, `[ip] look {c.Name} LookSet={tostring(got)} want={want}`)
				end
			end
		end
	else
		table.insert(problems, "[ip] ReplicatedStorage.Looks missing (run BuildTemplates)")
	end
	-- Player-reachable strings (deduplicated by kind and value)
	local strings: { IpString } = {}
	local seen: { [string]: boolean } = {}
	local function add(path: string, kind: string, value: string)
		local k = kind .. "\0" .. value
		if value ~= "" and not seen[k] and #strings < MAX_STRINGS then
			seen[k] = true
			table.insert(strings, { path = path, kind = kind, value = value })
		end
	end
	for _, serviceName in SCAN do
		for _, d in game:GetService(serviceName):GetDescendants() do
			local path = d:GetFullName()
			add(path, "name", d.Name)
			for an, av in d:GetAttributes() do
				if type(av) == "string" then
					add(path .. "@" .. an, "attr", av)
				end
			end
			if d:IsA("TextLabel") or d:IsA("TextButton") or d:IsA("TextBox") then
				add(path, "text", (d :: any).Text)
				if d:IsA("TextBox") then
					add(path, "text", (d :: TextBox).PlaceholderText)
				end
			elseif d:IsA("ProximityPrompt") then
				add(path, "text", (d :: ProximityPrompt).ActionText)
				add(path, "text", (d :: ProximityPrompt).ObjectText)
			elseif d:IsA("StringValue") then
				add(path, "value", (d :: StringValue).Value)
			elseif d:IsA("Tool") then
				add(path, "text", (d :: Tool).ToolTip)
			elseif d:IsA("LocalizationTable") then
				for _, e in (d :: LocalizationTable):GetEntries() do
					for _, v in e.Values do
						add(path, "loc", v)
					end
				end
			end
		end
	end
	if #strings >= MAX_STRINGS then
		table.insert(problems, "[ip] string dump truncated; raise MAX_STRINGS or split by service")
	end
	return { ok = #problems == 0, problems = problems, target = target, packs = packs, strings = strings }
end
```

Members used, all verified with read security None in the docs:
- `Instance:GetDescendants`, `GetAttributes`, `GetFullName`, `IsA`, `Clone`, `Destroy`;
- `TextLabel.Text`, `TextButton.Text`, `TextBox.Text`, `TextBox.PlaceholderText`;
- `ProximityPrompt.ActionText`, `ProximityPrompt.ObjectText`;
- `StringValue.Value`, `Tool.ToolTip`, `LocalizationTable:GetEntries`.

Script sources are excluded, because the host lint covers them. The agent writes the returned `res` JSON to `build/ip_tree.json` (gitignored) with its file tool.

#### 8.4.7 Publish procedure

**Dev publish (H-11 to the dev universe).**
1. Host gate green.
2. `run_builder {"builder":"Verify","fn":"verify"}` returns ok.
3. The agent prints `[ALG][GATE] publish-dev PASS commit=<sha7>` and posts a `human-step` blocker for H-11.

**Public publish:**
1. **Owner decision.** The owner names the target pack and the target universe in `PROGRESS.md`. The human creates or updates:
   - `PUBLISH_METADATA.md`, always;
   - `PUBLISH_ACK.md` plus `PublicCodenameAcknowledged = true` and `Ip.PublicAckExpiresUnix`, for Codename;
   - `LICENSE_CONFIRMATION.md` plus `LicenseConfirmed` and `Ip.LicenseRef`, for Canon;
   - `TITLE_CLEARANCE.md`, if available.
   
   For an Original target the agent may set `Defaults.SkinPack = "Original"` on a release branch `release/<date>-original`. For Codename and Canon, the owner edits the flags on the release branch and commits the signed files and flag lines personally (L09/L10 provenance, §8.4.5); the agent commits nothing in those lines. The agent applies any PUBLISH_ACK owner option (OPT-1…OPT-3) that is checked on the release branch before step 2.
2. Agent: `lune run tools/check.luau`, the host gate. Expect `[ALG][CHECK] SUMMARY … fail=0`.
3. Agent: `lune run tools/lint_strings.luau --public --pack <P>` (pre-strip; Part 7 R1 plus L07–L21; L14, L15 and L22 need `--tree`). Exit 0.
4. Agent: `run_builder {"builder":"BuildAll","fn":"build","opts":{"reset":true}}`. Templates and the map are rebuilt with the look set implied by `Defaults.SkinPack` (§8.5.6). Then `verify` returns ok.
5. Agent: `lune run tools/lint_strings.luau --strip <P>`, then the sync gate. The stripped pack files must show the stub hash in Studio.
6. Agent: `run_builder {"builder":"Verify","fn":"publishGate","opts":{"target":"<P>"}}`. Save the result to `build/ip_tree.json`.
7. Agent: `lune run tools/lint_strings.luau --public --pack <P> --tree build/ip_tree.json` (adds L14, L15 and L22). Expect `[ALG][GATE] publish-ip PASS target=<P> …`.
8. Agent: Part 7 §7.7.6 R2–R4. In M0 and M1-G, R2 is **expected to FAIL** on placeholders, so a public publish is blocked; run the dry run below instead.
9. Agent: print `[ALG][GATE] publish PASS|FAIL target=<P> commit=<sha7> items=<n>/<n>`. If step 3 or 7 printed any `W-DRYRUN`, stop here: no H-11 blocker. Otherwise post a `human-step` blocker for H-11 that includes:
   - the gate lines;
   - the W-TITLE and W-EXEMPT-ID warnings;
   - the human checklist below.
10. **Human (H-11, public variant).** Complete HP-01 to HP-08. Then:
    1. In the agent's Studio session: **File ⟩ Save to File As…** `build/release-<P>-<sha7>.rbxl`.
    2. Open that file in a **separate** Studio window.
    3. Publish it to the target universe.
    4. Close the window.
    
    Never publish the dev session document to another universe: the agent's preflight pins the dev place (Part 1 §1.3).
11. Agent, after the human confirms: `lune run tools/lint_strings.luau --unstrip`, then the sync gate. Switch back to the working branch; Script Sync restores `Defaults`. Then `BuildAll {reset:true}` (dev look set) and `verify`. Record the publish in `PROGRESS.md` and `docs/agent/TRIGGER_WATCH.md`.

**Dry run (`ip.gate_dry_run`, agent procedure).** This exercises the public procedure while a real public publish is impossible (§8.1.5). It is an **agent procedure**, not a Studio scenario: it needs host commands, Edit-mode builders and git, so it never appears in a `studio(…)` selector. Target: Original.
0. **Setup.** On the stage branch, record the sync `agg` (`lune run tools/sync_manifest.luau`). Create the throwaway branch `release/<YYYY-MM-DD>-original-dryrun` from the stage head. On it, set `Defaults.SkinPack = "Original"` (allowed for Original by step 1) and draft `docs/agent/PUBLISH_METADATA.md` with `target_pack: Original` and the placeholder `target_universe_id: 1`; commit both on that branch only.
1. **Run** steps 2–9 with target Original, passing `--dry-run` to both `lint_strings --public` runs (steps 3 and 7). L11, L17 and L22 then print `W-DRYRUN` instead of failing. While the packs are stripped (after step 5), also run `studio(ip.no_missing_keys_tour) ≥ 1`. Step 9 prints the verdict and posts no H-11 blocker.
2. **Restore.** `lune run tools/lint_strings.luau --unstrip` (restores the packs from the dry-run branch's `HEAD`, which holds the full packs), switch back to the stage branch, run the sync gate, `BuildAll {reset:true}` and `verify`, then delete the local dry-run branch (it is never pushed).

Expected evidence:
- `[ALG][GATE] publish-ip PASS target=Original commit=<sha7> errors=0 warnings=<m>` (`W-DRYRUN` included in `m`);
- `[ALG][GATE] publish FAIL target=Original … items=<n>/<n>` with R2 failing (graybox placeholders);
- `[ALG][GATE] publish-hook FAIL items=3/4`, recorded as expected only when R1 (run with `--dry-run`), R3 and R4 pass; R4 is taken from the enclosing gate card's own A1–A11 results (Part 7 §7.7.6), so it is not circular;
- the post-run sync `agg` equals the pre-run `agg`.

**Human checklist HP (public publish):**

| ID | Check (Creator Hub / Studio) |
|---|---|
| HP-01 | The target universe is not the dev universe. The dev universe is still private. The target universe belongs to the production group (Original, Canon) or, for Codename, to the group named in PUBLISH_ACK `target_group_id`, which is neither the dev group nor the production group (§8.4.4). |
| HP-02 | The experience title, description, genre, tags, icon and thumbnails match `PUBLISH_METADATA.md` exactly. |
| HP-03 | Configs: `ALG_SkinPack` is absent in the target universe, or equal to the target pack. |
| HP-04 | Developer products, passes and badges in the target universe use the names and descriptions in `PUBLISH_METADATA.md`. Prices are signed off (H-09). |
| HP-05 | The group name, description and emblem, the Discord name and the social handles contain no AT words or art. |
| HP-06 | The content-maturity questionnaire is updated (GDD §16.1). |
| HP-07 | The repository is private. |
| HP-08 | Codename and Canon: the owner personally created and committed PUBLISH_ACK.md (or LICENSE_CONFIRMATION.md) (commit `<sha>`) and set the flag lines; it is within its validity window, and for Codename O4 still holds. This check, with the L09/L10 provenance check, is the binding control (§8.4.5). |

#### 8.4.8 Metadata rules

These apply to every non-Canon public build, Codename included. For Canon, the licensor's brand guidelines replace them.

| Field | Rule |
|---|---|
| Title | IP-neutral. "Algebraic!" only as plain title text (no show-logo font, colour or layout), pending counsel (W-TITLE, §8.7; PUBLISH_ACK M5 for Codename). Never "Adventure Time", never "AT", never a character or place name. |
| Description | No list A, B, C or E terms. No "inspired by", "fan game of" or "based on" phrasing. Optional, and only if counsel wants it: a plain non-affiliation line such as "Not affiliated with any TV series or studio", which names no brand. |
| Tags, genre, keywords | Generic (battlegrounds, PvP, fighting, anime-style, RNG). No brand or character tags. No SEO stuffing; Roblox lists "misleading or mismatched metadata" as a cause of lost impressions [Docs `discovery-faq.md`]. |
| Icon and thumbnails | Show only the Original look set or original key art. No Codename gear, no canon palette-plus-prop combinations, no show screenshots, logo or logo font. Recognizability rows `meta:icon` and `meta:thumbnail_<n>` must pass (L17). |
| Update names (title tags such as JJS's "[UPDATE]") | Same as Title. Never an episode title. |
| Product, pass and badge names and descriptions | Original strings (§8.3.6), lint-clean (L08, L11). |
| Group, Discord, YouTube, TikTok, X handles and bios | Same as Description. No AT art in banners or emblems. |
| Ads, sponsored listings, influencer briefs | Same as Description. Never "the Adventure Time game". |

#### 8.4.9 Verify

**Lune (`tests/ip/*.spec.luau`; fixtures in `tests/ip/fixtures/`):**

| ID | Layer | Check | Spec | Phase | Milestone |
|---|---|---|---|---|---|
| `ip.rules_schema` | lune | The rules file parses. Every term is lowercase or `caseSensitive`. Every `keyPattern` compiles. Each term's `packs` and `scopes` equal its list's row in the §8.4.2 Lists table (a term with a `note` may drop `src`). | §8.4.2 | P1 | M0 |
| `ip.normalize_matches` | lune | "IceKingBoss" hits "ice king". "Banana Guards" hits "banana guard", "Gumball Guardians" hits "gumball guardian", "Gunters" hits "gunter", "Founders' Island" hits "founders island", "GOLBetty" hits "golbetty". "candy_kingdom" in code becomes `<id>`. "Finnish", "lichen", "Jakeson", "Fernwyn", "globes" (list B "glob") and "luau-lsp" (src scope) do not hit. | §8.4.2 | P1 | M0 |
| `ip.original_clean`, `ip.codename_clean` | lune | L04 and L13 on the real packs return 0 errors | §8.3 | P1 | M0 |
| `ip.leak_detect` | lune | A fixture Original pack that contains Codename's "Hero Leap" gives an L13 error. A fixture Codename pack that contains "Kaiju Jake" gives L04 plus L13 errors. | §8.4.3 | P1 | M0 |
| `ip.algebraic_allowlist` | lune | "Algebraic" is allowed in Codename `ui.hud.combo_big` and rejected in Original `ui.hud.combo_big` | §8.4.2 | P1 | M0 |
| `ip.gambling_terms` | lune | Fixture `ui.wish.title = "Spin!"` gives an L07 error | §8.4.3 | P1 | M0 |
| `ip.product_keys_original_only` | lune | Fixture Codename `product.vip.name` gives an L08 error | §8.4.3 | P1 | M0 |
| `ip.ack_parse` | lune | Valid gives pass. Each of these gives an L09 error: expired; a missing required box; MON-A plus MON-B; an unknown key; `counsel_consulted: yes` with an empty `counsel_ref`; `Ip.PublicAckExpiresUnix` not equal to `valid_until` 23:59:59 UTC; `target_group_id` equal to the dev group; M5 unchecked while W-TITLE is raised; S4 unchecked while L22 finds exempt identifiers. `counsel_consulted: no` with an empty `counsel_ref` passes. | §8.4.5 | P8 | M1-G |
| `ip.license_parse`, `ip.metadata_parse` | lune | As above for L10 and L11 (L11: missing file, empty field and `<…>` placeholder each give an error) | §8.4.5 | P8 | M1-G |
| `ip.signed_file_provenance` | lune | In a temporary git repository fixture: an owner-authored commit of the signed file and flag lines passes; an agent-authored commit, a `Co-Authored-By: Claude` trailer, an uncommitted signed file and an agent-authored flag line each give the provenance error | §8.4.5 | P8 | M1-G |
| `ip.no_tool_writes_signed_files` | lune | §8.4.5 (a) static scan of `tools/**` Lune scripts and (b) `SafeWrite.writeFile` refusals | §8.4.5 | P8 | M1-G |
| `ip.strip_unstrip` | lune | In a temporary git repository fixture: strip Original writes two stubs; unstrip restores byte-identical files; strip on a dirty tree is refused | §8.4.3 | P8 | M1-G |
| `ip.l02_skips_stubs` | lune | An Original target with Codename and Canon strip stubs gives 0 L02 errors; a Codename pack with `strings = {}` but no `-- STRIPPED for public build` line still gives L02b errors; L14 fails when Original itself is a stub | §8.4.3 | P8 | M1-G |
| `ip.dry_run_warnings` | lune | With fixture inputs (placeholder metadata, a `pending` row, an exempt identifier in a tree dump): without `--dry-run` L11, L17 and L22 give errors; with `--dry-run` they give `W-DRYRUN` warnings and exit 0; a list A hit in a metadata field stays an error under `--dry-run` | §8.4.3 | P8 | M1-G |
| `ip.original_palette_distance` | lune | Each Original colour is ≥ 60 RGB-Euclidean from **every** colour of the same style's Part 4 `StyleDef.palette` (3 × 3 minimum, so role swaps cannot pass) and from that style's T2 reference colours (§8.5.6) | §8.5.6 | P4 | M1-G |
| `ip.gear_not_codename_clone` | lune | Each Original `gear_*` and `summon_*` recipe (except entries marked "unchanged") differs from its Codename recipe in at least 2 primitives; a primitive differs when its shape differs, a size component differs by more than 0.1, an offset component by more than 0.15, or it has no counterpart | §8.5.6 | P4 | M1-G |
| `skin.resolve_pack_matrix`, `skin.look_active_only` | lune | §8.2.6 (seven rows), §8.2.7 | §8.2 | P1 | M0 |

**Studio scenarios:** `ip.runtime_guard_public_sim`, `ip.canon_locked_runtime`, `ip.pack_switch_live`, `ip.dev_universe_members_only` and `ip.no_missing_keys_tour` (§8.2.8).

**Agent procedure (not a Studio scenario):** `ip.gate_dry_run` (§8.4.7), run twice in a row in the publish-gate card (spike S-P8-1). Expected evidence is listed in §8.4.7; `studio(ip.no_missing_keys_tour) ≥ 1` runs inside it while the packs are stripped.

---

### 8.5 Asset provenance and recognizability

#### 8.5.1 Provenance fields in the Assets manifest

`AssetEntry` (Part 2 §2.9.1) gains the optional field `prov`:

```lua
--!strict
-- Addition to src/Shared/Assets.luau types. Spec §8.5.1.
export type AssetProvenance = {
	author: string,   -- "agent" | "roblox" | "roblox-ai" | "<creatorId>" | "<artist legal name>"
	origin: string,   -- "graybox" | "builtin" | "creator-store:<assetId>" | "ai:<ASSET_PROVENANCE row key>" | "artist:<delivery ref>"
	license: string,  -- "original" | "roblox-builtin" | "creator-store-on-platform" | "roblox-generative-ai" | "work-for-hire:<contract ref>"
	likeness: boolean, -- §8.5.2
	recog: "n/a" | "pending" | "pass" | "fail", -- result for the ORIGINAL look set (§8.5.3); "n/a" iff likeness == false
	reviewRef: string?, -- "R-012": row in docs/agent/RECOGNIZABILITY.md (required when recog is "pass" or "fail")
}
-- AssetEntry = { id, status, source, kind, note?, prov: AssetProvenance? }
```

Rules M09–M12 are added to Part 7's `check_manifest.luau`:

| Rule | Severity | Check |
|---|---|---|
| M09 | error | `prov` is required when `source ∈ {ai, creator-store, artist}`, or `status == "final"`, or the key matches `^model%.%l+%.gear_`, `^model%.%l+%.summon_`, `^model%.monarch%.guardian_bust$`, `^img%.style%.`, or the asset is music or VO |
| M10 | error | `source == "creator-store"` or `"ai"` ⇒ `prov.likeness == false` (no fan-made or AI likeness, ever; §0.5) |
| M11 | error | `recog == "n/a"` ⇔ `likeness == false`. `recog ∈ {pass, fail}` ⇒ `reviewRef` names an existing row in `RECOGNIZABILITY.md` with the same result. |
| M12 | error (`--release` with an Original target) | Every referenced entry with `likeness == true` has `recog == "pass"` |

Part 1's `ASSET_PROVENANCE.md` row format gains two trailing columns: `| <likeness yes/no> | <R-### or n/a> |`. Its prompt column is linted (L12).

#### 8.5.2 What counts as likeness-bearing

The following need a recognizability review before they ship in a non-Canon public build:
- Style Gear and summon or set-piece models (`model.<style>.gear_*`, `summon_*`, `guardian_bust`);
- style palettes, and VFX that restage a specific moment;
- style icons and busts (`img.style.*`);
- map landmarks named in GDD §11 (castle silhouette, guardian statues);
- Realm set pieces;
- music, VO and barks;
- UI art (title lettering, logo, fonts);
- the icon, thumbnails and key art.

Generic props are not likeness-bearing: crates, rocks, generic candy shapes, rubble, graybox walls.

#### 8.5.3 Recognizability test (human review gate)

The legal question is roughly: would an ordinary fan recognize a specific protected AT character or place? This part turns that question into six tests (from the Oct 2026 IP research; GDD §5.5 summarizes them). A subject passes only if all six pass. Any fail sends it back for redesign as a data change, followed by re-review.

| Test | Pass condition | Who | Agent's preparation |
|---|---|---|---|
| T1 Silhouette | Show a black silhouette at 64 px to 5 people who know AT. **Fewer than 2** of them name an AT character. | Human (5 testers) | Captures from Part 4/6 scenario `look.gear_silhouette` with the Original look set and an all-black silhouette on a white background (new option `silhouette = true`) |
| T2 Palette + prop | No canon palette is combined with a canon signature prop: white bear-ear hat with blue shirt and sword; pink with crown; blue skin with gold crown and white beard; grey-blue skin with an axe-shaped bass; yellow-orange stretchy dog; ram horns on a green skeletal figure | Human | Lune `ip.original_palette_distance` result (set-wise, including the T2 reference colours of §8.5.6), and the §8.5.6 recipe table |
| T3 Pairing | No canon relationship is recreated: boy hero with yellow stretchy dog; vampire rocker with candy monarch; ice wizard with penguin companion | Human | List of kit pairings and summons |
| T4 Line | No line, name or slang matches canon | Automatic | `lint_strings` PASS (lists A, B, C, H) |
| T5 Place | A fan cannot name an AT location from a screenshot | Human | Part 6 map captures (plaza from the south ramp, the streets at head height, the gardens from a canopy: Part 6 §6.3 Verify, TASKS T213), plus one capture of both gate towers with the Original look set (TASKS T211, outside Part 6's three-capture set) |
| T6 Episode | No move or quest restages a specific episode beat | Human | The Original move list (§8.3.3) beside the GDD effect text; the slots whose Original name is a synonym of the Canon name are flagged |

**Review notes the agent adds to specific rows** (in the Evidence cell):
- `map:candy_kingdom` (T5) and `style:monarch` (T6): "Do the gate towers, or the Brass Colossus summon that stomps in and fires an eye beam, read as the canon guardian motif (two giant guardians flanking the castle gates)?"
- `style:hero` (T1, T2): "Does the sprout cap, satchel and leaf-blade read as any canon hero or as a green grass-sword duplicate of one?"

`docs/agent/RECOGNIZABILITY.md` row format. The agent adds rows with `Result = pending`; only a human fills Result and Reviewer.

```
| Row | Subject | Look set | Evidence | T1 | T2 | T3 | T4 | T5 | T6 | Result | Reviewer, date |
| R-001 | style:hero | Original | look.gear_silhouette run 2027-01-08; §8.3 rows | 0/5 | pass | pass | lint PASS | n/a | pass | pending | |
```

**Required subjects** for L17 are listed in §8.4.4. Subjects use the forms `style:<id>`, `pair:<id>+<id>`, `map:<mapId>`, `meta:icon` and `meta:thumbnail_<n>`.

#### 8.5.4 AI-generation prompt rule

This extends Part 1 §1.7.3.
1. Before every `generate_mesh`, `generate_material` or `generate_procedural_model` call, the agent runs `lune run tools/lint_strings.luau --text "<prompt>" --scope prompt` and proceeds only on exit 0.
2. A prompt uses generic nouns plus shape, colour and material. It never contains:
   - show, character, place or episode names;
   - **names from any pack**, Original included (prompts must be meaningful without them);
   - canon-quoting descriptors (list C) or signature-design descriptors (list E);
   - "in the style of …" any show, studio or artist.
3. No reference images from the show, fan art or screenshots of the Codename look are passed to any generation tool.
4. Results are `source = "ai"`, `likeness = false` (M10), and are recorded in `ASSET_PROVENANCE.md` with the verbatim prompt (L12).

#### 8.5.5 Creator Store rule

This extends Part 1 §1.7.2.
- Every `search_asset` query passes `--text … --scope prompt` first.
- V2 rejects any asset whose name, description, thumbnail, tags or creator page references the show, its characters, places or creators, or that depicts a likeness. Examples: bear-ear hats, axe-shaped basses, a gumball-machine guardian.
- **No fan-made AT asset is ever inserted**, even into the scratch place.
- Creator Store assets are `likeness = false` (M10). Insertion is limited to generic meshes, decals and audio.

#### 8.5.6 Look sets: the Original look set

Style Gear and summon templates are built for one **look set**. `BuildTemplates.build(opts)` reads `opts.lookSet`, which defaults to `"Original"` when `Defaults.SkinPack == "Original"`, else `"Codename"`. It builds:
- `ReplicatedStorage.Looks.Gear_<style>_<name>` and `Summon_*` from **Part 4 §4.7.4–§4.7.5** for `Codename` (used for Canon too until licensed art exists);
- from **the table below** for `Original`.

Every built template carries the attribute `LookSet` (string), which `Verify.publishGate` checks. Asset keys, attachments and `kitProps` are unchanged, so runtime code is identical. Palettes switch at runtime through `Skin.look` (§8.2.7). BuildTemplates keeps both look sets as data (per key, a list of primitives `{ shape, size, offset, role }`), so the Lune test `ip.gear_not_codename_clone` can compare them without Studio.

Each Original recipe below is described from its archetype only. The canon cross-reference for each subject lives only in the T6/T1 review evidence in `RECOGNIZABILITY.md` (§8.5.3), never in this table or in code (R-IP-09).

**Palettes** (Original pack `looks`). Each Original colour is ≥ 60 RGB-Euclidean from **every** colour of the same style's Part 4 palette (3 × 3 minimum, so a role swap cannot pass) and from that style's T2 reference colours below (`ip.original_palette_distance`). Measured set-wise minimums: hero 117 / 111 / 80, stretch 74 / 87 / 100, monarch 120 / 86 / 66, rocker 121 / 74 / 106, frost 97 / 101 / 65, decay 118 / 89 / 161.

| Style | Primary | Secondary | Accent | Concept (from the archetype) |
|---|---|---|---|---|
| `hero` | `#C8553D` terracotta | `#8B5A2B` bark | `#E07A2E` orange | Seed Knight: sprout cap, satchel, leaf-blade |
| `stretch` | `#7D5A4F` capybara brown | `#6E4B30` | `#7FB069` leaf | Rubber Capybara |
| `monarch` | `#2F8F8A` teal | `#B87333` copper | `#F4D35E` brass | Alchemist Regent: goggles, gear-pack, clockwork servants |
| `rocker` | `#5B4B8A` dusk violet | `#3D5A40` moss | `#E9D985` moth gold | Night Rocker: keytar, moth antennae |
| `frost` | `#A3B18A` sage | `#3E5C76` slate | `#E07A5F` terracotta | Frost Hermit: hood and scarf, no crown, no beard; no white and no gold |
| `decay` | `#7A8B3C` olive | `#5A3E5D` plum | `#D96C3B` rust | Rot Sovereign: mushroom cap, moss cloak |

**T2 reference colours** (canon signature colours that the Part 4 palette does not already contain; tuning default; T2 reviewers may add more). The test embeds the hex values only, with IP-neutral comments.

| Style | Extra reference colours |
|---|---|
| `hero` | `#1E3A8A` (dark blue), `#3FA34D` (green) |
| `stretch` | none |
| `monarch` | `#E8B923` (gold) |
| `rocker` | none |
| `frost` | `#E3B341` (gold), `#D32F2F` (red) |
| `decay` | none |

**Gear and summon recipes** (Original). Shapes, sizes in studs and offsets follow the Part 4 conventions and build rules (Accessory container, Edit-time `GeometryService:UnionAsync`, the same flags). Colours are named by palette role.

| Key | Attach | Original recipe |
|---|---|---|
| `model.hero.gear_hat` | HatAttachment | Sprout cap: Ball (1.4, 0.85, 1.4) primary (Handle) at (0, −0.1, 0) + brim Cylinder (1.9 dia × 0.12) secondary at (0, −0.35, 0) + a single sprout: stem Cylinder (0.12 × 0.45) secondary at (0, 0.55, 0) and one leaf Wedge (0.1, 0.35, 0.55) accent at (0.15, 0.8, 0). No pair of top bumps. |
| `model.hero.gear_pack` | BodyBackAttachment | Satchel: Block (1.1, 0.9, 0.55) secondary at (0.45, −0.6, 0) + strap Block (0.15, 1.9, 0.1) secondary rotated 35° about Z |
| `model.hero.gear_sword` | RightGripAttachment (Part 4 spike S-P4-5 fallback applies) | Leaf blade: Block (0.12, 2.4, 0.55) + Wedge tip (0.12, 0.7, 0.55) at (0, 1.55, 0); no crossguard; grip Cylinder (0.2 × 0.8) secondary at (0, −0.4, 0). Blade colour by `Kit` as in Part 4. |
| `model.stretch.gear_ears` | HatAttachment | Band Block (1.3, 0.15, 1.3) primary + round ears Ball (0.3, 0.25, 0.2) ×2 at (±0.5, 0.1, 0) secondary |
| `model.stretch.gear_tail` | WaistBackAttachment | Leaf tuft Wedge (0.5, 0.6, 0.15) accent |
| `model.monarch.gear_crown` | HatAttachment | Goggles: ring Cylinder (1.25 dia × 0.2) secondary + lenses Cylinder (0.45 dia × 0.2) ×2 accent at (±0.3, 0, −0.6), facing front |
| `model.monarch.gear_coat` | BodyBackAttachment | Gear-pack Block (1.3, 1.5, 0.5) secondary + cog Cylinder (1.0 dia × 0.15) accent at (0, 0.3, 0.3) |
| `model.rocker.gear_bass` | BodyBackAttachment | Keytar: body Block (2.2, 0.8, 0.3) accent + keys Block (1.4, 0.22, 0.32) white at (0, 0.15, 0) + neck Block (0.2, 1.1, 0.2) secondary at (1.5, 0.2, 0); slung 30° about Z |
| `model.rocker.gear_hair` | HairAttachment | Bob Ball (1.3, 0.9, 1.3) secondary + antennae Cylinder (0.08 × 0.9) ×2 at (±0.25, 0.7, −0.2) tilted ±20° with tips Ball (0.18) accent |
| `model.frost.gear_crown` | HatAttachment | Hood Ball (1.45, 1.3, 1.45) primary + trim ring Cylinder (1.45 dia × 0.22) secondary at (0, −0.35, 0) |
| `model.frost.gear_beard` | FaceFrontAttachment | Scarf Block (1.4, 0.35, 0.7) secondary at (0, −0.75, 0.1) + tail Block (0.35, 0.9, 0.12) at (0.45, −1.1, 0.2) |
| `model.decay.gear_horns` | HatAttachment | Mushroom cap Ball (1.7, 0.7, 1.7) accent at (0, 0.35, 0) + spots Ball (0.25) ×4 primary |
| `model.decay.gear_cape` | BodyBackAttachment | Moss cloak Block (1.8, 2.6, 0.12) primary + clumps Ball (0.5) ×3 secondary along the top edge |
| `model.monarch.summon_guard` | n/a | Clockwork guard: Block torso (1.3, 2.0, 1.0) secondary + head Cylinder (0.9 dia × 0.9) accent + spear Cylinder (0.15 × 4) + Wedge tip |
| `model.monarch.summon_robot` | n/a | Duelist drone: Ball body (1.4) primary + blade Block (0.2, 2.2, 0.4) secondary + rotor Cylinder (1.6 dia × 0.1) at (0, 0.9, 0) |
| `model.monarch.summon_guardian` | n/a | Brass Colossus: Block body (6, 8, 4) secondary + head Cylinder (4 dia × 3) primary at (0, 5.5, 0) + eye Ball (1.5) accent facing front + Block base (6, 3, 6); 16 studs tall in total |
| `model.monarch.guardian_bust` | n/a | Upper half of the Original colossus at 60 % (Part 4 rule) |
| `model.frost.summon_penguin` | n/a | Seal pup: Ball body (1.6, 1.0, 1.1) (150, 150, 160) + head Ball (0.8) at (0, 0.4, −0.6) + flippers Wedge ×2 |
| `model.frost.summon_golem` | n/a | Unchanged: the same generic ice golem as the Codename look set (Part 4 §4.7.5); `ip.gear_not_codename_clone` skips it |

**Other Original-look deltas** (requests to Part 6, applied when the build look set is Original):
- `rocker` "colour drain" VFX tints with the palette accent glow instead of red-to-grey.
- `BuildMap_CandyKingdom` replaces `Guardian_N` / `Guardian_S` with generic gate towers: a `Tower` Block 12 × 22 × 12 plus 4 merlons (a crenellated top), in the default Cover colour `#3A7BD5` (no pink); no head, no eye, no Ball (Part 6 §6.3.7 Original variant row; `opts.lookSet`, default from `Defaults.SkinPack`, same rule as `BuildTemplates`). The function is kept (gate marker, event stub); the landmark is not.
- One build holds one look set: `ReplicatedStorage.Looks` and the map are rebuilt by `BuildAll {reset:true}` whenever the look set changes (§8.4.7 step 4). The Original recipes for `monarch`, `rocker`, `frost` and `decay` are built in the tasks that implement those kits, and silhouettes are captured with `mcp(look.gear_silhouette)` (at most 3 captures per run).
- Map and arena review rows (T5) decide whether any other landmark changes; the `map:candy_kingdom` and `style:monarch` rows carry the guardian-motif review note (§8.5.3).

#### 8.5.7 Voice and music

- **Placeholder voice.** `AudioTextToSpeech` barks run in Studio only (`Perf.Audio.TtsInStudioOnly`, Part 6; L19). Their text is `ui.vo.*`. No voice preset is chosen or described as imitating a person or character.
- **Future VO** (artist assets):
  - original actors only;
  - casting briefs never name show actors or characters and never say "sound like …";
  - contracts include an originality and consent clause;
  - **no AI voice cloning or voice conversion** of any person (GDD §5.4, §14.5);
  - VO assets are `likeness = true`, and their review row records "no sound-alike".
- **Music:**
  - 100 % original (GDD §14.1);
  - no melodic quotation of the theme, "Island Song" or any show song, and no covers;
  - the composer signs an originality warranty;
  - music assets are `likeness = true`, and their review row records the melody check.
- M1-G ships no music (Part 6).

#### 8.5.8 Verify

- `check_manifest` M09–M12 (Part 7 runs them).
- Lune `ip.prov_rules`: fixture entries cover each M09–M12 violation.
- Lune `ip.original_palette_distance` (set-wise, §8.5.6) and `ip.gear_not_codename_clone`.
- Scenario `look.gear_silhouette` with `lookSet = "Original"` and `silhouette = true`: one capture per implemented style, attached to the review request.
- `BuildMap_CandyKingdom.verify()` with the Original look set: `Guardian_N/S` hold the gate-tower parts (`Tower` + 4 `Merlon_<k>`, Part 6 §6.3.7) and no Ball part.
- `BuildTemplates.verify()`: every `Gear_*` and `Summon_*` has `LookSet ∈ {Codename, Original}`, all with the same value.

---

### 8.6 Trigger watch and response playbook

#### 8.6.1 Watch list

The agent cannot reach Creator Hub, the License Manager catalog or the press. A human owns the watch and records each check in `docs/agent/TRIGGER_WATCH.md`. Row format: `| date | trigger | source | what happened | decision | actions done | by |`.

| ID | Trigger | How detected | Cadence |
|---|---|---|---|
| TW-01 | AT, Cartoon Network, Skydance or Nickelodeon listed in the **License Manager catalog** (matches are visible only to the rights holder; offers are TW-03/TW-04) | Creator Hub ⟩ Licenses catalog search; notifications | Weekly |
| TW-02 | An **official or licensed AT Roblox game** is announced or launched (Skydance in-house, Gamefam, Wonder Works or any partner) | Roblox newsroom, trade press, partner studio news | Weekly |
| TW-03 | A **standard license offer** is received | Creator Hub or email | On receipt |
| TW-04 | A **conditional license offer** is received | Creator Hub or email | On receipt |
| TW-05 | A **DMCA or Rights Manager removal** of the game or an asset | Roblox notice, game status | On receipt |
| TW-06 | A **cease-and-desist** or legal letter | Email or post | On receipt |
| TW-07 | **Press or creator coverage** naming the brand about our game | Search alerts | Weekly |
| TW-08 | **Breakout scale** on a public Codename build (typical CCU ≥ 5,000) | Analytics | Weekly |
| TW-09 | Skydance publishes **fan-content or brand guidelines**, or a reorganization changes the games or kids units | Press | Weekly |
| TW-10 | An **AT film, *Side Quests* or *Fionna and Cake* premiere or marketing beat** (an active franchise moment raises the risk) | Trade press | Weekly |
| TW-11 | The **dev universe is no longer private** (§8.2.6) | Creator Hub ⟩ the dev experience ⟩ privacy setting | Weekly |
| TW-12 | A public Codename build's **PUBLISH_ACK expires within 14 days** (`valid_until`) | `docs/agent/PUBLISH_ACK.md` | Weekly |

#### 8.6.2 Response playbook

Timelines start at detection. Steps marked (A) are agent-executable technical steps. Everything else is a human decision.

| Trigger | Within 24 h | Within 14 days | Never |
|---|---|---|---|
| TW-01, TW-02 | If a public Codename build exists: run the emergency switch (§8.6.3 step 1: `ALG_SkinPack = "Original"` and `ALG_Features` `{"CodenameLooks": false, "PaidRandomItems": false}` in that universe; names, palettes and Style Gear switch live). (A) Prepare the Original release (§8.4.7). | Publish the Original build within 72 h (look set and stripped packs). If AT is listed, the owner and counsel evaluate a **license request**: the IP must be central to and present throughout the game [Docs `faq.md`]; the license may set a 7-day average DAU floor (none, > 1k or > 25k) [Docs `license-manager.md`]; the game must be public [Docs `creators.md`]. Contact Skydance Games & Experiences (GDD §5.3). | Add AT content while a request is pending ("early use of IP … may lead to early activation" [Docs `creators.md`]) |
| TW-03 | Record the offer (rate, duration, standards). Counsel review. **The offer auto-activates on day 14 if no one acts.** | Decide. **Accept**: the build becomes Canon-track; complete LICENSE_CONFIRMATION.md. **Decline**: publish the Original build first (§8.4.7: Codename stripped, Original look set), then dispute with "I modified my creation to remove the IP". A ConfigService switch alone leaves Codename strings and looks in the shipped place, so it is not enough for that dispute reason. A second dispute archives the offer, and the docs name DMCA as the rights holder's next step. | Dispute with "does not use this IP" or "fair use" while a Codename build is live |
| TW-04 | Record the requested changes, rate and standards. Counsel review. | Decide. **Accept**: complete LICENSE_CONFIRMATION.md, then make the changes and mark them done (accepting activates the license agreement at the rights holder's rate once they approve). **Decline**: publish the Original build, then dispute with "I modified my creation to remove the IP" (a rights-holder-initiated conditional offer can be disputed), or let the offer expire. | Mark changes done that were not made (Roblox does not verify; the rights holder does) |
| TW-05 | Do **not** counter-notice AT content: counter-notice is under penalty of perjury and consents to jurisdiction (17 U.S.C. §512(f), (g)). Freeze monetization. (A) Prepare the Original build. Preserve all records. | Republish as Original only if the game was not removed outright; otherwise follow Roblox's instructions. Compensate players through Part 5 `compensation` grants. | Delete docs or history (litigation hold) |
| TW-06 | Forward to counsel the same day. No substantive reply without counsel. Comply with takedown demands. | Switch to Original or take the game down, per counsel | Argue fair use publicly |
| TW-07 | Do not amplify. No official post uses the brand. | Owner re-decides; switching to Original is recommended | Reply with "it's not AT" claims |
| TW-08 | Owner re-signs PUBLISH_ACK.md. The likelihood row moves from "small" to "breakout". | Re-decide | Continue silently |
| TW-09 | Read and comply | Update §8.4.2 lists and contacts | none |
| TW-10 | Owner re-reads the likelihood table (§8.1.3) | If a public Codename build is at breakout scale, re-decide (switching to Original is recommended) | Run AT-themed events or marketing alongside the beat |
| TW-11 | Set the dev universe back to **Private** at once and record who changed it. (A) Scan the server logs for `dev_universe_non_member` kicks. | Treat the exposure as a public publish without the gate; the owner decides on any follow-up | Leave the dev universe public |
| TW-12 | Owner re-signs PUBLISH_ACK.md (new `date`, `valid_until` and `Ip.PublicAckExpiresUnix`) and the build is republished through §8.4.7, or the owner accepts the automatic fallback | If not re-signed: at expiry the runtime guard falls back to Original names and hides the Codename looks (§8.2.6); publish the Original build within 72 h | Edit `valid_until` or `Ip.PublicAckExpiresUnix` without re-signing |

#### 8.6.3 Emergency switch (Codename to Original)

1. **Human, in minutes:** Creator Hub ⟩ the public universe ⟩ Configs: `ALG_SkinPack = "Original"`, and `ALG_Features` with `{"CodenameLooks": false, "PaidRandomItems": false}`.
   - The server applies the change immediately (Part 2 §2.6.4) and the guard logs nothing, because Original needs no acknowledgement.
   - Clients re-run `Skin.setPack` from their `Runtime.changed` handler (Part 2 §2.3.4). `Skin.onChanged` fires, and screens, the HUD and nameplates re-render their text (Part 6 §6.9.1, §6.8.4). Palettes follow through `Skin.look` / `Theme.stylePalette` (§8.2.7).
   - With `CodenameLooks = false`, AppearanceController hides every `Gear_*` / `Summon_*` whose `LookSet == "Codename"`, summons render as a neutral Ball in the Original palette, and Character mode renders as `Simplified` (Part 6).
   - The switch hides the Codename content; it does not remove it. The Codename strings and the Codename look set still ship in the place until step 3.
2. **Agent, within hours:** run the Original public procedure (§8.4.7 steps 2–9). This rebuilds the look set, which cannot change live.
3. **Human:** publish the Original build within 72 h, then remove the ConfigService overrides.

**Verify:** scenario `ip.pack_switch_live` covers step 1 (names, palette and hidden Codename looks within 1 s). The agent procedure `ip.gate_dry_run` covers step 2.

---

### 8.7 Title clearance note

**"Algebraic!" is an AT catchphrase.** Its trademark status was not searched.
- AT marks owned by The Cartoon Network, Inc. include ADVENTURE TIME serial 85477828, which covers video game programs; current status is unverified (USPTO record found in the Oct 2026 IP research; counsel to verify).
- Counsel should run a **clearance search** for ALGEBRAIC, and for MATHEMATICAL as an adjacent catchphrase, in classes 9, 28 and 41, then record the result in `docs/agent/TITLE_CLEARANCE.md`. Until then, L20 raises warning W-TITLE on every public gate (GDD Appendix F11).

If the title changes, only a few strings are title-bound:
- `ui.game.title`;
- the Codename and Canon strings in `algebraicAllowKeys` (meter name, awaken prompt, combo callout, awaken bark, awaken tip);
- metadata.

Original already avoids the word everywhere else. Internal names (`[ALG]` log tags, the `AlgebraicTestHarness` plugin, the `Algebraic DEV` universe name) are not player-visible and may stay.

**Backup titles.** None is clearance-searched yet; counsel should run knockout searches. None refers to AT, mathematics, "adventure" or "time".

| # | Candidate | Why it fits |
|---|---|---|
| 1 | *Wishbrawl* | Names the two pillars (Wishes and brawling) |
| 2 | *Patchwork Brawl* | Matches the Original world (`ui.world.name` "The Patchwork Wilds") |
| 3 | *Doodle Rumble* | The hand-drawn cartoon look |
| 4 | *Rumble Jar* | The wish-jar reveal motif (GDD §8.8) |
| 5 | *Brawl of Wishes* | Plain-language fallback |

---

### 8.8 New contracts introduced

1. **Skin API additions** (Part 2 module): `Skin.resolvePack(requested, gate) -> (PackId, PackDowngrade?)`, `Skin.look(styleId, pack: PackId?) -> LookOverride?` (one pack only, no fallback), and the types `PackGate` (with `ackValid`), `PackDowngrade` (`canon_locked | codename_public_unacked | codename_ack_expired`), `LookOverride`. The `Pack` type gains the optional `looks`. Packs are require-free pure modules. Part 2's `Skin.activePack()` and `Skin.onChanged(fn)` drive live re-rendering (§8.6.3).
2. **ConfigReplicator** applies `applyPackGuard` to every candidate snapshot and publishes the **resolved** `SkinPack`; outside a dev context a downgrade also sets `Features.CodenameLooks = false`. A `GUARD_RECHECK_SEC = 3600` loop re-applies the guard while a non-dev server requests Codename (ACK expiry). Log line: `[ALG][Skin] ERROR pack_downgraded effective=<p> reason=<r> requested=<p>`. Test seam: `ConfigReplicator._devContextOverride: boolean?`. New DevCmd: `setSkinPack(pack)`.
3. **Config:** a new code-only section `Ip = { DevUniverseIds: {number}, DevGroupId: number, LicenseRef: string, PublicAckExpiresUnix: number }`, on Part 2's not-overridable list. Kill switches in Part 2 `Features`: `PaidRandomItems = true` (when false, Part 5 hides and server-rejects PRI items exactly as when `paidRandomRestricted` is true) and `CodenameLooks = true` (when false, Part 6 AppearanceController hides the Codename look set immediately, §8.6.3).
4. **Pack coverage semantics** for Part 7 L02 and `skin.pack_key_coverage`: inheritance for non-identity namespaces, completeness for identity namespaces (§8.2.3), and **strip stubs are skipped** (a pack whose second line starts with `-- STRIPPED for public build` and whose `strings` is empty; L14 checks which packs are stubs). Codename and Canon contain overrides only.
5. **Skin keys** introduced here (†, §8.3.7): `ui.game.title`, `ui.world.name`, `ui.hud.{meter_name, awaken_ready, level, treasure, shards, sparks, quests}`, `ui.quest.<id>` (one per `Progression.Quests.DailyPool[*].id`), `ui.notify.quest_done`, `ui.report.{button, title, reason_exploit, reason_teaming, reason_harassment, reason_other, send}`, `ui.wish.{title, make_1, make_10, odds, pity_title, new, locked_paid, auto10, secret}`, `ui.wheel.{shards, convert, mastery_combat_only, variant, variant_base}`, `ui.shop.{title, refresh, coming_soon, balance, confirm_in_dev}`, `ui.store.{title, mock, spent_week}`, `ui.settings.{title, character_look, style_gear, own_look_oosona, own_look_avatar, tab_safety, quiet_barks, damage_numbers, hold_toggle_run}`, `ui.training.damage_numbers`. Token names: `ui.hud.protected {sec}`, `ui.shop.refresh {time}`, `ui.training.slot {n}`, `ui.hud.level {level}`, `ui.hud.treasure|shards|sparks {n}`, `ui.quest.<id> {goal}`, `ui.wheel.shards {n}`, `ui.wish.locked_paid {n}`, `ui.shop.balance {n}`, `ui.store.spent_week {robux}`. Shop merchant lines: `ui.shop.goose_line_1…6`. New registry families: `ui.quest.<id>`, `ui.report.reason_<reason>`.
6. **Rules file** `tools/lint_strings.rules.json` schema v2 (`list`, `keyPattern`, `allowKeys`/`algebraicAllowKeys`, `caseSensitive`, `exemptIdentifiers`, `dynamic`, scopes `pack|src|meta|prompt|tree`). The `src` scope includes `tests/**` and `tools/**` with the stated exemptions. The Lists table is authoritative for packs and scopes (`ip.rules_schema`). Matching removes apostrophes and accepts `s`/`es` endings for lists A and C.
7. **Lint additions** to the Part 7 mechanism: rules L07–L22; flags `--tree`, `--meta`, `--strip`, `--unstrip`, `--text --scope`, `--dry-run` (L11, L17 and L22 become `W-DRYRUN` warnings; agent procedure `ip.gate_dry_run` only); the gate line `[ALG][GATE] publish-ip …` and the final `[ALG][GATE] publish … items=<n>/<n>`. **L04 checks Codename pack values in dev mode** too, per `term.packs`. L09/L10 include the owner-provenance check (§8.4.5).
8. **`check_manifest` rules M09–M12** and `AssetEntry.prov: AssetProvenance?`. `ASSET_PROVENANCE.md` gains the columns `likeness` and `R-###`.
9. **`Verify.publishGate(opts)`**, plus the `run_builder.luau` branch `fn == "publishGate"`. Output file `build/ip_tree.json` (gitignored).
10. **Look sets:** `BuildTemplates.build({lookSet})`, defaulting from `Defaults.SkinPack`, with both look sets kept as primitive data. The `LookSet` attribute goes on every `Looks.Gear_*` / `Summon_*`. `BuildMap_CandyKingdom` builds generic gate towers instead of `Guardian_N/S` when the look set is Original. `Theme.stylePalette(styleId, variantId?)` (Part 6) replaces direct `StyleDef.palette` use in presentation and honours `Features.CodenameLooks`. Scenario option `look.gear_silhouette {lookSet, silhouette}`.
11. **Docs and tool files:**
    - human-only: `docs/agent/PUBLISH_ACK.md`, `LICENSE_CONFIRMATION.md`, `TITLE_CLEARANCE.md` (committed by `dev-place.json.ownerGitEmail`, checked by L09/L10);
    - shared: `PUBLISH_METADATA.md`, `RECOGNIZABILITY.md`, `TRIGGER_WATCH.md`;
    - templates in `docs/agent/templates/`, one `<NAME>.template.md` per file above (`PUBLISH_ACK`, `LICENSE_CONFIRMATION`, `TITLE_CLEARANCE`, `PUBLISH_METADATA`, `RECOGNIZABILITY`, `TRIGGER_WATCH`);
    - `tools/lib/SafeWrite.luau`, the only file writer for Lune host tools (refuses the human-only files).
    
    Formats are in §8.4.5, §8.5.3 and §8.6.1. The Edit/Write deny entries for the human-only files are in the Part 1 example permissions file (`docs/agent/claude-settings.example.json`, §1.2.4).
12. **Blocker type `ip`** (Part 1 §1.8.1): any §8.6 trigger, a recognizability fail, or a lint finding that renaming cannot fix.
13. **Human checklist HP-01…HP-08** and the public H-11 variant ("Save to File As", then publish from a separate window). PUBLISH_ACK gains `target_group_id`, boxes M5 (title, with W-TITLE) and S4 (exempt IDs, with L22), and owner options OPT-1…OPT-3.
14. **Test names:** `ip.*` (§8.4.9, including `ip.l02_skips_stubs`, `ip.dry_run_warnings`, `ip.signed_file_provenance`, `ip.no_tool_writes_signed_files`, `ip.gear_not_codename_clone`), `skin.resolve_pack_matrix` (seven rows), `skin.look_active_only`, and the Studio scenarios `ip.runtime_guard_public_sim`, `ip.canon_locked_runtime`, `ip.pack_switch_live`, `ip.dev_universe_members_only`, `ip.no_missing_keys_tour`. `ip.gate_dry_run` is an **agent procedure** (§8.4.7), never a Studio scenario.
15. **Release branch naming:** `release/<YYYY-MM-DD>-<pack lowercase>`; the dry run uses the throwaway branch `release/<YYYY-MM-DD>-original-dryrun`, which is never pushed.
16. **Groups and dev-universe access:** the Codename dev universe belongs to a dedicated dev group; the production group never hosts Codename-look content; PlayerService kicks non-members of `Ip.DevGroupId` in the dev universe outside Studio (`[ALG][Skin] ERROR dev_universe_non_member userId=<id>`; seam `PlayerService._devAccessCheck`).
17. **Trigger watch:** TW-09 is weekly; new rows TW-10 (franchise moments), TW-11 (dev universe still private) and TW-12 (ACK expiry within 14 days).

### 8.9 Issues with spine

1. **Part 0 fixed IDs embed AT place names.** `candy_kingdom`, `Workspace.Map.CandyKingdom`, `BuildMap_CandyKingdom` and the Flair ID `nightosphere` are not player-visible. They still ship in every build: instance names can be inspected on clients, and the IDs travel in analytics fields (Part 5 `match_summary` CF02/CF03). This part follows the spine: the dev lint exempts them as whole identifiers and prints W-EXEMPT-ID. Recommendation: rename them to IP-neutral IDs (for example `sugarcrest`, `infernal`) before any public build, in one spine-level change. Until then, rule L22 fails an Original public gate on them and requires PUBLISH_ACK box S4 for a Codename one (§8.4.3).
2. **Codename move names are "GDD move names" minus 12 substitutions.** §0.4 says Codename uses "GDD move names". The 12 GDD names in §8.3.1 contain canonical character names, an episode title or show slang ("Kaiju Jake", "GUNTER!", "Giant Lich", "Shmowzow!"). Keeping them would defeat the purpose of a codename pack. All but one would fail list A or B (with this part's list additions); "Empress Eyes", an episode title, is caught only by L13. This part reads "from GDD §7.3 move names" as "GDD names, made codename-safe"; the verbatim names live in Canon. If the owner wants them verbatim in Codename, list A must be relaxed for Codename, which is an owner decision (Q-P8-4).
3. **The `decay` archetype contains "Lich".** The Codename epithet uses "Decay Sorcerer". Part 4's draft archetype ID (the GDD column's word, containing "Lich") would fail the `src` lint; Part 4 §4.5.7 and §4.11 Q10 now use `DecayUndead` (TASKS §16.1 C12).

**Cross-part (not spine):**

4. Part 3's draft code comments used canonical names in the `ChassisMods` type, and so did its stun test name. Part 3 now uses style IDs (`hero`, `stretch`) and `stun.stretch_ragdoll_minus10` (TASKS §16.1 C12); keep it that way, or the `src` scope lint fails.
5. Part 6 colour-table sources say "Banana Guard yellow" and "bubblegum". These must not be copied into code comments; write `GDD §11.2 A` instead.
6. Part 5's draft named the shop lines `ui.shop.goose.*`; Part 6 names them `ui.shop.goose_line_<n>`. This part adopts Part 6's form, and Part 5 §5.12 item 14 now uses it.
7. Part 1 §1.7.6 says canonical names appear only in `Canon.luau` and the GDD. Add `tools/lint_strings.rules.json`, `tests/ip/fixtures/**` and `docs/**`, which hold the trigger log and the GDD section.
8. Part 7 §7.5.1 L04 says the dev scope is "the Original pack". This part requires Codename values to be checked too (lists A, B and H) in dev mode.
9. Part 7 asked whether public builds ship an empty Canon pack. **Yes:** every public build without a license strips Canon, and Original builds also strip Codename (§8.4.4).
10. Part 6 zone IDs `candy_streets` and `earldom_border` echo GDD zone names; "Candy Streets" is also an episode title. As instance names and attributes they would trip L13 in the tree scan, so they are exempt identifiers (§8.2.5). Recommendation: switch to IP-neutral zone IDs (`streets`, `border`) before any public build. This is a data-only change in Part 6 §6.3.1 and the Skin keys.

### 8.10 Open questions / spikes

| ID | Question | How to resolve | Fallback |
|---|---|---|---|
| S-P8-1 | Does Script Sync pick up rapid on-disk rewrites (`--strip` / `--unstrip`) reliably, and does the sync gate match afterwards? | Run the agent procedure `ip.gate_dry_run` (§8.4.7) twice in a row; the post-run sync `agg` equals the pre-run `agg` both times | Human toggles the root sync off and on; re-run the sync gate |
| S-P8-2 | Does `git show HEAD:<path>` through Lune `process.exec` return the exact bytes on Windows (CRLF)? | `ip.strip_unstrip` on Windows CI or host | Compare after normalizing line endings; write with `\n` |
| S-P8-3 | What is the maximum `execute_luau` response size? Does the deduplicated tree dump fit for the Candy Kingdom map? | Run `publishGate` on the full map; measure the JSON length | Paginate: `opts.service` scans one service per call |
| S-P8-4 | Does "Publish to Roblox As…" re-point the open Studio document to the new universe? | Not tested on purpose; the procedure always publishes a saved copy from a separate window | Procedure as written |
| S-P8-5 | Can `MarketplaceService:GetProductInfoAsync` / `BadgeService:GetBadgeInfoAsync` run from Edit-mode `execute_luau` (with H-06), so live dev-universe product names could be echoed into the gate? (Members verified in docs; Edit-mode behaviour unverified.) | Call it on one dev product ID | Keep names in `PUBLISH_METADATA.md` (current design) |
| Q-P8-1 | Counsel: is "Algebraic!" clear as a title on Track B? MATHEMATICAL status? | Clearance search (§8.7) | Backup title list |
| Q-P8-2 | Owner: is the 90-day validity of PUBLISH_ACK.md right (tuning default)? It is enforced at runtime through `Ip.PublicAckExpiresUnix` (§8.2.6). | Owner decision | 90 days |
| Q-P8-3 | Owner: should a public Codename build ever enable paid random items (MON-B)? | Owner decision at signing | MON-A recommended |
| Q-P8-4 | Owner or counsel: keep or replace the canon-adjacent descriptive Codename names ("Vampire Queen Ascendant", "Moon Beast Pounce", "Fool's Frenzy", "The Wishing Eye", "Well of Power", "Starchild Shapeshift", and the epithets "The Hyooman" and "The Lumpy Royal")? Restore the 12 verbatim GDD names in Codename? | Owner decision (PUBLISH_ACK OPT-3 for a public Codename build) | As written |
| Q-P8-5 | Design: GDD §5.5's world name "Land of Whatever" mirrors "Land of Ooo". This part proposes "The Patchwork Wilds". | Design decision | Proposal |
| Q-P8-6 | Is License Manager image matching run on Roblox-captured gameplay screenshots of avatar-mounted gear (Codename look)? | Not knowable from docs | Assume yes (risk table already does) |
