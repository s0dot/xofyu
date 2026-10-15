# Project ALGEBRAIC — *Algebraic!*

An **Adventure Time–themed battlegrounds fighting game for Roblox**, in the spirit of *Jujutsu Shenanigans*:
- you roll ("make Wishes") for **Fighting Styles** inspired by show characters, with seven rarities from Common to a secret Cosmic tier
- you brawl in hand-painted Land of Ooo arenas that transform every season

| Document | What it is |
|---|---|
| 📄 **[Game Design Document → `docs/GDD.md`](docs/GDD.md)** | The design: what the game is, every tuning number, the business case and risks |
| 🛠️ **[Studio Build Spec → `docs/STUDIO_BUILD_SPEC.md`](docs/STUDIO_BUILD_SPEC.md)** | How an AI coding agent builds it in Roblox Studio: contracts, architecture, combat netcode, style kits, meta systems, UI, tests, IP layer (Parts 0–8, in [`docs/build-spec/`](docs/build-spec/)) |
| ✅ **[Task plan → `docs/agent/TASKS.md`](docs/agent/TASKS.md)** | Ordered task cards (T000–T299) with acceptance checks, from session setup to the M1-G graybox vertical slice |
| 🤖 **[`AGENTS.md`](AGENTS.md)** / [`CLAUDE.md`](CLAUDE.md) | The agent's operating manual (read first by Claude Code) |

## What's in the GDD

| Area | Sections |
|---|---|
| Executive summary, research, vision and pillars | §1–§4 |
| **IP and licensing strategy** (owner's decision and publish gate in §5.8) | §5 |
| Combat system (frame data, block/parry, Breakaway, Awakening, Realms) | §6 |
| 18-style launch roster with full move kits and canon episode references | §7 |
| Wish (rolling) system, pity, Flairs, odds and policy compliance | §8 |
| Progression, modes, world events, maps and the seasonal calendar | §9–§11 |
| Art direction, animation/VFX, audio | §12–§14 |
| UX/onboarding, safety, maturity and accessibility | §15–§16 |
| Monetization, economy and business case | §17 |
| Technical design (Roblox Server Authority netcode, data, tooling, performance) | §18 |
| Live ops, KPIs, production plan, risks | §19–§22 |
| Appendices: data templates, glossary, episode index, sources, open questions | A–F |

## Building it with an AI agent in Roblox Studio

The build runs on **your computer**, where Roblox Studio is: Studio's built-in MCP server only talks to local clients.

1. Clone this repo and open a terminal in it.
2. Do the one-time setup in build spec **§1.2**. In short:
   - install Claude Code and the Rokit toolchain
   - in Studio, enable the MCP server and connect Claude Code (Assistant → ⋯ → Manage MCP Servers → Quick connect)
   - review `docs/agent/claude-settings.example.json` and copy it to `.claude/settings.json` (the agent's tool permissions; never committed)
   - turn on Script Sync for the four synced folders (§0.3)
3. Complete the human-only steps **H-01…H-12** (§0.7) as the task plan reaches them. These include:
   - setting `Workspace.AuthorityMode = Server`
   - creating and publishing the dev place
   - the feel and device gates
4. Run `claude` in the repo folder. It reads `AGENTS.md` and works through `docs/agent/TASKS.md`, logging progress in `docs/agent/PROGRESS.md`.

## Status

- **GDD v1.2** (15 October 2026). Adds:
  - the owner's IP decision and publish gate (§5.7–§5.8)
  - the optional Character Look toggle
  - 2026 Roblox platform corrections
- **Studio Build Spec v1.0** (15 October 2026): reviewed. A 213-finding technical review has been applied and every blocker and major issue verified as resolved. 159 task cards.

> ⚠️ *Adventure Time* is owned by Cartoon Network, now part of Skydance (Paramount + Warner Bros. Discovery). Development uses a Codename skin pack. A **public** Codename build is blocked by an informed-consent publish gate that only the owner can clear. A licensed or original-world launch remains the recommendation. See GDD §5 and build spec Part 8.
