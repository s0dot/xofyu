[← Build spec index](../STUDIO_BUILD_SPEC.md) · **Part 8 of 8** · sections §8.x · Part 0 contracts (§0.x) are in the index file

## 8. Originality and naming layer

This part defines:
- the content and rules of the one shipped string pack, `Base`;
- where third-party names may never appear, and the rules data for the two string lints (`tools/lint_strings.luau` with the local denylist, and `tools/lint_retired_keys.luau`);
- the publish gate and the rules for public metadata;
- asset provenance, the AI-generation and Creator Store rules, and the human blind tests T7–T11 (`docs/agent/RECOGNIZABILITY.md`);
- title clearance for *Hullabrawl* (`docs/agent/TITLE_CLEARANCE.md`, §8.7);
- the trigger watch and response playbook for our own IP.

Ownership:
- Part 2 §2.9.2 owns the `Skin` API.
- Part 7 §7.5 owns the lint *mechanisms*: `tools/lint_strings.luau` (rules L01–L06, the denylist file format and the matching), `tools/lint_retired_keys.luau` (rules R01–R05 and the key-list format, §7.5.6) and `tools/check_manifest.luau` (rules M01–M08).
- Part 6 §6.9.4 owns the code of `Theme.stylePalette`; this part owns the palette rule it serves (§8.2.7).
- This part owns the rules *data*, the extra rules L07–L15 and M09–M13, the policy for the two owner-supplied lint lists (`tools/denylist.local.txt`, `tools/retired_keys.txt`), the gate procedure and the human-signed files.

Names, IDs and paths follow Part 0 verbatim. Player-facing names follow the Part 0 table §0.12.

---

### 8.1 Summary for the agent

#### 8.1.1 What the naming layer is for

*Hullabrawl* is an original IP: a battlegrounds fighter set in the Wishaway Isles, a chain of seaside holiday islands (GDD §3.7). The owner chose to ship it as an original IP on 16 Oct 2026 (GDD §5.8). The naming layer keeps that true in three ways:
- every player-visible string lives in one data pack, `Base`, written in the house voice (§8.3);
- two host lints keep third-party names and retired keys out of code, strings, metadata, prompts and anything that replicates to clients (§8.4.2, §8.4.3);
- a publish gate checks a public build mechanically, and human blind tests check that our characters, places and kits read as our own (§8.4, §8.5.3).

Combat, Wish, progression and UI code never hold display text. They use the IDs of §0.4 and Skin keys.

#### 8.1.2 One pack: `Base`

| Pack | What players see | Where it runs | Status |
|---|---|---|---|
| `Base` (`src/Shared/Skin/Packs/Base.luau`) | Every player-visible string: style names and epithets, move names, places, rarities, Finishes and every UI line, in the house voice (§8.3.1) | Everywhere: Studio, the dev universe and any public build | The only pack. No config key, ConfigService key, Dev command or runtime guard selects a pack (Part 2 §2.9.2) |

Looks follow the same rule: Charm Gear, Peak pieces, summon models and style palettes come only from Part 4 §4.7, and the Full Uniform is a per-player toggle that defaults to Off (§0.4). The publish gate is run by the agent and the humans; it has no config flag (§0.9).

#### 8.1.3 Risk in one paragraph

*This is not legal advice.* Counsel clears the title, the names, the roster and the store assets before art lock and before any public build (GDD §5.5, §21.2).

- **Copying risk is about expression, not ideas.** Moods, genres, mechanics and art styles are free to use. A specific cast, its designs, lines, places, lore and music are not, and neither is a move set that shows a known character's defining powers in play. Our cast, places and kits are written from our own fiction (GDD §3, §7). The guardrails of GDD §5.5 keep them there: the 13-factor distance rubric, the ensemble rule, the blind tests T7–T11, the neighbour check, the title knockout and counsel clearance.
- **Roblox makes claims cheap** [Docs `ip-licensing/*.md`, `production/publishing/rights-manager.md`]:
  - License Manager matches public, actively played games **daily by keywords and images**.
  - A standard licence offer **auto-activates after 14 days** unless it is disputed, at a revenue share the rights holder sets (**0–95 %**).
  - Rights Manager accepts removal requests of **up to 250 links**.
- **The same tools protect us.** Once Pip-Pip and the core characters are registered, Rights Manager can remove knock-offs of our own IP (TW-06, §8.6.1).
- **What raises the risk:** design drift (§8.5.6), marketing that names or hints at an existing show, film or game (§8.4.8), a skipped blind test, an uncleared title, and breakout scale.

**Likelihood of at least one adverse rights-holder action within 12 months of public launch** (a DMCA or Rights Manager removal, a cease-and-desist, or a licence offer we cannot accept). These are judgement estimates from precedent, not statistics (GDD §5.7). They assume a monetised game with paid Wishes, and that the blind tests and counsel clearance pass. Read them as a gradient.

| Build | Small (< ~500 CCU) | Breakout (~5k–50k CCU) | Hit + trigger event | Most likely form |
|---|---|---|---|---|
| This build: original IP, T7–T11 passed, title and names cleared by counsel | **< 1 %** | **1–3 %** | **3–6 %** | An exploratory letter after coverage calls the game similar to an existing work; a good-faith reply, dispute or counter-notice is available |

**What a removal costs:**
- the game;
- Robux-backed player progress, with refund backlash;
- the publishing group's standing under the repeat-infringer policy.

#### 8.1.4 Standing rules for the agent

| ID | Rule |
|---|---|
| R-IP-01 | Every player-visible string comes from `Skin.text` / `Skin.style` / `Skin.move`, served by the `Base` pack (§0.4). Surfaces are listed in §8.2.1. |
| R-IP-02 | No name of an existing show, film, game, character, place, catchphrase or brand appears in code, comments, identifiers, file names, asset keys, instance names, test names, fixtures, commit messages, PR text, log lines, AI prompts, asset searches, player-visible strings, metadata or the agent's own docs (`ARCHITECTURE.md`, `PROGRESS.md`, `RECOGNIZABILITY.md`, `TRIGGER_WATCH.md`, `ASSET_PROVENANCE.md`). Platform and tool names (Roblox, Rojo, Lune, vendored libraries) are fine. Test fixtures use invented words. |
| R-IP-03 | Our characters, places and kits are never described by comparison with an existing work, in any repository text, prompt or commit. Comparisons and the distance audit live only in counsel's files outside the repository. |
| R-IP-04 | Third-party names to block live only in the local denylist `tools/denylist.local.txt` (gitignored; supplied by the owner or counsel). The agent never creates, edits, opens into chat, quotes or copies it; only the lints read it (Part 7 §7.5.1). |
| R-IP-05 | Retired keys stay retired (`tools/lint_retired_keys.luau`, §8.4.3). The retired-key list `tools/retired_keys.txt` is supplied by the owner from counsel's archive and is gitignored. The agent never creates, edits, prints, quotes or copies it; only the lint reads it. |
| R-IP-06 | The agent never writes `docs/agent/TITLE_CLEARANCE.md`, the denylist or the retired-key list, by any tool (`lune run` scripts included; `tools/lib/SafeWrite.luau` refuses all three, §8.4.5). In `RECOGNIZABILITY.md` it only appends `pending` rows; it never fills a Testers, Same-work hits, Result or Signed cell, and never fills a sign-off cell in `ASSET_PROVENANCE.md`. If a provenance check fails (§8.4.5), the agent opens a blocker of type `safety`. |
| R-IP-07 | The agent never publishes (Part 1 §1.7.1). It runs the publish gate (§8.4.7) before it requests H-11. |
| R-IP-08 | Graybox only. AI generation is for generic props only, with linted prompts (§8.5.4). No Creator Store asset may depict, imitate or be named after anything from an existing show, film, game or brand (§8.5.5). No voice cloning, no sound-alike direction and no quoted music (§8.5.7). |
| R-IP-09 | The agent never decides originality questions. Any IP signal opens a blocker of type `ip` (Part 1 §1.8.1) with the facts and the matching playbook row. IP signals are: a §8.6 trigger, a lint finding that a rename cannot fix, and a recorded T7–T11 fail. |
| R-IP-10 | A task that makes a drift change (§8.5.6) to an implemented style or a built map appends the matching `pending` blind-test rows in the same task. Records are never deleted. |

#### 8.1.5 Status in M0 and M1-G

- All milestone work runs in the private dev universe. A public publish is impossible until assets are final: §0.5, and Part 7 item R2 (`check_manifest --release`) fails on placeholders.
- This part's gate is therefore built in phase P8 and run as a **dry run** (the agent procedure `ip.gate_dry_run`, §8.4.7), so it is ready when the owner decides to publish.
- The blind tests run on their own schedule (§8.5.3): T7–T10 on concept art before art lock; T11 after M0 on the Starters, before any public footage, and again for each slice style as it lands in M1-G.

---

### 8.2 Skin system rules

#### 8.2.1 Player-visible surfaces

| Surface | Members | Rule |
|---|---|---|
| GUI text | `TextLabel.Text`, `TextButton.Text`, `TextBox.Text`, `TextBox.PlaceholderText`; Part 6 `UI/Create` helpers | Skin key only |
| World text | BillboardGui and SurfaceGui labels (nameplate, signs); `ProximityPrompt.ActionText` / `.ObjectText` | Skin key only, set at runtime on the client (Part 6 §6.3: builders write `""`) |
| Server text | `Notify` payloads (keys only, Part 2 §2.7.4 row 22); `Player:Kick` reasons (`ui.error.*`) | Key only |
| Voice and subtitles | `AudioTextToSpeech.Text` (Studio placeholder barks, Part 6); subtitles | `ui.vo.*` |
| Names drawn by the core UI | `Tool.Name` / `Tool.ToolTip`, `Team.Name`, `leaderstats` child names, NPC `Humanoid.DisplayName` | None in M1-G. If one is added, set it from Skin at runtime. |
| Platform metadata | Experience title, description, tags, search keywords, icon, thumbnails, update names; product, pass and badge names and descriptions; group, Discord and social names | `docs/agent/PUBLISH_METADATA.md`, linted (§8.4.8) |

All members above were checked in the API docs (`TextLabel.yaml`, `TextButton.yaml`, `TextBox.yaml`, `ProximityPrompt.yaml`, `AudioTextToSpeech.yaml`, `Tool.yaml`, `Humanoid.yaml`).

#### 8.2.2 Keys and namespaces

**Grammar:** `^[a-z][a-z0-9_]*(%.[a-z0-9_]+)+$` (lowercase and dotted, Part 2 §2.10 item 3).

**Interpolation:** `{token}`. Tokens whose names end in `Id` are resolved by Part 2 §2.9.2. Every key's token set is fixed (§8.3.7 notes); rule L02 enforces it.

| Namespace | Pattern | Key list owner |
|---|---|---|
| style | `style.<id>.name`, `style.<id>.epithet` (all 18 IDs) | §0.4 / §8.3.2 |
| move | `move.<moveId>.name` for `m1, passive, q, 1–4, awaken, a1–a4` of every **implemented** style | §0.4 / §8.3.3 |
| map | `map.<mapId>.name`, `map.hub.zone.<zoneId>` | Part 6 / §8.3.4 |
| flair, rarity | `flair.<id>.name`, `rarity.<id>.name` | §0.4 / §8.3.5 |
| variant | `variant.<styleId>.<vid>.name` (`v1`, `v2`) | Part 5 / §8.3.6 |
| title | `title.level_<n>`, `title.mastery_<styleId>_5`, `title.master_<styleId>` | Part 5 / §8.3.6 |
| product, pass, badge | `product.<key>.name`, `pass.<key>.name`, `badge.<key>.name` | Part 5 / §8.3.6 |
| ui | `ui.<screen>.<key>` | Parts 2, 5, 6, 8 |

Part 7's L03 ("no unknown namespaces") accepts exactly the namespaces in this table.

#### 8.2.3 Pack file: shape and purity

Shape (Part 2 §2.9.2 `Pack`):

```lua
--!strict
-- Shape of the value returned by src/Shared/Skin/Packs/Base.luau (Part 2 §2.9.2 `Pack`).
export type Pack = {
	id: "Base",
	strings: { [string]: string },
}
```

Rules:
1. **Pure and require-free.** The pack file contains literal tables and loops only. It is loaded two ways: by Studio `require`, and by Part 7's Lune loader (which the lints use). It starts with `--!strict` and builds `strings` through a local `put()` that asserts no duplicate key (§8.3.8).
2. **Complete.** Every key referenced by `src/` resolves in `Base` (rule L02).
3. **Strings only.** Style palettes come from Part 4 `StyleDef.palette` and reach presentation only through `Theme.stylePalette` (§8.2.7). The pack holds no colours, asset IDs or look data.
4. **Future overrides.** A later locale or seasonal text pack would hold overrides only and fall back to `Base`. Adding one is a spec change to Part 2 §2.9.2 and this part. None ships in M0 or M1-G.

#### 8.2.4 Missing-key behaviour

- **Runtime** (Part 2 §2.9.2). Lookup order is `Base` → the literal `"[" .. key .. "]"`. Each key miss logs once: `[HB][Skin] WARN missing key=<key> pack=Base`. The bracket marker is deliberate: it is visible in dev and easy to spot in screenshots.
- **Host.** L02 fails for any key referenced in `src/` that is missing from `Base`, and for any registered family (§8.4.3) that is incomplete.
- **Studio.** Scenario `ip.no_missing_keys_tour` runs in the M1-G gate and inside the agent procedure `ip.gate_dry_run` (§8.4.7):
  1. Open every screen: the HUD (wallet and level, Daily Quests, the tide chip on the hub), the Charm Ring and the Scrapbook (with the Stamps and Off-Duty Look controls), the Wish screen (odds and a Mailbag reveal), Sundry's Rack, the Store, Settings (including the Safety tab and the Postcard Look editor), Report, Results, and the Staff Room panel.
  2. Trigger each Notify kind.
  3. Assert that `Skin.missing()` is empty on server and client and that no `[HB][Skin] WARN missing` line was printed.

#### 8.2.5 Where third-party names may never appear

