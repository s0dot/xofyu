# *Hullabrawl* — greetings from the Wishaway Isles

An original **battlegrounds fighting game for Roblox**, in the spirit of the genre's big brawlers:
- you make **Wishes** (postcards posted through Pip-Pip the postbox) to collect **Charms**: souvenirs that let your own avatar fight in a staff member's style, across seven rarities from Common to a secret Cosmic
- you brawl on the pier, the beach and the cliffs of a chain of seaside islands whose staff waited 365 summers for their guests to come back

| Document | What it is |
|---|---|
| 📄 **[Game Design Document → `docs/GDD.md`](docs/GDD.md)** | The design: the world, all 18 Charms with full move sets, the Wish system, maps and seasons, art and audio direction, every tuning number, the business case and risks |
| 🛠️ **[Studio Build Spec → `docs/STUDIO_BUILD_SPEC.md`](docs/STUDIO_BUILD_SPEC.md)** | How an AI coding agent builds it in Roblox Studio: contracts, architecture, Server Authority combat, style kits, meta systems, world and UI, tests, the originality layer (Parts 0–8, in [`docs/build-spec/`](docs/build-spec/)) |
| ✅ **[Task plan → `docs/agent/TASKS.md`](docs/agent/TASKS.md)** | Ordered task cards (T000–T299) with acceptance checks, from session setup to the graybox vertical slice |
| 🚀 **[Kickoff brief → `docs/agent/KICKOFF.md`](docs/agent/KICKOFF.md)** | Start here: move the project to a private repo, start Claude Code, paste the first message, and see when the agent needs you |
| 🤖 **[`AGENTS.md`](AGENTS.md)** / [`CLAUDE.md`](CLAUDE.md) | The agent's operating manual (read first by Claude Code) |

## What's in the GDD

| Area | Sections |
|---|---|
| Executive summary, research, vision, the world and tone | §1–§4 |
| IP and originality strategy (guardrails, blind tests, title clearance) | §5 |
| Combat system (frame data, block/parry, Breakaway, Peak, Getaways) | §6 |
| 18-Charm roster with full move sets | §7 |
| Wish system (postcards, Guaranteed Delivery, Finishes, odds and policy compliance) | §8 |
| Progression, modes, world events, maps and the seasonal calendar | §9–§11 |
| Art direction, animation/VFX, audio | §12–§14 |
| UX/onboarding, safety, maturity and accessibility | §15–§16 |
| Monetization, economy and business case | §17 |
| Technical design (Roblox Server Authority netcode, data, tooling, performance) | §18 |
| Live ops, KPIs, production plan, risks | §19–§22 |
| Appendices: data templates, glossary, lineage index, sources, open questions | A–F |

## Building it with an AI agent in Roblox Studio

The build runs on **your computer**, where Roblox Studio is: Studio's built-in MCP server only talks to local clients. Follow [`docs/agent/KICKOFF.md`](docs/agent/KICKOFF.md). In short:
1. Put the files in a **private** repository and open a terminal in it.
2. Run `claude` and paste the first message from the kickoff brief. The agent sets up the repository and walks you through the one-time human steps (build spec §1.2):
   - Studio's MCP server connected to Claude Code
   - the permissions file copied from `docs/agent/claude-settings.example.json`
   - Script Sync
   - the Server Authority properties
3. From then on the agent works through `docs/agent/TASKS.md`, logs progress in `docs/agent/PROGRESS.md`, and stops when it needs you (feel tests, device tests, publishing).

## Status

- **GDD v2.0** (16 October 2026): the original world, *Hullabrawl* and the Wishaway Isles.
- **Studio Build Spec v2.0** (16 October 2026): rewritten for the original world on top of the reviewed v1.0 engine spec. 159 task cards.

> The title *Hullabrawl* has had a first web search only. A full trademark knockout and a lawyer's clearance come before any public announcement (GDD §5, Appendix F).
