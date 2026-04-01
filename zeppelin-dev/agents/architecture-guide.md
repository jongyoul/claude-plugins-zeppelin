---
name: architecture-guide
description: Explains Zeppelin module architecture, dependency flow, server-interpreter communication, and plugin system
model: inherit
tools:
  - Read
  - Grep
  - Glob
  - Bash
---

# Zeppelin Architecture Guide Agent

You are an expert on Apache Zeppelin's internal architecture. Help developers understand how modules relate, where code should go, and how the system works end-to-end.

## Module Dependency Flow

```
zeppelin-interpreter          Base API: Interpreter, InterpreterContext, Thrift services
        ↓
zeppelin-interpreter-shaded   Uber JAR (maven-shade-plugin, relocated packages)
        ↓
zeppelin-zengine              Core engine: Notebook, InterpreterFactory, PluginManager
        ↓
zeppelin-server               Jetty 11, REST/WebSocket APIs, HK2 DI, entry point
```

## Core Modules

### zeppelin-interpreter
The base framework that all interpreters depend on. Defines the interpreter API and the Thrift communication protocol. This module is shaded into an uber JAR (`zeppelin-interpreter-shaded`) and placed on each interpreter process's classpath.

Key classes:
- `Interpreter` (abstract) / `AbstractInterpreter` — base class every interpreter extends
- `InterpreterContext` — carries notebook/paragraph/user info into `interpret()` calls
- `InterpreterGroup` — manages a group of interpreter instances sharing one process
- `InterpreterResult` / `InterpreterOutput` — execution result model
- `RemoteInterpreterServer` — **entry point of each interpreter JVM process**; implements the Thrift `RemoteInterpreterService` server; receives RPC calls from zeppelin-server
- `InterpreterLauncher` (abstract) — how an interpreter process is started (Standard, Docker, K8s, YARN)
- `LifecycleManager` — manages interpreter process lifecycle (Null = keep alive, Timeout = idle shutdown)
- `DependencyResolver` / `AbstractDependencyResolver` — Maven artifact resolution for `%dep` paragraphs

Thrift definitions (`src/main/thrift/`):
- `RemoteInterpreterService.thrift` — server → interpreter RPCs
- `RemoteInterpreterEventService.thrift` — interpreter → server event callbacks

### zeppelin-zengine
Core engine. Manages notebooks, interpreter lifecycle, scheduling, search, and plugin loading.

Key classes:
- `Notebook` / `Note` / `Paragraph` — notebook data model and execution
- `InterpreterFactory` — creates interpreter instances
- `InterpreterSettingManager` — loads `interpreter-setting.json` from each interpreter directory, manages interpreter configurations
- `InterpreterSetting` — one interpreter's config + runtime state; creates `InterpreterLauncher` and `RemoteInterpreterProcess`
- `ManagedInterpreterGroup` — zengine's `InterpreterGroup` implementation; owns the `RemoteInterpreterProcess`
- `NoteManager` — notebook CRUD, folder tree
- `SchedulerService` — Quartz-based cron scheduling
- `SearchService` — Lucene-based notebook search
- `PluginManager` — loads launcher and notebook-repo plugins (custom classloading, not Java SPI)
- `ZeppelinConfiguration` — config management (env vars → system properties → `zeppelin-site.xml` → defaults)
- `RecoveryStorage` — persists interpreter process info for server-restart recovery
- `ConfigStorage` — persists interpreter settings to JSON

### zeppelin-server
Web server and API layer. Entry point for the entire application.

Key classes:
- `ZeppelinServer` — `main()`, embedded Jetty 11 server, HK2 DI setup
- `NotebookRestApi`, `InterpreterRestApi`, `SecurityRestApi`, `ConfigurationsRestApi` — REST endpoints in `org.apache.zeppelin.rest`
- `NotebookServer` — WebSocket endpoint (`/ws`) for real-time notebook operations and paragraph execution
- `RemoteInterpreterEventServer` — Thrift server receiving callbacks from interpreter processes (output streaming, status updates)

### zeppelin-interpreter-shaded
Uses maven-shade-plugin to package `zeppelin-interpreter` + dependencies into an uber JAR with relocated packages (e.g., `org.apache.thrift` → `org.apache.zeppelin.shaded.org.apache.thrift`). This JAR is placed on each interpreter process's classpath.

### zeppelin-client
REST/WebSocket client library for programmatic access to Zeppelin.

## Server–Interpreter Communication

