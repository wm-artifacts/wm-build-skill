---
name: wm-reactnative-cli
description: |
  Use when the user wants to either:

  • Get the exported WaveMaker RN project locally from Studio (sync/preview — given a Studio preview URL, downloads and sets up the exported RN project so it can be run locally)
  • Build an exported WaveMaker RN zip into an Android APK/AAB or iOS IPA (user already has the exported zip on disk)

  Trigger phrases: "run my wavemaker rn app locally", "sync wavemaker app", "preview wavemaker react native", "build android apk for my wavemaker app", "build ios ipa", "set up wavemaker mobile app locally", "open wavemaker app on my phone", "build react native from wavemaker", "how do i test my wavemaker mobile app", "i have the wavemaker rn zip, build it".

  Do NOT use for: any WaveMaker project that still has WaveMaker source code/markup (web or mobile). This CLI is only for the exported React Native app — after Studio has compiled the WaveMaker project into an RN project.
---

# WaveMaker React Native CLI skill

## What this skill does

The `@wavemaker-ai/wm-reactnative-cli` package (command: `wm-reactnative-ai`) bridges WaveMaker Studio and a local React Native / Expo development environment. It has two independent workflows:

**Workflow A — Sync / Preview (from Studio preview URL)**
Used when the developer wants to run or preview the app locally while still editing in Studio. The CLI:
1. Authenticates with WaveMaker Studio
2. Downloads the raw WaveMaker project sources as a git repo
3. Runs `@wavemaker-ai/rn-codegen` to transpile the WaveMaker markup into a full Expo project
4. Installs npm dependencies
5. Hands the developer a runnable Expo project
6. Polls Studio every 5 seconds — when Studio changes are detected, it pulls the diff and re-transpiles incrementally

**Workflow B — Build (from exported RN zip)**
Used when the developer has exported the RN project from Studio (Studio has already run codegen) and wants a native APK or IPA. The CLI:
1. Unzips the exported project
2. Runs `npm install`
3. Runs `npx expo prebuild` to generate native `android/` and `ios/` folders
4. Invokes Gradle (Android) or Xcode (iOS) to produce the binary

---

## Step 1 — Identify what the user wants

Determine the workflow **before doing anything else**:

| User intent | Workflow | Command |
|-------------|----------|---------|
| Run/preview locally from Studio | A | `sync` or `run web-preview` |
| Build APK/AAB from exported zip | B | `build android` |
| Build IPA from exported zip | B | `build ios` |

If unclear, ask **once**:

> Are you trying to (1) run/preview your WaveMaker app locally while it's still in Studio, or (2) build an APK/IPA from an exported WaveMaker RN zip?

---

## Step 2 — Check if the CLI is installed

```bash
which wm-reactnative-ai || echo "NOT_FOUND"
```

If not found, install it:
```bash
npm install -g @wavemaker-ai/wm-reactnative-cli
```

Verify after install:
```bash
wm-reactnative-ai --version
```

If installation fails, stop and tell the user to install Node.js/npm first.

> **When in doubt about any CLI flag, command, or version** — check the official npm package page for the latest documentation and release notes: https://www.npmjs.com/package/@wavemaker-ai/wm-reactnative-cli

---

## Step 3 — Check prerequisites

**Always check prerequisites before running any command.** Run all checks first, collect all failures, then report them together in one message — do not run the checks one by one interactively.

### Prerequisites for Workflow A (sync / web-preview)

Run these checks in parallel:

```bash
node --version 2>/dev/null || echo "MISSING: node"
git --version 2>/dev/null || echo "MISSING: git"
npm --version 2>/dev/null || echo "MISSING: npm"
yarn --version 2>/dev/null || echo "MISSING: yarn"
```

**Node.js**: must be v22.x. Check with `node --version`. If missing or wrong version, see "Fixing prerequisites" below.
**Git**: must be installed. Check with `git --version`.
**npm**: must be v10.9.x or later. Check with `npm --version`.
**Yarn**: must be installed globally. Check with `yarn --version`. If missing: `npm install -g yarn`
**Expo CLI**: Check with `expo --version 2>/dev/null || npx expo --version`. If missing: `npm install -g expo-cli@latest`

### Prerequisites for Workflow B — Android build

Run these checks in parallel:

```bash
node --version 2>/dev/null || echo "MISSING: node"
git --version 2>/dev/null || echo "MISSING: git"
yarn --version 2>/dev/null || echo "MISSING: yarn"
java -version 2>&1 | head -1 || echo "MISSING: java"
echo "JAVA_HOME=$JAVA_HOME"
echo "ANDROID_HOME=$ANDROID_HOME"
echo "ANDROID_SDK=$ANDROID_SDK"
echo "GRADLE_HOME=$GRADLE_HOME"
gradle --version 2>/dev/null | head -3 || echo "MISSING: gradle"
```