| Place | Rule | Enforced by |
|---|---|---|
| `src/**`, `tests/**` and `tools/**`: code, comments, identifiers, string literals, file and folder names | No denylist entry; no retired key | L04 (`src` scope, Part 7 §7.5.1); `lint_retired_keys` (Part 7 §7.5.6) |
| `Base` pack values | No denylist entry; no rules-file term in its scope (§8.4.2) | L04, L07 |
| Asset keys and `Assets` `note` fields | IDs only (keys use style IDs, §0.4) | L04 (`Assets.luau` is in `src`) |
| Instance names, attribute names and values, StringValues and text in replicated services | No denylist entry; no retired key | L11 (Studio tree dump); `lint_retired_keys --tree` |
| Product, pass and badge names and descriptions | The `Base` names, which are also the Creator Hub names | L08, L09 |
| Telemetry event, funnel and step names; custom-field values | Lowercase IDs and enums only. Never pass `Skin.*` output to `Telemetry`. | L04, plus review |
| Log lines | Keys and IDs only, never resolved Skin text | Review |
| Commit messages, PR titles, branch names | IDs and plain words only | `lune run tools/lint_strings.luau --text "<msg>" --scope text` (optional `commit-msg` hook) |
| AI prompts and Creator Store search queries | Generic nouns only (§8.5.4, §8.5.5) | L10; `--text … --scope prompt` before every `generate_*` / `search_asset` call |
| Public metadata | §8.4.8 | L09, L08 |
| The agent's docs (`ARCHITECTURE.md`, `PROGRESS.md`, `RECOGNIZABILITY.md`, `TRIGGER_WATCH.md`, `ASSET_PROVENANCE.md`) | No third-party names. Blind-test answers are counted, never written down by name | Review; `--text … --scope text` when in doubt |

Docs under `docs/` are not scanned. The GDD may name genre and business benchmarks in its research, market, monetisation and live-ops text (GDD §2, Appendix E), never as a model or contrast for our characters, places or kits.

#### 8.2.6 Dev universe access and groups

Unreleased characters and placeholder assets stay private until the publish gate passes and the core characters are registered (§8.1.3). Three rules keep them there.

- **Groups.** The dev universe and every placeholder upload (for example the H-07/H-08 animations, graybox and AI placeholder assets) live in a **dedicated dev group** that owns no public experience (Part 1 H-01). The production group owns only public universes, receives only final, signed-off assets, and gets any asset it needs re-uploaded at release. A claim or moderation action about a placeholder then lands on the dev group, never on the group that publishes the game.
- **The dev universe stays private.** Making it public is a public publish and needs the gate. The human re-checks it weekly (TW-09, §8.6.1).
- **Dev universe members only** (defence in depth, in case the dev universe is made public by mistake). PlayerService runs the check (Part 2 §2.3.5 hosts the code; this part owns the rule). On join, before `DataService.loadAsync`, outside Studio, when `table.find(cfg.Ip.DevUniverseIds, game.GameId) ~= nil`: `local ok, member = pcall(player.IsInGroupAsync, player, cfg.Ip.DevGroupId)`. Unless `ok and member`, the player is kicked with `player:Kick(Skin.text("ui.error.restricted"))` and the server logs `[HB][Skin] ERROR dev_universe_non_member userId=<id>`. A failed call also kicks (fail closed).

**Config section** (code-only, never ConfigService-overridable; Part 2 §2.6.2 lists it):

```lua
--!strict
-- The code-only `Ip` section of src/Shared/Config/Defaults.luau (Part 8). JSON-safe. Spec §8.2.6.
-- Shown as a local; Defaults.luau places it in the Defaults table as the field `Ip`.
local Ip = {
	DevUniverseIds = {} :: { number }, -- the dev universe only: the agent copies `universeId` from docs/agent/dev-place.json (L12)
	DevGroupId = 0, -- the dedicated dev group: the agent copies `groupId` from docs/agent/dev-place.json (L12)
}
```

- The agent may write only the dev universe ID and the dev group ID from `docs/agent/dev-place.json` (human-confirmed at H-01). L12 enforces both.
- Overriding `Ip` through ConfigService would switch the members-only check off, so ConfigReplicator never reads an `HB_` key for it (Part 2 §2.6.4).

APIs used:
- `DataModel.GameId` (int64; read None; ReadOnly, NotReplicated) [Docs `DataModel.yaml`].
- `Player:IsInGroupAsync(groupId)` (Yields; security None; not deprecated, unlike `Player:IsInGroup`) and `Player:Kick(message)` [Docs `Player.yaml`].

**Studio-only seam:** `PlayerService._devAccessCheck(player: Player, opts: { forceDevUniverse: boolean?, member: boolean?, dryRun: boolean? }?): boolean` runs the members-only rule with the universe test and the group answer forced by `opts`. With `dryRun = true` it returns the decision and logs (adding `dry_run=true`) instead of kicking. `opts` is ignored outside Studio, like every seam.

#### 8.2.7 Style palettes in presentation

Players see one palette per style: the one the blind tests reviewed (T8, §8.5.3). To keep what is shown identical to what was reviewed, presentation code never reads `StyleDef.palette` directly and never invents style colours. This part owns that rule. Part 6 owns the helper `Theme.stylePalette` in `UI/Theme.luau` (the canonical code is in Part 6 §6.9.4; it is reproduced verbatim below) and uses it for:
- proxy-rig colours;
- the nameplate chip and ring (the Style Tell);
- Charm Ring chips (including the Off-Duty Look picker);
- VFX palette tints;
- Charm Gear colours;
- Sundry's Rack cosmetic cards.

