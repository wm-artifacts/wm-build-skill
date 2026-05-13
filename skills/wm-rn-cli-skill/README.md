# wm-reactnative-cli

A Claude Code skill for the `@wavemaker-ai/wm-reactnative-cli` tool. Guides an AI agent through all local WaveMaker React Native workflows — even if the developer doesn't know the CLI exists.

## What it covers

| Goal | CLI command |
|------|-------------|
| Run WaveMaker app locally from Studio preview URL | `wm-reactnative-ai sync <previewUrl>` |
| Preview WaveMaker app in browser from Studio URL | `wm-reactnative-ai run web-preview <previewUrl> --esbuild` |
| Build Android APK / AAB from exported WM RN zip | `wm-reactnative-ai build android <src>` |
| Build iOS IPA from exported WM RN zip | `wm-reactnative-ai build ios <src>` |

## How the sync workflow actually works (internals)

```
Preview URL
    │
    ▼
GET /services/application/wmProperties.js   ← get project name
    │
    ▼
Authenticate with Studio                    ← token from /studio/services/auth/token
    │
    ▼
GET /studio/services/projects/:id/vcs/gitBare  ← download full WM project as git bare repo zip
    │
    ▼
git clone locally (~/.wm-reactnative-cli/wm-projects/<name>/)
    │
    ▼
@wavemaker-ai/rn-codegen transpile          ← WM markup → Expo project
(version matched from project pom.xml)
    │
    ▼
npm install in generated Expo project
    │
    ▼
~/.wm-reactnative-cli/wm-projects/<name>/target/generated-expo-app/
    │
    ▼ (you run: npm start)
    │
    └──── polls Studio every 5s (if-modified-since on /rn-bundle/index.html)
              → change detected → git pull incremental bundle → re-transpile
```

## How the build workflow actually works (internals)

```
Exported WM RN zip (already transpiled by Studio)
    │
    ▼
Unzip → copy to dest (~/.wm-reactnative-cli/build/...)
    │
    ▼
npm install
    │
    ▼
npx expo prebuild              ← generates android/ and ios/ native folders
    │
    ▼
Gradle build (Android)         ← APK / AAB → output/android/
  or
Xcode build (iOS)              ← IPA → output/ios/
```

## Trigger phrases (for AI agent routing)

- "run my wavemaker rn app locally"
- "sync wavemaker app"
- "preview wavemaker react native in browser"
- "set up wavemaker mobile app locally"
- "build android apk for my wavemaker app"
- "build ios ipa from wavemaker"
- "open wavemaker app on my phone / emulator"
- "how do I test my wavemaker mobile app on device"

## Prerequisites by workflow

### Sync / Web-preview
- Node.js 22.x
- npm 10.9.x+
- Git
- Yarn (`npm install -g yarn`)
- Expo CLI (`npm install -g expo-cli@latest`)
- Active Studio preview session (keep the Studio tab open)

### Android build
- Node.js 22.x + Git + Yarn
- Java 17 with `JAVA_HOME` set
- Android SDK with `ANDROID_HOME`, `ANDROID_SDK`, `ANDROID_SDK_ROOT` set
- Gradle 8 with `GRADLE_HOME` set

### iOS build
- macOS only
- Xcode (latest) + CocoaPods
- Node.js 22.x + Git + Yarn
- LibreSSL (`brew install libressl`)
- Apple developer/distribution `.p12` certificate + `.mobileprovision` file
