# Kickoff brief: building *Hullabrawl* with Claude Code and Roblox Studio

This page is for you, the owner. It gets the project onto your computer, starts the build agent, and tells you when the agent will need you. The agent's own manual is [`AGENTS.md`](../../AGENTS.md); the full plan is the [Studio Build Spec](../STUDIO_BUILD_SPEC.md) and [`TASKS.md`](TASKS.md).

## 1. What you need installed

- **Roblox Studio** (signed in with the account that will own the dev group).
- **Claude Code** on the same computer as Studio. On Windows, run it natively (PowerShell or the VS Code extension), not inside WSL.
- **git** and the **GitHub CLI** (`gh`).

The agent installs the Roblox toolchain (Rojo, Lune, StyLua, Selene, luau-lsp) through Rokit during setup, with your approval.

## 2. Put the project in its own private repository

The build agent pushes work to a **private** GitHub repository (build spec H-01, step 9). Start it from the current files only, without any earlier history:

1. On GitHub, create a new **private** repository, for example `hullabrawl`, with no README.
2. Download the current files: open this repository on GitHub, switch to the branch that holds these docs, then **Code → Download ZIP**. Unzip it to a folder, for example `~/hullabrawl`.
3. In a terminal inside that folder:
   ```
   git init -b main
   git add -A
   git commit -m "Hullabrawl design docs and build spec"
   git remote add origin https://github.com/<your-account>/hullabrawl.git
   git push -u origin main
   gh auth login
   ```
4. If you were given the two private lint lists, copy them into `tools/` now: `tools/retired_keys.txt` and, later from your lawyer, `tools/denylist.local.txt`. They are gitignored and must never be committed. Without them the lints still pass in development (they print a warning or `SKIP`), which is fine; a public build needs both.

## 3. Start the agent

Open a terminal in the project folder, run `claude`, and paste this as your first message:

```
You are the build agent for Hullabrawl, a Roblox battlegrounds brawler set in the Wishaway Isles.
Read AGENTS.md and follow it exactly, then read docs/STUDIO_BUILD_SPEC.md (Part 0) and
docs/build-spec/part-1-operating.md. Start with the first card in docs/agent/TASKS.md (T000) and
work through the cards in the execution order in TASKS §3.2, one green task at a time, recording
evidence in docs/agent/PROGRESS.md. Whenever a card needs me (a human step H-01 to H-12, a restart,
an approval or a feel test), stop, tell me exactly what to click, and wait for me.
```

For every later session (and after each restart the agent asks for), paste:

```
Resume the Hullabrawl build: run the AGENTS.md session preflight, read docs/agent/PROGRESS.md,
and continue with the next card.
```

Start a fresh session at each phase boundary or when the conversation gets long; the agent picks up from `PROGRESS.md`.

## 4. When the agent will need you

The agent drives Studio itself (it starts playtests, reads the output and checks results). It stops and asks you only for the steps below. Full click paths are in build spec §1.2.2.

| Step | When | What you do |
|---|---|---|
| H-01 | First session | Create a dedicated dev group and a private place named **Hullabrawl DEV** (20 players), save a local backup, make a scratch place, give the agent the group ID and your git email |
| H-02 | First session | In Studio: Assistant → ⋯ → Manage MCP Servers → enable Studio as MCP server → Quick connect **Claude Code**. Review `docs/agent/claude-settings.example.json` and copy it to `.claude/settings.json` yourself |
| Restart | After H-02 and the toolchain install | Quit Claude Code and start it again in the project folder, then paste the resume message |
| H-03 | Early in setup | Turn on Script Sync for the four folders the agent names (choose **Keep Disk** on the first sync) |
| H-04, H-05 | Before combat work | Set the Server Authority and Avatar Settings properties the agent lists (these cannot be set by script) |
| H-06 to H-09 | When a card needs them | Studio API access, an Open Cloud key kept outside the repo, placeholder animation uploads, dev-only products and prices |
| H-10 | At each combat gate | Play the build and judge the feel |
| H-11 | At the M0 and M1-G gates | You publish the private dev place after the agent reports its checks green; the agent never publishes. A public build also needs the originality gate (blind tests, title clearance, your lawyer) |
| H-12 | At the M0 and M1-G gates | Test on a low-end Android phone and run the multi-client load check the agent prepares |

## 5. What gets built

1. **M0, the feel prototype:** Sonny and Paddles fully playable with the whole combat chassis, on the Bandstand duel arena and in the Staff Room, with Server Authority netcode and PC, touch and gamepad controls.
2. **M1-G, the graybox vertical slice:**
   - six Charms: Sonny, Paddles, Burrows, Dollop, Bellows and Mothball (with her Getaway)
   - the Elsewhere-on-Sea map in graybox, with its tide
   - Wishes (posted through Pip-Pip) with mock purchases, Sundry's Rack, progression, the HUD and the Charm Ring
   - the Full Uniform toggle and the Postcard Look
   - saving (mocked in Studio)

Graybox means simple shapes and placeholder animation. Final art is made by people under written contracts, after your lawyer and the blind tests sign off (GDD §5, §21).

## 6. Ground rules while it is private

- Keep the repository and the dev place **private**.
- Do not post footage, screenshots or the name publicly until the title search is done and the T11 gameplay-clip test has passed (GDD §5; build spec Part 8).
- Never put another company's characters, names or logos into prompts, assets, text or tags.
- The agent never publishes, never buys anything, and never sets real prices.
