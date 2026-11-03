# Wave Build Guide

This guide explains how to build **Wave** (the Seqera containers provisioning service, a Micronaut backend written in Groovy) on Linux, macOS (Intel or Apple Silicon) and Windows (through WSL2 or Docker), on x86_64 or ARM64.

Unless stated otherwise, paths are relative to the repository root (the folder that contains this file). Replace placeholders such as `<path-to-jdk>` with locations of your choice.

---

## 1. What you are building

| Item | Value |
|------|-------|
| Application | Wave **1.43.0** (`VERSION`) |
| Framework | Micronaut **4.10** (`micronautVersion` in `gradle.properties`; the build resolves `micronaut-core` 4.10.33), Netty HTTP server |
| Groovy | **4.0.31** at runtime (resolved from the Micronaut BOM; `groovy-4.0.31.jar` in the distribution). `buildSrc` pins 4.0.33 for tests only. |
| Build tool | Gradle **9.8.2** through the included wrapper (`gradlew`, `gradle/wrapper/gradle-wrapper.jar`). There is **no** `gradlew.bat`. |
| Required JDK | **Java 25** to compile and run (Gradle toolchain `languageVersion = 25` in `buildSrc`). Gradle itself can start on JDK 17 or newer and downloads JDK 25 automatically (section 2.2). |
| Also required to build | **Node.js 22 or newer** with `npm` / `npx` (section 2.3) |
| Database / cache | PostgreSQL and Redis are **optional**. In the default `local` environment Wave keeps its state in memory. |
| Output | `build/distributions/wave-1.43.0.zip` and `.tar` (runnable distribution: `bin/wave` + `lib/`), `build/libs/wave-1.43.0.jar` (application classes only, about 2.4 MB) |

The build also compiles the subprojects `wave-api` and `wave-utils`. The `wave-cli/` folder is a separate Gradle build (the Wave command line tool) and is not built by the commands below.

---

## 2. Things to know before building

### 2.1 Windows

The repository only has the Unix wrapper script `gradlew`. On Windows build inside **WSL2** (follow the Linux instructions) or with **Docker** (section 6).

### 2.2 JDK 25 is provisioned automatically

`settings.gradle` applies the `org.gradle.toolchains.foojay-resolver-convention` plugin. If no JDK 25 is installed, Gradle downloads one (Eclipse Temurin) into `~/.gradle/jdks` on the first build. This needs access to `api.foojay.io` and the Adoptium download servers. Behind a restrictive proxy, install JDK 25 yourself (section 3) so nothing has to be downloaded.

Running the built application needs **Java 25** too: on an older JVM it fails with `UnsupportedClassVersionError ... class file version 69.0`.

### 2.3 Node.js 22+ is needed at build time

`assemble` runs the tasks `installTypeSpec` (`npm install` in `typespec/`) and `generateOpenApi` (`npx tsp compile .`), which turn the TypeSpec API definitions into `openapi.yaml` and a Swagger UI page bundled into the application under `/openapi/`. Some of those npm packages require Node 22 or newer. With Node 20 the build fails at `:generateOpenApi` (`Process 'command 'npx'' finished with non-zero exit value 1`, after `npm warn EBADENGINE` warnings).

### 2.4 Dependency repositories

Besides Maven Central and the Gradle Plugin Portal, the build downloads from these repositories (declared in `build.gradle`), which must be reachable from the build machine:

- `https://s3-eu-west-1.amazonaws.com/maven.seqera.io/releases`
- `https://s3-eu-west-1.amazonaws.com/maven.seqera.io/snapshots`
- The npm registry (`https://registry.npmjs.org`) for the TypeSpec tools
- `api.foojay.io` and Adoptium, only if JDK 25 has to be downloaded (section 2.2)

---

## 3. Install the JDK and Node.js

### 3.1 JDK

Gradle needs **any JDK 17 or newer** to start. Installing **JDK 25** directly is recommended: Gradle then uses it as the toolchain, and the same JDK runs the application.

| Platform | JDK 25 |
|----------|--------|
| SDKMAN (Linux, macOS, WSL) | `sdk install java 25.<x>-tem` (pick a version from `sdk list java`) |
| Debian / Ubuntu | `sudo apt install openjdk-25-jdk` (if your release has it; otherwise SDKMAN or manual) |
| Fedora / RHEL | `sudo dnf install java-25-openjdk-devel` (if available) |
| macOS (Homebrew) | `brew install --cask temurin@25` |
| Manual | https://adoptium.net/temurin/releases/?version=25 |

