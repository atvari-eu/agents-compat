# agents-compat

A CLI tool that generates agent-specific configurations from standardized formats and keeps them in sync.

## Status

**Alpha** — core functionality working, more agents and features planned.

## Motivation

Standards like [AGENTS.md](https://agents.md), [Agent Skills](https://agentskills.io), and [Agent Plugins](https://agent-plugins.org) provide portable ways to describe agent instructions and tooling. However, many AI coding agents still expect their own proprietary configuration files and directory layouts.

`agents-compat` uses **open standards as the source of truth** and generates/syncs everything else.

## Install

```bash
cargo install --path .
```

Or build from source:

```bash
cargo build --release
```

## Usage

```bash
# Discover all source configs in the current project
agents-compat scan

# Generate agent-specific files for all supported agents
agents-compat generate

# Generate only for specific agents
agents-compat generate --agent claude --agent gemini

# Regenerate only changed/outdated files
agents-compat sync

# Remove all generated files
agents-compat clean

# Show what's in sync vs. missing
agents-compat status
```

## Supported Standards

| Standard | Status |
|---|---|
| [AGENTS.md](https://agents.md) | Supported (scan + bridge) |
| [Agent Skills](https://agentskills.io) | Supported (symlink management) |
| [Agent Plugins](https://agent-plugins.org) (v1.0) | Supported (scan + skills) |

## Supported Agents

| Agent | Native AGENTS.md? | Rules Bridging | Skills Symlinks |
|---|---|---|---|
| Claude Code | No | `CLAUDE.md` with `@AGENTS.md` import | `.claude/skills/` |
| Gemini CLI | Configurable | `.gemini/settings.json` context config | `.gemini/skills/` |
| Cursor | Yes | — | `.cursor/skills/` |
| OpenCode | Yes | — | `.opencode/skills/` |
| Codex | Yes | — | `.agents/skills/` |

## How It Works

1. **Scan** the project for `AGENTS.md` files and Agent Plugin directories (`plugin.json`)
2. **Generate** agent-specific config files:
   - Claude Code: `CLAUDE.md` that imports `AGENTS.md`
   - Gemini CLI: `.gemini/settings.json` with context file references
   - Skills: symlinks from each agent's expected skills path to the canonical source
3. **Sync** regenerates only files that have changed
4. **Clean** removes all generated files

## Project Structure

```
src/
├── main.rs              # CLI entry point
├── lib.rs               # Library root
├── cli.rs               # Clap CLI definitions
├── error.rs             # Error types
├── scan.rs              # Config discovery with colored output
├── sync.rs              # File writing, symlinks, diff detection
├── standards/
│   ├── mod.rs           # SourceConfig, project root detection
│   ├── agents_md.rs     # AGENTS.md file discovery
│   └── agent_plugins.rs # Plugin/SKILL.md parsing
└── agents/
    ├── mod.rs           # Agent trait + skills_symlinks helper
    ├── claude.rs        # CLAUDE.md bridge + skills
    ├── gemini.rs        # settings.json + skills
    ├── cursor.rs        # Skills symlinks
    ├── opencode.rs      # Skills symlinks
    └── codex.rs         # Skills symlinks
```

## License

[MIT](LICENSE) © [atvari GmbH](https://atvari.eu)
