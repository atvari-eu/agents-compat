# agents-compat

A CLI tool that generates agent-specific configurations from standardized formats and keeps them in sync.

## Status

**Work in progress** — no usable code yet.

## Motivation

Standards like [AGENTS.md](https://agents.md) and [Agent Plugins](https://agent-plugins.org) provide portable ways to describe agent instructions and tooling. However, many AI coding agents still require their own proprietary configuration files and directory layouts.

`agents-compat` bridges this gap: define your agent configuration once using open standards, and let the tool generate and synchronize the right files for each supported agent.

## Planned Features

- Generate agent-specific config from AGENTS.md and Agent Plugins
- Sync configurations across multiple agents automatically
- Support a broad range of AI coding agents, especially those without native standard support
- Watch mode for automatic re-generation on changes

## Supported Standards

| Standard | Status |
|---|---|
| [AGENTS.md](https://agents.md) | Planned |
| [Agent Plugins](https://agent-plugins.org) (v1.0) | Planned |

## License

[MIT](LICENSE) © [atvari GmbH](https://atvari.eu)
