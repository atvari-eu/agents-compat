# Design Document: agents-compat

**Status:** Draft
**Created:** 2026-08-26

## Overview

`agents-compat` is a CLI tool that generates agent-specific configurations from open standards and keeps them in sync. It uses AGENTS.md, Agent Skills (SKILL.md), and Agent Plugins (plugin.json + mcp.json) as the source of truth, then generates native config files for each supported AI coding agent.

## Problem

Each AI coding agent uses its own proprietary file formats, directory layouts, and configuration schemas for rules, skills, MCP servers, and hooks/permissions. Teams either maintain parallel configs per agent, or adopt a tool-specific canonical format that locks them into one ecosystem.

## Design Principles

1. **Open standards as source of truth** — AGENTS.md, SKILL.md, plugin.json/mcp.json. No proprietary canonical format.
2. **Symlinks over copies** — Generated files symlink to canonical sources so updates propagate without re-running.
3. **Idempotent** — Running `generate` multiple times is safe; only writes when content differs.
4. **Non-destructive** — Only creates/updates files it generated. Never touches user-authored files.
5. **No config required** — Auto-discovers everything by scanning the repo and respecting standard discovery paths.

## Target Agents

| Agent | Rules | Skills | MCP Config | Hooks/Permissions |
|---|---|---|---|---|
| Claude Code | Needs bridge (`CLAUDE.md`) | `.claude/skills/` | `.mcp.json` | `settings.json` |
| Cursor | Native (reads `AGENTS.md`) | `.cursor/skills/` | `.cursor/mcp.json` | `.cursor/hooks.json` |
| Codex | Native (reads `AGENTS.md`) | `.agents/skills/` | `.codex/config.toml` | `config.toml`/`hooks.json` |
| Gemini CLI | Configurable bridge | `.gemini/skills/` | `.gemini/settings.json` | `settings.json` |
| OpenCode | Native (reads `AGENTS.md`) | `.opencode/skills/` | `opencode.json` | `opencode.json` |
| Windsurf | Native (reads `AGENTS.md`) | — | `~/.codeium/windsurf/mcp_config.json` | `.windsurf/hooks.json` |
| VS Code / Copilot | Native (reads `AGENTS.md`) | — | `.vscode/mcp.json` | — |
| Cline | — | — | VS Code ext storage | file-based `Hooks/` |

## Architecture

### Crate Structure

```
agents-compat/
├── Cargo.toml
├── build.rs
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── cli.rs                   # Clap parser
│   ├── error.rs                 # Error types + exit codes
│   ├── scan.rs                  # Source discovery orchestration
│   ├── standards/
│   │   ├── mod.rs
│   │   ├── agents_md.rs         # Find & parse AGENTS.md files
│   │   ├── agent_plugins.rs     # Parse plugin.json, skills/, mcp.json
│   │   └── mcp.rs               # MCP server config normalization
│   ├── agents/
│   │   ├── mod.rs               # Agent trait + registry
│   │   ├── claude.rs
│   │   ├── cursor.rs
│   │   ├── codex.rs
│   │   ├── gemini.rs
│   │   ├── opencode.rs
│   │   ├── windsurf.rs
│   │   └── copilot.rs
│   ├── sync.rs                  # Symlink/file management, diff detection
│   └── hooks.rs                 # Hook & permission translation
├── templates/                   # minijinja templates
│   ├── claude/
│   │   ├── CLAUDE.md.j2
│   │   └── settings.json.j2
│   ├── aider/
│   │   └── aider.conf.yml.j2
│   ├── gemini/
│   │   └── settings.json.j2
│   └── codex/
│       └── config.toml.j2
├── tests/
│   ├── integration/
│   └── fixtures/
```

### Dependencies

```toml
[dependencies]
clap = { version = "4", features = ["derive", "env"] }
anyhow = "1"
thiserror = "2"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
toml = "0.8"
walkdir = "2"
ignore = "0.4"
minijinja = "2"
colored = "2"

[dev-dependencies]
assert_cmd = "2"
predicates = "3"
tempfile = "3"
```

### CLI Interface

```
agents-compat scan              # Discover all source configs
agents-compat generate          # Generate agent-specific files
agents-compat sync              # Regenerate only changed/outdated files
agents-compat clean             # Remove all generated files
agents-compat status            # Show sync state
```

### Core Trait