```lua
--!strict
-- Theme.stylePalette — style colours for presentation (canonical). Spec §6.9.4.
-- Excerpt of UI/Theme.luau: the require goes at the top of the module; `Theme` is the module table above.
-- Off-Duty Look rule: §6.8.2 (v1 swaps secondary/accent; v2 and later darken 25 %).
local Styles = require("@game/ReplicatedStorage/Shared/Styles")

export type StylePalette = { primary: Color3, secondary: Color3, accent: Color3 }
local BLACK = Color3.new(0, 0, 0)

function Theme.stylePalette(styleId: string, variantId: string?): StylePalette
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

The `§` references in the code comments are Part 6's (§6.9.4, §6.8.2).

`Color3.new` and `Color3:Lerp` are verified [Docs `datatypes/Color3.yaml`]. Off-Duty Looks recolour a style's own palette only; there are no themed variants of anyone else's characters (§0.12).

#### 8.2.8 Verify

**Lune tests:**
- Part 2's `skin.fallback_order`, `skin.interpolate_ids`, `skin.missing_logged_once` and `skin.on_changed`.
- Part 7's `skin.pack_key_coverage` (every referenced key exists in `Base`; families from the §8.4.3 registry).
- This part's `ip.base_clean` and `ip.dev_ids_match` (§8.4.9).

**Studio scenarios:**
- `ip.dev_universe_members_only` (§8.2.6 seam). `PlayerService._devAccessCheck(P, {forceDevUniverse = true, member = false, dryRun = true})` returns false and logs `[HB][Skin] ERROR dev_universe_non_member dry_run=true userId=<id>` (run with `allowErrors {"ERROR dev_universe_non_member"}`, Part 1 §1.4.6); `member = true` returns true with no log; with `opts = nil` in Studio it returns true (Studio skip).
- `ip.no_missing_keys_tour` (§8.2.4).

---

### 8.3 Base pack content

The tables are ready to paste into `src/Shared/Skin/Packs/Base.luau`.

**Who owns the text.** Each key's `Base` text is owned by the part that defines the key:
- Part 4: style and move keys (`style.*`, `move.*`; Part 4 §4.5.10, with the names from the GDD §7.3.N kit tables);
- Part 5: Wish, shop, progression and appearance-data keys (Part 5 §5.12 item 14);
- Part 6: HUD, UI and world keys (Part 6 §6.9, §6.13 item 16);
- this part: the remaining core keys (for example `ui.game.title`, `ui.world.name`, the `ui.error.*` lines and the † keys of §8.3.7 that Part 5 does not list).

These tables copy the owners' text verbatim. Where an owner names a key but gives no text, the text below is the `Base` text, and the owner copies it if it ever quotes the line. A wording change is made in the owner's part and here in the same task.

#### 8.3.1 Writing rules

- **Our own words, from our own fiction.** Every name and line is written from the world and the roster (GDD §3.7, §7) and the player-facing names (§0.12). Nothing is adapted from another work's names, lines, catchphrases or slang, and no string winks at another show, film or game.
- **Plain words for players.** Short and concrete, readable by 9–15-year-olds in the US, Brazil and Southeast Asia. Britishness lives in signs and props, not in move names or UI text.
- **The house voice.** Sincere first, silly second; nobody is cynical about Guests. UI lines carry a seaside hotel's courtesy ("Enjoy your stay!", "Thank you kindly"). The same courtesy turns eerie only in the Haar's voice and the Management's notices, never in friendly UI.
- **Our words** (GDD §3.8). "Checked out!" is the only KO call-out (a puff of sand and a waving hankie). "Go Peak!" is the Peak call-out. "Fully booked" means a Getaway is open. "Off-season" means gloomy. No other line reuses these.
- **Punch at pomp, never at grief.** Committees and the Management are fair game. The Sleepwalkers, the empty deckchair and the Last Ferry are not.
- **Slapstick, not violence.** KO text is "Checked out!", never words of death or injury (Roblox Mild label, GDD §16.1).
- **Lore time counts in summers** ("365 summers"). The rules file blocks thousand-year phrasing in pack values and metadata (§8.4.2).
- **One catchphrase per staff member**, for example "TAKE A SEAT!", "You're SAFE now!" and "Measure twice!".
- **Sundry is plain-spoken.** No rhymes, never money-mad, never pushes a purchase.
- **Coined words** from the kids' workshop (candidates "Gullcrumbs!" and "ploff") enter the pack only after the owner approves them and counsel has knocked them out (§8.7).
- **Names are knocked out before public use.** The title, the world name, place names, the mascot and NPC names and the 18 style names pass the counsel knockout (§8.7) and the local denylist (L04).

#### 8.3.2 Styles (all 18; `style.<id>.name` / `style.<id>.epithet`)

| ID | Rarity | Name | Epithet |
|---|---|---|---|
| `attendant` | Starter | Sonny | Take a Seat |
| `lifeguard` | Starter | Paddles | The Lifeguard Who Can't Swim |
| `skimstone` | Common | Skimmer | Never Sinks |
| `masseur` | Common | Kneady | The Mandatory Massage |
| `bellhop` | Common | Pocket | The Express Bellhop |
| `angler` | Uncommon | Stilts | The Patient Hook |
| `kite` | Uncommon | Fret | The Kite Afraid of Heights |
| `boxcamera` | Uncommon | Shutter | Say Cheese |
| `sandcastle` | Rare | Burrows | The Tide-Proof Architect |
| `scoop` | Rare | Dollop | Every Flavour Ever |
| `diver` | Rare | Plunge | The Perfect Ten |
| `busker` | Epic | Bellows | The Pier Busker |
| `lamplighter` | Epic | Flicker | The Light That Never Went Out |
| `cablecar` | Epic | Yodel | The Summit Echo |
| `dustsheet` | Legendary | Mothball | Under Wraps |
| `raincloud` | Legendary | Bluster | Rain Stopped Play |
| `serpent` | Mythic | Seldom | The Seldom Seen |
| `ferry` | Cosmic (secret) | Homeward | The Last Ferry |

#### 8.3.3 Move names for the six vertical-slice kits (`move.<moveId>.name`)

The names are Part 4's (§4.5.10), copied word for word. Each comes from its kit table (GDD §7.3.N); the `m1` names are Part 4's tuning defaults, because the GDD describes each M1 chain by its action and does not name it.

**`attendant`** (GDD §7.3.1)

| Slot | Name |
|---|---|
| m1 | Chair Jabs |
| passive | Seen It All |
| q | Reserved |
| 1 | Hop In |
| 2 | Ticket Punch |
| 3 | Chair Toss |
| 4 | Take a Seat! |
| awaken | Rush Hour |
| a1 | Not on My Beach |
| a2 | Domino Deckchairs |
| a3 | Tea Break |
| a4 | Musical Chairs! |

**`lifeguard`** (GDD §7.3.2)

| Slot | Name |
|---|---|
| m1 | Buoy Slaps |
| passive | Walk, Don't Run! |
| q | Pole Vault |
| 1 | Buoy Toss |
| 2 | Belly Flop |
| 3 | Ring Toss |
| 4 | Rescue Hold |
| awaken | Red Flag Day |
| a1 | No Swimming! |
| a2 | Wave Machine |
| a3 | Bodyboard |
| a4 | Whirlpool |

**`sandcastle`** (GDD §7.3.9)

| Slot | Name |
|---|---|
| m1 | Spade Swipes |
| passive | Hard Hat |
| q | Spirit Level |
| 1 | Wet Sand |
| 2 | Sandcastle |
| 3 | Dig a Moat |
| 4 | Let the Tide In! |
| awaken | Topping Out |
| a1 | Drawbridge! |
| a2 | Grand Castle |
| a3 | Sandbags |
| a4 | Castle Keep |

**`scoop`** (GDD §7.3.10)

| Slot | Name |
|---|---|
| m1 | Cone Bops |
| passive | Toppings |
| q | Ding-Ding! |
| 1 | Scoop Shot |
| 2 | Brain Freeze |
| 3 | Melt Away |
| 4 | Meltdown |
| awaken | Soft Serve |
| a1 | Sprinkle Shower |
| a2 | Squirty Cream |
| a3 | Wafer Volley |
| a4 | Last Call |

**`busker`** (GDD §7.3.12)

| Slot | Name |
|---|---|
| m1 | Wing Slaps |
| passive | Tip Jar |
| q | Pass the Hat |
| 1 | Squeezebox Puff |
| 2 | Heckler |
| 3 | Snack Snatch |
| 4 | Cymbal Crash! |
| awaken | One-Man Band |
| a1 | Off-Key |
| a2 | Drum Roll |
| a3 | Gull Gang |
| a4 | Sing-Along! |

**`dustsheet`** (GDD §7.3.15)

| Slot | Name |
|---|---|
| m1 | Sheet Flicks |
| passive | Gathering Dust |
| q | Hands Off! |
| 1 | Moth Puff |
| 2 | Peekaboo |
| 3 | Left Behind |
| 4 | Draped! |
| awaken | All Covered |
| a1 | Sheet Flap |
| a2 | Haunt |
| a3 | Linen Whirl |
| a4 | Greetings from the Closed Season |

When a new kit is implemented (Part 4 §4.8), its 12 rows are added to `Base` in the same task, written from that kit's GDD §7.3.N table. L02 fails until they exist. A rename of a move in an implemented kit is a drift change only if the move's effect changes too (§8.5.6).

#### 8.3.4 Maps and zones

| Key | Base |
|---|---|
| `map.arena_duel_m0.name` | The Bandstand |
| `map.hub.name` | Elsewhere-on-Sea |
| `map.training.name` | The Staff Room |
| `map.hub.zone.under_pier` | Under the Pier |
| `map.hub.zone.pavilion` | The Pavilion |
| `map.hub.zone.promenade` | The Promenade |
| `map.hub.zone.beach` | The Beach |
| `map.hub.zone.bandstand_gardens` | Bandstand Gardens |
| `map.hub.zone.jubilee_clock` | The Jubilee Clock |

- Part 6 §6.1.3 and §6.3.1 own the hub's zone IDs; this table uses exactly those six. If Part 6 changes a zone ID, this table changes in the same task.
- The Year-1 maps (`gullcliff`: Gullcliff; `mountcocoa`: Mount Cocoa) get their `map.<mapId>.name` keys in the task that builds them.

#### 8.3.5 Rarity, Finish and postal-zone names

- **Rarity** (`rarity.<id>.name`; shown by its plain name everywhere): `starter` Starter, `common` Common, `uncommon` Uncommon, `rare` Rare, `epic` Epic, `legendary` Legendary, `mythic` Mythic, `cosmic` Cosmic.
- **Finish** (`flair.<id>.name`; the `Flair` data field): `plain` Plain, `glossy` Glossy, `gilt` Gilt, `seaglass` Sea-Glass, `pearl` Pearl, `glowtide` Glowtide, `misprint` Misprint.
- **Postal zone** (`ui.wish.zone_<rarity>`; printed on the stamp as flavour only, Part 5 §5.2.13): `common` Local, `uncommon` Island, `rare` Inter-Island, `epic` Far Isles, `legendary` Fog's Edge, `mythic` Deep Water, `cosmic` Beyond the Haar. Starters are never rolled, so `starter` has no zone.

#### 8.3.6 Generated families

These are built by loops inside the pack file (§8.3.8). The values are Part 5's (§5.12 item 14).

| Family | Base |
|---|---|
| `variant.<styleId>.v1.name` / `.v2.name` (18 × 2; the placeholder Off-Duty Looks) | "Day Off" / "Evening Out" (graybox placeholders; the designed names per style are in GDD §7.3) |
| `title.level_<n>`, n = 10…100 step 10 | 10 Paddler, 20 Beachcomber, 30 Promenader, 40 Crowd-Pleaser, 50 Pier Performer, 60 Star Turn, 70 Showstopper, 80 Top of the Bill, 90 Isle Favourite, 100 Grand Entertainer |
| `title.mastery_<styleId>_5` | `"<style name>'s Pen Pal"`, e.g. "Sonny's Pen Pal" |
| `title.master_<styleId>` | `"<style name>'s Understudy"`, e.g. "Sonny's Understudy" |
| `product.<key>.name` | `wishes_10` "10 Wishes", `wishes_55` "55 Wishes", `wishes_120` "120 Wishes", `wishes_300` "300 Wishes" |
| `pass.<key>.name` | `vip` "First Class", `quick_wish` "Quick Post", `lucky_jar` "Lucky Stamp" |

Product and pass names are also the **Creator Hub** names. The purchase prompt shows the platform name, not Skin, so L08 checks that they match `PUBLISH_METADATA.md` (§8.4.3).

#### 8.3.7 Core UI strings

- Keys marked † were added to the key set by this part (decision D-42); Parts 5 and 6 use them instead of inventing their own. The mark records where a key came from, not who owns its text. Their text follows the ownership rule of §8.3: the † keys listed in Part 5 §5.12 item 14 (the wallet and quest lines, `ui.notify.quest_done`, `ui.wheel.shards` and `ui.wheel.convert`, and the `ui.wish.*` and `ui.shop.*` keys) are Part 5's text, copied verbatim; this part owns the text of the other † keys.
- `{tokens}` are fixed per key; L02 enforces them.

| Key | Base |
|---|---|
| `ui.game.title` † | Hullabrawl *(title pending clearance, §8.7)* |
| `ui.world.name` † | The Wishaway Isles |
| `ui.hud.meter_name` † | Peak Meter |
| `ui.hud.go_peak` | Go Peak! |
| `ui.hud.fully_booked` | Fully booked! |
| `ui.hud.checked_out` | Checked out! |
| `ui.hud.breakaway` | Breakaway! |
| `ui.hud.protected` | Protected {sec}s |
| `ui.hud.combo` | {n} HIT! |
| `ui.hud.combo_big` | {n} HITS! WHAT A HULLABRAWL! |
| `ui.hud.tide_low` | Low tide |
| `ui.hud.tide_high` | High tide |
| `ui.hud.tide_in` | Tide coming in: {s}s |
| `ui.hud.tide_out` | Tide going out: {s}s |
| `ui.hud.level` † | Lv {level} |
| `ui.hud.treasure` † | {n} Shells |
| `ui.hud.shards` † | {n} Stamps |
| `ui.hud.sparks` † | {n} Messages in a Bottle |
| `ui.hud.quests` † | Daily Quests |
| `ui.quest.deal_damage` † | Deal {goal} damage |
| `ui.quest.perfect_blocks` † | Land {goal} Perfect Blocks |
| `ui.quest.kos` † | Get {goal} KOs |
| `ui.quest.active_minutes` † | Play for {goal} minutes |
| `ui.quest.awaken` † | Go Peak ×{goal} |
| `ui.feed.bot` | Bot |
| `ui.nameplate.rookie` | New Arrival |
| `ui.totem.action` | Change Charm |
| `ui.totem.object` | Charm Rack |
| `ui.wheel.title_ring` | Charm Ring |
| `ui.wheel.title_scrapbook` | Scrapbook |
| `ui.wheel.in_dev` | In development |
| `ui.wheel.trial` | Trial |
| `ui.wheel.free` | Free this week |
| `ui.wheel.get_hint` | Post a Wish for it, or find it on Sundry's Rack |
| `ui.wheel.shards` † | Stamps: {n} |
| `ui.wheel.convert` † | Trade Stamps for Mastery |
| `ui.wheel.mastery_combat_only` † | Mastery 21–30 is earned in battle |
| `ui.wheel.variant` † | Off-Duty Look |
| `ui.wheel.variant_base` † | On Duty |
| `ui.wish.title` † | Post a Wish |
| `ui.wish.make_1` † | Post 1 Wish |
| `ui.wish.make_10` † | Post a Mailbag (10 Wishes) |
| `ui.wish.odds` † | Odds |
| `ui.wish.pity_title` † | Guaranteed Delivery |
| `ui.wish.new` † | New! |
| `ui.wish.shards` | +{n} Stamps |
| `ui.wish.fakeout_note` | Now and then an Epic Wish lands a Rare stamp first and is then re-stamped Epic. It only ever goes up, and it never changes what you get. |
| `ui.wish.locked_paid` † | Locked paid Wishes: {n} |
| `ui.wish.auto10` † | Auto Mailbag |
| `ui.wish.secret` † | ??? |
| `ui.wish.zone_<rarity>` | §8.3.5 |
| `ui.shop.title` † | Sundry's Rack |
| `ui.shop.tab_rack` | Sundry's Rack |
| `ui.shop.tab_store` | Store |
| `ui.shop.refresh` † | New cards on the rack in {time} |
| `ui.shop.coming_soon` † | Coming soon |
| `ui.shop.sundry_line_1` | Have a browse. Everything on the rack has a fixed price. |
| `ui.shop.sundry_line_2` | New cards go up when the clock runs down. |
| `ui.shop.sundry_line_3` | No luck needed here. What you see is what you get. |
| `ui.shop.sundry_line_4` | Save your Shells for the one you really want. |
| `ui.shop.sundry_line_5` | Try a Charm in the Staff Room before you buy it. |
| `ui.shop.sundry_line_6` | Take your time. I'm not going anywhere. I'm a rack. |
| `ui.shop.balance` † | Your Shells: {n} |
| `ui.shop.confirm_in_dev` † | This Charm is still being made. Collect it now and play it when it's ready? |
| `ui.store.title` † | Store |
| `ui.store.mock` † | Test mode |
| `ui.store.confirm_4th` | You've bought Wishes several times in the last 24 hours. Buy again? |
| `ui.store.spent_week` † | Spent this week: R$ {robux} |
| `ui.settings.title` † | Settings |
| `ui.settings.tab_safety` † | Safety |
| `ui.settings.quiet_barks` † | Quiet barks |
| `ui.settings.damage_numbers` † | Damage numbers |
| `ui.settings.hold_toggle_run` † | Hold-to-toggle run |
| `ui.settings.character_look` † | Full Uniform |
| `ui.settings.style_gear` † | Charm Gear |
| `ui.settings.own_look_postcard` † | Postcard Look |
| `ui.settings.own_look_avatar` † | Roblox Avatar |
| `ui.settings.layout_editor_soon` | Touch layout editor coming soon |
| `ui.postcard.body.<id>` | `classic` Classic · `round` Round · `tall` Tall · `tiny` Tiny |
| `ui.results.environment` | The seaside |
| `ui.training.slot` | Dummy {n} |
| `ui.training.damage_numbers` † | Damage numbers |
| `ui.training.mode_idle` / `_block` / `_perfect_block` / `_m1_spam` / `_dash_spam` | Stand still / Block / Perfect Block / Attack / Dash |
| `ui.report.button` † | Report |
| `ui.report.title` † | Report a player |
| `ui.report.reason_exploit` / `_teaming` / `_harassment` / `_other` † | Cheating or exploits / Teaming / Harassment / Something else |
| `ui.report.send` † | Send report |
| `ui.toast.emotes_soon` | Emotes are coming soon! |
| `ui.toast.key_in_use` | That key is already in use. |
| `ui.tip.punch` | Attack 4 times in a row for a combo! |
| `ui.tip.block` | Hold block to guard. Time it right for a Perfect Block! |
| `ui.tip.dash` | Dash to dodge, or to close the gap. |
| `ui.tip.awaken` | Your Peak Meter is full. Go Peak! |
| `ui.tip.wish` | You have a Wish! Post it through Pip-Pip in the Wish menu. |
| `ui.vo.awaken` | Go Peak! |
| `ui.vo.fully_booked` | Fully booked! |
| `ui.vo.ko` | Checked out! |
| `ui.vo.legendary` | Gilt stamp! |
| `ui.notify.wish_epic` | Post for you: {styleId}! ({rarityId}) |
| `ui.notify.wish_legendary` | Gilt stamp! {who} got post from {styleId}! |
| `ui.notify.wish_mythic_global` | ❂ A Guest just had a reply from the Deep Bay! |
| `ui.notify.wish_cosmic_global` | A postcard just came back from beyond the Haar! |
| `ui.notify.cosmic_arrival` | A ferry horn sounds in the fog… {who} got post from {styleId}! |
| `ui.notify.level_up` | Level {level}! Enjoy your stay. |
| `ui.notify.mastery_up` | {styleId} Mastery {level}! |
| `ui.notify.daily_login` | Welcome back! Your daily post has arrived. |
| `ui.notify.streak_3` / `_5` / `_10` | {who} is on a 3-KO streak! / {who} is on a 5-KO streak! / {who} is the talk of the pier: 10 KOs! |
| `ui.notify.streak_ended` | Streak over! |
| `ui.notify.rookie_ended` | Your new-arrival protection has ended. Enjoy your stay! |
| `ui.notify.shop_bought` | Sold! Thank you kindly. |
| `ui.notify.purchase_granted` | Purchase complete. Thank you kindly! |
| `ui.notify.report_thanks` | Thank you. Your report has been sent. |
| `ui.notify.quest_done` † | Daily quest done! |
| `ui.error.data_load` | We couldn't load your data. Please rejoin. |
| `ui.error.session_ended` | Your game was opened somewhere else. |
| `ui.error.not_ready` | Still loading. Try again in a moment. |
| `ui.error.rate_limited` | Slow down a little! |
| `ui.error.restricted` | That isn't available for your account. |
| `ui.error.disabled` | That feature is turned off right now. |
| `ui.error.insufficient` | You don't have enough for that yet. |
| `ui.error.stale` | That changed. Refreshing… |
| `ui.error.cap_reached` | You've reached the limit. |
| `ui.error.timeout` | That took too long. Try again. |
| `ui.error.internal` | Something went wrong. |

Notes:
- Token names fixed here for Parts 3, 5 and 6: `ui.hud.protected {sec}`, `ui.hud.combo|combo_big {n}`, `ui.hud.tide_in|tide_out {s}` (whole seconds, Part 6 §6.9.5), `ui.hud.level {level}`, `ui.hud.treasure|shards|sparks {n}`, `ui.quest.<id> {goal}`, `ui.wheel.shards {n}`, `ui.wish.shards {n}`, `ui.wish.locked_paid {n}`, `ui.shop.refresh {time}`, `ui.shop.balance {n}`, `ui.store.spent_week {robux}`, `ui.training.slot {n}`, `ui.notify.wish_epic {styleId} {rarityId}`, `ui.notify.wish_legendary|cosmic_arrival {who} {styleId}`, `ui.notify.level_up {level}`, `ui.notify.mastery_up {styleId} {level}`, `ui.notify.streak_<tier> {who}`. `ui.wish.fakeout_note`, `ui.wish.zone_<rarity>`, `ui.notify.wish_mythic_global` and `ui.notify.wish_cosmic_global` take no token (the cross-server lines name no one, GDD §8.6).
- `ui.quest.<id>` has one key per `Progression.Quests.DailyPool[*].id` (Part 5 §5.5.9: `deal_damage`, `perfect_blocks`, `kos`, `active_minutes`, `awaken`). A task that adds a pool entry adds its line in the same task (L02).
- Parts 3 and 6 own the `<sub>` IDs of `ui.training.m1_<sub>`, `ender_<sub>` and `dash_<sub>`. Each one gets a `Base` value in the same task that introduces it (L02).
- Sundry's lines use the Part 6 key form `ui.shop.sundry_line_<n>`, n = 1…6. Part 6 picks n uniformly from 1 to the highest n for which `Skin.has` is true.
- `ui.postcard.body.<id>` has one key per `Appearance.PostcardStyles[*].id` (Part 5 §5.10). The skin-tone swatches (`Appearance.PostcardSkins`) are colour chips only and have no text keys.
- Part 6 keys carry Part 6's text (Part 6 §6.13 item 16), copied verbatim: among them `ui.hud.go_peak`, `ui.hud.fully_booked` and `ui.hud.checked_out` (the call-outs of §8.3.1), `ui.hud.combo_big` (Part 6 §6.9.5), the tide lines, the Charm Ring and Scrapbook titles, the shop tabs and the `ui.vo.*` barks (Part 6 §6.10). A wording change starts in Part 6 and is copied here in the same task.
- `ui.hud.combo_big` contains the title word, so it is title-bound (§8.7).

#### 8.3.8 Pack file skeleton

```lua
--!strict
-- src/Shared/Skin/Packs/Base.luau — the one shipped pack: every player-visible string.
-- Realm: shared. Pure: yes (require-free). Spec §8.3.
local S: { [string]: string } = {}
local function put(t: { [string]: string })
	for k, v in t do
		assert(S[k] == nil, "duplicate skin key " .. k)
		S[k] = v
	end
