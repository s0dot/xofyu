[← Build spec index](../STUDIO_BUILD_SPEC.md) · **Part 7 of 8** · sections §7.x · Part 0 contracts (§0.x) are in the index file

## 7. Testing, verification and performance

This part defines how every requirement in Parts 2–6 is proven: the Lune host tests, the in-Studio scenario harness, MCP input tests, multi-client and network-condition tests, performance measurement, human checklists, milestone gates and CI. It owns:
- `tests/run.luau`, `tests/shim/*` (runner, test API, module loader, datatype shims) and the Part 7 spec files;
- `src/DevTools/Tests/**` (scenario framework, client test driver, Part 7 suites) and `src/ServerScriptService/Services/TestHarness.luau`;
- `tools/check_manifest.luau`, the mechanism of `tools/lint_strings.luau` (rules data is Part 8's), `tools/export_framedata.luau`;
- `tools/plugin/AlgebraicTestHarness/` and `tools/plugin/build.luau`;
- the test snippets in `tools/studio/` listed in §7.3.6–§7.6;
- `.github/workflows/ci.yml`;
- the gate checklists (§7.7).

Reading order for the agent before any test task: §0.4 (log format) → Part 1 §1.4 (inner loop, `log_scan`) → this part §7.1–§7.3 → the test names in the owning part's "Verify" lines.

Conventions:
- Frames are 60 Hz sim frames; `f` = `Frames.fromTime(time())` (Part 3 §3.1.3). Times are seconds (`Sec`), distances studs.
- `(td)` = tuning default chosen here where the GDD gives no number; it never changes a GDD number.
- `(verify in spike T7-Sn)` = an engine or tool behaviour the docs do not settle; the spike is in §7.11.
- Every engine API named here was checked in the offline creator-docs snapshot (2026-10-10). Lune APIs were checked in `lune-org/lune` and run on Lune 0.10.5.
- Test result lines follow §0.4 exactly. Informational lines use `[ALG][Test] …` (Part 2 tag `Test`), never a third `[ALG][TEST]` form.

---

### 7.1 Test strategy (pyramid)

#### 7.1.1 Levels

| Level | ID in catalog | Tool / runtime | Triggered by | When it runs | Budget (td) |
|---|---|---|---|---|---|
| L1 Host unit | `lune` | Lune 0.10.5 on the host or CI: pure `Core/*`, `Wish/*`, `Config/*`, `Types`, `Log`, `Skin`, `Net/Schemas`, `Styles` data, kit validator, lints, tools | `lune run tests/run.luau` (inside `tools/check.luau`) | Every task (host gate, Part 1 §1.4.2), every CI push | Full suite ≤ 60 s on CI; any single test > 1 s is tagged `slow` |
| L2 In-Studio scenario | `studio` | `TestHarness` + `DevTools/Tests` suites in a solo Play session (server + 1 client), driven through `execute_luau` | `tools/studio/test_run.luau` (§7.3.6) | Tasks whose card lists scenarios; phase end; gates | ≤ 180 s per playtest (Part 1 §1.4.5); gate batches ≤ 900 s |
| L3 MCP input-driven | `mcp` | Same session; real input through `user_keyboard_input`, `user_mouse_input`, `character_navigation` | `test_run` ops `setup` → MCP input → `assert` (§7.3.7) | Input, HUD and UI tasks; gates | ≤ 120 s per test |
| L4 Multi-client | `multi` | `StudioTestService:ExecuteMultiplayerTestAsync` (server + 2–8 clients): Option A `execute_luau`, else the `AlgebraicTestHarness` plugin | `tools/studio/multi_run.luau` (§7.3.8) | P3+ networking tasks; gates | ≤ 600 s per run |
| L5 Network conditions | tag `net` on `studio`/`multi` tests | `NetworkSettings` delay/jitter/loss (Network Simulator) | `tools/studio/net_profile.luau` before Play/launch | Rewind, prediction, input-timing tasks; M0 gate | — |
| L6 Perf and soak | `perf` | Server `PerfSampler`, client `PerfProbe`, server bots (`Dev.Command spawnBots`) | `perf.*` selectors, multi mode | P8; gates | ≤ 15 min per run |
| L7 Human | `human` | Human playtests (H-10) and device test (H-12) | Gate checklists §7.7.4–§7.7.5 | Gates; per-style feel checks | — |

#### 7.1.2 Placement rule (push every check down)

1. A rule that can be stated as a pure function of plain data is tested in L1, even if it is also exercised in Studio. Studio scenarios prove *integration* (attributes, replication, physics, IAS, remotes), not arithmetic.
2. A Studio scenario uses server bots with scripted brains (§7.3.3) instead of a client whenever the client adds nothing. Use a client (L3/L4) only for input, prediction, rendering, UI and replication facts.
3. L4 proves networking facts only (prediction, input timing under latency, replication to other clients, join/leave).
4. L7 judges feel and device performance only. It never replaces an automated check that could exist.
5. The MCP `subagent` (type `playtest`) is advisory. Its verdict is never evidence (Part 1 §1.5.9).

#### 7.1.3 Determinism, isolation and flakiness

- **Seeds.** Randomized L1 tests use Part 3's `Rng` or Part 5's `Odds.stream` with literal seeds. `math.random` is never used in tests. Bot brains use `Rng.hash01(f, rid, salt)` (Part 3 §3.15.1).
- **Isolation.** Every Studio scenario builds its own fixtures on the runtime `TestPad` (§7.3.3), moves real players to the pad's bench, and undoes every reversible dev command in teardown. Scenarios must pass in any order.
- **No wall-clock assertions** inside Studio scenarios except perf scenarios. Timing assertions use sim frames from the tap log (§7.3.3).
- **Statistical tests** have a fixed sample size and threshold stated in the catalog (e.g. 20 Perfect Block attempts, ≥ 90 %).
- **Flaky policy.** A scenario that fails, then passes on an immediate unchanged rerun, is marked `flaky` in `docs/agent/PROGRESS.md` (Spike results table, row `flaky:<id>`). It counts as a failure for gates until fixed. Gate runs of `net`-tagged scenarios must pass twice in a row.

#### 7.1.4 IDs, tags and evidence

- **Test ID** (all levels): matches the Lua pattern `^[%w_]+%.[%w_%.%-]+$`, lower-case domain first (`combat.wall_splat`, `spike.SA-08`, `styles.rule_K06_detects`). IDs are unique across Lune and Studio. Names fixed by Parts 2–6 are binding and are reused verbatim in §7.4, except the IP-neutral renames of TASKS §16.1 C12 (e.g. `stun.stretch_ragdoll_minus10`).
- **Tags** (Studio scenarios): exactly one milestone tag `m0` or `m1g`, plus any of `net`, `slow`, `spike`, `release`, `selftest`. Lune tests may carry `slow` and `release`. The Lune tests tagged `release` (decision D-46; `T.test(id, fn, {tags = {"release"}})`) are exactly: `lint.manifest_release_detects_placeholder`, `lint.manifest_raw_id_detected`, `net.validators_reject_fuzz`, `skin.canon_locked`, `ip.leak_detect`, `ip.product_keys_original_only`, so `lune run tests/run.luau --tag release` runs ≥ 6 tests (zero matches exits 2, §7.2.2). Scenarios tagged `spike` (e.g. `store.receipt_live_spike`, the P0 `Spike_*` wrappers) need optional human steps or record exploratory results; every gate selector excludes them with `-@spike`.
- **Evidence** (Part 1 §1.4.7): host `[ALG][TEST] SUMMARY pass=<n> fail=0`; Studio `studio: <selector> SUMMARY pass=<n> fail=0 id=<first 8 chars>`; multi `multi(<n>,<profile>): <selector> SUMMARY pass=<n> fail=0`; perf `perf: <id> <key metrics>`; human `H-10 HF-01..HF-14 PASS (notes)`; agent procedure (`proc(<name>)`, TASKS §1.3; never a `studio(…)` selector) `proc(<name>): <the lines its card lists>`, e.g. `proc(ip.gate_dry_run): publish-ip PASS target=Original commit=<sha7> errors=0 warnings=<m>; publish FAIL target=Original items=<n>/<n> (R2); sync agg equal before and after` (§7.7.6, Part 8 §8.4.7).

#### 7.1.5 Network-condition profiles

Profiles live in `src/DevTools/Tests/Lib/NetProfiles.luau` (pure data, also read by `net_profile.luau`). Values are milliseconds and percent per direction; `NetworkSettings` loss is capped at 0.50 % per direction [Docs `NetworkSettings.yaml`, `studio/network-simulator.md`].

| Profile | Inbound min / jitter / loss | Outbound min / jitter / loss | Purpose |
|---|---|---|---|
| `Off` | 0 / 0 / 0 | 0 / 0 / 0 | Reset after every run |
| `Baseline30` | 15 / 0 / 0 | 15 / 0 / 0 | GDD §18.2 "30 ms baseline" (≈ 30 ms added RTT) |
| `M0Gate` | 60 / 30 / 0.5 | 60 / 30 / 0.5 | GDD §18.2 gate "150 ms ±30 ms jitter, 1 % loss", read as round-trip (see §7.10 item 1); calibrated by `netcond.rtt_m0gate` |
| `Wifi` | 30 / 12 / 0.2 | 30 / 15 / 0.3 | Docs preset "Home Wi-Fi" |
| `Mobile4G` | 45 / 20 / 0.4 | 55 / 30 / 0.5 | Docs preset "Standard Mobile (4G/LTE)" |
| `Bad3G` | 150 / 70 / 0.5 | 180 / 90 / 0.5 | Docs preset "Bad Connection (3G)"; resilience only |

```lua
--!strict
-- src/DevTools/Tests/Lib/NetProfiles.luau — Network Simulator profiles (test data, never read by live code). Spec §7.1.5.
export type Profile = {
	InboundNetworkMinDelayMs: number, InboundNetworkJitterMs: number, InboundNetworkLossPercent: number,
	OutboundNetworkMinDelayMs: number, OutboundNetworkJitterMs: number, OutboundNetworkLossPercent: number,
}
local function p(iMin: number, iJit: number, iLoss: number, oMin: number, oJit: number, oLoss: number): Profile
	return {
		InboundNetworkMinDelayMs = iMin, InboundNetworkJitterMs = iJit, InboundNetworkLossPercent = iLoss,
		OutboundNetworkMinDelayMs = oMin, OutboundNetworkJitterMs = oJit, OutboundNetworkLossPercent = oLoss,
	}
end
local P: { [string]: Profile } = {
	Off = p(0, 0, 0, 0, 0, 0),
	Baseline30 = p(15, 0, 0, 15, 0, 0),       -- GDD §18.2 baseline
	M0Gate = p(60, 30, 0.5, 60, 30, 0.5),     -- GDD §18.2 gate (td interpretation, calibrated by netcond.rtt_m0gate)
	Wifi = p(30, 12, 0.2, 30, 15, 0.3),       -- docs preset
	Mobile4G = p(45, 20, 0.4, 55, 30, 0.5),   -- docs preset
	Bad3G = p(150, 70, 0.5, 180, 90, 0.5),    -- docs preset
}
return P
```

**Calibration rule.** `netcond.rtt_m0gate` samples `Player:GetNetworkPing()` (round-trip seconds [Docs `Player.yaml`]) on the server every 0.5 s for 30 s with `M0Gate` applied. Pass: mean RTT 140–160 ms, P5 ≥ 115 ms, P95 ≤ 185 ms (td). If it fails, the agent changes only the `M0Gate` min/jitter values (never the loss) and records the result in *Spike results*.

---

### 7.2 Lune setup (host tests)

#### 7.2.1 Files

| Path | Purpose |
|---|---|
| `tests/run.luau` | Runner: discovery, CLI, output, exit code (§7.2.2) |
| `tests/shim/T.luau` | `describe`/`it`/`pending`/`expect` API (§7.2.3) |
| `tests/shim/Loader.luau` | Loads `src/` modules with Roblox require semantics and datatype globals (§7.2.4) |
| `tests/shim/FakeCtx.luau` | Part 4's behavior context factory (Part 4 §4.2.11) |
| `tests/core/*.spec.luau`, `tests/wish/*.spec.luau` | Specs (Parts 2–6 names, incl. Part 6's `tests/core/upload_anims.spec.luau`, which loads the pure helper `tools/base64.luau` natively through `@tools/`; Part 7 adds `shim`, `runner`, `tools`, `gdd`, `perf_budgets` specs) |
| `tests/core/golden/*.json` | Part 3 golden traces |
| `tests/core/fixtures/**` | Inputs for tool and runner self-tests (never discovered as specs) |

Part 7 spec files: `tests/core/shim.spec.luau` (`shim.*`, `runner.*`), `tests/core/tools.spec.luau` (`lint.*`, `framedata.*`, `skin.pack_key_coverage`, `standards.devtools_isolation`, `standards.test_loader_only`), `tests/core/perf_budgets.spec.luau` (`perf.budgets_match_gdd`), `tests/wish/gdd.spec.luau` (`wish.gdd_*`, `wish.per_style_*`, `wish.cosmic_path_bound_4400`).

#### 7.2.2 Runner (`tests/run.luau`)

Contract:
- Discovers `tests/**/*.spec.luau` (or `--root <dir>`), sorted by path, skipping folders named `shim`, `fixtures` and `golden`.
- Each spec file returns `function(T: TestApi)`; the runner calls it once. A spec that fails to load or register prints `[ALG][TEST] load.<path_with_underscores> FAIL <error>` and counts as one failure.
- Tests run in file order, then registration order, each inside `pcall`. Output: `[ALG][TEST] <id> PASS` or `[ALG][TEST] <id> FAIL <one-line detail>`, then `[ALG][TEST] SUMMARY pass=<n> fail=<m>`. Info lines: `[ALG][Test] slow id=<id> ms=<n>` (> 1,000 ms), `[ALG][Test] pending id=<id> task=<T0xx>`.
- Flags: `--filter <substring of id>`, `--tag <tag>`, `--file <substring of path>`, `--root <dir>`, `--fast` (skip tag `slow`), `--list`, `--json <path>`.
- Exit code: `0` if `fail == 0`; `1` on any failure or load error; `2` on a usage error or when a filter matches nothing.

Verified on Lune 0.10.5:

```lua
--!strict
-- tests/run.luau — Lune test runner. Usage: lune run tests/run.luau [--filter s] [--tag t] [--file s] [--root dir]
--   [--fast] [--list] [--json path]. Exit 0 = all pass; 1 = failures or load errors; 2 = usage error.
-- Spec §7.2.2.
local fs = require("@lune/fs")
local process = require("@lune/process")
local serde = require("@lune/serde")
local T = require("./shim/T")

type Args = { filter: string?, tag: string?, file: string?, root: string?, fast: boolean, list: boolean, json: string? }
local function parseArgs(argv: { string }): Args
	local a: Args = { fast = false, list = false }
	local i = 1
	while i <= #argv do
		local k = argv[i]
		if k == "--fast" then
			a.fast = true
		elseif k == "--list" then
			a.list = true
		elseif k == "--filter" or k == "--tag" or k == "--file" or k == "--root" or k == "--json" then
			local v = argv[i + 1]
			if v == nil then
				print(`[ALG][Test] ERROR missing value for {k}`)
				process.exit(2)
			end
			(a :: any)[string.sub(k, 3)] = v
			i += 1
		else
			print(`[ALG][Test] ERROR unknown argument {k}`)
			process.exit(2)
		end
		i += 1
	end
	return a
end

local function discover(dir: string, out: { string })
	local names = fs.readDir(dir)
	table.sort(names)
	for _, name in names do
		local p = dir .. "/" .. name
		if fs.isDir(p) then
			if name ~= "shim" and name ~= "fixtures" and name ~= "golden" then
				discover(p, out)
			end
		elseif string.match(name, "%.spec%.luau$") then
			table.insert(out, p)
		end
	end
end

local function oneLine(s: string): string
	return (string.gsub(s, "[\r\n]+", " "))
end

local args = parseArgs(process.args)
local files: { string } = {}
discover(args.root or "tests", files)

local pass, fail = 0, 0
local results: { { id: string, ok: boolean, detail: string?, ms: number } } = {}

for _, file in files do
	if args.file == nil or string.find(file, args.file, 1, true) then
		-- "./core/m1.spec" relative to this runner (Lune require-by-string, Part 1 §1.2.7)
		local modPath = "./" .. string.sub(file, #"tests/" + 1, -#".luau" - 1)
		local ok, err = pcall(function()
			local specFn = (require :: any)(modPath)
			assert(type(specFn) == "function", "spec must return function(T: TestApi)")
			specFn(T.api(file))
		end)
		if not ok then
			fail += 1
			local id = "load." .. string.gsub(string.sub(file, 7, -6), "[/%.]", "_")
			print(`[ALG][TEST] {id} FAIL {oneLine(tostring(err))}`)
			table.insert(results, { id = id, ok = false, detail = tostring(err), ms = 0 })
		end
	end
end

local function selected(c: T.TestCase): boolean
	if args.filter and string.find(c.id, args.filter, 1, true) == nil then
		return false
	end
	if args.tag and table.find(c.tags, args.tag) == nil then
		return false
	end
	if args.fast and table.find(c.tags, "slow") ~= nil then
		return false
	end
	return true
end

local ran = 0
for _, c in T.cases() do
	if selected(c) then
		ran += 1
		if args.list then
			print(c.id)
		else
			local t0 = os.clock()
			local ok, err = pcall(c.fn)
			local ms = math.floor((os.clock() - t0) * 1000)
			if ok then
				pass += 1
				print(`[ALG][TEST] {c.id} PASS`)
			else
				fail += 1
				local msg = if type(err) == "table" then tostring((err :: any).msg) else tostring(err)
				print(`[ALG][TEST] {c.id} FAIL {oneLine(msg)}`)
			end
			if ms > 1000 then
				print(`[ALG][Test] slow id={c.id} ms={ms}`)
			end
			table.insert(results, { id = c.id, ok = ok, detail = if ok then nil else tostring(err), ms = ms })
		end
	end
end
for _, p in T.pendings() do
	print(`[ALG][Test] pending id={p.id} task={p.taskId}`)
end

if args.list then
	process.exit(0)
end
if ran == 0 and (args.filter or args.tag or args.file) then
	print("[ALG][Test] ERROR no tests matched the selection")
	process.exit(2)
end
print(`[ALG][TEST] SUMMARY pass={pass} fail={fail}`)
if args.json then
	fs.writeFile(args.json, serde.encode("json", { pass = pass, fail = fail, results = results }, true))
end
process.exit(if fail == 0 then 0 else 1)
```

#### 7.2.3 Test API (`tests/shim/T.luau`)

The API is passed to each spec (Lune sandboxes every module's globals, so a global `describe` cannot be injected [Lune `crates/lune/src/rt/runtime.rs`: `lua.sandbox(true)`]).

```lua
--!strict
-- tests/shim/T.luau (types and registry; matchers abridged). Spec §7.2.3.
export type TestOpts = { tags: { string }? }
export type TestCase = { id: string, file: string, label: string, fn: () -> (), tags: { string } }
export type Matchers = {
	toBe: (expected: any) -> (),             -- ==
	toEqual: (expected: any) -> (),          -- deep, reports the first differing path ("$.b.2: 3 ~= 4")
	toBeNear: (expected: number, eps: number?) -> (), -- default eps 1e-6
	toBeBetween: (lo: number, hi: number) -> (),
	toBeTruthy: () -> (),
	toBeFalsy: () -> (),
	toBeNil: () -> (),
	toContain: (item: any) -> (),            -- substring for strings, table.find for arrays
	toMatch: (pattern: string) -> (),        -- Lua pattern
	toThrow: (pattern: string?) -> (),       -- actual must be a function
}
export type TestApi = {
	describe: (label: string, fn: () -> ()) -> (), -- grouping label only; never part of the id
	it: (id: string, fn: () -> (), opts: TestOpts?) -> (),
	pending: (id: string, taskId: string) -> (),    -- reserved id for a not-yet-implemented test
	expect: (actual: any) -> Matchers,
	fail: (msg: string) -> never,
}
local T = {}
local ID_PATTERN = "^[%w_]+%.[%w_%.%-]+$"
local cases: { TestCase } = {}
local pendings: { { id: string, taskId: string, file: string } } = {}
local seen: { [string]: string } = {}
local function fail(msg: string): never
	error({ alg = "assert", msg = msg }, 0) -- the runner prints .msg
end
local function expect(actual: any): Matchers
	return {
		toBe = function(expected: any)
			if actual ~= expected then
				fail(`expected {tostring(expected)}, got {tostring(actual)}`)
			end
		end,
		toBeNear = function(expected: number, eps: number?)
			local e = eps or 1e-6
			if type(actual) ~= "number" or math.abs(actual - expected) > e then
				fail(`expected {expected} ± {e}, got {tostring(actual)}`)
			end
		end,
		-- toEqual, toBeBetween, toBeTruthy, toBeFalsy, toBeNil, toContain, toMatch, toThrow: same pattern
	} :: any
end
function T.api(file: string): TestApi
	local labels: { string } = {}
	return {
		describe = function(label: string, fn: () -> ())
			table.insert(labels, label)
			fn()
			table.remove(labels)
		end,
		it = function(id: string, fn: () -> (), opts: TestOpts?)
			assert(string.match(id, ID_PATTERN), "bad test id " .. id)
			assert(seen[id] == nil, `duplicate test id {id} ({seen[id]} and {file})`)
			seen[id] = file
			local tags = if opts and opts.tags then opts.tags else {}
			table.insert(cases, { id = id, file = file, label = table.concat(labels, " > "), fn = fn, tags = tags })
		end,
		pending = function(id: string, taskId: string)
			assert(seen[id] == nil, "duplicate test id " .. id)
			seen[id] = file
			table.insert(pendings, { id = id, taskId = taskId, file = file })
		end,
		expect = expect,
		fail = fail,
	}
end
function T.cases(): { TestCase }
	return cases
end
function T.pendings(): { { id: string, taskId: string, file: string } }
	return pendings
end
return T
```

The full file (166 lines, all ten matchers) was run on Lune 0.10.5 during spec writing; the agent writes it from this contract. `toEqual` treats `NaN == NaN` as equal and compares keys both ways.

#### 7.2.4 Loader and datatype shims (`tests/shim/Loader.luau`)

Why a loader: pure modules call `Vector3.new`, `CFrame.Angles`, `Color3.fromRGB` (Parts 3–4). Lune's native `require` gives each module a sandboxed global table without these names, so tests and tools load `src/` modules **only** through `L.require`, which runs each file with `@lune/luau` `luau.load(source, {environment = env})` [Lune `crates/lune-std-luau/types.d.luau`]. The environment holds `Vector3`, `CFrame`, `Color3` from `@lune/roblox` (preferred over hand-written shims) and a per-module `require` with Roblox require-by-string semantics. `game`, `workspace`, `script`, `Instance`, `Enum` and `task` are absent, so impure code fails loudly with its file path.

Resolution (identical to Studio for Script Sync trees, Part 1 §1.2.7):
- `./X` starts at the script's parent: the file's directory, or the parent directory for `init.luau`. `../X` goes one level up from there.
- `@self/X` is valid only in `init.luau` and starts at its directory.
- `@shared/`, `@tests/`, `@tools/` come from the repo-root `.luaurc`.
- Entry calls from a test or tool (not from inside a loaded module) may also name a repo-root path starting with `src/`, e.g. `L.require("src/DevTools/Tests/Lib/Budgets")`.
- `X` resolves to `X.luau` or `X/init.luau`; both existing is an error.

**Lune 0.10.5 bug.** `CFrame.lookAt`, `CFrame.lookAlong` and `CFrame.new(pos, target)` return wrong rotations unless the rotation is purely about Y (fixed upstream in 0.11.0, changelog #419). Reproduced during spec writing: `lookAt(zero, (0,1,-1))` gave LookVector `(0, -0.707, 0.707)`. The loader detects the bug and substitutes a `CFrame.fromMatrix` implementation; `shim.cframe_lookat_reference` guards it.

Verified on Lune 0.10.5:

```lua
--!strict
-- tests/shim/Loader.luau — loads src/ modules in Lune with Roblox require-by-string semantics
-- and the Roblox datatype globals (Vector3, CFrame, Color3). Realm: host. Spec §7.2.4.
local fs = require("@lune/fs")
local luau = require("@lune/luau")
local roblox = require("@lune/roblox")
local serde = require("@lune/serde")

export type Env = { [string]: any }
export type Instance = {
	require: (spec: string) -> any,
	resolve: (spec: string) -> string,
	loaded: () -> { string },
}

local Loader = {}

-- Aliases come from the repo-root .luaurc (Part 1 §1.2.7); paths are relative to the repo root.
local ALIASES: { [string]: string } = {}
do
	local rc = serde.decode("json", fs.readFile(".luaurc"))
	for name, path in rc.aliases or {} do
		ALIASES[string.lower(name)] = (string.gsub(path, "^%./", ""))
	end
end

local function dirname(p: string): string
	return string.match(p, "^(.*)/[^/]*$") or "."
end

local function normalize(p: string): string
	local out: { string } = {}
	for part in string.gmatch(p, "[^/]+") do
		if part == ".." then
			assert(#out > 0, "path escapes repo root: " .. p)
			table.remove(out)
		elseif part ~= "." then
			table.insert(out, part)
		end
	end
	return table.concat(out, "/")
end

local function isInit(file: string): boolean
	return string.match(file, "/init%.luau$") ~= nil
end

-- Roblox semantics: "./" starts at script.Parent. For a/b/X.luau that is a/b; for a/b/init.luau
-- (ModuleScript "b") it is a. "@self/" starts at the script itself (only meaningful for init.luau).
local function resolveSpec(fromFile: string?, spec: string): string
	local base: string
	local rest: string
	local alias, aliasRest = string.match(spec, "^@([%w_%-]+)/(.*)$")
	if alias == "self" then
		assert(fromFile ~= nil and isInit(fromFile), "@self/ is only valid inside init.luau: " .. spec)
		base, rest = dirname(fromFile :: string), aliasRest :: string
	elseif alias ~= nil then
		local root = ALIASES[string.lower(alias)]
		assert(root ~= nil, "unknown alias @" .. alias)
		base, rest = root, aliasRest :: string
	elseif string.sub(spec, 1, 2) == "./" or string.sub(spec, 1, 3) == "../" then
		assert(fromFile ~= nil, "relative require needs a calling file: " .. spec)
		local f = fromFile :: string
		base = if isInit(f) then dirname(dirname(f)) else dirname(f)
		rest = spec
	elseif fromFile == nil and string.match(spec, "^src/") then
		base, rest = ".", spec -- entry point by repo-root path, e.g. "src/DevTools/Tests/Lib/Budgets"
	else
		error("require path must start with ./, ../ or @: " .. spec)
	end
	local stem = normalize(base .. "/" .. rest)
	local asFile, asDir = stem .. ".luau", stem .. "/init.luau"
	local a, b = fs.isFile(asFile), fs.isFile(asDir)
	assert(not (a and b), "ambiguous module " .. stem)
	assert(a or b, "module not found: " .. stem .. " (from " .. tostring(fromFile) .. ")")
	return if a then asFile else asDir
end

-- Lune 0.10.5 returns wrong rotations from CFrame.lookAt / lookAlong / new(pos, target) when the
-- rotation is not purely about Y (fixed upstream in 0.11.0). Patch only when the bug is detected.
local function lookAtFixed(at: any, target: any, up: any?): any
	local look = (target - at).Unit
	local right = look:Cross(up or roblox.Vector3.yAxis).Unit
	return roblox.CFrame.fromMatrix(at, right, right:Cross(look), -look)
end
local function lookAtBroken(): boolean
	local cf = roblox.CFrame.lookAt(roblox.Vector3.zero, roblox.Vector3.new(0, 1, -1))
	return (cf.LookVector - roblox.Vector3.new(0, 1, -1).Unit).Magnitude > 1e-4
end
local CFrameGlobal: any = roblox.CFrame
if lookAtBroken() then
	CFrameGlobal = setmetatable({
		lookAt = lookAtFixed,
		lookAlong = function(at: any, dir: any, up: any?): any
			return lookAtFixed(at, at + dir, up)
		end,
		new = function(...: any): any
			local a, b = ...
			if select("#", ...) == 2 and typeof(a) == "Vector3" and typeof(b) == "Vector3" then
				return lookAtFixed(a, b)
			end
			return roblox.CFrame.new(...)
		end,
	}, { __index = roblox.CFrame })
end
Loader.cframePatched = CFrameGlobal ~= roblox.CFrame

local BASE_ENV: Env = { Vector3 = roblox.Vector3, CFrame = CFrameGlobal, Color3 = roblox.Color3 }

function Loader.new(extraEnv: Env?): Instance
	local cache: { [string]: any } = {}
	local loading: { [string]: boolean } = {}
	local order: { string } = {}

	local function load(file: string): any
		if cache[file] ~= nil then
			return cache[file]
		end
		assert(not loading[file], "require cycle at " .. file)
		loading[file] = true
		local env: Env = table.clone(BASE_ENV)
		for k, v in extraEnv or {} do
			env[k] = v
		end
		env.require = function(spec: string): any
			return load(resolveSpec(file, spec))
		end
		local chunk = luau.load(fs.readFile(file), { debugName = file, environment = env })
		local ok, result = pcall(chunk)
		loading[file] = nil
		if not ok then
			error(string.format("loading %s failed: %s", file, tostring(result)), 0)
		end
		cache[file] = if result == nil then true else result
		table.insert(order, file)
		return cache[file]
	end

	return {
		require = function(spec: string): any
			return load(resolveSpec(nil, spec))
		end,
		resolve = function(spec: string): string
			return resolveSpec(nil, spec)
		end,
		loaded = function(): { string }
			return table.clone(order)
		end,
	}
end

local default = Loader.new()
Loader.require = default.require
Loader.resolve = default.resolve

return Loader
```

`Loader.new(extraEnv)` gives a fresh module graph with extra globals for fakes (e.g. Part 3's `query.sweep_excludes` passes a fake `workspace` table). Modules loaded through different `Loader` instances are different tables; never mix them in one test.

#### 7.2.5 Spec file format

```lua
--!strict
-- tests/wish/gdd.spec.luau — GDD cross-checks for the Wish tables. Spec §7.4.5.
local TT = require("@tests/shim/T")
local L = require("@tests/shim/Loader")
type OddsMod = typeof(require("@shared/Wish/Odds")) -- type only: erased at run time

return function(T: TT.TestApi)
	local Defaults = L.require("@shared/Config/Defaults")
	local Odds = L.require("@shared/Wish/Odds") :: OddsMod

	T.it("wish.gdd_tier_odds", function()
		local o = Defaults.Wish.TierOdds -- ppm (Part 5 §5.12 item 2)
		T.expect({ o.common, o.uncommon, o.rare, o.epic, o.legendary, o.mythic, o.cosmic })
			.toEqual({ 500000, 280000, 140000, 60000, 16000, 3900, 100 }) -- GDD §8.3
	end)
	T.it("wish.cosmic_path_bound_4400", function()
		-- worst case: d1 = 0 every roll (always the first non-zero tier) -> only floors produce Mythic+
		local ctx = Odds.context(Defaults.Wish, Odds.buildPool(L.require("@tests/core/fixtures/pool_m1g")))
		-- iterate Odds.tierTable/Odds.pickPpm(ppm, 0)/Pity.advance; count Mythic+; assert the 11th Mythic+ arrives at Wish 4,400
	end, { tags = { "slow" } })
end
```

Rules: one spec file per module or concern; test IDs as in §7.1.4; no `print` in tests (the runner prints); no file writes outside `build/`; tests never require `@shared/...` natively (lint `test-loader`, §7.5.5).

#### 7.2.6 What is not testable on the host

| Not testable in Lune | Where it is tested instead |
|---|---|
| Instances, services, attributes, replication, `BindToSimulation`, rollback/resim, `time()` | L2/L4 scenarios |
| InputAction System, `VirtualInput`, MCP input | L3, client driver |
| Physics movers, spatial queries against real geometry (Lune only tests query *logic* with fakes) | L2 (`combat.*`) |
| ProfileStore, DataStore, MarketplaceService, PolicyService, ConfigService, AnalyticsService | L2 with mocks and Studio seams |
| Rendering, VFX budgets, UI layout, frame time | L3, L6, L7 |
| Runtime performance (Lune runs without native codegen; numbers do not transfer) | L6 |
| Luau type errors | `luau-lsp analyze` (host gate) |
| Exact float equality of datatypes (both use 32-bit floats but different math paths) | Compare with tolerance ≥ 1e-4 |

#### 7.2.7 Acceptance (§7.2)

- `lune run tests/run.luau` prints one result line per test plus `SUMMARY`, and exits 0/1/2 as specified. **Verify:** `runner.cli_contract` (subprocess runs against `tests/core/fixtures/runner/` with `--root`, checking the exit codes, `--list`, `--fast` and duplicate-id detection).
- The loader passes `shim.*` (§7.4.2).
- The full suite finishes in ≤ 60 s on `ubuntu-24.04` (CI log timestamp).

---

### 7.3 In-Studio TestHarness

#### 7.3.1 Components

| Component | Path (Studio path) | Runs in | Purpose |
|---|---|---|---|
| `TestHarness` service | `src/ServerScriptService/Services/TestHarness.luau` | Server, Studio only | Boot hook, request watcher, `Dev.Command` host, `run()` API (Part 2 §2.3.5) |
| Types | `src/DevTools/Tests/Lib/Types.luau` | Server | Scenario, Ctx, Fighter, report types |
| Registry | `src/DevTools/Tests/Lib/Registry.luau` | Server | Loads suites, resolves selectors |
| Runner | `src/DevTools/Tests/Lib/Runner.luau` | Server | Runs scenarios: needs, timeouts, teardown, report JSON |
| Ctx and fixtures | `Lib/Ctx.luau`, `Lib/Fixtures.luau`, `Lib/Tap.luau` | Server | Scenario context, TestPad, bots, tap recorder |
| Dev commands | `Lib/DevCommands.luau` | Server | `Dev.Command` dispatch table (Part 2 §2.11.5 + Parts 3/5/7 additions) |
| Multi | `Lib/Multi.luau` | Server | Multi-client session, roles, client reports, `EndTest` |
| Perf | `Lib/PerfSampler.luau`, `Lib/Budgets.luau` | Server | §7.6 |
| Network profiles | `Lib/NetProfiles.luau` | Edit (snippet), Server | §7.1.5 |
| Client driver | `src/DevTools/Tests/Client/Driver.luau`, `Client/Steps.luau`, `Client/PerfProbe.luau` | Client (cloned at run time) | Client steps, input, reports (§7.3.9) |
| Suites | `src/DevTools/Tests/Suites/<Name>.luau` | Server | One module per domain, written by the owning part's tasks |
| Spikes | `src/DevTools/Tests/Spike_<ID>.luau` (dashes removed: `Spike_SA07`, `Spike_SP4_2`, `Spike_SP6_2`; scenario id `spike.<ID>`, TASKS C4) | Server | Part 3 §3.16.3 (fixed path) for `Spike_SAxx`; Part 4 §4.11 S-P4-2, S-P4-3; Part 6 §6.15 S-P6-2 |
| Plugin | `tools/plugin/AlgebraicTestHarness/Main.luau` | Edit, plus client DMs | Launches multi-client tests; misprediction attribution (§7.3.8) |
| Snippets | `tools/studio/test_run.luau`, `test_result.luau`, `multi_run.luau`, `net_profile.luau`, `device_sim.luau`, `mispred_probe.luau` | via `execute_luau` | Agent entry points |

Suite files (names fixed; owner part writes the scenarios, Part 7 writes the framework and the suites marked P7): `Boot`, `Config`, `Net`, `Data`, `Player`, `Security`, `Tree` (P2); `Sim`, `Input`, `CombatM1`, `CombatDefense`, `CombatMovement`, `CombatStun`, `CombatMeter`, `CombatRealm`, `CombatKo`, `CombatRewind`, `Bot`, `Render` (P3); `Style`, `Kit`, `Anim`, `Look` (P4); `Wish`, `Policy`, `Shop`, `Progress`, `Store`, `Appearance`, `Tel`, `Mod` (P5); `Harness`, `NetCond`, `MultiClient`, `Mcp`, `Perf` (P7). Part 6 adds `Ui`, `World`, `Vfx` as needed under the same rules.

#### 7.3.2 Scenario format

```lua
--!strict
-- src/DevTools/Tests/Lib/Types.luau — harness types. Realm: server (Studio). Spec §7.3.2.
export type Level = "studio" | "multi" | "mcp" | "perf"
export type McpStep =
	{ kind: "key", action: "down" | "up" | "press", key: string }        -- Enum.KeyCode name, e.g. "F", "One"
	| { kind: "mouse", action: "click" | "down" | "up", target: string } -- "viewport-center" or a UI instance path
	| { kind: "wait", sec: number }
	| { kind: "navigate", target: string }                             -- instance path or "x,y,z"
	| { kind: "capture", note: string }                                -- screen_capture for the evidence log
	| { kind: "snippet", file: string, datamodel: "Edit" | "Server" | "Client", args: { [string]: any } } -- tools/studio/<file>
export type Brain = (f: number, e: number) -> (number, Vector3?, Vector3?) -- (Fsm.IN bits, aim, moveDir); e = frames since set
export type Fighter = {
	model: Model, root: BasePart, rid: number, player: Player?, styleId: string,
	attr: (name: string) -> any,                -- HRP attribute by Attr.N name (Part 3 §3.3)
	attrs: () -> { [string]: any },
	patch: (p: { [string]: any }) -> (),         -- CombatService._queuePatch (applied at the next server step)
	place: (cf: CFrame) -> (),                   -- setup-only teleport outside the sim; waits 2 frames
	brain: (b: Brain?) -> (),                    -- bots only: BotService._setBrain; nil restores the default brain
}
export type Pad = {
	origin: CFrame, a: CFrame, b: CFrame, behindB: CFrame,
	wallAttacker: CFrame, wallVictim: CFrame, bench: CFrame,
}
export type TapRecord = {
	k: string, f: number, rid: number, a: number?, b: number?, s: string?, -- Part 3 SimEvent fields
	victim: number?, outcome: string?, damage: number?, moveId: string?,  -- k == "contact" only
}
export type Recorder = {
	all: () -> { TapRecord },
	of: (k: string, rid: number?, victim: number?) -> { TapRecord },
	first: (k: string, rid: number?) -> TapRecord?,
	stop: () -> (),
}
export type Ctx = {
	name: string, runId: string, args: { [string]: any }, state: { [string]: any }, net: string,
	frame: () -> number,                                   -- Frames.fromTime(time())
	waitFrames: (n: number) -> (),
	waitUntil: (pred: () -> boolean, timeoutF: number, what: string) -> (),
	expect: (cond: boolean, msg: string) -> (),            -- hard: aborts the scenario
	eq: (actual: any, expected: any, what: string) -> (),  -- hard
	near: (actual: number, expected: number, eps: number, what: string) -> (), -- hard
	check: (cond: boolean, msg: string) -> (),             -- soft: recorded, scenario continues
	metric: (key: string, value: number) -> (),            -- report.metrics["<scenario>.<key>"]
	cfg: () -> any,                                        -- Runtime.get()
	pad: () -> Pad,
	bot: (spec: { styleId: string, at: CFrame, kind: ("dummy" | "brawl")? }) -> Fighter,
	dummy: (mode: string, at: CFrame) -> Fighter,          -- Training.Dummy modes (Part 3 §3.15.2)
	player: (index: number?) -> Fighter,                   -- real players in join order (solo: the local player)
	tap: () -> Recorder,
	trace: (f: Fighter, attrs: { string }) -> { [number]: { [string]: any } }, -- frame -> sampled attrs + Pos
	dev: (player: Player?, cmd: string, args: { [string]: any }?) -> any,      -- DevCommands, auto-reverted
	remote: (player: Player, name: string, payload: any) -> any,               -- Net.server._dispatch
	clients: () -> { [string]: Player },                   -- role ("A".."H") -> Player
	clientStep: (role: string, step: string, params: { [string]: any }?) -> (),
	report: (role: string, key: string, timeoutF: number) -> any,
	perf: () -> any,                                       -- PerfSampler.start() (§7.6.2)
	log: (msg: string) -> (),                              -- [ALG][Test] <scenario> <msg>
}
export type Scenario = {
	name: string, level: Level, tags: { string }, needs: { string }?, timeoutF: number?,
	clients: number?,                                      -- multi/perf: minimum client count
	mcp: { McpStep }?,                                     -- level "mcp": steps the agent performs between setup and assert
	fallback: string?,                                     -- level "mcp": client step used when MCP input cannot reach IAS (§7.3.7)
	setup: ((ctx: Ctx) -> ())?, act: ((ctx: Ctx) -> ())?, assert: (ctx: Ctx) -> (), teardown: ((ctx: Ctx) -> ())?,
}
export type Suite = { suite: string, scenarios: { Scenario } }
export type ScenarioResult = { name: string, ok: boolean, detail: string, frames: number, ms: number }
export type RunReport = {
	v: number, id: string, mode: "solo" | "multi" | "inline", op: string, sel: string,
	state: "queued" | "running" | "ready" | "done" | "error",
	pass: number, fail: number, results: { ScenarioResult }, metrics: { [string]: number },
	clients: { [string]: { [string]: number } }?, net: string?, hints: { McpStep }?, err: string?,
}
return {}
```

Example scenario (the format every part uses):

```lua
--!strict
-- src/DevTools/Tests/Suites/CombatDefense.luau — defense scenarios (Part 3 §3.6). Format: Spec §7.3.2.
local T = require("../Lib/Types")
local Fx = require("../Lib/Fixtures")

local suite: T.Suite = {
	suite = "combat_defense",
	scenarios = {
		{
			name = "combat.block_front_back",
			level = "studio",
			tags = { "m0" },
			needs = { "sa" },
			timeoutF = 900,
			setup = function(ctx: T.Ctx)
				local pad = ctx.pad()
				ctx.state.att = ctx.bot({ styleId = "hero", at = pad.a })
				ctx.state.def = ctx.dummy("block", pad.b)
				ctx.state.rec = ctx.tap()
			end,
			act = function(ctx: T.Ctx)
				local att: T.Fighter = ctx.state.att
				att.brain(Fx.timeline({ { at = 0, press = "M1" } }))
				ctx.waitFrames(60)
				att.place(ctx.pad().behindB)
				att.brain(Fx.timeline({ { at = 0, press = "M1" } }))
				ctx.waitFrames(60)
			end,
			assert = function(ctx: T.Ctx)
				local def: T.Fighter = ctx.state.def
				local hits = (ctx.state.rec :: T.Recorder).of("contact", nil, def.rid)
				ctx.eq(#hits, 2, "contacts on the blocking dummy")
				ctx.eq(hits[1].outcome, "block", "front M1 outcome")
				ctx.eq(hits[1].damage, 0, "front M1 damage (GDD §6.4)")
				ctx.eq(hits[2].outcome, "hit", "rear M1 outcome (GDD §6.4)")
				ctx.near(hits[2].damage or 0, 3.5, 0.05, "rear M1 damage (GDD §6.3)")
				ctx.eq(def.attr("Grd"), 88, "guard after one blocked M1 (GDD §6.4)")
			end,
		},
	},
}
return suite
```

Authoring rules (all parts):
1. Every number in an assertion cites `GDD §x.y`, a spec section, or `(td)`.
2. Fixtures only through `ctx` (bots, dummies, pad, patches, dev commands); never `Instance.new` of gameplay objects in a suite.
3. Assert on tap records and attributes, not on log text, except where the owning part fixes a log line.
4. A scenario that changes config, policy, passes, time offset or seeds does it through `ctx.dev`, which reverts it in teardown.
5. `timeoutF` defaults to 1,800 (30 s, td); maximum 36,000 (10 min) for perf.

#### 7.3.3 Ctx, fixtures and seams

**TestPad.** `ctx.pad()` creates, once per run, a runtime-only Model `Workspace.TestPad` centered at `(2000, 100, 2000)` (td, far from every map): an anchored 200 × 2 × 200 floor, an anchored wall 4 × 24 × 80 whose face is 10 studs in front of `wallVictim`, all in collision group `Default` (map geometry, Part 3 §3.12.1). Markers (CFrames, facing −Z unless stated): `a` = floor center, `b` = `a` + 5 studs forward facing `a`, `behindB` = 3 studs behind `b` facing `b`'s back, `wallAttacker`/`wallVictim` = a lane perpendicular to the wall with the victim 10 studs from it, `bench` = 300 studs away. Before the first scenario the runner moves every real player to `bench` and patches `ProtF = 0` only when the scenario asks for the player.

**Bots and brains.** `ctx.bot` calls `BotService.spawnAsync({kind = spec.kind or "brawl", styleId, mapId = <current map>})` (Part 3 §3.15.1), places it, and installs an idle brain. `spawnAsync` returns `Model?`: it returns `nil` when a kill switch refuses the bot (`Features.Bots == false` refuses `brawl` bots; dummies are still allowed). `ctx.bot` and `ctx.dummy` then fail the scenario hard with `bot refused kind=<kind>` and never return a nil `Fighter`. A scenario that tests the refusal itself (`bot.loadtest_off_in_studio`) calls `BotService.spawnAsync` directly. `Fighter.brain(b)` calls the Studio seam `BotService._setBrain(model, b)`: BotService's intent provider calls `b(f, e)` inside the sim on the server and writes `BIn`/`BAim`/`BMv` from its result. Brains must not yield and must be deterministic. `Fixtures.timeline(steps)` builds a brain from `{ { at: number, press: string?, hold: number?, mv: Vector3?, aim: Vector3? } }` (`press` is an `InputMap` action name; `hold` default 2 frames). `Fixtures.IN` re-exports `Fsm.IN`.

**Tap.** `ctx.tap()` subscribes to the Studio seam `CombatService._tap(fn) -> () -> ()`. Part 3's flush calls `fn` for every `SimEvent` in queue order and once per resolved contact with `k = "contact"`, `rid` = attacker, `victim`, `outcome` (`Damage.Outcome`), `damage` (after scaling) and `moveId`. Records arrive after the step that produced them, carrying that step's `f`.

**Trace.** `ctx.trace(fighter, attrs)` samples the named HRP attributes plus `Pos` (root position as `{x, y, z}`) on `RunService.PostSimulation`, keyed by frame, for the rest of the scenario (ring of 3,600 frames).

**Dev commands.** `ctx.dev(player, cmd, args)` calls the same handler table as the `Dev.Command` remote (`Lib/DevCommands.luau`). Reversible commands are undone in teardown, in reverse order:

| Command | Teardown action |
|---|---|
| `setConfig(section, json)` | Re-apply the section JSON captured before the first change; for `Combat` or `Kits` also respawn every real player (pinned config applies at respawn, Part 2 §2.6.6) |
| `setPolicy`, `setPass`, `setTimeOffset`, `setWishSeedMaster` | Restore the previous value (`setWishSeedMaster` → 0) |
| `spawnBots` | `clearBots` |
| `grant`, `giveStyle`, `setPity`, `setLevel`, `setMastery`, `resetProfile` | `resetProfile` for every touched player (needs `mockData`) |
| `combatPatch`, `setMeter`, `kill` | Respawn the touched player with `CombatService.spawnAsync` |

**Remote dispatch.** `ctx.remote(player, name, payload)` calls the Studio seam `Net.server._dispatch(player, name, payload)`, which runs the full server pipeline (bucket, validator, ready check, handler) as if the client had sent it, and returns the invoke response (or `nil` for events). End-to-end client calls use the client driver (§7.3.9).

**Preconditions (`needs`).**

| Need | Check | On failure |
|---|---|---|
| `sa` | `Runtime.get().NetModel == "ServerAuthority"` and the server log history contains a `[ALG][Sim] bound hz=60` line (tag compared case-insensitively) | FAIL `unmet need sa` |
| `anims` | Every `anim.*` key used by the implemented kits resolves to `rbxassetid://…` (H-08) | FAIL |
| `mockData` | `Defaults.Data.StudioLiveData == false` | FAIL |
| `clients:<n>` | ≥ n players with `DataLoaded == true` | FAIL |
| `kit:<id>` | `Styles.isPlayable(id, Runtime.get())` (Part 4 §4.1.2; full config) | FAIL |
| `map:<mapId>` | `Fixtures.hasMap(mapId)` using Part 6's map marker | FAIL |
| `net:<profile>` | Request field `net` equals the profile | FAIL |

With request arg `allowSkip = true` (never in gate runs) an unmet need prints `[ALG][Test] skip id=<name> reason=unmet:<need>` and is not counted.

#### 7.3.4 Runner semantics

- **Registration:** `Registry.load()` requires every ModuleScript under `DevTools.Tests.Suites` and every `DevTools.Tests.Spike_*` once per play session; each returns a `Suite`. A duplicate scenario name, a name not matching the ID pattern (§7.1.4), or a scenario without exactly one milestone tag fails the load: every request then ends with `state = "error"` and `err` naming the problem.
- **Selectors** (`sel`): comma-separated terms; a term is an exact name, a prefix `combat.*`, or a tag `@m0`. Terms without `-` are unioned; terms with a leading `-` are then removed (`@m0,-@slow` = every `m0` scenario except `slow` ones). There is no intersection operator. Order = registry order (suites alphabetical, scenarios as declared). A bare word is **not** a prefix: `combat` is looked up as the exact name `combat` (which no scenario has); write `combat.*`. If the whole selection matches zero scenarios, the request ends with `state = "error"`, `err = "empty_selection"` (never `done` with `pass = 0`).
- **Level filter:** in `op = "run"`, `mcp` scenarios are skipped (`[ALG][Test] skip … reason=mcp_needs_setup_assert`); `multi` scenarios and `perf` scenarios with `clients ≥ 2` are skipped outside multi mode.
- **Execution** of one scenario: check `needs` → spawn a thread running `setup`, `act`, `assert` → wait on `RunService.PostSimulation` until done or `frame() ≥ start + timeoutF` → on timeout `task.cancel` the thread → always run teardown (`sc.teardown`, reversible dev commands, Trove cleanup: bots, TestPad children, tap and trace connections, `ReplicatedStorage.TestRun` steps) → result.
- **Result:** `ok` = no hard failure, no soft failures, teardown succeeded. `detail` = the first failure message (≤ 200 chars) or `""`. Output via Part 2 `Log.test(name, ok, detail)`; after the selection `Log.summary(pass, fail)`.
- **Report:** after every scenario the runner rewrites `ServerStorage.AgentTestResult.Value` with the `RunReport` JSON (`state = "running"`), and finally with `state = "done"` (or `"error"` if the runner itself failed). A StringValue is used, not an attribute, because under Server Authority string attribute values replicate only up to 50 characters (attribute names ≤ 50 characters; only the first 64 attributes of an instance replicate) [Docs `projects/server-authority/index.md#attribute-limits`], while a StringValue holds 200,000 characters. (The "100 characters or less" rule in `Instance:SetAttribute` is a naming requirement for attribute names, not a cap on string values.)

Core of `Runner.runOne` (abridged):

```lua
--!strict
local RunService = game:GetService("RunService")
local function runOne(sc: T.Scenario, req: Request): T.ScenarioResult
	local ctx = Ctx.new(sc, req)
	local unmet = Ctx.checkNeeds(sc, req)
	if unmet then
		return Ctx.finish(ctx, false, "unmet need " .. unmet, 0, 0)
	end
	local t0, f0 = os.clock(), ctx.frame()
	local done, okRun, errRun = false, true, nil :: any
	local thread = task.spawn(function()
		local ok, err = pcall(function()
			if sc.setup then sc.setup(ctx) end
			if sc.act then sc.act(ctx) end
			sc.assert(ctx)
		end)
		okRun, errRun, done = ok, err, true
	end)
	local limit = f0 + (sc.timeoutF or 1800)
	while not done and ctx.frame() < limit do
		RunService.PostSimulation:Wait()
	end
	if not done then
		task.cancel(thread)
		okRun, errRun = false, `timeout after {sc.timeoutF or 1800}f`
	end
	local okTd, errTd = pcall(Ctx.teardown, ctx)
	local msg = if not okRun then Ctx.message(errRun) elseif not okTd then "teardown: " .. tostring(errTd) else Ctx.firstSoftFailure(ctx)
	return Ctx.finish(ctx, msg == nil, msg or "", ctx.frame() - f0, math.floor((os.clock() - t0) * 1000))
end
```

#### 7.3.5 `TestHarness` service and Studio-only guards

```lua
--!strict
-- src/ServerScriptService/Services/TestHarness.luau — Studio-only scenario runner and Dev.Command host.
-- Realm: server (Studio only). Pure: no. Spec §7.3.5, §2.11.5.
local HttpService = game:GetService("HttpService")
local RunService = game:GetService("RunService")
local ServerStorage = game:GetService("ServerStorage")
local StudioTestService = game:GetService("StudioTestService")

export type RunCounts = { pass: number, fail: number }
local TestHarness = {}
TestHarness._isStudio = RunService:IsStudio() -- test seam for harness.studio_only_guard

local runnerMod: any = nil
local function runner(): any
	if runnerMod == nil then -- DevTools is required lazily and only here (§7.5.5 lint devtools-isolation)
		local dt = ServerStorage:FindFirstChild("DevTools")
		local lib = dt and dt:FindFirstChild("Tests") and (dt :: any).Tests:FindFirstChild("Lib")
		local mod = lib and lib:FindFirstChild("Runner")
		assert(mod ~= nil and mod:IsA("ModuleScript"), "DevTools.Tests.Lib.Runner missing")
		runnerMod = require(mod) :: any
	end
	return runnerMod
end

local function watchSolo()
	local function hook(inst: Instance)
		if inst.Name ~= "AgentTestRequest" or not inst:IsA("StringValue") then
			return
		end
		local sv = inst :: StringValue
		local function onValue()
			local ok, req = pcall(function()
				return HttpService:JSONDecode(sv.Value)
			end)
			if ok and type(req) == "table" and req.mode == "solo" then
				task.spawn(runner().handleRequest, req)
			end
		end
		sv:GetPropertyChangedSignal("Value"):Connect(onValue) -- module-lifetime connection
		onValue()
	end
	ServerStorage.ChildAdded:Connect(hook) -- module-lifetime connection
	local existing = ServerStorage:FindFirstChild("AgentTestRequest")
	if existing then
		hook(existing)
	end
end

function TestHarness.init()
	if not TestHarness._isStudio then
		return
	end
end

function TestHarness.start()
	if not TestHarness._isStudio then
		return
	end
	runner().installDevCommand() -- Net.server.handle("Dev.Command", …) (Part 2 §2.11.5)
	runner().recordJoins()       -- PlayerAdded timestamps for perf.boot_time_to_fight
	local okArgs, raw = pcall(function()
		return StudioTestService:GetTestArgs()
	end)
	if not (okArgs and type(raw) == "string") then
		local copy = ServerStorage:FindFirstChild("AgentTestMultiArgs") -- spike T7-S14 fallback (written by the plugin)
		raw = if copy and copy:IsA("StringValue") then (copy :: StringValue).Value else nil
	end
	if type(raw) == "string" then
		local ok, req = pcall(function()
			return HttpService:JSONDecode(raw)
		end)
		if ok and type(req) == "table" and req.alg == 1 and req.mode == "multi" then
			task.spawn(runner().runMulti, req) -- always ends with StudioTestService:EndTest(<json>)
			return
		end
	end
	watchSolo()
end

function TestHarness.run(scenario: string, args: { [string]: any }?): RunCounts
	if not TestHarness._isStudio then
		return { pass = 0, fail = 0 }
	end
	local rep = runner().runSelection({
		v = 1, id = HttpService:GenerateGUID(false), mode = "inline", op = "run", sel = scenario, args = args or {},
	})
	return { pass = rep.pass, fail = rep.fail }
end

return TestHarness
```

Studio-only guards (all MUST):
1. `TestHarness.init/start/run` do nothing unless `RunService:IsStudio()` (seam `_isStudio`).
2. `ServerStorage.DevTools` is required only by `TestHarness.luau`; `ReplicatedStorage.TestRun` is referenced only by `TestHarness`, `DevTools/**` and the `Main.client.luau` hook (lint `devtools-isolation`, §7.5.5).
3. `Dev.Command` exists only in Studio (Part 2 §2.11.5); the client hook (§7.3.9) runs only when `RunService:IsStudio()`.
4. Every Studio seam used here (`_tap`, `_setBrain`, `_dispatch`, `_queuePatch`, `_setMeter`, `_getMeter`, `_setRtt`, `_setSeedMaster`, `_forceFail`, `_setPerfSampled`, `_leaveTest`, `_readStored`, `_devAccessCheck`, `_devContextOverride`) is a no-op unless `RunService:IsStudio()` (`_getMeter` returns 0, `_readStored` returns nil and `_leaveTest` returns false outside Studio; `_devAccessCheck` ignores its `opts` there). `CombatService._getMeter(target: Player | Model) -> number` (Part 3 §3.13.7) reads the server-memory meter of a player, bot or dummy; `combat.perfect_block` and `combat.assist_credit` use it.
5. `UserInputService:CreateVirtualInput()` is used only in `DevTools/Tests/Client/**` and in the P0 probe files `DevTools/Tests/Spikes/**` and `DevTools/Tests/Spike_*.luau` (exempt per decision D-12), and its result is nil-checked [Docs `UserInputService.yaml`: "or nil if the feature is not available"].
6. **Verify:** `harness.studio_only_guard`, `security.dev_hooks_studio_only` (Part 2), `standards.devtools_isolation`.

#### 7.3.6 Agent trigger protocol (solo Play)

**Request** (`ServerStorage.AgentTestRequest`, StringValue, written in the **Server** DataModel):
`{"v":1,"id":"<guid>","mode":"solo","op":"run"|"setup"|"assert","sel":"<selector>","args":{…}}`

**Result** (`ServerStorage.AgentTestResult`, StringValue): the `RunReport` JSON of §7.3.2.

`tools/studio/test_run.luau` (Server; 894 bytes):

```lua
--!strict
-- tools/studio/test_run.luau | Server | ARGS {"op":"run","sel":"combat.*","args":{},"id":null}
local HS = game:GetService("HttpService")
local SS = game:GetService("ServerStorage")
local A = HS:JSONDecode([==[__ARGS__]==])
local function sv(name: string): StringValue
	local v = SS:FindFirstChild(name)
	if v and v:IsA("StringValue") then
		return v :: StringValue
	end
	local n = Instance.new("StringValue")
	n.Name = name
	n.Parent = SS
	return n
end
local op: string = A.op or "run"
local id: string = if op == "assert" then A.id else HS:GenerateGUID(false)
if op ~= "assert" then
	sv("AgentTestResult").Value = HS:JSONEncode({ v = 1, id = id, state = "queued" })
end
sv("AgentTestRequest").Value = HS:JSONEncode({
	v = 1,
	id = id,
	mode = "solo",
	op = op,
	sel = A.sel,
	args = A.args or {},
})
return HS:JSONEncode({ ok = game:GetService("RunService"):IsRunning(), id = id })
```

`tools/studio/test_result.luau` (Server for solo, Edit for multi; 1,295 bytes):

```lua
--!strict
-- tools/studio/test_result.luau | Server (solo) or Edit (multi) | ARGS {"id":"<id>","page":1,"consume":false}
local HS = game:GetService("HttpService")
local SS = game:GetService("ServerStorage")
local A = HS:JSONDecode([==[__ARGS__]==])
local v = SS:FindFirstChild("AgentTestResult")
if not (v and v:IsA("StringValue")) then
	return HS:JSONEncode({ ok = true, state = "absent" })
end
local ok, r = pcall(function()
	return HS:JSONDecode((v :: StringValue).Value)
end)
if not ok or type(r) ~= "table" then
	return HS:JSONEncode({ ok = false, err = "bad_result_json" })
end
if A.id ~= nil and r.id ~= A.id then
	return HS:JSONEncode({ ok = true, state = "other", id = r.id })
end
local res = r.results or {}
local failed = {}
for _, x in res do
	if not x.ok then
		table.insert(failed, x)
	end
end
local p: number = A.page or 1
local done = r.state == "done" or r.state == "error"
if A.consume and done then
	v:Destroy()
	local q = SS:FindFirstChild("AgentTestRequest")
	if q then
		q:Destroy()
	end
end
return HS:JSONEncode({
	ok = true,
	id = r.id,
	state = r.state,
	pass = r.pass,
	fail = r.fail,
	n = #res,
	failed = failed,
	page = table.move(res, (p - 1) * 40 + 1, math.min(p * 40, #res), 1, {}),
	metrics = if p == 1 then r.metrics else nil,
	hints = r.hints,
	err = r.err,
})
```

**Procedure** (extends Part 1 §1.4.5; P-numbers are Part 1's):
1. P1–P2 (preconditions, `clear_output.luau`). If the card is `net`-tagged, run `net_profile.luau` `{"profile":"<name>"}` (Edit) now.
2. P4–P7 (start play; wait for `boot_status` and `client_status`).
3. `execute_luau` (Server) `test_run.luau` with `{"op":"run","sel":"<selector>"}` → keep `id`.
4. Poll `execute_luau` (Server) `test_result.luau` `{"id":"<id>"}` at most every 2 s until `state` is `done` or `error`. Maximum polls = ⌈Σ timeoutF / 120⌉ + 10.
5. If `n > 40`, read pages 2… for the evidence log; failures are always on page 1 in `failed`.
6. P9–P13 (`log_scan` Server and Client, `get_console_output`, stop, wait for Edit). Then `net_profile.luau` `{"profile":"Off"}` if a profile was applied.
7. Pass rule: `state == "done"`, `fail == 0`, `pass` ≥ the card's expected count, and both `log_scan` results `ok`.

Part 5's recipe `require(game.ServerScriptService.Services.TestHarness).run(...)` from `execute_luau` is allowed only after spike T7-S4 shows that `execute_luau` (Server) shares the game's module cache and that a run completes inside the tool timeout. Otherwise use the request/result protocol above.

#### 7.3.7 MCP input-driven protocol (level `mcp`)

1. Start play as in §7.3.6 steps 1–2.
2. `test_run.luau` `{"op":"setup","sel":"<one mcp scenario>"}` → `id`. Poll `test_result.luau` until `state == "ready"`; read `hints` (the scenario's `mcp` steps).
3. Perform each hint with the live MCP tools, mapping fields to the schema in `docs/agent/mcp-tools.snapshot.json` (Part 1 §1.5.1): `key` → `user_keyboard_input`, `mouse` → `user_mouse_input` (`viewport-center` = centre coordinates of the capture size), `navigate` → `character_navigation`, `wait` → the tool's wait action (or a separate call after the delay), `capture` → `screen_capture` (≤ 3 per task, Part 1 §1.5.8), `snippet` → `execute_luau` with `tools/studio/<file>` in the named datamodel and the given ARGS.
4. `test_run.luau` `{"op":"assert","id":"<id>"}` → poll until `done`.
5. The harness waits up to `timeoutF` between `ready` and `assert`; on expiry the scenario FAILs with `mcp assert not received`.
6. If MCP input does not reach IAS bindings (Part 1 spike S9 negative), the agent sends `{"op":"setup","sel":"<name>","args":{"useFallback":true}}`; the harness then runs the scenario's `fallback` client step (Script bindings via the client driver) instead of waiting for MCP input, and appends ` (fallback)` to `detail`. The result still counts, but the gate records the fallback.

Assertions in `mcp` scenarios read tap records and traces collected since `setup`. They never depend on how fast the agent's calls arrive.

#### 7.3.8 Multi-client protocol

**Launch paths.**
- **Option A (direct).** `multi_run.luau` with `"via":"direct"` calls `StudioTestService:ExecuteMultiplayerTestAsync(n, req)` inside `task.spawn` from `execute_luau` (Edit). The method is `PluginSecurity` and documented as "can only be called from plugin scripts" [Docs `StudioTestService.yaml`], so Option A is used only if spike T7-S1 passes.
- **Option B (plugin, default).** `multi_run.luau` writes `ServerStorage.AgentTestRequest` (`mode = "multi"`) in the Edit DataModel. The local plugin `AlgebraicTestHarness` picks it up, destroys the request, launches the test and writes the result to `ServerStorage.AgentTestResult` (Edit).

Both paths pass the request **as a JSON string** (a string is always Roblox-serializable) and receive the server's `EndTest` value **as a JSON string**.

`tools/studio/multi_run.luau` (Edit; 1,342 bytes):

```lua
--!strict
-- tools/studio/multi_run.luau | Edit | ARGS {"n":2,"sel":"multi.*","net":"M0Gate","timeoutSec":240,"via":"plugin"}
local HS = game:GetService("HttpService")
local SS = game:GetService("ServerStorage")
local STS = game:GetService("StudioTestService")
local A = HS:JSONDecode([==[__ARGS__]==])
if not STS.EditModeActive then
	return HS:JSONEncode({ ok = false, err = "test_running" })
end
local function put(name: string, s: string)
	local v = SS:FindFirstChild(name)
	if not (v and v:IsA("StringValue")) then
		local n = Instance.new("StringValue")
		n.Name = name
		n.Parent = SS
		v = n
	end
	(v :: StringValue).Value = s
end
local id = HS:GenerateGUID(false)
local req = HS:JSONEncode({
	alg = 1,
	v = 1,
	id = id,
	mode = "multi",
	sel = A.sel,
	n = A.n,
	net = A.net or "Off",
	timeoutSec = A.timeoutSec or 240,
	args = A.args or {},
})
put("AgentTestResult", HS:JSONEncode({ v = 1, id = id, state = "queued" }))
if A.via == "direct" then
	task.spawn(function()
		local ok, r = pcall(function()
			return STS:ExecuteMultiplayerTestAsync(A.n, req)
		end)
		local fail = HS:JSONEncode({ v = 1, id = id, state = "error", err = tostring(r) })
		put("AgentTestResult", if ok and type(r) == "string" then r else fail)
	end)
else
	put("AgentTestRequest", req)
end
return HS:JSONEncode({ ok = true, id = id, via = A.via or "plugin" })
```

`tools/studio/net_profile.luau` (Edit; 1,201 bytes). `settings()` requires the Plugin capability and `GlobalSettings` inherits `ServiceProvider` [Docs `GlobalSettings.yaml`, `GenericSettings.yaml`]; the six properties are `PluginSecurity` [Docs `NetworkSettings.yaml`]. Applied values affect an active playtest immediately [Docs `studio/network-simulator.md`].

```lua
--!strict
-- tools/studio/net_profile.luau | Edit | ARGS {"profile":"M0Gate"} (profile "read" = read back only)
local HS = game:GetService("HttpService")
local A = HS:JSONDecode([==[__ARGS__]==])
local KEYS = {
	"InboundNetworkMinDelayMs",
	"InboundNetworkJitterMs",
	"InboundNetworkLossPercent",
	"OutboundNetworkMinDelayMs",
	"OutboundNetworkJitterMs",
	"OutboundNetworkLossPercent",
}
local NS = settings():GetService("NetworkSettings") :: any
if A.profile ~= "read" then
	local dt = game:GetService("ServerStorage"):FindFirstChild("DevTools")
	local lib = dt and dt:FindFirstChild("Tests") and (dt :: any).Tests:FindFirstChild("Lib")
	local src = lib and lib:FindFirstChild("NetProfiles")
	if not (src and src:IsA("ModuleScript")) then
		return HS:JSONEncode({ ok = false, err = "NetProfiles missing" })
	end
	local m = src:Clone()
	local ok, P = pcall(require, m)
	m:Destroy()
	local p = ok and (P :: any)[A.profile]
	if not p then
		return HS:JSONEncode({ ok = false, err = "unknown profile " .. tostring(A.profile) })
	end
	for _, k in KEYS do
		NS[k] = p[k]
	end
end
local cur = {}
for _, k in KEYS do
	cur[k] = NS[k]
end
return HS:JSONEncode({ ok = true, profile = A.profile, values = cur })
```

**Plugin** (`tools/plugin/AlgebraicTestHarness/Main.luau`, built into `build/AlgebraicTestHarness.rbxm` by `lune run tools/plugin/build.luau [--install]`; `--install` writes the file into `roblox.studioPluginPath()` [Lune `@lune/roblox`], then the human restarts Studio once; verify in spike T7-S2; `build.luau` writes both files only through `tools/lib/SafeWrite.luau`, §7.5):

```lua
--!strict
-- tools/plugin/AlgebraicTestHarness/Main.luau — local Studio plugin. Edit DM: launches multi-client tests requested
-- through ServerStorage.AgentTestRequest. Running client DMs: misprediction attribution. Spec §7.3.8.
local HttpService = game:GetService("HttpService")
local RunService = game:GetService("RunService")
local ServerStorage = game:GetService("ServerStorage")
local StudioTestService = game:GetService("StudioTestService")

if RunService:IsRunning() then
	if RunService:IsClient() then -- verify in spike T7-S11 that plugins load in client DMs
		local lp = game:GetService("Players").LocalPlayer
		local total, counts = 0, {} :: { [string]: number }
		RunService.Misprediction:Connect(function(_t: number, instances: { any }, _stats: any)
			total += 1
			for _, e in instances do
				for name in e.Attributes or {} do
					counts[name] = (counts[name] or 0) + 1
				end
				for name in e.Properties or {} do
					counts["p." .. name] = (counts["p." .. name] or 0) + 1
				end
			end
			if lp then -- client-local attributes, never replicated; read by the client driver
				lp:SetAttribute("AlgMpN", total)
				lp:SetAttribute("AlgMpTop", (require(script:FindFirstChild("Top") :: any) :: any)(counts)) -- ≤ 48 chars
			end
		end)
	end
	return
end
if not RunService:IsEdit() then
	return
end
local busy = false
local function put(name: string, value: string)
	local v = ServerStorage:FindFirstChild(name)
	if not (v and v:IsA("StringValue")) then
		local n = Instance.new("StringValue")
		n.Name = name
		n.Parent = ServerStorage
		v = n
	end
	(v :: StringValue).Value = value
end
local function launch(sv: StringValue)
	local ok, req = pcall(function()
		return HttpService:JSONDecode(sv.Value)
	end)
	if busy or not ok or type(req) ~= "table" or req.mode ~= "multi" then
		return
	end
	if not StudioTestService.EditModeActive then
		put("AgentTestResult", HttpService:JSONEncode({ v = 1, id = req.id, state = "error", err = "test_running" }))
		return
	end
	busy = true
	local raw = sv.Value
	sv:Destroy() -- the Play copy of the Edit DataModel must not contain the request
	put("AgentTestMultiArgs", raw) -- copied into the session; read only if GetTestArgs() is nil (spike T7-S14)
	put("AgentTestResult", HttpService:JSONEncode({ v = 1, id = req.id, state = "running" }))
	print(`[ALG][Harness] multi start id={req.id} n={req.n} net={req.net}`)
	task.spawn(function()
		local okRun, ret = pcall(function()
			return StudioTestService:ExecuteMultiplayerTestAsync(req.n, raw)
		end)
		local args = ServerStorage:FindFirstChild("AgentTestMultiArgs")
		if args then
			args:Destroy()
		end
		local out = if okRun and type(ret) == "string"
			then ret
			else HttpService:JSONEncode({ v = 1, id = req.id, state = "error", err = tostring(ret) })
		put("AgentTestResult", out)
		print(`[ALG][Harness] multi end id={req.id} ok={okRun}`)
		busy = false
	end)
end
local function hook(c: Instance)
	if c.Name == "AgentTestRequest" and c:IsA("StringValue") then
		local sv = c :: StringValue
		sv:GetPropertyChangedSignal("Value"):Connect(function()
			launch(sv)
		end)
		launch(sv)
	end
end
ServerStorage.ChildAdded:Connect(hook)
local existing = ServerStorage:FindFirstChild("AgentTestRequest")
if existing then
	hook(existing)
end
```

(The `Top` helper is a ModuleScript child of the plugin Script that formats the three largest counts as `St:41,HP:12,p.CFrame:9`; `build.luau` adds it from `tools/plugin/AlgebraicTestHarness/Top.luau`.)

**Server side** (`Lib/Multi.luau`, called by `runner().runMulti(req)`):
1. Start a watchdog: after `req.timeoutSec + 30` s without completion, `EndTest` with `state = "error"`, `err = "watchdog"`.
2. Wait ≤ 90 s for `req.n` players with `DataLoaded == true` and a character root. Assign roles `"A"`, `"B"`, … in join order.
3. Create `ReplicatedStorage.TestRun` (Folder) with a StringValue `Spec` and a clone of `ServerStorage.DevTools.Tests.Client` (the driver folder). `Spec` JSON: `{"id","run":"<first 8 chars of id>","roles":{"<userId>":"A",…},"seq":0,"scenario":"","steps":{},"params":{}}`.
4. Run the selection like the solo runner. `ctx.clientStep(role, step, params)` bumps `seq`, sets `scenario`, `steps[role] = step` and `params`, and rewrites `Spec`. Clients report through `Dev.Command` `testReport` (§7.3.9). `ctx.report(role, key, timeoutF)` waits for a value.
5. Collect each client's final metrics into `report.clients[role]`, destroy `TestRun`, print the SUMMARY, then `StudioTestService:EndTest(HttpService:JSONEncode(report))` (server DataModel only [Docs]).
6. Join/leave tests use `StudioTestService:AddPlayers(n)` (server) and the client step `leave` (`CanLeaveTest()` then `LeaveTest()` [Docs]). Whether a client added after another client's `LeaveTest` reuses the departed test UserId is spike T7-S16; until it records reuse, `data.rejoin_persists` checks the stored profile through `DataService._readStored` instead of a true rejoin (Part 2 §2.8.7, §2.14 item 11).

**Agent procedure (multi):**
1. No playtest running (`get_studio_state` shows Edit only).
2. `net_profile.luau` `{"profile":"<name>"}`; check the read-back values.
3. `multi_run.luau` `{"n":<2..8>,"sel":"<selector>","net":"<name>","timeoutSec":<s>,"via":"plugin"}` → `id`.
4. Within 30 s `test_result.luau` (Edit) must show `state = "running"` (plugin) or the Studio state must show a running test. If it stays `queued`: blocker `human-step` "install/enable AlgebraicTestHarness and restart Studio", or retry with `"via":"direct"` if T7-S1 passed.
5. Poll `test_result.luau` (Edit) every ≥ 5 s until `done`/`error`, at most `timeoutSec + 120` s.
6. Read all pages, then call `test_result.luau` with `"consume":true` (removes both StringValues from Edit; `Verify` flags leftovers, §7.9 item 6).
7. `net_profile.luau` `{"profile":"Off"}`.
8. Evidence: `multi(<n>,<profile>): <sel> SUMMARY pass=<p> fail=0 id=<8>` plus the client metrics.

#### 7.3.9 Client test driver and the `Main.client.luau` hook

Part 2's `Main.client.luau` gets this Studio-only block after `client ready` (`mods` = the started controllers table):

```lua
-- Studio-only test hook (Spec §7.3.9). Never active outside Studio.
if RunService:IsStudio() then
	local function attach(tr: Instance)
		if tr.Name ~= "TestRun" then
			return
		end
		local client = tr:WaitForChild("Client", 10)
		local driver = client and client:FindFirstChild("Driver")
		if driver and driver:IsA("ModuleScript") then
			task.spawn(function()
				local ok, err = pcall(function()
					(require(driver) :: any).start(tr, mods)
				end)
				if not ok then
					Log.error("Test", "client driver failed", { err = tostring(err) })
				end
			end)
		end
	end
	ReplicatedStorage.ChildAdded:Connect(attach) -- module-lifetime connection
	local tr = ReplicatedStorage:FindFirstChild("TestRun")
	if tr then
		attach(tr)
	end
end
```

`Client/Driver.luau` decodes `Spec`, finds its role by `tostring(Players.LocalPlayer.UserId)`, and on every `seq` change runs `Steps[steps[role]](dctx, params)` if it has a step, then reports `done = seq`. It counts `RunService.Rollback` events from start (the event has `Basic` capability and is usable from game scripts [Docs `RunService.yaml`]).

```lua
--!strict
-- src/DevTools/Tests/Client/Steps.luau (types). Spec §7.3.9.
export type DCtx = {
	role: string, run: string, seq: number,
	frame: () -> number,                                         -- Frames.fromTime(time()) on this client
	waitFrames: (n: number) -> (),
	fire: (action: string, state: boolean | Vector3) -> (),      -- InputController.fire (Script bindings, Part 3 §3.10.3)
	press: (action: string, holdF: number?) -> (),               -- fire true, wait holdF (default 2), fire false
	vi: any?,                                                    -- UserInputService:CreateVirtualInput() or nil
	report: (key: string, value: number | string | boolean) -> (), -- Dev.Command testReport
	rollbacks: () -> number,
	attr: (name: string) -> any,                                 -- local character HRP attribute
	invoke: (name: string, payload: any) -> any,                 -- Net.client.invoke
	on: (name: string, fn: (any) -> ()) -> () -> (),             -- Net.client.on
}
-- Steps (name -> function(d: DCtx, params)): pressM1, rhythmBlock, reactiveBlock, dashSeries, brawl, leave,
-- countFeed, invokeWishes, watchNotify, reportRealm, reportAppearance, perfProbe, layoutCheck.
```

New Dev command (added to Part 2's `DevCmd` enum): `testReport(run: string, key: string, num: number?, str: string?, seq: number?)`. The handler stores `reports[player][key]` only when `run` matches the active run; otherwise it drops the call silently. Keys ≤ 24 chars and strings ≤ 64 chars (Part 2's `DevCommandReq` limits).

#### 7.3.10 Seams and hooks this part needs from other parts

| Seam / hook | Owner | Signature / behaviour (Studio only unless noted) |
|---|---|---|
| `CombatService._tap` | Part 3 | `(fn: (TapRecord) -> ()) -> () -> ()`; called from the flush for every SimEvent and every contact (§7.3.3) |
| `BotService._setBrain` | Part 3 / 7 | `(model: Model, brain: Brain?) -> ()`; brain replaces the bot's intent source inside the sim |
| `CombatService._getMeter` | Part 3 | `(target: Player \| Model) -> number`; the server-memory meter of a player, bot or dummy (§3.13.7); used by `combat.perfect_block`, `combat.assist_credit` |
| `Telemetry._setPerfSampled` | Part 5 | `(player: Player, on: boolean) -> ()`; overrides the join-time `PerfSampled` attribute (§5.8.3); used by `tel.perf_sample_sent` |
| `Net.server._dispatch` | Part 2 | `(player: Player, name: string, payload: any) -> any`; full server pipeline as if received |
| `PlayerService._leaveTest` | Part 2 | `(player: Player) -> boolean` (false outside Studio); runs the leave path (`CombatService.onPlayerRemoving`, `DataService.releaseAsync`, trove cleanup) for a player who stays in game, then the join path again (§2.3.5); used by `player.leave_releases` in a solo run |
| `DataService._readStored` | Part 2 | `(userId: number) -> Types.PlayerData?`; yields; the stored profile (nil outside Studio; §2.8.5); used by `data.rejoin_persists` (§2.8.7) |
| `PlayerService._devAccessCheck` | Part 2 (hosted for Part 8) | `(player: Player, opts: { forceDevUniverse: boolean?, member: boolean?, dryRun: boolean? }?) -> boolean`; the dev-universe members-only rule with forced answers; `dryRun = true` logs `dry_run=true` instead of kicking; `opts` ignored outside Studio (§2.3.5, Part 8 §8.2.6); used by `ip.dev_universe_members_only` |
| `ConfigReplicator._devContextOverride` | Part 2 (hosted for Part 8) | `boolean?`; TestHarness only; overrides the pack guard's dev-context test (§2.6.4 step 9, Part 8 §8.2.6); used by `ip.runtime_guard_public_sim` |
| `Main.client.luau` test hook | Part 2 | §7.3.9 block |
| DevCmd `testReport` | Part 2 enum, Part 7 handler | §7.3.9 |
| `Moves.frameRows` | Part 3 (`Core/Moves.luau`) | Pure; §7.5.3 (live code: Part 6 Training panel) |
| `VfxController.stats` | Part 6 | `() -> {tier: string, emitters: number, particlesEst: number, beamsTrails: number, telegraphs: number, degradeLevel: number}` (also live code) |
| `Features.LoadTestBots`, `LoadTestMapId`, `LoadTestGameIds` | Part 2 config, Part 3 BotService | §7.6.7 (live code) |
| `Verify` leftovers | Part 1 §1.6.7 item 5 | Also flag `ServerStorage.AgentTestRequest`, `ServerStorage.AgentTestResult`, `ServerStorage.AgentTestMultiArgs`, `Workspace.TestPad`, `ReplicatedStorage.TestRun` in Edit |

---

### 7.4 Test catalog

#### 7.4.1 Conventions

- **Level:** `lune`, `studio`, `mcp`, `multi`, `perf`, `human` (§7.1.1). `net` marks a network-condition test.
- **Owner:** the part whose task implements the test (P2–P7). Names from other parts are binding.
- **M:** milestone tag (`M0` tests also run at M1-G).
- A row like `kit.<id>.smoke ×6` stands for one test per implemented style id.
- Part 8's tests (`ip.*`, `skin.resolve_pack_matrix`, `skin.look_active_only` and the Studio scenarios `ip.runtime_guard_public_sim`, `ip.canon_locked_runtime`, `ip.pack_switch_live`, `ip.dev_universe_members_only`, `ip.no_missing_keys_tour`) are catalogued with level and milestone in Part 8 §8.4.9 and not repeated here; `ip.gate_dry_run` is an agent procedure, not a test.

#### 7.4.2 Infrastructure, standards and harness

| ID | Level | Asserts | Ref | Owner | M |
|---|---|---|---|---|---|
| `shim.datatypes_typeof` | lune | `typeof` of loader-env `Vector3`, `CFrame`, `Color3` values is the Roblox class name | §7.2.4 | P7 | M0 |
| `shim.loader_relative_and_self` | lune | `./`, `../`, `@self/`, `@shared/` resolve like Studio for plain files and `init.luau` | §7.2.4, §1.2.7 | P7 | M0 |
| `shim.loader_same_instance` | lune | Two `L.require` calls for one path return the same table | §7.2.4 | P7 | M0 |
| `shim.loader_rejects_impure` | lune | A fixture module indexing `game` fails with its path in the error | §7.2.4 | P7 | M0 |
| `shim.cframe_lookat_reference` | lune | Env `CFrame.lookAt` LookVector equals the normalized direction for 3 pitched targets (± 1e-4) | §7.2.4 | P7 | M0 |
| `runner.cli_contract` | lune | Exit codes 0/1/2, `--list`, `--fast`, `--filter`, duplicate id → load failure | §7.2.2 | P7 | M0 |
| `standards.strict_headers`, `standards.no_deprecated` | lune | `--!strict` first line; no deprecated APIs | §2.10 | P2 | M0 |
| `standards.sim_banned_tokens` | lune | The Part 3 §3.16.1 L checks (decision D-12): no banned token (§3.1.5) in `Core/**` and `Behaviors/**` (whole files) or between `-- @sim-begin` / `-- @sim-end` marker lines in `Sim/**` and `Combat/**` (outside-sim code such as `Adapter.flush`, `Hurtbox.prepare`, `InputMap.attach` stays outside the markers; an unmatched or nested marker is a finding); inside the markers `Animation.AnimationId`, `AnimationTrack.Animation` and `AnimationTrack.Length` are also banned (not Sim-Access, §3.1.5; `Adapter.track` caches clip id, `clipF` and `looped` outside the sim); allow-list: `Simulation.luau`'s single load-time `IsServer()` and `Adapter.spawn`'s `:Clone(`; every `BindToSimulation(`/`BindToAnimation(` passes `Enum.StepFrequency.Hz60`; `BindToSimulation(` appears exactly once across `src/Shared`, `src/ServerScriptService`, `src/StarterPlayerScripts` (`src/DevTools/Tests/Spikes/**` and `src/DevTools/Tests/Spike_*.luau` are exempt) | §3.1.5, §3.16.1 | P3 | M0 |
| `core.require_graph_acyclic` | lune | `require` graph of `Core`/`Sim`/`Combat`/`Behaviors`/`Styles`/`Types.luau` has no cycle and no edge against the §3.4.2 order | §3.4.2 | P3 | M0 |
| `standards.devtools_isolation` | lune | `DevTools`, `TestRun`, `CreateVirtualInput` appear only in allowed files (`CreateVirtualInput` also allowed in the spike probes, D-12) | §7.5.5 | P7 | M0 |
| `standards.test_loader_only` | lune | `tests/**`, `tools/**` never natively require `@shared/…` | §7.5.5 | P7 | M0 |
| `harness.selftest` | studio | Runner detects PASS, hard FAIL, soft FAIL, timeout (FAIL `timeout after …`); teardown runs after failure; report JSON round-trips | §7.3.4 | P7 | M0 |
| `harness.studio_only_guard` | studio | With `TestHarness._isStudio = false`, `init/start` install nothing and `run` returns 0/0 | §7.3.5 | P7 | M0 |
| `harness.request_roundtrip` | studio (agent-run) | Agent runs `test_run` → `test_result` on `harness.selftest`: id echoed, states `queued → running → done`, paging by 40; evidence is the snippet JSON | §7.3.6 | P7 | M0 |
| `harness.client_driver` | studio | Solo `TestRun`: driver reports `ready`; step `pressM1` makes the server tap show an M1 state entry for the local rid | §7.3.9 | P7 | M0 |
| `netcond.profile_applied` | studio (Edit snippet, agent-run) | `net_profile.luau` read-back equals `NetProfiles.M0Gate`; evidence is the snippet JSON | §7.1.5 | P7 | M0 |
| `netcond.rtt_m0gate` | studio, net | Server-sampled `GetNetworkPing` mean 140–160 ms, P5 ≥ 115, P95 ≤ 185 (td) under `M0Gate` | GDD §18.2, §7.1.5 | P7 | M0 |
| `boot.*`, `player.*`, `tree.runtime_objects`, `config.*`, `net.rate_limit_drops`, `net.invalid_payload_rejected`, `net.remotes_match_catalog`, `data.*` (except `data.rejoin_persists`, below), `attrs.player_public_set`, `security.*` | studio | Part 2 definitions | §2.12 item 21 | P2 | M0 |
| `config.not_overridable` | studio | `Dev.Command setConfig` for `Ip`, `Data` and `NetModel` each returns `ok = false` and leaves `Runtime.version()` unchanged; `setConfig Kits {"MomentCooldownFrames":300}` is accepted and bumps the version | §2.6.4, §2.6.6 | P2 | M0 |
| `player.leave_releases` | studio | Solo run through the Studio seam `PlayerService._leaveTest(player)` (§7.3.10): `[ALG][Data] released user=<id>` within 2 s, then `DataLoaded == true` again within 30 s | §2.3.5 | P2 | M0 |
| `data.rejoin_persists` | multi (`multi(2, Baseline30, data.rejoin_persists)` only) | Client B is granted 500 Treasure (`Dev.Command grant`) and calls `LeaveTest`; after `[ALG][Data] released user=<idB>` the server asserts `DataService._readStored(idB).currencies.Treasure` includes the grant. A true rejoin of the same user is asserted only after spike T7-S16 records that `AddPlayers` reuses the departed UserId | §2.8.7, §2.14 item 11 | P2 | M0 |
| `log.*`, `types.*`, `schema.*`, `playerdata.*`, `net.validators_reject_fuzz`, `skin.*`, `assets.*` | lune | Part 2 definitions | §2.12 item 20 | P2 | M0 |

#### 7.4.3 Combat chassis (Part 3 names unless marked P7)

| ID | Level | Asserts | Ref | Owner | M |
|---|---|---|---|---|---|
| `frames.to_seconds`, `frames.phase_boundaries`, `frames.window_1indexed` | lune | f/60; phase edges; 1-indexed windows | §0.4 | P3 | M0 |
| `rng.deterministic`, `rng.range_uniform_chi2`, `rng.hash_stable` | lune | xorshift32 determinism and uniformity | §3.4.3 | P3 | M0 |
| `fsm.buffer_6f`, `fsm.buffer_latest_wins`, `fsm.buffer_cleared_on_stun`, `fsm.priority_awaken_over_dash`, `fsm.no_input_in_stun`, `fsm.pure_no_mutation`, `fsm.q_rule` | lune | State machine rules | §3.2 | P3 | M0 |
| `fsm.golden_m1_string`, `fsm.golden_uppercut_string`, `fsm.golden_hop_cancel` | lune | Per-frame St/StF/Sub/HP equal golden traces | §3.4.3 | P3 | M0 |
| `m1.frame_table_gdd` | lune | M1-1..4 startup/active/recovery/damage = 6/3/12/3.5, 6/3/12/3.5, 7/3/13/3.5, 10/4/22/5.0 | GDD §6.3 | P3 | M0 |
| `m1.chain_window_36f`, `m1.chain_reset_37f` | lune | Next M1 ≤ 36f after recovery continues; 37f resets to M1-1 | GDD §6.3 | P3 | M0 |
| `m1.ender_endlag_60f` | lune | M1 locked 60f after the Ender | GDD §6.3 | P3 | M0 |
| `m1.variant_select` | lune | Hold Jump → Uppercut; airborne M1 → Downslam; hold A/D → Side Ender | GDD §6.3 | P3 | M0 |
| `m1.air_string_fits_hang`, `m1.downslam_once_per_combo`, `m1.dashm1_continues_at_2` | lune | 3 air M1s fit 48f hang; one Downslam per combo; Dash-Ender | GDD §6.3, §6.5 | P3 | M0 |
| `combat.m1_chain_frames` | studio | Hits at e = 6, 6, 7, 10; total 15.5; victim Ragdoll 60f; `M1LockF` = Ender exit + 60 | GDD §6.3 | P3 | M0 |
| `combat.chain_window_reset` | studio | 37f gap → `Sub = 1` | GDD §6.3 | P3 | M0 |
| `combat.uppercut_air_string` | studio | 8 hits, 28.0 damage, DOWNSLAM flag set | GDD §6.3 | P3 | M0 |
| `combat.side_ender_angle` | studio | Holding D on the 4th hit: victim's horizontal travel heading is 45° ± 5° to the attacker's right of its facing; holding A mirrors it | GDD §6.3 | P7 | M0 |
| `combat.wall_splat` | studio | Ender toward a wall 10 studs away → `WallSplat` within 18f, stuck 30f; second Ender in the same combo → no splat | GDD §6.3 | P3 | M0 |
| `defense.front_180`, `defense.behind_unblocked` | lune | Block covers the front 180°; rear hits ignore block | GDD §6.4 | P3 | M0 |
| `defense.pb_window_9f`, `defense.antimash_20f` | lune | Press ≤ 9f before the hit → Perfect Block, 10f → normal block; empty tap → block locked 20f | GDD §6.4 | P3 | M0 |
| `defense.guard_regen_lazy`, `defense.guardbreak_84f`, `defense.blockstun_from_adv` | lune | 25/s after 1.0 s; break → 84f stun, +10 % damage; blockstun matches GDD advantage | GDD §6.4 | P3 | M0 |
| `combat.block_front_back` | studio | Front M1 0 damage, `Grd` 88; rear M1 3.5 | GDD §6.4 | P3 | M0 |
| `combat.perfect_block` | studio | Dummy mode `perfect_block` vs player M1: attacker `Stagger` 36f; defender meter +6, read with the Studio seam `CombatService._getMeter(dummy)` (dummies have no Player `Meter` attribute) | GDD §6.4, §6.9 | P3 | M0 |
| `combat.antimash` | studio | `BlkLockF` = release + 20; Block during the lock does nothing | GDD §6.4 | P3 | M0 |
| `combat.guard_break` | studio | Nine isolated M1-1 presses, each ≥ 37f after the previous M1's recovery ends (chain reset), into a holding-block dummy → `Grd` = 4 before the 9th (100 − 8 × 12); `GuardBroken` on the 9th; `StDur = 84`; Breakaway refused at e = 20, accepted at e = 31 (a buffered chain would break on the 8th hit, the second Ender, so the presses are spaced on purpose) | GDD §6.4, §6.8 | P3 | M0 |
| `movement.classify_dirs`, `movement.dash_distances` | lune | Direction classification; 30/20/16/14 studs over 18/15/14/12f | GDD §6.5 | P3 | M0 |
| `movement.side_iframes_1_7`, `movement.back_iframes_1_5` | lune | I-frames exactly frames 1–7 (side), 1–5 (back); none front/air | GDD §6.5 | P3 | M0 |
| `movement.cooldowns`, `movement.airdash_once` | lune | Independent cooldowns 2.0/1.6/2.0 s; air dash once per airtime | GDD §6.5 | P3 | M0 |
| `combat.dash_distances` | studio | Displacement within ±10 % of the table; cooldown attributes = start + 120/96/120 | GDD §6.5 | P3 | M0 |
| `combat.side_dash_iframes` | studio | Dummy hits during side-dash frames 1–7 whiff; frame 8+ hits | GDD §6.5 | P3 | M0 |
| `combat.hop_cancel` | studio | Side dash accepted in M1-2/M1-3 recovery after a hit; refused after a whiff | GDD §6.5 | P3 | M0 |
| `stun.decay_after_150f`, `stun.shakeoff_210f` | lune | Hitstun ×0.6 after 2.5 s continuous stun; Shake-Off at 3.5 s (18f i-frames, 6 studs) | GDD §6.6 | P3 | M0 |
| `stun.otg_once` | lune | Max 1 OTG hit per combo | GDD §6.6 | P3 | M0 |
| `stun.combo_scale_tiers`, `stun.scale_floor` | lune | ×1.0 hits 1–8, ×0.85 hits 9–14, ×0.70 hits 15+, floor ×0.5 | GDD §6.6 | P3 | M0 |
| `stun.stretch_ragdoll_minus10` | lune | `stretch` passive −10f ragdoll (reads the kit's `ragdollFramesAdd`; IP-neutral name per TASKS §16.1 C12) | GDD §7.3.2 | P3 | M0 |
| `combat.stun_decay` | studio | Hitstun applied at continuous stun ≥ 150f is 16 not 27; Shake-Off at 210 | GDD §6.6 | P3 | M0 |
| `combat.otg_once` | studio | Second OTG hit → event `otg_denied` | GDD §6.6 | P3 | M0 |
| `damage.design_target_no_meter` | lune | Clean no-meter, no-wall string totals 28–36 | GDD §6.6 | P3 | M0 |
| `damage.m1_blocked_zero`, `damage.move_chip_20`, `damage.guardbroken_plus10`, `damage.iframes_whiff`, `damage.grab_vs_block`, `damage.grab_vs_airborne`, `damage.pb_parryable_only` | lune | Damage resolution rules | GDD §6.4 | P3 | M0 |
| `damage.pileon_25`, `breakaway.pileon_2x` | lune | 3+ attackers within 4 s → ×0.75 damage; Breakaway refill ×2 for 4 s | GDD §6.6 | P3 | M0 |
| `combat.pileon` | studio | Three bots within 4 s → `PileF` set; next 3.5 hit deals 2.6 | GDD §6.6 | P3 | M0 |
| `breakaway.lockout_12f`, `breakaway.refill_25s`, `breakaway.damage_refund`, `breakaway.guardbreak_30f` | lune | 12f lockout; 25 s per pip; −1 s per 2 damage; none in first 30f of Guard Break | GDD §6.8, §6.4 | P3 | M0 |
| `combat.breakaway` | studio | Dash at hit + 5f refused, + 16f accepted; `BkP` 2 → 1; attacker within 8 studs in `Stagger` | GDD §6.8 | P3 | M0 |
| `combat.breakaway_refill` | studio | `BkP` back to 2 after 25 s ± 30f; 10 damage at `BkP = 1` → `BkProg` +300 | GDD §6.8 | P3 | M0 |
| `meter.gains_table`, `meter.realm_half` | lune | +1.0/HP dealt, +0.5/HP taken, +6 PB, +10 style moment, +15 KO, +7 assist; Realm damage ×0.5 | GDD §6.9 | P3 | M0 |
| `meter.awaken_effects`, `meter.awaken_60s_plus5_cap90` | lune | 72f invulnerable activation, +10 HP, +1 pip, move swap; 60 s, +5 s per KO, cap 90 s | GDD §6.9 | P3 | M0 |
| `combat.awakening` | studio | `Dev.Command setMeter 100` → `AwkRdy`; precondition `Dev.Command combatPatch {"HP":60,"BkP":0}` (so the clamps at 100 HP / 2 pips cannot hide the effects); G → `Awakening` 72f, invulnerable, HP 70 (60 + 10, against HP sampled 1f before activation), `BkP` 1, `AwkEndF − AwkF0 = 3672` | GDD §6.9 | P3 | M0 |
| `combat.awaken_ko_extend` | studio | 3 KOs → +900 frames, capped at `AwkF0 + 5472` | GDD §6.9 | P3 | M0 |
| `realm.interrupt_window_45f`, `realm.contains`, `realm.max2`, `realm.stubborn_heart_once` | lune | Interruptible only in the first 45f of the 90f cast; 40-stud radius; ≤ 2 per server; one ward per Realm per player | GDD §6.10 | P3 | M1-G |
| `combat.realm_cast_interrupt` | studio | Parryable hit at cast e = 30 cancels; at e = 60 does not | GDD §6.10 | P3 | M1-G |
| `combat.realm_open_members`, `combat.realm_cap_queue` | studio | Bots at 10/35/45 studs → **3** members (`RlmId` set on the caster and the 2 bots inside 40 studs; the bot at 45 keeps `RlmId = 0`); a third cast is queued until one ends | GDD §6.10 | P3 | M1-G |
| `combat.stubborn_heart` | studio | Hold Block 1 s inside → `RlmWardF` set, `BkP` −1; a second attempt in the same Realm does nothing | GDD §6.10 | P3 | M1-G |
| `combat.ko_respawn` | studio | KO → new character after 4 ± 0.1 s; `ProtF − StF = 480` | GDD §6.2 | P3 | M0 |
| `combat.spawn_protection` | studio | Hits during protection whiff; attacking ends protection | GDD §6.2, §15.5 | P3 | M0 |
| `combat.assist_credit` | studio | Bot A (`"b1"`) deals 16, bot B kills → KoEvent `assistBots = {"b1"}`, `assists` empty; `CombatService._getMeter(A)` +7 | GDD §6.11, §6.9 | P3 | M0 |
| `combat.streak_bounty` | studio | Setup `Dev.Command setConfig Combat {"Streak":{"CountBots":true}}`, respawn P (pinned config applies at respawn); P KOs 3 brawl bots → `streak` tier-3 event; a bot KOs P → `ended`, bounty 25; teardown restores `CountBots = false` | GDD §6.11 | P3 | M0 |
| `combat.feed_fired`, `combat.regen` | studio | One feed per KO; 2 HP/s after 8 s | GDD §6.11, §6.2 | P3 | M0 |
| `combat.env_ko_credit` | studio | Needs Part 6's void kill: an `m1_spam` dummy hits the player once; 3 s later the player drops below the map's `KillY` → `killAsync(…, "environment")` credits the dummy (`killerBot`); with 6 s → no killer (`Combat.Streak.EnvKoCreditWindowSec = 5`, GDD §11.1) | GDD §11.1 | P3 | M0 |
| `combat.disabled_move_kill_switch` | studio | `setConfig Features {"DisabledMoves":["hero.1"]}`, respawn → Move1 leaves the player in `Idle`; Move2 still works; teardown restores `DisabledMoves = {}` | §3.4.2 | P3 | M0 |
| `moves.status_codec_roundtrip`, `moves.status_capacity_evicts`, `moves.status_codec_long_uptime`, `moves.validate_bands`, `moves.validate_tell_8f` | lune | Status codec round-trips and capacity eviction; long uptime: four statuses with `untilF` ≈ 6e7 (above 62^5) → kept entries round-trip, the result is ≤ 50 chars, the smallest-`untilF` entry is evicted, no error; MoveDef band and 8f-tell validation | §3.4.4 | P3 | M0 |
| `moves.disabled_move_blocked` | lune | A `StyleRuntime` with `disabled["1"] = true`: `Moves.canUse` false for slot 1, true for slot 2 | §3.4.2 | P3 | M0 |
| `world.travel_streams_destination` | studio | Travel from the battle map (`arena_duel_m0` at M0, `candy_kingdom` once TASKS T211 builds it) to `training`: no `stream_around_failed` log; the client's root never goes below the destination's `KillY + 5`; `GetPredictionStatus(root)` back to `Predicted` within 2 s of the new character | §3.13.4 | P3 | M0 |
| `config.combat_gdd_values` | lune | `Defaults.Combat` equals a literal table of GDD numbers | §3.17 | P3 | M0 |
| `sim.hz60_clock`, `sim.attr_budget` | studio | Δf = 1 per call at Hz60; 49/52 attributes with correct types | §3.1.3, §3.3 | P3 | M0 |
| `input.attach_contexts`, `input.scriptable_aim_replicates` | studio | 15 actions under `Player.Combat`; Script `Aim` reaches the server | §3.10 | P3 | M0 |
| `bot.dummy_modes`, `bot.intent_path` | studio | Dummy behaviours; bot intents equal player outcomes | §3.15 | P3 | M0 |
| `bot.loadtest_off_in_studio` | studio | `setConfig Features {"LoadTestBots":4,"LoadTestGameIds":[<game.GameId>]}`, respawn → no brawl bot appears in `Workspace.Bots` (Studio guard, §7.6.7); with `Features.Bots == false`, `spawnAsync({kind = "brawl"})` returns nil while `kind = "dummy"` still spawns | §3.15.1, §7.6.7 | P3 | M0 |
| `rewind.ring_wraps`, `rewind.frames_for_rtt`, `query.sweep_excludes`, `classic.*` | lune | Rewind indexing; query sweep (fake `workspace` via `Loader.new`) | §3.12 | P3 | M0 |
| `spike.SA-01` … `spike.SA-16` | studio/multi | Part 3 §3.16.4 | §3.16 | P3 | M0 |
| `spike.S-P4-2` | multi; tags `m0`, `spike` | `Spike_SP4_2`: misprediction count per grab hold at 150 ms; run as `multi(2, M0Gate, spike.S-P4-2)` with `allowFail = {"^spike%."}` (recorded, not gated) | Part 4 §4.11 | P4 | M0 |
| `spike.S-P4-3` | multi; tags `m1g`, `spike` | `Spike_SP4_3`: status write and pull cost — rollbacks per second on a victim in a zone, inside `stretch.a4` (B-class zone pull) and on `kit.pull` victims; `multi(2, M0Gate, spike.S-P4-3)`, `allowFail` as above | Part 4 §4.11 | P4 | M1-G |
| `spike.S-P6-2` | multi; tags `m0`, `spike` | `Spike_SP6_2`: client-local massless parts welded to the local predicted character cause no attributable mispredictions (5 min run, misprediction log); `multi(2, Baseline30, spike.S-P6-2)`, `allowFail` as above | Part 6 §6.15 | P6 | M0 |

#### 7.4.4 Style kits and the kit validator

| ID | Level | Asserts | Ref | Owner | M |
|---|---|---|---|---|---|
| `styles.registry_18`, `styles.ids_match_types`, `styles.rarities_match_gdd`, `styles.implemented_set` | lune | 18 defs in `STYLE_IDS` order with GDD rarities; exactly 6 implemented | GDD §7.2, §0.10 | P4 | M0 |
| `styles.validate_all_clean` | lune | Kit validator `Styles.validate(Defaults)` returns `true, {}` | §4.1.4 | P4 | M0 |
| `styles.rule_<ID>_detects` ×31 (S01–S06, K01–K25) | lune | Each rule fires on a one-field mutation of a valid kit | §4.1.4 | P4 | M0 |
| `styles.overrides_apply`, `styles.overrides_invalid_rejected`, `styles.move_ids_unique`, `styles.frozen`, `styles.behavior_strike_nil` | lune | Overrides layer, immutability | §4.1.6 | P4 | M0 |
| `kits.<id>.gdd_numbers` ×6 | lune | Kit numbers equal the GDD §7.3 tables | §4.5.9 | P4 | M0 (hero, stretch) / M1-G |
| `kits.anim_specs_complete` | lune | Rule K24 on every implemented kit: `anim` specs for `m1` (12-entry `alt`), `q`, `1..4`, `a1..a4`, `awaken`, and every pose name in the pose library | §4.6 | P4 | M0 |
| `behaviors.*`, `kit_status.*` | lune | Behavior purity and statuses | §4.2.11, §4.3.7 | P4 | M0 |
| `style.registry_boot` | studio | `[ALG][Style] registry ok … problems=0` within 5 s of play start | §4.1.5 | P4 | M0 |
| `kit.<id>.smoke` ×6 | studio | Every slot used on a dummy; before each slot the dummy's `Debuffs` and the attacker's `Buffs`/`Kit` are cleared with `CombatService._queuePatch` (or a fresh dummy slot is used), so flat adds from earlier slots (decay stacks, rocker's next-hit buff) cannot carry over; HP delta = `maxDamage` ± 0.1 (or `maxDamage` + the active flat adds, clamped to the band cap, when a slot is deliberately chained); cooldown attribute = f + toFrames(cooldown) | §4.8 | P4 | M0 / M1-G |
| `kit.awaken.swap`, `kit.awaken.hurtbox` | studio | Move swap; hurtbox only scales up | GDD §6.2, §6.9 | P4 | M0 |
| `config.kits_override_next_life` | studio | Override applies after respawn only (created and run in TASKS T172, before the M0 gate) | §4.1.6 | P4 | M0 |
| `kit.stretch.shield_reflect`, `kit.rocker.batshift_iframes`, `kit.frost.hover`, `kit.grab_vs_jump`, `kit.grab_breakaway`, `kit.moment_meter` | studio | Part 4 definitions | §4.9 item 21 | P4 | M0 / M1-G |
| `kit.decay.well_heal` | studio | `decay` at 80 HP (`Dev.Command combatPatch {"HP":80}`) first lands one M1 on an `idle` dummy, then casts `decay.3` with the dummy inside the pool (the pool ticks on the dummy keep `CombatF` fresh, so out-of-combat regen cannot add); HP sampled 1f before the cast and at k = 179 → +3 HP exactly (owner ticks at k = 0, 60, 120) | §4.2.6 | P4 | M1-G |
| `kit.decay.realm` | studio | Realm opens with **3** members (caster + the 2 targets inside 40 studs), as `combat.realm_open_members` | §4.9 item 21, §3.9.3 | P4 | M1-G |

#### 7.4.5 Wish system (Part 5 names unless marked P7)

| ID | Level | Asserts | Ref | Owner | M |
|---|---|---|---|---|---|
| `wish.gdd_tier_odds` | lune | `Wish.TierOdds` ppm = 500000, 280000, 140000, 60000, 16000, 3900, 100 | GDD §8.3 | P7 | M1-G |
| `wish.gdd_flair_odds` | lune | `Wish.Flair` ppm shiny 20000, golden 4000, crystal 1000, rainbow 200, nightosphere 40, glitched 10; standard remainder 974,750 | GDD §8.7 | P7 | M1-G |
| `wish.tier_tables_sum_1e6`, `wish.flair_tables` | lune | Every tier table sums to 1,000,000; normal and lucky flair tables each sum to 1,000,000 | GDD §8.8 | P5 | M1-G |
| `wish.view_sums_100` | lune | Odds view: tier rows sum to 100 %; per-style rows sum to 100 %; flair rows sum to 100 % (separately) | GDD §8.8 | P5 | M1-G |
| `wish.per_style_division` | lune | Per-style odds shown to 3 decimals = 16.667 / 9.333 / 4.667 / 2.000 / 0.800 / 0.390 / 0.010 % | GDD §8.3 | P7 | M1-G |
| `wish.per_style_redivide` | lune | A fixture pool with a 4th common style shows 12.500 % per common; totals stay 100 % | GDD §8.3 | P7 | M1-G |
| `wish.tier_tables`, `pity.soft_bonus_values`, `pity.floor_precedence`, `pity.advance_resets` | lune | Soft pity from Wish 60 (+0.5 %/Wish); hard pity 30 / 120 / 400; precedence | GDD §8.4 | P5 | M1-G |
| `wish.roll_vectors` | lune | R1–R16, incl. new-style priority (R4, R5) and Mythic duplicate → Spark (R16) | GDD §8.4 | P5 | M1-G |
| `wish.view_priority_rows`, `wish.cosmic_fixed`, `wish.hash_stable`, `wish.hash_changes`, `wish.pool_matches_styles` | lune | Priority rows; Cosmic 0.01 % in every state | GDD §8.3–§8.4 | P5 | M1-G |
| `wish.cosmic_path_bound_4400` | lune (slow) | Worst-case schedule: the 11th Mythic+ arrives at Wish 4,400, so 10 Sparks are guaranteed by then | GDD §8.4 | P7 | M1-G |
| `wish.lucky_flair_only` | lune | Lucky Jar leaves tier tables identical | GDD §8.7 | P5 | M1-G |
| `wish.mc_spine_tolerance`, `wish.mc_hard_pity_bounds`, `wish.mc_reference_exact` | lune (slow) | 2,000-run Monte Carlo means within ±10 % of §0.11; hard-pity worst cases; bit-exact reference | §0.11 | P5 | M1-G |
| `wish.tier_frequencies_fresh`, `wish.flair_frequencies_fresh` | lune (slow) | 200,000 rolls within ±4σ | GDD §8.3, §8.7 | P5 | M1-G |
| `wish.mul32_vectors`, `wish.fmix32_vectors`, `wish.stream_vectors`, `wish.batch10_vector`, `wish.ftue_styles_match_rarity`, `ledger.*` | lune | Part 5 vectors | §5.2.15 | P5 | M1-G |
| `wish.ftue_first_rare` | studio | 20 fresh profiles: first roll is Rare `monarch` or `flame`, both occur | GDD §15.5 | P5 | M1-G |
| `wish.tutorial_grant_once` | studio | 3 Wishes once; not again on rejoin | GDD §15.5 | P5 | M1-G |
| `wish.pity_epic_hard`, `wish.pity_mythic_hard` | studio | Server honours hard pity | GDD §8.4 | P5 | M1-G |
| `wish.cosmic_craft` | studio | 10 Sparks → `chaos` owned; second craft denied | GDD §8.4 | P5 | M1-G |
| `wish.batch10_server`, `wish.insufficient`, `wish.busy`, `wish.odds_view_matches_server`, `wish.lucky_odds_live`, `wish.escrow_release`, `wish.config_change_rehash`, `wish.anomaly_signal` | studio | Part 5 §5.2.17 (`wish.batch10_server` is the Studio run of the Lune vector `wish.batch10_vector`; IDs are unique across levels, §7.1.4) | §5.2.17 | P5 | M1-G |
| `data.overflow_converts`, `progress.overflow_to_treasure` | studio | Earned Wishes over 1,000 → 10 Treasure each | GDD §17.2 | P2/P5 | M1-G |
| `store.cap_blocked` | studio | Wish purchases that would exceed 1,000 are blocked | GDD §17.2 | P5 | M1-G |

#### 7.4.6 Policy, monetization, shop, progression, telemetry

| ID | Level | Asserts | Ref | Owner | M |
|---|---|---|---|---|---|
| `security.pri_server_rejects` | studio | Restricted player: `Store.Prompt{wishes_10}` → `restricted` | GDD §8.8 | P2 | M1-G |
| `store.pri_hidden_restricted` | studio | Restricted: `wishes_*` and `lucky_jar` hidden, `vip` visible | GDD §8.8 | P5 | M1-G |
| `wish.restricted_paid_locked`, `wish.restricted_no_luck` | studio | Paid Wishes unusable and Lucky Jar ignored when restricted | GDD §8.8 | P5 | M1-G |
| `policy.failsafe_restricted`, `policy.dev_override` | studio | Policy fetch failure → restricted | GDD §8.8 | P5 | M1-G |
| `store.receipt_idempotent` | studio | Same `PurchaseId` twice → granted once | GDD §18.5 | P5 | M1-G |
| `store.mock_grant_wishes`, `store.mock_ignored_live`, `security.mock_ignored_live` | studio | Mock purchases only in Studio | §2.11.4 | P2/P5 | M1-G |
| `store.receipt_live_spike` | studio (human-assisted, H-09); tags `m1g`, `spike` | Real dev-product test purchase reaches `onReceipt` (MON-1); excluded from every gate selector by `-@spike` | §5.6.7 | P5 | M1-G |
| `store.pass_mock_vip` | studio | `setPass vip true` → `Vip` attribute true; Treasure from a KO ×1.1 | Part 5 §5.6.7 | P5 | M1-G |
| `shop.rotation_vectors`, `shop.rotation_id_roundtrip`, `shop.rotation_owned_weighting`, `style.free_rotation_vectors`, `time.week_index_monday` | lune | Deterministic 12-hour rotation from (period, salt); unowned weighting | GDD §8.5 | P5 | M1-G |
| `shop.rotation_stable`, `shop.buy_style`, `shop.buy_insufficient`, `shop.buy_stale`, `shop.unimplemented_purchasable` | studio | Same window → same `rotationId`; +43,200 s → new rotation | GDD §8.5 | P5 | M1-G |
| `progression.*` (lune), `progress.*` (studio) | lune/studio | Part 5 §5.5.12 | GDD §9 | P5 | M1-G |
| `style.equip_*`, `style.totem_swap`, `style.training_trial`, `style.disabled_kill_switch`, `style.free_rotation_equip` | studio | Part 5 §5.1.8 (incl. "In development" styles not equippable) | §0.10 | P5 | M1-G |
| `tel.ftue_monotonic`, `tel.echo_wish`, `tel.echo_economy`, `tel.fields_stringified`, `tel.budget_drop`, `tel.no_analytics_in_studio` | studio | Studio `[ALG][TEL]` echo of the KPI source events (§7.6.8) | GDD §20.4 | P5 | M1-G |
| `tel.perf_sample_sent` | studio | `setConfig Telemetry {"PerfWindowSec":5,"PerfSendEverySec":10}`, seam `Telemetry._setPerfSampled(P, true)` → within 15 s the ring has a `perf_sample` record from P's VfxController sender; unsampled sender → dropped | §5.8.3, Part 6 §6.6.3 | P5 | M1-G |
| `mod.report_logged`, `mod.report_self_rejected`, `mod.ban_dry_run` | studio | Part 5 §5.9 | GDD §18.10 | P5 | M1-G |

#### 7.4.7 Appearance, skin packs, assets and lints

| ID | Level | Asserts | Ref | Owner | M |
|---|---|---|---|---|---|
| `appearance.effective_look` | lune | Pure truth table of `CharacterLook` per style | §0.4 | P5 | M1-G |
| `appearance.per_style_override`, `appearance.ownlook_on_respawn`, `appearance.attrs_replicated`, `appearance.r15_only`, `appearance.trial_override_allowed` | studio | Part 5 §5.7.7 | §0.4 | P5 | M1-G |
| `appearance.hurtbox_invariant` | studio | Hurtbox size and offset identical across Character, Oosona and Avatar lives | GDD §6.2, §0.4 | P5 | M1-G |
| `multi.appearance_per_viewer` | multi | Viewer B with `OthersAppearance = AlwaysOwn` renders A in A's own look while A's server hurtbox is unchanged | §0.4 | P7 | M1-G |
| `skin.fallback_order`, `skin.canon_locked`, `skin.interpolate_ids`, `skin.missing_logged_once` | lune | Part 2 §2.9.2 | §0.4 | P2 | M0 |
| `skin.on_changed` | lune | A `Skin.onChanged` callback fires once when `setPack` switches Codename → Original, not when the pack is unchanged; the returned disconnect stops further calls | §2.9.2 | P2 | M0 |
| `skin.pack_key_coverage` | lune | Every key referenced in `src/` (literal `Skin.text` keys plus generated `style.*`, `move.*`, `map.*`, `flair.*`, `rarity.*`, `ui.error.*`, `product.*`, `pass.*` keys) exists in `Codename` and `Original` with Part 8 §8.2.3 coverage semantics (every key in Original; in Codename a non-identity key may be inherited from Original, identity-namespace keys must be in Codename itself), skipping strip-stub packs (Part 8 §8.4.3: second line starts `-- STRIPPED for public build`, `strings` empty) | §0.4 | P7 | M0 |
| `lint.strings_clean` | lune | `lint_strings.luau` (dev mode) exits 0 on the repo | §7.5.1 | P7 | M0 |
| `lint.strings_detects_fixture` | lune | Fixture tree with a banned term in `Original` and a literal `Text = "Fight!"` → exit 1 with rules L04 and L01 | §7.5.1 | P7 | M0 |
| `lint.manifest_clean` | lune | `check_manifest.luau` exits 0 on the repo | §7.5.2 | P7 | M0 |
| `lint.manifest_release_detects_placeholder` | lune | Fixture manifest with a referenced `placeholder` key + `--release` → exit 1, rule M08 | §0.5 | P7 | M0 |
| `lint.manifest_raw_id_detected` | lune | Fixture source with `rbxassetid://123` outside `Assets.luau` → rule M04 | §0.4 | P7 | M0 |
| `framedata.export_up_to_date` | lune | `export_framedata.luau --check` finds no stale file | §7.5.3 | P7 | M0 |
| `framedata.m1_on_block_gdd` | lune | Computed on-block advantage of M1-1..4 = −2, −2, −3, −10 | GDD §6.3 | P7 | M0 |
| `anim.placeholders_built`, `anim.timing_lock` | studio | Part 4 definitions; a final clip that is neither `KeyframeSequence` nor `CurveAnimation` FAILs with `unsupported clip class <ClassName>` (§4.6.4) | §4.6 | P4 | M0 |
| `look.stealth_ranges` | multi (2 clients; `multi(2, Baseline30, look.stealth_ranges)` only) | An enemy client at 20 studs sees the shimmer only; at 8 studs it sees the look | §4.7.6 | P4 | M1-G |
| `look.modes_matrix` | multi (2 clients; `multi(2, Baseline30, look.modes_matrix)` only) | Part 6 §6.8.6: owner Character/Oosona/Avatar × gear on/off against each viewer `OthersAppearance` | Part 6 §6.8.6 | P6 | M1-G |

#### 7.4.8 Network conditions and multi-client (Part 7 unless marked)

| ID | Level | Asserts | Ref | Owner | M |
|---|---|---|---|---|---|
| `combat.rewind_hit_150ms` | studio, net | Under `M0Gate`, ≥ 9/10 timed M1s hit a strafing bot; ≤ 4/10 with rewind disabled | GDD §18.2 | P3 | M0 |
| `combat.rewind_cap`, `combat.trade_same_frame` | studio | Rewind clamp at 200 ms; same-frame trades | GDD §18.2 | P3 | M0 |
| `render.undo_spark` | studio, net | Mispredicted spark removed within 15f | §3.11.8 | P3 | M0 |
| `render.remote_telegraph_full` | studio, net (solo, under `M0Gate`) | A bot uses `hero.2`; the observer client draws the Arc telegraph for 8 ± 1 render frames before the move's VFX | Part 6 §6.6.1 | P6 | M0 |
| `sim.client_writes_local_only`, `sim.carrier_stitch` | multi | Part 3 §3.11 | §3.11 | P3 | M0 |
| `multi.join_leave_rejoin` | multi | Client B `LeaveTest` → `[ALG][Data] released user=<id>` within 2 s; `AddPlayers(1)` → new player `DataLoaded` and spawned within 30 s | §2.3.5 | P7 | M0 |
| `multi.ko_feed_both` | multi | `killAsync(B)` → both clients receive exactly one `Combat.Feed`; B respawns after 4 ± 0.1 s | GDD §6.11 | P7 | M0 |
| `multi.pb_rhythm` | multi, net | Bot M1-1 every 120f at client A; A presses Block at hit − 4f (own sim frame), holds 12f; 20 attempts; metric `pbRatePct`; pass ≥ 90 % (td) | GDD §18.2 | P7 | M0 |
| `multi.pb_reactive` | multi, net | Same attack; A blocks when it first sees the bot enter the M1 state; metric only (latency penalty) | GDD §18.2 | P7 | M0 |
| `multi.prediction` | multi, net | 2 clients brawl each other near 4 bots for 120 s; each client `predSuccessPct` (rollback proxy, §7.6.4) ≥ 97 % | GDD §18.2 | P7 | M0 |
| `multi.dash_no_rubberband` | multi, net | 10 dashes by client A; at dash end + 10f, client-reported position vs server trace position differ ≤ 1.0 stud | GDD §18.2, §3.16.4 SA-08 | P7 | M0 |
| `multi.wish_under_loss` | multi, net | Under `M0Gate`, 10 sequential `Wish.Make{1}` from client A: all `ok`, ledger +10, Wishes −10 exactly | GDD §18.5 | P7 | M1-G |
| `multi.realm_two_clients` | multi | A (decay) casts the Realm with B 10 studs away: server membership and B's client sees `Realm_<id>` and its own `RlmId` within 30f of opening | GDD §6.10 | P7 | M1-G |
| `multi.wish_announce` | multi | A's forced Legendary pull → B receives the server-wide notify (`ui.notify.wish_legendary`) within 2 s | GDD §8.6 | P7 | M1-G |
| `ui.report_submit` | multi (`multi(2, Baseline30, ui.report_submit)` only) | Part 6 §6.9.12: A reports B from Settings ▸ Safety with reason `teaming` → `ModerationService.reports(1)` holds `{reporter = A, target = B, reason = "teaming"}` and A gets the `ui.notify.report_thanks` toast; with `Features.Reports = false` the Safety list shows no report buttons | Part 6 §6.9.12 | P6 | M1-G |

#### 7.4.9 Performance

| ID | Level | Asserts | Ref | Owner | M |
|---|---|---|---|---|---|
| `perf.budgets_match_gdd` | lune | `Lib/Budgets.luau` (loaded with `L.require("src/DevTools/Tests/Lib/Budgets")`) equals the literal GDD §18.7/§13.5 tables | GDD §18.7, §13.5 | P7 | M0 |
| `perf.static_vfx_caps` | studio | Per VFX template: emitter count ≤ per-move cap for the style's rarity; unique flipbook textures ≤ 40 | GDD §13.5, §18.7 | P7 | M0 |
| `perf.vfx_budget_worst` | studio | 20 bots using moves within 30 studs + 2 Realms + 5 awakenings: client emitters/particles/beams ≤ the active tier budget; `degradeLevel` rises instead; telegraph count never decreases | GDD §13.5, §16.3 | P7 | M1-G |
| `perf.boot_time_to_fight` | studio | Join → spawned character in a neutral state ≤ 20 s | GDD §20.2 | P7 | M0 |
| `perf.leak_after_clear` | perf | 20 bots brawl 120 s, `clearBots`, +10 s: instances ≤ baseline × 1.02, Lua heap ≤ baseline × 1.10, unparented instances ≤ 200 (td) | GDD §18.7 | P7 | M1-G |
| `perf.m0_gate_load` | perf (multi 4) | 4 scripted clients + 20 brawl bots, `M0Gate`, 300 s: ≥ 95 % of seconds at ≥ 55 Hz server heartbeat; every client `predSuccessPct` ≥ 97 %; 0 errors | GDD §18.2, §18.7 | P7 | M0 |
| `perf.m0_clients8` | perf (multi 8) | 8 clients + 12 bots, `M0Gate`, 180 s: same server and client criteria | §0.0 | P7 | M0 |
| `perf.m1g_soak` | perf (multi 8) | 8 clients + 12 bots on `candy_kingdom`, 2 decay Realms, 5 forced awakenings, `Baseline30`, 600 s: heartbeat criterion; heap growth (t = 120 → 600 s) ≤ 10 %; client draw calls ≤ 900, triangles ≤ 600k, emitters ≤ 400, particles ≤ 6,000, beams/trails ≤ 150; 0 errors | GDD §18.7, §13.5 | P7 | M1-G |

#### 7.4.10 MCP input-driven

| ID | Level | MCP steps (hints) | Asserts | Ref | Owner | M |
|---|---|---|---|---|---|---|
| `input.mcp_m1_reaches_server` | mcp | mouse click viewport-center | Tap shows an M1 state entry for the local rid after setup | §3.10 | P3 | M0 |
| `input.default_jump_works` | mcp | key press Space | Trace root Y rises ≥ 2 studs | §3.10 | P3 | M0 |
| `mcp.block_hold_f` | mcp | key down F, wait 0.5, key up F | `Block` state entered; no `BlkLockF` (hold, not a tap) | GDD §15.2, §6.4 | P7 | M0 |
| `mcp.dash_q_side` | mcp | key down D, key press Q, key up D | Side dash `Sub`; `CdDS` set | GDD §15.2, §6.5 | P7 | M0 |
| `mcp.awaken_g` | mcp | (setup sets Meter 100) key press G | `Awakening` state entered | GDD §6.9 | P7 | M0 |
| `mcp.move_keys` | mcp | One, wait 1.5, Two, wait 1.5, Three, wait 1.5, Four | Move states for slots 1–4 entered (hero) | GDD §15.2 | P7 | M0 |
| `mcp.touch_layout_phone` | mcp | snippet `device_sim.luau` `androidLow` (Client), capture, snippet `device_sim.luau` `phone1334` (Client), capture, snippet `device_sim.luau` `stop` | Client `layoutCheck` at each preset: every touch button exists, is visible, inside the screen, no overlaps, ≥ 44 px (td); HUD text ≥ 14 px at `phone1334` | GDD §15.2, §16.3 | P7 | M0 |
| `mcp.ui_wish_click` | mcp | mouse click on Part 6's Wish ×1 button path | `Wish.Make` handled once (ledger +1) | GDD §15.4 | P7 | M1-G |
| `mcp.wheel_totem` | mcp | navigate to a Style Totem, key press (Part 6 interaction key) | Style Wheel open in the client store | GDD §15.4 | P7 | M1-G |
| `mcp.tab_no_playerlist` | mcp | key press Tab | Client `StarterGui:GetCoreGuiEnabled(Enum.CoreGuiType.PlayerList) == false` and `(Enum.CoreGuiType.Backpack) == false`, `GuiService.AutoSelectGuiEnabled == false` (Part 3 §3.10.1 CoreGui calls), reported by the client driver (`dctx.report`) after the key press; once the Style Wheel exists (M1-G, T215) the Style Wheel screen is also open in the client store | GDD §15.2, Part 3 §3.10.1 | P7 | M0 |
| `mcp.gamepad_menu_nav` | mcp | gamepad `DPadDown`, then `ButtonB` (if the MCP snapshot cannot send gamepad buttons, run with `useFallback`: the client driver fires the same `UI` bindings) | `DPadDown` sets `GuiService.SelectedObject` to the menu strip's Style button; `ButtonB` clears it (Part 6 §6.9.5) | GDD §15.2 | P6 | M1-G |
| `look.gear_silhouette` | mcp | (setup: `BuildTemplates {lookSet}`; args `lookSet`, `silhouette = true`) one `screen_capture` per implemented style, ≤ 3 per run | Captures attached to a human review request (GDD §12.3 64 px silhouette; Part 8 §8.5.3 T1 rows recorded `pending`); the run itself must end `fail=0` | §4.7, Part 8 §8.5.3 | P4 | M1-G |

#### 7.4.11 Human

| ID | Level | Content | Ref | M |
|---|---|---|---|---|
| `human.m0_feel` | human | HF-01 … HF-14 (§7.7.4) | GDD §6, §21.2 | M0 |
| `human.m1g_feel` | human | HF-01 … HF-19 | GDD §6–§8, §15 | M1-G |
| `human.device_low_android` | human | HD-01 … HD-09 (§7.7.5) at M0; HD-01 … HD-10 at M1-G (HD-10 menus on touch) | GDD §18.7, §20.2, §15.1 | M0, M1-G |
| `human.sa_visualizer_reading` | human | SA visualizer (Ctrl+Shift+F6) prediction success ≥ 97 % during `perf.m0_gate_load` with the human in client A | GDD §18.2 | M0 |

#### 7.4.12 World, presentation and UI (Part 6 names)

Part 6 §6.12 is the full list; rows group its names by level and milestone (from the TASKS card that adds each one). Part 6's multi and net scenarios are in §7.4.7 (`look.modes_matrix`) and §7.4.8 (`render.remote_telegraph_full`, `ui.report_submit`); its MCP scenario `mcp.gamepad_menu_nav` is in §7.4.10.

| ID | Level | Asserts | Ref | Owner | M |
|---|---|---|---|---|---|
| `movement.volume_*` (4), `theme.contrast`, `theme.reveal_durations`, `theme.cooldown_frac`, `theme.touch_no_overlap`, `perf.defaults_gdd`, `perf.json_safe`, `perf.tiers_ordered`, `perf.audio_map_complete`, `perf.lighting_presets_complete` | lune | Part 6 definitions (`movement_volumes.spec.luau`, `theme.spec.luau`, `perf.spec.luau`) | §6.12 | P6 | M0 |
| `theme.style_palette_variants` | lune | Part 6 definition (`Theme.stylePalette` variants) | §6.9.4 | P6 | M1-G |
| `render.no_attr_signal`, `anim.no_track_cache` | lune (source lints) | Part 6 definitions (source-text checks a Studio scenario cannot make) | §6.6.1, §6.6.2 | P6 | M0 |
| `anims.base64_roundtrip`, `anims.upload_refuses_without_approve` | lune | `tests/core/upload_anims.spec.luau` (loads `tools/base64.luau` through `@tools/`): 3-byte, 4-byte and 1 KB seeded buffers round-trip; a dry pass with a fake key prints the pending count and makes no request | §6.6.9 | P6 | M0 (only when conditional card T143 runs) |
| `world.spring_launch`, `world.void_kill`, `map.arena_spawn_8`, `map.arena_no_escape`, `training.dummy_mode_ui`, `training.framedata_m1`, `training.damage_numbers_toggle`, `training.reset_travel` | studio | Part 6 definitions | §6.12 | P6 | M0 |
| `world.river_slow`, `world.house_break_rebuild`, `world.hatch_fall_and_spring`, `world.rebuild_postponed_when_occupied`, `map.ck_zone_label`, `map.ck_boundary_soft`, `map.ck_totem_prompt`, `training.trial_equip` | studio | Part 6 definitions | §6.12 | P6 | M1-G |
| `render.hitstop_not_reentry`, `render.no_double_move_vfx`, `render.focus_target`, `render.haptic_gate`, `anim.cosmetic_budget`, `vfx.*` (incl. `vfx.attach_unanchored`), `sfx.voice_cap`, `sfx.bark_throttle`, `sfx.bus_volumes`, `cam.shift_lock_default`, `cam.reduced_motion`, `cam.flash_rate`, `ragdoll.*`, `look.style_tell_always`, `look.face_expressions` | studio | Part 6 definitions (`look.face_expressions`: programmatic frame checks plus one 3×3 `screen_capture` for human review) | §6.12 | P6 | M0 |
| `sfx.map_complete`, `cam.lighting_follows_map`, `look.oosona_three_colour`, `look.variant_render` | studio | Part 6 definitions | §6.12 | P6 | M1-G |
| `hud.hp_drain`, `hud.cooldown_wipe`, `hud.meter_ring`, `hud.breakaway_swap`, `hud.touch_binding`, `hud.combo_counter`, `hud.simple_preset`, `hud.edge_indicators` | studio | Part 6 definitions | §6.9.5 | P6 | M0 |
| `hud.wallet_level`, `hud.quests_progress`, `hud.move_lock_glyph`, `ui.*` (Style Wheel, Wish, Shop, Store, Settings, Results, Notify, FTUE; incl. `ui.no_offer_after_loss`; `ui.report_submit` is level `multi`, §7.4.8), `input.block_toggle`, `input.run_toggle` | studio | Part 6 definitions | §6.12 | P6 | M1-G |

---

### 7.5 Lints and tools

All tools are Lune scripts run from the repo root, load `src/` only through `tests/shim/Loader.luau`, print `[ALG][LINT] <path>:<line> <rule> <detail>` (or `[ALG][LINT] <key> <rule> <detail>` for non-file findings) and end with `[ALG][LINT] <tool> SUMMARY errors=<n> warnings=<m>`. Exit codes: `0` clean (warnings allowed), `1` errors, `2` usage error.

File writes: every Lune script under `tools/**` (except the Studio snippets `tools/studio/**` and the plugin source `tools/plugin/AlgebraicTestHarness/**`) writes files only through `tools/lib/SafeWrite.luau` (Part 8 §8.4.5), never with `fs.writeFile`, `fs.copy` or `fs.move`. In this part that covers `tools/export_framedata.luau` (§7.5.3) and `tools/plugin/build.luau` (§7.3.8); Part 8's `lint_strings --strip/--unstrip` and Part 6's `tools/upload_anims.luau` follow the same rule. Part 8's Lune test `ip.no_tool_writes_signed_files` scans for it. `tests/run.luau` (its `--json` report, §7.2.2) is outside `tools/**` and outside that scan.

#### 7.5.1 `tools/lint_strings.luau` (mechanism; rules data from Part 8)

Usage: `lune run tools/lint_strings.luau [--public] [--pack <Codename|Original|Canon>] [--rules <path>]`. Part 8 §8.4.3 adds the publish-gate flags `--tree <json>`, `--strip <P>` / `--unstrip` and `--dry-run` (L11, L17 and L22 report `W-DRYRUN` warnings instead of errors; used only by the agent procedure `ip.gate_dry_run`, §7.7.6).

| Rule | Mode | Check |
|---|---|---|
| L01 literal UI text | always | In `src/StarterPlayerScripts/**` no `Text`, `PlaceholderText` or `Title` assignment (or `Create` prop) whose literal contains a letter; use `Skin.text` (Part 2 §2.10 item 12) |
| L02 key coverage | always | Every referenced key (literal `Skin.text("…")`/`Skin.has` keys plus the generated families of `skin.pack_key_coverage`) exists in `Codename` and `Original` with Part 8's coverage semantics (§8.2.3: Original is the complete base; Codename and Canon hold overrides and inherit non-identity keys from Original, while identity namespaces must be complete in every pack; Part 8 §8.4.3 L02a/L02b), skipping strip-stub packs (Part 8 §8.4.3: a pack whose second line starts `-- STRIPPED for public build` and whose `strings` is empty; L14 checks which packs must be stubs) |
| L03 key format | always | Pack keys match §0.4 key formats; no unknown namespaces |
| L04 denylist | dev: `Original` pack and `src/**` minus `Skin/Packs/Codename.luau` and `Canon.luau`; `--public`: the pack to be published (`--pack`, default `Defaults.SkinPack`) plus all of `src/**` (code, comments, instance names in builders) | No denylisted term from the rules file (case-insensitive, word boundary), per Part 8's per-pack rules |
| L05 Canon lock | always | `Defaults.SkinPack ~= "Canon"` unless `Defaults.LicenseConfirmed == true` |
| L06 public acknowledgement | `--public` | If the published pack is `Codename`, `Defaults.PublicCodenameAcknowledged == true` (§0.9) |

Under `--public` every file in `src/**` ships, including `Skin/Packs/Canon.luau`; without `LicenseConfirmed` that file is in L04 scope, so Part 8 strips Canon (and Codename for Original targets) to stubs before a public publish (§8.4.3 `--strip`). The rules file is owned by Part 8 (proposed default path `tools/lint_strings.rules.json`; Part 8 may rename it). Schema: `{ "version": 1, "terms": [ { "term": string, "packs": ["Original" | "Codename" | "Canon"], "scopes": ["pack" | "src"], "mode": "always" | "public", "note": string? } ] }`. The file is never under `src/`, so denylisted names never ship.

#### 7.5.2 `tools/check_manifest.luau` (placeholder-asset lint)

Usage: `lune run tools/check_manifest.luau [--release] [--game <universeId>] [--verbose]`. Loads `Shared/Assets.luau` (load-pure, Part 2 §2.9.1) and `Shared/Styles` through the loader, and scans `src/**` text.

| Rule | Severity | Check |
|---|---|---|
| M01 key format | error | Key matches `^%l+%.[%l%d_]+%.[%l%d_]+$`, first segment ∈ {`anim`, `vfx`, `sfx`, `img`, `mesh`, `model`}, and `kind` matches the prefix (`sfx` → `audio`, `img` → `image`) |
| M02 entry shape | error | `status` ∈ {placeholder, final}; `source` ∈ {builtin, agent, ai, creator-store, artist}; `id` is `rbxassetid://<digits>`, `rbxasset://<path>`, `tpl:<Anim\|VFX\|Looks\|Templates>/<Name>` or `""` |
| M03 provenance | error | `final` needs an `rbxassetid://` id; `source = "ai"` is never `final` and never under `model.<styleId>.*` (§0.5); `creator-store` ids have a row in `docs/agent/ASSET_PROVENANCE.md` |
| M04 raw ids | error | No `rbxassetid://`, `rbxasset://`, `roblox.com/asset` or numeric `Content.fromAssetId(` literal outside `src/Shared/Assets.luau` |
| M05 referenced keys exist | error | Every literal key passed to `Assets.get/entry/template/isPlaceholder`, every MoveDef `anim/vfx/sfx`, every `StyleDef.m1` key and every `kitProps` entry exists |
| M06 unused entries | warning | Entry never referenced (dynamic keys are listed with `--verbose`) |
| M07 products | error | `Assets.products[0]` has every `ProductKey` and `PassKey` (Studio mock); other universes have numeric ids > 0; `--release --game <id>` requires the full set for that universe |
| M08 release placeholders | error (`--release` only) | No referenced key has `status = "placeholder"` or `id = ""` |

In M0/M1-G (graybox) `--release` is expected to fail; that is the §0.5 rule blocking public releases.

#### 7.5.3 `tools/export_framedata.luau`

Usage: `lune run tools/export_framedata.luau [--check] [--golden] [--pack Original]`.
- Builds rows with the pure `Moves.frameRows(style: Fsm.StyleRuntime?, cfg): { FrameRow }` (new export of `Core/Moves.luau`; `nil` style = chassis rows: M1-1..4, the three Ender variants, air M1, Dash M1, the four dashes, block/Perfect Block/Guard Break, Breakaway, Awakening activation). The same function feeds Part 6's Training Grounds frame-data panel, so docs and game cannot drift.

```lua
export type FrameRow = {
	id: string,                     -- "chassis.m1_1", "hero.3", "chassis.dash_side"
	labelKey: string,               -- Skin key, e.g. "move.hero.3.name"
	startup: number, active: number, recovery: number, total: number,  -- frames @ 60 Hz
	onHit: number?, onBlock: number?,   -- advantage = stun − (active − 1) − recovery (Part 4 K04 formula)
	damage: number?, guardDamage: number?, chip: number?, hitstun: number?,
	cooldownSec: number?, iFrames: { number }?, telegraphLead: number?, tags: { string },
}
```
- Writes (through `tools/lib/SafeWrite.luau`, §7.5) `docs/reference/framedata.json` (Part 4 contract) and `docs/reference/framedata.md` (one table per style plus the chassis table, labels from the `--pack` pack, default `Original`). Output is byte-deterministic: sorted rows, no timestamps, header line `generatedFrom: config v<Version> hash <fnv32 of Defaults JSON>`.
- `--golden` regenerates Part 3's `tests/core/golden/*.json`.
- `--check` regenerates in memory, compares byte-for-byte, prints `[ALG][LINT] <file> stale` for each difference, exits 1 if any. CI runs it.

#### 7.5.4 Kit validator invocation

- Host: `styles.validate_all_clean` and the 31 `styles.rule_<ID>_detects` tests (Part 4 §4.1.4) run in every host gate.
- Studio: `StyleService.init()` calls `Styles.validate(Runtime.get())`; `style.registry_boot` checks the `registry ok` line.
- Ad hoc: `lune run tests/run.luau --filter styles.` prints each rule result; failures print Part 4's `[ALG][Style] ERROR rule=<id> …` text in the FAIL detail.

#### 7.5.5 Lint rule additions (`tools/lint_rules.luau`, Part 1 owner)

| Rule ID | Scope | Check |
|---|---|---|
| `devtools-isolation` | `src/**` | The string `DevTools` appears only in `src/DevTools/**` and `Services/TestHarness.luau`; `TestRun` only there and in `Main.client.luau`; `CreateVirtualInput` only in `src/DevTools/Tests/Client/**` and in the P0 probe files `src/DevTools/Tests/Spikes/**` and `src/DevTools/Tests/Spike_*.luau` (decision D-12: Probe_SA03/SA14 keep their `CreateVirtualInput` calls; the same files are exempt from Part 3's one-`BindToSimulation(` rule) |
| `test-loader` | `tests/**`, `tools/**` | `require("@shared/` appears only inside `typeof(require("@shared/…"))` |

---

### 7.6 Performance measurement

#### 7.6.1 Budgets oracle (`src/DevTools/Tests/Lib/Budgets.luau`)

Test oracle only; live code reads Part 6's `Defaults.Perf`. `perf.budgets_match_gdd` pins the numbers.

| Budget | Low (reference) | Mid | High | Source | Gate use |
|---|---|---|---|---|---|
| Target fps (20 players, 5 awakenings, 2 Realms) | 30 | 45 | 60 | GDD §18.7 ("45–60": lower bound used) | Device (HD-02) |
| Draw calls | 600 | 900 | 1,500 | GDD §18.7 | Studio proxy at Mid; device at Low |
| Triangles on screen | 350k | 600k | 1M | GDD §18.7 | Studio proxy at Mid; device at Low |
| Client memory (MB, 1 GB read as 1,000 MB, td) | 1,100 | 1,600 | 2,500 | GDD §18.7 | Device (HD-03) |
| Unique flipbook textures | 40 | 70 | 120 | GDD §18.7 | Static, at Low |
| Server heartbeat | ≥ 55 Hz P95 at 20 players | — | — | GDD §18.7, §20.2 | Studio and published |
| Active particle emitters | 120 | 220 | 400 | GDD §13.5 | Studio at the active tier |
| Max particles on screen | 1,500 | 3,000 | 6,000 | GDD §13.5 | Studio at the active tier (estimate) |
| Beams/trails | 40 | 80 | 150 | GDD §13.5 | Studio at the active tier |
| Per-move emitter cap | Starter–Common 3, Uncommon–Rare 5, Epic 6, Legendary 7, Mythic–Cosmic 8 | same | same | GDD §13.5 | Static |
| Prediction success | ≥ 97 % | | | GDD §18.2 | Multi proxy + SA visualizer |
| Perfect Block delta vs 30 ms baseline | ≤ 5 points | | | GDD §18.2 | `multi.pb_rhythm` pair + HF-13 |
| Time to first fight | ≤ 20 s (P50) | | | GDD §20.2 | Studio proxy + HD-01 |

#### 7.6.2 Server sampler (`Lib/PerfSampler.luau`)

Samples on `RunService.Heartbeat`. Per frame: `Stats.HeartbeatTime` (seconds; `HeartbeatTimeMs` is deprecated [Docs `Stats.yaml`]). Every wall second: heartbeat count (→ Hz), `Stats:GetTotalMemoryUsageMb()`, `Stats.InstanceCount`, `Stats:GetMemoryUsageMbForTag(Enum.DeveloperMemoryTag.LuaHeap)` only if `Stats.MemoryTrackingEnabled` (it returns 0 and warns otherwise [Docs]), `Stats.DataSendKbps`/`DataReceiveKbps`, bot count (`Workspace.Bots`), carrier count (`workspace.Sim`), Realm markers (`Realm_*`).

```lua
export type Series = { first: number, last: number, max: number }
export type ServerPerf = {
	sec: number, hzMean: number, hzMin: number, hzP5: number, pctSecAtOrAbove55: number,
	heartbeatMsMean: number, heartbeatMsP95: number,
	memMb: Series, luaHeapMb: Series?, instances: Series,
	botsMax: number, carriersMax: number, realmsMax: number, sendKbpsMean: number, recvKbpsMean: number,
}
-- PerfSampler.start(): { stop: () -> ServerPerf }   percentiles use nearest rank: sorted[clamp(ceil(p·n), 1, n)]
```

Heartbeat criterion (GDD "≥ 55 Hz P95"): `pctSecAtOrAbove55 ≥ 95`. Log on stop: `[ALG][Perf] server scn=<name> hzP5=<n> pct55=<n> hbMsP95=<n> heapMb=<a>-><b> inst=<a>-><b>`.

Leak checks use `SceneAnalysisService:GetUnparentedInstancesAsync()` (`Size` field; security None [Docs `SceneAnalysisService.yaml`]; availability in Play DataModels: verify in spike T7-S10).

#### 7.6.3 Client probe (`Client/PerfProbe.luau`)

Per `RenderStepped`: `fps = 1 / Stats.FrameTime` (client only, seconds [Docs]). Every second: `Stats.SceneDrawcallCount`, `Stats.SceneTriangleCount`, `Stats.RenderCPUFrameTime`, `Stats.RenderGPUFrameTime`; memory tags `GraphicsParticles`, `GraphicsTexture`, `LuaHeap`, `Gui`, `Animation` (if tracking is enabled); VFX counts from `VfxController.stats()` when present, else `workspace:QueryDescendants("ParticleEmitter[Enabled = true]")` (selector grammar [Docs `Instance.yaml`]), `Beam[Enabled = true]`, `Trail[Enabled = true]`, and `particlesEst = Σ Rate × Lifetime.Max` (ParticleEmitter has no live-count property [Docs]). Reports `fpsP50`, `fpsP5`, `dcMax`, `triMax`, `emMax`, `ptMax`, `btMax`, `rb`, `pred` via `testReport`. Frame-time mode (spike S-P6-13, Part 6 §6.15, decision D-51): the probe also samples `Stats.RenderCPUFrameTime` every `RenderStepped` frame for a 10 s baseline window and a 10 s test window (while 12 Highlights/s are toggled) and reports `rcpuBaseP95Ms` and `rcpuTestMaxMs`; pass when no test-window frame is more than 2 ms above the baseline P95.

Studio-machine fps and total memory are recorded but never gated (Studio shares the process with the editor). Low-tier fps and memory are gated only on the device (HD-02, HD-03).

#### 7.6.4 Server Authority metrics

- **Prediction success proxy** (automated): `predSuccessPct = 100 × (1 − rollbacks / (60 × windowSec))`, where `rollbacks` counts `RunService.Rollback` on the client (Basic capability). Gate: ≥ 97 % (GDD §18.2), i.e. ≤ 1.8 rollbacks per second in sustained combat. Solo sessions use `tools/studio/mispred_probe.luau` (Client); multi sessions use the client driver.
- **Authoritative reading** (human): the SA visualizer (Ctrl+Shift+F6) shows prediction success % over 8 s and RCC heartbeat [Docs `projects/server-authority`]. It is recorded in `human.sa_visualizer_reading`. If proxy and visualizer differ by > 2 points, the visualizer wins and the difference goes to *Spike results* (T7-S6).
- **Attribution and misprediction rate:** `RunService.Misprediction` has capability `PluginOrOpenCloud` [Docs], so only the plugin (client DMs) or `execute_luau` may connect. The plugin publishes `AlgMpN` and the top three mispredicted names (`AlgMpTop`); the driver reports them. When `AlgMpN` is available the same threshold applies to it: `AlgMpN / (60 × windowSec) ≤ 0.03` (≤ 108 events per minute). Any name outside Part 3 §3.16.2's expected list accounting for > 10 % of events is logged `[ALG][Perf] WARN mispredict name=<n> share=<p>` (warning, not a gate).
- **Local prediction status:** `client_status.luau` (Part 1) must report `Predicted` for the local root.

`tools/studio/mispred_probe.luau` (Client; 1,096 bytes):

```lua
--!strict
-- tools/studio/mispred_probe.luau | Client | ARGS {"op":"start"} then {"op":"read"}
local HS = game:GetService("HttpService")
local RS = game:GetService("RunService")
local A = HS:JSONDecode([==[__ARGS__]==])
local lp = game:GetService("Players").LocalPlayer :: Player
local function bump(k: string)
	lp:SetAttribute(k, ((lp:GetAttribute(k) or 0) :: number) + 1)
end
if A.op == "start" then
	lp:SetAttribute("AlgProbeT0", os.clock())
	lp:SetAttribute("AlgProbeRb", 0)
	lp:SetAttribute("AlgProbeMp", 0)
	RS.Rollback:Connect(function()
		bump("AlgProbeRb")
	end)
	local ok, err = pcall(function()
		RS.Misprediction:Connect(function()
			bump("AlgProbeMp")
		end)
	end)
	return HS:JSONEncode({ ok = true, misprediction = ok, err = if ok then nil else tostring(err) })
end
local sec = os.clock() - ((lp:GetAttribute("AlgProbeT0") or os.clock()) :: number)
local rb = (lp:GetAttribute("AlgProbeRb") or 0) :: number
return HS:JSONEncode({
	ok = true,
	sec = sec,
	rollbacks = rb,
	mispredictions = lp:GetAttribute("AlgProbeMp"),
	predSuccessPct = 100 * (1 - rb / math.max(1, sec * 60)),
})
```

Whether connections made by an `execute_luau` call keep firing after the call returns is verified in spike T7-S7.

#### 7.6.5 Device Simulator presets

`StudioDeviceSimulatorService` methods are `PluginSecurity`, work in Edit and Play Client, and the state-changing ones error in PlayServer [Docs `StudioDeviceSimulatorService.yaml`]. Custom presets (td) are created once and found by name:

| Preset key | Name | Resolution | DPI | Form | Use |
|---|---|---|---|---|---|
| `phone1334` | ALG Phone 1334x750 | 1334 × 750 | 326 | Phone | GDD §16.3 text-size reference |
| `androidLow` | ALG Android Low 1600x720 | 1600 × 720 | 270 | Phone | Low-end layout |
| `tablet` | ALG Tablet 2048x1536 | 2048 × 1536 | 264 | Tablet | Tablet layout |
| `console` | ALG Console 1920x1080 | 1920 × 1080 | 96 | Console | Gamepad UI |

`tools/studio/device_sim.luau` (Edit or Client; 1,507 bytes):

```lua
--!strict
-- tools/studio/device_sim.luau | Edit or Client | ARGS {"op":"set","preset":"phone1334"} ops: list|set|stop
local HS = game:GetService("HttpService")
local DS = game:GetService("StudioDeviceSimulatorService")
local A = HS:JSONDecode([==[__ARGS__]==])
local F = Enum.DeviceForm
local PRESETS: { [string]: { [string]: any } } = {
	phone1334 = { Name = "ALG Phone 1334x750", Width = 1334, Height = 750, PixelDensity = 326, DeviceForm = F.Phone },
	androidLow = { Name = "ALG Android Low 1600x720", Width = 1600, Height = 720, PixelDensity = 270, DeviceForm = F.Phone },
	tablet = { Name = "ALG Tablet 2048x1536", Width = 2048, Height = 1536, PixelDensity = 264, DeviceForm = F.Tablet },
	console = { Name = "ALG Console 1920x1080", Width = 1920, Height = 1080, PixelDensity = 96, DeviceForm = F.Console },
}
local ok, res = pcall(function(): any
	if A.op == "stop" then
		DS:StopSimulationAsync()
		return "default"
	end
	local ids = DS:GetDeviceListAsync()
	if A.op == "list" then
		return ids
	end
	local cfg = PRESETS[A.preset]
	assert(cfg ~= nil, "unknown preset")
	local found: string? = nil
	for _, id in ids do
		if DS:GetDeviceInfoAsync(id).Name == cfg.Name then
			found = id
			break
		end
	end
	local id = found or DS:CreateDeviceAsync(cfg)
	DS:SetDeviceAsync(id)
	if cfg.DeviceForm ~= F.Console then
		DS:SetOrientationAsync(Enum.ScreenOrientation.LandscapeRight)
	end
	return { id = id, resolution = tostring(DS:GetResolutionAsync()) }
end)
return HS:JSONEncode({ ok = ok, res = res })
```

The simulator emulates screen size and touch, not device CPU/GPU, so it is used only for layout tests (`mcp.touch_layout_phone`) and captures, never for fps.

#### 7.6.6 Perf run procedure

1. Apply the network profile (§7.1.5) and launch the multi test (§7.3.8) with the perf selector; the scenario spawns bots through `ctx.dev(nil, "spawnBots", {count, mapId})` (Part 3, ≤ 20).
2. The scenario starts `PerfSampler` after a 20 s warm-up (bots spawned, clients ready), runs the scripted load, stops the sampler, and collects client probes.
3. Pass/fail per §7.4.9. Metrics land in `report.metrics` and `report.clients`.
4. Evidence: `perf: perf.m0_gate_load hzP5=<n> pct55=<n> pred(min)=<n> errors=0`.

#### 7.6.7 Load-test bots in the published dev place

`Dev.Command` does not exist outside Studio, so device tests (H-12) need another way to spawn bots. New config keys in Part 2's `Features` section (td defaults):

```lua
LoadTestBots = 0,          -- 0..20; brawl bots spawned by BotService on the server (td)
LoadTestMapId = "",        -- MapId the bots spawn in; "" = the server's default map
LoadTestGameIds = {},      -- game.GameId allow-list; bots spawn only if the running universe is listed
```

BotService (Part 3 §3.15.1) spawns `LoadTestBots` brawl bots only when

```lua
not RunService:IsStudio() and table.find(cfg.Features.LoadTestGameIds, game.GameId) ~= nil and cfg.Features.Bots ~= false
```

at `start()` and on `Runtime.changed`, and logs `[ALG][Bot] loadtest bots=<n> map=<id>`. The Studio guard matters because Studio play sessions of the dev place run with the dev universe's `game.GameId` and receive its ConfigService values (staged ones included), so without it every later Studio playtest, including gate runs, would spawn the load-test bots. Scenario `bot.loadtest_off_in_studio` (§7.4.3) pins the guard.

The human sets `ALG_Features` in the **dev** universe's ConfigService only (H-12 preparation). Staging and prod never list their ids. Order (decision D-20): the gate card posts the H-12 preparation (`LoadTestBots`, `LoadTestMapId`, `LoadTestGameIds`) only **after** every automated item has run, just before the device item (M0 H3, M1-G H2); after each H-12 run is recorded, the human sets `LoadTestBots = 0` again.

#### 7.6.8 KPI instrumentation (GDD §20)

| KPI (GDD §20.2) | Target | Source event (GDD §20.4) | Pre-launch verification |
|---|---|---|---|
| FTUE completion | ≥ 85 % | `ftue_step` | `tel.ftue_monotonic` (step 1 `cold_open` is absent in M1-G: Part 6 §6.9.10 FTUE-lite emits steps 2–7 and the server 8–9, so the M1-G funnel starts at step 2) |
| First-Wish completion | ≥ 95 % | `wish_made` | `tel.echo_wish` |
| Time to first fight (P50) | ≤ 20 s | FTUE timing | `perf.boot_time_to_fight`, HD-01 |
| Client fps (P50, low tier) | ≥ 30 | `perf_sample` (1 % of clients; sender: Part 6 VfxController while `PerfSampled`, Part 5 §5.8.3) | HD-02; `tel.perf_sample_sent`; `Telemetry.Perf` validators (Part 2) |
| Server heartbeat (P95) | ≥ 55 Hz | live server stats | `perf.*` server sampler, `human.sa_visualizer_reading` |
| Crash-free sessions | ≥ 99.5 % | platform | Proxy: 0 errors in `perf.m1g_soak`; HD-06 |
| Payer conversion, pass attach, subscriptions | targets | `purchase`, `store_view`, `currency_flow` | `tel.echo_economy`, `store.*` |
| Reports per 1,000 sessions | downward | report events | `mod.report_logged` |
| Retention, session length, rage-quit, WAF | targets | platform analytics, `match_summary` | Not testable before launch (future hook) |

---

### 7.7 Gates

#### 7.7.1 Gate record format

The agent appends to `docs/agent/PROGRESS.md`:

```markdown
## Gate M0 — 2027-01-04 — commit 3f2a9c1
| Item | Result | Evidence |
|---|---|---|
| A1 Host gate | PASS | [ALG][CHECK] SUMMARY pass=9 fail=0 |
| H1 Feel (H-10) | PASS | HF-01..HF-14 PASS; notes: … |
Verdict: [ALG][GATE] M0 PASS automated=13/13 human=5/5
```

The verdict line also goes into the PR body. `PASS` needs every automated item PASS and every human item PASS or "accepted by owner" with a note. The human creates the `m0-gate` / `m1g-gate` tag (Part 1 §1.9).

#### 7.7.2 M0 gate (after P5 `hero`/`stretch` and P6 arena tasks)

Preconditions: H-01 … H-05 and H-08 done; `AlgebraicTestHarness` installed; spike results recorded.

| # | Item | Command / call | Pass |
|---|---|---|---|
| A1 | Host gate | `lune run tools/check.luau` | `[ALG][CHECK] SUMMARY … fail=0`; tests `fail=0` |
| A2 | CI | GitHub Actions `ci` on the phase branch head | success |
| A3 | World | `run_builder` `{"builder":"Verify","fn":"verify"}` | `res.ok == true` |
| A4 | Studio scenarios | `test_run` once per batch with selector `<batch>,-@m1g,-@slow,-@net,-@spike` (there is no intersection operator, so the M0 filter is the `-@m1g` removal; `-@spike` drops spike scenarios such as `store.receipt_live_spike`, decision D-47). Batches: (1) `boot.*,config.*,net.*,data.*,player.*,security.*,tree.*,attrs.*`; (2) `sim.*,input.*,bot.*,render.*,harness.*`; (3) `combat.*`; (4) `style.*,kit.*,anim.*`; (5) `world.*,map.*,training.*,vfx.*,sfx.*,cam.*,ragdoll.*,look.*,hud.*,ip.*`. After each batch, Server and Client `log_scan` with `allowErrors = {"ERROR rollback", "ERROR out_of_band_mutation", "ERROR pack_downgraded", "ERROR dev_universe_non_member"}`: the `data.*` and `ip.*` scenarios provoke these lines on purpose (Part 1 §1.4.6 `algErrors`, D-11; TASKS T039) | every batch `state = "done"`, `fail=0` (an `empty_selection` error fails the item); `log_scan` `ok == true` |
| A5 | MCP tests | `@m0` `mcp` scenarios via §7.3.7 | PASS (fallbacks noted) |
| A6 | SA spikes | `spike.SA-01`, `-03`, `-08`, `-13` | PASS, or the Part 3 in-SA fallback recorded |
| A7 | Network solo | `netcond.profile_applied`, then `test_run` `@net,-@m1g` (`netcond.rtt_m0gate`, `combat.rewind_hit_150ms`, `render.undo_spark`, `render.remote_telegraph_full` and every other `net`-tagged M0 solo scenario) under `M0Gate`, twice | PASS both runs |
| A8 | Multi baseline | `multi_run` `{"n":2,"sel":"multi.*,-@m1g","net":"Baseline30"}` | `fail=0`; record `pbRatePct` = B |
| A9 | Multi gate profile | Same with `"net":"M0Gate"`, twice | `fail=0` both; `pbRatePct` = M ≥ 90 and \|B − M\| ≤ 5 (GDD §18.2) |
| A10 | Load (GDD §18.2) | `perf.m0_gate_load` (n = 4, `M0Gate`) | §7.4.9 criteria |
| A11 | Load (§0.0) | `perf.m0_clients8` (n = 8, `M0Gate`) | §7.4.9 criteria |
| A12 | Static perf and boot | `perf.static_vfx_caps`, `perf.boot_time_to_fight` | PASS |
| A13 | Frame data | `lune run tools/export_framedata.luau --check` | exit 0 |
| H1 | Feel (H-10) | §7.7.4 HF-01 … HF-14 on KB+M and gamepad | PASS or accepted |
| H2 | SA visualizer | Human plays client A during an A10 rerun | `human.sa_visualizer_reading` ≥ 97 % |
| H3 | Device (H-12) | §7.7.5 HD-01 … HD-09; published dev place (H-11) with `LoadTestBots = 12` on the M0 map **plus ≥ 7 additional live clients** (team members or alt accounts) in the same server, so the phone renders 8 replicating clients as §0.0 states. If 7 extra clients are not available, the owner explicitly accepts bots as stand-ins and the gate note records it. The agent posts the H-12 preparation only after A1–A13 have run (§7.6.7); the human resets `LoadTestBots = 0` after recording H3 | fps P50 ≥ 30, memory ≤ 1,100 MB |
| H4 | NetModel decision | Part 3 §3.16.4 rule (SA stays if SA-01/03/08/13 pass) | Human confirms in `PROGRESS.md` |
| H5 | Non-technical GDD §21.2 items (IP path, pillars) | Owner | Recorded as owner sign-off (outside the agent's scope) |

#### 7.7.3 M1-G gate (end of P8)

| # | Item | Command / call | Pass |
|---|---|---|---|
| A1–A3 | As M0 on the release branch | | |
| A4 | All Studio scenarios | `test_run` once per batch with selector `<batch>,-@slow,-@net,-@spike` (both milestones; `-@spike` per D-47). Batches: the five M0 batches of §7.7.2 A4 (they already contain `look.*`, `combat.realm_*`, `combat.stubborn_heart`), plus (6) `wish.*,policy.*,shop.*,progress.*,store.*`; (7) `appearance.*,tel.*,mod.*,ui.*`; `log_scan` with the same `allowErrors` as §7.7.2 A4 | every batch `state = "done"`, `fail=0`; `log_scan` `ok == true` |
| A5 | Slow host tests | `lune run tests/run.luau --tag slow` (Monte Carlo, frequencies, cosmic bound) | `fail=0` |
| A6 | MCP tests | All `mcp` scenarios incl. `mcp.ui_wish_click`, `mcp.wheel_totem`, `mcp.touch_layout_phone` | PASS |
| A7 | Multi | `multi.*,look.modes_matrix,look.stealth_ranges` (n = 2) under `Baseline30` and `M0Gate` (the two `look.*` scenarios are level `multi` and run only here, D-48); also `multi(2, Baseline30, ui.report_submit)` (level `multi`, Part 6 §6.9.12) | `fail=0` |
| A8 | Soak | `perf.m1g_soak`, `perf.leak_after_clear`, `perf.vfx_budget_worst` | §7.4.9 criteria |
| A9 | M0 regression | A7–A11 of §7.7.2 | PASS |
| A10 | Lints | `check_manifest.luau` (dev), `lint_strings.luau` (dev), `export_framedata --check` | exit 0 |
| A11 | Docs | `ARCHITECTURE.md` matches the tree (`lint_rules` `architecture`) | PASS |
| H1 | Feel (H-10) | HF-01 … HF-19 incl. all 6 kits, Realm, Wish flow, Style Wheel, Shop/Store/Settings | PASS or accepted |
| H2 | Device (H-12) | HD-01 … HD-10 on `candy_kingdom` with `LoadTestBots = 19` (20 fighters); HD-10 (menus on touch) is M1-G only. H-12 preparation is posted after A1–A11 (§7.6.7); the human resets `LoadTestBots = 0` afterwards | fps P50 ≥ 30, memory ≤ 1,100 MB; HD-10 PASS |
| H3 | Design sign-offs | Part 3 §3.20, Part 4 §4.11, Part 5 DES-1 … DES-7 (DES-6 free-rotation ratio, DES-7 M1-G earn rate) | Owner decisions recorded |

#### 7.7.4 Human feel checklist (H-10)

Run in Studio (Test → Play, then Server & Clients with 2 clients), at `Baseline30` unless stated. Each item: PASS / FAIL / accepted + one-line note.

| ID | Check | Procedure | Pass |
|---|---|---|---|
| HF-01 | M1 chain rhythm | 10 chains on an idle dummy | All 4 hits land at a comfortable rhythm; a pause resets as expected (GDD §6.3) |
| HF-02 | Hitstop and weight | Compare M1s with the Ender | Ender feels heavier (3f vs 6f hitstop, GDD §6.13); no floaty hits |
| HF-03 | Ender variants | 5 tries each: Uppercut (hold Space), Downslam (M1 in air), Side Ender (hold A/D) | ≥ 4/5 each on KB+M and gamepad |
| HF-04 | Perfect Block | Dummy `m1_spam`, 20 taps on the flash | Learnable; ≥ 10/20 (td); anti-mash lock noticed only when mashing |
| HF-05 | Guard Break | Hold block vs `m1_spam` until break | Break readable; Breakaway disabled for the first 30f |
| HF-06 | Dashes | 10 per direction; side dash through `m1_spam` | Side dash dodges in its i-frames; cooldowns understandable |
| HF-07 | Movement tech | Hop Cancel, Dash-Ender, Wall Carry, Tech Roll ×5 each | Possible with practice, never accidental (GDD §6.5) |
| HF-08 | Breakaway | Get comboed by a bot; press Q | Reliable after the 12f lockout; lockout not felt as a dropped input |
| HF-09 | Combo limits | Bot juggle | No infinite; Shake-Off by 3.5 s |
| HF-10 | Awakening | Meter 100 → G | Readable, invulnerable, heal visible, moves swap, camera not disorienting |
| HF-11 | Style identity | `hero` vs `stretch` | M1 re-skins and moves feel distinct |
| HF-12 | Accessibility | Reduced Motion, Reduced VFX on | No shake or impact frames; telegraphs still full contrast (GDD §16.3) |
| HF-13 | Perfect Block under latency | 20 attempts at `Baseline30`, 20 at `M0Gate` vs a rhythm bot | Success rates within 5 points (GDD §18.2) |
| HF-14 | Rubber-banding | Watch 10 own dashes and 10 enemy dashes at `M0Gate` | No visible snap-back (GDD §18.2) |
| HF-15 | M1-G kits | 3 min each with `monarch`, `rocker`, `frost`, `decay` vs bots | Identity readable; nothing feels unpunishable |
| HF-16 | Realm | `decay` Realm near 2 bots | 90f telegraph readable; escape possible; Stubborn Heart works |
| HF-17 | Wish flow | 1× and 10× Wishes | Reveal durations per rarity; odds button and pity counters visible (GDD §8.6, §8.8) |
| HF-18 | Style Wheel, Totem, Character Look | Respawn swap, Totem swap, toggle look | Discoverable; respawn swap works |
| HF-19 | Shop, Store and Settings flows | Buy a style and a Variant in Choose Goose's Shop; open the Store (mock) and read the weekly spend line; change a Settings value in every tab (incl. the Ooo-sona editor) and see it applied | Every flow completes without confusion; prices, balances and confirmations readable (GDD §8.5, §8.8, §15.4) |

#### 7.7.5 Device checklist (H-12)

Low-end Android reference (3 GB RAM, GDD §18.7), published private dev place (H-11), `LoadTestBots` set as in §7.7.2/§7.7.3.

| ID | Check | Procedure | Pass |
|---|---|---|---|
| HD-01 | Time to first fight | Stopwatch from tapping Play to controllable character, 3 tries | Median ≤ 20 s (GDD §20.2) |
| HD-02 | Frame rate | 3 min in the bot fight; read the on-device performance stats or MicroProfiler (Ctrl+Alt+F6 per GDD §18.7; the phone gesture or menu toggle: verify in spike T7-S15) | P50 ≥ 30 fps (GDD §18.7) |
| HD-03 | Memory | Developer Console → Memory after 10 min | ≤ 1,100 MB (GDD §18.7, td reading of 1.1 GB) |
| HD-04 | Touch controls | Every button; Dash ↔ Breakaway swap; Launcher toggle; stick-directed dash | All work; no overlap with system UI (GDD §15.2) |
| HD-05 | Text | Read HUD and menus | Legible; nothing below the GDD §16.3 minimum |
| HD-06 | Stability | 15-minute session | No crash or disconnect |
| HD-07 | Reduced VFX | Toggle during the fight | Fewer particles; telegraphs unchanged |
| HD-08 | Feel on touch | M1 chain, block, dash, Breakaway, awakening | Executable and responsive (GDD §21.2 M0 "good feel on mobile") |
| HD-09 | Thermal / battery | Observe over 15 min | Recorded only |
| HD-10 | Menus on touch (M1-G only) | Open each of: Style Wheel (incl. the detail pane), Wish with the odds view and a 10× reveal, Shop (both tabs), Store, Settings (all tabs incl. the Ooo-sona editor), the Totem prompt | All targets tappable (≥ 44 px), no text below 14 px, nothing under the notch or system UI (GDD §15.1 mobile-first at 1334×750, §16.3) |

#### 7.7.6 Publish checklist hook (Part 8 gate)

Part 8 owns the publish gate; Part 7 provides these automated items, run before H-11 for any **public** publish:

| # | Item | Command | Pass |
|---|---|---|---|
| R1 | Strings (public) | `lune run tools/lint_strings.luau --public --pack <pack>` | exit 0 |
| R2 | Assets (release) | `lune run tools/check_manifest.luau --release --game <prod universeId>` | exit 0 (no referenced placeholder, all products present) |
| R3 | Release-tagged tests | `lune run tests/run.luau --tag release` (the six Lune tests of §7.1.4: `lint.manifest_release_detects_placeholder`, `lint.manifest_raw_id_detected`, `net.validators_reject_fuzz`, `skin.canon_locked`, `ip.leak_detect`, `ip.product_keys_original_only`) and Studio `@release` (`security.dev_hooks_studio_only`, `harness.studio_only_guard`, `security.mock_ignored_live`, `store.mock_ignored_live`, `tel.no_analytics_in_studio`) | Lune ≥ 6 tests and Studio ≥ 5 scenarios, `fail=0` |
| R4 | Full M1-G gate on the publish commit | §7.7.3 automated items | PASS |

The agent prints `[ALG][GATE] publish-hook PASS|FAIL items=<n>/4`; Part 8 combines it with its IP checklist into the final verdict. A private dev-place publish follows Part 8's dev rule (R2 is expected to fail on graybox placeholders there).

**Dry run (`ip.gate_dry_run`, decision D-23).** `ip.gate_dry_run` is an **agent procedure**, not a Studio scenario: it needs host commands (`lint_strings`, the sync gate), Edit-mode builders and git, none of which a play-session scenario can run, so it never appears in a `studio(…)` selector. Part 8 §8.4.7/§8.4.9 define its steps (throwaway branch `release/<YYYY-MM-DD>-original-dryrun`, target Original, placeholder metadata). When it evaluates this hook: R1 runs as `lint_strings --public --pack Original --dry-run` (L11/L17/L22 become `W-DRYRUN` warnings); R2 is expected to FAIL (graybox placeholders); R3 runs as written; R4 is taken from the enclosing gate card's own automated results (§7.7.3 A1–A11 of that run), not from a second full gate run, so it is not circular. The agent records `[ALG][GATE] publish-hook FAIL items=3/4` as the expected result only when R1, R3 and R4 pass; any other failure is a real failure.

---

### 7.8 CI on GitHub Actions (Linux)

#### 7.8.1 Workflow (`.github/workflows/ci.yml`)

```yaml
# .github/workflows/ci.yml — host gate on GitHub-hosted Linux. Spec §7.8.
name: ci
on:
  push:
    branches: [main, "phase/**"]
  pull_request:
  workflow_dispatch:
    inputs:
      release:
        description: "Also run the release (publish-hook) host checks"
        type: boolean
        default: false
permissions:
  contents: read
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
jobs:
  host:
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    defaults:
      run:
        shell: bash
    steps:
      - uses: actions/checkout@v4
      - name: Install Rokit
        run: |
          curl -sSf https://raw.githubusercontent.com/rojo-rbx/rokit/main/scripts/install.sh | bash
          echo "$HOME/.rokit/bin" >> "$GITHUB_PATH"
      - name: Cache tools and selene std
        uses: actions/cache@v4
        with:
          path: |
            ~/.rokit
            roblox.yml
          key: tools-${{ runner.os }}-${{ hashFiles('rokit.toml', 'selene.toml') }}
      - name: Install pinned tools (rokit.toml)
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          rokit trust rojo-rbx/rojo lune-org/lune JohnnyMorganz/StyLua Kampfkarren/selene JohnnyMorganz/luau-lsp
          rokit install
          rojo --version && lune --version && stylua --version && selene --version && luau-lsp --version
      - name: Host gate (stylua, selene, sourcemap, luau-lsp, lint_rules, lint_strings, check_manifest, Lune tests)
        run: |
          set -o pipefail
          mkdir -p build
          lune run tools/check.luau 2>&1 | tee build/check.log
      - name: Frame data is current
        run: lune run tools/export_framedata.luau --check
      - name: Rojo build
        run: rojo build default.project.json --output build/Algebraic.rbxl
      - name: Release host checks (manual)
        if: ${{ inputs.release }}
        run: |
          lune run tools/lint_strings.luau --public
          lune run tools/check_manifest.luau --release
          lune run tests/run.luau --tag release
      - name: Upload logs
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: ci-logs
          path: build/check.log
          retention-days: 7
```

Rules:
- CI runs exactly the host gate the agent runs locally (`tools/check.luau`, Part 1 §1.4.2), never `--fast`, plus the frame-data check and `rojo build` (GDD §18.9 steps 1–4 and 6).
- The human marks `ci / host` as a required status check on `main` (repository settings; human step attached to the CI task).
- No secrets beyond the default `GITHUB_TOKEN`; no Open Cloud publishing in M0/M1-G (the agent never publishes; GDD §18.9 step 5 and 7 are future hooks).
- Rokit trust and install behaviour on a fresh runner and the selene std generation are verified in spike T7-S13.

#### 7.8.2 What stays agent- or human-driven

| Not in CI | Why | Who runs it |
|---|---|---|
| L2 Studio scenarios, L3 MCP tests | GitHub-hosted Linux runners cannot run Studio | Agent via MCP |
| L4 multi-client, L5 network profiles | Need Studio and `StudioTestService` | Agent via MCP + plugin |
| L6 perf and soak | Need Studio, many DataModels, real devices | Agent (Studio), human (device) |
| L7 feel and device | Human judgement | Human (H-10, H-12) |

A future self-hosted Windows/macOS runner with Studio could run L2 via the Studio command line; it is out of scope for M0/M1-G.

#### 7.8.3 Acceptance

- A push to a `phase/**` branch runs `ci` and fails if any host check fails (verify by pushing a commit with a deliberately failing Lune test on a scratch branch, then deleting the branch; the agent asks before pushing, Part 1 §1.9).
- The `ci` job finishes in ≤ 20 minutes; the Lune test step in ≤ 60 s.

---

### 7.9 New contracts introduced

1. **Lune test infrastructure:** `tests/run.luau` CLI and exit codes (§7.2.2); spec files return `function(T: TestApi)`; `tests/shim/T.luau` API; `tests/shim/Loader.luau` (`L.require`, `Loader.new(extraEnv)`, Roblox require semantics, `Vector3`/`CFrame`/`Color3` from `@lune/roblox`, the Lune 0.10.5 `CFrame.lookAt` patch); folders `tests/core/fixtures/` and `tests/core/golden/` are not scanned.
2. **Test ID pattern** `^[%w_]+%.[%w_%.%-]+$`; tags `m0`, `m1g`, `net`, `slow`, `spike`, `release`, `selftest`; info lines `[ALG][Test] slow|pending|skip …`.
3. **DevTools test layout:** `src/DevTools/Tests/Lib/{Types, Registry, Runner, Ctx, Fixtures, Tap, DevCommands, Multi, PerfSampler, Budgets, NetProfiles}.luau`, `Client/{Driver, Steps, PerfProbe}.luau`, `Suites/<Name>.luau` (names in §7.3.1), `Spike_<ID>.luau` with scenario id `spike.<ID>` (`Spike_SAxx` Part 3; `Spike_SP4_2`, `Spike_SP4_3` Part 4; `Spike_SP6_2` Part 6; TASKS C4).
4. **Scenario, Ctx, Fighter, Pad, TapRecord, Recorder, RunReport, McpStep types** (§7.3.2) and the authoring rules.
5. **Request/result protocol:** `ServerStorage.AgentTestRequest` / `AgentTestResult` StringValues, request JSON fields, ops `run|setup|assert`, states `queued|running|ready|done|error`, multi requests and `EndTest` values as JSON strings.
6. **Verify leftovers** (Part 1 §1.6.7 item 5 and §1.3 step 8): in Edit, `ServerStorage.AgentTestRequest`, `ServerStorage.AgentTestResult`, `ServerStorage.AgentTestMultiArgs`, `Workspace.TestPad` and `ReplicatedStorage.TestRun` are `[leftover]` problems the agent may remove (only between runs, never while a multi test is running). Part 1's list also holds `Workspace.ALG_Local` (Part 6) and `ReplicatedStorage.__AlgSpike` (TASKS C4); §1.3 step 8 removes only listed names.
7. **Studio snippets** (Part 1 catalog): `test_run.luau` (Server), `test_result.luau` (Server/Edit), `multi_run.luau` (Edit), `net_profile.luau` (Edit), `device_sim.luau` (Edit/Client), `mispred_probe.luau` (Client).
8. **Plugin:** `tools/plugin/AlgebraicTestHarness/{Main, Top}.luau`, `tools/plugin/build.luau [--install]` (writes only through `tools/lib/SafeWrite.luau`, Part 8 §8.4.5; so does `export_framedata.luau`, §7.5), output `build/AlgebraicTestHarness.rbxm`; client-local Player attributes `AlgMpN`, `AlgMpTop` (and `AlgProbe*` from the probe snippet).
9. **Runtime-only instances:** `Workspace.TestPad`, `ReplicatedStorage.TestRun` (Folder: `Spec` StringValue + cloned `Client` folder).
10. **Seams** (§7.3.10): `CombatService._tap`, `BotService._setBrain`, `Net.server._dispatch`, the `Main.client.luau` Studio hook, DevCmd `testReport`; used from other parts: `CombatService._getMeter` (Part 3 §3.13.7), `Telemetry._setPerfSampled` (Part 5 §5.8.3), `PlayerService._leaveTest` (Part 2 §2.3.5), `DataService._readStored` (Part 2 §2.8.5), `PlayerService._devAccessCheck` and `ConfigReplicator._devContextOverride` (Part 2 §2.3.5, §2.6.4; Part 8 §8.2.6).
11. **`Moves.frameRows` and `FrameRow`** in `Core/Moves.luau` (Part 3 owner), shared with Part 6's Training panel; outputs `docs/reference/framedata.json` and `framedata.md`; `export_framedata --check/--golden/--pack`.
12. **`VfxController.stats()`** (Part 6) return shape (§7.3.10).
13. **Config keys** `Features.LoadTestBots`, `Features.LoadTestMapId`, `Features.LoadTestGameIds` (Part 2 section, BotService behaviour; spawn guard `not RunService:IsStudio() and table.find(LoadTestGameIds, game.GameId) ~= nil and Features.Bots ~= false`, §7.6.7) and log line `[ALG][Bot] loadtest bots=<n> map=<id>`.
14. **Network profiles** `Off`, `Baseline30`, `M0Gate`, `Wifi`, `Mobile4G`, `Bad3G` (§7.1.5), with the calibration test `netcond.rtt_m0gate`.
15. **Device presets** `phone1334`, `androidLow`, `tablet`, `console` (§7.6.5).
16. **Tool contracts:** `lint_strings.luau` rules L01–L06 and flags (rules data path/schema proposed for Part 8; Part 8 adds `--tree`, `--strip`/`--unstrip`, `--dry-run`; L02 skips strip-stub packs); `check_manifest.luau` rules M01–M08 and flags; tool output format `[ALG][LINT] … SUMMARY errors=<n> warnings=<m>`.
17. **Lint rules** `devtools-isolation` and `test-loader` added to Part 1's `tools/lint_rules.luau`.
18. **Log tags** `Perf` (`[ALG][Perf] …`), `Harness` (plugin lines) and the chat/PR verdict line `[ALG][GATE] <gate> PASS|FAIL …`.
19. **Human checklists** HF-01 … HF-19 and HD-01 … HD-10 (HD-10 M1-G only); gate record format (§7.7.1).
20. **CI workflow** `.github/workflows/ci.yml` (job `host`), required check `ci / host` on `main` (human).
21. **Test names** introduced here: `shim.*`, `runner.cli_contract`, `standards.devtools_isolation`, `standards.test_loader_only`, `harness.*`, `netcond.profile_applied`, `netcond.rtt_m0gate`, `combat.side_ender_angle`, `wish.gdd_tier_odds`, `wish.gdd_flair_odds`, `wish.per_style_division`, `wish.per_style_redivide`, `wish.cosmic_path_bound_4400`, `skin.pack_key_coverage`, `lint.*`, `framedata.*`, `multi.*`, `perf.*` (except Part 6's Lune `perf.defaults_gdd`, `perf.json_safe`, `perf.tiers_ordered`, `perf.audio_map_complete`, `perf.lighting_presets_complete`), `mcp.*` (incl. `mcp.tab_no_playerlist`; `mcp.gamepad_menu_nav` is Part 6's), `human.*`. Catalogued from other parts after the review: `config.not_overridable`, `player.leave_releases`, `data.rejoin_persists` (redefined: stored-profile check, level `multi`), `skin.on_changed` (Part 2); `core.require_graph_acyclic`, `moves.status_codec_long_uptime`, `moves.disabled_move_blocked`, `combat.disabled_move_kill_switch`, `combat.env_ko_credit`, `world.travel_streams_destination`, `bot.loadtest_off_in_studio`, `stun.stretch_ragdoll_minus10` (renamed, TASKS C12) (Part 3); `spike.S-P4-2`, `spike.S-P4-3` (Part 4); `wish.batch10_server` (Studio; renamed from the duplicate `wish.batch10_vector`), `tel.perf_sample_sent`, `progress.daily_quest_claim`, `progression.rookie_rule_per_earner` (Part 5); `look.modes_matrix`, `look.stealth_ranges`, `ui.report_submit` (level `multi`); the Part 6 §6.12 set in §7.4.12, `render.remote_telegraph_full` (net, `M0Gate`) and `spike.S-P6-2` (Part 6).
22. **Selector rules:** a bare word is not a prefix; a selection matching zero scenarios ends with `state = "error"`, `err = "empty_selection"` (§7.3.4). Every gate A4 selector appends `-@spike`.
23. **Release-tagged Lune tests:** `lint.manifest_release_detects_placeholder`, `lint.manifest_raw_id_detected`, `net.validators_reject_fuzz`, `skin.canon_locked`, `ip.leak_detect`, `ip.product_keys_original_only` (§7.1.4, R3).
24. **`ip.gate_dry_run` is an agent procedure** (§7.7.6), never a Studio scenario.

### 7.10 Issues with spine

1. **GDD §18.2 network gate vs the Network Simulator.** `NetworkSettings` loss is capped at 0.50 % per direction [Docs], so "1 % loss" is approximated by 0.5 % inbound + 0.5 % outbound (≈ 1.0 % per round trip). "150 ms ±30 ms jitter" is read as round-trip (Studio's docs: inbound and outbound delay both add to ping), and jitter is assumed additive on top of the minimum, hence 60 + [0, 30] ms per direction. `netcond.rtt_m0gate` calibrates this. No GDD number changes; if the owner meant one-way 150 ms, change only `NetProfiles.M0Gate`.
2. **Two M0 load definitions.** §0.0 says "≥ 30 fps on the low-end reference with 8 clients plus server bots"; GDD §18.2 says "20 bots + 4 humans". Both are gated in Studio (`perf.m0_clients8`, `perf.m0_gate_load`), but those runs measure server Hz and prediction on the dev PC, not low-end client fps. Studio multi tests cap at 8 clients [Docs], so neither can include 20 *clients*. The low-end fps with 8 replicating clients is therefore measured on the device: M0 H3 requires ≥ 7 live clients besides the phone, or the owner's recorded acceptance of bots as stand-ins (§7.7.2). §0.0 is not edited.
3. **§0.4 test-line format** allows only PASS/FAIL. Skips, pending tests and slow tests use `[ALG][Test] …` info lines; no Part 0 change is needed, but the spine could list them.
4. **Cross-part (not spine): Part 1 vs Part 2 boot tag.** Resolved by decision D-02: the bootstrap logs `[ALG][BOOT] server ready …` / `[ALG][BOOT] client ready …` (tag `BOOT`), and readiness is read from `Runtime.ServerReady` and the client-local `AlgClientReady` attribute. The harness also matches tags case-insensitively.
5. **Cross-part: require convention.** Resolved by decision D-05: Part 1 C1 wins (`require("@game/<Service>/<path>")` across roots; spike T006 may switch every file at once in one `OPS:` commit). Part 7 code follows it.
6. **Cross-part: Lune pin.** Part 1 pins Lune 0.10.5, whose `CFrame.lookAt` is wrong for pitched rotations (fixed in 0.11.0, released 2026-10-11). The loader patch makes tests correct on either version. Recommend a dedicated upgrade task to 0.11.x after 2026-10-25; the only breaking change (`net.serve` handle `:stop()`) does not affect this repo.
7. **Cross-part: Part 5's MCP recipe** (`require(TestHarness).run(...)` via `execute_luau`) depends on module-cache sharing (spike T7-S4). The request/result protocol of §7.3.6 is the primary path.
8. **Cross-part: Part 3's `input.mcp_m1_reaches_server`** says "within 10 frames `execute_luau` reads `St == 3`". MCP call latency makes a 10-frame read unreliable; Part 7 asserts the M1 state entry from tap records collected since setup, which proves the same fact.
9. **Cross-part: BuildAll order.** Resolved by decision D-01: Part 2 §2.2 points to Part 1 §1.6.6 `ORDER` (TASKS C6), and `BuildAll` runs a containers step first.

### 7.11 Open questions / spikes

| ID | Question | How to test | Fallback |
|---|---|---|---|
| T7-S1 | Can `execute_luau` (Edit) call `StudioTestService:ExecuteMultiplayerTestAsync`, and does a `task.spawn`ed thread survive after the call returns? | `multi_run.luau` with `"via":"direct"`, `n = 1`, selector `harness.selftest` | Option B plugin (default) |
| T7-S2 | Does a Lune-built `.rbxm` written to `roblox.studioPluginPath()` load as a local plugin after a Studio restart? Does Lune serialize `Script.Source` as Studio expects? | `lune run tools/plugin/build.luau --install`; restart; look for `[ALG][Harness]` lines | Human drags the `.rbxm` into the Plugins folder via Studio; or `rojo build --plugin` (verify) |
| T7-S3 | `settings():GetService("NetworkSettings")` from `execute_luau`/plugin; do values apply to `ExecuteMultiplayerTestAsync` sessions; jitter distribution | `net_profile.luau` then `netcond.rtt_m0gate` in solo and multi | Human applies the preset in the Network pill; adjust `M0Gate` values |
| T7-S4 | Does `execute_luau` (Server) share the game's module cache (same `TestHarness` table)? Its timeout for long calls? | Compare a table identity marker set by `TestHarness.start` | Request/result protocol only |
| T7-S5 | Do client windows of a multi test appear in `list_roblox_studios`, and does `get_console_output` include them? | Run a 2-client test and list | Use only the result JSON |
| T7-S6 | How does the SA visualizer's prediction success relate to the rollback proxy? | Read both during `perf.m0_gate_load` | Calibrate the proxy threshold; the visualizer stays authoritative |
| T7-S7 | Do connections created by an `execute_luau` (Client) call keep firing after it returns; can it connect `RunService.Misprediction` (`PluginOrOpenCloud`)? | `mispred_probe.luau` start/read | Use the client driver (rollbacks) and the plugin (attribution) |
| T7-S8 | Is `UserInputService:CreateVirtualInput()` non-nil in Play clients and multi-test clients; do its keys drive IAS `KBM` bindings and default movement? | Driver step sends `D` + `Q`; server tap shows a side dash | Script bindings via `InputController.fire`; `Humanoid:Move` for direction |
| T7-S9 | `StudioDeviceSimulatorService` from `execute_luau` in Edit and Client; does `NetworkSettings.EmulatedTotalMemoryInMB` change client behaviour? | `device_sim.luau` set/list/stop | Human selects the device in the Device Simulator |
| T7-S10 | `SceneAnalysisService:GetUnparentedInstancesAsync` in Play server/client DataModels | Call from a scenario | Skip the unparented criterion; keep instance/heap criteria |
| T7-S11 | Do plugins load inside the server and client DataModels of `ExecuteMultiplayerTestAsync` sessions? | Plugin prints in each DM | Attribution via the probe snippet in solo only |
| T7-S12 | `LogService:GetLogHistory` keeps 512 entries [Docs]; how fast do long runs evict `[ALG][TEST]` lines? | Run 150 scenarios, then `log_scan` | Rely on `AgentTestResult` JSON (already primary) |
| T7-S13 | Rokit on a fresh `ubuntu-24.04` runner (trust, rate limits with `GITHUB_TOKEN`), selene `roblox.yml` generation, `luau-lsp` flags (Part 1 S12) | First CI run | Download pinned release binaries in the workflow |
| T7-S14 | Is `StudioTestService:GetTestArgs()` populated on the server for `ExecuteMultiplayerTestAsync` sessions (the method docs mention only Play/Run modes)? | `harness.selftest` in multi mode with the plugin's `AgentTestMultiArgs` removed | Already built in: the plugin writes `ServerStorage.AgentTestMultiArgs` before launch and `TestHarness.start` reads it when `GetTestArgs()` is nil |
| T7-S15 | How to read fps and memory on the reference Android device (performance-stats toggle, Developer Console gesture)? | Human during H-12 | Record via MicroProfiler capture |
| T7-S16 | Does a client added with `StudioTestService:AddPlayers` after another client's `LeaveTest` reuse the departed test UserId? (Part 2 §2.14 item 11; TASKS §14 ADD-1; not the same as T7-S11) | In a 2-client run, B calls `LeaveTest`, then `AddPlayers(1)`; compare the new player's `UserId` with B's | `data.rejoin_persists` keeps the `DataService._readStored` stored-profile check (no true rejoin) |

Design questions for the owner:
- **Q1** `multi.pb_rhythm` threshold 90 % and HF-04's "≥ 10/20" are tuning defaults; confirm.
- **Q2** M0 device test: H3 now asks for ≥ 7 live clients besides the phone (§0.0 wording); the owner may instead accept bots as stand-ins, recorded in the gate note (§7.7.2). Confirm which the owner prefers.
- **Q3** Memory budget reading: 1.1 GB as 1,100 MB (used here) or 1,126 MiB?