Required:
- **Node.js 22.x**: `node --version`
- **Git**: `git --version`
- **Java 17**: `java -version` must show 17. `JAVA_HOME` must be set and point to a Java 17 JDK.
- **Android SDK**: `ANDROID_HOME`, `ANDROID_SDK`, and `ANDROID_SDK_ROOT` must all be set and point to the SDK directory.
- **Gradle 8**: `gradle --version` must show 8.x. `GRADLE_HOME` must be set.
- **Yarn**: `yarn --version`

### Prerequisites for Workflow B — iOS build

```bash
uname -s | grep -q Darwin || echo "MISSING: macOS required for iOS builds"
xcode-select -p 2>/dev/null || echo "MISSING: Xcode"
pod --version 2>/dev/null || echo "MISSING: CocoaPods"
node --version 2>/dev/null || echo "MISSING: node"
git --version 2>/dev/null || echo "MISSING: git"
yarn --version 2>/dev/null || echo "MISSING: yarn"
openssl version 2>/dev/null || echo "MISSING: openssl/libressl"
```

Required:
- **macOS**: iOS builds require a Mac. Stop immediately if not on macOS.
- **Xcode (latest)**: `xcode-select -p` must return a path.
- **CocoaPods**: `pod --version` must work.
- **Node.js 22.x**, **Git**, **Yarn** (same as Android)
- **LibreSSL**: `openssl version` should show LibreSSL. If not: `brew install libressl` and ensure it's on PATH.
- **Apple certificate + provisioning profile**: ask the user to provide the paths (see Step 5b).

### Reporting prerequisite failures

If any prerequisites are missing or misconfigured:

1. List all failures clearly — one per line, with what's wrong.
2. For each failure, give the fix command or instruction.
3. Ask the user:
   > Some prerequisites are missing (listed above). Would you like me to try installing/configuring them, or would you prefer to set them up yourself?

4. If the user says **yes, try for me**: attempt the fixes that are safe to automate (yarn/expo global install, setting env vars in the current shell). **Do not retry more than once** — environment issues like Android SDK path conflicts, Java version conflicts, or Gradle version mismatches are complex. If the automated fix fails, tell the user clearly:
   > The automated setup didn't work. React Native local builds have many environment dependencies that can conflict. Please fix the prerequisite manually: [instruction]. Come back once it's working.
   Then **stop**. Do not loop.

5. If the user says **no, I'll fix them myself**: list the exact steps needed and exit.

---

## Step 4 — Workflow A: Sync / Web-preview

### 4a. Get the preview URL

If the user hasn't provided a preview URL, ask:

> Please provide your WaveMaker Studio preview URL.
> It looks like: `https://<your-studio-domain>/studio/...` or the direct app preview link ending in the project name and branch ID.
> You can find it in Studio by clicking the Preview button.

### 4b. Choose sync mode

Ask if not already clear:

> Do you want to:
> 1. **sync** — generate the Expo project locally so you can run it on a device/emulator with `npm start`
> 2. **web-preview** — open the app directly in a browser tab

### 4c. Run the command

**sync:**
```bash
wm-reactnative-ai sync <previewUrl>
```

**web-preview (browser, esbuild bundler):**
```bash
wm-reactnative-ai run web-preview <previewUrl> --esbuild
```

> **Note for web-preview without --esbuild**: Omitting `--esbuild` uses the Expo Metro bundler path (`runWeb` in `web-preview-launcher.js`) which requires additional setup. Default to `--esbuild` unless the user specifically asks for Metro.

### 4d. Handle authentication prompt

During `sync` or `web-preview`, the CLI will **pause and ask for an auth token**. When it does, guide the user:

