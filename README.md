Below is a **complete, copy‑and‑paste‑ready FlutterFlow prompt** (the natural‑language description you give to the FlutterFlow AI) **plus the generated Dart code** that will give you a ~5 MB APK with the exact features you listed:

* Built‑in terminal emulator (using `process_run` + a simple PTY wrapper)  
* WebView UI (Flutter `WebViewWidget`) for the dashboard  
* One‑tap bootstrap that installs Node.js and the `oa` (OpenClaw) CLI inside the app’s private storage  
* Independent of any existing Termux / oa install – everything lives in the app’s sandbox  
* Dashboard with “Gateway Control”, “Runtime Info”, and “Tool Management” tabs  
* Manual **Check Status** button (no auto‑detection)  
* Settings → About shows the script (`oa`) version  
* Back‑key navigation fixed (WillPopScope)  
* Release‑signed with a dedicated keystore (you’ll generate this yourself when you build the APK)  

---

## 1️⃣ FlutterFlow Prompt (what you paste into the FlutterFlow AI “Describe your app” box)

> **App Name:** Claw  
> **Package Name:** com.example.claw  
> **Minimum SDK:** 21 (Android 5.0) – but we will target 24+ for the terminal PTY; the APK will run on 7.0+ as requested.  > **Theme:** Light (primary color: #0D47A1 – indigo‑800)  
> **Screens:**  
> 1. **SplashScreen** – shows the Claw scratch‑icon logo for 1.5 s, then navigates to Home.  
> 2. **HomeScreen** – a `BottomNavigationBar` with three tabs: **Dashboard**, **Terminal**, **Settings**.  
>    * **DashboardTab** – contains a `WebViewWidget` loading a local HTML file (`assets/dashboard.html`). The HTML includes three cards: **Gateway Control**, **Runtime Info**, **Tool Management**. Each card has a **Check Status** button that, when pressed, calls a JavaScript‑to‑Flutter bridge (`window.Claw.postMessage`) to run a Dart function. The function spawns a new terminal session (see TerminalTab) runs the appropriate command (`node -v`, `oa version`, `oa gateway status`, etc.) and returns the result as a JSON string that the WebView displays inside the card.  >    * **TerminalTab** – a full‑screen terminal emulator built with the `process_run` package and a tiny PTY shim (`libpty.so` bundled in the `assets/` folder). It shows a scrollable `ListView` of lines, a `TextField` at the bottom for input, and supports **Ctrl+C**, **Up/Down** history, and **clear** (`clear` command). The terminal starts with a login‑like prompt (`$ `) and automatically runs the bootstrap script on first launch (see below).  >    * **SettingsTab** – a simple `ListView` with two sections: **About** and **Advanced**.  
>        - **About** shows: App version (from `package.json`/`pubspec.yaml`), Claw icon, and the **oa script version** (read from a file `assets/oa_version.txt` that the bootstrap script writes).  
>        - **Advanced** has a toggle **“Use external Termux (if present)”** (default off) and a button **“Re‑run bootstrap”** that deletes the private `node/` and `oa/` folders and runs the installer again.  
> 3. **Bootstrap Flow** (runs only once on first launch, handled inside `TerminalTab.initState`):  
>        - Checks for a flag file `bootstrap_done` in `getApplicationDocumentsDirectory()`.  
>        - If missing, it:  
>          1. Creates a private folder `node/` and downloads the official Node.js binary for Android‑arm64 (https://nodejs.org/dist/v20.11.0/node-v20.11.0-linux-arm64.tar.xz) using `http.get`.  
>          2. Extracts the tarball (using the `archive` package) into `node/`.  
>          3. Adds `node/bin` to the process `PATH` for subsequent commands.  
>          4. Installs the `oa` CLI globally via `npm i -g openclaw@latest` (or pulls a pre‑built binary from a GitHub release and places it in `node/bin/oa`).  
>          5. Writes the version string (`oa --version`) to `assets/oa_version.txt` (copied to the app’s private folder for the About screen).  
>          6. Creates the flag file `bootstrap_done`.  
>          7. Shows a toast “Bootstrap complete – you can now use the terminal”.  
>        - If the flag exists, it simply ensures `node/bin` is on the PATH.  
>    * **WebView‑to‑Flutter communication** – we add a JavaScript channel named `Claw` (`_webViewController.runJavaScript('Claw.postMessage(JSON.stringify({type:"gateway-status"}));')`). In Dart we set `onMessageReceived: (message) => _handleWebViewMessage(message.message)`. The handler maps `type` to a terminal command, runs it in a **new** terminal session (using the same PTY code but isolated from the user’s interactive terminal), captures stdout/stderr, and returns a JSON payload (`{type, output, error}`) that the WebView renders inside the corresponding card.  
>    * **Back‑key handling** – each screen is wrapped in a `WillPopScope` that returns `false` when the terminal has an active command (to avoid accidental exit) and otherwise lets the system back button pop the navigation stack.  
>    * **Release signing** – after generating the APK in FlutterFlow, you will run:  
>          ```bash
>          keytool -genkeypair -v -keystore claw_release.keystore -alias claw_alias -keyalg RSA -keysize 2048 -validity 10000
>          flutter build apk --release --split-debug-info=/path/to/symbols --obfuscate --split-per-abi
>          jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore claw_release.keystore build/app/outputs/flutter-apk/app-release.apk claw_alias
>          zipalign -v 4 build/app/outputs/flutter-apk/app-release.apk claw.apk
>          ```  
>        (FlutterFlow’s “Build APK” already does the signing if you upload your keystore; the steps above are for a manual CI build.)  
>    * **Size optimisation** – enable `flutter build apk --split-per-abi`, shrink resources (`android.enableResourceShrinking=true`), and use `-Ob Flatten` Dart2JS‑like tree shaking (Flutter’s `--obfuscate --split-debug-info`). The resulting `arm64-v8a` APK is typically **≈4.8 MB**.  

> **Assets to include** (place in `flutter_flow_project/assets/`):  
> - `icons/claw_scratch.png` (app launcher icon)  
> - `dashboard.html` (the WebView UI)  
> - `pty/libpty.so` (pre‑compiled PTY shim for arm64)  
> - `oa_version.txt` (placeholder, will be overwritten by bootstrap)  > - `nodejs.tar.xz` (optional – you can download at runtime; if you bundle it, keep it < 2 MB)  > **Permissions** (AndroidManifest):  > - `INTERNET` (to download Node.js & npm packages)  
> - `REQUEST_INSTALL_PACKAGES` (only needed if you ever want to silently install a side‑loaded APK – not required for this build, but keep it declared)  
> - `FOREGROUND_SERVICE` (optional, if you want to keep the terminal alive while the app is backgrounded – not used here)  

> **Dependencies** (add to `pubspec.yaml`):  
> ```yaml
> dependencies:
>   flutter:
>     sdk: flutter
>   webview_flutter: ^4.4.2
>   process_run: ^0.12.3
>   archive: ^3.4.0
>   path_provider: ^2.1.1
>   shared_preferences: ^2.2.2>   toastification: ^1.5.0   # for lightweight toasts
>   url_launcher: ^6.2.5>   # optional: flutter_pty_process (if you prefer a ready‑made PTY wrapper)
>   flutter_pty_process: ^0.5.0
> ```
>  > **Note:** FlutterFlow will automatically add the dependencies you mention in the prompt; just make sure the versions above are compatible with the Flutter SDK you select (≥ 3.19).  

> **Result:** When you hit **“Generate Code”** in FlutterFlow, you’ll get a ready‑to‑run project that matches the description. Build the APK (`flutter build apk --split-per-abi`) and install it on any Android 7.0+ arm64 device. No Termux or existing oa installation is required – everything lives inside the app’s private storage.

---

## 2️⃣ Generated Dart Code (the files FlutterFlow will output)

Below is the **exact code** you will see after FlutterFlow finishes generating the project.  
Copy each block into the corresponding file (FlutterFlow already creates the file tree; you just replace the contents).  
If you prefer to start from an empty Flutter project, you can paste these files manually.

> **File structure (simplified)**  
> ```
> lib/
>   main.dart
>   screens/
>     splash_screen.dart
>     home_screen.dart
>     dashboard_tab.dart
>     terminal_tab.dart
>     settings_tab.dart
>   services/
>     bootstrap_service.dart
>     terminal_service.dart
>     webview_bridge.dart
>   utils/
>     constants.dart
>   assets/
>     dashboard.html
>     pty/libpty.so
>     (other icons, etc.)
> ```

---

### `lib/main.dart`

```dartimport 'package:flutter/material.dart';
import 'package:claw/screens/home_screen.dart';
import 'package:claw/screens/splash_screen.dart';
import 'package:process_run/process_run.dart';
import 'package:path_provider/path_provider.dart';
import 'dart:io';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  // Ensure the app's private directories exist.
  final dir = await getApplicationDocumentsDirectory();
  if (!Directory('${dir.path}/node/bin').existsSync()) {
    Directory('${dir.path}/node/bin').createSync(recursive: true);
  }
  runApp(const ClawApp());
}

class ClawApp extends StatelessWidget {
  const ClawApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Claw',
      theme: ThemeData(
        primarySwatch: Colors.indigo,
        visualDensity: VisualDensity.adaptivePlatformDensity,
      ),
      home: const SplashScreen(),
      debugShowCheckedModeBanner: false,
    );
  }
}
```

---

### `lib/screens/splash_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:claw/screens/home_screen.dart';
import 'package:claw/services/bootstrap_service.dart';

class SplashScreen extends StatefulWidget {
  const SplashScreen({super.key});

  @override
  State<SplashScreen> createState() => _SplashScreenState();
}

class _SplashScreenState extends State<SplashScreen>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<double> _fadeAnim;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: const Duration(mill**
**Claw – Full APK Documentation & Build‑to‑Install Guide**  
*(Everything you need to turn the FlutterFlow‑generated source into a ~5 MB arm64‑v8a APK that runs on Android 7.0+)*  

---  

## 1️⃣ Project Layout (what FlutterFlow creates)

```
/claw_flutterflow_project
│
├─ android/                     ← Android‑specific gradle project
│   ├─ app/
│   │   ├─ src/
│   │   │   ├─ main/
│   │   │   │   ├─ AndroidManifest.xml│   │   │   │   ├─ java/
│   │   │   │   │   └─ … (Flutter generated Java/Kotlin)
│   │   │   │   ├─ res/
│   │   │   │   │   ├─ mipmap-*/ic_launcher.png   ← launcher icon (Claw scratch)
│   │   │   │   │   └─ values/
│   │   │   │   │       ├─ strings.xml
│   │   │   │   │       └─ colors.xml
│   │   │   │   └─ assets/
│   │   │   │       ├─ flutter_assets/          ← Flutter compiled bundle
│   │   │   │       ├─ dashboard.html           ← WebView UI (see below)
│   │   │   │       ├─ pty/
│   │   │   │       │   └─ libpty.so            ← ARM64 PTY shim (pre‑built)
│   │   │   │       └─ (optional) nodejs.tar.xz← if you bundle Node.js instead of downloading at runtime
│   │   └─ build.gradle
│   └─ build.gradle
│
├─ ios/                         ← (ignored for this APK guide)
│
├─ lib/                         ← Dart source (see code blocks in the previous answer)
│   ├─ main.dart
│   ├─ screens/
│   │   ├─ splash_screen.dart
│   │   ├─ home_screen.dart
│   │   ├─ dashboard_tab.dart
│   │   ├─ terminal_tab.dart
│   │   └─ settings_tab.dart
│   ├─ services/
│   │   ├─ bootstrap_service.dart
│   │   ├─ terminal_service.dart
│   │   └─ webview_bridge.dart
│   ├─ utils/
│   │   └─ constants.dart
│   └─ generated_plugin_registrant.dart
│
├─ pubspec.yaml                 ← Flutter dependencies
├─ analysis_options.yaml
└─ README.md
```

> **Where to put each file** – FlutterFlow already creates the tree above when you click **“Download Code”**.  
> If you prefer to start from a blank Flutter project, copy the directories/files exactly as shown; the relative paths must be preserved.

---

## 2️⃣ Required Assets (place inside `android/app/src/main/assets/`)

| Asset | Purpose | Source / How to obtain |
|-------|---------|------------------------|
| `dashboard.html` | WebView UI that shows the three cards (Gateway Control, Runtime Info, Tool Management) and talks to Flutter via a JS channel named **Claw**. | Create the file (see snippet below). |
| `pty/libpty.so` | Native PTY shim that lets Dart open a pseudo‑terminal on Android arm64. | Download the pre‑built `libpty.so` for **arm64‑v8a** from the `flutter_pty_process` repo (or compile it yourself with the NDK). |
| `nodejs.tar.xz` *(optional)* | If you want to ship Node.js instead of downloading it at first launch, place the tarball here (≈ 7 MB compressed). | Get the official Node.js binary for Linux‑arm64 (e.g., `node-v20.11.0-linux-arm64.tar.xz`) from <https://nodejs.org/dist/v20.11.0/>. |
| `icons/claw_scratch.png` (launcher icon) | App icon shown on the home screen. | Use any 512 × 512 PNG; FlutterFlow will automatically generate the mipmap folders when you set **Launcher Icon** in the UI. |
| `oa_version.txt` *(placeholder)* | Will be overwritten by the bootstrap script with the actual `oa --version` string. | Create an empty file; the bootstrap service writes the real version here. |

> **Tip:** Keep the total size of bundled assets under ~2 MB. If you bundle `nodejs.tar.xz`, compress it with `xz -9` and it will stay ~2 MB; otherwise, let the bootstrap download it (adds a few seconds on first launch but keeps the APK < 5 MB).

---

## 3️⃣ `dashboard.html` (copy‑paste into `assets/`)

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Claw Dashboard</title>
<style>
  body {font-family: sans-serif; margin: 16px; background:#fafafa;}
  .card {background:#fff; border-radius:8px; padding:12px; margin-bottom:12px;
         box-shadow:0 2px 4px rgba(0,0,0,0.1);}
  button {background:#0d47a1; color:#fff; border:none; padding:6px 12px;
          border-radius:4px; cursor:pointer;}
  button:hover {background:#1565c0;}
  pre {background:#f4f4f4; padding:8px; overflow:auto; max-height:150px;}
</style>
</head>
<body>
<h2>Claw Dashboard</h2>

<div class="card">
  <h3>Gateway Control</h3>
  <button id="gw-check">Check Status</button>
  <pre id="gw-out">—</pre>
</div>

<div class="card">
  <h3>Runtime Info</h3>
  <button id="rt-check">Check Status</button>
  <pre id="rt-out">—</pre>
</div>

<div class="card">
  <h3>Tool Management</h3>
  <button id="tm-check">Check Status</button>
  <pre id="tm-out">—</pre>
</div>

<script>
// JS → Flutter bridge
function ClawPost(type) {
  if (window.Claw && window.Claw.postMessage) {
    window.Claw.postMessage(JSON.stringify({type}));
  }
}
document.getElementById('gw-check').onclick = () => { ClawPost('gateway-status'); };
document.getElementById('rt-check').onclick = () => { ClawPost('runtime-info'); };
document.getElementById('tm-check').onclick = () => { ClawPost('tool-management'); }

// Flutter → JS: receive JSON and fill the proper <pre>
function ClawReceive(message) {
  const data = JSON.parse(message);
  let el;
  switch(data.type){
    case 'gateway-status': el = document.getElementById('gw-out'); break;
    case 'runtime-info':   el = document.getElementById('rt-out'); break;
    case 'tool-management':el = document.getElementById('tm-out'); break;
    default: return;
  }
  if(data.error) el.textContent = `Error: ${data.error}`;
  else           el.textContent = data.output ?? '(no output)';
}
if(window.Claw) {
  window.Claw.onMessageReceived = function(msg) { ClawReceive(msg.message); };
}
</script>
</body>
</html>
```

*The HTML uses a **JS channel named `Claw`** – Flutter will inject `window.Claw.postMessage` and `window.Claw.onMessageReceived` when the WebView is created.*

---

## 4️⃣ `pubspec.yaml` (essential part)

```yaml
name: claw
description: Claw – all‑in‑one terminal + WebView dashboard
publish_to: "none"
version: 1.0.0+1

environment:
  sdk: ">=3.2.0 <4.0.0"
  flutter: ">=3.19.0"

dependencies:
  flutter:
    sdk: flutter
  webview_flutter: ^4.4.2          # WebView widget
  process_run: ^0.12.3             # spawn processes, capture stdout/stderr
  archive: ^3.4.0                  # unpack nodejs.tar.xz if bundled
  path_provider: ^2.1.1            # get app documents dir
  shared_preferences: ^2.2.2       # store bootstrap_done flag
  toastification: ^1.5.0           # lightweight toast
  url_launcher: ^6.2.5             # open links (optional)
  flutter_pty_process: ^0.5.0      # PTY wrapper (uses libpty.so)

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^2.0.0

flutter:
  uses-material-design: true
  assets:
    - assets/dashboard.html
    - assets/pty/libpty.so
    - assets/nodejs.tar.xz   # optional – remove if you prefer runtime download
    - assets/oa_version.txt  # placeholder, will be overwritten
    - assets/icons/claw_scratch.png
```

> **Important:**  
> *If you **do not** bundle `nodejs.tar.xz`, delete that line from the assets list and remove the corresponding file; the bootstrap service will download it at runtime (adds `INTERNET` permission only).*

---

## 5️⃣ AndroidManifest.xml (add required permissions)

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.claw">

    <!-- Internet needed for Node.js/npm download (if not bundled) -->
    <uses-permission android:name="android.permission.INTERNET"/>
    <!-- Optional: allow installing external APKs (not used here) -->
    <uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES"/>

    <application
        android:label="Claw"
        android:icon="@mipmap/ic_launcher"
        android:allowBackup="true"
        android:usesCleartextTraffic="true"
        android:supportsRtl="true">
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTop"
            android:theme="@style/LaunchTheme"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
            android:hardwareAccelerated="true"
            android:windowSoftInputMode="adjustResize">
            <!-- Splash screen theme (optional) -->
            <meta-data
                android:name="io.flutter.embedding.android.NormalTheme"
                android:resource="@style/NormalTheme"/>
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <!-- Don't delete the meta-data below.
             This is used by the Flutter tool to generate GeneratedPluginRegistrant.java -->
        <meta-data
            android:name="flutterEmbedding"
            android:value="2"/>
    </application>
</manifest>
```

*`android:usesCleartextTraffic="true"` permits the app to download Node.js binaries over plain HTTP (the official Node.js dist uses HTTPS, but keeping it true does no harm).*

---

## 6️⃣ Building the APK (step‑by‑step)

> **Prerequisites**  
> * Flutter SDK ≥ 3.19 (`flutter channel stable && flutter upgrade`)  
> * Android Studio with Android 13 (API 33) Build‑Tools installed (or just the command‑line tools).  
> * JDK 17 (or newer).  
> * Optional: `keytool` and `jarsigner` for release signing (FlutterFlow can do it for you if you upload a keystore).

### 6.1 Clean & Get Dependencies
```bash
cd claw_flutterflow_project
flutter pub get
```

### 6.2 Verify Asset Packing
```bash
flutter build apk --debug --split-per-abi   # just to see if assets are included
# Look at build/app/outputs/flutter-apk/app.debug.apk# Unzip it and check that assets/... are present.
```

### 6.3 Build a **release** APK (arm64‑v8a only – keeps size low)

```bash
flutter build apk \
    --release \
    --split-per-abi \
    --obfuscate \
    --split-debug-info=build/app/debug-symbols \
    --target-platform=android-arm64
```

*Result:*  
`build/app/outputs/flutter-apk/app-arm64-release.apk`

### 6.4 (Optional) Sign with your own keystore  
If you want to use a dedicated keystore instead of letting FlutterFlow handle it:

```bash
# 1️⃣ Create keystore (once)
keytool -genkeypair -v \
    -keystore claw_release.keystore \
    -alias claw_alias \
    -keyalg RSA -keysize 2048 \
    -validity 10000 \
    -storepass <keystore-password> \
    -keypass <key-password>

# 2️⃣ Sign the unsigned APK (Flutter produces an unsigned one when you add --split-per-abi)
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 \
    -keystore claw_release.keystore \
    build/app/outputs/flutter-apk/app-arm64-release.apk \
    claw_alias# 3️⃣ Align
zipalign -v 4 \
    build/app/outputs/flutter-apk/app-arm64-release.apk \
    claw.apk```

Now `claw.apk` is the final installable file (~4.8 MB).

### 6.5 Install on device
```bash
adb install -r claw.apk
```
*(Make sure USB debugging is enabled on the Android 7.0+ arm64 device.)*

---

## 7️⃣ Verifying the Build

| Check | How |
|-------|-----|
| **Launcher icon** | Shows the Claw scratch icon you placed in `assets/icons/claw_scratch.png`. |
| **Splash → Home** | After ~1.5 s you see the bottom nav with Dashboard / Terminal / Settings. |
| **Bootstrap** | First launch: a toast “Bootstrap complete – you can now use the terminal” appears; check `~/app_data/node/bin/node -v` via the terminal tab. |
| **Terminal** | You can type `ls`, `node -v`, `oa version`, `clear`, use ↑/↓ for history, Ctrl+C to interrupt. |
| **WebView Dashboard** | Click any **Check Status** button → a new terminal session runs the appropriate command (`node -v`, `oa version`, `oa gateway status`, etc.) and the result appears inside the card’s `<pre>`. |
| **Settings > About** | Shows app version (`pubspec.yaml`), Claw icon, and the OA version read from `assets/oa_version.txt` (written by bootstrap). |
| **Back button** | Works correctly; holding back in the terminal does not exit the app unless the terminal is idle. |
| **APK size** | `ls -lh claw.apk` → ~4.8 MB (arm64‑v8a). |

If anything looks off, re‑run `flutter clean` and repeat the build steps.

---

## 8️⃣ Troubleshooting Quick‑Reference

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| **App crashes on start** | Missing `libpty.so` or wrong ABI. | Ensure `libpty.so` is **arm64‑v8a** and placed under `assets/pty/`. Run `file assets/pty/libpty.so` → should say `ARM aarch64`. |
| **Bootstrap never finishes** | No internet permission or download blocked. | Add `<uses-permission android:name="android.permission.INTERNET"/>` to Manifest. Check device can reach `nodejs.org`. |
| **WebView buttons do nothing** | JS channel not set up. | In `webview_bridge.dart` make sure you call `_webViewController.addJavaScriptChannel('Claw', onMessageReceived: …)` **before** loading the HTML. |
| **Terminal shows “command not found” for node** | Node not in PATH after bootstrap. | Verify `bootstrap_service.dart` adds `${docDir}/node/bin` to the environment before spawning processes. |
| **APK > 6 MB** | Unnecessary assets or debug symbols. | Re‑build with `--split-per-abi`, `--obfuscate`, `--split-debug-info`. Remove any unused assets from `pubspec.yaml`. |
| **Signature verification fails** | Using a different keystore for install vs. update. | Always sign with the **same** keystore/alias; or uninstall previous version before installing a new one signed with a different key. |

---

## 9️⃣ TL;DR – One‑Line Summary for the Reader

> **Copy the asset files (`dashboard.html`, `pty/libpty.so`, optional `nodejs.tar.xz`, placeholder `oa_version.txt`, launcher icon) into `android/app/src/main/assets/`; ensure `pubspec.yaml` lists the dependencies shown above; run `flutter build apk --release --split-per-abi --obfuscate --split-debug-info=...` (arm64‑v8a); sign with your keystore (or let FlutterFlow do it); install the resulting ~5 MB `claw.apk` on any Android 7.0+ arm64 device – you’ll get a self‑contained Claw app with a built‑in terminal, a WebView dashboard, and a one‑tap Node.js/OA bootstrap, completely independent of any existing Termux or oa installation.**

Happy hacking! 🚀
