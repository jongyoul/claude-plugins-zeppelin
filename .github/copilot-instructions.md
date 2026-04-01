# Copilot Instructions

## For Coding Agent (Issue → PR)

### Rules

- **Your PR description MUST start with `Closes #N` where N is the issue number you are working on. This is mandatory — the issue must be auto-closed when the PR is merged.**
- Only modify files under `zeppelin-dev/skills/` and `zeppelin-dev/agents/`
- Do NOT modify `.claude-plugin/plugin.json`, `README.md`, `.github/`, or `.gitignore`
- Do NOT create new files — only update existing files
- Do NOT delete existing files
- Preserve YAML frontmatter (name, description, tools) in SKILL.md and agent files — only update the body content
- Keep each file focused on its original topic — do not merge or split files
- Do not make cosmetic-only changes (whitespace, formatting) unless the content itself changed
- If AGENTS.md changes are minor, make minimal corresponding updates — do not rewrite entire files

### PR Description Requirements

When creating a PR, the description MUST include:

1. **AGENTS.md diff summary** — Which sections of AGENTS.md changed
2. **Mapping** — For each changed section, which plugin file was updated and how
3. **No-change justification** — If a section changed but no plugin file was updated, explain why

Example format:

```
## AGENTS.md Changes

- Section "Build & Test": Added new Maven profile `-Pspark-4.0`
- Section "Contributing Guide": Updated JDK requirement from 11 to 17

## Plugin File Updates

- `zeppelin-dev/skills/build/SKILL.md`: Added `-Pspark-4.0` to Maven profiles table
- `zeppelin-dev/skills/contribute/SKILL.md`: Updated JDK version in prerequisites table

## Unchanged Files

- `zeppelin-dev/agents/architecture-guide.md`: No architecture changes in this update
```

## For Code Review

When reviewing a PR created by the coding agent:

1. Verify that changes correspond ONLY to the AGENTS.md diff provided in the issue
2. Verify no unrelated content was added, removed, or rewritten
3. Verify YAML frontmatter is unchanged
4. Verify no files outside `zeppelin-dev/` were modified
5. Verify the PR description accurately maps AGENTS.md changes to file updates
6. Flag if the coding agent made changes that go beyond what the AGENTS.md diff requires
