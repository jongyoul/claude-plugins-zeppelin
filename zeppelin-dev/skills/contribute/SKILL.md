---
name: contribute
description: Apache Zeppelin contribution guide for PR submission, code style, license checks, and REST API patterns
tools:
  - Read
  - Edit
  - Write
  - Bash
  - Grep
  - Glob
---

# Zeppelin Contribution Guide

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| JDK | pinned in `pom.xml` (`java.version`) | Required — use exactly that major, not a newer/older JDK |
| Maven | provided by `./mvnw` (pinned in `.mvn/wrapper/maven-wrapper.properties`) | No separate install needed |
| Node.js | see `zeppelin-web-angular/package.json` (`engines.node`) | Only for frontend (`zeppelin-web-angular/`) |

## Initial Setup

```bash
# Clone the repository
git clone https://github.com/apache/zeppelin.git
cd zeppelin

# First build — skip tests to verify environment works
./mvnw clean package -DskipTests
# This takes ~10 minutes. If it succeeds, your environment is ready.

# Frontend setup (only if working on UI)
cd zeppelin-web-angular
npm install
cd ..
```

## Development Workflow

```bash
# Create a worktree for your feature branch
git worktree add ../zeppelin-ZEPPELIN-XXXX -b ZEPPELIN-XXXX-description
cd ../zeppelin-ZEPPELIN-XXXX

# When done, clean up
git worktree remove ../zeppelin-ZEPPELIN-XXXX

# Build only the module you're changing (--am builds required upstream modules)
./mvnw clean package -pl zeppelin-server --am -DskipTests

# Run tests for your module
./mvnw test -pl zeppelin-server --am

# Run a specific test
./mvnw test -pl zeppelin-server --am -Dtest=NotebookServerTest#testMethod

# Start the dev frontend (proxies API to localhost:8080)
cd zeppelin-web-angular && npm start
```

For Spark or Flink work, add the version profile:
```bash
./mvnw clean package -pl spark -Pspark-3.5 -Pspark-scala-2.12 -DskipTests
```

## Before Submitting a PR

1. **Write unit tests**. Every code change must include corresponding unit tests. Bug fixes should include a test that reproduces the bug. New features should have tests covering the main paths.

2. **Run tests for affected modules**:
   ```bash
   ./mvnw test -pl <your-module>
   ```

3. **Check license headers** — all new files must have the Apache License 2.0 header:
   ```bash
   ./mvnw clean org.apache.rat:apache-rat-plugin:check -Prat
   ```

4. **Lint frontend changes** (if applicable):
   ```bash
   cd zeppelin-web-angular && npm run lint:fix
   ```

5. **Create a JIRA issue** at [issues.apache.org/jira/browse/ZEPPELIN](https://issues.apache.org/jira/browse/ZEPPELIN) and use the issue number in PR title: `[ZEPPELIN-XXXX] description`.

## Code Style

- **Java**: Google Java Style (2-space indent). Checkstyle enforced — no tabs, LF line endings, newline at EOF
- **Frontend**: ESLint + Prettier, auto-enforced via pre-commit hook (Husky + lint-staged)
- **Testing**: JUnit 5 (Jupiter) + Mockito (Java; a small number of legacy JUnit 4 tests still exist), Playwright (frontend E2E)
- **Logging**: SLF4J + Log4j2
- **License**: Apache License 2.0 — all new files need the ASF header

## REST API Pattern

All REST endpoints follow this pattern:

```java
@Path("/notebook")
@Produces("application/json")
@Singleton
public class NotebookRestApi extends AbstractRestApi {
    @Inject
    public NotebookRestApi(Notebook notebook, ...) {
        super(authenticationService);
    }

    @GET
    @Path("/{noteId}")
    @ZeppelinApi
    public Response getNote(@PathParam("noteId") String noteId) {
        // Authorization check
        checkIfUserCanRead(noteId, "Insufficient privileges");
        // Business logic via service layer
        Note note = notebook.getNote(noteId);
        // Return JsonResponse
        return new JsonResponse<>(Status.OK, "", note).build();
    }
}
```

Key conventions:
- Extend `AbstractRestApi` (provides `getServiceContext()` for auth)
- Use `@Inject` constructor for HK2 DI
- Annotate public methods with `@ZeppelinApi`
- Return `JsonResponse<T>(status, message, body).build()`
- Authorization via `checkIfUserCan{Read|Write|Run}()`

## Module Boundaries — Where New Code Goes

| If the code... | Put it in |
|----------------|-----------|
| Is a base interface/class that all interpreters need | `zeppelin-interpreter` |
| Handles notebook state, interpreter lifecycle, scheduling, search, REST/WebSocket, or authentication realm | `zeppelin-server` |
| Is specific to one backend (Spark, Flink, JDBC, etc.) | That interpreter's module |
| Is a new way to launch interpreter processes | `zeppelin-plugins/launcher/` |
| Is a new notebook storage backend | `zeppelin-plugins/notebookrepo/` |

**Important**: Code added to `zeppelin-interpreter` is exposed to **every interpreter process** via the shaded JAR. Only add code there if all interpreters genuinely need it.