```bash
export JAVA_HOME=<path-to-jdk>
export PATH="$JAVA_HOME/bin:$PATH"
java -version    # 25.x recommended (17+ is enough to start Gradle)
```

### 3.2 Node.js 22+

| Option | Command |
|--------|---------|
| nvm (Linux, macOS, WSL) | `nvm install 22 && nvm use 22` |
| Homebrew (macOS) | `brew install node@22` |
| Debian / Ubuntu | NodeSource packages, or the official binaries |
| Manual | https://nodejs.org/en/download (any 22.x or newer release) |

```bash
node --version    # v22.x or newer
npx --version
```

Make sure this Node is first on the `PATH` of the shell that runs Gradle.

---

## 4. Build

From the repository root:

```bash
./gradlew --no-daemon assemble
```

(`make compile` runs the same `./gradlew assemble`.)

The first build downloads Gradle, the dependencies (cached in `~/.gradle`), the npm packages (into `typespec/node_modules`) and, if needed, JDK 25. It takes about two minutes. Outputs:

| File | Description |
|------|-------------|
| `build/distributions/wave-1.43.0.zip` / `.tar` | Runnable distribution: `bin/wave` start script plus every dependency in `lib/`. This is the artifact to run outside Gradle (section 5.2). |
| `build/libs/wave-1.43.0.jar` | Wave classes and resources only (not a fat jar) |
| `typespec/tsp-output/@typespec/openapi3/openapi.yaml` | Generated OpenAPI specification (also bundled in the jar under `public/openapi/`) |

Other useful commands:

```bash
./gradlew --no-daemon compileGroovy    # compile only (no Node needed)
./gradlew --no-daemon clean            # delete build/
./gradlew --no-daemon test             # unit tests (Spock); some tests start containers and need Docker
```

---

## 5. Run

Wave listens on **port 9090** (`WAVE_SERVER_PORT` changes it).

### 5.1 With Gradle

```bash
./gradlew --no-daemon run
```

The `run` task starts the service with the Micronaut environments from `gradle.properties` (`micronautEnvs=local,mail,aws-ses`) and reads `config.yml` from the repository root. Without Redis or PostgreSQL configured, the `local` environment keeps everything in memory. Wait for `Startup completed ... Server Running: http://localhost:9090` in the output.

(`./run.sh` does the same with continuous rebuilds on file changes, `./gradlew run --continuous --watch-fs`, and writes to `wave.log`.)

### 5.2 From the distribution (no Gradle)

```bash
unzip build/distributions/wave-1.43.0.zip -d <some-dir>
export JAVA_HOME=<path-to-jdk25>
export WAVE_CONFIG_FILE=$PWD/config.yml
export JAVA_OPTS="-Dmicronaut.environments=local"
<some-dir>/wave-1.43.0/bin/wave
```

`WAVE_CONFIG_FILE` is required unless you start Wave from a folder that contains `config.yml`; otherwise it exits with `Config file does not exist or cannot be accessed: <cwd>/config.yml`.

### 5.3 Check that it is up

```bash
curl http://localhost:9090/service-info
# {"serviceInfo":{"version":"1.43.0","commitId":"(unknown)"}}

curl -s -o /dev/null -w '%{http_code}\n' -X POST -H 'Content-Type: application/json' -d '{}' http://localhost:9090/container-token
# 400 (the API is up and validates the request)
```

### 5.4 Real container requests (optional)

Building and proxying real container images needs more than the steps above:

- A **Docker engine** on the same machine (local builds run in Docker).
- Registry credentials in environment variables, read by `config.yml`: `DOCKER_USER` / `DOCKER_PAT`, `QUAY_USER` / `QUAY_PAT`, `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`, `AZURECR_USER` / `AZURECR_PAT`.
- A build repository you can push to. `src/main/resources/application-local.yml` points `wave.build.repo` and `wave.build.cache` at Seqera's private AWS ECR; override them, for example with `WAVE_BUILD_REPO` and `WAVE_BUILD_CACHE` (Micronaut maps these environment variables to the properties).
- On macOS, a public URL for the service (`WAVE_SERVER_URL`), see the README ("Get started").