end

put({
	["style.attendant.name"] = "Sonny",
	["style.attendant.epithet"] = "Take a Seat",
	-- … every row of §8.3.2, §8.3.3, §8.3.4, §8.3.5 and §8.3.7 …
	["ui.game.title"] = "Hullabrawl",
	["ui.world.name"] = "The Wishaway Isles",
})

local STYLE_IDS = { "attendant", "lifeguard", "skimstone", "masseur", "bellhop", "angler", "kite", "boxcamera",
	"sandcastle", "scoop", "diver", "busker", "lamplighter", "cablecar", "dustsheet", "raincloud", "serpent", "ferry" }
local LEVEL_TITLES = { "Paddler", "Beachcomber", "Promenader", "Crowd-Pleaser", "Pier Performer", "Star Turn",
	"Showstopper", "Top of the Bill", "Isle Favourite", "Grand Entertainer" }
for i, title in LEVEL_TITLES do
	S["title.level_" .. tostring(i * 10)] = title
end
for _, id in STYLE_IDS do
	local name = S["style." .. id .. ".name"]
	S["variant." .. id .. ".v1.name"] = "Day Off"
	S["variant." .. id .. ".v2.name"] = "Evening Out"
	S["title.mastery_" .. id .. "_5"] = name .. "'s Pen Pal"
	S["title.master_" .. id] = name .. "'s Understudy"
end

return { id = "Base", strings = S }
```

`STYLE_IDS` repeats `Types.STYLE_IDS` because the pack is require-free; Lune `skin.base_shape` (§8.4.9) checks that the two lists are equal.

**Verify:**
- Lune `skin.base_shape`: the pack loads through the Part 7 loader, has `id == "Base"` and no duplicate keys, every key matches the grammar, and its `STYLE_IDS` equals `Types.STYLE_IDS`.
- Lune `ip.base_clean`: L04 and L07 on the real pack return 0 errors.
- Part 7 `skin.pack_key_coverage`.

---

### 8.4 Publish gate

#### 8.4.1 Profiles

| Profile | When | Requirements | Result line |
|---|---|---|---|
| **Dev publish** | H-11 to the private dev universe (`docs/agent/dev-place.json`) | Host gate green (Part 1 §1.4.2), which includes `lint_strings` in dev mode and `lint_retired_keys`. `Verify.verify()` ok. The human confirms the universe is **private**. Part 7 R2 is not required. | `[HB][GATE] publish-dev PASS commit=<sha7>` |
| **Public publish** | Any universe not listed in `Ip.DevUniverseIds`, or making the dev universe public | Everything in §8.4.4, plus Part 7 §7.7.6 R1–R4 | `[HB][GATE] publish PASS commit=<sha7> items=<n>/<n>` |

The final verdict combines Part 7's `[HB][GATE] publish-hook …` with this part's `[HB][GATE] publish-ip …`. Both must PASS.

#### 8.4.2 Rules data: the repo rules file and the local denylist

`lint_strings` reads two lists (Part 7 §7.5.1 owns the format, the matching and rules L04–L06).

**1. Repo rules file `tools/lint_strings.rules.json`** (committed; never under `src/`, so it never ships). It holds generic terms only and never a third-party name. Schema version 2 (Part 7 §7.5.1), plus two optional fields owned by this part: `list` (a label used in findings and by L07) and `keyPattern` (a Lua pattern; restricts a `pack`-scope term to pack keys that match it; alternatives are written with `|` and split on `|`).

```json
{
  "version": 2,
  "terms": [
    { "term": "slot", "list": "gambling", "scopes": ["pack", "meta"], "mode": "always", "keyPattern": "^ui%.wish%.|^ui%.store%.|^product%.|^pass%." },
    { "term": "spin", "list": "gambling", "scopes": ["pack", "meta"], "mode": "always", "keyPattern": "^ui%.wish%.|^ui%.store%.|^product%.|^pass%." },
    { "term": "jackpot", "list": "gambling", "scopes": ["pack", "meta"], "mode": "always", "keyPattern": "^ui%.wish%.|^ui%.store%.|^product%.|^pass%." },
    { "term": "roulette", "list": "gambling", "scopes": ["pack", "meta"], "mode": "always", "keyPattern": "^ui%.wish%.|^ui%.store%.|^product%.|^pass%." },
    { "term": "casino", "list": "gambling", "scopes": ["pack", "meta"], "mode": "always", "keyPattern": "^ui%.wish%.|^ui%.store%.|^product%.|^pass%." },
    { "term": "bet", "list": "gambling", "scopes": ["pack", "meta"], "mode": "always", "keyPattern": "^ui%.wish%.|^ui%.store%.|^product%.|^pass%." },
    { "term": "gamble", "list": "gambling", "scopes": ["pack", "meta"], "mode": "always", "keyPattern": "^ui%.wish%.|^ui%.store%.|^product%.|^pass%." },
    { "term": "gambling", "list": "gambling", "scopes": ["pack", "meta"], "mode": "always", "keyPattern": "^ui%.wish%.|^ui%.store%.|^product%.|^pass%." },
    { "term": "thousand year", "list": "loretime", "scopes": ["pack", "meta"], "mode": "always" },
    { "term": "millennium", "list": "loretime", "scopes": ["pack", "meta"], "mode": "always" },
    { "term": "millennia", "list": "loretime", "scopes": ["pack", "meta"], "mode": "always" }
  ]
}
```

| List | Meaning | Rule |
|---|---|---|
| `gambling` | Gambling vocabulary on Wish, Store, product and pass text and in metadata (GDD §8.8; Part 6 `ui.wish_no_gambling_terms`). Plural forms match automatically (Part 7 matching) | L07 |
| `loretime` | Lore counts time in summers (§8.3.1), so thousand-year phrasing never appears in player text or metadata | L04 |

**2. Local denylist `tools/denylist.local.txt`** (gitignored; never committed, quoted or copied; Part 7 §7.5.1 format: one term per line, `#` comments).
- **Who supplies it:** the owner or counsel. Counsel refreshes it before each public gate and whenever a §8.6 trigger adds a name (TW-10).
- **What it holds** (this spec never lists entries): names of existing shows, films, games, characters, places, catchphrases, invented slang, episode and song titles, studios and brands that our work must never echo, starting with the neighbours named in the T9 check (§8.5.3); their spelling variants and sound-alikes; and descriptor phrases that would steer an art prompt toward an existing character's signature look.
- **Where it applies:** every scope. It is required for a public publish (Part 7 L05) and optional in dev, where its absence prints `W-NO-DENYLIST`.
- **Findings** name the entry by its line number only (Part 7 §7.5.1), so logs, evidence lines and blockers never contain a denylisted name.

**Scopes this part adds** to Part 7's `pack` and `src`:
- `meta`: the text fields of `PUBLISH_METADATA.md` (L09).
- `prompt`: the prompt and query columns of `ASSET_PROVENANCE.md` (L10), and `--text … --scope prompt`.
- `tree`: the Studio dump written by `Verify.publishGate` (§8.4.6; L11).
- `text`: ad-hoc text checked with `--text … --scope text` (commit messages, PR titles, branch names, drafts).

Counsel may add rules-file terms; the agent adds them only on a human instruction recorded in `PROGRESS.md`, and never adds a third-party name to the rules file.

#### 8.4.3 Lint rules: `tools/lint_strings.luau` and `tools/lint_retired_keys.luau`

**`lint_strings`.** Part 7 owns the mechanism and rules L01–L06. This part supplies the data semantics of L02 and L04 and adds rules L07–L15 to the same tool.

Flags added by this part (Part 7 owns `--public`, `--rules`, `--denylist`):
- `--tree <json>`: the Studio dump (§8.4.6); adds L11 and the `tree` scope of L04.
- `--meta <path>`: default `docs/agent/PUBLISH_METADATA.md`.
- `--text "<s>" --scope <prompt|meta|text>`: ad-hoc check of prompts, search queries, commit messages and drafts.
- `--dry-run`: used only by the agent procedure `ip.gate_dry_run` (§8.4.7). These findings print the warning `W-DRYRUN` instead of an error: L05 missing denylist (Part 7), L09 missing or placeholder metadata, L13 missing or `pending` blind-test rows, and L15 missing or uncleared title clearance. All other rules are unchanged: a term hit in a real metadata field, or a `fail` row, stays an error. A run that printed any `W-DRYRUN` is never valid for H-11.

**Output.** Lines follow Part 7: `[HB][LINT] <path>:<line> <rule> <detail>`, and the summary `[HB][LINT] lint_strings SUMMARY errors=<n> warnings=<m>`. The `--public --tree` run also prints `[HB][GATE] publish-ip PASS|FAIL commit=<sha7> errors=<n> warnings=<m>` (`warnings` counts `W-NO-DENYLIST` and `W-DRYRUN`). Exit codes follow Part 7 (0 clean, 1 errors, 2 usage).

| Rule | Mode | Check |
|---|---|---|
| L02 (data) | always | (a) Every referenced key exists in `Base`. (b) Every family in the registry below is complete in `Base`. (c) A value's `{tokens}` are exactly the key's tokens (§8.3.7 notes); a key with no listed token has none. |
| L04 (data) | always / `--public` | Every rules-file list except `gambling` (§8.4.2) by `scopes` and `keyPattern`, and every denylist entry, in the scopes of Part 7 L04 plus `prompt` and `text` (always) and `meta` and `tree` (under `--public`). `tools/lint_strings.rules.json` and `tools/retired_keys.txt` are not scanned. |
| L07 | always | The `gambling` list on `ui.wish.*`, `ui.store.*`, `product.*` and `pass.*` values, and on metadata under `--public` |
| L08 | always; metadata part under `--public` | Every Part 5 `ProductKey` and `PassKey` has `product.<key>.name` / `pass.<key>.name` in `Base`, and every badge listed in `PUBLISH_METADATA.md` has `badge.<key>.name`. Under `--public`, each `Base` name equals the matching name in `PUBLISH_METADATA.md`. |
| L09 | `--public` | `PUBLISH_METADATA.md` exists and parses (§8.4.5), with no empty field and no `<…>` placeholder value. `target_universe_id ∉ Ip.DevUniverseIds` and `≠ dev-place.json.universeId`. Every text field passes the `meta` scope. Under `--dry-run`, a missing file and empty or placeholder values are `W-DRYRUN`. |
| L10 | always | Every prompt and query cell in `docs/agent/ASSET_PROVENANCE.md` passes the `prompt` scope |
| L11 | `--public --tree` | Every string in the Verify dump passes the `tree` scope, and the dump's `problems` list is empty |
| L12 | always | `Defaults.Ip.DevUniverseIds` ⊆ { `dev-place.json.universeId` }, and `Defaults.Ip.DevGroupId == dev-place.json.groupId` |
| L13 | `--public` | Blind tests: for every required (test, subject) pair of §8.4.4, the newest row in `RECOGNIZABILITY.md` is `pass` and meets the pass rule of §8.5.3; every non-`pending` row passes the provenance check (§8.4.5). Under `--dry-run`, missing or `pending` rows are `W-DRYRUN`; a `fail` row stays an error. |
| L14 | `--public` | `Defaults.Perf.Audio.TtsInStudioOnly == true` (Part 6) |
| L15 | `--public` | `TITLE_CLEARANCE.md` exists and is valid (§8.4.5): `status: cleared`, every search box checked, `title` equals `ui.game.title`, `world_name` equals `ui.world.name`, the metadata `experience_title` contains `title` as a whole word, and the provenance check passes. Under `--dry-run`, every L15 finding except a failed provenance check is `W-DRYRUN`. |

**Key-family registry** (inside the lint; parts register new dynamic families in the same task):

| Family | Expansion |
|---|---|
| `style.<id>.name`, `style.<id>.epithet` | 18 style IDs |
| `move.<id>.<slot>.name` | implemented style IDs (from `Styles`, through the Part 7 loader) × `m1, passive, q, 1, 2, 3, 4, awaken, a1, a2, a3, a4` |
| `map.<mapId>.name`, `map.hub.zone.<zoneId>` | `Types.MAP_IDS` (3 maps); the 6 hub zones of Part 6 §6.3.1 |
| `flair.<id>.name`, `rarity.<id>.name` | §0.4 IDs |
| `variant.<id>.v1.name`, `variant.<id>.v2.name` | 18 IDs |
| `title.level_<n>`, `title.mastery_<id>_5`, `title.master_<id>` | n = 10…100; 18 IDs |
| `product.<key>.name`, `pass.<key>.name` | Part 5 `ProductKey`, `PassKey` |
| `ui.error.<ErrCode>` | the 11 player-facing error keys listed in Part 2 §2.9.2 (`data_load` … `internal`; §8.3.7) |
| `ui.notify.wish_<rarity>_global`, `ui.notify.streak_<tier>` | rarity ∈ {mythic, cosmic}; tier ∈ {3, 5, 10} |
| `ui.wish.zone_<rarity>` | rarity ∈ {common, uncommon, rare, epic, legendary, mythic, cosmic} (Part 5 §5.2.13) |
| `ui.training.mode_<mode>` | idle, block, perfect_block, m1_spam, dash_spam |
| `ui.hud.tide_<phase>` | low, high, in, out (Part 6 §6.9.5) |
| `ui.vo.<line>` | awaken, fully_booked, ko, legendary (Part 6 §6.10) |
| `ui.postcard.body.<id>` | `Appearance.PostcardStyles[*].id` (Part 5 §5.10) |
| `ui.shop.sundry_line_<n>` | n = 1…6 |
| `ui.quest.<id>` | `Progression.Quests.DailyPool[*].id` (Part 5 §5.5.9) |
| `ui.report.reason_<reason>` | exploit, teaming, harassment, other (Part 6 §6.9.12) |

Any string literal in `src/` that ends with `"."` and is followed by `..` must match a registered family prefix; otherwise the error is `L02 unregistered-family`.