```rust
trait Agent {
    fn name(&self) -> &str;
    fn detect(&self, dir: &Path) -> bool;
    fn generate(&self, source: &Source, target: &Path) -> Result<Vec<GeneratedFile>>;
    fn skill_dirs(&self) -> Vec<PathBuf>;
    fn is_native_agents_md(&self) -> bool;
}
```

## Use Case 1: Rules & Instructions Sync

### Source: AGENTS.md

AGENTS.md is freeform Markdown with no required fields. The tool scans the directory tree from cwd upward to the repo root, collecting all AGENTS.md files (nested files are directory-scoped in monorepos).

### Target: Claude Code

Claude Code does NOT read AGENTS.md natively. Generate a `CLAUDE.md` at project root:

```markdown
@AGENTS.md
```

This single import line makes Claude Code load the shared instructions. Also generate `.claude/settings.json` if MCP or permissions need bridging.

### Target: Gemini CLI

Gemini CLI defaults to `GEMINI.md`. Generate `.gemini/settings.json`:

```json
{
  "context": {
    "fileName": ["AGENTS.md", "GEMINI.md"]
  }
}
```

### Other Agents

Codex, Cursor, OpenCode, Windsurf, Copilot all read AGENTS.md natively — no bridging needed.

## Use Case 2: Skills Symlink Management

### Source: .agents/skills/ or Agent Plugins skills/

Skills discovered at:
- `.agents/skills/*/SKILL.md` (standard)
- `.opencode/skills/*/SKILL.md`
- `.claude/skills/*/SKILL.md`
- `skills/*/SKILL.md` (Agent Plugins)

### Target Mapping

| Agent | Project Skills Path | Global Skills Path |
|---|---|---|
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| Cursor | `.cursor/skills/` | — |
| Gemini CLI | `.gemini/skills/` | `~/.gemini/skills/` |
| OpenCode | `.opencode/skills/` | `~/.config/opencode/skills/` |
| Codex | `.agents/skills/` | `~/.agents/skills/` |

### Strategy

Symlink each agent's skills directory to the canonical `.agents/skills/` directory. For agents that already read from `.agents/skills/` (Codex, Gemini CLI, OpenCode via compat), no symlinking needed — they already work.

For Claude Code: symlink `.claude/skills/` → `.agents/skills/`.
For Cursor: symlink `.cursor/skills/` → `.agents/skills/`.

## Use Case 3: MCP Server Config Normalization

### Source: Agent Plugins mcp.json

