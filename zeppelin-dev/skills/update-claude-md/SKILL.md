---
name: update-claude-md
description: Sync AGENTS.md content into the project CLAUDE.md file for this plugin's rules and guidelines
tools:
  - Read
  - Edit
  - Write
  - Bash
---

# Update CLAUDE.md

Fetch the latest AGENTS.md from the Apache Zeppelin repository and generate a minimal CLAUDE.md for the project.

## Steps

1. Read the current AGENTS.md from the project root (or fetch from upstream if not present):
   ```bash
   curl -s https://raw.githubusercontent.com/apache/zeppelin/master/AGENTS.md -o /tmp/AGENTS.md.upstream
   ```

2. Read the existing CLAUDE.md if it exists.

3. Generate a minimal CLAUDE.md with only:
   - Project overview (language, version, build tool)
   - Basic build/test commands
   - A pointer to the plugin for detailed guides

4. Write to `CLAUDE.md` at the project root. If the file already exists, only update the `# Zeppelin Dev Plugin` section — do not touch other sections.

## CLAUDE.md Template

```markdown
# CLAUDE.md

## Project Overview

Apache Zeppelin — web-based notebook for interactive data analytics.
- **Language**: Java, Scala — versions in root `pom.xml` (`java.version`, `scala.binary.version`)
- **Build**: Maven multi-module (wrapper: `./mvnw`)
- **Frontend**: Angular + TypeScript in `zeppelin-web-angular/` — versions in its `package.json`
- **Version**: see `<version>` in root `pom.xml`

## Quick Reference

- Full build: `./mvnw clean package -DskipTests`
- Single module: `./mvnw clean package -pl <module> -DskipTests`
- Run tests: `./mvnw test -pl <module>`
- Single test: `./mvnw test -pl <module> -Dtest=ClassName#method`
- License check: `./mvnw clean org.apache.rat:apache-rat-plugin:check -Prat`
- Frontend dev: `cd zeppelin-web-angular && npm start`

For detailed build guides, architecture reference, and contribution workflow,
use the zeppelin-dev plugin skills: /build, /contribute, /frontend
```

## Notes

- Keep CLAUDE.md as small as possible — detailed info lives in plugin skills
- Do not duplicate AGENTS.md content verbatim into CLAUDE.md
- Preserve any user-added sections in the existing CLAUDE.md
