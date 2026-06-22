# Android App Development with Android Studio — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I have never built an Android app" to "I ship a maintainable, modern, Compose-first Android app" — without an internet connection. Every concept is explained in prose first (what it is, the logic/why it exists, what it is used for and when, then how to use it) and only then shown with heavily-commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets the **2026** Android toolchain. Things worth knowing about the modern Android world:
> - **Android Studio** ships on the "Ladybug / Meerkat / Narwhal"-era release train (the IDE follows IntelliJ-platform yearly naming). The IDE is the official, Google-supported way to build Android apps and bundles the SDK, emulator, profilers, and an AI assistant (Gemini in Android Studio).
> - **Kotlin is the default, Google-recommended language.** New APIs, samples, and docs are Kotlin-first. Java still works but this guide is Kotlin throughout. Cross-references to **KOTLIN_GUIDE.md** are noted, but this guide is self-contained.
> - **Jetpack Compose is the modern default UI toolkit.** New apps are built with Compose (declarative Kotlin UI). The older **XML layouts + View system** is still everywhere in existing apps and is fully covered here (§5) because you will read, maintain, and sometimes write it.
> - **Material 3 (Material You)** is the current design system, with dynamic color on Android 12+.
> - **Build tooling:** **Gradle** with the **Kotlin DSL** (`build.gradle.kts`), the **Android Gradle Plugin (AGP)**, and **version catalogs** (`libs.versions.toml`) are the standard. `compileSdk`/`targetSdk` are around **35–36** (Android 15 "VanillaIceCream" / Android 16 "Baklava"). Google Play requires apps to target a recent API level, so this moves every year.
> - **Coroutines and `Flow`** are the standard concurrency model; **ViewModel + StateFlow + MVVM + a repository layer** is the recommended architecture; **Room** is the recommended local database; **Retrofit/OkHttp** the standard networking stack; **Hilt** the standard dependency-injection framework.
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (`gradlew.bat` vs `./gradlew`, path separators, emulator/HAXM-vs-WHPX) are called out. Confirm exact APIs at developer.android.com.

---

## Table of Contents