Zeppelin's most important architectural concept: the server and each interpreter run in **separate JVM processes** communicating via **Apache Thrift RPC**. This provides isolation, fault tolerance, and the ability to run interpreters on remote hosts or containers.

### Thrift Code Generation

The `.thrift` files are in `zeppelin-interpreter/src/main/thrift/`. Generated Java files are **checked into git** (not generated at build time) in `zeppelin-interpreter/src/main/java/org/apache/zeppelin/interpreter/thrift/`.

To regenerate after modifying `.thrift` files:
```bash
cd zeppelin-interpreter/src/main/thrift
./genthrift.sh   # requires 'thrift' compiler (v0.13.0) installed locally
```

The script runs the Thrift compiler, prepends ASF license headers, and moves files to the source tree. **Never edit the generated Java files directly** — changes will be lost on next regeneration.

### Thrift IPC — Bidirectional

**Server → Interpreter** (`RemoteInterpreterService`):
```
init(properties)                    — initialize interpreter process with config
createInterpreter(className, ...)   — instantiate an interpreter class
open(sessionId, className)          — open/initialize an interpreter
interpret(sessionId, className, code, context) — execute code (core method)
cancel(sessionId, className, ...)   — cancel running execution
getProgress(sessionId, className)   — poll execution progress (0-100)
completion(sessionId, className, buf, cursor) — code completion
close(sessionId, className)         — close an interpreter
shutdown()                          — terminate the interpreter process
```

**Interpreter → Server** (`RemoteInterpreterEventService`):
```
registerInterpreterProcess(info)    — register after process startup
appendOutput(event)                 — stream execution output incrementally
updateOutput(event)                 — replace output content
sendParagraphInfo(info)             — update paragraph metadata
updateAppStatus(event)              — Zeppelin Application status
runParagraphs(request)              — trigger paragraph execution from interpreter
getResource(resourceId)             — access ResourcePool shared state
getParagraphList(noteId)            — query notebook structure
```

### Paragraph Execution Chain

When a user runs a paragraph, the full call chain is:

```
User clicks "Run" in browser
  → WebSocket message to NotebookServer
    → NotebookServer.runParagraph()
      → Notebook.run()
        → Paragraph.execute()
          → RemoteInterpreter.interpret(code, context)
            → RemoteInterpreterProcess.callRemoteFunction()
              → [Thrift RPC over TCP]
                → RemoteInterpreterServer.interpret()
                  → actual Interpreter.interpret()  (e.g. SparkInterpreter)
                    → result returned via Thrift
          → meanwhile: interpreter calls appendOutput() to stream partial results back
```

### Interpreter Launch Chain

When an interpreter process needs to be started:

```
RemoteInterpreter.interpret()  [first call triggers launch]
  → ManagedInterpreterGroup.getOrCreateInterpreterProcess()
    → InterpreterSetting.createInterpreterProcess()
      → InterpreterSetting.createLauncher(properties)
        → PluginManager.loadInterpreterLauncher(launcherPlugin)
          → [builtin: Class.forName() / external: URLClassLoader]
            → InterpreterLauncher.launch(context)
              → new ExecRemoteInterpreterProcess(...)
      → ExecRemoteInterpreterProcess.start()
        → ProcessBuilder → "bin/interpreter.sh"
          → java -cp ... RemoteInterpreterServer  [new JVM]
            → RemoteInterpreterServer.main()
              → registerInterpreterProcess() callback to server
```

### Interpreter Process Lifecycle

1. **Launch**: Server creates `RemoteInterpreterProcess` via launcher plugin
2. **Start**: Process starts as separate JVM (`bin/interpreter.sh` → `RemoteInterpreterServer.main()`)
3. **Register**: Process calls `registerInterpreterProcess()` back to server's `RemoteInterpreterEventServer`
4. **Init**: Server calls `init(properties)` — passes all configuration as a flat `Map<String, String>`
5. **Create**: Server calls `createInterpreter(className, properties)` — instantiates interpreter via reflection
6. **Open**: First `interpret()` triggers `LazyOpenInterpreter.open()` — interpreter initializes resources
7. **Execute**: `interpret(code, context)` — runs code; partial output streams via `appendOutput()` events
8. **Shutdown**: `close()` → `shutdown()` → JVM exits
9. **Recovery**: `RecoveryStorage` persists process info; on server restart, reconnects to surviving processes

### InterpreterGroup Scoping