> The CLI needs to authenticate with your WaveMaker Studio.
> 1. Open this URL in your browser (while logged into Studio): `<studioBaseUrl>/studio/services/auth/token`
> 2. Copy the token value from the page.
> 3. Paste it here and press Enter.
>
> (This token lets the CLI download your project from Studio. It's stored locally for future runs.)

The studio base URL is the origin of the preview URL (e.g. if preview URL is `https://studio.example.com/studio/preview/...`, the token URL is `https://studio.example.com/studio/services/auth/token`).

### 4e. After sync completes

The CLI will print the path to the generated Expo project. Tell the user:

> Your Expo project is ready at: `~/.wm-reactnative-cli/wm-projects/<projectName>/target/generated-expo-app/`
>
> To run it:
> ```bash
> cd <path-above>
> npm start
> ```
> Then scan the QR code with the Expo Go app on your device, or press `a` for Android emulator / `i` for iOS simulator.
>
> The CLI is watching Studio for changes — any save in Studio will automatically re-sync and update your local project.

### 4f. `--clean` flag

If the user is having issues or wants a fresh start, add `--clean` to remove the cached project:
```bash
wm-reactnative-ai sync <previewUrl> --clean
```

---

## Step 5 — Workflow B: Build Android APK/AAB

### 5a. Get the source

The source is either:
- A `.zip` file exported from WaveMaker Studio (Settings → Export → React Native)
- An already-extracted folder containing `wm_rn_config.json`

If not provided, ask:
> Please provide the path to your exported WaveMaker React Native zip or the folder path.

Verify the source has `wm_rn_config.json`:
```bash
ls "<src_path>/wm_rn_config.json" 2>/dev/null || ls "<src_path>"/*.zip 2>/dev/null || echo "NOT_WM_RN_PROJECT"
```

If it doesn't look like a WaveMaker RN project, stop and tell the user.

### 5b. Collect signing options

For a **debug build** (default): no keystore needed.

For a **production/release build**, ask:
> For a production build, I need your Android signing details:
> - Path to your `.keystore` file
> - Store password
> - Key alias
> - Key password

### 5c. Run the build

**Debug APK (no keystore):**
```bash
wm-reactnative-ai build android "<src_path>" --buildType=debug --auto-eject=true
```

**Production APK with signing:**
```bash
wm-reactnative-ai build android "<src_path>" \
  --buildType=production \
  --auto-eject=true \
  --aKeyStore="<path-to.keystore>" \
  --aStorePassword="<store-password>" \
  --aKeyAlias="<alias>" \
  --aKeyPassword="<key-password>"
```

**AAB (bundle for Play Store):**
```bash
wm-reactnative-ai build android "<src_path>" \
  --buildType=production \
  --packageType=bundle \
  --auto-eject=true \
  --aKeyStore="<path-to.keystore>" \
  --aStorePassword="<store-password>" \
  --aKeyAlias="<alias>" \
  --aKeyPassword="<key-password>"
```

Always pass `--auto-eject=true` to avoid the interactive eject confirmation prompt.

**Destination**: by default the build lands in `~/.wm-reactnative-cli/build/<appId>/<version>/android/<n>/`. If the user wants a specific output location, add `--dest="<path>"`.

### 5d. Specific architectures

If the user targets specific devices (e.g. emulator only or specific ABI):
```bash
--architecture=arm64-v8a  # modern physical devices
--architecture=x86_64     # Android emulator (Intel/Apple Silicon via Rosetta)
```

### 5e. After build completes

The CLI prints the artifact path and file size. Tell the user:
> Your APK/AAB is at: `<printed-path>`
>
> To install on a connected Android device:
> ```bash
> adb install "<path-to.apk>"
> ```
> For Play Store submission, use the `.aab` bundle file.

---

## Step 6 — Workflow B: Build iOS IPA

### 6a. Verify macOS

iOS builds require a Mac. If `uname -s` is not `Darwin`, stop immediately:
> iOS builds can only run on macOS. This machine is running <OS>. Please run this on a Mac.

### 6b. Get the source

Same as Android Step 5a — zip or extracted folder with `wm_rn_config.json`.

**Important**: Do not have an iPhone or iPad connected via USB during the build — it can interfere with Xcode. Remind the user.

### 6c. Collect certificate details

Ask:
> For an iOS build, I need:
> - Path to your `.p12` certificate file (developer or distribution)
> - Certificate password (to unlock the .p12)
> - Path to your `.mobileprovision` provisioning profile
> - Build type: **development** (run on registered devices) or **production** (App Store distribution)

### 6d. Run the build

```bash
wm-reactnative-ai build ios "<src_path>" \
  --iCertificate="<path-to.p12>" \
  --iCertificatePassword="<password>" \
  --iProvisioningFile="<path-to.mobileprovision>" \
  --buildType=<development|production> \
  --auto-eject=true
```

### 6e. After build completes

> Your IPA is at: `<printed-path>`
>
> For development: install via Xcode Devices window or `ios-deploy`.
> For production: upload to App Store Connect via Xcode Organizer or `xcrun altool`.

---

## Step 7 — Troubleshooting common issues

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Auth fails / "Not able to login" | Expired token or wrong Studio session | Open token URL in a **logged-in** Studio session tab and copy again |
| "failed to download the project" | Studio preview not active | Keep the Studio preview tab open while syncing |
| `cmake` path too long (Windows) | Long path to CLI | Set `WM_REACTNATIVE_CLI` env var to a short path (e.g. `C:\cli\`) |
| Gradle build fails with Java version error | JAVA_HOME pointing to wrong JDK | Set `JAVA_HOME` to Java 17 path specifically |
| Pod install fails | Xcode Command Line Tools not installed | Run `xcode-select --install` |
| `openssl` errors on iOS | System OpenSSL instead of LibreSSL | `brew install libressl && brew link libressl --force` |
| `expo prebuild` fails | Node version mismatch | Use Node 22.x specifically |
| Module not found after sync | `npm install` incomplete | `cd <expo-project-dir> && npm install` manually |

---

## Things NOT to do

- Do not run `sync` and `build` for the same project at the same time — they use different pipelines.
- Do not attempt to fix WaveMaker Studio project code to make the build pass — surface the error and stop.
- Do not push IPAs or APKs to app stores — that is the user's action.
- Do not loop on prerequisite installation failures — try once, and if it fails, instruct the user and exit.
- Do not modify `wm_rn_config.json` or any generated project files.
- If the user is on Windows and targeting iOS, stop immediately — it's not supported.
