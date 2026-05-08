---
name: build
description: Apache Zeppelin build and test guide including shaded JAR chain, module build order, and Maven profiles
tools:
  - Read
  - Bash
  - Grep
  - Glob
---

# Zeppelin Build & Test Guide

## Basic Commands

```bash
# Full build (skip tests)
./mvnw clean package -DskipTests

# Build single module (--am builds required upstream modules)
./mvnw clean package -pl zeppelin-server --am -DskipTests

# Run module tests
./mvnw test -pl zeppelin-interpreter --am

# Run single test class/method
./mvnw test -pl zeppelin-server --am -Dtest=NotebookServerTest
./mvnw test -pl zeppelin-server --am -Dtest=NotebookServerTest#testMethod

# Common profiles
#   -Pspark-3.5 -Pspark-scala-2.12   Spark version
#   -Pflink-117                        Flink version
#   -Pbuild-distr                      Full distribution
#   -Prat                              Apache RAT license check
#   -Pweb-classic                      Additionally builds the classic UI web module when specified
```

## Build Gotchas

### Shaded JAR Rebuild Chain

The most common build mistake: modifying `zeppelin-interpreter` without rebuilding `zeppelin-interpreter-shaded`. The shaded JAR is an uber JAR that all interpreter processes use. If it's stale, you get `ClassNotFoundException` or `NoSuchMethodError` at runtime.

```bash
# After changing zeppelin-interpreter, ALWAYS rebuild in order:
./mvnw clean package -pl zeppelin-interpreter -DskipTests
./mvnw clean package -pl zeppelin-interpreter-shaded -DskipTests
# Then rebuild affected interpreter modules

# Shorthand:
./mvnw clean package -pl zeppelin-interpreter,zeppelin-interpreter-shaded -DskipTests
```

The shaded JAR is also copied to `interpreter/` directory by maven-antrun-plugin after packaging. If this directory has a stale JAR, interpreter processes will load old code.

### Module Build Order

Maven modules are ordered in the root `pom.xml`. Key sequence:

```
zeppelin-interpreter → zeppelin-interpreter-shaded → zeppelin-server
```

All interpreter modules build after `zeppelin-interpreter-shaded`. A second shading chain exists for Jupyter:

```
zeppelin-jupyter-interpreter → zeppelin-jupyter-interpreter-shaded → python, rlang
```

## Maven Profiles

| Profile | Purpose |
|---------|---------|
| `-Pspark-3.3/3.4/3.5/4.0` | Spark version |
| `-Pspark-scala-2.12/2.13` | Scala version for Spark |
| `-Pflink-115/116/117` | Flink version |
| `-Pweb-classic` | Include legacy AngularJS frontend |
| `-Pweb-e2e` | Enable Angular E2E tests |
| `-Pweb-dist` | Build web distribution |
| `-Pbuild-distr` | Full distribution build |
| `-Pintegration` | Include integration test modules |
| `-Prat` | Apache RAT license check |

## Test JVM Configuration

Test JVM args are configured as `-Xmx2g -Xms1g`. Integration tests require MySQL.
