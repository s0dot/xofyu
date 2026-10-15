@AGENTS.md

# Claude Code specifics

- **MCP server:** `Roblox_Studio` (Studio's built-in server, stdio). Its tools appear to you as
  `mcp__Roblox_Studio__<tool>` (e.g. `mcp__Roblox_Studio__execute_luau`). If the client defers MCP tool schemas,
  load all `mcp__Roblox_Studio__*` schemas before preflight. If no such tools exist, MCP is not connected:
  write a `tool-failure` blocker pointing the human to build spec §1.2.3; do not try to work around it.
- **Permissions:** The human installs `.claude/settings.json` from `docs/agent/claude-settings.example.json`
  (build spec §1.2.4; never committed); it allowlists the read/test MCP tools and the host commands
  (`stylua`, `selene`, `luau-lsp analyze`, `lune run`, `rojo sourcemap/build`, safe `git` subcommands).
  `multi_edit`, `insert_asset`, `generate_*`, `store_image`, `subagent`, `rokit`, `git push`, `git restore`,
  `git stash pop` and `lune run tools/upload_anims.luau` ask first; `upload_image`, history-rewriting git commands and Edit/Write on the human-signed files
  (`docs/agent/PUBLISH_ACK.md`, `LICENSE_CONFIRMATION.md`, `TITLE_CLEARANCE.md`) are denied. Never create or edit
  `.claude/settings*.json`; to change permissions, propose a diff to the example in a blocker.
- **New sessions after setup:** MCP servers added and Rokit `PATH` changes made after a session started are not
  visible to it. After H-02 / the Rokit install (and after T000 if `CLAUDE.md` was missing at session start), the
  human restarts Claude Code in the repo root and you resume from `docs/agent/PROGRESS.md`.
- **Drive playtests yourself.** Start, poll, assert, read logs and stop through MCP (build spec §1.4.5). The
  human plays only for feel (H-10) and device (H-12) gates.
- **Keep `ARCHITECTURE.md` current in the same turn** as any file or instance-folder change, and update the
  task row in `docs/agent/PROGRESS.md` with evidence lines before committing.
- **Skill:** `.claude/skills/algebraic-studio/SKILL.md` holds the conventions in compact form; load it whenever
  you edit code, run builders, or touch Studio.
- **Subagents:** do not run parallel Claude Code subagents that edit files or call `Roblox_Studio` at the same
  time; one writer per checkout and per Studio window. Read-only research subagents are fine.
- **Context hygiene:** start a new session at each phase boundary or when context runs long; preflight
  (AGENTS.md) restores state from `ARCHITECTURE.md` and `PROGRESS.md`. Parked work is in `git stash list`.
- **Windows:** run Claude Code natively (PowerShell or the VS Code extension), not inside WSL, so the MCP server
  command (`cmd.exe /c %LOCALAPPDATA%\Roblox\mcp.bat`) and Script Sync file watching work.