Then, for example:

```bash
curl -H "Content-Type: application/json" -X POST http://localhost:9090/container-token \
  -d '{"containerImage":"ubuntu:latest"}'
```

### 5.5 Redis and PostgreSQL (optional)

Production-like setups add the Micronaut environments `redis` and `postgres` (`src/main/resources/application-redis.yml`, `application-postgres.yml`):

- Redis: `REDIS_HOST` (default `redis`) and `REDIS_PORT` (default `6379`).
- PostgreSQL: `wave.db.uri`, `wave.db.user`, `wave.db.password` (defaults are placeholders: `jdbc:postgresql://localhost:5432/changeme`, `changeme` / `changeme`), for example through `WAVE_DB_URI`, `WAVE_DB_USER`, `WAVE_DB_PASSWORD`.

Example with Gradle: `./gradlew run -PmicronautEnvs=local,redis,postgres` with Redis and PostgreSQL already running.

---

## 6. Build with Docker (no local JDK or Node)

Works on any OS that runs Docker. From the repository root:

```bash
docker run --rm -v "$PWD":/src -w /src eclipse-temurin:25-jdk bash -c '
  set -e
  apt-get update -qq && apt-get install -y -qq curl >/dev/null
  curl -fsSL https://nodejs.org/dist/v22.23.4/node-v22.23.4-linux-x64.tar.gz | tar -xz -C /opt
  export PATH=/opt/node-v22.23.4-linux-x64/bin:$PATH
  ./gradlew --no-daemon assemble
  chown -R '"$(id -u):$(id -g)"' /src
'
```

- The `eclipse-temurin` image has no `curl` (or `xz`), hence the `apt-get` line and the `.tar.gz` Node archive.
- The command downloads the **x64** Node build. On ARM64 hosts replace `linux-x64` with `linux-arm64` in both places (or use any Node 22+ release).
- The final `chown` gives the generated files (`build/`, `typespec/node_modules`, `.gradle/`) back to your user. On Windows PowerShell remove that line and replace `"$PWD":/src` with `"${PWD}:/src"`.

### 6.1 Container image

The project builds its image with **Jib** (`./gradlew jibDockerBuild`, needs a Docker daemon). The base image is `public.cr.seqera.io/seqeralabs/nf-jdk:corretto-25...` and the default target name is Seqera's private ECR repository, so pass your own name with `-PjibRepo=<name>`, for example `./gradlew jibDockerBuild -PjibRepo=wave:local`. This step was not verified for this guide.

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `:generateOpenApi` fails with `Process 'command 'npx'' finished with non-zero exit value 1` | Node.js is older than 22 (look for `EBADENGINE` warnings above the error). Use Node 22+ (section 3.2). |
| `A problem occurred starting process 'command 'npm''` | Node.js / npm is not on the `PATH` (section 3.2). |
| `Cannot find a Java installation ... languageVersion=25` | JDK 25 is not installed and the automatic download is blocked (section 2.2). Install JDK 25. |
| `UnsupportedClassVersionError ... class file version 69.0` when starting | The application runs on a JVM older than 25. Set `JAVA_HOME` to JDK 25. |
| `Config file does not exist or cannot be accessed: .../config.yml` | Start from the repository root or set `WAVE_CONFIG_FILE` (section 5.2). |
| `Could not resolve io.seqera:...` | The Seqera Maven repository on S3 is blocked (section 2.4). |
| `./gradlew: not found` or `gradlew.bat` missing on Windows | There is no Windows wrapper; use WSL2 or Docker (section 2.1). |
| `Address already in use` on 9090 | Another process uses the port; stop it or set `WAVE_SERVER_PORT`. |
| Container builds fail or try to push to `195996028523.dkr.ecr...` | Override the build repository and provide registry credentials (section 5.4). |

---

## 8. Quick reference

```bash
# Requirements: JDK 17+ (25 recommended) and Node.js 22+ (section 3)
java -version
node --version

# Build
./gradlew --no-daemon assemble        # -> build/distributions/wave-1.43.0.zip

# Run (in-memory, no Redis/PostgreSQL)
./gradlew --no-daemon run             # http://localhost:9090
curl http://localhost:9090/service-info
```
