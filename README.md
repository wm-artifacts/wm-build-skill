# wm-local-dev-skill

A collection of Claude Code skills for local WaveMaker development tasks.

## Skills

| Skill | Folder | What it does |
|-------|--------|--------------|
| **wavemaker-build** | `skills/wm-build-skill/` | Build a WaveMaker web app into a WAR, Docker image, frontend zip, or backend WAR |
| **wm-reactnative-cli** | `skills/wm-rn-cli-skill/` | Run, preview, and build WaveMaker React Native apps locally |

---

### wavemaker-build

Produces deployment artifacts from a WaveMaker web application — either a checked-out project or an exported `.zip`.

| Mode | Output | Deploy target |
|------|--------|---------------|
| WAR | `target/<app>.war` | Tomcat / JBoss / WaveMaker Cloud |
| Docker image | `<app>:latest` | EC2 / ECS / Kubernetes / Azure |
| Frontend only | `target/<app>-frontend.zip` | S3 + CloudFront / Netlify / any CDN |
| Backend only | `target/<app>-backend.war` | Tomcat / Beanstalk / App Service |

→ See [`skills/wm-build-skill/README.md`](skills/wm-build-skill/README.md)

---

### wm-reactnative-cli

Wraps the [`@wavemaker-ai/wm-reactnative-cli`](https://www.npmjs.com/package/@wavemaker-ai/wm-reactnative-cli) tool. Guides an AI agent through all local WaveMaker React Native workflows — even if the developer doesn't know the CLI exists.

| Goal | Command |
|------|---------|
| Run app locally from Studio preview URL | `wm-reactnative-ai sync <previewUrl>` |
| Preview app in browser from Studio URL | `wm-reactnative-ai run web-preview <previewUrl>` |
| Build Android APK / AAB | `wm-reactnative-ai build android <src>` |
| Build iOS IPA | `wm-reactnative-ai build ios <src>` |

→ See [`skills/wm-rn-cli-skill/README.md`](skills/wm-rn-cli-skill/README.md)
