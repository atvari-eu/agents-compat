# agents-compat

A CLI tool that generates agent-specific configurations from standardized formats and keeps them in sync.

## Status

**Work in progress** — no usable code yet.

## Motivation

Standards like [AGENTS.md](https://agents.md), [Agent Skills](https://agentskills.io), and [Agent Plugins](https://agent-plugins.org) provide portable ways to describe agent instructions and tooling. However, each AI coding agent still expects its own proprietary file formats, directory layouts, and configuration schemas.

The result: teams either maintain parallel config files for each agent, or adopt a tool-specific canonical format that locks them into one ecosystem.

`agents-compat` uses **open standards as the source of truth** and generates/syncs everything else.

## Use Cases

### 1. Rules & Instructions Sync

Generate agent-specific instruction files from a single `AGENTS.md`:

| Source | Target | Generated File |
|---|---|---|
| `AGENTS.md` | Claude Code | `CLAUDE.md` (with `@AGENTS.md` import) |
| `AGENTS.md` | Aider | `.aider.conf.yml` (`read: AGENTS.md`) |
| `AGENTS.md` | Gemini CLI | `.gemini/settings.json` (`context.fileName`) |

Most agents (Codex, Cursor, OpenCode, Windsurf, Copilot) already read `AGENTS.md` natively and need no bridging.

### 2. Skills Symlink Management

Install portable [Agent Skills](https://agentskills.io) (`SKILL.md` files) into each agent's expected directory:

| Agent | Project Skills Path | Global Skills Path |
|---|---|---|
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| Cursor | `.cursor/skills/` | — |
| Gemini CLI | `.gemini/skills/` | `~/.gemini/skills/` |
| OpenCode | `.opencode/skills/` | `~/.config/opencode/skills/` |
| Codex | `.agents/skills/` | `~/.agents/skills/` |

Skills are symlinked, not copied, so updates propagate automatically.

### 3. MCP Server Config Normalization

Generate MCP server configurations from [Agent Plugins](https://agent-plugins.org) `mcp.json` into each agent's native format:

| Agent | Config File | Root Key | Remote URL Field | Type Values |
|---|---|---|---|---|
| Claude Code | `.mcp.json` | `mcpServers` | `url` | `stdio`, `http`, `sse` |
| Cursor | `.cursor/mcp.json` | `mcpServers` | `url` | auto-detect |
| VS Code / Copilot | `.vscode/mcp.json` | `servers` | `url` | `stdio`, `http`, `sse` |
| OpenCode | `opencode.json` | `mcp.servers` | `url` | `local`, `remote` |
| Codex | `.codex/config.toml` | `[mcp_servers.X]` | `url` | implicit from `command`/`url` |
| Gemini CLI | `.gemini/settings.json` | `mcpServers` | `httpUrl`/`url` | command/httpUrl/url |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` | `mcpServers` | `serverUrl` | auto-detect |

Each agent uses different field names, transport type values, env var syntax, and command formats. `agents-compat` handles the translation.

### 4. Hook & Permission Portability

Generate agent-specific hook configurations from a unified definition:

| Agent | Hooks Support | Config Format | Permission Model |
|---|---|---|---|
| Claude Code | 21+ events | `settings.json` | declarative allow/deny/ask |
| Cursor | 20+ events | `.cursor/hooks.json` | hooks-only |
| Codex | 10 events | `config.toml` / `hooks.json` | hooks + approval policy |
| Gemini CLI | 12+ events | `settings.json` | declarative allow/deny/ask |
| Windsurf | 12+ events | `hooks.json` | exit-code only |
| OpenCode | none | `opencode.json` | declarative allow/deny/ask |
| Cline | 8 events | file-based (`Hooks/PreToolUse`) | UI toggles |

## Supported Standards

| Standard | Status |
|---|---|
| [AGENTS.md](https://agents.md) | Planned |
| [Agent Skills](https://agentskills.io) | Planned |
| [Agent Plugins](https://agent-plugins.org) (v1.0) | Planned |

## Supported Agents

| Agent | Rules | Skills | MCP Config | Hooks/Permissions |
|---|---|---|---|---|
| Claude Code | Planned | Planned | Planned | Planned |
| Cursor | Native | Planned | Planned | Planned |
| Codex | Native | Planned | Planned | Planned |
| Gemini CLI | Planned | Planned | Planned | Planned |
| OpenCode | Native | Planned | Planned | Planned |
| Windsurf | Native | — | Planned | Planned |
| VS Code / Copilot | Native | — | Planned | — |
| Cline | — | — | — | Planned |

## Comparison with Existing Tools

This project is inspired by and aims to improve upon existing sync tools.

### rulesync

[rulesync](https://github.com/dyoshikawa/rulesync) uses its own `.rulesync/` canonical format and generates native configs for 30+ tools. Supports a `convert` command for direct format-to-format conversion without a canonical source.

### AgentsMesh

[AgentsMesh](https://github.com/agentsmesh/agentsmesh) (2.3K stars) uses `.agentsmesh/` as its canonical format. Offers bidirectional import/generate, `diff` preview, `merge` after git conflicts, and a plugin system for custom targets.

### Feature Comparison

| Feature | agents-compat | rulesync | AgentsMesh |
|---|---|---|---|
| **Canonical format** | Open standards (AGENTS.md, Agent Plugins) | `.rulesync/` (proprietary) | `.agentsmesh/` (proprietary) |
| **Rules/instructions sync** | Planned | ✅ 30+ targets | ✅ 15+ targets |
| **Skills symlinking** | Planned | ✅ | ✅ |
| **MCP config normalization** | Planned | Partial | Partial |
| **Hooks/permissions** | Planned | ✅ | ✅ |
| **Import from existing setups** | — | ✅ `convert` | ✅ bidirectional |
| **Drift detection** | Planned | `--check` | `check` + lockfile |
| **Watch mode** | Planned | — | — |
| **Monorepo support** | Planned | Multiple `outputRoots` | `extends` chain |
| **Plugin/extension system** | — | — | ✅ |
| **Open standard source** | ✅ | ❌ | ❌ |

### Key Differentiator

Existing tools require adopting a **proprietary canonical format** (`.rulesync/`, `.agentsmesh/`). `agents-compat` instead uses **open standards themselves** as the source of truth — `AGENTS.md` for instructions, `SKILL.md` for skills, `plugin.json` + `mcp.json` for tooling. This means:

- No new format to learn or migrate to
- Configs are portable even without `agents-compat` installed
- Aligns with the [Agentic AI Foundation](https://aaif.io) ecosystem

## License

[MIT](LICENSE) © [atvari GmbH](https://atvari.eu)