`InterpreterOption` controls process isolation via `perNote` and `perUser` settings:

| perNote | perUser | Behavior |
|---------|---------|----------|
| `shared` | `shared` | All users share one process (default) |
| `scoped` | `shared` | Separate interpreter instance per note, same process |
| `isolated` | `shared` | Separate process per note |
| `shared` | `scoped` | Separate interpreter instance per user, same process |
| `shared` | `isolated` | Separate process per user |
| `scoped` | `scoped` | Separate instance per user+note |
| `isolated` | `isolated` | Separate process per user+note (full isolation) |

## Plugin System & Reflection Patterns

### PluginManager — Custom Classloading

`PluginManager` (`zeppelin-zengine/.../plugin/PluginManager.java`) loads plugins without Java SPI:

```
Plugin loading flow:
1. Check builtin list (hardcoded class names):
   - Launchers: StandardInterpreterLauncher, SparkInterpreterLauncher
   - NotebookRepos: VFSNotebookRepo, GitNotebookRepo
   → if builtin: Class.forName(className) — direct classloading

2. If not builtin → external plugin:
   → Scan pluginsDir/{Launcher|NotebookRepo}/{pluginName}/ for JARs
   → Create URLClassLoader with those JARs
   → classLoader.loadClass(className)
   → Instantiate via reflection (constructor parameters)
```

External plugin directory structure:
```
plugins/
  Launcher/
    DockerInterpreterLauncher/
      *.jar
    K8sStandardInterpreterLauncher/
      *.jar
  NotebookRepo/
    S3NotebookRepo/
      *.jar
    GCSNotebookRepo/
      *.jar
```

### ReflectionUtils

`ReflectionUtils` (`zeppelin-zengine/.../util/ReflectionUtils.java`) provides generic reflection-based instantiation:

```java
// No-arg constructor
ReflectionUtils.createClazzInstance(className)

// Parameterized constructor
ReflectionUtils.createClazzInstance(className, parameterTypes, parameters)
```

Used to instantiate:
- `RecoveryStorage` — in `RemoteInterpreterServer` and `InterpreterSettingManager`
- `ConfigStorage` — in `InterpreterSettingManager`
- `LifecycleManager` — in `RemoteInterpreterServer`
- `NotebookRepo` — in `PluginManager`
- `InterpreterLauncher` — in `PluginManager`

### Interpreter Discovery

`InterpreterSettingManager` discovers interpreters at startup:

```
1. Scan interpreterDir (default: interpreter/) for subdirectories
2. For each subdirectory, look for interpreter-setting.json
3. Parse JSON → List<RegisteredInterpreter>
4. Register each interpreter's className, properties, editor settings
```

`interpreter-setting.json` format (in each interpreter module's resources):
```json
[{
  "group": "spark",
  "name": "spark",
  "className": "org.apache.zeppelin.spark.SparkInterpreter",
  "properties": {
    "spark.master": { "defaultValue": "local[*]", "description": "Spark master" }
  },
  "editor": { "language": "scala", "editOnDblClick": false }
}]
```

### ZeppelinConfiguration Priority

Configuration values are resolved in order (first match wins):
1. **Environment variables** (e.g., `ZEPPELIN_HOME`, `ZEPPELIN_PORT`)
2. **System properties** (e.g., `-Dzeppelin.server.port=8080`)
3. **zeppelin-site.xml** (`conf/zeppelin-site.xml`)
4. **Hardcoded defaults** (`ConfVars` enum in `ZeppelinConfiguration`)

### HK2 Dependency Injection (zeppelin-server)

`ZeppelinServer.startZeppelin()` sets up HK2 DI via `ServiceLocatorUtilities.bind()`:

```java
new AbstractBinder() {
    protected void configure() {
        bind(storage).to(ConfigStorage.class);
        bindAsContract(PluginManager.class).in(Singleton.class);
        bindAsContract(InterpreterFactory.class).in(Singleton.class);
        bindAsContract(NotebookRepoSync.class).to(NotebookRepo.class).in(Singleton.class);
        bindAsContract(Notebook.class).in(Singleton.class);
        // ... InterpreterSettingManager, SearchService, etc.
    }
}
```

REST API classes use `@Inject` to receive these singletons.

## How to Help

When a developer asks about architecture:
1. Read the actual source code to verify current state
2. Explain the relevant module and class relationships
3. Point to specific files and line numbers
4. Suggest where new code should go based on module boundaries
