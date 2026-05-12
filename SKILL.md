---
name: wavemaker-build
description: Build a WaveMaker application in one or more ways - WAR package, Docker image, frontend-only artifacts (UI bundle), or backend-only artifacts (services WAR/JAR). Use when the user says things like "build my wavemaker app", "package wavemaker as war", "give me only the frontend artifacts from my wm app", "build only the backend", "build wm app multiple ways", or asks how to package a WaveMaker app for deployment in any of the modes from https://docs.wavemaker.ai/docs/build-and-deploy/overview.
---

# WaveMaker → build artifacts (WAR / Docker / frontend / backend)

Goal: take a WaveMaker application and produce the artifact(s) the user wants — a deployable WAR, a Docker image, just the UI bundle, or just the backend services.

This skill is self-contained. It works whether the source is a checked-out project tree or an exported WaveMaker `.zip`, and it ships every Dockerfile template inline.

## What each artifact is for

The skill exists to produce the right *shape* of artifact for whichever deployment style the user is targeting:

| Artifact | Typical deploy targets |
|----------|------------------------|
| **WAR** | Any servlet container — Tomcat, JBoss, WebSphere; WaveMaker Cloud; on-prem Tomcat on a VM |
| **Docker image (full app)** | Any Docker host — EC2, ECS, GKE, Kubernetes, Azure Container Instances, on-prem |
| **Frontend zip** | Any static host — S3 + CloudFront, Netlify, Vercel, Azure Static Web Apps, GitHub Pages, nginx from disk, any CDN |
| **Backend WAR/JAR** | Tomcat (drop into `webapps/`), `java -jar` on a VM, AWS Elastic Beanstalk, Azure App Service, Heroku |

If the user's deployment style isn't obvious, ask which target they're aiming for and pick the matching mode in Step 2.

## Step 1 — locate the app

Decide the source mode **before** doing anything else:

- **Mode A — checked-out project**: the current working directory (or a folder the user names) already contains a WaveMaker app. Detect by looking for any of:
  - `pom.xml` plus a `services/` folder
  - `.wmproject.json` or `.wmproject.properties`
  - `src/main/webapp/WEB-INF/wm.xml`
- **Mode B — exported zip**: the user has a WaveMaker app `.zip` on disk.

If neither is obvious, ask the user **once**:

> Is your WaveMaker app already checked out in this folder, or do you have an exported `.zip`? If it's a zip, please give me the absolute path.

Don't guess.

- **Mode A** → set `APP_DIR` to the folder that contains `pom.xml`. Run `ls -la "$APP_DIR/pom.xml"` to confirm.
- **Mode B** → verify the zip exists, then:
  ```bash
  mkdir -p build && unzip -q "<zip-path>" -d build
  ```
  WaveMaker zips usually unpack into a single top-level folder — set `APP_DIR` to that folder. If `APP_DIR/pom.xml` doesn't exist after extraction, stop and tell the user the zip doesn't look like a WaveMaker Maven project.

## Step 2 — ask which build(s) the user wants

If the user already named the build mode in their prompt (e.g. "give me only the frontend"), skip the menu and confirm in one line. Otherwise ask **once**, in a single message:

> What do you want to build? Pick one or more:
> 1. **WAR** — standard deployable, output in `target/*.war`
> 2. **Docker image (full app)** — runnable container of the whole app
> 3. **Frontend only** — UI bundle (html/js/css) as a zip, for CDN/static hosting
> 4. **Backend only** — services WAR/JAR with the UI stripped
> 5. **All artifacts (1, 3, 4)**

Don't assume; multi-select is fine.

## Step 3 — read packaging and profiles from `pom.xml`

Read `<artifactId>`, `<version>`, `<packaging>`, and the list of `<profiles>` from `APP_DIR/pom.xml`. You'll use:

- `artifactId` (lowercased) for output filenames and Docker tags
- `<packaging>` (`war` or `jar`) — default to `war` if missing
- `<java.version>` / `<maven.compiler.target>` if present, to pick the JDK in any Dockerfile
- `<profiles>` — list each `<profile><id>...</id>` so you know which profile flag to pass to Maven

Note the built artifact will be `target/<artifactId>-<version>.war` (or `.jar`).

### Profile selection

WaveMaker apps typically declare a `deployment` profile (production build settings — minified UI, prod logging, prod DB datasource swaps) alongside a default `dev` profile. Pick the profile to pass to Maven in this order:

