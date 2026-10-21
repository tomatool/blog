+++
title = 'Code coverage of tomato tests: the JVM (Kotlin, Java, Scala)'
linkTitle = 'Kotlin, Java, Scala'
weight = 30
tool = 'JaCoCo agent · JAVA_OPTS'
date = 2026-10-21T03:00:00+02:00
summary = "Nothing to rebuild: the JaCoCo agent instruments classes as they load and writes its data from a shutdown hook, which the JVM runs on SIGTERM by itself."
tags = ['coverage', 'kotlin', 'java', 'jvm']
+++

On the JVM nothing has to be rebuilt. The [JaCoCo](https://www.jacoco.org/)
agent instruments classes as they load and writes its data from a shutdown
hook. The JVM runs shutdown hooks on SIGTERM by itself, so this works even if
the application does nothing special about the signal.

## Run it

```bash
JACOCO=0.8.13
curl -fsSLO https://repo1.maven.org/maven2/org/jacoco/org.jacoco.agent/$JACOCO/org.jacoco.agent-$JACOCO-runtime.jar
curl -fsSLO https://repo1.maven.org/maven2/org/jacoco/org.jacoco.cli/$JACOCO/org.jacoco.cli-$JACOCO-nodeps.jar

# Gradle's start script (from installDist) passes JAVA_OPTS to the JVM.
export JAVA_OPTS="-javaagent:$PWD/org.jacoco.agent-$JACOCO-runtime.jar=destfile=$PWD/coverage/blackbox.exec,includes=demo.*"
tomato run

java -jar org.jacoco.cli-$JACOCO-nodeps.jar report coverage/blackbox.exec \
  --classfiles build/install/payments/lib/payments.jar \
  --sourcefiles src/main/kotlin \
  --xml coverage/jacoco.xml --html coverage/html
```

`tomato.yml` keeps its `command: ./build/install/payments/bin/payments`; the
start script picks up `JAVA_OPTS` from the environment tomato passes on.

## Notes

- **`includes=demo.*`** limits measurement to your packages. Without it,
  every library class that loads is measured too, and the run is slower.
- **A plain `java -jar`** command does not read `JAVA_OPTS`. Use
  `JAVA_TOOL_OPTIONS` instead: every JVM reads it, whatever starts it.
- **Keep your own shutdown hooks short.** They run alongside JaCoCo's, and
  tomato sends SIGKILL five seconds after SIGTERM. Stop the consumer, join it
  for a few seconds at most, close connections.
- **Point `--classfiles` at what you shipped**: the jar from the build, so the
  report matches the bytecode that ran.
- **Per-scenario dumps.** The agent can serve its data over TCP
  (`output=tcpserver`), and `jacococli dump` collects it without stopping the
  service. That is more plumbing than most suites need.
- **Kotlin** compiles to classes JaCoCo understands, including data classes and
  `when`; JaCoCo filters most of the compiler-generated code from the counts.

## In CI

```yaml
build:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v7
    - uses: actions/setup-java@v6
      with: { distribution: temurin, java-version: 17 }
    - run: ./gradlew test installDist
    - uses: actions/upload-artifact@v7
      with: { name: build, path: build/install/payments/ }

blackbox:
  needs: build
  runs-on: ubuntu-latest
  env:
    JACOCO: "0.8.13"
    JAVA_OPTS: -javaagent:${{ github.workspace }}/jacoco/agent.jar=destfile=${{ github.workspace }}/coverage/blackbox.exec,includes=demo.*
  steps:
    - uses: actions/checkout@v7
    - uses: actions/download-artifact@v8
      with: { name: build, path: build/install/payments/ }
    - uses: actions/setup-java@v6
      with: { distribution: temurin, java-version: 17 }
    - run: |
        chmod +x build/install/payments/bin/payments
        mkdir -p jacoco
        curl -fsSL -o jacoco/agent.jar "https://repo1.maven.org/maven2/org/jacoco/org.jacoco.agent/$JACOCO/org.jacoco.agent-$JACOCO-runtime.jar"
        curl -fsSL -o jacoco/cli.jar "https://repo1.maven.org/maven2/org/jacoco/org.jacoco.cli/$JACOCO/org.jacoco.cli-$JACOCO-nodeps.jar"
    - run: tomato run
    - if: always()
      run: |
        java -jar jacoco/cli.jar report coverage/blackbox.exec \
          --classfiles build/install/payments/lib/payments.jar --sourcefiles src/main/kotlin \
          --xml coverage/jacoco.xml
```