**`lint_retired_keys`.** Part 7 §7.5.6 owns the mechanism, the rules R01–R05 and the format of the key list `tools/retired_keys.txt`. This part owns the policy:
- **What the list covers:** retired style IDs (found only as style-ID literals or move-ID prefixes, never as plain English words and never in the Wish system), retired map, finish, status and archetype keys, the retired project word, and the retired log, instance, config and attribute prefixes. Plain keys only: no descriptions, and no current key beside a retired one.
- **Who supplies it:** the owner, from counsel's archive. Like the local denylist, the list is **gitignored**: it is never committed, and the agent never writes, prints, quotes or copies it (R-IP-05). The owner adds keys to it. Findings print only `<path>:<line> retired-key`, never the matched text (output line format: Part 7 §7.5.6).
- **When the list is missing:** the lint prints `SKIP no retired_keys list` and exits 0, so dev work can proceed. A public publish requires the list to be present and the lint to PASS (§8.4.4 item 3).
- **Where it runs:** host gate step 8 (Part 1 §1.4.2), CI, Part 7 R1, and on the Studio tree dump in the public procedure (§8.4.7 step 6, `--tree build/ip_tree.json`), which covers instance names, attributes and strings that replicate to clients.
- **What a finding means:** use the current key of §0.4 instead. A finding that a rename cannot fix opens a blocker of type `ip` (R-IP-09).

#### 8.4.4 Requirements for a public publish

| # | Requirement | Checked by |
|---|---|---|
| 1 | Host gate green in dev mode (Part 1 §1.4.2), including `lint_strings` and `lint_retired_keys` | `tools/check.luau` |
| 2 | `lint_strings --public` clean, with the current local denylist present | L01–L15 (L05 needs the denylist) |
| 3 | No retired key in files or in the Studio tree dump, with the owner's `tools/retired_keys.txt` present (a `SKIP no retired_keys list` run fails this item) | `lint_retired_keys`, `lint_retired_keys --tree` |
| 4 | Every player-reachable string in the place is clean | `Verify.publishGate` (§8.4.6) + L11 |
| 5 | Metadata final and clean; product and pass names match `Base` | L09, L08; §8.4.8 |
| 6 | Asset provenance complete: no ripped or fan-made assets, no AI or Creator Store likeness, artist assets under written assignment, every likeness asset reviewed | Part 7 M03 + M09–M13 (§8.5.1) |
| 7 | Blind tests passed and signed (rows below) | L13 |
| 8 | Title cleared by counsel and signed by the owner | L15 |
| 9 | Placeholder TTS is Studio-only | L14 |
| 10 | The target universe belongs to the production group; the dev universe is still private | HP-01 (§8.2.6 Groups) |
| 11 | Part 7 R1–R4 | Part 7 §7.7.6 |
| 12 | Human checklist HP-01…HP-07 | §8.4.7 |

**Required blind-test rows** (L13; the newest row for each pair must be `pass`):
- `lineup`: T7.
- `style:<id>` for every implemented style: T8 and T11.
- `pair:attendant+lifeguard`: T8 and T11.
- `npc:mascot` (Pip-Pip): T8.
- `map:<mapId>` for every built map: T8.
- `meta:icon` and every `meta:thumbnail_<n>` in `PUBLISH_METADATA.md`: T8.
- `set:neighbours`: T9.
- `set:lineage`: T10.

#### 8.4.5 Human-signed and shared files