1. [The Android Platform & Android Studio](#1-the-android-platform--android-studio) **[B]**
2. [Project & Folder Structure](#2-project--folder-structure) **[B]**
3. [The Gradle Build System](#3-the-gradle-build-system) **[B/I]**
4. [App Fundamentals: Activities, Lifecycle, Fragments, Intents](#4-app-fundamentals-activities-lifecycle-fragments-intents) **[B/I]**
5. [The Layout Editor & XML Layouts (the View system)](#5-the-layout-editor--xml-layouts-the-view-system) **[B/I]**
6. [Jetpack Compose — the Modern Default UI](#6-jetpack-compose--the-modern-default-ui) **[B/I/A]**
7. [State & Architecture: ViewModel, StateFlow, MVVM, Hilt](#7-state--architecture-viewmodel-stateflow-mvvm-hilt) **[I/A]**
8. [Lists: RecyclerView and LazyColumn](#8-lists-recyclerview-and-lazycolumn) **[I]**
9. [Data & Persistence: DataStore, Room, Files](#9-data--persistence-datastore-room-files) **[I]**
10. [Networking: Retrofit, OkHttp, Serialization](#10-networking-retrofit-okhttp-serialization) **[I/A]**
11. [Concurrency: Coroutines & Flow in Android](#11-concurrency-coroutines--flow-in-android) **[I/A]**
12. [Resources & Assets](#12-resources--assets) **[B/I]**
13. [Permissions](#13-permissions) **[I]**
14. [Navigation](#14-navigation) **[I]**
15. [Debugging & Tooling](#15-debugging--tooling) **[I]**
16. [Testing](#16-testing) **[I/A]**
17. [Building & Publishing](#17-building--publishing) **[I/A]**
18. [Gotchas & Best Practices](#18-gotchas--best-practices) **[A]**
19. [Study Path & Build-to-Learn Projects](#19-study-path--build-to-learn-projects)

---

## 1. The Android Platform & Android Studio

### 1.1 What an Android app actually is **[B]**

Before touching the IDE, understand what you are building. An **Android app** is a package — historically an `.apk` (Android Package), now usually published as an `.aab` (Android App Bundle) — that contains:

- **Compiled code.** Your Kotlin/Java is compiled to **JVM bytecode** (`.class`), then to **DEX** (Dalvik Executable) bytecode that runs on the **Android Runtime (ART)**. ART ahead-of-time / just-in-time compiles DEX to native machine code on the device.
- **Resources** — images, layouts, strings, colors, fonts (the `res/` folder, §2, §12).
- **The manifest** (`AndroidManifest.xml`) — a declaration of what the app is, what components it has, and what permissions it needs (§2).
- **Native libraries** (optional `.so` files for C/C++).

The logic of why it is packaged this way: Android runs on a huge range of devices with different CPUs (ARM, ARM64, x86), screen densities, and OS versions. The system needs a declarative description (the manifest + resource qualifiers) so it can install, lay out, and launch your app correctly on each device without running your code first. An App Bundle lets Google Play generate a tailored APK for each device (only the right CPU, the right screen density), shrinking download size.

An app does **not** have a `main()` you call. Instead the OS launches one of your **components** (an `Activity`, `Service`, etc., §4) in response to an **Intent** (a request, §4). The framework owns the lifecycle; you write callbacks the framework calls. This inversion of control is the single most important mental shift coming from desktop or backend programming.

Every app runs in its own **sandbox**: its own Linux user ID, its own process, its own private data directory. Apps can only talk to each other through well-defined channels (Intents, content providers, permissions). This is the security model.

### 1.2 The Android Studio IDE **[B]**

**What it is:** Android Studio is the official IDE, built on **IntelliJ IDEA**. It is what you use for everything: editing code, designing UI (both the XML Layout Editor and Compose previews), managing the SDK, running emulators, debugging, and profiling.

**Why use it over a plain editor:** it bundles and wires together the Android SDK, the build system (Gradle), the emulator, the debugger (`adb`-based), code completion that understands Android APIs, the Layout Editor, Logcat, and the profilers. Doing Android development without it is possible but painful.

**Key windows you will live in:**

| Window | Shortcut (Win/Linux) | What it does |
|---|---|---|
| **Project** | `Alt+1` | File/folder tree of your project (see §2) |
| **Editor** | — | Where you write code; tabs across the top |
| **Logcat** | `Alt+6` | Live device/emulator logs (`Log.d` output, crashes) |
| **Run** | `Shift+F10` | Build + install + launch on device/emulator |
| **Debug** | `Shift+F9` | Run with the debugger attached |
| **Build / Build Output** | — | Gradle build progress and errors |
| **Gradle** | — | List of Gradle tasks you can run |
| **Device Manager** | — | Create/start emulators (AVDs) and see connected devices |
| **SDK Manager** | — | Install SDK platforms, build-tools, emulator images |
| **Profiler** | — | CPU/memory/network/energy profiling |
| **Layout Inspector** | — | Inspect a running UI's view/composable tree |

**⚡ Version note:** Android Studio also bundles **Gemini in Android Studio**, an AI assistant for code generation, explanation, and crash analysis. It is optional and account-gated.

### 1.3 The SDK Manager **[B]**

**What it is:** the **Android SDK** is the collection of platform libraries, build tools, and supporting packages your app compiles and runs against. The **SDK Manager** (Tools → SDK Manager, or the toolbar icon) installs and updates these.

**What you install and why:**

- **SDK Platforms** — the `android.jar` and APIs for a given Android version. You need the platform matching your `compileSdk` (the version you compile against, §3). Install the latest (e.g. API 35/36) plus any you specifically target.
- **SDK Build-Tools** — the tools that turn your code/resources into an APK (`aapt2`, `d8`/`r8`, `zipalign`, `apksigner`). Gradle picks a version automatically.
- **SDK Platform-Tools** — `adb` (Android Debug Bridge) and `fastboot`. `adb` is how the IDE talks to devices/emulators.
- **Android Emulator** + **System Images** — the emulator engine and the OS images (e.g. "API 35, Google APIs, x86_64") it runs.
- **Command-line Tools** — `sdkmanager`, `avdmanager` for headless/CI use.

The SDK lives at a path like `C:\Users\<you>\AppData\Local\Android\Sdk` on Windows. The environment variable is `ANDROID_HOME` (older: `ANDROID_SDK_ROOT`).

### 1.4 The emulator (AVD) vs a physical device **[B]**

You run your app on either an **emulator** or a **real device**. Understand the trade-offs:

**Emulator / AVD (Android Virtual Device):**
- **What it is:** a full virtual Android device running on your PC via hardware virtualization (Intel HAXM is deprecated; modern Windows uses the **Windows Hypervisor Platform / WHPX**, or **AEHD**; macOS uses Hypervisor.framework; Linux uses KVM).
- **Why/when:** no physical hardware needed; you can spin up any API level, screen size, or density; great for quick iteration and testing many configurations. Snapshots make cold-boot fast.
- **Downsides:** slower than real hardware for some tasks; sensors/camera/GPS are simulated; not a perfect performance proxy.
- **Create one:** Device Manager → "Create Device" → pick hardware profile (e.g. Pixel 8) → pick a system image (download if needed) → Finish.

**Physical device:**
- **Why/when:** real performance, real sensors, real touch, testing manufacturer skins. Always test on real hardware before release.
- **Setup:** on the device, enable **Developer options** (Settings → About phone → tap "Build number" 7 times), then enable **USB debugging**. Plug in via USB, accept the RSA fingerprint prompt. **Wireless debugging** (pair over Wi-Fi) is also available on Android 11+.
- The device then appears in the run-target dropdown.

```bash
# adb (platform-tools) — the bridge between your PC and devices/emulators
adb devices                 # list connected devices/emulators
adb install app-debug.apk   # install an APK manually
adb logcat                  # stream logs from the command line
adb shell                   # open a shell on the device
adb uninstall com.example.myapp
adb tcpip 5555              # switch a USB device to wireless debugging
```

### 1.5 Install & first run **[B]**

1. Install Android Studio (the bundled JDK and SDK come with it). On first launch the **Setup Wizard** downloads an SDK platform, build-tools, and an emulator image.
2. **New Project** → choose a template. For modern apps pick **"Empty Activity"** (this is the Compose template — it generates a `MainActivity` using `setContent { }`). The **"Empty Views Activity"** template is the XML/View-system one.
3. Set **Name**, **Package name** (reverse-DNS, e.g. `com.yourname.myapp` — this is your app's unique ID), **Save location**, **Language: Kotlin**, **Minimum SDK** (the oldest Android you support, e.g. API 24 = Android 7.0; lower = more devices but more compatibility work).
4. Wait for **Gradle Sync** (downloads dependencies, configures the project — see §3).
5. Pick a device in the toolbar dropdown, press **Run** (`Shift+F10` / the green ▶). Android Studio builds, installs, and launches.

### 1.6 The build/run cycle — what actually happens **[B/I]**

When you press Run, this pipeline executes (orchestrated by Gradle + AGP):

1. **Gradle configuration** — Gradle reads your `build.gradle.kts` files and builds a task graph.
2. **Compile resources** — `aapt2` compiles `res/` into binary form and generates the **`R` class** (§2.9) so code can reference resources by ID.
3. **Compile code** — the Kotlin compiler turns `.kt` into JVM `.class` bytecode (and runs annotation processors / KSP for Room, Hilt, etc.).
4. **Dex** — `d8`/`r8` converts bytecode to **DEX**; on release, **R8** also shrinks/obfuscates (§3, §17).
5. **Package** — everything is zipped into an APK; resources, DEX, manifest, native libs.
6. **Sign** — the APK is signed (debug key automatically; release key for publishing, §17).
7. **Install** — `adb install` pushes it to the device.
8. **Launch** — `adb` starts the launcher Activity.

**Incremental builds** mean only changed parts recompile. **⚡ Live Edit** (Compose) and **Apply Changes** can push code/UI changes without a full reinstall for fast iteration.

---

## 2. Project & Folder Structure

This section is detailed on purpose: understanding the folder layout is the difference between fumbling and fluency. An Android Studio project is a **Gradle multi-project build**. The top level is the *project*; inside it are one or more *modules* (the main one is conventionally `app`).

### 2.1 Top-level project layout **[B]**

```
MyApp/                          ← project root (open THIS folder in Android Studio)
├── app/                        ← the main application MODULE (your code lives here)
│   ├── build.gradle.kts        ← MODULE build script (deps, SDK versions, app config)
│   ├── proguard-rules.pro      ← R8/ProGuard shrinking & obfuscation rules
│   └── src/                    ← all source for this module
│       ├── main/               ← the main source set (production code + resources)
│       ├── test/               ← local unit tests (run on your PC's JVM)        (§16)
│       └── androidTest/        ← instrumented tests (run on a device/emulator)  (§16)
├── build.gradle.kts            ← PROJECT-level build script (plugins, repos)
├── settings.gradle.kts         ← declares which modules + dependency repositories
├── gradle/
│   ├── libs.versions.toml      ← the VERSION CATALOG (central dependency versions) (§3)
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties  ← which Gradle version this project uses
├── gradlew                     ← Gradle wrapper script (macOS/Linux)
├── gradlew.bat                 ← Gradle wrapper script (Windows)  ← USE THIS on Win11
├── gradle.properties           ← project-wide Gradle/JVM/AndroidX flags
└── local.properties            ← machine-specific paths (sdk.dir); NOT in version control
```

**The logic:** the project root is the unit you version-control and open. The `app` module is where your application lives; larger apps split into many modules (`:feature:login`, `:core:network`, etc.) for build speed and separation. The Gradle files configure *how* it all builds; `src/` holds *what* builds.

### 2.2 Inside `app/src/main/` — the main source set **[B]**

```
app/src/main/
├── AndroidManifest.xml         ← declares the app & its components (see §2.8)
├── kotlin/  (or java/)         ← your Kotlin/Java source, organized into PACKAGES
│   └── com/yourname/myapp/
│       ├── MainActivity.kt
│       ├── ui/                 ← (your convention) screens, composables, themes
│       ├── data/               ← (your convention) repositories, Room, network
│       └── ...
├── res/                        ← RESOURCES (compiled, addressable by R.* — see §2.6)
│   ├── layout/                 ← XML layouts (View system)            (§5)
│   ├── drawable/               ← images & vector drawables            (§12)
│   ├── mipmap-*/               ← launcher icons (density buckets)     (§2.7, §12)
│   ├── values/                 ← strings.xml, colors.xml, themes.xml, dimens.xml
│   ├── font/                   ← bundled fonts
│   ├── menu/                   ← XML menu definitions
│   ├── xml/                    ← misc XML (backup rules, network config, prefs)
│   └── raw/                    ← raw files kept as-is (e.g. a bundled JSON/audio)
└── assets/                     ← raw files accessed by path/stream (see §2.10)
```

> **⚡ Note on `java/` vs `kotlin/`:** the source folder is historically named `java/` even for Kotlin code, and that still works. Newer projects may use a `kotlin/` folder. Both are valid "source roots"; the **package** (the `com.yourname.myapp` directory chain) is what matters for imports — it must match your `package` declarations.

### 2.3 What a "package" is and why folders mirror it **[B]**

A **package** is a namespace for your code, written reverse-DNS (`com.yourname.myapp`). On disk it is a chain of folders: `com/yourname/myapp/`. A file declaring `package com.yourname.myapp.ui` must live in `.../kotlin/com/yourname/myapp/ui/`. The package keeps class names unique and organizes code. The **applicationId** (in `build.gradle.kts`) is the app's unique identity on the device and Play Store; it often equals the package but is technically independent (you can change one without the other).

### 2.4 The three source sets: `main`, `test`, `androidTest` **[B/I]**

- **`main`** — your real app code + resources. Ships in the APK.
- **`test`** — **local unit tests** (JUnit) that run on your development machine's JVM. Fast; no Android device. Use for pure logic (ViewModels, repositories with fakes). (§16)
- **`androidTest`** — **instrumented tests** that run on a device/emulator (Espresso, Compose UI tests, Room DB tests). Slower but exercise real Android APIs. (§16)

Build variants can add more source sets (e.g. `src/debug/`, `src/free/`) — files there override/merge with `main` for that variant (§3.4).

### 2.5 `res/` vs `assets/` — a crucial distinction **[B]**

Both hold non-code files, but they behave very differently:

| | `res/` | `assets/` |
|---|---|---|
| Access | By generated ID: `R.drawable.logo`, `getString(R.string.app_name)` | By path/stream: `assets.open("data/config.json")` |
| Processing | Compiled & optimized by `aapt2`; supports **resource qualifiers** (density, locale, night mode) | Stored raw, untouched |
| Use for | Anything the framework selects by configuration (layouts, icons, strings, themes) | Arbitrary files you read yourself (large JSON, fonts for a custom loader, ML models, web content) |
| Subfolders | Fixed names (`layout`, `drawable`, `values`...); no arbitrary nesting | Any nested folder structure you like |

**Why two systems:** `res/` gives you automatic, configuration-aware selection (the system hands you the French strings on a French device, the high-density icon on a high-density screen). `assets/` is an escape hatch for raw bytes when you want full control.

### 2.6 The `res/values/` files **[B]**

These XML files define *named* resources. The folder is `values/`; the filename is conventional, not magic (you could put everything in one file, but splitting is clearer).

```xml
<!-- res/values/strings.xml — all user-facing text (enables localization, §12) -->
<resources>
    <string name="app_name">My App</string>
    <string name="greeting">Hello, %1$s!</string>   <!-- %1$s is a format placeholder -->
    <plurals name="item_count">
        <item quantity="one">%d item</item>
        <item quantity="other">%d items</item>
    </plurals>
</resources>
```

```xml
<!-- res/values/colors.xml — named colors referenced as R.color.* / @color/* -->
<resources>
    <color name="purple_500">#FF6200EE</color>
    <color name="black">#FF000000</color>
</resources>
```

```xml
<!-- res/values/themes.xml — the app's theme (default styling for the whole app) -->
<resources>
    <style name="Theme.MyApp" parent="Theme.Material3.DayNight.NoActionBar">
        <item name="colorPrimary">@color/purple_500</item>
    </style>
</resources>
```

```xml
<!-- res/values/dimens.xml — named dimensions (reuse spacing/sizes consistently) -->
<resources>
    <dimen name="padding_default">16dp</dimen>
    <dimen name="text_title">20sp</dimen>
</resources>
```

You reference these from XML as `@string/app_name`, `@color/purple_500`, `@dimen/padding_default`, `@style/Theme.MyApp`, and from Kotlin as `R.string.app_name`, `getString(R.string.greeting, userName)`, etc.

### 2.7 `drawable/` vs `mipmap/` **[B]**

- **`drawable/`** — general images and vector drawables used inside your UI. Has density variants (`drawable-mdpi`, `-hdpi`, `-xhdpi`, ...) so the right resolution is picked (§12).
- **`mipmap-*/`** — specifically the **launcher icon**. It is separated from `drawable` because launchers may display an icon at a higher density than the device's own (so the system keeps all densities even after density-stripping). Modern apps use **adaptive icons** (`mipmap-anydpi-v26/ic_launcher.xml` referencing foreground + background layers, §12).

### 2.8 `AndroidManifest.xml` — what it declares and why **[B, important]**

**What it is:** the app's manifest — a single XML file the OS reads *before* running any of your code. It is the contract between your app and the system.

**Why it exists:** the OS must know, ahead of time, what your app is, which components it has (so it can launch them on demand via Intents), which permissions it needs (so it can prompt the user), the minimum hardware/features required, and which Activity is the entry point. None of this can be discovered by "running" the app — it has to be declared.

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- PERMISSIONS your app needs. INTERNET is install-time; others (CAMERA,
         location) are "dangerous" and require a runtime request too (§13). -->
    <uses-permission android:name="android.permission.INTERNET" />

    <!-- Optional hardware/feature requirements. required=false = nice-to-have. -->
    <uses-feature android:name="android.hardware.camera" android:required="false" />

    <application
        android:name=".MyApplication"          <!-- optional custom Application class -->
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"      <!-- launcher icon (from mipmap, §2.7) -->
        android:label="@string/app_name"        <!-- app name (from strings.xml) -->
        android:theme="@style/Theme.MyApp">     <!-- app-wide theme (from themes.xml) -->

        <!-- Each component your app exposes must be declared here. -->
        <activity
            android:name=".MainActivity"
            android:exported="true">            <!-- exported=true: other apps/launcher can start it -->

            <!-- An intent-filter advertises what intents this Activity handles.
                 This one (MAIN + LAUNCHER) marks it as the app's ENTRY POINT,
                 so it shows in the launcher and is started when you tap the icon. -->
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- Other components are also declared here when present:
             <service>, <receiver> (BroadcastReceiver), <provider> (ContentProvider) -->

    </application>
</manifest>
```

Key points:
- **`exported`** is mandatory (and security-critical) on any component with an intent-filter since Android 12. `true` lets other apps start it; default to `false` unless you intend it to be public.
- The `MAIN`/`LAUNCHER` intent-filter is what makes an Activity the launchable entry point.
- AGP **merges** your manifest with the manifests of your libraries (manifest merging) into one final manifest in the APK.

### 2.9 The generated `R` class **[B/I]**

**What it is:** `R` is an auto-generated Kotlin/Java class (one per app + per library) holding integer IDs for every resource. When `aapt2` compiles `res/`, it emits `R` with nested objects: `R.string.app_name`, `R.drawable.logo`, `R.id.button_submit`, `R.layout.activity_main`, `R.color.purple_500`.

**Why:** code references resources by a stable compile-time constant, not a string path. This gives you autocompletion and compile-time checking — a typo'd `R.string.greting` won't compile. It is generated, so you never edit it; if it's "red" (unresolved), the resource is missing or the project failed to compile its resources.

```kotlin
// Using R from code:
val name: String = getString(R.string.app_name)            // resolve a string resource
val greeting = getString(R.string.greeting, "Ada")         // with a format arg -> "Hello, Ada!"
imageView.setImageResource(R.drawable.logo)                // set a drawable
setContentView(R.layout.activity_main)                     // inflate an XML layout (View system)
val color = ContextCompat.getColor(this, R.color.purple_500)
```

### 2.10 Reading from `assets/` **[I]**

```kotlin
// Read a bundled file from assets/ by path. Returns an InputStream you manage.
val json = context.assets.open("data/config.json").bufferedReader().use { it.readText() }
// 'use { }' auto-closes the reader (Kotlin's try-with-resources equivalent).
```

### 2.11 `gradle.properties` & `local.properties` **[I]**

```properties
# gradle.properties — project-wide build flags (committed to VCS)
org.gradle.jvmargs=-Xmx2048m            # heap for the Gradle daemon
org.gradle.parallel=true                # build independent modules in parallel
org.gradle.caching=true                 # reuse outputs across builds
android.useAndroidX=true                # use AndroidX (the modern support libs)
kotlin.code.style=official
```

```properties
# local.properties — MACHINE-SPECIFIC; never commit (it's in .gitignore)
sdk.dir=C\:\\Users\\Phoniex\\AppData\\Local\\Android\\Sdk
```

---

## 3. The Gradle Build System

### 3.1 What Gradle and AGP are **[B/I]**

**What it is:** **Gradle** is the build automation tool Android uses. It is configured by scripts (`build.gradle.kts` in Kotlin DSL, or `.gradle` in Groovy — Kotlin DSL is the modern default and what this guide uses). The **Android Gradle Plugin (AGP)** teaches Gradle how to build Android apps (compile resources, dex, package, sign).

**Why a build system at all:** an Android build is dozens of ordered steps with hundreds of inputs (your code, resources, dozens of library dependencies, multiple build variants, signing). Gradle expresses this as a dependency graph of tasks, caches outputs, builds incrementally, and downloads dependencies automatically. You declare *what* you want; Gradle figures out *how*.

**The wrapper (`gradlew`/`gradlew.bat`):** every project pins a specific Gradle version via the wrapper. You run `./gradlew` (Unix) or `gradlew.bat` (Windows) — it downloads the exact Gradle version the project expects. This means everyone (and CI) builds with the same Gradle, regardless of what's installed globally. **Always use the wrapper.**

### 3.2 The two build scripts **[B/I]**

**Project-level `build.gradle.kts`** — declares plugins available to modules and (sometimes) repositories:

```kotlin
// build.gradle.kts (project root)
plugins {
    // Declared here with `apply false` so versions are defined once; modules opt in.
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.kotlin.compose) apply false   // Compose compiler plugin (Kotlin 2.0+)
}
```

**Module-level `app/build.gradle.kts`** — the one you edit most. Configures the Android build and dependencies:

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.android.application)   // this module IS an app (vs library)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)        // enables Jetpack Compose
}

android {
    namespace = "com.yourname.myapp"   // the package for the generated R class & BuildConfig
    compileSdk = 36                     // API level you COMPILE against (use the latest)

    defaultConfig {
        applicationId = "com.yourname.myapp" // unique app ID on device/Play (§2.3)
        minSdk = 24                          // OLDEST Android you support (API 24 = 7.0)
        targetSdk = 36                       // API level you've TESTED against (behavior opt-in)
        versionCode = 1                      // integer, MUST increase every Play release (§17)
        versionName = "1.0.0"                // human-readable version shown to users

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = true            // run R8: shrink + obfuscate (§17)
            isShrinkResources = true          // strip unused resources
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            // signingConfig = signingConfigs.getByName("release")  // §17
        }
        // 'debug' exists by default: not minified, debuggable, uses the debug signing key.
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }   // (or the kotlin{} jvmToolchain block in newer setups)

    buildFeatures {
        compose = true                    // turn on Compose for this module
        // viewBinding = true             // turn on View Binding (§5) if using XML
        // buildConfig = true             // generate BuildConfig with custom fields
    }
}

dependencies {
    // ---- Compose (managed by the BOM so all Compose libs share one version) ----
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.ui)
    implementation(libs.androidx.material3)
    implementation(libs.androidx.activity.compose)
    implementation(libs.androidx.lifecycle.viewmodel.compose)

    // ---- Core ----
    implementation(libs.androidx.core.ktx)

    // ---- Testing ----
    testImplementation(libs.junit)                       // local unit tests
    androidTestImplementation(libs.androidx.junit)       // instrumented tests
    androidTestImplementation(platform(libs.androidx.compose.bom))
    androidTestImplementation(libs.androidx.ui.test.junit4)
    debugImplementation(libs.androidx.ui.tooling)        // Compose preview tooling (debug only)
}
```

**`compileSdk` vs `targetSdk` vs `minSdk` — the most-confused trio:**
- **`compileSdk`** — which API's classes/methods you can *call at compile time*. Use the latest. Doesn't affect runtime behavior.
- **`minSdk`** — the *oldest* OS the app installs/runs on. Lower = more devices, but you can't use newer APIs unconditionally (you must guard with `Build.VERSION.SDK_INT` checks).
- **`targetSdk`** — "I have tested against this API level and opt into its behavior changes." The OS uses this to decide whether to apply newer behavior or back-compat shims. Google Play requires a recent `targetSdk` for updates.

### 3.3 Dependencies & the version catalog (`libs.versions.toml`) **[B/I]**

**What it is:** a centralized TOML file declaring all dependency versions, library coordinates, and plugins in one place. Modules reference them as `libs.something`.

**Why:** before catalogs, versions were scattered as string literals across many `build.gradle` files — easy to drift, hard to bump. The catalog gives one source of truth, type-safe accessors (`libs.androidx.core.ktx`) with autocompletion, and easy upgrades.

```toml
# gradle/libs.versions.toml

[versions]                                    # named version numbers
agp = "8.7.0"
kotlin = "2.0.21"
coreKtx = "1.15.0"
composeBom = "2024.10.00"
activityCompose = "1.9.3"
lifecycle = "2.8.7"
retrofit = "2.11.0"
room = "2.6.1"
hilt = "2.52"
junit = "4.13.2"
androidxJunit = "1.2.1"

[libraries]                                   # group:name pinned to a version
androidx-core-ktx       = { group = "androidx.core", name = "core-ktx", version.ref = "coreKtx" }
androidx-compose-bom    = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
androidx-ui             = { group = "androidx.compose.ui", name = "ui" }              # version from BOM
androidx-ui-tooling     = { group = "androidx.compose.ui", name = "ui-tooling" }
androidx-ui-test-junit4 = { group = "androidx.compose.ui", name = "ui-test-junit4" }
androidx-material3      = { group = "androidx.compose.material3", name = "material3" }
androidx-activity-compose = { group = "androidx.activity", name = "activity-compose", version.ref = "activityCompose" }
androidx-lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycle" }
retrofit                = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
room-runtime            = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-compiler           = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
hilt-android            = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler           = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }
junit                   = { group = "junit", name = "junit", version.ref = "junit" }
androidx-junit          = { group = "androidx.test.ext", name = "junit", version.ref = "androidxJunit" }

[plugins]                                     # Gradle plugins
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android      = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose      = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
hilt                = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
ksp                 = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

**Dependency configurations** (the keyword before each dependency) control *scope*:

| Configuration | Meaning |
|---|---|
| `implementation` | Available to this module at compile + runtime; NOT leaked to modules depending on this one (faster builds). The default choice. |
| `api` | Like `implementation` but *does* leak to consumers (use sparingly in library modules). |
| `compileOnly` | Available at compile time only (e.g. annotations). Not packaged. |
| `runtimeOnly` | Packaged but not on the compile classpath. |
| `testImplementation` | Only for `src/test` (local unit tests). |
| `androidTestImplementation` | Only for `src/androidTest` (instrumented tests). |
| `debugImplementation` | Only for the `debug` build type. |
| `ksp` / `annotationProcessor` | Code-generation processors (Room, Hilt). KSP is the modern, faster Kotlin Symbol Processing. |

### 3.4 Build types & product flavors (build variants) **[I]**

**Build types** describe *how* you build (debug vs release). **Product flavors** describe *what* you build (free vs paid, dev vs prod backend). A **build variant** is the cross-product: `freeDebug`, `paidRelease`, etc.

**Why:** ship multiple versions of the same app from one codebase with different config, applicationIds, resources, or code (via per-variant source sets, §2.4).

```kotlin
android {
    flavorDimensions += "tier"              // a dimension groups mutually-exclusive flavors
    productFlavors {
        create("free") {
            dimension = "tier"
            applicationIdSuffix = ".free"   // -> com.yourname.myapp.free (separate install)
            versionNameSuffix = "-free"
            buildConfigField("boolean", "PREMIUM", "false")  // readable as BuildConfig.PREMIUM
        }
        create("paid") {
            dimension = "tier"
            buildConfigField("boolean", "PREMIUM", "true")
        }
    }
}
// Variant-specific code/resources go in src/free/, src/paid/, src/freeDebug/, etc.
```

### 3.5 Common Gradle tasks **[B/I]**

```bash
# On Windows use gradlew.bat ; on macOS/Linux use ./gradlew
gradlew.bat tasks                 # list available tasks
gradlew.bat assembleDebug         # build the debug APK -> app/build/outputs/apk/debug/
gradlew.bat assembleRelease       # build the release APK
gradlew.bat bundleRelease         # build the release AAB (for Play, §17)
gradlew.bat installDebug          # build + install debug on the connected device
gradlew.bat test                  # run local unit tests
gradlew.bat connectedAndroidTest  # run instrumented tests on a device/emulator
gradlew.bat lint                  # run Android Lint static analysis
gradlew.bat clean                 # delete build/ outputs
gradlew.bat dependencies          # print the dependency tree (debug version conflicts)
gradlew.bat --refresh-dependencies assembleDebug   # force re-download deps
```

> **Gotcha:** if a build behaves strangely, **File → Invalidate Caches / Restart** in the IDE and `gradlew.bat clean` often fixes it. "Gradle sync" must succeed before code completion works.

### 3.6 The Gradle build lifecycle — what "sync" and "configure" mean **[I]**

It helps to know the three Gradle phases, because error messages reference them:

1. **Initialization** — Gradle reads `settings.gradle.kts` and decides which modules (projects) are in the build.
2. **Configuration** — Gradle evaluates every `build.gradle.kts`, building the task graph. The `android { }` and `dependencies { }` blocks run here. Slow configuration = slow every-build overhead; this is why you avoid heavy logic in build scripts.
3. **Execution** — Gradle runs the tasks you asked for (`assembleDebug`, etc.), skipping any whose inputs/outputs are unchanged (incremental + cached).

**"Gradle Sync"** inside the IDE is essentially the configuration phase: it re-reads your build scripts so the IDE knows your modules, dependencies, and source sets (and thus can offer completion and resolve `R`). You must sync after editing any `build.gradle.kts`, the version catalog, or `settings.gradle.kts` — the IDE shows a "Sync Now" banner.

```kotlin
// settings.gradle.kts — initialization phase: declare repositories + which modules exist.
pluginManagement {
    repositories { google(); mavenCentral(); gradlePluginPortal() }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)   // repos centralized here
    repositories { google(); mavenCentral() }
}
rootProject.name = "MyApp"
include(":app")                        // the modules in this build
// include(":core:network", ":feature:login")   // a multi-module app would list more
```

### 3.7 BuildConfig and reading build values from code **[I]**

When `buildConfig = true`, AGP generates a `BuildConfig` class per variant with constants you can read at runtime — useful for switching behavior between debug/release or flavors without scattering checks.

```kotlin
if (BuildConfig.DEBUG) { /* extra logging only in debug builds */ }
// Custom fields you defined per build type/flavor (§3.4):
val baseUrl = if (BuildConfig.PREMIUM) "https://api.example.com/" else "https://free.example.com/"
```

---

## 4. App Fundamentals: Activities, Lifecycle, Fragments, Intents

### 4.1 The four app components **[B]**

Android apps are built from four kinds of **components**, each a class the system can instantiate:

| Component | What it is / when |
|---|---|
| **Activity** | A single screen with a UI. The primary building block. Your `MainActivity` is one. |
| **Service** | Background work with no UI (e.g. music playback). Modern apps prefer `WorkManager`/coroutines for most background work. |
| **BroadcastReceiver** | Responds to system-wide events (broadcasts), e.g. "battery low", "boot completed". |
| **ContentProvider** | Exposes structured data to other apps (e.g. contacts). |

All are declared in the manifest (§2.8). This section focuses on **Activities** (the screen) since that's where UI lives.

### 4.2 The Activity & `setContent`/`setContentView` **[B]**

**What an Activity is:** a screen. The system creates it, gives it a window, and drives it through a lifecycle. You subclass it and tell it what UI to show.

```kotlin
// MainActivity.kt — a Compose-based Activity (the modern default template)
class MainActivity : ComponentActivity() {           // ComponentActivity supports Compose
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {                                 // hand Compose the UI for this screen
            MyAppTheme {                             // apply Material 3 theme (§6, §12)
                Greeting(name = "Android")
            }
        }
    }
}

// The XML/View-system equivalent would be:
//   class MainActivity : AppCompatActivity() {
//       override fun onCreate(savedInstanceState: Bundle?) {
//           super.onCreate(savedInstanceState)
//           setContentView(R.layout.activity_main)   // inflate the XML layout (§5)
//       }
//   }
```

### 4.3 The Activity lifecycle — when each callback fires and why it matters **[B, critical]**

**The logic:** the OS, not you, decides when your screen appears, is backgrounded, or is destroyed (a phone call arrives, the user rotates the device, memory runs low). The framework signals these transitions through **lifecycle callbacks**. If you do the right work in the right callback, your app behaves correctly; if not, you leak memory, lose data, or waste battery.

```
            onCreate()    ← created (set up UI, do one-time init)
                ↓
            onStart()     ← becoming visible
                ↓
            onResume()    ← in the foreground, interactive  ───┐
                ↑                                              │ (app is running here)
            onPause()     ← losing focus (another screen on top)
                ↓                                              ↑ (back to foreground)
            onStop()      ← no longer visible (backgrounded) ──┘
                ↓
            onDestroy()   ← being destroyed (finished or reclaimed)
```

| Callback | Fires when | Do here |
|---|---|---|
| `onCreate(bundle)` | Activity first created | One-time setup: `setContent`, read `savedInstanceState`, get ViewModel. |
| `onStart()` | Becoming visible | Start things tied to visibility (register a listener). |
| `onResume()` | Now interactive/foreground | Start camera preview, resume animations, acquire exclusive resources. |
| `onPause()` | Losing focus (dialog/another activity) | **Quickly** release exclusive resources; *don't* save heavy state here (it blocks the next screen). Brief. |
| `onStop()` | No longer visible | Persist data, stop heavy work (location, sensors), unregister listeners. |
| `onDestroy()` | Finishing or being killed | Final cleanup. Not guaranteed to run if the process is killed. |

**Why it matters in practice:** rotating the device (a **configuration change**) destroys and recreates the Activity (`onDestroy` → `onCreate`). UI state in plain fields is lost. This is exactly why **`ViewModel`** exists (it survives config changes, §7) and why you use `rememberSaveable` in Compose. Acquiring resources in `onResume` and releasing in `onPause`/`onStop` prevents battery drain and leaks (§18).

```kotlin
class MyActivity : ComponentActivity() {
    override fun onCreate(s: Bundle?) { super.onCreate(s); Log.d("LC", "onCreate") }
    override fun onStart()   { super.onStart();   Log.d("LC", "onStart") }
    override fun onResume()  { super.onResume();  Log.d("LC", "onResume") }
    override fun onPause()   { super.onPause();   Log.d("LC", "onPause") }
    override fun onStop()    { super.onStop();    Log.d("LC", "onStop") }
    override fun onDestroy() { super.onDestroy(); Log.d("LC", "onDestroy") }
    // Watch Logcat (§15) as you rotate/background to SEE the sequence.
}
```

### 4.4 Saving transient UI state **[I]**

For small UI state that must survive process death (not just rotation), use `onSaveInstanceState`:

```kotlin
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    outState.putInt("scroll", currentScroll)   // small key/value state only
}
// Read it back in onCreate from savedInstanceState?.getInt("scroll")
```
In Compose, `rememberSaveable { mutableStateOf(...) }` does this for you (§6). For larger data, persist to Room/DataStore (§9).

### 4.5 Fragments — what and when **[I]**

**What it is:** a `Fragment` is a reusable, modular piece of UI with its own lifecycle, hosted inside an Activity. Multiple fragments can share one Activity.

**Why/when:** in the View system, fragments enable reusable UI chunks, master/detail layouts on tablets, and are the unit the **Navigation component** (§14) moves between (single-Activity architecture: one Activity hosting many fragment destinations). 

**⚡ With Compose, you usually do NOT use fragments** — navigation-compose moves between composable destinations directly. Learn fragments because you will encounter them in existing/XML-based apps; for new Compose apps, prefer a single Activity + Compose navigation.

```kotlin
// A minimal Fragment that inflates an XML layout.
class ProfileFragment : Fragment(R.layout.fragment_profile) {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        // Use view binding (§5) here, not the Activity's. Fragment view lifecycle
        // is SHORTER than the Fragment's own — null out bindings in onDestroyView.
    }
}
```

### 4.6 Intents — explicit, implicit, passing data **[B/I]**

**What it is:** an **Intent** is a message object describing an operation to perform — "start this screen", "share this text", "open this URL". It's the glue that lets components (and apps) cooperate.

**Explicit intent** — you name the exact component (typically your own Activity):

```kotlin
// From an Activity: navigate to DetailActivity, passing data via "extras".
val intent = Intent(this, DetailActivity::class.java).apply {
    putExtra("ITEM_ID", 42)                 // attach data as key/value extras
    putExtra("ITEM_NAME", "Widget")
}
startActivity(intent)                       // the system creates & starts DetailActivity

// In DetailActivity.onCreate, read the extras:
val id = intent.getIntExtra("ITEM_ID", -1)        // -1 is the default if missing
val name = intent.getStringExtra("ITEM_NAME")
```

**Implicit intent** — you describe an *action*; the system finds an app that can handle it:

```kotlin
// "Share this text" — the system shows a chooser of capable apps.
val share = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "Check out this app!")
}
startActivity(Intent.createChooser(share, "Share via"))

// "Open this web page" — hands off to a browser.
val view = Intent(Intent.ACTION_VIEW, Uri.parse("https://developer.android.com"))
// Guard: only start if SOMETHING can handle it, else it crashes.
if (view.resolveActivity(packageManager) != null) startActivity(view)
```

**Getting a result back** — the modern, lifecycle-safe way is the **Activity Result API** (replaces the deprecated `startActivityForResult`):

```kotlin
// Register a launcher (must be done before the Activity is STARTED, e.g. as a field).
private val pickImage = registerForActivityResult(
    ActivityResultContracts.GetContent()        // a built-in contract: pick content
) { uri: Uri? ->
    if (uri != null) { /* use the chosen image Uri */ }
}
// Later, launch it:
pickImage.launch("image/*")
```

### 4.7 The back stack & tasks **[B/I]**

**What it is:** when you `startActivity`, the new Activity is pushed onto a **back stack** within a **task**. Pressing **Back** (or the system back gesture) pops the top Activity, returning to the previous one. When the stack empties, you leave the app.

**Why it matters:** the back stack is the user's navigation history. Getting it right (not duplicating screens, handling Up vs Back, surviving from notifications) is core UX. **`launchMode`** and intent flags tune this:

- `singleTop` — if the target is already on top, reuse it (no duplicate).
- `FLAG_ACTIVITY_CLEAR_TOP` — pop everything above an existing instance.
- In Compose, **navigation-compose** (§14) manages an analogous back stack of composable destinations.

> **⚡ Predictive back:** Android 14+ has a predictive back gesture (a peek of where Back will take you). Opt in with `android:enableOnBackInvokedCallback="true"` and use the modern `OnBackPressedDispatcher` / Compose `BackHandler`.

---

## 5. The Layout Editor & XML Layouts (the View system)

This is the **classic View system**: UI described in XML, inflated into a tree of `View` objects. New apps use Compose (§6), but XML is everywhere in existing code, and the user explicitly wants it covered. Understanding both makes you fluent.

### 5.1 What an XML layout is and how it maps to objects **[B]**

**What it is:** an XML file in `res/layout/` describing a tree of UI widgets (`View`s) and containers (`ViewGroup`s). Each XML element is a Kotlin/Java class; each attribute sets a property.

**The logic / how it maps:** at runtime the **`LayoutInflater`** reads the XML and constructs the corresponding objects — `<TextView .../>` becomes a `TextView` instance, `android:text="Hi"` calls `setText("Hi")`. `setContentView(R.layout.activity_main)` triggers this. So XML is just a declarative way to build a view tree you could also build in code.

```xml
<!-- res/layout/activity_main.xml -->
<?xml version="1.0" encoding="utf-8"?>
<!-- The ROOT ViewGroup. ConstraintLayout is the recommended flexible container. -->
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"      
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/titleText"          
        android:layout_width="wrap_content"  
        android:layout_height="wrap_content"
        android:text="@string/app_name"      
        android:textSize="20sp"
        app:layout_constraintTop_toTopOf="parent"      
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />   

</androidx.constraintlayout.widget.ConstraintLayout>
```

### 5.2 The Layout Editor: Design, Code & Split views **[B]**

Open any `res/layout/*.xml` and the **Layout Editor** appears with three modes (toggle top-right):

- **Code** — the raw XML. Fast for precise edits and when you know exactly what attribute you want.
- **Design** — a visual, WYSIWYG canvas. Drag widgets from the **Palette**, drop them, draw constraints by hand. Best for learning and for laying out constraints visually.
- **Split** — XML and the visual canvas side by side; edits in one update the other live. The recommended everyday mode.

**Editor panels:**
- **Palette** (top-left) — the catalog of widgets/containers to drag onto the canvas.
- **Component Tree** (lower-left) — the hierarchy of views as a tree; click to select, drag to reparent. Essential for tangled layouts where clicking the canvas is fiddly.
- **Attributes panel** (right) — every attribute of the selected view. The wrench-style top section shows common attributes; expand "All Attributes" for the full list. This is where you set `id`, text, size, margins, constraints, colors.
- **Toolbar** — device/orientation/API/theme preview switchers (preview your layout on different screens without running it), plus alignment and "infer/clear constraints" tools.

**How to use it (typical flow):** drag a widget from the Palette onto the canvas → it appears in the Component Tree → use the Attributes panel to set its `id` and properties → draw constraints (drag from the round handles on the widget's edges to parent edges or sibling edges) so it's positioned. Switch to Split to confirm the generated XML.

### 5.3 `dp` and `sp` units **[B]**

- **`dp` (density-independent pixels)** — a virtual unit that scales with screen density so UI is the same *physical* size on a low-density and high-density screen. Use `dp` for all sizes, margins, paddings. 1dp = 1px on a 160-dpi "baseline" screen.
- **`sp` (scale-independent pixels)** — like `dp` but ALSO scales with the user's font-size accessibility setting. Use `sp` for **text sizes only**, so users who enlarge fonts get larger text.
- Raw `px` — avoid; it's device-dependent and breaks across screens.

### 5.4 View IDs **[B]**

`android:id="@+id/titleText"` declares a new ID named `titleText`. The `+` means "create this ID in `R.id`". Code then finds the view via that ID (`R.id.titleText`). IDs must be unique within an inflated layout and are how view binding and `findViewById` connect XML to code.

### 5.5 ConstraintLayout — constraints, chains, guidelines **[I]**

**What it is:** a flexible `ViewGroup` where you position each child by **constraints** — relationships to the parent or sibling views ("my start edge aligns to the parent's start", "my top is below view X"). 

**Why:** it builds complex, flat layouts (no deep nesting) that perform well and adapt to screen sizes. It largely replaced nested `LinearLayout`s.

**Core ideas:**
- Every view needs at least **one horizontal and one vertical constraint**, or it collapses to position (0,0) at runtime (the editor warns you).
- `app:layout_constraintStart_toStartOf="parent"` reads "constrain MY start to the START of parent." Pattern: `constraint<MySide>_to<TargetSide>Of="targetId|parent"`.
- **Bias** — when constrained on both sides, `app:layout_constraintHorizontal_bias="0.3"` shifts the view (0=start, 1=end, 0.5=center).
- **Chains** — link views with bidirectional constraints to distribute them as a group (`spread`, `spread_inside`, `packed`).
- **Guidelines** — invisible anchor lines at a fixed dp or percentage you constrain views to.
- **`0dp` (match_constraint)** — set width/height to `0dp` to mean "stretch to fill the constraints" (not zero size).

```xml
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent" android:layout_height="match_parent">

    <!-- A guideline at 50% of the width; views can constrain to it. -->
    <androidx.constraintlayout.widget.Guideline
        android:id="@+id/midline"
        android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:orientation="vertical"
        app:layout_constraintGuide_percent="0.5" />

    <EditText
        android:id="@+id/nameInput"
        android:layout_width="0dp"                     
        android:layout_height="wrap_content"
        android:hint="@string/name_hint"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toStartOf="@id/midline" />  

    <Button
        android:id="@+id/submitBtn"
        android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:text="@string/submit"
        app:layout_constraintTop_toBottomOf="@id/nameInput"  
        app:layout_constraintStart_toStartOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

### 5.6 LinearLayout & FrameLayout **[B]**

- **`LinearLayout`** — stacks children in a single row or column (`android:orientation="vertical|horizontal"`). Use `layout_weight` to distribute leftover space proportionally. Simple, but deep nesting hurts performance — prefer ConstraintLayout for complex screens.
- **`FrameLayout`** — stacks children on top of each other (z-order). Use for a single child or overlapping views (e.g. an image with a badge), and as fragment containers.

```xml
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent" android:layout_height="wrap_content"
    android:orientation="horizontal">
    <!-- Two buttons split the width 50/50 via equal weights (width=0dp + weight). -->
    <Button android:layout_width="0dp" android:layout_height="wrap_content"
            android:layout_weight="1" android:text="@string/yes" />
    <Button android:layout_width="0dp" android:layout_height="wrap_content"
            android:layout_weight="1" android:text="@string/no" />
</LinearLayout>
```

### 5.7 Common Views **[B]**

| View | Purpose |
|---|---|
| `TextView` | Display text. |
| `EditText` | Editable text input (`inputType` controls keyboard/validation). |
| `Button` / `MaterialButton` | Tappable button. |
| `ImageView` | Display an image (`src`/`setImageResource`). |
| `CheckBox`, `Switch`, `RadioButton` | Toggles. |
| `RecyclerView` | Efficient scrolling list of many items (§8). |
| `ProgressBar` | Spinner / progress. |
| `Toolbar` | App bar. |

### 5.8 View Binding — connecting XML to code safely **[I]**

**What it is:** a generated, type-safe class per layout that gives direct, null-safe references to every view with an `id`. Enable `viewBinding = true` in `build.gradle.kts` (§3.2).

**Why:** it replaces `findViewById` (which is verbose and returns loosely-typed, crash-prone references). For a layout `activity_main.xml`, AGP generates `ActivityMainBinding` with fields named after the IDs (`binding.titleText`).

```kotlin
class MainActivity : AppCompatActivity() {
    // Lazily created binding; lateinit because we set it in onCreate.
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)  // inflate the XML
        setContentView(binding.root)                           // root = the layout's top view

        binding.titleText.text = "Bound from code"             // typed access, no findViewById
        binding.submitBtn.setOnClickListener {                 // wire a click
            binding.titleText.text = "Clicked: ${binding.nameInput.text}"
        }
    }
}
```

> **Fragment gotcha:** a Fragment's view is destroyed before the Fragment, so hold the binding in a nullable backing field and set it to `null` in `onDestroyView` to avoid leaking the view (§18).

### 5.9 EditText input types & handling text **[I]**

`EditText` adapts its keyboard and validation to `android:inputType`. Pick the right one so users get the correct keyboard and the field behaves sensibly.

```xml
<EditText
    android:id="@+id/emailInput"
    android:layout_width="match_parent" android:layout_height="wrap_content"
    android:hint="@string/email_hint"
    android:inputType="textEmailAddress"   
    android:imeOptions="actionNext" />     

<EditText
    android:id="@+id/passwordInput"
    android:layout_width="match_parent" android:layout_height="wrap_content"
    android:hint="@string/password_hint"
    android:inputType="textPassword" />    
```

Common `inputType` values: `text`, `textEmailAddress`, `textPassword`, `number`, `numberDecimal`, `phone`, `textMultiLine`, `textCapSentences`. Read text in code with `binding.emailInput.text.toString()`; observe changes with `addTextChangedListener { }` (the KTX extension).

### 5.10 When to still reach for XML layouts **[I]**

Even in a Compose-first app, you may meet or need XML for: existing screens you maintain, `RemoteViews` (home-screen widgets — though Jetpack Glance now does these in Compose), custom `View`s, and embedding a Compose UI inside a View hierarchy (via `ComposeView`) or vice versa (`AndroidView` in Compose). Interop both ways is fully supported, so migration can be incremental — you do not have to rewrite an app to start using Compose.

```kotlin
// Host Compose inside an XML layout: put a <androidx.compose.ui.platform.ComposeView
//   android:id="@+id/composeRoot" .../> in XML, then:
binding.composeRoot.setContent { MyAppTheme { Greeting("from Compose inside a View") } }

// The reverse — embed a legacy View inside Compose:
@Composable fun LegacyMapView() {
    AndroidView(factory = { ctx -> MapView(ctx) }, modifier = Modifier.fillMaxSize())
}
```

---

## 6. Jetpack Compose — the Modern Default UI

### 6.1 Declarative UI: what it is and why it replaced XML for new apps **[B, foundational]**

**The old (imperative) way (View system):** you build a view tree (XML) once, then *mutate* it from code as data changes — `textView.text = newValue`, `view.visibility = GONE`. You must remember to update *every* affected view whenever state changes, and keep the UI and your data in sync by hand. This is error-prone (forgotten updates, inconsistent state) and verbose (findViewById, binding, manual diffing).

**The new (declarative) way (Compose):** you write functions that *describe* what the UI should look like **for a given state**. When the state changes, Compose re-runs the relevant functions and updates only what changed. You never imperatively poke at widgets — you change state and the UI follows. The mental model is "UI = f(state)."

**Why it replaced XML for new apps:** less boilerplate, no XML/code split, no findViewById/binding, far fewer "UI out of sync with data" bugs, powerful built-in state handling, easy custom components (just functions), and Kotlin-native (loops, conditionals, reuse are just Kotlin). It is Google's recommended toolkit for all new UI.

### 6.2 Composable functions **[B]**

**What it is:** a `@Composable` function is a UI building block. It emits UI when called and can call other composables. By convention they are named in `PascalCase` (like a class) and return `Unit`.

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp

@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    // This describes UI: a Text showing "Hello, <name>!".
    Text(text = "Hello, $name!", modifier = modifier)
}
```

### 6.3 Previews — design without running the app **[B]**

**What it is:** annotate a composable with `@Preview` and Android Studio renders it live in the editor's **Preview** pane (or Split view) — no emulator needed. This is Compose's answer to the Layout Editor's Design view.

```kotlin
@Preview(showBackground = true, name = "Greeting Light")
@Preview(showBackground = true, name = "Greeting Dark",
         uiMode = android.content.res.Configuration.UI_MODE_NIGHT_YES) // preview dark theme too
@Composable
private fun GreetingPreview() {
    MyAppTheme {                       // wrap previews in your theme for accurate colors
        Greeting(name = "Android")
    }
}
```

### 6.4 State, `remember`, `mutableStateOf`, and recomposition **[B/I, core concept]**

**The logic:** Compose tracks reads of special **state** objects. When a state's value changes, Compose schedules **recomposition** — it re-invokes the composables that *read* that state, recomputing their UI. So you make UI dynamic by holding data in state and changing it.

- **`mutableStateOf(x)`** — creates an observable state holder. Reading `.value` subscribes the current composable; writing `.value` triggers recomposition of readers.
- **`remember { }`** — caches a value across recompositions. Without it, every recomposition would re-run the initializer and reset your state. `remember` says "compute this once and keep it."
- **`by` delegate** — `var count by remember { mutableStateOf(0) }` lets you use `count` directly instead of `count.value`.
- **`rememberSaveable`** — like `remember` but ALSO survives configuration changes and process death (saved into the Bundle, §4.4). Use for UI state the user shouldn't lose on rotation.

```kotlin
@Composable
fun Counter() {
    // 'remember' keeps the state across recompositions; 'by' unwraps .value.
    var count by remember { mutableStateOf(0) }

    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("Count: $count", style = MaterialTheme.typography.headlineMedium)
        Button(onClick = { count++ }) {   // writing 'count' -> recomposition -> Text updates
            Text("Increment")
        }
    }
}
```

**Recomposition rules to internalize:** composables can run *often*, in *any order*, and even be *skipped*. Therefore: keep them free of side effects, don't rely on execution order, and never do long/blocking work in them (use `LaunchedEffect`/coroutines, §6.8). Reading a state you didn't change won't re-run you.

### 6.5 State hoisting & unidirectional data flow **[I, important]**

**What it is:** **state hoisting** means moving state *up* out of a composable, so the composable becomes **stateless** — it receives the value to show and a callback to report events. The state lives in a common ancestor (or a ViewModel, §7).

**Why:** stateless composables are reusable, testable, and previewable; a single source of truth avoids inconsistency. This is the **unidirectional data flow (UDF)** pattern: **state flows down** (as parameters), **events flow up** (as lambda callbacks). The owner of the state is the single place that changes it.

```kotlin
// STATELESS: it knows nothing about WHERE count lives; it just renders + reports clicks.
@Composable
fun CounterDisplay(count: Int, onIncrement: () -> Unit, modifier: Modifier = Modifier) {
    Column(modifier, horizontalAlignment = Alignment.CenterHorizontally) {
        Text("Count: $count")
        Button(onClick = onIncrement) { Text("Increment") }
    }
}

// STATEFUL owner: holds the state and passes value down + event up. (A ViewModel is even better, §7.)
@Composable
fun CounterScreen() {
    var count by rememberSaveable { mutableStateOf(0) }   // survives rotation
    CounterDisplay(count = count, onIncrement = { count++ })
}
```

### 6.6 Modifiers — styling and behavior **[B/I]**

**What it is:** a `Modifier` is an ordered chain of decorations applied to a composable — size, padding, background, click handling, etc. Almost every composable takes a `modifier` parameter; pass it through so callers can customize.

**Why order matters:** modifiers apply in sequence, wrapping outward. `padding(16.dp).background(Red)` paints red *inside* the padding region; `background(Red).padding(16.dp)` paints red first (full size) then pads content within. Think of each modifier as wrapping the previous result.

```kotlin
@Composable
fun ModifierDemo() {
    Text(
        "Tap me",
        modifier = Modifier
            .fillMaxWidth()              // take full width
            .padding(16.dp)              // outer padding (margin-like)
            .background(MaterialTheme.colorScheme.primaryContainer)
            .padding(12.dp)              // inner padding (between bg edge and text)
            .clickable { /* handle tap */ }
    )
}
```

### 6.7 Layouts: Column, Row, Box, LazyColumn **[B]**

- **`Column`** — children stacked vertically.
- **`Row`** — children stacked horizontally.
- **`Box`** — children stacked on top of each other (z-order); align children with `Modifier.align` / `contentAlignment`.
- **`LazyColumn` / `LazyRow`** — scrolling lists that only compose visible items (the Compose equivalent of `RecyclerView`, §8). Use these for long/unbounded lists, never a `Column` in a scroll for many items.

```kotlin
@Composable
fun LayoutShowcase() {
    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)   // gap between children
    ) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Text("Left"); Spacer(Modifier.weight(1f)); Text("Right")  // weight pushes them apart
        }
        Box(modifier = Modifier.fillMaxWidth().height(80.dp)) {
            Text("Bottom-end", modifier = Modifier.align(Alignment.BottomEnd))
        }
    }
}
```

### 6.8 Side effects: `LaunchedEffect`, `rememberCoroutineScope` **[I]**

**The logic:** since composables can recompose constantly, you can't kick off coroutines or one-time work directly in their body. **Effect APIs** scope side effects to the composition lifecycle.

- **`LaunchedEffect(key)`** — runs a suspend block when first composed (and re-runs if `key` changes); cancelled when the composable leaves. Use for one-shot loads, snackbars, animations tied to state.
- **`rememberCoroutineScope()`** — a scope tied to the composition, to launch coroutines from *event* callbacks (like a button click).
- **`DisposableEffect`** — register/cleanup pairs (listeners), with `onDispose { }`.

```kotlin
@Composable
fun UserScreen(userId: String, repo: UserRepository) {
    var user by remember { mutableStateOf<User?>(null) }

    // Re-runs only when userId changes; cancels the previous load automatically.
    LaunchedEffect(userId) {
        user = repo.fetchUser(userId)   // suspend call (coroutines, §11)
    }

    if (user == null) CircularProgressIndicator() else Text(user!!.name)
}
```

### 6.8a Text input in Compose **[B/I]**

There is no `EditText` in Compose — you use `TextField`/`OutlinedTextField`, which are **stateless**: they show whatever `value` you pass and report edits via `onValueChange`. You hold the text in your own state (the UDF pattern, §6.5). This is initially surprising to View-system developers: the field does not "own" its text.

```kotlin
@Composable
fun NameField() {
    var name by rememberSaveable { mutableStateOf("") }   // YOU own the text
    OutlinedTextField(
        value = name,                                     // what to show
        onValueChange = { name = it },                    // update state on each keystroke
        label = { Text("Name") },
        singleLine = true,
        keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
        modifier = Modifier.fillMaxWidth()
    )
    Text("Hello, ${name.ifBlank { "stranger" }}")          // recomposes as you type
}
```

### 6.8b `derivedStateOf` — computing state from state **[I]**

When a value is *derived* from other state and you want to recompute it only when its inputs change (not on every recomposition), wrap it in `derivedStateOf`. Common use: enabling a button only when input is valid, or reacting to "scrolled past the first item."

```kotlin
@Composable
fun SubmitForm() {
    var text by remember { mutableStateOf("") }
    // Recomputes ONLY when 'text' changes; readers recompose only when the boolean flips.
    val isValid by remember { derivedStateOf { text.trim().length >= 3 } }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(value = text, onValueChange = { text = it }, label = { Text("Title") })
        Button(onClick = { /* submit */ }, enabled = isValid) { Text("Submit") }
    }
}
```

### 6.9 Material 3 components & theming **[B/I]**

**What it is:** **Material 3 (Material You)** is the current design system; Compose provides ready-made M3 components (`Button`, `Card`, `Scaffold`, `TopAppBar`, `TextField`, `FloatingActionButton`, etc.) and a theming system (color scheme, typography, shapes).

**`Scaffold`** lays out the standard app structure (top bar, bottom bar, FAB, content) and supplies content padding you must apply so content isn't hidden behind bars.

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun HomeScreen() {
    Scaffold(
        topBar = { TopAppBar(title = { Text("Home") }) },
        floatingActionButton = {
            FloatingActionButton(onClick = { /* add */ }) { Icon(Icons.Default.Add, "Add") }
        }
    ) { innerPadding ->                       // padding for the bars — MUST be applied
        LazyColumn(
            modifier = Modifier.fillMaxSize().padding(innerPadding),
            contentPadding = PaddingValues(16.dp)
        ) {
            items(20) { i -> Card(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
                Text("Item $i", Modifier.padding(16.dp))
            } }
        }
    }
}
```

**Theming** — define a `MaterialTheme` wrapper (the project template generates `ui/theme/Theme.kt`):

```kotlin
@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),         // follow system dark mode
    dynamicColor: Boolean = true,                       // Material You dynamic color (Android 12+)
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        // Dynamic color pulls accent colors from the user's wallpaper (Android 12+).
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val ctx = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(ctx) else dynamicLightColorScheme(ctx)
        }
        darkTheme -> darkColorScheme()
        else -> lightColorScheme()
    }
    MaterialTheme(colorScheme = colorScheme, typography = Typography, content = content)
}
// Access theme values anywhere: MaterialTheme.colorScheme.primary, .typography.bodyLarge
```

### 6.9a Snackbars & one-off events in Compose **[I]**

State drives the UI, but some things are **events** that should happen once (show a snackbar, navigate) — not state that re-renders. The pattern: hold a `SnackbarHostState`, trigger it from a coroutine scope, and wire it into the `Scaffold`. For ViewModel-originated one-off events, expose a `SharedFlow` (not `StateFlow`) and consume it in a `LaunchedEffect`.

```kotlin
@Composable
fun MessageScreen() {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()                 // to launch the suspend show() call
    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        Button(
            onClick = { scope.launch { snackbarHostState.showSnackbar("Saved!") } },
            modifier = Modifier.padding(padding)
        ) { Text("Save") }
    }
}
```

### 6.10 Navigation-compose **[I]**

Covered in depth in §14, but here is the essence: define a `NavHost` with composable destinations addressed by route strings, and a `NavController` to navigate.

```kotlin
@Composable
fun AppNav() {
    val nav = rememberNavController()
    NavHost(navController = nav, startDestination = "home") {
        composable("home") {
            HomeScreen(onItemClick = { id -> nav.navigate("detail/$id") })
        }
        composable("detail/{id}") { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id")
            DetailScreen(id = id, onBack = { nav.popBackStack() })
        }
    }
}
```

---

## 7. State & Architecture: ViewModel, StateFlow, MVVM, Hilt

### 6.11 Building your own reusable composables & state stability **[I/A]**

Because composables are just functions, you build a component library by writing functions that take parameters and a `modifier`. Follow these conventions so your components compose well: accept a `modifier: Modifier = Modifier` (and apply it to the *outermost* element), hoist state (take `value` + `onValueChange`), and put required parameters before optional ones.

```kotlin
// A reusable labeled stat card. Stateless, modifier-accepting, previewable.
@Composable
fun StatCard(label: String, value: String, modifier: Modifier = Modifier) {
    Card(modifier = modifier) {                               // modifier applied to the root
        Column(Modifier.padding(16.dp)) {
            Text(value, style = MaterialTheme.typography.headlineSmall)
            Text(label, style = MaterialTheme.typography.bodyMedium)
        }
    }
}

@Preview @Composable
private fun StatCardPreview() = MyAppTheme { StatCard("Tasks done", "12") }
```

**Stability & skipping (performance):** Compose can *skip* recomposing a composable if all its parameters are "stable" and unchanged. Stable types include primitives, `String`, and classes Compose can prove are immutable (`data class` of stable fields, types marked `@Immutable`/`@Stable`). A `List<T>` from the standard library is treated as *unstable* (Compose can't know it won't mutate), which can defeat skipping; use `kotlinx.collections.immutable`'s `ImmutableList` or annotate your holder `@Immutable`. The Layout Inspector's recomposition counts (§15.3) reveal where skipping is failing.

### 7.1 Why architecture matters **[I]**

A real app has UI, business rules, data sources, and async work. If they're tangled in an Activity/composable, you get untestable, fragile code that breaks on rotation. The **recommended architecture** separates concerns into layers and uses a few standard components. The payoff: rotation-safe, testable, maintainable code.

### 7.2 ViewModel — and why it survives config changes **[I, important]**

**What it is:** a `ViewModel` is a class that holds and prepares UI state, separate from the UI. It's scoped to a screen/navigation entry.

**Why it survives configuration changes (the key reason it exists):** when an Activity is destroyed and recreated on rotation (§4.3), its `ViewModel` is *retained* by the framework and reattached to the new Activity instance. So in-flight data and UI state aren't lost, and you don't re-fetch on every rotation. It is cleared (`onCleared()`) only when the screen is *really* gone (the Activity finishes for good).

**What goes in it:** UI state, and the logic to update it (call repositories, run coroutines via `viewModelScope`, §11). **Never** put a reference to a View, Activity, or Context that outlives the ViewModel (memory leak, §18) — use `AndroidViewModel` if you need the *application* Context.

```kotlin
// A ViewModel exposing immutable UI state via StateFlow; mutations stay private.
class CounterViewModel : ViewModel() {
    // Private MUTABLE state; public READ-ONLY view. UDF: UI can't mutate directly.
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()

    fun increment() { _count.value += 1 }   // the ONE place that changes count
    fun reset()     { _count.value = 0 }
}
```

### 7.3 StateFlow vs LiveData **[I]**

Both are **observable state holders** the UI subscribes to so it updates when data changes.

- **`StateFlow`** — a Kotlin coroutines/`Flow`-based holder; always has a current value; the modern, Compose-friendly choice. Collect it with `collectAsStateWithLifecycle()` in Compose (lifecycle-aware: stops collecting when the screen is stopped).
- **`LiveData`** — the older lifecycle-aware holder from the View/Java era. Still common in existing apps; observe with `observe(lifecycleOwner) { }`. Prefer `StateFlow` for new Kotlin/Compose code.

```kotlin
// Consuming StateFlow in Compose, lifecycle-aware:
@Composable
fun CounterRoute(vm: CounterViewModel = viewModel()) {
    val count by vm.count.collectAsStateWithLifecycle()   // re-renders when count emits
    CounterDisplay(count = count, onIncrement = vm::increment)   // event flows up to the VM
}
```

### 7.4 MVVM with a repository layer **[I/A]**

**The layers:**

```
UI (Composables/Activity)  ── observes state, sends events ──►  ViewModel
        ▲                                                          │ calls
        └────────────── exposes UI state (StateFlow) ◄────────────┘
                                                                   │
                                            ViewModel ──► Repository (the single source of truth
                                                            for a data type; hides WHERE data
                                                            comes from)
                                                                   │
                                  Repository ──► data sources: Room (local, §9), Retrofit (remote, §10)
```

- **UI layer** — composables + ViewModel. The UI is "dumb": render state, forward events.
- **ViewModel** — holds UI state, orchestrates use cases, survives config changes.
- **Repository** — mediates between data sources; decides cache vs network; exposes clean data (often `Flow`). The rest of the app never talks to Retrofit/Room directly.
- **(Optional) Domain layer** — use-case classes for complex business logic shared across ViewModels.

```kotlin
// Repository: the single source of truth; here it merges a local cache (Room) + network (Retrofit).
class ArticleRepository(
    private val api: ArticleApi,        // Retrofit service (§10)
    private val dao: ArticleDao         // Room DAO (§9)
) {
    // Expose a Flow the ViewModel observes. Room emits on every DB change automatically.
    fun observeArticles(): Flow<List<Article>> = dao.observeAll()

    suspend fun refresh() {             // suspend = runs off the main thread when called from a scope
        val remote = api.getArticles()  // network call
        dao.upsertAll(remote.map { it.toEntity() })  // cache locally; the Flow above re-emits
    }
}

class ArticleViewModel(private val repo: ArticleRepository) : ViewModel() {
    // Turn the repo Flow into UI state. stateIn caches the latest value for new collectors.
    val articles: StateFlow<List<Article>> = repo.observeArticles()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())

    fun refresh() = viewModelScope.launch {       // viewModelScope auto-cancels on onCleared (§11)
        runCatching { repo.refresh() }            // handle errors -> update an error state in real apps
    }
}
```

### 7.5 Modeling loading / error state **[I]**

Real screens are not just "data" — they're loading, error, or content. Model this explicitly so the UI can render each case. A `sealed interface` (KOTLIN_GUIDE.md §8) is ideal:

```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}

class FeedViewModel(private val repo: ArticleRepository) : ViewModel() {
    private val _state = MutableStateFlow<UiState<List<Article>>>(UiState.Loading)
    val state: StateFlow<UiState<List<Article>>> = _state.asStateFlow()

    init { load() }
    fun load() = viewModelScope.launch {
        _state.value = UiState.Loading
        _state.value = try {
            UiState.Success(repo.observeArticles().first())
        } catch (e: Exception) {
            UiState.Error(e.message ?: "Unknown error")
        }
    }
}
```

```kotlin
// The UI renders each branch — exhaustive 'when' over the sealed type.
@Composable
fun FeedScreen(vm: FeedViewModel = viewModel()) {
    when (val s = vm.state.collectAsStateWithLifecycle().value) {
        UiState.Loading      -> CircularProgressIndicator()
        is UiState.Error     -> Text("Error: ${s.message}")
        is UiState.Success   -> ArticleList(s.data)
    }
}
```

### 7.6 Dependency injection with Hilt — overview **[A]**

**The problem:** the ViewModel needs a Repository, which needs an API and a DAO, which need Retrofit/Room... Manually constructing this graph everywhere is tedious and couples classes to their dependencies' construction.

**What DI is:** instead of a class *creating* its dependencies, they are *provided* to it. **Hilt** (built on Dagger) generates this wiring at compile time based on annotations.

**Why Hilt:** standard for Android, integrates with ViewModel/Activity/WorkManager, compile-time safe, removes boilerplate factories. 

```kotlin
// 1) Annotate the Application and add @AndroidEntryPoint to Activities.
@HiltAndroidApp class MyApplication : Application()

// 2) A module tells Hilt HOW to create things it can't construct itself.
@Module @InstallIn(SingletonComponent::class)
object DataModule {
    @Provides @Singleton
    fun provideRetrofit(): Retrofit = Retrofit.Builder()
        .baseUrl("https://api.example.com/")
        .addConverterFactory(/* §10 */ Json.asConverterFactory("application/json".toMediaType()))
        .build()

    @Provides @Singleton
    fun provideApi(retrofit: Retrofit): ArticleApi = retrofit.create(ArticleApi::class.java)
}

// 3) Hilt injects constructor parameters automatically (no manual wiring).
class ArticleRepository @Inject constructor(
    private val api: ArticleApi,
    private val dao: ArticleDao
) { /* ... */ }

// 4) Hilt provides the ViewModel + its dependencies.
@HiltViewModel
class ArticleViewModel @Inject constructor(
    private val repo: ArticleRepository
) : ViewModel() { /* ... */ }

// In Compose, obtain it with hiltViewModel():  val vm: ArticleViewModel = hiltViewModel()
```

---

## 8. Lists: RecyclerView and LazyColumn

Showing a scrolling list of many items is one of the most common tasks. Both worlds share one core idea: **only build views for what's on screen and recycle them** — you can't keep thousands of views in memory.

### 8.1 LazyColumn (Compose) — the modern way **[B/I]**

**What it is:** `LazyColumn`/`LazyRow` compose only the items currently visible (plus a small buffer); as you scroll, off-screen items are discarded and new ones composed. You describe items in a DSL.

**Why it's simpler than RecyclerView:** no adapter, no ViewHolder, no `notifyItemChanged` — you give it a list and a per-item composable; state changes recompose automatically. Provide a stable `key` so Compose can track items across data changes (correct animations, preserved scroll).

```kotlin
data class Task(val id: Long, val title: String, val done: Boolean)

@Composable
fun TaskList(tasks: List<Task>, onToggle: (Long) -> Unit) {
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(items = tasks, key = { it.id }) { task ->   // key = stable identity per item
            ListItem(
                headlineContent = { Text(task.title) },
                trailingContent = {
                    Checkbox(checked = task.done, onCheckedChange = { onToggle(task.id) })
                },
                modifier = Modifier.clickable { onToggle(task.id) }
            )
            HorizontalDivider()
        }
    }
}
```

### 8.2 RecyclerView + Adapter + ViewHolder (View system) **[I]**

**What it is:** the View-system list widget. You write:
- an **item layout** XML (one row),
- a **ViewHolder** — caches the row's view references (avoids repeated findViewById),
- an **Adapter** — binds data items to ViewHolders and tells RecyclerView how many items there are,
- a **LayoutManager** — controls arrangement (linear, grid).

**Why this structure:** RecyclerView reuses a small pool of row views as you scroll (the "recycler"); the Adapter rebinds data into recycled ViewHolders. `ListAdapter` + `DiffUtil` computes minimal changes when the list updates (efficient, animated). You'll meet this constantly in existing apps.

```kotlin
// 1) item_task.xml has a TextView @+id/title and a CheckBox @+id/done (omitted for brevity).

// 2) ListAdapter handles diffing; ViewHolder caches the row's views via View Binding.
class TaskAdapter(private val onToggle: (Task) -> Unit) :
    ListAdapter<Task, TaskAdapter.VH>(DIFF) {

    // ViewHolder = one reusable row.
    class VH(val binding: ItemTaskBinding) : RecyclerView.ViewHolder(binding.root)

    // Inflate a new row view when the pool needs one.
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val binding = ItemTaskBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return VH(binding)
    }

    // Bind data into a (possibly recycled) row.
    override fun onBindViewHolder(holder: VH, position: Int) {
        val task = getItem(position)
        holder.binding.title.text = task.title
        holder.binding.done.isChecked = task.done
        holder.binding.root.setOnClickListener { onToggle(task) }
    }

    companion object {
        // DiffUtil tells the adapter which rows actually changed (efficient updates).
        val DIFF = object : DiffUtil.ItemCallback<Task>() {
            override fun areItemsTheSame(a: Task, b: Task) = a.id == b.id
            override fun areContentsTheSame(a: Task, b: Task) = a == b
        }
    }
}

// 3) Wire it up in the Activity/Fragment:
//    val adapter = TaskAdapter(onToggle = { vm.toggle(it.id) })
//    recyclerView.layoutManager = LinearLayoutManager(this)
//    recyclerView.adapter = adapter
//    adapter.submitList(tasks)   // ListAdapter diffs old vs new and animates changes
```

---

## 9. Data & Persistence: DataStore, Room, Files

Apps persist data so it survives restarts. Choose by shape and size: small key/values → DataStore; structured/relational → Room; arbitrary blobs → files.

### 9.1 SharedPreferences & DataStore — small key/value data **[I]**

**SharedPreferences** is the legacy key/value store (XML-backed). It works but its API is synchronous-ish and easy to misuse. **DataStore** is the modern replacement: asynchronous (coroutines/Flow), safe, and type-safe (with Proto DataStore). Use **Preferences DataStore** for simple keys; **Proto DataStore** for typed objects.

**When:** user settings, flags, last-opened tab, onboarding-seen — small bits of state. **Not** for lists/relational data (use Room).

```kotlin
// Preferences DataStore — one instance per file, declared at top level.
private val Context.dataStore by preferencesDataStore(name = "settings")

class SettingsRepository(private val context: Context) {
    private val DARK = booleanPreferencesKey("dark_mode")   // typed key

    // Reading is a Flow: emits the current value and on every change.
    val darkMode: Flow<Boolean> = context.dataStore.data.map { prefs -> prefs[DARK] ?: false }

    // Writing is a suspend function (no blocking the main thread).
    suspend fun setDarkMode(on: Boolean) {
        context.dataStore.edit { prefs -> prefs[DARK] = on }
    }
}
```

### 9.2 Room — the recommended local database **[I, important]**

**What it is:** Room is an ORM over **SQLite** (the embedded SQL database every Android device has). You define data as annotated classes; Room generates the SQL boilerplate and verifies your queries **at compile time**.

**Why Room over raw SQLite:** compile-time query checking (a typo or wrong column fails the build, not at runtime), `Flow`/coroutine integration (observe queries reactively), migrations, and far less boilerplate.

**Three pieces:**
- **`@Entity`** — a class = a table; properties = columns.
- **`@Dao`** (Data Access Object) — an interface of annotated query/insert/update/delete methods.
- **`@Database`** — ties entities + DAOs together and is the DB handle.

```kotlin
// 1) Entity = a table.
@Entity(tableName = "tasks")
data class TaskEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    val done: Boolean = false,
    val createdAt: Long = System.currentTimeMillis()
)

// 2) DAO = how you read/write. Room generates the implementation.
@Dao
interface TaskDao {
    @Query("SELECT * FROM tasks ORDER BY createdAt DESC")
    fun observeAll(): Flow<List<TaskEntity>>          // reactive: re-emits on any change

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun upsert(task: TaskEntity): Long         // suspend = runs off the main thread

    @Query("UPDATE tasks SET done = :done WHERE id = :id")
    suspend fun setDone(id: Long, done: Boolean)

    @Delete suspend fun delete(task: TaskEntity)
}

// 3) Database = the handle. abstract; Room generates the concrete class.
@Database(entities = [TaskEntity::class], version = 1, exportSchema = true)
abstract class AppDatabase : RoomDatabase() {
    abstract fun taskDao(): TaskDao
}

// 4) Build it once (a singleton — building the DB is expensive). Hilt usually provides this.
fun buildDatabase(context: Context): AppDatabase =
    Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
        // .addMigrations(MIGRATION_1_2)   // define migrations when you bump 'version'
        .build()
```

> **Migrations:** when you change the schema, bump `version` and supply a `Migration` (or `fallbackToDestructiveMigration()` in dev to wipe). Never ship schema changes without a migration or users lose data / the app crashes. `exportSchema = true` writes the schema to a folder so you can review and test migrations.

```kotlin
// A migration from schema v1 -> v2 that adds a column. Test migrations with MigrationTestHelper.
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE tasks ADD COLUMN priority INTEGER NOT NULL DEFAULT 0")
    }
}
```

**Type converters & relations.** Room columns must be primitive/`String`/`ByteArray`. To store a custom type (an enum, a `Date`, a list), provide a `@TypeConverter`. To model relationships (one-to-many), use `@Relation` with an embedded query result class.

```kotlin
// Convert a non-primitive type so Room can store it as a primitive column.
class Converters {
    @TypeConverter fun fromInstant(v: Long?): Instant? = v?.let { Instant.ofEpochMilli(it) }
    @TypeConverter fun toInstant(i: Instant?): Long? = i?.toEpochMilli()
}
// Register on the database:  @Database(...) @TypeConverters(Converters::class) abstract class ...

// A one-to-many read: each Project with its Tasks, assembled by Room.
data class ProjectWithTasks(
    @Embedded val project: ProjectEntity,
    @Relation(parentColumn = "id", entityColumn = "projectId") val tasks: List<TaskEntity>
)
@Dao interface ProjectDao {
    @Transaction @Query("SELECT * FROM projects")
    fun observeProjectsWithTasks(): Flow<List<ProjectWithTasks>>
}
```

> **Threading rule:** Room forbids database access on the main thread (it would freeze the UI). `suspend` DAO methods run on a background dispatcher automatically; `Flow`-returning queries observe on a background thread and emit to your collector. Never call a synchronous DAO method from the main thread.

### 9.3 File storage & scoped storage basics **[I]**

- **Internal storage** (`context.filesDir`, `context.cacheDir`) — private to your app, no permission needed, deleted on uninstall. Use for app-private files; `cacheDir` for disposable caches the OS may clear.
- **External / shared storage** — historically free-for-all, now governed by **scoped storage**: apps get their own scoped external dir (`context.getExternalFilesDir(...)`, no permission), and to read/write *shared* media you use the **MediaStore** API or the **Storage Access Framework** (the system file picker) rather than raw file paths. This protects user files and privacy.

```kotlin
// Write/read a private file in internal storage (no permission needed).
context.openFileOutput("notes.txt", Context.MODE_PRIVATE).use { it.write("hello".toByteArray()) }
val text = context.openFileInput("notes.txt").bufferedReader().use { it.readText() }
```

---

## 10. Networking: Retrofit, OkHttp, Serialization

### 10.1 The stack and why **[I]**

Most apps talk to a REST/JSON backend. The standard stack:
- **OkHttp** — the HTTP client (connections, timeouts, interceptors, caching).
- **Retrofit** — a type-safe layer over OkHttp: you declare an *interface* describing endpoints with annotations; Retrofit generates the implementation.
- **A converter** — turns JSON into Kotlin objects: **kotlinx.serialization** (Kotlin-native, no reflection, the modern choice), or Moshi/Gson (older).
- **Coroutines** — Retrofit supports `suspend` functions, so calls integrate with structured concurrency (§11) and don't block the main thread.

### 10.2 Defining and calling an API **[I]**

```kotlin
// 1) Data model. @Serializable lets kotlinx.serialization parse JSON into it.
@Serializable
data class Post(
    val id: Int,
    val title: String,
    val body: String,
    @SerialName("userId") val userId: Int   // map a differently-named JSON field
)

// 2) The API as an interface. Annotations describe HTTP method + path + params.
interface JsonPlaceholderApi {
    @GET("posts")                                   // GET {baseUrl}/posts
    suspend fun getPosts(): List<Post>              // suspend -> off the main thread

    @GET("posts/{id}")                              // path parameter
    suspend fun getPost(@Path("id") id: Int): Post

    @GET("posts")
    suspend fun getByUser(@Query("userId") userId: Int): List<Post>  // ?userId=...

    @POST("posts")
    suspend fun create(@Body post: Post): Post      // serialize 'post' as the request body
}

// 3) Build Retrofit (do this once; Hilt typically provides it, §7.6).
val json = Json { ignoreUnknownKeys = true }        // tolerate extra JSON fields
val client = OkHttpClient.Builder()
    .addInterceptor(HttpLoggingInterceptor().apply { level = HttpLoggingInterceptor.Level.BODY })
    .build()
val retrofit = Retrofit.Builder()
    .baseUrl("https://jsonplaceholder.typicode.com/")   // MUST end with '/'
    .client(client)
    .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
    .build()
val api: JsonPlaceholderApi = retrofit.create(JsonPlaceholderApi::class.java)
```

### 10.3 Calling it from a ViewModel with loading/error states **[I/A]**

```kotlin
class PostsViewModel(private val api: JsonPlaceholderApi) : ViewModel() {
    private val _state = MutableStateFlow<UiState<List<Post>>>(UiState.Loading)  // §7.5
    val state: StateFlow<UiState<List<Post>>> = _state.asStateFlow()

    init { load() }

    fun load() = viewModelScope.launch {           // coroutine on the main-safe scope (§11)
        _state.value = UiState.Loading
        _state.value = try {
            UiState.Success(api.getPosts())        // Retrofit runs the IO on a background dispatcher
        } catch (e: IOException) {
            UiState.Error("Network error — check your connection")
        } catch (e: HttpException) {               // non-2xx response
            UiState.Error("Server error ${e.code()}")
        }
    }
}
```

```kotlin
// The screen reacts to the three states (loading spinner / error / list).
@Composable
fun PostsScreen(vm: PostsViewModel = viewModel()) {
    val state by vm.state.collectAsStateWithLifecycle()
    when (val s = state) {
        UiState.Loading    -> Box(Modifier.fillMaxSize(), Alignment.Center) { CircularProgressIndicator() }
        is UiState.Error   -> Column(Modifier.fillMaxSize().padding(16.dp)) {
            Text(s.message); Button(onClick = vm::load) { Text("Retry") }
        }
        is UiState.Success -> LazyColumn {
            items(s.data, key = { it.id }) { post ->
                ListItem(headlineContent = { Text(post.title) },
                         supportingContent = { Text(post.body, maxLines = 2) })
            }
        }
    }
}
```

> **Don't forget the manifest:** `<uses-permission android:name="android.permission.INTERNET" />`. Cleartext (non-HTTPS) traffic is blocked by default on modern Android — use HTTPS or a network-security-config exception.

---

## 11. Concurrency: Coroutines & Flow in Android

### 11.1 Why concurrency, and the cardinal rule **[I, critical]**

**The cardinal rule:** **never block the main (UI) thread.** All UI work runs on it; if you do network/disk/heavy work there, the UI freezes and you get an **ANR** ("Application Not Responding") dialog. So long work goes to background threads — and **coroutines** are how Kotlin/Android do that cleanly (see KOTLIN_GUIDE.md §11 for the language fundamentals).

**Why coroutines over raw threads/callbacks:** they let you write asynchronous code that *reads* sequentially, with **structured concurrency** (work is scoped and auto-cancelled when the scope dies — no leaks), and `suspend` functions that pause without blocking a thread.

### 11.2 Scopes: `viewModelScope` and `lifecycleScope` **[I]**

A **`CoroutineScope`** bounds the lifetime of coroutines launched in it; when the scope is cancelled, all its coroutines are cancelled. Android gives you lifecycle-tied scopes so you don't leak work:

- **`viewModelScope`** — tied to a ViewModel; cancelled in `onCleared()`. **Launch most app work here** (data loads, repository calls). Survives config changes (the ViewModel does, §7.2).
- **`lifecycleScope`** — tied to an Activity/Fragment lifecycle; cancelled when destroyed. Use for UI-bound work that should stop when the screen is gone.
- In Compose, use `LaunchedEffect`/`rememberCoroutineScope` (§6.8) — backed by the composition lifecycle.

```kotlin
class ProfileViewModel(private val repo: UserRepository) : ViewModel() {
    fun load(id: String) = viewModelScope.launch {   // auto-cancelled when the VM is cleared
        val user = repo.fetchUser(id)                 // suspend; doesn't block the main thread
        // update state...
    }
}
```

### 11.3 Dispatchers — choosing the thread pool **[I]**

A **`Dispatcher`** decides which thread(s) a coroutine runs on:
- **`Dispatchers.Main`** — the UI thread (touch UI/state here). `viewModelScope`/`lifecycleScope` default to Main.
- **`Dispatchers.IO`** — a large pool for blocking IO (disk, network if a library blocks). 
- **`Dispatchers.Default`** — CPU-bound work (parsing, sorting big lists).

**Key idea:** suspend functions should be **main-safe** — safe to call from Main; they switch internally with `withContext(Dispatchers.IO)` for the blocking part. Retrofit/Room already do this for `suspend` methods, so you usually don't switch manually.

```kotlin
suspend fun loadConfig(): Config = withContext(Dispatchers.IO) {  // do blocking parse off Main
    val text = File(path).readText()                              // blocking IO — OK here
    parseConfig(text)                                             // result returns to caller's thread
}
```

### 11.4 Flow for streams **[I/A]**

**What it is:** a `Flow` is a cold asynchronous stream of values over time (vs a suspend function returning one value). Room queries and DataStore expose `Flow`s so the UI reacts to changes automatically.

**`StateFlow`/`SharedFlow`** are hot Flows for state/events (§7.3). Collect lifecycle-safely in the UI:

```kotlin
// In a ViewModel: transform a repo Flow into UI state, run mapping on Default.
val titles: StateFlow<List<String>> = repo.observeArticles()
    .map { list -> list.map { it.title } }          // operators run on the collector's context
    .flowOn(Dispatchers.Default)                    // ...but move THIS upstream work to Default
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())

// In Compose, collect lifecycle-aware so collection pauses when the screen is stopped:
//   val titles by vm.titles.collectAsStateWithLifecycle()
```

> **⚡ Lifecycle-aware collection:** in the View system use `repeatOnLifecycle(Lifecycle.State.STARTED) { flow.collect { } }` inside `lifecycleScope`; in Compose use `collectAsStateWithLifecycle()`. This stops collecting (and frees resources) when the screen isn't visible — avoiding wasted work and leaks (§18).

### 11.5 Structured concurrency & parallel work **[I/A]**

**Structured concurrency** is the principle that a coroutine launched inside a scope is a *child* of it: the parent does not complete until its children do, cancelling the parent cancels the children, and a child's failure propagates to the parent. This is *why* coroutines don't leak — work is bounded by a scope (`viewModelScope`) that the framework tears down at the right time. (Contrast `GlobalScope`, which has no parent and leaks — avoid it.)

Run independent work **in parallel** with `async`/`await` to cut latency:

```kotlin
fun loadDashboard() = viewModelScope.launch {
    // Both calls start immediately and run concurrently; we wait for both.
    val profileDeferred = async { repo.fetchProfile() }   // returns Deferred<Profile>
    val feedDeferred    = async { repo.fetchFeed() }
    val profile = profileDeferred.await()                 // suspends until ready
    val feed    = feedDeferred.await()
    _state.value = Dashboard(profile, feed)               // combine results
    // If either throws, structured concurrency cancels the other automatically.
}
```

### 11.6 Cancellation & exceptions **[I/A]**

Cancellation is **cooperative**: a coroutine is cancelled by throwing `CancellationException` at the next suspension point. Your loops should check `isActive` or call a suspending function periodically so cancellation can take effect. Never swallow `CancellationException` in a broad `catch (e: Exception)` — rethrow it, or the coroutine won't actually stop.

```kotlin
viewModelScope.launch {
    try {
        repo.refresh()
    } catch (e: CancellationException) {
        throw e                       // let cancellation propagate — do NOT swallow it
    } catch (e: Exception) {
        _state.value = UiState.Error(e.message ?: "Failed")
    }
}
```

---

### 11.7 Background work that must outlive the screen: WorkManager **[I/A]**

`viewModelScope` work dies when the screen does — fine for UI loads, wrong for "upload this file even if the user leaves" or periodic sync. For **deferrable, guaranteed background work** (survives process death, respects battery/network constraints), use **WorkManager**. It is the recommended API for most background jobs (it replaced `JobScheduler`/alarms/services for these cases).

```kotlin
// A Worker does the actual background job; WorkManager runs it when constraints are met.
class SyncWorker(ctx: Context, params: WorkerParameters) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result = try {
        // ... perform a sync (network, Room) — runs off the main thread ...
        Result.success()
    } catch (e: Exception) { Result.retry() }   // WorkManager retries with backoff
}

// Schedule it with constraints (only on unmetered network, when charging, etc.).
val request = OneTimeWorkRequestBuilder<SyncWorker>()
    .setConstraints(Constraints.Builder().setRequiredNetworkType(NetworkType.UNMETERED).build())
    .build()
WorkManager.getInstance(context).enqueue(request)
```

---

## 12. Resources & Assets

(See §2.5–2.7 for the folder layout; this section is about using resources well.)

### 12.1 Strings, colors, dimens, themes **[B]**

**Always externalize user-facing text** into `strings.xml` (never hardcode in layouts/code). Reasons: localization (§12.4), reuse, and easy review. Same logic motivates `colors.xml` (consistent palette), `dimens.xml` (consistent spacing), and `themes.xml` (centralized styling). Reference as `@string/...`, `@color/...`, `@dimen/...` in XML; `R.string....`, `MaterialTheme.colorScheme....` in code/Compose.

### 12.2 Drawables: vector vs raster **[B/I]**

- **Vector drawables** (`<vector>` XML, or imported SVGs via Asset Studio) — resolution-independent: one file scales crisply to every density. Prefer for icons/simple shapes. Smaller APK.
- **Raster** (PNG/WebP/JPG) — pixel images; needed for photos. Provide density variants (below). Prefer **WebP** over PNG for size.

### 12.3 Density buckets **[B/I]**

Screens vary in pixel density. Android groups them into **buckets**: `mdpi` (baseline, 1×), `hdpi` (1.5×), `xhdpi` (2×), `xxhdpi` (3×), `xxxhdpi` (4×). For raster images, supply a version per bucket in `drawable-hdpi/`, `drawable-xhdpi/`, etc.; the system picks the closest and scales as needed. Vectors sidestep this entirely (one file, any density). Sizes in `dp`/`sp` (§5.3) already abstract density for layout.

### 12.4 Dark theme **[B/I]**

Provide night variants with the `-night` qualifier (`values-night/colors.xml`, `drawable-night/...`). The system swaps them when dark mode is on. In Compose, `isSystemInDarkTheme()` + your `MaterialTheme` color schemes handle it (§6.9). Test both with the preview `uiMode` (§6.3).

### 12.5 Localization **[I]**

Provide translated `strings.xml` under locale-qualified folders: `values-es/strings.xml` (Spanish), `values-fr/`, `values-ja/`, etc. The system picks the right one for the device locale; if missing, it falls back to the default `values/`. This is *why* you externalize strings — translation becomes adding folders, no code changes. Use `%1$s`-style positional placeholders so word order can differ per language.

### 12.6 App icons (adaptive icons) **[I]**

Modern launcher icons are **adaptive**: two layers (foreground + background) that the launcher masks into different shapes (circle, squircle, rounded square) and can animate. Defined in `mipmap-anydpi-v26/ic_launcher.xml` referencing `@drawable/ic_launcher_foreground` and a background color/drawable. Use **Image Asset Studio** (right-click `res` → New → Image Asset) to generate all densities + the adaptive XML correctly.

```xml
<!-- res/mipmap-anydpi-v26/ic_launcher.xml — an adaptive icon (two layers). -->
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@color/ic_launcher_background" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
    <!-- <monochrome .../> enables themed icons on Android 13+ -->
</adaptive-icon>
```

---

## 13. Permissions

### 13.1 The two kinds and the logic **[I]**

Permissions guard sensitive capabilities (camera, location, contacts). Two categories:
- **Install-time / normal** (e.g. `INTERNET`) — granted automatically when declared in the manifest. No runtime prompt.
- **Runtime / "dangerous"** (camera, location, microphone, contacts) — must be **declared in the manifest AND requested at runtime**, and the user can deny or revoke them at any time. 

**Why runtime requests exist:** users should grant access in context ("this app wants your location"), not blindly at install. Your code must handle "not yet granted", "denied", and "permanently denied".

### 13.2 Requesting at runtime (Activity Result API) **[I]**

```kotlin
class CameraActivity : ComponentActivity() {
    // Register a permission-request launcher (the modern, lifecycle-safe API).
    private val requestCamera = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted: Boolean ->
        if (granted) openCamera()
        else showRationaleOrSettings()   // explain why, or guide to Settings if permanently denied
    }

    private fun ensureCamera() {
        when {
            // Already granted?
            ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA)
                == PackageManager.PERMISSION_GRANTED -> openCamera()
            // Should we explain first? (user denied once before)
            shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) ->
                showRationaleThenRequest()
            // Otherwise just ask.
            else -> requestCamera.launch(Manifest.permission.CAMERA)
        }
    }
    private fun openCamera() {}
    private fun showRationaleOrSettings() {}
    private fun showRationaleThenRequest() {}
}
```

In Compose, the **Accompanist Permissions** library (or the same Activity Result API via `rememberLauncherForActivityResult`) provides `rememberPermissionState`. Always design for denial: degrade gracefully, never trap the user.

> **⚡ Notifications:** since Android 13 (API 33), posting notifications requires the runtime `POST_NOTIFICATIONS` permission. Location has a tiered model (approximate vs precise, foreground vs background) with extra requirements for background access.

---

## 14. Navigation

Moving between screens. Two systems: **navigation-compose** (for Compose apps — preferred) and the **Navigation component** (XML nav graphs + fragments, for the View system).

### 14.1 navigation-compose **[I]**

**What it is:** a `NavHost` holds composable **destinations** keyed by **route** strings; a `NavController` performs navigation and owns the back stack. Arguments are encoded in routes; deep links map URIs to destinations.

```kotlin
@Composable
fun AppNavGraph() {
    val nav = rememberNavController()
    NavHost(navController = nav, startDestination = "list") {

        composable("list") {
            ListScreen(onOpen = { id -> nav.navigate("detail/$id") })  // navigate + push
        }

        composable(
            route = "detail/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.IntType }),  // typed arg
            deepLinks = listOf(navDeepLink { uriPattern = "myapp://item/{itemId}" })
        ) { entry ->
            val itemId = entry.arguments?.getInt("itemId") ?: return@composable
            DetailScreen(itemId = itemId, onBack = { nav.popBackStack() })  // pop = go back
        }
    }
}
```

> **⚡ Type-safe navigation:** newer Navigation Compose supports **type-safe routes** using `@Serializable` route objects/data classes instead of string concatenation — define a `data class Detail(val itemId: Int)` and `navigate(Detail(42))`. Prefer this when available; it eliminates stringly-typed bugs.

### 14.2 Navigation component (XML / fragments) **[I]**

**What it is:** a visual **nav graph** (`res/navigation/nav_graph.xml`) defines destinations (fragments) and actions (edges) between them; the **Navigation editor** lets you draw them. A `NavHostFragment` hosts the current destination; the **Safe Args** Gradle plugin generates type-safe argument classes.

```xml
<!-- res/navigation/nav_graph.xml -->
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    app:startDestination="@id/listFragment">

    <fragment android:id="@+id/listFragment"
        android:name="com.yourname.myapp.ListFragment">
        <!-- An action = a navigable edge to another destination. -->
        <action android:id="@+id/toDetail" app:destination="@id/detailFragment" />
    </fragment>

    <fragment android:id="@+id/detailFragment"
        android:name="com.yourname.myapp.DetailFragment">
        <argument android:name="itemId" app:argType="integer" />   <!-- typed argument -->
        <deepLink app:uri="myapp://item/{itemId}" />
    </fragment>
</navigation>
```

```kotlin
// Navigate from a Fragment using the generated Safe Args directions (type-safe):
findNavController().navigate(ListFragmentDirections.toDetail(itemId = 42))
```

---

## 15. Debugging & Tooling

### 15.1 Logcat **[B]**

**What it is:** the device's system log stream; your primary "print debugging" tool. Use `android.util.Log` with levels:

```kotlin
private const val TAG = "MainActivity"     // a tag to filter by in Logcat
Log.v(TAG, "verbose")
Log.d(TAG, "debug — most common during development")
Log.i(TAG, "info")
Log.w(TAG, "warning")
Log.e(TAG, "error", throwable)             // pass the exception to log its stack trace
```

In the **Logcat** window (`Alt+6`) filter by package, tag, or level, and search. **Crashes** (uncaught exceptions) print a red stack trace here — read it bottom-up to find the "Caused by" line and the file:line in *your* code. Strip noisy logs from release builds (or use a logging library like Timber that no-ops in release).

### 15.2 The debugger & breakpoints **[I]**

Run with **Debug** (`Shift+F9`). Click the gutter to set a **breakpoint**; execution pauses there and you can:
- inspect variables (Variables pane), hover to evaluate,
- **Step Over** (`F8`), **Step Into** (`F7`), **Step Out** (`Shift+F8`), **Resume** (`F9`),
- use **Evaluate Expression** to run code in the paused context,
- set **conditional breakpoints** (right-click → condition) to pause only when, say, `id == 42`.

### 15.3 Layout Inspector & Compose tools **[I]**

The **Layout Inspector** captures a *running* app's UI tree (Views and/or composables) so you can inspect hierarchy, attributes, and (for Compose) recomposition counts — invaluable for "why is this misaligned / why does it recompose so much." For Compose, the preview pane (§6.3), interactive preview, and the **Layout Inspector's recomposition highlighting** are your design/debug loop.

### 15.4 Profilers **[I/A]**

The **Profiler** (and the standalone profiling tools in newer Studio) measure:
- **CPU** — method traces / sampling to find hot spots and jank.
- **Memory** — allocations and heap dumps to find leaks (e.g. an Activity retained after destroy, §18).
- **Network** — requests, timing, payload sizes.
- **Energy** — battery impact.

Reach for these when the app is slow, janky, leaking, or draining battery — measure, don't guess.

### 15.5 Common errors & fixes **[B/I]**

| Symptom | Likely cause / fix |
|---|---|
| `NullPointerException` on a view | Accessed a view before inflation, or wrong layout — use View Binding (§5.8). |
| App freezes / ANR | Blocking the main thread — move work to coroutines/Dispatchers (§11). |
| `R` is unresolved (red) | Resource error or failed compile — fix the resource, rebuild, sync Gradle. |
| `ClassNotFoundException` for an Activity | Not declared in the manifest, or wrong `android:name`. |
| Network call returns nothing / crashes | Missing `INTERNET` permission, cleartext blocked, or wrong base URL (must end `/`). |
| Resource not found at runtime | Wrong qualifier/locale or referencing a stripped resource (R8 shrinking). |
| Gradle sync fails | Version mismatch (AGP/Gradle/Kotlin) — check the catalog & `gradle-wrapper.properties`. |

---

## 16. Testing

### 16.1 Local vs instrumented tests **[I]**

- **Local unit tests** (`src/test`, §2.4) — run on your PC's JVM, no device. Fast. For pure logic: ViewModels, repositories (with fakes), mappers. JUnit (+ MockK/Mockito, + `kotlinx-coroutines-test`).
- **Instrumented tests** (`src/androidTest`) — run on a device/emulator. For things needing the real framework: Room DAOs, UI (Espresso for Views, Compose test API for Compose).

### 16.2 A unit test for a ViewModel **[I]**

```kotlin
class CounterViewModelTest {
    @Test
    fun increment_increasesCount() = runTest {     // runTest from kotlinx-coroutines-test
        val vm = CounterViewModel()
        assertEquals(0, vm.count.value)            // initial state
        vm.increment()
        assertEquals(1, vm.count.value)            // after event
    }
}
```

### 16.3 A Compose UI test **[I/A]**

```kotlin
class CounterUiTest {
    @get:Rule val rule = createComposeRule()       // hosts composables for testing

    @Test fun clickingButton_incrementsDisplayedCount() {
        rule.setContent { MyAppTheme { CounterScreen() } }    // set the UI under test

        rule.onNodeWithText("Count: 0").assertIsDisplayed()   // find by text, assert
        rule.onNodeWithText("Increment").performClick()       // act
        rule.onNodeWithText("Count: 1").assertIsDisplayed()   // assert new state
    }
}
```

### 16.4 An Espresso UI test (View system) **[I]**

```kotlin
@RunWith(AndroidJUnit4::class)
class MainActivityTest {
    @get:Rule val activityRule = ActivityScenarioRule(MainActivity::class.java)

    @Test fun typingName_updatesGreeting() {
        onView(withId(R.id.nameInput)).perform(typeText("Ada"), closeSoftKeyboard())
        onView(withId(R.id.submitBtn)).perform(click())
        onView(withId(R.id.titleText)).check(matches(withText(containsString("Ada"))))
    }
}
```

### 16.5 Testing a ViewModel that uses coroutines + a fake repository **[I/A]**

Make dependencies *interfaces* so tests inject **fakes** (in-memory implementations) instead of real network/DB. Use a test dispatcher so coroutines run deterministically (no real delays, no real threads).

```kotlin
// 1) The dependency is an interface — production uses Retrofit/Room, tests use a fake.
interface PostRepository { suspend fun getPosts(): List<Post> }

class FakePostRepository(private val posts: List<Post>) : PostRepository {
    var shouldThrow = false
    override suspend fun getPosts(): List<Post> =
        if (shouldThrow) throw IOException("boom") else posts
}

// 2) A JUnit rule that swaps Dispatchers.Main for a test dispatcher (ViewModels default to Main).
class MainDispatcherRule(private val dispatcher: TestDispatcher = StandardTestDispatcher()) : TestWatcher() {
    override fun starting(d: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(d: Description) = Dispatchers.resetMain()
}

// 3) The test: arrange a fake, act, advance coroutines, assert the resulting state.
class PostsViewModelTest {
    @get:Rule val mainRule = MainDispatcherRule()

    @Test fun load_success_emitsSuccessState() = runTest {
        val repo = FakePostRepository(listOf(Post(1, "t", "b", 1)))
        val vm = PostsViewModel(repo)
        advanceUntilIdle()                                  // run all scheduled coroutines
        val state = vm.state.value
        assertTrue(state is UiState.Success && state.data.size == 1)
    }

    @Test fun load_failure_emitsErrorState() = runTest {
        val repo = FakePostRepository(emptyList()).apply { shouldThrow = true }
        val vm = PostsViewModel(repo)
        advanceUntilIdle()
        assertTrue(vm.state.value is UiState.Error)
    }
}
```

> **Tip:** make ViewModels and repositories depend on *interfaces* so tests can inject fakes. Use `kotlinx-coroutines-test`'s `runTest` and a test dispatcher for deterministic coroutine tests. Test Room DAOs as instrumented tests with an `inMemoryDatabaseBuilder` so they're fast and isolated.

---

## 17. Building & Publishing

### 17.1 Debug vs release builds **[I]**

- **Debug** — auto-signed with a debug key, debuggable, not minified. For development only.
- **Release** — minified/obfuscated by **R8** (smaller, harder to reverse-engineer), shrinks unused code/resources, and **signed with YOUR private release key**. This is what users get.

### 17.2 App signing & the release key **[I, important]**

Every APK/AAB must be **digitally signed**. The signature proves updates come from the same author; Play won't accept an update signed by a different key. **Guard your release keystore — losing it means you can't update your app** (mitigated by **Play App Signing**, where Google holds the app signing key and you keep an upload key).

```bash
# Generate a release keystore (do this ONCE; back it up securely, never commit it).
keytool -genkey -v -keystore release.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

```kotlin
// app/build.gradle.kts — wire the signing config (read secrets from gradle.properties / env, NOT hardcoded).
android {
    signingConfigs {
        create("release") {
            storeFile = file(System.getenv("KEYSTORE_PATH") ?: "release.jks")
            storePassword = System.getenv("KEYSTORE_PASSWORD")
            keyAlias = "upload"
            keyPassword = System.getenv("KEY_PASSWORD")
        }
    }
    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
            isMinifyEnabled = true
            isShrinkResources = true
        }
    }
}
```

### 17.3 APK vs AAB; building the release artifact **[I]**

- **APK** (`.apk`) — directly installable; what you sideload/test.
- **AAB** (`.aab`, **Android App Bundle**) — the publishing format. You upload the bundle; Play generates optimized per-device APKs (right density, right ABI), so users download less. **AAB is required for new apps on Play.**

```bash
gradlew.bat assembleRelease   # -> app/build/outputs/apk/release/app-release.apk
gradlew.bat bundleRelease     # -> app/build/outputs/bundle/release/app-release.aab  (upload THIS)
```
Or in the IDE: **Build → Generate Signed Bundle / APK**.

### 17.4 Versioning **[I]**

- **`versionCode`** — an integer Play uses to order releases; **must strictly increase** every upload. Users never see it.
- **`versionName`** — the human string ("1.4.2") shown to users. Use semantic-ish versioning.

### 17.5 Play Console basics & release checklist **[I]**

The **Play Console** is where you create the app listing, upload the AAB, and manage release tracks (**internal → closed (alpha/beta) → open → production**). You provide a store listing (title, description, screenshots, feature graphic), content rating, target audience, data-safety form, and a privacy policy.

**Release checklist:**
- [ ] `applicationId` final and unique; `targetSdk` at the Play-required level.
- [ ] Release signing configured; keystore backed up; Play App Signing enrolled.
- [ ] `versionCode` incremented; sensible `versionName`.
- [ ] `isMinifyEnabled`/`isShrinkResources` on; ProGuard/R8 rules keep reflection-used classes (Retrofit models, etc.); **test the release build** (shrinking can break reflection-based code).
- [ ] Remove debug logging / test endpoints; no secrets in the APK.
- [ ] Permissions minimal and justified; data-safety form accurate.
- [ ] Tested on a physical device and multiple API levels/densities; dark mode and a non-English locale checked.
- [ ] Crash-free on the main flows; ANR-free.

---

## 18. Gotchas & Best Practices

**Lifecycle / memory leaks.**
- Never hold a reference to an `Activity`/`View`/`Fragment` view from something that outlives it (a static field, a singleton, a long-lived listener, a coroutine on `GlobalScope`). The classic leak: an Activity destroyed on rotation but kept alive by a registered callback → its whole view tree leaks. Use `viewModelScope`/`lifecycleScope` (auto-cancel), unregister listeners in `onStop`/`onDestroyView`, and null out fragment view bindings in `onDestroyView`.

**Context misuse.**
- Don't store an Activity `Context` in a long-lived object — use `applicationContext` for things that outlive the screen (DB, prefs). But don't use `applicationContext` for UI/theming (use the Activity context). Wrong context → leaks or wrong theming.

**Main-thread blocking.**
- No network, disk, big JSON, or heavy CPU on the main thread. Symptom: jank/ANR. Fix: coroutines + the right `Dispatcher` (§11). Room/Retrofit `suspend` functions are already main-safe.

**Configuration changes.**
- Rotation/locale/dark-mode changes recreate the Activity (§4.3). Keep state in a `ViewModel` and `rememberSaveable`; don't fight it with `android:configChanges` unless you truly need to handle the change yourself.

**Large/deep layouts (View system).**
- Deeply nested `LinearLayout`s are slow (multiple measure passes). Flatten with `ConstraintLayout`. Use `RecyclerView`/`LazyColumn` for lists — never a giant scrolling `Column`/`LinearLayout`.

**Compose recomposition pitfalls.**
- Don't do side effects or heavy work in a composable body (§6.4) — use effect APIs. Provide stable `key`s in lazy lists (§8.1). Hoist state (§6.5). Avoid passing unstable lambdas/objects that defeat skipping; remember expensive computations with `remember`/`derivedStateOf`.

**Hardcoded strings/dimensions.**
- Externalize to resources (§12) for localization, theming, and consistency.

**Ignoring errors/null.**
- Model loading/error/empty states explicitly (§7.5); handle `IOException`/`HttpException` on network calls (§10). Lean on Kotlin null safety (KOTLIN_GUIDE.md §3).

**Shipping debug behavior.**
- Strip logs, disable cleartext, test the *release* (minified) build before publishing — R8 can break reflection-based libraries without keep rules.

**Permissions UX.**
- Request in context, handle denial gracefully, never loop the user (§13).

---

## 19. Study Path & Build-to-Learn Projects

### 19.1 Suggested learning order

1. **[B]** Install Android Studio; create an Empty Activity (Compose) project; run it on an emulator and a physical device (§1). Read Logcat as you rotate the screen (§4.3, §15).
2. **[B]** Internalize the project/folder structure and the manifest (§2). Edit `strings.xml`, `themes.xml`, swap the app icon (§12).
3. **[B]** Learn Gradle basics + the version catalog; add a dependency (§3).
4. **[B/I]** Build UI two ways: an XML screen in the Layout Editor with ConstraintLayout + View Binding (§5), then the same screen in Compose (§6). Compare. Going forward, build new UI in Compose.
5. **[I]** Master Compose state, hoisting, and UDF (§6.4–6.5), then ViewModel + StateFlow (§7).
6. **[I]** Lists with `LazyColumn` (and read RecyclerView, §8).
7. **[I]** Persist with Room and DataStore (§9); structured concurrency with coroutines/Flow (§11).
8. **[I/A]** Networking with Retrofit + serialization, with loading/error states (§10).
9. **[I]** Navigation (§14), permissions (§13), resources/localization/dark mode (§12).
10. **[I/A]** Architecture polish (repository + Hilt, §7), testing (§16), then build, sign, and (optionally) publish (§17). Profile and fix leaks/jank (§15, §18).

### 19.2 Build-to-learn projects (all in Compose)

**Project 1 — Counter app [B].** One screen: a number and +/− buttons.
- Goals: composables, `remember`/`mutableStateOf`, state hoisting, then move state into a `ViewModel` (§6, §7).
- Stretch: `rememberSaveable` so it survives rotation; a reset FAB in a `Scaffold`; a Compose UI test (§16).

**Project 2 — To-do list with Room [I].** Add, list, check off, delete tasks; persists across restarts.
- Goals: `LazyColumn` (§8), Room `@Entity`/`@Dao`/`@Database` (§9), MVVM (ViewModel + repository, §7), coroutines/`Flow` reactive list (§11).
- Stretch: DataStore for a "hide completed" setting (§9); swipe-to-delete; a DAO instrumented test (§16); dark theme (§12).

**Project 3 — REST API list/detail app [I/A].** Fetch a list from a public API (e.g. JSONPlaceholder), show it, tap an item for a detail screen.
- Goals: Retrofit + OkHttp + kotlinx.serialization (§10), navigation-compose with a typed argument (§14), explicit loading/error/retry states (§7.5, §10.3), `viewModelScope` (§11).
- Stretch: cache responses in Room for offline (§9); Hilt for DI (§7.6); pull-to-refresh; deep link to a detail screen (§14); profile a network call (§15).

After these three you will have touched every core area of modern Android: the IDE and build system, both UI systems (with Compose as your default), architecture, persistence, networking, concurrency, navigation, permissions, testing, and publishing. From here, deepen into WorkManager (background jobs), CameraX, Jetpack Glance (widgets), Compose Multiplatform, and performance with Baseline Profiles.
