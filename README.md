# claude-plugins-zeppelin

Claude Code plugin marketplace for Apache Zeppelin development.

## Plugins

### zeppelin-dev

Development tools for Apache Zeppelin contributors.

**Skills:**
- `/build` — Build & test guide (shaded JAR chain, Maven profiles, module order)
- `/contribute` — Contribution workflow (PR checklist, code style, license checks)
- `/frontend` — Frontend development (Angular, proxy, E2E tests)
- `/update-claude-md` — Sync AGENTS.md into project CLAUDE.md

**Agents:**
- `architecture-guide` — Module architecture, dependency flow, server-interpreter communication
- `code-reviewer` — Code review against Zeppelin conventions

## Installation

```bash
# Add marketplace
/plugin marketplace add jongyoul/claude-plugins-zeppelin

# Install plugin
/plugin install zeppelin-dev@jongyoul/claude-plugins-zeppelin
```

## Auto-sync

A GitHub Actions workflow runs weekly to check if `AGENTS.md` in `apache/zeppelin` has changed, and creates a PR when updates are needed.

## License

Apache License 2.0