The templates live in `docs/agent/templates/`: `TITLE_CLEARANCE.template.md`, `PUBLISH_METADATA.template.md`, `RECOGNIZABILITY.template.md` and `TRIGGER_WATCH.template.md`. The agent writes the templates. `RECOGNIZABILITY.template.md` is needed first (the M0 gate's T11 capture prep, Part 7 §7.7.2 A14); the other three come with the publish-gate work.
- The human copies `TITLE_CLEARANCE.template.md` to `docs/agent/TITLE_CLEARANCE.md`, fills it in with counsel, and commits it.
- The agent creates `RECOGNIZABILITY.md` and `TRIGGER_WATCH.md` from their templates. In `RECOGNIZABILITY.md` it only appends `pending` rows (§8.5.3).
- The agent may draft `PUBLISH_METADATA.md`; the human owns the final text.
- The local denylist and the retired-key list have no template: the owner or counsel supplies the denylist (§8.4.2), and the owner supplies the retired-key list (§8.4.3).

**Who can write the human-owned files.** The example permissions file (`docs/agent/claude-settings.example.json`, Part 1 §1.2.4) denies Edit/Write on these files (these four entries of its `permissions.deny` array):

```json
[
  "Edit(docs/agent/TITLE_CLEARANCE.md)", "Write(docs/agent/TITLE_CLEARANCE.md)",
  "Edit(tools/denylist.local.txt)", "Write(tools/denylist.local.txt)"
]
```

`RECOGNIZABILITY.md` is not on the list, because the agent appends `pending` rows to it. The owner's retired-key list `tools/retired_keys.txt` is not on it either. The example file keeps these four entries; Q-P8-7 asks the owner whether to add two more for the list. SafeWrite refuses the list, and R-IP-05 forbids writing it. The deny entries do not cover `lune run`, which can write any file. They are a guard against accidents, not a mechanism the agent cannot bypass. **The binding controls are HP-06 plus the provenance check below.** As a second guard, every Lune host tool writes files only through one helper that refuses the protected paths:

```lua
--!strict
-- tools/lib/SafeWrite.luau (Lune) — the only file writer for host tools. Spec §8.4.5.
-- Refuses the human-signed title file and the two owner-supplied lint lists (the local denylist and the
-- retired-key list) by base name, whatever the directory or case.
-- A guard, not a boundary (HP-06 and the provenance check are the binding controls).
local fs = require("@lune/fs")

local PROTECTED: { [string]: boolean } = {
	["title_clearance.md"] = true,
	["denylist.local.txt"] = true,
	["retired_keys.txt"] = true,
}

local SafeWrite = {}

function SafeWrite.isProtected(path: string): boolean
	local unix = (string.gsub(path, "\\", "/"))
	local base = string.lower(string.match(unix, "([^/]*)$") or "")
	return PROTECTED[base] == true
end

function SafeWrite.writeFile(path: string, contents: string)
	if SafeWrite.isProtected(path) then
		error(`[HB][LINT] {path} write refused: human-owned or owner-supplied file (Spec §8.4.5)`)
	end
	fs.writeFile(path, contents)
end

return SafeWrite
```

Lune test `ip.no_tool_writes_signed_files`: (a) no Lune script under `tools/**` (excluding the Studio snippets `tools/studio/**` and the plugin source `tools/plugin/HullabrawlTestHarness/**`) calls `fs.writeFile`, `fs.copy` or `fs.move` except `tools/lib/SafeWrite.luau`; (b) `SafeWrite.writeFile` raises for `docs/agent/TITLE_CLEARANCE.md`, `./docs/agent/title_clearance.md`, `tools\denylist.local.txt`, `x/../DENYLIST.LOCAL.TXT` and `tools/Retired_Keys.txt`, and writes `docs/agent/templates/TITLE_CLEARANCE.template.md` in a temp directory.

**Provenance check (L13 and L15).** It runs whenever the rule runs. A failure is an error `[HB][LINT] <path>:<line> L13|L15 provenance <detail>`, and the agent then opens a `safety` blocker (R-IP-06).
1. The file is committed and unchanged: `git status --porcelain -- <file>` prints nothing.
2. **`TITLE_CLEARANCE.md`:** for every line, `git blame --porcelain -- <file>` names a commit whose author email equals `ownerGitEmail` in `docs/agent/dev-place.json` (recorded at H-01 and confirmed by the human; case-insensitive), and whose message (`git log -1 --format=%B <sha>`) has no line starting with `Co-Authored-By: Claude` (case-insensitive).
3. **`RECOGNIZABILITY.md`:** the same two checks for every row whose Result is not `pending`, using `git blame --porcelain -L <line>,<line> -- <file>` on that row's line. Agent-appended `pending` rows are not checked.

Lune test `ip.signed_file_provenance` (temporary git repository fixture): an owner-authored `TITLE_CLEARANCE.md` passes; an agent-authored line, a commit with the `Co-Authored-By: Claude` trailer and an uncommitted file each give the provenance error; a `pass` row last changed in an agent commit gives the error; an agent-appended `pending` row gives none.

**Parse rules** (`TITLE_CLEARANCE.md` and `PUBLISH_METADATA.md`):
- `key: value` lines (`^([a-z_0-9.]+):%s*(.*)$`).
- `|` starts a block whose lines are indented by two spaces.
- Checkboxes are `- [x] <ID> <text>` or `- [ ] <ID> <text>`.
- Dates are `YYYY-MM-DD`.
- An unknown key is an error, so typos fail loudly.

**`docs/agent/TITLE_CLEARANCE.md`** (required for every public publish; template `TITLE_CLEARANCE.template.md`; procedure §8.7):

```markdown
# TITLE_CLEARANCE — title knockout and counsel sign-off
title: Hullabrawl
world_name: The Wishaway Isles
status: <cleared | not-cleared | pending>
knockout_date: <YYYY-MM-DD>
counsel: <name or firm>
counsel_ref: <opinion or matter reference>
classes: <e.g. 9, 16, 25, 28, 41>
signed_by: <full name>
date: <YYYY-MM-DD>

## Searches (all required for status cleared; §8.7)
- [ ] K1 Roblox experience search, group names and Marketplace names, for the title and its spelling variants
- [ ] K2 Steam, Apple App Store and Google Play
- [ ] K3 USPTO classes 9, 41, 28, 25 and 16, including sound-alike and look-alike marks in the games space
- [ ] K4 WIPO Global Brand Database
- [ ] K5 Domains (.com, .gg, .net) and social handles (YouTube, TikTok, X, Instagram, Discord, Roblox group)
- [ ] K6 Secondary names: the world name, place names, the mascot and NPC names, the 18 style names, currencies and passes
- [ ] K7 The logo lockup is original (lettering, crest and layout)
notes: |
  A summary of the findings that names no third-party mark. Counsel keeps the details.
```

**Validation (L15):**
- the file parses with no unknown key, and every field is non-empty;
- `status: cleared`, and boxes K1–K7 are checked;
- `knockout_date ≤ date ≤ today`;
- `title == Skin.text("ui.game.title")` and `world_name == Skin.text("ui.world.name")` (read from `Base`), and the metadata `experience_title` contains `title` as a whole word;
- the provenance check above passes.

**`docs/agent/PUBLISH_METADATA.md`** (every public publish; the agent may draft it, and the human owns the final text):
- `target_universe_id`.
- `experience_title`, `experience_description` (block), `genre`, `tags`, `search_keywords`.
- `update_name`.
- `icon_description`, and `thumbnail_<n>_description` for every thumbnail.
- `group_name`, `group_description` (block), `discord_server_name`, `social_handles`.
- `ad_copy` (block, or `none`).
- `press_line` (block): the counsel-approved answer to "what inspired it?" (§8.4.8).
- `product.<key>.name` and `product.<key>.description` for every live product; the same for `pass.<key>.*` and `badge.<key>.*`.

**`docs/agent/RECOGNIZABILITY.md`**: format in §8.5.3. **`docs/agent/TRIGGER_WATCH.md`**: format in §8.6.1.

#### 8.4.6 Studio step: `Verify.publishGate`

This is an addition to Part 1's `Verify.luau`. It is invoked through `run_builder.luau` with `{"builder":"Verify","fn":"publishGate"}`. Part 1's snippet gains one branch: `elseif A.fn == "publishGate" then return mod.publishGate(A.opts or {})`.

```lua
--!strict
-- src/DevTools/Builders/Verify.luau — Part 8 addition. Edit only. Spec §8.4.6.
-- Returns the Studio half of the publish gate: every player-reachable string in the place.
-- Matching runs on the host (L04 `tree` scope, L11, lint_retired_keys --tree), so no term lists live in the place.
type GateString = { path: string, kind: string, value: string }
type GateResult = { ok: boolean, problems: { string }, strings: { GateString } }

local SCAN = { "Workspace", "ReplicatedStorage", "ReplicatedFirst", "StarterGui", "StarterPack", "StarterPlayer",
	"Lighting", "SoundService", "Teams" }
local MAX_STRINGS = 4000 -- (tuning default) keeps the MCP response small; see spike S-P8-3

-- `M` is Verify.luau's module table (Part 1 §1.6.7).
function M.publishGate(_opts: { [string]: any }?): GateResult
	local problems: { string } = {}
	local strings: { GateString } = {}
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
				add(path .. "@" .. an, "attrName", an)
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
	return { ok = #problems == 0, problems = problems, strings = strings }
end
```

Members used, all verified with read security None in the docs:
- `Instance:GetDescendants`, `GetAttributes`, `GetFullName`, `IsA`;
- `TextLabel.Text`, `TextButton.Text`, `TextBox.Text`, `TextBox.PlaceholderText`;
- `ProximityPrompt.ActionText`, `ProximityPrompt.ObjectText`;
- `StringValue.Value`, `Tool.ToolTip`, `LocalizationTable:GetEntries`.

Script sources are excluded, because the host lints cover them. The agent writes the returned `res` JSON to `build/ip_tree.json` (gitignored) with its file tool.

#### 8.4.7 Publish procedure

**Dev publish (H-11 to the dev universe).**
1. Host gate green.
2. `run_builder {"builder":"Verify","fn":"verify"}` returns ok.
3. The agent prints `[HB][GATE] publish-dev PASS commit=<sha7>` and posts a `human-step` blocker for H-11.

**Public publish:**
1. **Owner decision.** The owner names the target universe in `PROGRESS.md`. Before the agent starts, the humans provide:
   - the final `PUBLISH_METADATA.md` (the agent may draft it);
   - a signed `TITLE_CLEARANCE.md`, committed by the owner;
   - signed blind-test results in `RECOGNIZABILITY.md` for every required row of §8.4.4, committed by the owner;
   - the current local denylist from counsel and the owner's retired-key list `tools/retired_keys.txt`, both on the publishing machine.
2. Agent: `lune run tools/check.luau`, the host gate. Expect `[HB][CHECK] SUMMARY pass=10 fail=0` (every tool exists by then; Part 1 §1.4.2).
3. Agent: `lune run tools/lint_strings.luau --public` (L01–L10 and L12–L15; L11 needs `--tree`) and `lune run tools/lint_retired_keys.luau`. Both exit 0, and `lint_retired_keys` prints no `SKIP no retired_keys list` line.
4. Agent: `run_builder {"builder":"BuildAll","fn":"build","opts":{"reset":true}}`, then `verify` returns ok.
5. Agent: `run_builder {"builder":"Verify","fn":"publishGate"}`. Save the result to `build/ip_tree.json`.
6. Agent: `lune run tools/lint_strings.luau --public --tree build/ip_tree.json` (adds L11 and the `tree` scope of L04) and `lune run tools/lint_retired_keys.luau --tree build/ip_tree.json`. Expect `[HB][GATE] publish-ip PASS commit=<sha7> …`.
7. Agent: Part 7 §7.7.6 R2–R4 (steps 3 and 6 are R1). In M0 and M1-G, R2 is **expected to FAIL** on placeholders, so a public publish is blocked; run the dry run below instead.
8. Agent: print `[HB][GATE] publish PASS|FAIL commit=<sha7> items=<n>/<n>`. If step 3 or 6 printed any `W-DRYRUN`, stop here: no H-11 blocker. Otherwise post a `human-step` blocker for H-11 that includes:
   - the gate lines;
   - any `W-NO-DENYLIST` warning (it must not appear on a public run);
   - the human checklist below.
9. **Human (H-11, public variant).** Complete HP-01 to HP-07. Then:
   1. In the agent's Studio session: **File ⟩ Save to File As…** `build/release-<sha7>.rbxl`.
   2. Open that file in a **separate** Studio window.
   3. Publish it to the target universe.
   4. Close the window.

   Never publish the dev session document to another universe: the agent's preflight pins the dev place (Part 1 §1.3).
10. Agent, after the human confirms: record the publish (date, commit, universe) in `PROGRESS.md` and `docs/agent/TRIGGER_WATCH.md`.

**Dry run (`ip.gate_dry_run`, agent procedure).** This exercises the public procedure while a real public publish is impossible (§8.1.5). It is an **agent procedure**, not a Studio scenario: it needs host commands, Edit-mode builders and git, so it never appears in a `studio(…)` selector.
0. **Setup.** On the stage branch, record the sync `agg` (`lune run tools/sync_manifest.luau`). Create the throwaway branch `release/<YYYY-MM-DD>-dryrun` from the stage head. On it, draft `docs/agent/PUBLISH_METADATA.md` from its template with the placeholder `target_universe_id: 1`, and commit it on that branch only. Nothing under `src/` changes.
1. **Run** steps 2–8, passing `--dry-run` to both `lint_strings --public` runs (steps 3 and 6). L05, L09, L13 and L15 then print `W-DRYRUN` instead of failing. If the owner has not supplied `tools/retired_keys.txt` yet, `lint_retired_keys` prints `SKIP no retired_keys list`; the dry run records it as a warning, not a failure. Also run `studio(ip.no_missing_keys_tour) ≥ 1`. Step 8 prints the verdict and posts no H-11 blocker.
2. **Restore.** Switch back to the stage branch, run the sync gate, then delete the local dry-run branch (it is never pushed).

Expected evidence:
- `[HB][GATE] publish-ip PASS commit=<sha7> errors=0 warnings=<m>` (`W-DRYRUN` included in `m`);
- `[HB][GATE] publish FAIL commit=<sha7> items=<n>/<n>` with R2 failing (graybox placeholders);
- `[HB][GATE] publish-hook FAIL items=3/4`, recorded as expected only when R1 (run with `--dry-run`), R3 and R4 pass; R4 is taken from the enclosing gate card's own automated results (Part 7 §7.7.3 A1–A12, §7.7.6), so it is not circular;
- the post-run sync `agg` equals the pre-run `agg`.

**Human checklist HP (public publish):**

| ID | Check (Creator Hub / Studio) |
|---|---|
| HP-01 | The target universe is not the dev universe and belongs to the production group. The dev universe is still private (§8.2.6). |
| HP-02 | The experience title, description, genre, tags, search keywords, icon and thumbnails match `PUBLISH_METADATA.md` exactly and follow §8.4.8. |
| HP-03 | Developer products, passes and badges in the target universe use the names and descriptions in `PUBLISH_METADATA.md`. Prices are signed off (H-09). |
| HP-04 | The group name, description, banner and emblem, the Discord name and the social handles and bios carry no third-party name or art, and match the metadata. |
| HP-05 | The content-maturity questionnaire is updated (GDD §16.1); the target label is Mild. |
| HP-06 | The owner personally committed `TITLE_CLEARANCE.md` (commit `<sha>`) and every non-`pending` result row in `RECOGNIZABILITY.md`, and the local denylist on this machine is the version counsel last sent (date). The owner's `tools/retired_keys.txt` is present on this machine. This check, with the provenance check, is the binding control (§8.4.5). |
| HP-07 | No public footage, thumbnail or creator kit shows a style whose newest T11 row is not `pass` (§8.4.8). |

#### 8.4.8 Metadata rules

These apply to every public build. The design rules are GDD §5.6 and §19.1; this table is how the gate checks them.

| Field | Rule |
|---|---|
| Title | "Hullabrawl", only once `TITLE_CLEARANCE.md` says `cleared` (L15). Plain title text, plus at most a bracketed update tag (see Update names); the logo lockup is "HULLABRAWL · Greetings from the Wishaway Isles" in the original sign-painter display face. Never a third-party name or a title borrowed from another work. |
| Description | No denylist entry (L09). No "inspired by", "for fans of", "spiritual successor" or "based on" phrasing, and no show, film, game or decade of cartoons named as an influence. Our own characters, places and fiction only. |
| Tags, genre, search keywords | Generic: battlegrounds, PvP, fighting, brawl, cartoon, toon, RNG. No brand, character or other game's name. No keyword stuffing; Roblox lists "misleading or mismatched metadata" as a cause of lost impressions [Docs `discovery-faq.md`]. |
| Icon | Pip-Pip mid-yell with a boxing glove punching out of its slot, impact lines and postcards flying; three flat colours; no text; readable at 64 px; it says "fighting game" at a glance. `meta:icon` must pass T8 (L13). |
| Thumbnails | Five, all real gameplay (GDD §5.6): (1) Pip-Pip punching a postcard out over a beach brawl; (2) a gilt Legendary stamp landing; (3) the tide coming in over twenty avatars brawling on the beach; (4) Plunge's Perfect Ten scorecards; (5) Pocket's Wrong Room! trunk landing. A style or move appears only after its newest T11 row is `pass` (HP-07). Mothball and her Getaway never appear in the icon, the thumbnails or key art. Every `meta:thumbnail_<n>` must pass T8 (L13). |
| Update names | Same as Title. The store title keeps the base name with a bracketed update tag, for example "[NEW STYLE] Hullabrawl", "[CLOSED FOR THE SEASON] Hullabrawl" or another season name (GDD §19.1). Never another work's name or title. |
| Product, pass and badge names and descriptions | The `Base` strings (§8.3.6), lint-clean (L08, L09). |
| Group, Discord, YouTube, TikTok, X and Instagram names and bios | Same as Description. No third-party art in banners or emblems. |
| Ads, sponsored listings, creator briefs | Same as Description. Seed creators with our characters. Never run "which character is this?" or "what does this remind you of?" posts, and never repost fan comparisons. |
| Press line | If asked what inspired the game, use only the counsel-approved line (`press_line`; GDD §5.6, Appendix F15): "Old seaside postcards, music hall and travel posters. Our world is our own." |

#### 8.4.9 Verify

**Lune (`tests/ip/*.spec.luau`; fixtures in `tests/ip/fixtures/`, invented words only, with the fixture denylist `tests/ip/fixtures/denylist.txt` passed through `--denylist`):**

| ID | Layer | Check | Spec | Phase | Milestone |
|---|---|---|---|---|---|
| `skin.base_shape` | lune | §8.3.8 Verify | §8.3.8 | P1 | M0 |
| `ip.rules_schema` | lune | The rules file parses (version 2). Every term is lowercase. Every `scopes` entry is one of `pack`, `src`, `meta`, `prompt`, `tree`, `text`. Every `keyPattern` compiles and appears only on a term whose scopes include `pack`. Every term with `list = "gambling"` carries the §8.4.2 `keyPattern`. | §8.4.2 | P1 | M0 |
| `ip.base_clean` | lune | L04 (rules file and fixture denylist) and L07 on the real `Base` pack return 0 errors | §8.3 | P1 | M0 |
| `ip.gambling_terms` | lune | Fixture `ui.wish.title = "Spin!"` gives an L07 error; the same word under a `ui.hud.*` key gives none | §8.4.3 | P1 | M0 |
| `ip.lore_time` | lune | Fixture `ui.tip.wish` holding "a thousand-year wait" gives an L04 `rules term` finding; "365 summers" gives none | §8.4.2 | P1 | M0 |
| `ip.prompt_cells` | lune | A fixture `ASSET_PROVENANCE.md` row whose prompt holds a fixture-denylist term gives an L10 error that names the entry number, never the term; `--text "<s>" --scope prompt` gives the same finding | §8.4.3 | P1 | M0 |
| `ip.dev_ids_match` | lune | L12: a `DevGroupId` or a `DevUniverseIds` entry that differs from the fixture `dev-place.json` gives an error | §8.2.6 | P1 | M0 |
| `ip.prov_rules` | lune | Fixture manifest entries cover each M09–M13 violation (§8.5.1) | §8.5.1 | P1 | M0 |
| `ip.product_names` | lune | A fixture `ProductKey` without `product.<key>.name` gives L08; under `--public`, a metadata product name that differs from `Base` gives L08 | §8.4.3 | P8 | M1-G |
| `ip.metadata_parse` | lune | L09: a missing file, an empty field, a `<…>` placeholder and a `target_universe_id` in `Ip.DevUniverseIds` each give an error; a fixture-denylist term in a field gives an L04 error | §8.4.5 | P8 | M1-G |
| `ip.recog_rows` | lune | L13 with fixture `Styles`: the newest row per (test, subject) wins; a missing or `pending` required row and a `fail` row each give an error; a `pass` row with fewer than 10 testers or with 2 or more same-work hits gives an error | §8.5.3 | P8 | M1-G |
| `ip.title_clearance_parse` | lune | L15: a valid file passes; `status: not-cleared`, an unchecked K box, `title` ≠ `ui.game.title`, an `experience_title` without the title, and an unknown key each give an error | §8.4.5 | P8 | M1-G |
| `ip.signed_file_provenance` | lune | §8.4.5 provenance cases | §8.4.5 | P8 | M1-G |
| `ip.no_tool_writes_signed_files` | lune | §8.4.5 (a) static scan of `tools/**` Lune scripts and (b) `SafeWrite.writeFile` refusals | §8.4.5 | P8 | M1-G |
| `ip.dry_run_warnings` | lune | With fixture inputs (placeholder metadata, a `pending` row, no `TITLE_CLEARANCE.md`, no denylist): without `--dry-run`, L09, L13, L15 and L05 give errors; with `--dry-run` they give `W-DRYRUN` warnings and exit 0; a fixture-denylist hit in a metadata field stays an error under `--dry-run` | §8.4.3 | P8 | M1-G |

Part 7 owns the mechanism tests `lint.strings_clean`, `lint.strings_detects_fixture`, `lint.denylist_public_required`, `lint.denylist_stays_local`, `lint.retired_keys_clean` and `lint.retired_keys_detects_fixture` (Part 7 §7.4.7).

**Studio scenarios:** `ip.dev_universe_members_only` and `ip.no_missing_keys_tour` (§8.2.8). The blind-test capture scenarios `look.gear_silhouette` and `look.clip_stage` are Part 7's (§8.5.3).

**Agent procedure (not a Studio scenario):** `ip.gate_dry_run` (§8.4.7), run twice in a row in the publish-gate card (spike S-P8-1). Expected evidence is listed in §8.4.7; `studio(ip.no_missing_keys_tour) ≥ 1` runs inside it.

---

### 8.5 Asset provenance and blind tests

#### 8.5.1 Provenance fields in the Assets manifest

`AssetEntry` (Part 2 §2.9.1) gains the optional field `prov`:

```lua
--!strict
-- Addition to src/Shared/Assets.luau types. Spec §8.5.1.
export type AssetProvenance = {
	author: string, -- "agent" | "roblox" | "roblox-ai" | "<creatorId>" | "<artist legal name>"
	origin: string, -- "graybox" | "builtin" | "creator-store:<assetId>" | "ai:<ASSET_PROVENANCE row key>" | "artist:<delivery ref>"
	license: string, -- "original" | "roblox-builtin" | "creator-store-on-platform" | "roblox-generative-ai" | "assignment:<contract ref>"
	likeness: boolean, -- §8.5.2: carries one of our characters, places or key art
	recog: "n/a" | "pending" | "pass" | "fail", -- newest blind-test result for this asset's subject (§8.5.3); "n/a" iff likeness == false
	reviewRef: string?, -- "R-012": row in docs/agent/RECOGNIZABILITY.md (required when recog is "pass" or "fail")
}
-- AssetEntry = { id, status, source, kind, note?, prov: AssetProvenance? }
```

Rules M09–M13 are added to Part 7's `check_manifest.luau`:

| Rule | Severity | Check |
|---|---|---|
| M09 | error | `prov` is required when `source ∈ {ai, creator-store, artist}`, or `status == "final"`, or the key matches `^model%.%l+%.gear_`, `^model%.%l+%.peak_`, `^model%.%l+%.summon_` or `^img%.style%.`, or the asset is music or VO |
| M10 | error | `source == "creator-store"` or `"ai"` ⇒ `prov.likeness == false` (no AI or store likeness, ever; §0.5) |
| M11 | error | `recog == "n/a"` ⇔ `likeness == false`. `recog ∈ {pass, fail}` ⇒ `reviewRef` names an existing row in `RECOGNIZABILITY.md` with the same result. |
| M12 | error (`--release`) | Every referenced entry with `likeness == true` has `recog == "pass"` |
| M13 | error | `source == "artist"` ⇒ `prov.license` starts with `assignment:` (character, face and mascot art is drawn by people under written assignment, never generated, §0.5) |

Part 1's `ASSET_PROVENANCE.md` row format gains two trailing columns: `| <likeness yes/no> | <R-### or n/a> |`. Its prompt and query columns are linted (L10).

#### 8.5.2 What counts as likeness-bearing

These carry our characters, places or key art, so they need a blind-test row (§8.5.3) before they ship in a public build:
- Charm Gear, Peak pieces and summon models (`model.<style>.gear_*`, `model.<style>.peak_*`, `model.<style>.summon_*`);
- style palettes in combination with their props, and VFX that stage a style's signature moment;
- style icons and portraits (`img.style.*`), the mascot and NPC art;
- map landmarks (GDD §11): the Pavilion, the Jubilee Clock, the pier, the lighthouse;
- Getaway set pieces (the "Greetings from…" postcard frames);
- music, VO and barks;
- UI art: the logo lockup, the display face, stamp and postcard frames;
- the icon, thumbnails and key art.

Generic props are not likeness-bearing: crates, rocks, driftwood, rope coils, rubble, graybox walls, plain beach huts and deckchairs.

#### 8.5.3 Blind tests T7–T11 (human review gate)

The question behind every test: would an ordinary player connect a subject with an existing show, film or game, or with a character from one? GDD §5.5 defines the tests; this part fixes how they are prepared, recorded and checked. **The producer runs them; the agent only prepares captures and appends `pending` rows.** A subject passes only when its tests pass. A fail sends it back for redesign as a data change (names in `Base`, the palette in Part 4 `StyleDef.palette`, the Charm Gear, Peak piece or summon recipe in Part 4 §4.7.4–§4.7.5, a map builder), followed by a new `pending` row and a re-test.

**Testers** (T7, T8, T11): at least 10 per round, aged 9–15, recruited by the producer with a parent's or guardian's permission, familiar with popular cartoons and Roblox games. They see no title, names or context, and answer one question: "Does this remind you of anything you have seen before? What?" The producer counts answers; nobody writes down a named work in the repository (counsel keeps the notes). The Starter pair always goes first.

| Test | What is tested | Who | Pass | Agent's preparation |
|---|---|---|---|---|
| **T7** Full-lineup recognition | All 18 styles side by side (concept art) and the six ★ faces, plus the lineup as a whole | 10+ testers, run by the producer | Fewer than 2 testers name the same existing work for the lineup or for any single style | One `pending` row `lineup` |
| **T8** Blind art comparison | Each subject alone: character sheets, Charm Gear silhouettes at 48 px (GDD §12.3), the mascot, map frames, the icon and every thumbnail | 10+ testers | Same as T7, per subject | Stills from `look.gear_silhouette` (templates built with `silhouette = true`) and Part 6 map captures; one `pending` row per subject and preparation run, whose Material cell lists every still or capture of that run (for example the three `hub` captures of Part 6 §6.3 go in one `map:hub` row) |
| **T9** Neighbour check | Each style, place, the mascot and the key art against well-known existing works that share a theme or a look (the list stays with counsel, and its names go into the local denylist) | Producer with counsel | Every neighbour has a recorded guardrail and none is crossed; counsel signs | One `pending` row `set:neighbours` |
| **T10** Lineage Index | Every reference used for art, audio and writing, listed in GDD Appendix D | Producer | Every reference is labelled public domain, generic or ours; none comes from an existing show, film or game | One `pending` row `set:lineage` |
| **T11** Gameplay clip | 30 s of play per style, every character masked as a flat black silhouette, names and nameplates hidden | 10+ testers | Same as T7, per style | `look.clip_stage` per style (Part 7; the producer records the clip and keeps it outside the repository); one `pending` row per style |

**When they run:**
- T7–T10 run on concept art before art lock (GDD §5.5, §21.2). T9 runs again for each new map and style; T10 runs before anyone works from a reference.
- T11 runs after M0 on `attendant` and `lifeguard` (the pair first), before any public footage, and again for each slice style as it lands in M1-G (Part 7 M0 A14, M1-G A12), and for every later style before its footage (GDD §19.2).
- Any drift change (§8.5.6) re-opens the subject's T8 and, for a style, T11.

**`docs/agent/RECOGNIZABILITY.md` row format.** The agent creates the file from its template and appends rows with Result `pending`; humans fill in Testers, Same-work hits, Result and Signed. Rows are never deleted, and a signed row is never edited: a re-test appends a new row.

```
| Row | Test | Subject | Material | Prepared (by, date) | Testers | Same-work hits | Result | Signed (initials, date) |
|---|---|---|---|---|---|---|---|---|
| R-001 | T11 | pair:attendant+lifeguard | look.clip_stage attendant and lifeguard, run 3f2a9c1e; clips kept by the producer | agent, 2027-01-08 | | | pending | |
```

- **Subject forms:** `lineup`, `style:<id>`, `pair:<id>+<id>`, `npc:mascot`, `map:<mapId>`, `meta:icon`, `meta:thumbnail_<n>`, `set:neighbours`, `set:lineage`.
- **Same-work hits** is the largest number of testers who named the same existing work, written as a count (`1`), never as a name.
- **Pass rule** (L13): T7, T8 and T11 need Testers ≥ 10 and Same-work hits ≤ 1. T9 and T10 write `n/a` in both count columns and pass on the signer's judgement.
- **Newest row wins:** L13 reads the newest row for each (test, subject) pair, so a drift change's new `pending` row blocks a public publish until the re-test passes.
- **Required subjects** for a public publish: §8.4.4.
- A recorded `fail` opens a blocker of type `ip` (R-IP-09) and a redesign task for that subject.

#### 8.5.4 AI-generation prompt rule

This extends Part 1 §1.7.3.
1. Before every `generate_mesh`, `generate_material` or `generate_procedural_model` call, the agent runs `lune run tools/lint_strings.luau --text "<prompt>" --scope prompt` and proceeds only on exit 0.
2. A prompt uses generic nouns plus shape, colour and material ("blue and white striped wooden beach hut"). It never contains:
   - the name of an existing show, film, game, character, place, artist, studio or brand, or a title or episode name;
   - **names from our own pack** (they mean nothing to the generator, and generic prompts keep the provenance log clean);
   - "in the style of …" anything.
3. No reference images, fan art or screenshots of any existing work are passed to a generation tool.
4. AI generation is never used for characters, Pip-Pip or any face, Charm Gear, summons, signature props, UI art or any final asset (§0.5).
5. Results are `source = "ai"`, `likeness = false` (M10), and are recorded in `ASSET_PROVENANCE.md` with the verbatim prompt (L10).

#### 8.5.5 Creator Store rule

This extends Part 1 §1.7.2.
- Every `search_asset` query passes `--text … --scope prompt` first and is recorded in the asset's provenance row.
- V2 rejects any asset whose name, description, thumbnail, tags or creator page names or resembles an existing show, film, game, character or brand, or that depicts one.
- **No fan-made asset of any existing work is ever inserted**, even into the scratch place.
- Creator Store assets are `likeness = false` (M10). Insertion is limited to generic meshes, decals and audio.

#### 8.5.6 Looks and design drift

Each style has one look. Charm Gear, Peak piece and summon recipes are Part 4 §4.7.4–§4.7.5, palettes are Part 4 §4.7.1, and presentation reads palettes only through `Theme.stylePalette` (§8.2.7). These rules keep the looks ours:
- **Charm Gear is the read.** In a fight a Guest is their own avatar plus Charm Gear: one oversized held or back prop that defines the silhouette, plus a head piece. The plush keyring sold with a Charm is a cosmetic extra and never the read. The full staff member appears on the postcard, in the lobby, as the Peak uniform for 60 s, on victory screens, and on a Guest only when that player turns the Full Uniform on (default Off, §0.4).
- **Silhouette.** Each style reads from its prop in pure black at 48 px (GDD §12.3). `look.gear_silhouette` stills prove it in graybox (T8 material).
- **No borrowed references.** Art, audio and writing use only references labelled in the Lineage Index (GDD Appendix D, T10).
- **Faces and line** (final art, GDD §12): glossy oval eyes with one highlight, one warm navy-plum line colour. Character, face and mascot art is drawn by people under written assignment (M13).

**Drift changes.** Later edits could slide a style, place or key art toward an existing work. A **drift change** is any of these:
- a change to what a move in an implemented kit does or shows (its behaviour, effect, summon or VFX staging), not only its numbers;
- a change to a style palette, a Charm Gear, Peak piece or summon recipe, or a style icon;
- a change to a built map's landmark, or to the icon or a thumbnail.

Numbers tuned inside the GDD §6.7 bands are not drift changes. The task that makes a drift change appends `pending` rows in the same task: T8 for the subject, and T11 as well for a style. The design team re-runs the kit and look factors of the 13-factor distance rubric (GDD §5.5) before the change merges, outside the repository. Because L13 reads the newest row, a public publish waits for the re-test.

#### 8.5.7 Voice and music

- **Placeholder voice.** `AudioTextToSpeech` barks run in Studio only (`Perf.Audio.TtsInStudioOnly`, Part 6; L14). Their text is `ui.vo.*`. No voice preset is chosen or described as imitating a person or character.
- **Future VO** (artist assets):
  - original actors only, under written agreements that assign the recordings to us (GDD §5.4, §14.5);
  - casting briefs describe our characters, never a real actor or an existing character, and never say "sound like …";
  - contracts include written assignment and an originality clause;
  - **no AI voice cloning or voice conversion** of any person (GDD §5.4, §14.5);
  - VO assets are `likeness = true`, and their blind-test row records "no sound-alike".
- **Music:**
  - 100 % original (GDD §5.4, §14.1): accordion and melodica over a seaside brass band, with a genre per place (GDD §14.2);
  - briefs name moods and instruments only, never an existing song, theme or score;
  - the composer signs an originality warranty and a written assignment;
  - before release, a musicologist compares the theme with well-known themes (the list stays with counsel);
  - music assets are `likeness = true`, and their row records the melody check.
- M1-G ships no music (Part 6).

#### 8.5.8 Verify

- `check_manifest` M09–M13 (Part 7 runs them).
- Lune `ip.prov_rules`: fixture entries cover each M09–M13 violation.
- Scenario `look.gear_silhouette` (Part 7) with templates built with `silhouette = true`: one still per implemented style, attached to a `pending` T8 row.
- Scenario `look.clip_stage` (Part 7): T11 material per style, attached to a `pending` T11 row.
- Lune `ip.recog_rows` (L13).

---

### 8.6 Trigger watch and response playbook

#### 8.6.1 Watch list

The agent cannot reach Creator Hub, License Manager, trademark registers or the press. A human (the owner, the producer or counsel) owns the watch and records each check in `docs/agent/TRIGGER_WATCH.md`. Row format: `| date | trigger | source | what happened | decision | actions done | by |`. Rows name the trigger and the kind of source only; they never name a third-party work or mark, and counsel keeps the details.

| ID | Trigger | How detected | Cadence |
|---|---|---|---|
| TW-01 | A **lookalike claim**: a rights holder, platform notice or letter says a character, place, kit, piece of art or the game copies an existing work | Roblox notice, email or post | On receipt |
| TW-02 | A **License Manager offer** (standard or conditional) that names our game | Creator Hub or email | On receipt |
| TW-03 | A **DMCA or Rights Manager removal** of the game or an asset | Roblox notice, game status | On receipt |
| TW-04 | A **title conflict**: a game, app, mark or filing with the same or a confusingly similar name, or an opposition or letter about our title | Counsel's title watch (Roblox, stores, USPTO, WIPO); email | Weekly, and on receipt |
| TW-05 | **Coverage or community posts** that call the game a copy of an existing work | Search alerts, social | Weekly |
| TW-06 | **Knock-offs of our IP**: experiences, UGC or merchandise using our names, mascot or characters | Experience and Marketplace search; Rights Manager once our characters are registered | Weekly |
| TW-07 | **Breakout scale** (typical CCU ≥ 5,000) | Analytics | Weekly |
| TW-08 | **Design drift**: a merged drift change (§8.5.6) without a newer `pending` or `pass` row | Agent at each gate: `git log` on the style, template and map files since the subject's newest signed row; human weekly | Weekly, and at each gate |
| TW-09 | The **dev universe is no longer private** (§8.2.6) | Creator Hub ⟩ the dev experience ⟩ privacy setting | Weekly |
| TW-10 | A **denylist update** from counsel (a new neighbour, a newly launched work) | Counsel | On receipt |

#### 8.6.2 Response playbook

Timelines start at detection. Steps marked (A) are agent-executable technical steps. Everything else is a human decision.

| Trigger | Within 24 h | Within 14 days | Never |
|---|---|---|---|
| TW-01 | Forward to counsel the same day. No substantive reply without counsel. Preserve the records of independent creation: the dated World and Roster Bibles, the Lineage Index (GDD Appendix D), `RECOGNIZABILITY.md`, `ASSET_PROVENANCE.md` and the git history. If counsel asks: (A) prepare the fast actions of §8.6.3 for the named style. | Counsel decides: a good-faith reply or dispute backed by those records, or a redesign as a data change (`Base` strings, palette, Charm Gear recipe), re-tested (§8.5.3) and published through §8.4.7 | Delete docs or history (litigation hold); argue publicly; add anything that refers to the other work |
| TW-02 | Record the offer (rate, duration, standards). Counsel review. **The offer auto-activates on day 14 if no one acts.** | Dispute it, per counsel, with "does not use this IP", backed by the records above | Let it auto-activate; accept it without counsel |
| TW-03 | Do **not** counter-notice without counsel: a counter-notice is sworn under penalty of perjury and accepts a court's jurisdiction (17 U.S.C. §512(f), (g)). Freeze monetisation (§8.6.3). Preserve all records. | Counsel decides: a good-faith counter-notice, or a redesign and republish. Compensate players through Part 5 `compensation` grants. | Delete docs or history |
| TW-04 | Counsel review the same week. Pause title-bearing marketing spend. | Decide: keep the title on counsel's opinion, or rename: pick a backup, knock it out (§8.7), change `ui.game.title`, the logo lockup and the metadata, and republish through §8.4.7 | Ship a new title that has not been knocked out |
| TW-05 | Do not amplify. Reply, if at all, only with the press line (§8.4.8). | The owner re-reads the risk table (§8.1.3); counsel review if it spreads | Reply with comparisons; repost fan comparisons |
| TW-06 | Record. Once our characters are registered, file Rights Manager removals (up to 250 links per request). | Counsel on repeat offenders | Retaliate against players |
| TW-07 | Re-run T9 on the current build and marketing. Counsel refreshes the denylist and the title watch. | Owner and counsel decide on trademark filings | Continue without the re-check |
| TW-08 | (A) Append the missing `pending` rows (§8.5.3). | The producer re-runs T8 (and T11 for a style) | Publish with an open drift row |
| TW-09 | Set the dev universe back to **Private** at once and record who changed it. (A) Scan the server logs for `dev_universe_non_member` kicks. | Treat the exposure as a public publish without the gate; the owner decides on any follow-up | Leave the dev universe public |
| TW-10 | (A) Run `lint_strings` in dev mode and `lint_strings --public --dry-run` on the current metadata draft. | Rename anything flagged (a data change) and re-run the gate | Copy the new entries anywhere |

#### 8.6.3 Fast actions (live config)

Names and art cannot change live: the pack and the templates ship in the place, so a redesign is always a new build through §8.4.7. These live levers buy time. The human sets them in Creator Hub ⟩ the universe ⟩ Configs; the server applies them as Part 2 §2.6.4 describes.

| Lever | Effect | Use when |
|---|---|---|
| `HB_Features` `{"DisabledStyles": ["<styleId>"]}` | The style cannot be equipped from the next spawn and drops out of Sundry's Rack (Parts 2, 5) | A claim or a failed re-test names one style |
| `HB_Features` `{"Store": false}` | The Store is closed (Part 2 §2.6.2) | Freezing monetisation during a removal or claim |
| `HB_Features` `{"PaidRandomItems": false}` | Wish packs and the Lucky Stamp pass are hidden and server-rejected (Part 5) | Counsel asks |

Also take any affected style or art out of the icon, thumbnails, ads and creator kits by hand (HP-02, HP-07). The agent then prepares the redesign and runs §8.4.7 steps 2–8; the human publishes. Remove the overrides after the new build is live.

---

### 8.7 Title clearance for *Hullabrawl*

**Status.** *Hullabrawl* is the chosen title (GDD §5.8). The name joins "hullabaloo" and "brawl", and in the fiction the Grand Hullabrawl is the Staff's end-of-season brawl on the pier. A first web-indexed check found no game with this name; the full knockout below has not run. Until `TITLE_CLEARANCE.md` says `status: cleared`, L15 fails every public gate, and the title appears in no public metadata, ad, store listing or filing.

**Knockout procedure** (GDD §5.5, Appendix F11; run by the human and counsel; counsel signs; results go into `TITLE_CLEARANCE.md`, §8.4.5):
1. **Spellings (K1–K5).** Search "Hullabrawl", "Hulla Brawl", "Hullabrall" and "Hulabrawl". Close variants are candidates for defensive domains and handles.
2. **Roblox (K1).** Experience search, group names and Marketplace names.
3. **Stores (K2).** Steam, Apple App Store and Google Play.
4. **Marks (K3, K4).** USPTO classes 9 (game software), 41 (online entertainment), 28 (toys and games), 25 (clothing) and 16 (printed matter), including sound-alike and look-alike marks in the games space; the WIPO Global Brand Database for international filings; and any common-law use the searches find.
5. **Domains and handles (K5).** .com, .gg and .net; YouTube, TikTok, X, Instagram, Discord and a Roblox group.
6. **Secondary names (K6).** The Wishaway Isles, Elsewhere-on-Sea, the Tidings, Pip-Pip, Sundry, the 18 style names, faction and currency names, the pass names and any coined words: at least Roblox search and USPTO classes 9 and 41.
7. **Logo (K7).** The lockup "HULLABRAWL · Greetings from the Wishaway Isles" uses an original sign-painter display face, with no crest, shield, lettering or layout borrowed from an existing game or show.
8. **Record.** Counsel summarises the findings in `notes` without naming any third-party mark (the details stay in counsel's files), and advises whether to file marks before the announcement. The owner commits the file.

**If the title changes**, only these are title-bound:
- `ui.game.title`, and every `Base` value that contains the title word (today `ui.hud.combo_big`, §8.3.7);
- the logo lockup art;
- `PUBLISH_METADATA.md`;
- `TITLE_CLEARANCE.md` (a new knockout).

Internal names (the `[HB]` log tags, the `HB_` and `Hb` prefixes, the `HullabrawlTestHarness` plugin and the "Hullabrawl DEV" place) are not player-visible and stay.

**Backup titles.** The first backup candidate, *Scrumble*, clashes with an existing knockout party game, so backup titles are an open question (GDD Appendix F1). Every candidate goes through the same knockout before it is announced or used in metadata.

**World name.** If T8 testers connect "the Wishaway Isles" with an existing work, the fallback world name is "the Wishaway Coast" (GDD Appendix F11; `ui.world.name`, `TITLE_CLEARANCE.md` `world_name` and the logo lockup change together).

---

### 8.8 New contracts introduced

1. **The `Base` pack** (`src/Shared/Skin/Packs/Base.luau`): returns `{ id = "Base", strings }`; pure and require-free; built with `put()`; `STYLE_IDS` equals `Types.STYLE_IDS` (`skin.base_shape`). Fallback order `Base` → `[key]` (Part 2 §2.9.2). No pack selector anywhere.
2. **Skin keys** introduced here (†, §8.3.7; their text follows the ownership rule of §8.3): `ui.game.title`, `ui.world.name`, `ui.hud.{meter_name, level, treasure, shards, sparks, quests}`, `ui.quest.<id>` (one per `Progression.Quests.DailyPool[*].id`), `ui.notify.quest_done`, `ui.report.{button, title, reason_exploit, reason_teaming, reason_harassment, reason_other, send}`, `ui.wish.{title, make_1, make_10, odds, pity_title, new, locked_paid, auto10, secret}`, `ui.wheel.{shards, convert, mastery_combat_only, variant, variant_base}`, `ui.shop.{title, refresh, coming_soon, balance, confirm_in_dev}`, `ui.store.{title, mock, spent_week}`, `ui.settings.{title, character_look, style_gear, own_look_postcard, own_look_avatar, tab_safety, quiet_barks, damage_numbers, hold_toggle_run}`, `ui.training.damage_numbers`. Token names: §8.3.7 notes. Shop lines: `ui.shop.sundry_line_1…6`. Part 6 keys carried in `Base` with Part 6's text (Part 6 §6.13 item 16), among them `ui.hud.{go_peak, fully_booked, checked_out, tide_low, tide_high, tide_in, tide_out}`, `ui.wheel.{title_ring, title_scrapbook}`, `ui.shop.{tab_rack, tab_store}` and `ui.vo.{awaken, fully_booked, ko, legendary}`.
3. **Key-family registry** (§8.4.3), including `ui.wish.zone_<rarity>` (Part 5), `ui.postcard.body.<id>` (Part 5), `ui.quest.<id>`, `ui.report.reason_<reason>`, `ui.hud.tide_<phase>` and `ui.vo.<line>` (Part 6). L03 accepts exactly the namespaces of §8.2.2.
4. **Rules file** `tools/lint_strings.rules.json`: Part 7's schema version 2 plus the optional fields `list` and `keyPattern`; lists `gambling` (L07) and `loretime` (L04); generic terms only. Scopes added: `meta`, `prompt`, `tree`, `text`.
5. **Local denylist policy** (§8.4.2): supplied by the owner or counsel, refreshed before each public gate and on TW-10, applies in every scope, findings by entry number only. Format and matching: Part 7 §7.5.1.
6. **Lint additions** to the Part 7 mechanism: rules L07–L15; flags `--tree`, `--meta`, `--text --scope`, `--dry-run` (L05, L09, L13 and L15 become `W-DRYRUN`; agent procedure `ip.gate_dry_run` only); the gate line `[HB][GATE] publish-ip PASS|FAIL commit=<sha7> errors=<n> warnings=<m>` and the final `[HB][GATE] publish PASS|FAIL commit=<sha7> items=<n>/<n>`.
7. **Retired-key policy** (§8.4.3): the owner supplies `tools/retired_keys.txt` from counsel's archive (format Part 7 §7.5.6); it is gitignored and never written, printed or quoted by the agent; findings print only `<path>:<line> retired-key`; a missing list prints `SKIP no retired_keys list` (dev may proceed), while a public publish needs the list present and a PASS; the public procedure runs `lint_retired_keys --tree build/ip_tree.json`.
8. **`check_manifest` rules M09–M13** and `AssetEntry.prov: AssetProvenance?` (`license` value `assignment:<contract ref>` for artist assets; M09 key patterns `gear_`, `peak_`, `summon_` and `img.style`). `ASSET_PROVENANCE.md` gains the columns `likeness` and `R-###`.
9. **`Verify.publishGate(opts)`**, plus the `run_builder.luau` branch `fn == "publishGate"`; the dump includes attribute names (`kind = "attrName"`). Output file `build/ip_tree.json` (gitignored).
10. **Config `Ip = { DevUniverseIds: {number}, DevGroupId: number }`** (code-only, Part 2's not-overridable list) and the dev-universe members-only rule in PlayerService (`[HB][Skin] ERROR dev_universe_non_member userId=<id>`; seam `PlayerService._devAccessCheck`).
11. **`Theme.stylePalette(styleId, variantId?)`** (canonical code in Part 6 §6.9.4, reproduced verbatim in §8.2.7; this part owns the rule): the only way presentation reads style colours; Off-Duty Look rule `v1` swap, `v2` darken 25 %.
12. **Docs and tool files:**
    - human-signed: `docs/agent/TITLE_CLEARANCE.md` (every line committed by `dev-place.json.ownerGitEmail`, checked by L15);
    - human-supplied and local (gitignored): `tools/denylist.local.txt` and `tools/retired_keys.txt`;
    - shared: `RECOGNIZABILITY.md` (the agent appends `pending` rows only; result rows owner-committed, checked by L13), `PUBLISH_METADATA.md`, `TRIGGER_WATCH.md`;
    - templates in `docs/agent/templates/`: `TITLE_CLEARANCE`, `PUBLISH_METADATA`, `RECOGNIZABILITY`, `TRIGGER_WATCH` (`<NAME>.template.md`);
    - `tools/lib/SafeWrite.luau`, the only file writer for Lune host tools; it refuses `title_clearance.md`, `denylist.local.txt` and `retired_keys.txt` by base name (`SafeWrite.isProtected`).

    The Edit/Write deny entries for the title file and the denylist are in the Part 1 example permissions file (`docs/agent/claude-settings.example.json`, §1.2.4).
13. **Blind tests T7–T11** (§8.5.3): tester rule, pass rule (Testers ≥ 10, Same-work hits ≤ 1 for T7, T8, T11), subject forms, the `RECOGNIZABILITY.md` row format, newest-row-wins, and the drift-change rule (§8.5.6).
14. **Blocker type `ip`** (Part 1 §1.8.1): any §8.6 trigger, a recorded T7–T11 fail, or a lint finding that a rename cannot fix.
15. **Human checklist HP-01…HP-07** and the public H-11 variant ("Save to File As", then publish from a separate window).
16. **Test names:** `skin.base_shape` and `ip.*` (§8.4.9), and the Studio scenarios `ip.dev_universe_members_only` and `ip.no_missing_keys_tour`. `ip.gate_dry_run` is an **agent procedure** (§8.4.7), never a Studio scenario.
17. **Dry-run branch naming:** the throwaway branch `release/<YYYY-MM-DD>-dryrun`, never pushed.
18. **Trigger watch** TW-01…TW-10 and the fast actions of §8.6.3.

### 8.9 Issues with spine

1. **§0.9 says the publish gate has no config flag.** This part keeps the code-only `Ip` section (Part 2 §2.6.2). It holds only the dev universe and dev group IDs for the members-only rule (§8.2.6) and is not a publish switch, so the spine stands as written.
2. **§0.3's `tools/` list (resolved).** §0.3 now names `tools/lint_strings.rules.json`, `tools/lib/SafeWrite.luau` and `tools/retired_keys.txt` (owner-supplied and gitignored, like `tools/denylist.local.txt`), as listed in §8.8.
3. **§0.4's key formats (resolved).** §0.4 now lists the `variant`, `title`, `product`, `pass` and `badge` namespaces beside `style`, `move`, `map`, `ui`, `flair` and `rarity`, and names §8.2.2 as the full list; Part 7's L03 accepts the §8.2.2 table.

**Cross-part (not spine):**

4. Part 6 §6.1.3 and §6.3.1 own the hub's zone IDs. §8.3.4 uses them verbatim; a change in Part 6 changes §8.3.4 in the same task.
5. Part 5 §5.12 item 14 owns the wording of its lines; §8.3.5–§8.3.7 carry them verbatim. A wording change is made in both places in the same task.
6. Part 6 owns the wording of its HUD, UI and world lines (Part 6 §6.13 item 16; for example `ui.hud.combo_big`, `ui.hud.fully_booked`, `ui.vo.ko`), and Part 4 owns the style and move keys (Part 4 §4.5.10; §8.3). §8.3.7 and §8.3.3 carry them verbatim; the same both-places rule applies.
7. `Theme.stylePalette` is a Part 6 file's helper: Part 6 §6.9.4 holds the canonical code and §8.2.7 reproduces it. A change to the code is made in Part 6 first and copied here in the same task.

### 8.10 Open questions / spikes

| ID | Question | How to resolve | Fallback |
|---|---|---|---|
| S-P8-1 | Does Script Sync stay quiet across the dry-run branch switches (only `docs/` changes), so the sync gate matches before and after? | Run the agent procedure `ip.gate_dry_run` (§8.4.7) twice in a row; the post-run sync `agg` equals the pre-run `agg` both times | Human toggles the root sync off and on; re-run the sync gate |
| S-P8-2 | Does `git blame --porcelain` through Lune `process.exec` report the right author for each row on Windows (CRLF files)? | `ip.signed_file_provenance` on a Windows host | Normalize line endings before matching rows; compare emails case-insensitively |
| S-P8-3 | What is the maximum `execute_luau` response size? Does the deduplicated tree dump fit for the Elsewhere-on-Sea map? | Run `publishGate` on the full map; measure the JSON length | Paginate: `opts.service` scans one service per call |
| S-P8-4 | Does "Publish to Roblox As…" re-point the open Studio document to the new universe? | Not tested on purpose; the procedure always publishes a saved copy from a separate window | Procedure as written |
| S-P8-5 | Can `MarketplaceService:GetProductInfoAsync` / `BadgeService:GetBadgeInfoAsync` run from Edit-mode `execute_luau` (with H-06), so live product names could be echoed into L08? (Members verified in docs; Edit-mode behaviour unverified.) | Call it on one dev product ID | Keep names in `PUBLISH_METADATA.md` (current design) |
| Q-P8-1 | Counsel: is *Hullabrawl* clear as a title, and which backup titles should be knocked out (GDD Appendix F1, F11)? | Knockout (§8.7) | Backup list from the owner |
| Q-P8-2 | Owner and counsel: who maintains the local denylist, and how often is it refreshed between gates? | Owner decision | Counsel, before each public gate and on TW-10 |
| Q-P8-3 | Producer: how are testers aged 9–15 recruited, with a parent's or guardian's permission and without storing personal data? | Producer and counsel | Adult testers for T9-style reviews only; T7, T8 and T11 wait |
| Q-P8-4 | Owner: switch to "the Wishaway Coast" if T8 connects the world name with an existing work? | T8 result | Keep "the Wishaway Isles" |
| Q-P8-5 | Owner: how British should the Isles feel after the kids' humour test? It changes `Base` wording, not keys. | Kids' humour test (GDD Appendix F12) | The current plain wording |
| Q-P8-6 | Is License Manager image matching run on Roblox-captured gameplay screenshots of avatar-mounted gear? | Not knowable from docs | Assume yes (the risk table already does) |
| Q-P8-7 | Owner: should the example permissions file (Part 1 §1.2.4) also deny Edit and Write on `tools/retired_keys.txt`, as it does for the local denylist? | Owner decision; Part 1 changes the example file and the four entries of §8.4.5 become six | SafeWrite refusal (§8.4.5) and R-IP-05 only |