The Agent Plugins spec defines a portable MCP config:

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/mcp.schema.json",
  "mcpServers": {
    "server-name": {
      "type": "stdio",
      "command": "./bin/validator",
      "args": ["--data"],
      "env": { "API_KEY": "value" }
    }
  }
}
```

### Translation Challenges

| Field | Agent Plugins | Claude Code | OpenCode | Codex | VS Code | Gemini | Windsurf |
|---|---|---|---|---|---|---|---|
| Root key | `mcpServers` | `mcpServers` | `mcp.servers` | `[mcp_servers.X]` | `servers` | `mcpServers` | `mcpServers` |
| Format | JSON | JSON | JSON/JSONC | TOML | JSON | JSON | JSON |
| Stdio type | `stdio` | `stdio` | `local` | implicit | `stdio` | implicit | auto |
| HTTP type | `streamable-http` | `http` | `remote` | implicit | `http` | `httpUrl` field | auto |
| URL field | `url` | `url` | `url` | `url` | `url` | `httpUrl` | `serverUrl` |
| Command | string | string | **array** | string | string | string | string |
| Env field | `env` | `env` | `environment` | `env` | `env` | `env` | `env` |

### Translation Rules

1. **Root key remapping**: `mcpServers` → `mcp.servers` (OpenCode), `servers` (VS Code), `[mcp_servers.X]` (Codex/TOML)
2. **Type value mapping**: `stdio` → `local` (OpenCode), implicit (Codex/Gemini/Windsurf)
3. **Command format**: Split `command` string + `args` array into separate fields (all except OpenCode which takes `["cmd", "arg1"]`)
4. **URL field rename**: `url` → `serverUrl` (Windsurf), `httpUrl` (Gemini for streamable HTTP)
5. **Env var syntax**: `${VAR}` (Claude, Windsurf, Cline), `{env:VAR}` (OpenCode), `$VAR` (Gemini)
6. **Format conversion**: JSON → TOML for Codex

## Use Case 4: Hook & Permission Portability

### Source

Hooks and permissions are defined in a unified format (TBD — possibly YAML or JSON with event name normalization).

### Event Name Mapping

| Unified Event | Claude Code | Cursor | Codex | Gemini | Windsurf | Cline |
|---|---|---|---|---|---|---|
| `pre_tool_use` | `PreToolUse` | `preToolUse` | `PreToolUse` | `BeforeTool` | `pre_write_code`/`pre_run_command` | `PreToolUse` |
| `post_tool_use` | `PostToolUse` | `postToolUse` | `PostToolUse` | `AfterTool` | `post_write_code`/`post_run_command` | `PostToolUse` |
| `session_start` | `SessionStart` | `sessionStart` | `SessionStart` | `SessionStart` | — | `TaskStart` |
| `user_prompt_submit` | `UserPromptSubmit` | `beforeSubmitPrompt` | `UserPromptSubmit` | `UserPromptSubmit` | `user_prompt_submit` | `UserPromptSubmit` |
| `stop` | `Stop` | `stop` | `Stop` | `Stop` | — | `TaskComplete` |
| `pre_compact` | `PreCompact` | `preCompact` | `PreCompact` | `PreCompress` | — | `PreCompact` |

### Blocking Semantics

| Agent | Block Signal | Allow Signal |
|---|---|---|
| Claude Code | Exit code 2 + JSON | Exit code 0 + JSON |
| Cursor | `{"permission": "deny"}` | `{"permission": "allow"}` |
| Codex | Exit code 2 (deny only) | N/A (can't grant via hooks) |
| Gemini CLI | Exit code 2 | Exit code 0 |
| Windsurf | Exit code 2 | N/A |
| Cline | `{"cancel": true}` | `{"cancel": false}` |

### Translation Strategy

1. **Event mapping table** (static, above) — translates event names per agent
2. **Handler format generation** — each agent's hook handler format is different:
   - Claude Code/Codex: `{ "command": "path/to/script" }` or `{ "http": { "url": "..." } }` or `{ "prompt": "..." }`
   - Cursor: `{ "command": "bash -c '...'" }` or `{ "prompt": "..." }`
   - Gemini: `{ "command": "bash -c '...'" }`
   - Windsurf: `{ "command": "bash -c '...'" }`
   - Cline: executable files named `PreToolUse`, `PostToolUse`, etc.
3. **Blocking wrapper** — wrap the user's hook script in a per-agent wrapper that translates exit codes and JSON output formats
4. **Matcher translation** — regex patterns may need adjustment per agent's matcher syntax

### Declarative Permissions

For agents that support declarative permissions (Claude Code, OpenCode, Codex, Gemini CLI), generate allow/deny/ask rules from the unified definition. For hook-only agents (Cursor, Windsurf), generate hook scripts that enforce the same semantics.

## Implementation Phases

### Phase 1: Core Infrastructure + Rules Sync
- Cargo project, CLI, error types, agent trait
- AGENTS.md scanning (walk tree, collect all files)
- Claude Code bridge (`CLAUDE.md` generation)
- Gemini CLI bridge (`.gemini/settings.json`)
- `scan`, `generate`, `status`, `clean` commands

### Phase 2: Skills Symlink Management
- Skill discovery across standard paths
- Symlink creation/update/removal per agent
- Conflict detection (skill name collisions)

### Phase 3: MCP Config Normalization
- Agent Plugins `mcp.json` parsing
- Translation engine for each agent's format
- TOML generation for Codex
- Env var syntax normalization

### Phase 4: Hooks & Permissions
- Unified hook definition format
- Event name mapping per agent
- Handler format generation per agent
- Blocking semantics wrapper generation
- Declarative permission generation

### Phase 5: Polish
- Watch mode (notify crate)
- Shell completions (clap_complete)
- Colored output (colored crate)
- .gitignore-aware scanning (ignore crate)

## Comparison with Existing Tools

| Feature | agents-compat | rulesync | AgentsMesh |
|---|---|---|---|
| Canonical format | Open standards | `.rulesync/` | `.agentsmesh/` |
| Rules sync | Planned | 30+ targets | 15+ targets |
| Skills symlinking | Planned | Yes | Yes |
| MCP normalization | Planned | Partial | Partial |
| Hooks/permissions | Planned | Yes | Yes |
| Open standard source | Yes | No | No |
| Drift detection | Planned | `--check` | check + lockfile |
| Plugin system | — | — | Yes |