1. **User specified a profile** in their prompt (e.g. *"build with profile staging"*, *"-P prod"*) → use exactly that.
2. **`deployment` profile exists in pom.xml** → default to `-P deployment`. Mention the choice in one line so the user can override (*"using `-P deployment` profile — say so if you want a different one"*). Don't ask first.
3. **No `deployment` profile, only one named profile** → use that one and tell the user.
4. **Multiple profiles, none called `deployment`, user didn't specify** → ask once which profile to use.
5. **No profiles at all** → omit `-P` entirely.

Set `MVN_PROFILE_ARG` to either `-P <profile-name>` or empty, and reuse it for every Maven invocation in Step 4.

## Step 4 — build

Run from `APP_DIR`. Prefer the Maven wrapper (`./mvnw`) when it exists and is executable; otherwise use `mvn`. Always pass `-B -DskipTests` unless the user explicitly asks for tests. Always pass `$MVN_PROFILE_ARG` from Step 3 (it's either `-P deployment`, another profile name, or empty).

WaveMaker's parent pom triggers a `ui-build.js` antrun step that runs Node — make sure Node 20+ is on PATH locally, or warn the user. Node 18 fails on `undici`'s `File` global referenced by recent angular-codegen releases. If the local Node version is the blocker, suggest the user run the Docker mode below instead — its Dockerfile build stage pins Node 20 internally.

### 4a. WAR (mode 1)

```bash
./mvnw -B -DskipTests $MVN_PROFILE_ARG clean package
```

Verify `target/<artifactId>-<version>.war` exists. If not, surface the failing Maven module and stop — don't try to fix the WaveMaker source.

### 4b. Docker image, full app (mode 2)

Use a multi-stage Dockerfile so the user doesn't need Maven or Node locally.

**Decide which Dockerfile to use, in this order:**

1. **`APP_DIR/Dockerfile` exists** → use it as-is. Don't overwrite, don't back up. WaveMaker zips and checked-out projects often ship a Dockerfile on purpose; trust it. Skip straight to `docker build`.
2. **Only a build-stage variant exists** (e.g. `Dockerfile.build`, `Dockerfile.builder`, `build.Dockerfile`) and there's no plain `Dockerfile` → fall through to the official WaveMaker template below. Don't try to compose the variant into the main image; treat it as a leftover from a fragmented setup. Mention to the user which variant file you found and that you're writing a new `Dockerfile` alongside it.
3. **No Dockerfile at all** → write the official WaveMaker template below to `APP_DIR/Dockerfile`.

**Default template — official WaveMaker images (preferred when no `Dockerfile` is present):**

This is the canonical WaveMaker Docker recipe from the [WaveMaker docs](https://docs.wavemaker.com/learn/app-development/deployment/build-with-docker/#build-docker-image). It uses the `wavemakerapp/app-builder` and `wavemakerapp/app-runtime-tomcat` images, which already bundle the right Maven, Node, and Tomcat versions — no need to install Node 20 or pin Java by hand.

The template lives at `~/.claude/skills/wavemaker-build/references/Dockerfile.wavemaker`. Copy it verbatim to `APP_DIR/Dockerfile` — don't inline-paste from memory; always pull the latest version from the reference file so changes to the canonical recipe propagate automatically.

```bash
cp ~/.claude/skills/wavemaker-build/references/Dockerfile.wavemaker "$APP_DIR/Dockerfile"
```

Build with (BuildKit is required for the `--mount=type=cache` lines):

```bash
cd "$APP_DIR" && DOCKER_BUILDKIT=1 docker build \
    --build-arg wavemaker_version=latest \
    --build-arg profile=<profile-name-from-Step-3-or-omit> \
    -t <artifactId-lowercased>:latest .
```

Use the same profile resolution from Step 3:
- User-specified profile → pass it.
- Otherwise `deployment` if present in pom.xml → pass `--build-arg profile=deployment`.
- No profiles in pom.xml → drop the `profile` build-arg entirely.

If the user wants a specific WaveMaker runtime version (e.g. `11.7.0`), pass `--build-arg wavemaker_version=11.7.0`.

**Local-browse companion image:**

The base image built above is the right artifact to *deploy* — but WaveMaker apps default to forcing HTTPS via `request.isSecure()` (see https://docs.wavemaker.com/learn/how-tos/ssl-offloading), so opening `http://localhost:8080/<app>/` in a browser after a plain `docker run -p 8080:8080` redirects to a dead `https://localhost:443` and looks broken. In production a TLS-terminating proxy/LB sets `X-Forwarded-Proto: https` and the redirect is fine; locally there's no proxy.

After every successful mode-2 build, also build a `:local`-tagged companion image that patches Tomcat's 8080 Connector to report `isSecure()=true` unconditionally. This is a local-testing convenience — never deploy the `:local` image to production.

Use `~/.claude/skills/wavemaker-build/references/Dockerfile.local`. It expects a `--build-arg base=<base-image>` pointing at the image you just built:

```bash
docker build \
    --build-arg base=<artifactId-lowercased>:latest \
    -f ~/.claude/skills/wavemaker-build/references/Dockerfile.local \
    -t <artifactId-lowercased>:latest-local \
    ~/.claude/skills/wavemaker-build/references/
```

The user can then open `http://localhost:8080/<artifactId>/` in a browser with:

```bash
docker run --rm -p 8080:8080 <artifactId-lowercased>:latest-local
```

If the user explicitly says they don't want the local-browse image (e.g. they're building only for production deploy), skip it. Otherwise build both by default and mention both in the report-back.

**Fallback templates — generic Maven + Tomcat / JRE (only if the user explicitly opts out of `wavemakerapp/*` images, e.g. air-gapped or no DockerHub access):**

WAR packaging:

```dockerfile
# syntax=docker/dockerfile:1
FROM maven:3.9-eclipse-temurin-11 AS build
WORKDIR /src

# WaveMaker's Maven build invokes ui-build.js (Angular codegen) which needs Node.
# Use Node 20 — Node 18 fails on undici's File global referenced by recent angular-codegen releases.
RUN mkdir -p /usr/local/content/node \
    && cd /usr/local/content/node \
    && curl -fsSL https://nodejs.org/dist/v20.18.0/node-v20.18.0-linux-x64.tar.gz -o node.tar.gz \
    && tar -xzf node.tar.gz \
    && ln -s /usr/local/content/node/node-v20.18.0-linux-x64/bin/node /usr/local/bin/node \
    && ln -s /usr/local/content/node/node-v20.18.0-linux-x64/bin/npm /usr/local/bin/npm \
    && rm -f node.tar.gz

COPY . .
RUN mvn -B -DskipTests clean package

FROM tomcat:9.0-jre11-temurin
RUN rm -rf /usr/local/tomcat/webapps/ROOT
COPY --from=build /src/target/*.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

JAR packaging (Spring Boot fat jar):

```dockerfile
# syntax=docker/dockerfile:1
FROM maven:3.9-eclipse-temurin-11 AS build
WORKDIR /src

RUN mkdir -p /usr/local/content/node \
    && cd /usr/local/content/node \
    && curl -fsSL https://nodejs.org/dist/v20.18.0/node-v20.18.0-linux-x64.tar.gz -o node.tar.gz \
    && tar -xzf node.tar.gz \
    && ln -s /usr/local/content/node/node-v20.18.0-linux-x64/bin/node /usr/local/bin/node \
    && ln -s /usr/local/content/node/node-v20.18.0-linux-x64/bin/npm /usr/local/bin/npm \
    && rm -f node.tar.gz

COPY . .
RUN mvn -B -DskipTests clean package

FROM eclipse-temurin:11-jre
WORKDIR /app
COPY --from=build /src/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Notes:
- The official `wavemakerapp/*` template is the preferred default — it matches what WaveMaker docs prescribe and the runtime images are tuned for WaveMaker apps. Only fall back to the generic Maven/Tomcat template when the user can't pull from `wavemakerapp/*`.
- For the fallback template: pin Java 11 by default. Bump both stages to `17` if `pom.xml` declares `<java.version>17</java.version>` or `maven.compiler.target` 17.
- Keep the Node 20 install step even on "backend-looking" projects — the parent pom still runs `ui-build.js`.
- If `application.properties` / `application.yml` declares a non-default `server.port`, update `EXPOSE`.
- Don't add `HEALTHCHECK`, non-root users, or other hardening unless asked.

Also write `APP_DIR/.dockerignore` if it doesn't already exist:

```
target/
.git/
.idea/
*.iml
node_modules/
build/
```

Build (only when using one of the fallback templates above — the official template has its own build command earlier in this section):

```bash
cd "$APP_DIR" && docker build -t <artifactId-lowercased>:latest .
```

If the build fails: surface Maven errors as-is; if the Docker daemon isn't running, tell the user to start it — don't retry blindly.

### 4c. Frontend only (mode 3)

WaveMaker has no first-class "frontend-only" Maven goal, so build the WAR and extract the UI:

```bash
./mvnw -B -DskipTests $MVN_PROFILE_ARG clean package
WAR=$(ls target/*.war | head -1)
mkdir -p target/frontend
unzip -q "$WAR" -d target/frontend
# Drop server-side bits — what's left is the UI bundle
rm -rf target/frontend/WEB-INF target/frontend/META-INF
(cd target && zip -qr "<artifactId>-frontend.zip" frontend)
```

Output: `target/<artifactId>-frontend.zip` containing `index.html`, `pages/`, `services/` (UI service descriptors, not Java), `app.min.js`, `assets/`, etc. — a deployable static bundle.

If the user only wants the unzipped folder, skip the zip step and point them at `target/frontend/`.

### 4d. Backend only (mode 4)

First check whether the project has a multi-module `services/` layout (each backend service compiles to its own JAR):

```bash
ls APP_DIR/services/*/target/*.jar 2>/dev/null
```

- **If those JARs exist** → those *are* the backend artifacts. Tell the user the paths and stop. Don't repackage.
- **Otherwise** → derive a backend-only WAR from the full WAR by stripping the UI:

  ```bash
  ./mvnw -B -DskipTests $MVN_PROFILE_ARG clean package
  WAR=$(ls target/*.war | head -1)
  mkdir -p target/backend-extract
  unzip -q "$WAR" -d target/backend-extract
  # Keep only WEB-INF and META-INF — drop UI assets
  find target/backend-extract -mindepth 1 -maxdepth 1 \
      ! -name 'WEB-INF' ! -name 'META-INF' -exec rm -rf {} +
  (cd target/backend-extract && zip -qr "../<artifactId>-backend.war" .)
  ```

  Output: `target/<artifactId>-backend.war` — Spring services, web.xml, classes, libs only.

## Step 5 — preflight checks

- **Maven**: if neither `./mvnw` nor `mvn` is on PATH, stop and tell the user; don't try to install.
- **Java**: `java -version` must be ≥ the project's target JDK.
- **Node**: only required for the local Maven build (the `ui-build.js` step). Skip the check if the user only picked Docker modes — the Dockerfile build stage has its own Node.
- **Docker**: only required for mode 2. Run `command -v docker` and `docker info >/dev/null 2>&1` before any `docker build`. If it's missing or the daemon's down, stop with a clear message — don't retry.

## Step 6 — report back

For each artifact built, give one short bullet:

- **WAR** → path + `deploy by dropping into Tomcat's webapps/`
- **Docker image (full)** → image tag + `docker run --rm -p 8080:8080 <tag>`. Also report the `:latest-local` companion (built automatically) with: "open http://localhost:8080/<artifactId>/ in a browser after `docker run --rm -p 8080:8080 <tag>-local`. Local-testing only — deploy the non-`-local` image to production behind a TLS-terminating proxy."
- **Frontend zip** → path + "extract to any static host / S3 / nginx"
- **Backend WAR/JAR** → path + how to run (Tomcat drop-in or `java -jar`)

Keep the report compact — the user can see the file tree themselves.

## Things to NOT do

- Do not run `mvn install` or `mvn deploy` — `package` is enough for build artifacts.
- Do not modify `pom.xml`, `app.scripts.json`, `wm.xml`, or any WaveMaker source to "make the build pass". If it doesn't build, surface the error and stop.
- Do not push any image to a registry — building is local. Pushing is the user's call.
- Do not overwrite an existing `Dockerfile`. Use it as-is (the user shipped it on purpose).
- Do not invent database credentials, env vars, or `docker-compose.yml`. If the app needs a DB at runtime, mention it once but don't scaffold it.
- Do not delete the user's `target/` beyond what `mvn clean` does. Leave intermediate `target/frontend/` and `target/backend-extract/` for inspection. Don't delete a user-provided `.zip` or its extracted `build/` folder either.
- Do not run tests by default. Only run `mvn test` (or omit `-DskipTests`) if the user explicitly asks.
