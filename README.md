# wavemaker-build

A Claude Code skill that builds a WaveMaker application into any of six artifact shapes — WAR, full-app Docker image, frontend-only zip, backend-only WAR/JAR, frontend nginx container, or backend Tomcat container — so the resulting artifact can be deployed to whichever target you're aiming for (Tomcat, EC2, Kubernetes, S3 + CDN, App Service, etc.).

The skill **produces artifacts**. It does not push to registries or deploy remotely.

## What it builds

| Mode | Output | Typical deploy target |
|------|--------|-----------------------|
| WAR | `target/<app>-<version>.war` | Tomcat / JBoss / WaveMaker Cloud |
| Full Docker image | `<app>:latest` in local Docker | EC2 / ECS / Kubernetes / GKE / Azure CI |
| Frontend only | `target/<app>-frontend.zip` | S3 + CloudFront / Netlify / Vercel / any CDN |
| Backend only | `target/<app>-backend.war` (or per-service JARs) | Tomcat / Beanstalk / App Service / Heroku |
| Frontend container | `<app>-frontend:latest` (nginx + UI) | Any Docker host; pairs with split deploys |
| Backend container | `<app>-backend:latest` (Tomcat or JRE + stripped WAR) | Behind an internal LB; UI on a CDN calls it |

## Inputs the skill accepts

- A **checked-out WaveMaker app folder** (the one with `pom.xml`), or
- An **exported WaveMaker `.zip`** — the skill will unzip and use it.

It detects a WaveMaker app by looking for any of:
- `pom.xml` + `services/`
- `.wmproject.json` or `.wmproject.properties`
- `src/main/webapp/WEB-INF/wm.xml`

## How to trigger it

Open Claude Code inside your WaveMaker app folder and use natural phrasing. The skill auto-loads on any of these:

| You say | Mode triggered |
|---------|----------------|
| `build my wavemaker app as war` | WAR |
| `package this wm app as a docker image` | Full Docker image |
| `give me only the frontend artifacts` | Frontend zip |
| `build only the backend` | Backend WAR/JAR |
| `split frontend and backend into separate containers` | Both Docker variants |
| `build only the frontend as an nginx container` | Frontend container |
| `build my wm app multiple ways` | Asks which modes |
| `build with profile staging` | Same as above, but uses `-P staging` |

## Maven profile selection

WaveMaker apps typically declare a `deployment` profile (production settings) alongside `dev`. The skill picks the profile in this order:

1. **Explicitly named by you** in the prompt (`-P prod`, `with profile staging`) → that wins
2. **`deployment` profile present in `pom.xml`** → defaults to `-P deployment`, mentioned in one line so you can override
3. **Only one profile in the pom, not named `deployment`** → uses that one, with a heads-up
4. **Multiple profiles, none called `deployment`, you said nothing** → asks once
5. **No profiles in pom** → omits `-P` entirely

## Example walkthrough — WAR build

You say:
```
build my wavemaker app as war
```

The skill:
1. Detects `pom.xml` in the current folder → sets `APP_DIR`
2. Reads `artifactId=MyApp`, `version=1.0.0`, profiles `[dev, deployment]`
3. Picks `-P deployment` (default), tells you in one line
4. Preflight: `mvn`/`mvnw`, Java, Node 20 — stops with a clear error if missing
5. Runs `./mvnw -B -DskipTests -P deployment clean package`
6. Verifies `target/MyApp-1.0.0.war`
7. Reports: path + deploy hint (drop into Tomcat `webapps/`)

## Example walkthrough — split containers

You say:
```
split frontend and backend into separate docker containers
```

The skill:
1. Locates the app
2. Builds the WAR (same profile selection as above)
3. Extracts UI → `target/frontend/` and strips the WAR → `target/<app>-backend.war`
4. Writes `Dockerfile.frontend` (nginx) and `Dockerfile.backend` (Tomcat)
5. Builds both images: `<app>-frontend:latest`, `<app>-backend:latest`
6. Prints run commands for each

## What it deliberately does NOT do

- Push images to any registry (ECR, GHCR, Docker Hub) — that's your call
- Upload the frontend zip to S3 / Netlify / a CDN
- SSH or `kubectl` to a remote host
- Modify your `pom.xml`, `wm.xml`, or any WaveMaker source to "fix" a broken build
- Overwrite an existing `Dockerfile` (uses it as-is if present)
- Run tests (`-DskipTests` by default; override only on request)
- Scaffold `docker-compose.yml`, databases, or environment variables

## Files in this folder

- [SKILL.md](SKILL.md) — the actual skill definition that Claude Code loads
- [README.md](README.md) — this file

## Where the skill is installed

Two locations:

- **Active copy** (auto-loaded by Claude Code): `~/.claude/skills/wavemaker-build/SKILL.md`
- **This copy** (for browsing / version control): `wavemaker_skills/wavemaker-build/SKILL.md`

If you edit this local copy, copy it back to the active location for the change to take effect:

```bash
cp wavemaker-build/SKILL.md ~/.claude/skills/wavemaker-build/SKILL.md
```

## Pairing with other skills

The skill stops at build artifacts. For deployment, pair with target-specific tooling:

| Target | Tool |
|--------|------|
| EC2 / SSH host | `wavemaker-github-deploy` skill (in `~/.claude/skills/`) generates a GitHub Actions workflow |
| ECR / ECS | `aws ecr get-login-password` + `docker push` + `aws ecs update-service` |
| S3 (frontend zip) | `aws s3 sync target/frontend/ s3://<bucket>/` |
| Kubernetes | `docker push` + `kubectl set image` |
