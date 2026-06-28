# React Native + Expo — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I know some React on the web" (or even "I'm new to React") to "I can design, build, debug, and ship a production iOS + Android app with Expo" — without an internet connection. Every concept comes with prose that explains *what it is*, *the logic / why it works this way*, *what it's for and when you reach for it*, *how to use it*, the *key props/parameters*, *best practices*, and *gotchas* — followed by runnable, heavily-commented code. Read top-to-bottom the first time; afterward use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Expo SDK 54** (current as of June 2026; SDK 52/53 are recent and near-identical for this guide), **React Native 0.81+** with the **New Architecture** (Fabric + TurboModules + JSI) enabled by default, and **React 19**. Things worth knowing about the modern stack:
> - **New Architecture is the default** in SDK 52+. The legacy bridge is gone for new projects; UI and native calls go through JSI (direct C++ binding). Covered in §1 and §23.
> - **React 19** ships with React Native 0.77+ — Actions, `use()`, the `ref`-as-prop change, and improved Suspense all work on native. Cross-referenced against `REACT_19_GUIDE.md` throughout.
> - **Expo Router v4** (file-based routing built on React Navigation) is the default navigator; there is no root `App.tsx` anymore — the `app/` directory is the entry point.
> - **`expo-av` is splitting** into `expo-audio` and `expo-video`; **React Native DevTools** replaced Flipper; **FlashList** is the high-performance list of choice.
>
> Where behavior is version-sensitive it is flagged with **⚡ Version note**. To understand how this compares to writing a *truly native* Android app (Kotlin + Jetpack Compose), see the companion `ANDROID_STUDIO_GUIDE.md` — §1 below contrasts the two approaches directly. Confirm exact APIs at docs.expo.dev.

---

## Table of Contents

1. [React Native + Expo — The Big Picture](#1-react-native--expo--the-big-picture) **[B]**
2. [Setup: Create, Run & Tooling](#2-setup-create-run--tooling) **[B]**
3. [Project Structure & the `app/` Directory](#3-project-structure--the-app-directory) **[B]**
4. [Expo Router — File-Based Routing](#4-expo-router--file-based-routing) **[B/I]**
5. [Core Components](#5-core-components) **[B]**
6. [Lists & Performance: FlatList vs FlashList](#6-lists--performance-flatlist-vs-flashlist) **[I]**
7. [Styling: StyleSheet, Flexbox, Responsive & NativeWind](#7-styling-stylesheet-flexbox-responsive--nativewind) **[B/I]**
8. [Input, Forms & Keyboard Handling](#8-input-forms--keyboard-handling) **[I]**
9. [State & Data Fetching](#9-state--data-fetching) **[I]**
10. [Expo SDK — Device & Native APIs](#10-expo-sdk--device--native-apis) **[I]**
11. [Navigation Patterns & Params](#11-navigation-patterns--params) **[I]**
12. [Animations & Gestures](#12-animations--gestures) **[A]**
13. [Configuration: app.json / app.config.ts](#13-configuration-appjson--appconfigts) **[I]**
14. [Building & Shipping with EAS](#14-building--shipping-with-eas) **[A]**
15. [Debugging](#15-debugging) **[I]**
16. [Testing](#16-testing) **[I/A]**
17. [Secure Storage & Encrypted Caching (Banking-Grade)](#17-secure-storage--encrypted-caching-banking-grade) **[I/A]**
18. [Biometric Authentication (Face ID / Touch ID / Fingerprint)](#18-biometric-authentication-face-id--touch-id--fingerprint) **[I/A]**
19. [Device APIs & Permissions — The Complete Reference](#19-device-apis--permissions--the-complete-reference) **[I/A]**
20. [Mobile Security Hardening (Banking-Grade)](#20-mobile-security-hardening-banking-grade) **[A]**
21. [Production Release: App Store & Play Store](#21-production-release-app-store--play-store) **[A]**
22. [Tips, Tricks & Gotchas](#22-tips-tricks--gotchas) **[I/A]**
23. [Study Path & Build-to-Learn Projects](#23-study-path--build-to-learn-projects)

---

## 1. React Native + Expo — The Big Picture

Before you write a single component, you need an accurate mental model of *what is actually happening* when a React Native app runs. Many bugs and performance problems become obvious once you understand the architecture, and stay mysterious if you don't. This section builds that model from the ground up.

### 1.1 What React Native Is — and What It Is Not **[B]**

**What it is:** React Native (RN) lets you describe your UI with React components in TypeScript/JavaScript, and renders them as **real, native platform UI widgets**. A `<View>` becomes a `UIView` on iOS and an `android.view.View` on Android; a `<Text>` becomes a native text label. There is no HTML, CSS, or browser involved.

**What it is *not*:** *not* a WebView wrapper (that's a hybrid framework like Cordova/Ionic — a website inside a native shell). Because RN uses genuine native widgets, scrolling, text, and touch feel native *because they are*. The JavaScript only describes *what* the UI should be; the platform draws it.

**The logic / why this matters:** Your business logic (state, data fetching, events) runs in a JavaScript engine on the device; layout and rendering happen in native code. *How* the two worlds communicate is the single most important architectural fact about RN — and it changed dramatically with the "New Architecture."

### 1.2 How It Works — The Engine, the Renderer, and JSI **[I]**

The pipeline, top to bottom:

```text
Your TSX  →  Metro (bundles JS+assets)  →  Hermes (runs JS on device)
   ↕ JSI (direct, synchronous C++ calls — no JSON)
Fabric renderer (C++, Yoga layout) + TurboModules (lazy native)  →  iOS UIKit / Android Views (real pixels)
```

Each box is a real component you'll hear about and occasionally debug:

- **Metro** — the bundler (think Webpack for RN). Transforms TS/JSX, resolves imports, bundles assets, and streams live edits via **Fast Refresh** (§2).
- **Hermes** — Meta's mobile JS engine. Compiles JS to bytecode *ahead of time*, so the app starts faster and uses less memory. It's the default; you rarely touch it.
- **Yoga** — a cross-platform Flexbox layout engine in C. When you write `flex: 1`, Yoga computes the pixel positions — *why* RN styling is Flexbox-based (§7).
- **JSI (JavaScript Interface)** — the heart of the New Architecture: a lightweight C++ API letting JS hold direct references to C++ objects ("HostObjects") and call their methods *synchronously*, with no JSON serialization. This replaced the old "bridge."

### 1.3 The Old Bridge vs the New Architecture **[I]**

**The old "bridge" (legacy):** JS and native lived on separate threads and communicated via **serialized JSON messages** over an asynchronous queue. Every interaction was a JSON message marshaled, queued, and unmarshaled — the source of classic RN jank: a busy JS thread delayed UI, and high-frequency events (fast scrolling, JS-driven animations) flooded the bridge.

**The New Architecture (default in SDK 52+):** three pillars remove the bridge:

- **Fabric** — the new C++ renderer; builds/mutates the native view tree *synchronously* with JS via JSI. Result: fewer dropped frames, plus Concurrent React (Suspense, transitions) on native.
- **TurboModules** — native modules are lazily loaded and called through JSI; faster startup, no JSON-serialization tax.
- **JSI** — the underlying glue (above) that makes the other two possible.

**Why you care:** mostly you don't have to *do* anything — it's on by default. Two practical consequences: (1) some old native libraries aren't yet New-Architecture-compatible — check `reactnative.directory`, filter "New Architecture" (disabling is a last resort, §22); (2) Reanimated UI-thread animations (§12) stay smooth even when JS is busy, since they no longer cross a JSON bridge.

> **⚡ Version note:** In SDK 52 and 53, `newArchEnabled: true` is the default in `app.json` for new projects. The legacy bridge still exists as a fallback for incompatible libraries, but is being removed. Treat the New Architecture as the baseline.

### 1.4 React Native vs Native (Kotlin/Swift) vs Flutter — When to Choose Which **[B]**

This is a strategic decision, not a religious one. Each approach is *correct* for some projects and *wrong* for others.

| Dimension | React Native + Expo | Native (Kotlin/Compose + Swift/SwiftUI) | Flutter (Dart) |
|---|---|---|---|
| Language | TypeScript/JS | Kotlin **and** Swift (two codebases) | Dart |
| Code sharing iOS↔Android | One codebase | None — write everything twice | One codebase |
| UI rendering | Real native widgets | Real native widgets | Own rendering engine (Skia/Impeller) — pixels, not native widgets |
| Performance ceiling | Very high (New Arch); occasional native module needed | Highest possible | Very high |
| Access to brand-new OS APIs | Lags slightly (wait for Expo/community module) | Day-one | Lags slightly |
| Team fit | Web/React teams move fastest | Mobile specialists | Teams willing to learn Dart |
| Hiring pool | Huge (JS/React devs) | Smaller, more expensive | Growing |
| Hot reload DX | Excellent (Fast Refresh) | Improving (Compose previews) | Excellent |
| OTA JS updates | Yes (EAS Update, §14) | No (store review every time) | Limited |

**How to choose:**
- **React Native + Expo** when you have a web/React team, want one codebase, fast iteration, and OTA updates — the vast majority of business/consumer apps. The companion `ANDROID_STUDIO_GUIDE.md` shows the *native* Kotlin/Compose path so you can compare the same screen built both ways.
- **Fully native** for performance/hardware-critical apps (custom camera pipelines, AR, games), day-one OS-feature access, or when you already staff separate iOS/Android teams.
- **Flutter** when you want pixel-perfect identical UI across platforms (it draws its own widgets) and your team is happy in Dart.

> **Best practice:** You are not locked in. RN lets you drop down to native (write a TurboModule in Kotlin/Swift) for the 5% of your app that needs it, while keeping the other 95% in shared TS. This "mostly RN, native where it counts" pattern is extremely common in production.

### 1.5 Expo vs Bare React Native — and Why Expo in 2026 **[B]**

"React Native" is the framework. "Expo" is a *platform and toolchain* built on top of it. In 2026 Expo is the **officially recommended way to start any new RN project** (this recommendation comes from the React Native team itself). Here's the spectrum:

| | Managed Expo (start here) | Expo Dev Build (Expo + custom native) | Bare React Native |
|---|---|---|---|
| Create with | `npx create-expo-app` | `npx expo prebuild` | `npx @react-native-community/cli init` |
| Native folders (`ios/`, `android/`) | Hidden/generated | Generated by prebuild | Always present, hand-edited |
| Add native modules | Expo SDK + config plugins | Any native module | Any native module |
| Build in the cloud (EAS) | Yes | Yes | Yes (with config) |
| Test in Expo Go | Yes (SDK APIs only) | No — needs a dev build | No |
| OTA updates (EAS Update) | Yes | Yes | Yes |
| Recommended in 2026 | **Yes — default** | When you need custom native code | Rarely (only with legacy native infra) |

**What Expo actually gives you:**
- **The Expo SDK** — a large, version-matched library of device APIs (camera, location, notifications, secure storage, sensors, file system…) that "just work" without wiring native code (§10).
- **Expo Router** — Next.js-style file-based navigation (§4).
- **EAS** — cloud builds, store submission, and OTA updates (§14). You can build an iOS app *without owning a Mac*.
- **Config plugins** — modify native files (`Info.plist`, `AndroidManifest.xml`) declaratively from `app.json`, so you rarely open Xcode/Android Studio (§13).

> **⚡ Version note:** "Managed vs bare" is now better framed as "Continuous Native Generation (CNG)." You describe your app in `app.json`/plugins, and `npx expo prebuild` *generates* the `ios/` and `android/` folders on demand. You can delete and regenerate them anytime, so you get bare-level power without manually maintaining native files.

### 1.6 Expo Go vs Development Build — the Most Common Beginner Confusion **[B]**

- **Expo Go** is a pre-built app you install from the App Store / Play Store. You run `npx expo start`, scan a QR code, and your JS runs *inside* Expo Go. It's perfect for learning and for apps that only use Expo SDK APIs. Its limitation: it contains only the Expo SDK's native code — **you cannot use third-party native modules** (Stripe, Firebase, react-native-maps, etc.) in Expo Go.
- **Development Build** is *your own* version of Expo Go: a custom native shell that includes exactly the native dependencies your project uses. You build it once with EAS (or `npx expo run:ios/android`), install it on your device, then `npx expo start --dev-client` and develop as usual with Fast Refresh. This is the modern recommended workflow for any non-trivial app.

> **Gotcha:** The moment you `npx expo install` a library with native code that isn't bundled in Expo Go, your app will crash or warn in Expo Go. That is your signal to switch to a development build (§14). It is *not* a bug.

---

## 2. Setup: Create, Run & Tooling

### 2.1 Prerequisites **[B]**

```bash
# Node.js 20+ (LTS). Verify which one you actually have:
node --version           # should be 20.x or 22.x
# (Optional) install the EAS CLI globally for cloud builds; npx works too.
npm install -g eas-cli
# iOS development (simulator + local builds): macOS + Xcode 16+ with Command Line Tools.
#   You do NOT need a Mac to BUILD for iOS if you use EAS cloud builds (§14).
# Android development: Android Studio (for the emulator + SDK), JAVA_HOME set.
#   See ANDROID_STUDIO_GUIDE.md for installing the SDK, AVDs, and adb.
```

**The logic:** You can do an enormous amount with *just Node and a phone running Expo Go* — no Xcode, no Android Studio. You only need the heavy native toolchains when you build a development build or a production binary locally. EAS can do those in the cloud, which is why Windows users (no Xcode) can still ship iOS apps.

### 2.2 Create a New Project **[B]**

```bash
# Default template: Expo Router + TypeScript + a tabs example. Best starting point.
npx create-expo-app@latest MyApp
# Minimal template: no Router, a single App-style screen (good for tiny experiments).
npx create-expo-app@latest MyApp --template blank-typescript
# Explicitly request the tabs template (this is what the default gives you).
npx create-expo-app@latest MyApp --template tabs
```

After creation, the default project already runs. Open it and `cd MyApp`.

### 2.3 Running the App — the Dev Loop **[B]**

```bash
cd MyApp
# Start the Metro bundler + dev server. This is your main command.
npx expo start
#   In the terminal that opens you can press:
#     i  → open iOS simulator (Mac only)
#     a  → open Android emulator
#     w  → open in a web browser (Expo supports web too)
#     j  → open React Native DevTools (debugger) — see §15
#     r  → reload the app
#     s  → toggle between Expo Go and a development build
#   Or scan the QR code with the Expo Go app on a physical phone.
# Build + run on a simulator/emulator (triggers a local native build the first time):
npx expo run:ios
npx expo run:android
# The single most useful troubleshooting command — clears Metro's cache:
npx expo start --clear
```

> **Best practice:** When something behaves bizarrely (stale code, "module not found" after an install, weird red screens), your first move is `npx expo start --clear`. A surprising fraction of "bugs" are just stale bundler cache.

### 2.4 Metro Bundler — What It Does and How to Extend It **[I]**

Metro is to React Native what Webpack/Vite is to the web. It watches your files, bundles JS + assets, serves them to the device, and pushes live edits via **Fast Refresh** (which preserves component state across edits where possible). It also resolves **platform-specific files**: given `Button.tsx`, `Button.ios.tsx`, and `Button.android.tsx`, Metro automatically picks the right one per platform (§7).

You configure it in `metro.config.js`. You only touch this for non-default needs (SVG-as-component, monorepo resolution, custom asset extensions).

```js
// metro.config.js — example: enable importing .svg files as React components.
const { getDefaultConfig } = require('expo/metro-config'); // Expo's preset, not bare RN's
const config = getDefaultConfig(__dirname);
// Route .svg through a transformer that turns them into components:
config.transformer.babelTransformerPath = require.resolve('react-native-svg-transformer');
// Tell the resolver that .svg is source, not a static asset:
config.resolver.assetExts = config.resolver.assetExts.filter((ext) => ext !== 'svg');
config.resolver.sourceExts = [...config.resolver.sourceExts, 'svg'];
module.exports = config;
```

> **Gotcha:** Always start from `getDefaultConfig` (Expo's version). If you hand-write a Metro config from scratch you'll lose Expo's asset handling, the `app/` route resolution, and more.

### 2.5 Simulators & Emulators **[B]**

```bash
# iOS Simulator (Mac only) — list devices, then run on a specific one:
xcrun simctl list devices
npx expo run:ios --device "iPhone 16 Pro"
# Android Emulator — start an AVD from Android Studio's Device Manager first.
#   (Creating AVDs and using adb is covered in ANDROID_STUDIO_GUIDE.md.)
adb devices                                   # confirm the emulator/phone is detected
npx expo run:android --device emulator-5554   # target a specific device id
```

> **Tip for physical-device testing:** Your computer and phone must be on the **same Wi-Fi network** for the QR-code workflow. On locked-down networks use `npx expo start --tunnel` (routes through Expo's servers; slower but works anywhere).

---

## 3. Project Structure & the `app/` Directory

The default Expo project is organized around **convention**: the `app/` folder *is* your navigation tree. Understanding the conventions here unlocks Expo Router (§4).

```
MyApp/
├── app/                        # ★ Expo Router file-based routes — the heart of the app
│   ├── _layout.tsx             # Root layout (REQUIRED) — wraps everything, sets up providers
│   ├── index.tsx               # "/" — the home screen
│   ├── +not-found.tsx          # rendered for any unmatched URL (404)
│   │
│   ├── (tabs)/                 # Route GROUP (parentheses) — organizes files, NOT in the URL
│   │   ├── _layout.tsx         # defines the <Tabs> navigator
│   │   ├── index.tsx           # first tab
│   │   └── explore.tsx         # second tab
│   │
│   ├── details/
│   │   └── [id].tsx            # DYNAMIC route → /details/42  (id === "42")
│   │
│   └── modal.tsx               # a screen you can present as a modal sheet
│
├── components/                 # reusable UI (NOT routes — files here never become screens)
├── hooks/                      # custom React hooks (useAuth, useColorScheme…)
├── constants/                  # design tokens: Colors.ts, Spacing.ts
├── assets/                     # bundled images, fonts, icons
│   ├── images/
│   └── fonts/
│
├── app.json                    # static Expo config (name, icons, plugins…) — §13
├── app.config.ts               # OPTIONAL dynamic config (env-driven) — overrides app.json
├── eas.json                    # EAS Build / Submit / Update profiles — §14
├── tsconfig.json               # TypeScript config (Expo sets sensible defaults + path aliases)
├── metro.config.js             # bundler config (§2.4)
└── package.json
```

**The rules that govern `app/` (memorize these four):**
1. **Every** `.tsx`/`.jsx` file in `app/` becomes a **route**; the filename maps to a URL segment (`app/settings.tsx` → `/settings`).
2. **`_layout.tsx`** files define **navigators** (Stack, Tabs, Drawer) and persistent UI/providers that wrap their sibling/child routes.
3. Files/folders starting with **`_`** (private) or **`+`** (special, e.g. `+not-found.tsx`, `+html.tsx`) are **not** routes.
4. Folders in **`(parentheses)`** are **route groups** — they exist to organize and to attach a shared layout, but the group name is **stripped from the URL**.

> **Mental model vs the web:** If you know Next.js's App Router, this is nearly identical — `_layout.tsx` ≈ `layout.tsx`, `[id].tsx` ≈ `[id]/page.tsx`, `(group)` ≈ `(group)`. The difference: there is no server; routes are screens, and "navigation" is pushing native screens onto a stack. The web React Router knowledge cross-references to `REACT_19_GUIDE.md`.

> **Best practice:** Keep `app/` *thin* — each route file should mostly compose components from `components/` and call hooks from `hooks/`. Route files are about *wiring*, not implementation. This keeps screens testable and reusable.

---

## 4. Expo Router — File-Based Routing

Expo Router brings Next.js-style file-based routing to native. Under the hood it is built on **React Navigation** (the long-standing RN navigation library), so everything you learn maps to React Navigation concepts (stacks, tabs, screens, options) — but you declare them with files instead of imperative config.

### 4.1 The Root Layout (`app/_layout.tsx`) **[B]**

**What it is:** the single mandatory file that wraps your whole app. **Why it exists:** you need one place to install global providers (theme, query client, gesture handler, safe-area), load fonts, control the splash screen, and declare the *root navigator*. **When it runs:** once, before any route renders.

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';
import { useFonts } from 'expo-font';
import * as SplashScreen from 'expo-splash-screen';
import { useEffect } from 'react';
import { StatusBar } from 'expo-status-bar';
// Keep the native splash screen visible until we're ready (e.g. fonts loaded).
// Called at module scope so it runs immediately on app start.
SplashScreen.preventAutoHideAsync();
export default function RootLayout() {
  // useFonts returns [loaded, error]. While `loaded` is false, render nothing
  // so the user keeps seeing the splash instead of unstyled text.
  const [loaded] = useFonts({
    SpaceMono: require('../assets/fonts/SpaceMono-Regular.ttf'),
  });
  useEffect(() => {
    if (loaded) {
      SplashScreen.hideAsync(); // reveal the app once fonts are ready
    }
  }, [loaded]);
  if (!loaded) return null; // still loading fonts → keep splash up
  return (
    <>
      {/* expo-status-bar manages the OS status bar color; "auto" adapts to theme */}
      <StatusBar style="auto" />
      {/* The root navigator. A Stack pushes screens left-to-right (iOS) /
          bottom-to-top depending on platform/options. */}
      <Stack>
        {/* Register child routes and customize their options.
            name matches the file/folder name in app/. */}
        <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
        <Stack.Screen name="+not-found" />
        {/* modal.tsx will slide up as a modal sheet instead of pushing. */}
        <Stack.Screen name="modal" options={{ presentation: 'modal' }} />
      </Stack>
    </>
  );
}
```

**Key options you'll set on `<Stack.Screen>`:**
- `headerShown` — show/hide the native header bar.
- `title` — the header title text.
- `presentation` — `'card'` (default push), `'modal'`, `'transparentModal'`, `'fullScreenModal'`.
- `animation` — `'slide_from_right'`, `'fade'`, `'none'`, etc.
- `gestureEnabled` — allow the swipe-back gesture (iOS).

> **Gotcha:** Provider order matters. `GestureHandlerRootView` (needed for gestures/drawer, §12) and `SafeAreaProvider` (§5) usually wrap *outside* the `<Stack>`. A common mistake is forgetting `GestureHandlerRootView`, which makes swipe gestures silently do nothing.

### 4.2 Stack Navigator (`<Stack>`) **[B]**

A **stack** is a pile of screens. Navigating "forward" *pushes* a screen on top; going "back" *pops* it off. This is the default app navigation model (think drilling into a detail and backing out).

```tsx
// app/details/[id].tsx — a screen reached by pushing onto the stack.
import { Stack, useLocalSearchParams } from 'expo-router';
import { View, Text } from 'react-native';
export default function DetailsScreen() {
  // Read the dynamic [id] segment from the URL (see §4.4).
  const { id } = useLocalSearchParams<{ id: string }>();
  return (
    <View>
      {/* <Stack.Screen> rendered INSIDE a screen overrides that screen's options.
          Here we set a dynamic header title using the param. */}
      <Stack.Screen options={{ title: `Item #${id}` }} />
      <Text>Showing details for {id}</Text>
    </View>
  );
}
```

### 4.3 Tab Navigator (`<Tabs>`) **[B]**

A **tab navigator** shows a persistent bar (usually at the bottom) where each tab is its own screen/stack. You declare it in a group's `_layout.tsx`.

```tsx
// app/(tabs)/_layout.tsx
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons'; // bundled icon set, no extra install
export default function TabLayout() {
  return (
    <Tabs
      screenOptions={{
        tabBarActiveTintColor: '#007AFF', // color of the focused tab
        headerShown: false,               // let each screen manage its own header
      }}
    >
      <Tabs.Screen
        name="index" // → app/(tabs)/index.tsx
        options={{
          title: 'Home', // label under the icon
          // tabBarIcon receives { color, size, focused } so icons match the active state.
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="home" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="explore" // → app/(tabs)/explore.tsx
        options={{
          title: 'Explore',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="search" size={size} color={color} />
          ),
        }}
      />
    </Tabs>
  );
}
```

> **Best practice:** Put the tabs *inside* a `(tabs)` group and hide its header at the parent Stack (`<Stack.Screen name="(tabs)" options={{ headerShown: false }} />`). This avoids a double header (Stack header + Tab header).

### 4.4 Dynamic Routes `[id]` and Catch-All `[...slug]` **[I]**

**What they are:** square brackets in a filename create a *parameterized* segment. `[id].tsx` matches any single segment; `[...slug].tsx` matches *all* remaining segments as an array.

```
app/
└── posts/
    ├── index.tsx        →  /posts                 (the list)
    └── [id].tsx         →  /posts/123             (id = "123")
app/
└── docs/
    └── [...slug].tsx    →  /docs/a/b/c            (slug = ["a", "b", "c"])
```

```tsx
// app/posts/[id].tsx
import { useLocalSearchParams } from 'expo-router';
import { Text, View } from 'react-native';
export default function PostScreen() {
  // useLocalSearchParams reads BOTH the path param ([id]) AND any query string.
  // The generic types the result for autocomplete + safety.
  const { id } = useLocalSearchParams<{ id: string }>();
  return (
    <View>
      <Text>Post ID: {id}</Text>
    </View>
  );
}
```

> **Gotcha:** All URL params are **strings**, always. `/posts/42` gives you `id === "42"`, not the number `42`. Convert explicitly (`Number(id)`), and remember that complex objects cannot be passed this way (§11.3).

### 4.5 Navigating: `<Link>` and the `router` Object **[B]**

There are two ways to navigate: **declaratively** with `<Link>` (like an `<a href>`), and **imperatively** with the `router` object (inside event handlers, after an async action, etc.).

```tsx
import { Link, router } from 'expo-router';
import { Pressable, Text } from 'react-native';
/* ---------- Declarative: <Link> ---------- */
// Simple string href:
<Link href="/posts/42">
  <Text>Go to Post 42</Text>
</Link>;
// Object href with typed params (preferred — survives route refactors):
<Link href={{ pathname: '/posts/[id]', params: { id: '42' } }}>
  <Text>Post 42</Text>
</Link>;
// Make the whole Link a button by passing asChild + a Pressable:
<Link href="/settings" asChild>
  <Pressable>
    <Text>Settings</Text>
  </Pressable>
</Link>;
/* ---------- Imperative: router ---------- */
function handlePress() {
  router.push('/posts/42');               // push a new screen onto the stack
  router.replace('/login');               // replace current screen (no "back" to it)
  router.back();                          // pop the top screen
  router.dismiss();                       // dismiss a modal / pop to a previous group
  router.dismissAll();                    // pop every screen back to the first
  router.push({ pathname: '/posts/[id]', params: { id: '42' } }); // with params
  router.setParams({ q: 'updated' });     // change the CURRENT screen's params
}
```

**`push` vs `replace` vs `navigate`:** `push` always adds to the stack (you can have the same screen twice). `replace` swaps the current screen (used after login so the user can't go "back" to the login form). `router.navigate` is smart — it reuses an existing screen of that route if present instead of stacking a duplicate.

### 4.6 Route Groups `(group)` **[I]**

Route groups organize routes and attach shared layouts *without* adding a URL segment. The killer use case: completely different layouts for authenticated vs unauthenticated users.

```
app/
├── (auth)/
│   ├── _layout.tsx     # a bare Stack — no tabs, no app chrome
│   ├── login.tsx       # → /login
│   └── register.tsx    # → /register
│
└── (app)/
    ├── _layout.tsx     # the main app shell (providers, etc.)
    └── (tabs)/
        ├── _layout.tsx # the tab bar
        └── home.tsx    # → /home   (note: neither (app) nor (tabs) is in the URL)
```

This means `/login` and `/home` live under totally different layouts, yet the URLs stay clean. Combine with the auth-guard pattern (§11.5) to gate access.

### 4.7 Typed Routes **[I]**

> **⚡ Version note:** Expo Router can generate TypeScript types for every route so `href` strings are autocompleted and typo'd routes become compile errors. Enable it once:

```json
// app.json
{
  "expo": {
    "experiments": {
      "typedRoutes": true
    }
  }
}
```

```tsx
// After enabling + restarting, href is fully typed:
import { Link } from 'expo-router';
<Link href="/settings">Settings</Link>;   // ✅ autocompletes; ❌ /setttings is a TS error
```

> **Best practice:** Turn this on at the start of every project. It catches an entire class of "navigated to a route that doesn't exist" bugs at build time instead of runtime.

---

## 5. Core Components

React Native gives you a small set of primitive components that map to native widgets. Master these and you can build almost any screen. The mental shift from the web: there is no `<div>`, `<span>`, `<p>`, `<button>`, or `<img>` — there are typed components, and **only certain components can contain certain children** (text must live in `<Text>`).

### 5.1 `View` — the Container **[B]**

**What it is:** the fundamental layout box, equivalent to a web `<div>`. **Maps to:** `UIView` (iOS) / `android.view.View` (Android). **What it's for:** grouping, layout (Flexbox), backgrounds, borders, shadows. **It does not scroll** and cannot directly contain raw text.

```tsx
import { View, StyleSheet } from 'react-native';
export default function Card({ children }: { children: React.ReactNode }) {
  return <View style={styles.card}>{children}</View>;
}
const styles = StyleSheet.create({
  card: {
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 16,
    // Flexbox is the ONLY layout system (see §7). Default direction is COLUMN.
    flexDirection: 'row',
    alignItems: 'center',
    gap: 12, // gap works in RN, like CSS gap
    // Shadows are PLATFORM-SPECIFIC — this is a classic gotcha:
    shadowColor: '#000',                 // iOS shadow ↓
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,                        // Android shadow (single number)
  },
});
```

> **Key props:** `style`, `pointerEvents` (`'none'` lets touches pass through), `onLayout` (fires with measured size — useful for animations), `accessible`/`accessibilityLabel` (screen-reader support — always add these to interactive views).

> **Gotcha:** iOS shadows use the four `shadow*` props; Android ignores them and uses `elevation`. To get a consistent shadow you must set both (or use `Platform.select`, §7).

### 5.2 `Text` — All Text Lives Here **[B]**

**The hard rule:** every string you render must be inside a `<Text>`. Putting a raw string directly in a `<View>` throws `Text strings must be rendered within a <Text> component`. **Why:** native text rendering is a distinct widget with its own measurement; RN enforces the boundary explicitly.

```tsx
import { Text, StyleSheet } from 'react-native';
// Truncation + interactivity:
<Text
  style={styles.heading}
  numberOfLines={2}        // clamp to 2 lines
  ellipsizeMode="tail"     // "…" at the end ('head' | 'middle' | 'tail' | 'clip')
  selectable               // long-press to select/copy
  onPress={() => {}}       // Text itself can be tappable (e.g. links)
  accessibilityRole="header"
>
  Hello World
</Text>;
// Nested Text inherits + overrides style — like <span> inside <p>:
<Text style={styles.body}>
  This is <Text style={{ fontWeight: 'bold' }}>bold</Text> and{' '}
  <Text style={{ color: '#007AFF' }} onPress={() => router.push('/terms')}>
    a link
  </Text>.
</Text>;
const styles = StyleSheet.create({
  heading: { fontSize: 22, fontWeight: '700' },
  body: { fontSize: 16, lineHeight: 22, color: '#333' },
});
```

> **Gotcha:** Style inheritance is *limited*. Unlike CSS, a `color` set on a parent `<View>` does **not** cascade to `<Text>`. Only nested `<Text>` inside `<Text>` inherits text styles. Set font styles on the `<Text>` itself.

### 5.3 `ScrollView` — Scroll Everything That Fits in Memory **[B]**

**What it is:** a scrollable container that renders **all** its children at once. **When to use:** content that is bounded and modest (a settings page, a form, a few cards). **When NOT to use:** long/unbounded lists — every child mounts immediately, so a 1,000-item `ScrollView` will be slow and memory-hungry. For those, use `FlatList`/`FlashList` (§6).

```tsx
import { ScrollView, RefreshControl, Text } from 'react-native';
<ScrollView
  style={{ flex: 1 }}                         // the scroll viewport fills its parent
  contentContainerStyle={{ padding: 16, gap: 12 }} // styles the INNER content, not the viewport
  showsVerticalScrollIndicator={false}
  horizontal={false}                          // true → horizontal scrolling
  keyboardShouldPersistTaps="handled"         // taps on buttons work while keyboard is open
  keyboardDismissMode="on-drag"               // dragging dismisses the keyboard
  refreshControl={                            // pull-to-refresh
    <RefreshControl refreshing={refreshing} onRefresh={onRefresh} />
  }
>
  {items.map((item) => (
    <Text key={item.id}>{item.name}</Text>
  ))}
</ScrollView>;
```

> **Key prop to understand: `contentContainerStyle` vs `style`.** `style` styles the scroll *frame* (its size/position). `contentContainerStyle` styles the *scrollable content* (padding, gap, `flexGrow: 1` to center sparse content). Beginners constantly put `padding` on `style` and wonder why it clips — put layout/padding on `contentContainerStyle`.

### 5.4 `Image` (use `expo-image`) **[B]**

> **⚡ Version note:** Prefer **`expo-image`** over RN's built-in `<Image>`. It has far better caching, blurhash/thumbhash placeholders, smooth transitions, and works better with the New Architecture.

```bash
npx expo install expo-image
```

```tsx
import { Image } from 'expo-image';
<Image
  // Remote image:
  source={{ uri: 'https://example.com/photo.jpg' }}
  // Local image instead: source={require('../assets/photo.png')}
  style={{ width: 200, height: 200, borderRadius: 100 }} // dimensions are REQUIRED for remote
  contentFit="cover"                  // like CSS object-fit: cover | contain | fill | none
  transition={300}                    // ms cross-fade when the image loads
  placeholder={{ blurhash: 'L6PZfSi_.AyE_3t7t7R**0o#DgR4' }} // tiny blurred preview
  cachePolicy="memory-disk"           // cache in RAM + on disk (most aggressive)
  priority="high"
/>;
```

> **Gotchas:** (1) Local images MUST use `require()` — a string path like `{ uri: './photo.png' }` does not work because Metro needs to *see* the require at build time to bundle the asset. (2) Remote images have no intrinsic size; you must give explicit `width`/`height` or they render 0×0. (3) For local images, Metro auto-picks `@2x`/`@3x` variants on high-DPI screens.

### 5.5 `Pressable` — the Modern Touchable **[B]**

**What it is:** the recommended way to handle taps; it gives you the *press state* so you can style feedback precisely. **Why it replaced `TouchableOpacity`:** `Pressable` is more flexible — `style` and `children` can be functions of `{ pressed }`, and it supports `hitSlop`, `onLongPress`, `onPressIn/Out`, and `delayLongPress`.

```tsx
import { Pressable, Text, StyleSheet } from 'react-native';
<Pressable
  onPress={() => console.log('pressed')}
  onLongPress={() => console.log('long press')}
  onPressIn={() => {}}                 // fires the instant the finger touches
  onPressOut={() => {}}                // fires on release
  disabled={false}
  // style can be a function of press state → dynamic feedback without state hooks:
  style={({ pressed }) => [styles.button, pressed && styles.buttonPressed]}
  // Expand the *touchable* area beyond the visual bounds (great for small icons):
  hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
  // Android ripple effect:
  android_ripple={{ color: '#005EC4' }}
  accessibilityRole="button"
  accessibilityLabel="Submit form"
>
  {/* children can ALSO be a function of press state */}
  {({ pressed }) => (
    <Text style={[styles.label, pressed && { opacity: 0.7 }]}>Press Me</Text>
  )}
</Pressable>;
const styles = StyleSheet.create({
  button: { backgroundColor: '#007AFF', borderRadius: 8, padding: 12 },
  buttonPressed: { backgroundColor: '#005EC4' },
  label: { color: '#fff', fontWeight: '600', textAlign: 'center' },
});
```

> **Best practice:** Use `Pressable` for new code. Add `hitSlop` to anything smaller than ~44×44 pt (Apple's minimum touch target) so users don't miss. Always set `accessibilityRole="button"` and a label.

### 5.6 `TouchableOpacity` (Legacy but Common) **[B]**

Simpler and older: it fades (dims) the content while pressed. Still everywhere in tutorials and existing code.

```tsx
import { TouchableOpacity, Text } from 'react-native';
<TouchableOpacity onPress={handlePress} activeOpacity={0.7}>
  <Text>Tap me</Text>
</TouchableOpacity>;
```

> **When to choose:** new code → `Pressable`. Maintaining old code → `TouchableOpacity` is fine. There's also `TouchableHighlight` (shows an underlay color) and `TouchableWithoutFeedback` (no visual feedback — used to dismiss keyboards, §8).

### 5.7 `TextInput` — Text Entry **[B]**

The controlled-input primitive. **The logic:** like a web `<input>`, you keep the value in state and update it on every keystroke via `onChangeText`. The huge number of props mostly configure the on-screen keyboard and autofill behavior.

```tsx
import { TextInput, StyleSheet } from 'react-native';
import { useState, useRef } from 'react';
function SearchBar() {
  const [value, setValue] = useState('');
  const inputRef = useRef<TextInput>(null); // for imperative .focus()/.blur()
  return (
    <TextInput
      value={value}
      onChangeText={setValue}             // fires on EVERY keystroke (controlled)
      placeholder="Search..."
      placeholderTextColor="#999"
      style={styles.input}
      ref={inputRef}
      /* ----- keyboard configuration ----- */
      keyboardType="email-address"        // 'default'|'numeric'|'phone-pad'|'url'|'email-address'
      returnKeyType="search"              // label on the keyboard's return key
      autoCapitalize="none"               // 'none'|'sentences'|'words'|'characters'
      autoCorrect={false}
      autoComplete="email"                // OS autofill hint (email, password, one-time-code…)
      textContentType="emailAddress"      // iOS-specific autofill hint
      secureTextEntry={false}             // true → password dots
      /* ----- events ----- */
      onSubmitEditing={({ nativeEvent }) => handleSearch(nativeEvent.text)}
      onFocus={() => {}}
      onBlur={() => {}}
      /* ----- multiline ----- */
      multiline={false}                   // true → grows; set textAlignVertical="top" on Android
    />
  );
}
const styles = StyleSheet.create({
  input: {
    borderWidth: 1,
    borderColor: '#ccc',
    borderRadius: 8,
    padding: 12,
    fontSize: 16,
    color: '#000', // ALWAYS set color — default can be invisible in dark mode
  },
});
```

> **Gotcha:** Set an explicit `color` on inputs. On some devices/dark themes the default text color matches the background and the user types invisibly. Also: for `multiline` on Android, add `textAlignVertical="top"` so the cursor starts at the top.

### 5.8 `SafeAreaView` / Safe Area Insets **[B]**

**What it is:** padding that keeps content clear of the notch, Dynamic Island, status bar, and home indicator. **Why:** device chrome varies wildly; hardcoding `paddingTop: 44` breaks on the next device. **How:** use the `react-native-safe-area-context` library (Expo includes it) — *not* RN's built-in `SafeAreaView`, which is unreliable.

```tsx
// In the root layout — provider must wrap the whole app once:
import { SafeAreaProvider } from 'react-native-safe-area-context';
export default function RootLayout() {
  return <SafeAreaProvider>{/* navigator / screens */}</SafeAreaProvider>;
}
```

```tsx
// Option A: declarative SafeAreaView in a screen:
import { SafeAreaView } from 'react-native-safe-area-context';
export default function HomeScreen() {
  return (
    <SafeAreaView style={{ flex: 1 }} edges={['top', 'bottom']}>
      {/* content is inset away from notch/home indicator */}
    </SafeAreaView>
  );
}
// Option B: the hook, for fine control (e.g. a custom header):
import { useSafeAreaInsets } from 'react-native-safe-area-context';
function Header() {
  const insets = useSafeAreaInsets(); // { top, bottom, left, right } in px
  return <View style={{ paddingTop: insets.top }}>{/* ... */}</View>;
}
```

> **Gotcha:** Never import `SafeAreaView` from `'react-native'`. Always import from `'react-native-safe-area-context'`. The built-in one only handles older iPhones.

### 5.9 `ActivityIndicator` — Loading Spinner **[B]**

```tsx
import { ActivityIndicator, View } from 'react-native';
{isLoading && (
  <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
    <ActivityIndicator size="large" color="#007AFF" />
  </View>
)}
```

### 5.10 `Modal` — Built-In Overlay **[B]**

```tsx
import { Modal, View, Text, Pressable } from 'react-native';
import { useState } from 'react';
function MyModal() {
  const [visible, setVisible] = useState(false);
  return (
    <>
      <Pressable onPress={() => setVisible(true)}>
        <Text>Open Modal</Text>
      </Pressable>
      <Modal
        visible={visible}
        transparent                              // let the dimmed background show through
        animationType="slide"                    // 'fade' | 'slide' | 'none'
        onRequestClose={() => setVisible(false)} // REQUIRED on Android (hardware back button)
        statusBarTranslucent                     // Android: draw under the status bar
      >
        <View style={{ flex: 1, justifyContent: 'flex-end', backgroundColor: 'rgba(0,0,0,0.4)' }}>
          <View style={{ backgroundColor: '#fff', borderTopLeftRadius: 20, borderTopRightRadius: 20, padding: 20 }}>
            <Text>Modal Content</Text>
            <Pressable onPress={() => setVisible(false)}>
              <Text>Close</Text>
            </Pressable>
          </View>
        </View>
      </Modal>
    </>
  );
}
```

> **Best practice:** For most "sheet" UIs prefer Expo Router's `presentation: 'modal'` (it integrates with navigation/back) or the `@gorhom/bottom-sheet` library (draggable sheets with gestures). Reach for the bare `<Modal>` only for simple, self-contained overlays. Always handle `onRequestClose` or the Android back button won't dismiss it.

### 5.11 How These Map to Native — Quick Reference **[I]**

| RN component | iOS native | Android native | Web `<…>` analogy |
|---|---|---|---|
| `View` | `UIView` | `ViewGroup`/`View` | `div` |
| `Text` | `UILabel`/text | `TextView` | `span`/`p` |
| `ScrollView` | `UIScrollView` | `ScrollView` | scrollable `div` |
| `FlatList` | `UICollectionView`-style virtualization | `RecyclerView`-style | virtualized list |
| `Image` (`expo-image`) | `UIImageView` | `ImageView` | `img` |
| `TextInput` | `UITextField`/`UITextView` | `EditText` | `input`/`textarea` |
| `Pressable` | gesture-recognized `UIView` | touch-handling `View` | `button` |
| `Modal` | presented view controller | `Dialog`/`Window` | modal `dialog` |

> Cross-reference: in `ANDROID_STUDIO_GUIDE.md` you'll see these same widgets (`TextView`, `RecyclerView`, `EditText`) used directly in Kotlin/Compose. RN sits one abstraction layer above them.

---

## 6. Lists & Performance: FlatList vs FlashList

Lists are where naive RN apps die. The fix is **virtualization/recycling**: only render the rows currently on (or near) screen, and reuse row components as the user scrolls. This section covers the three list tools and exactly when to use each.

### 6.1 `FlatList` — the Built-In Virtualized List **[I]**

**What it is:** RN's built-in list that virtualizes — it mounts only visible rows (plus a buffer) and unmounts off-screen ones. **When to use:** the default for any list beyond a handful of items, especially variable-height rows. **The logic:** instead of rendering 10,000 rows up front (like `ScrollView` would), it renders ~20 and swaps content as you scroll, keeping memory flat.

```tsx
import { FlatList, Text, View, StyleSheet, ActivityIndicator, RefreshControl } from 'react-native';
import { useCallback } from 'react';
type Item = { id: string; title: string; subtitle: string };
const DATA: Item[] = [
  { id: '1', title: 'First', subtitle: 'subtitle one' },
  { id: '2', title: 'Second', subtitle: 'subtitle two' },
  // ...
];
// Defining the row OUTSIDE render + memoizing avoids re-creating it each render.
const ItemCard = ({ title, subtitle }: Omit<Item, 'id'>) => (
  <View style={styles.card}>
    <Text style={styles.title}>{title}</Text>
    <Text style={styles.subtitle}>{subtitle}</Text>
  </View>
);
export default function MyList() {
  // useCallback so renderItem identity is stable (helps FlatList skip work).
  const renderItem = useCallback(
    ({ item }: { item: Item }) => <ItemCard title={item.title} subtitle={item.subtitle} />,
    []
  );
  return (
    <FlatList
      data={DATA}
      keyExtractor={(item) => item.id}      // MUST return a unique STRING per row
      renderItem={renderItem}
      /* ---------- performance props (see §6.5) ---------- */
      initialNumToRender={10}               // rows rendered on first paint
      maxToRenderPerBatch={10}              // rows rendered per scroll batch
      windowSize={5}                        // render window = 5× the visible area
      removeClippedSubviews                 // detach off-screen views (Android win)
      getItemLayout={(_, index) => ({       // FIXED-height rows → skip measurement (big win)
        length: 80,                         // row height in px
        offset: 80 * index,
        index,
      })}
      /* ---------- UX slots ---------- */
      ListHeaderComponent={<Text style={styles.header}>My Items</Text>}
      ListFooterComponent={null}
      ListEmptyComponent={<Text>No items found.</Text>}
      ItemSeparatorComponent={() => <View style={styles.separator} />}
      /* ---------- infinite scroll + pull to refresh ---------- */
      onEndReached={() => {/* fetchMore() */}}
      onEndReachedThreshold={0.5}           // fire when within 50% of the bottom
      refreshControl={<RefreshControl refreshing={false} onRefresh={() => {}} />}
    />
  );
}
const styles = StyleSheet.create({
  card: { padding: 16, backgroundColor: '#fff', height: 80, justifyContent: 'center' },
  title: { fontSize: 16, fontWeight: '600' },
  subtitle: { fontSize: 14, color: '#666', marginTop: 4 },
  header: { fontSize: 20, fontWeight: 'bold', padding: 16 },
  separator: { height: 1, backgroundColor: '#eee' },
});
```

> **Key props to know:** `data`, `renderItem`, `keyExtractor` (required for correctness), `getItemLayout` (the single biggest perf lever for fixed-height rows), `onEndReached`/`onEndReachedThreshold` (infinite scroll), `numColumns` (grid), `horizontal` (carousel), `inverted` (chat — newest at bottom), `stickyHeaderIndices`.

### 6.2 `SectionList` — Grouped Lists **[I]**

Like `FlatList` but for grouped data with section headers (a contacts list A–Z, settings grouped into sections).

```tsx
import { SectionList, Text } from 'react-native';
const SECTIONS = [
  { title: 'Fruits', data: ['Apple', 'Banana'] },
  { title: 'Veggies', data: ['Carrot', 'Broccoli'] },
];
<SectionList
  sections={SECTIONS}
  keyExtractor={(item, index) => item + index}
  renderItem={({ item }) => <Text>{item}</Text>}
  renderSectionHeader={({ section: { title } }) => (
    <Text style={{ fontWeight: 'bold', backgroundColor: '#f0f0f0', padding: 8 }}>{title}</Text>
  )}
  stickySectionHeadersEnabled
/>;
```

### 6.3 `FlashList` (Shopify) — the High-Performance List **[A]**

> **⚡ Version note:** `@shopify/flash-list` is the go-to for large/complex lists. Instead of virtualization (mount/unmount), it **recycles** row components (like Android's `RecyclerView` — see `ANDROID_STUDIO_GUIDE.md`), reusing the same view objects for new data. The result is dramatically less work while scrolling and far fewer blank frames.

```bash
npx expo install @shopify/flash-list
```

```tsx
import { FlashList } from '@shopify/flash-list';
// Drop-in: nearly identical API to FlatList. The one extra prop is estimatedItemSize.
<FlashList
  data={DATA}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <ItemCard {...item} />}
  estimatedItemSize={80}   // REQUIRED: your best guess at average row height (px)
  onEndReached={() => {}}
  onEndReachedThreshold={0.5}
/>;
```

> **Why `estimatedItemSize` matters:** FlashList uses it to compute how many rows fit and how to recycle. A good estimate = smooth scrolling; a wildly wrong one = visible blanking. Measure a typical row once and use that number. You generally do **not** need `getItemLayout` with FlashList.

### 6.4 Decision Table — Which List to Use **[I]**

| Scenario | Use |
|---|---|
| < ~30 short items, bounded | `ScrollView` (simplest) |
| 30–1000 items, uniform height | `FlashList` (fastest) or `FlatList` + `getItemLayout` |
| 30–1000 items, variable height | `FlatList` (no `getItemLayout`) or `FlashList` |
| Grouped/sectioned data | `SectionList` |
| Horizontal carousel | `FlatList`/`FlashList` with `horizontal` |
| Chat (newest at bottom) | `FlatList`/`FlashList` with `inverted` |
| 10,000+ items, heavy rows | `FlashList` (effectively mandatory) |

### 6.5 List Performance Rules (Memorize) **[A]**

1. **`keyExtractor` must return a stable, unique string.** Using the array index as a key breaks recycling when items reorder/insert.
2. **`getItemLayout` for fixed-height rows** (FlatList) eliminates async measurement — the biggest single win.
3. **Memoize the row** with `React.memo` and keep `renderItem` stable with `useCallback`, so unrelated state changes don't re-render every row.
4. **Never nest a `FlatList`/`FlashList` inside a `ScrollView`** of the same scroll direction — it disables virtualization and warns "VirtualizedList... slow to update." Use `ListHeaderComponent`/`ListFooterComponent` for content above/below the list instead.
5. **Keep row components light** — avoid heavy shadows, large images without `expo-image` caching, and inline anonymous functions/objects in the row (they defeat memoization).

```tsx
// Optimized pattern: memoized row + stable renderItem.
const Row = React.memo(({ item }: { item: Item }) => (
  <View style={styles.card}>
    <Text>{item.title}</Text>
  </View>
));
const renderItem = useCallback(({ item }: { item: Item }) => <Row item={item} />, []);
```

---

## 7. Styling: StyleSheet, Flexbox, Responsive & NativeWind

There is **no CSS** in React Native. You style with JavaScript objects whose keys are a subset of CSS (camelCased: `backgroundColor`, not `background-color`). Values are usually unitless numbers interpreted as density-independent pixels. No cascade, no selectors, no media queries, no pseudo-classes — styles are explicit and local. This is *less* magic and, once you adjust, more predictable.

### 7.1 `StyleSheet.create` **[B]**

**What it is:** the canonical way to define styles. **Why use it over plain objects:** it validates keys in development (typos error early), and historically optimized style passing to native. **How:** define styles *outside* the component so the object isn't re-created on every render; reference by name; compose with arrays.

```tsx
import { StyleSheet, View, Text } from 'react-native';
export default function Card() {
  return (
    <View style={styles.container}>
      {/* Arrays compose styles left→right; later entries win. Falsey entries are ignored. */}
      <Text style={[styles.text, styles.bold]}>Hello</Text>
    </View>
  );
}
const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5', padding: 16 },
  text: { fontSize: 16, color: '#333' },
  bold: { fontWeight: '700' },
});
```

> **Best practice:** Compose with arrays for conditional styles: `style={[styles.btn, disabled && styles.btnDisabled]}`. A `false`/`undefined`/`null` entry is simply skipped, so `cond && styles.x` is the idiom for "apply when true."

### 7.2 Flexbox in RN — Critical Differences From the Web **[B]**

RN uses Flexbox (via the Yoga engine) for *all* layout, but the **defaults differ from the web**. Memorize this table — it explains 90% of "why is my layout wrong" confusion for web developers:

| Property | Web default | React Native default |
|---|---|---|
| `flexDirection` | `row` | **`column`** (top-to-bottom!) |
| `alignContent` | `stretch` | `flex-start` |
| `flexShrink` | `1` | **`0`** |
| `flexBasis` | `auto` | `auto` |
| `position` | `static` | `relative` |
| `display` | `block`/`inline` | `flex` |

**The single biggest surprise:** the main axis is **vertical** by default (`flexDirection: 'column'`), because phones are tall. So `justifyContent` controls *vertical* distribution and `alignItems` controls *horizontal* — until you set `flexDirection: 'row'`, which swaps them.

```tsx
// Center a child both axes — the most common pattern in the language:
<View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
  <Text>Centered</Text>
</View>;
// A horizontal row with spacing. `gap` works in RN like CSS gap.
<View style={{ flexDirection: 'row', gap: 8 }}>
  <View style={{ flex: 1, backgroundColor: 'red' }} />   {/* takes 1 share */}
  <View style={{ flex: 2, backgroundColor: 'blue' }} />  {/* takes 2 shares */}
</View>;
const styles = StyleSheet.create({
  screen: { flex: 1 }, // fill the parent — the foundation of almost every screen
  row: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' },
  // Absolute overlay (full-bleed):
  overlay: {
    position: 'absolute',
    top: 0, left: 0, right: 0, bottom: 0,
    backgroundColor: 'rgba(0,0,0,0.5)',
  },
});
```

> **Gotcha — `flex: 1` "not working":** flex distributes the *parent's* free space. If the parent has no defined height (or isn't itself `flex: 1`), the child has no space to fill. The fix is almost always "make the parent `flex: 1` too" — flex must bubble up from a sized root (your screen container).

> **Units:** numbers are density-independent pixels. Percentages (`'50%'`) work for width/height. There is no `rem`/`em`/`vh`/`vw` — use `useWindowDimensions` for viewport-relative sizing (§7.3).

### 7.3 Responsive Design: `Dimensions` & `useWindowDimensions` **[I]**

```tsx
import { Dimensions, useWindowDimensions, View, Text } from 'react-native';
// STATIC — read once at module load. Does NOT update on rotation / iPad resize. Avoid for layout.
const { width, height } = Dimensions.get('window'); // app window
const { width: screenW } = Dimensions.get('screen'); // full physical screen
// DYNAMIC — the hook re-renders on rotation, multitasking resize, and font-scale changes.
function ResponsiveCard() {
  const { width, height, fontScale } = useWindowDimensions();
  const isLandscape = width > height;
  const numColumns = width > 600 ? 3 : 2; // simple "tablet vs phone" breakpoint
  return (
    <View style={{ width: width * 0.9 }}>
      {/* Respect the user's OS text-size accessibility setting: */}
      <Text style={{ fontSize: 16 * fontScale }}>Scaled Text</Text>
    </View>
  );
}
```

> **Best practice:** Prefer `useWindowDimensions` over `Dimensions.get` for anything that affects layout — otherwise rotating the device leaves stale sizes. Use `> 600`-style width breakpoints for phone/tablet branching.

### 7.4 Platform-Specific Code **[I]**

Three mechanisms, from inline to file-level:

```tsx
import { Platform, StyleSheet } from 'react-native';
// 1. Platform.OS — a string: 'ios' | 'android' | 'web'
if (Platform.OS === 'android') {/* ... */}
// 2. Platform.select — pick a value per platform (great for the shadow problem):
const shadow = Platform.select({
  ios: { shadowColor: '#000', shadowOffset: { width: 0, height: 2 }, shadowOpacity: 0.15, shadowRadius: 4 },
  android: { elevation: 4 },
  default: {}, // web / fallback
});
// 3. Platform.Version — OS version (number on Android, string on iOS)
```

```tsx
// 4. File-based platform split — Metro auto-resolves the right file:
//    Button.ios.tsx       ← used on iOS
//    Button.android.tsx   ← used on Android
//    Button.tsx           ← fallback (e.g. web)
import Button from './Button'; // you import the base name; Metro picks the variant
```

> **When to use which:** small divergences → `Platform.select`. Entirely different implementations (a native iOS-only widget vs an Android one) → separate `.ios.tsx`/`.android.tsx` files. Cross-reference: this mirrors how truly native apps must implement each platform separately (see `ANDROID_STUDIO_GUIDE.md`); RN lets you share the common 95% and split only the divergent 5%.

### 7.5 NativeWind — Tailwind CSS for React Native **[I]**

> **⚡ Version note:** **NativeWind v4** is stable in 2026 and works with Expo SDK 52/53/54. It lets you use Tailwind utility classes via a `className` prop; at build time it compiles them to RN `StyleSheet` objects (no runtime CSS). If you know Tailwind on the web (`TAILWIND_CHEATSHEET.md`), the classes are the same.

```bash
npx expo install nativewind tailwindcss
npx tailwindcss init
```

```js
// tailwind.config.js
module.exports = {
  content: ['./app/**/*.{tsx,ts}', './components/**/*.{tsx,ts}'],
  presets: [require('nativewind/preset')],
  theme: { extend: {} },
};
```

```js
// babel.config.js — add the NativeWind jsxImportSource + babel plugin
module.exports = {
  presets: [['babel-preset-expo', { jsxImportSource: 'nativewind' }], 'nativewind/babel'],
};
```

```tsx
import { View, Text, Pressable } from 'react-native';
export default function Card() {
  return (
    <View className="flex-1 bg-white rounded-xl p-4 shadow-md">
      <Text className="text-xl font-bold text-gray-900">Hello NativeWind</Text>
      <Pressable className="mt-4 bg-blue-500 rounded-lg px-4 py-2 active:opacity-70">
        <Text className="text-white font-semibold text-center">Press Me</Text>
      </Pressable>
    </View>
  );
}
```

> **When to choose NativeWind vs StyleSheet:** NativeWind is fantastic for speed and consistency if your team already thinks in Tailwind. Plain `StyleSheet` has zero dependencies and is what every RN tutorial uses. Both are valid; don't mix them haphazardly in one file.

---

## 8. Input, Forms & Keyboard Handling

Forms on mobile are harder than on the web because the **on-screen keyboard covers part of the screen**. Most of this section is about keeping the focused input visible and making multi-field forms pleasant.

### 8.1 `KeyboardAvoidingView` — Push Content Above the Keyboard **[I]**

**What it is:** a container that resizes/repositions its children when the keyboard appears so the focused input isn't hidden. **The logic:** iOS and Android handle the keyboard differently, so the right `behavior` differs per platform.

```tsx
import {
  KeyboardAvoidingView, Platform, ScrollView, TextInput, Text, Pressable, StyleSheet,
} from 'react-native';
export default function LoginForm() {
  return (
    <KeyboardAvoidingView
      style={{ flex: 1 }}
      // iOS: 'padding' adds padding equal to keyboard height.
      // Android: usually 'height' (or rely on windowSoftInputMode in app.json).
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      keyboardVerticalOffset={90} // subtract a fixed header height so it lines up
    >
      <ScrollView
        contentContainerStyle={styles.container}
        keyboardShouldPersistTaps="handled" // taps on buttons work while keyboard is open
      >
        <TextInput style={styles.input} placeholder="Email" keyboardType="email-address" autoCapitalize="none" />
        <TextInput style={styles.input} placeholder="Password" secureTextEntry />
        <Pressable style={styles.button}>
          <Text style={styles.buttonText}>Login</Text>
        </Pressable>
      </ScrollView>
    </KeyboardAvoidingView>
  );
}
const styles = StyleSheet.create({
  container: { flexGrow: 1, padding: 24, justifyContent: 'center' },
  input: { borderWidth: 1, borderColor: '#ccc', borderRadius: 8, padding: 12, marginBottom: 12 },
  button: { backgroundColor: '#007AFF', borderRadius: 8, padding: 14 },
  buttonText: { color: '#fff', fontWeight: '600', textAlign: 'center' },
});
```

> **⚡ Best practice (2026):** `KeyboardAvoidingView` is finicky. Many teams now use **`react-native-keyboard-controller`** (`KeyboardAwareScrollView`, `KeyboardAvoidingView` replacements) for smooth, consistent behavior across both platforms. If the built-in component fights you, reach for that library.

### 8.2 Managing Focus Between Inputs **[I]**

A polished form lets the keyboard's "Next" button jump to the following field. Use refs and `onSubmitEditing`.

```tsx
import { useRef } from 'react';
import { TextInput } from 'react-native';
function LoginForm() {
  const passwordRef = useRef<TextInput>(null);
  return (
    <>
      <TextInput
        placeholder="Email"
        returnKeyType="next"                              // keyboard shows a "Next" key
        onSubmitEditing={() => passwordRef.current?.focus()} // jump to password
        submitBehavior="submit"                           // keep keyboard up (RN 0.72+; was blurOnSubmit={false})
      />
      <TextInput
        ref={passwordRef}
        placeholder="Password"
        secureTextEntry
        returnKeyType="done"
        onSubmitEditing={() => {/* handleSubmit() */}}
      />
    </>
  );
}
```

### 8.3 Forms with React Hook Form **[I]**

For anything beyond two fields, use **React Hook Form** (see the dedicated `REACT_HOOK_FORM_GUIDE.md`). It minimizes re-renders and centralizes validation. With RN you wrap each `TextInput` in a `<Controller>` (since `TextInput` isn't a native form element RHF can register directly).

```bash
npm install react-hook-form
```

```tsx
import { useForm, Controller } from 'react-hook-form';
import { TextInput, Text, Pressable, View } from 'react-native';
type FormData = { email: string; password: string };
export default function SignUpForm() {
  const { control, handleSubmit, formState: { errors, isSubmitting } } = useForm<FormData>({
    defaultValues: { email: '', password: '' },
  });
  const onSubmit = async (data: FormData) => {
    // call your API here; isSubmitting is true while this awaits
    console.log(data);
  };
  return (
    <View>
      <Controller
        control={control}
        name="email"
        rules={{
          required: 'Email is required',
          pattern: { value: /\S+@\S+\.\S+/, message: 'Invalid email' },
        }}
        render={({ field: { onChange, value, onBlur } }) => (
          <TextInput
            value={value}
            onChangeText={onChange}   // RHF tracks the value
            onBlur={onBlur}           // RHF marks the field "touched"
            placeholder="Email"
            keyboardType="email-address"
            autoCapitalize="none"
          />
        )}
      />
      {errors.email && <Text style={{ color: 'red' }}>{errors.email.message}</Text>}
      <Controller
        control={control}
        name="password"
        rules={{ required: 'Password is required', minLength: { value: 8, message: 'Min 8 chars' } }}
        render={({ field: { onChange, value, onBlur } }) => (
          <TextInput value={value} onChangeText={onChange} onBlur={onBlur} placeholder="Password" secureTextEntry />
        )}
      />
      {errors.password && <Text style={{ color: 'red' }}>{errors.password.message}</Text>}
      <Pressable onPress={handleSubmit(onSubmit)} disabled={isSubmitting}>
        <Text>{isSubmitting ? 'Submitting…' : 'Sign Up'}</Text>
      </Pressable>
    </View>
  );
}
```

> **Best practice:** Pair RHF with a schema validator (Zod) via `@hookform/resolvers` for type-safe validation shared with your backend. React 19 Actions (`REACT_19_GUIDE.md`) are mostly a web/server concern; on native, RHF remains the standard.

### 8.4 Dismissing the Keyboard **[B]**

```tsx
import { Keyboard, TouchableWithoutFeedback, View } from 'react-native';
import { useEffect } from 'react';
// Tap anywhere outside an input to dismiss:
<TouchableWithoutFeedback onPress={Keyboard.dismiss} accessible={false}>
  <View style={{ flex: 1 }}>{/* form */}</View>
</TouchableWithoutFeedback>;
// Imperatively dismiss:
Keyboard.dismiss();
// React to keyboard show/hide (e.g. to animate a footer):
useEffect(() => {
  const show = Keyboard.addListener('keyboardDidShow', (e) =>
    console.log('Keyboard height:', e.endCoordinates.height)
  );
  const hide = Keyboard.addListener('keyboardDidHide', () => {});
  return () => { show.remove(); hide.remove(); }; // ALWAYS clean up listeners
}, []);
```

---

## 9. State & Data Fetching

State management in RN is *identical to React on the web* — same hooks, same Context. What's mobile-specific is **persistence** (AsyncStorage / SecureStore) and the realities of flaky mobile networks (which make TanStack Query especially valuable). For deep React state patterns see `REACT_19_GUIDE.md`; for the standalone store library see `ZUSTAND_GUIDE.md`; for the query library see `TANSTACK_QUERY_GUIDE.md`.

### 9.1 Local State & Context **[B]**

`useState`, `useReducer`, `useContext`, `useMemo`, `useCallback`, `useRef` all behave exactly as on the web.

```tsx
import { useState, useContext, createContext, ReactNode } from 'react';
type Theme = 'light' | 'dark';
const ThemeContext = createContext<{ theme: Theme; toggle: () => void } | null>(null);
export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light');
  const toggle = () => setTheme((t) => (t === 'light' ? 'dark' : 'light'));
  return <ThemeContext.Provider value={{ theme, toggle }}>{children}</ThemeContext.Provider>;
}
// Custom hook that enforces the provider exists — fail loudly, not silently.
export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used inside ThemeProvider');
  return ctx;
}
```

> **When to reach beyond Context:** Context is great for low-frequency global values (theme, current user). For frequently-updated or large global state, Context causes broad re-renders — use **Zustand** (`ZUSTAND_GUIDE.md`) or Redux Toolkit. For *server* state (data from APIs), don't use any of these — use TanStack Query (§9.3).

### 9.2 The Native `fetch` API **[B]**

`fetch` is built in (no install). The classic pattern: track `data`/`loading`/`error`, and **guard against setting state after unmount** (a screen the user navigated away from).

```tsx
import { useState, useEffect } from 'react';
type Post = { id: number; title: string; body: string };
function usePosts() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  useEffect(() => {
    const controller = new AbortController(); // lets us cancel the request on unmount
    (async () => {
      try {
        const res = await fetch('https://jsonplaceholder.typicode.com/posts?_limit=20', {
          signal: controller.signal,
        });
        if (!res.ok) throw new Error(`HTTP ${res.status}`); // fetch does NOT throw on 4xx/5xx!
        setPosts(await res.json());
      } catch (e) {
        if ((e as Error).name !== 'AbortError') setError((e as Error).message);
      } finally {
        setLoading(false);
      }
    })();
    return () => controller.abort(); // cancel if the component unmounts mid-flight
  }, []);
  return { posts, loading, error };
}
```

> **Gotcha:** `fetch` only rejects on *network* failure. An HTTP 404 or 500 is a *successful* fetch with `res.ok === false`. You must check `res.ok` yourself, or you'll happily parse error pages as data.

### 9.3 TanStack Query (React Query) — the Right Way to Do Server State **[I]**

**Why:** mobile networks drop, screens remount, users pull-to-refresh. TanStack Query handles caching, deduplication, background refetching, retries, and stale-while-revalidate so you stop hand-writing the loading/error/cache dance. See `TANSTACK_QUERY_GUIDE.md` for depth.

```bash
npm install @tanstack/react-query
```

```tsx
// app/_layout.tsx — install the provider once at the root.
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // data is "fresh" for 5 min → no refetch during that window
      retry: 2,                 // retry failed queries twice (good for flaky mobile networks)
    },
  },
});
export default function RootLayout() {
  return <QueryClientProvider client={queryClient}>{/* navigator */}</QueryClientProvider>;
}
```

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { FlatList, Text, ActivityIndicator, RefreshControl } from 'react-native';
type Post = { id: number; title: string };
const fetchPosts = async (): Promise<Post[]> => {
  const res = await fetch('https://jsonplaceholder.typicode.com/posts?_limit=10');
  if (!res.ok) throw new Error('Network error');
  return res.json();
};
export default function PostsScreen() {
  const { data: posts, isLoading, isError, error, refetch, isRefetching } = useQuery({
    queryKey: ['posts'], // cache key — uniquely identifies this data
    queryFn: fetchPosts,
  });
  if (isLoading) return <ActivityIndicator />;
  if (isError) return <Text>Error: {(error as Error).message}</Text>;
  return (
    <FlatList
      data={posts}
      keyExtractor={(p) => String(p.id)}
      renderItem={({ item }) => <Text>{item.title}</Text>}
      refreshControl={<RefreshControl refreshing={isRefetching} onRefresh={refetch} />}
    />
  );
}
// Mutations (POST/PUT/DELETE) — invalidate the cache so dependent queries refetch.
function useCreatePost() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (newPost: { title: string }) => {
      const res = await fetch('https://jsonplaceholder.typicode.com/posts', {
        method: 'POST',
        body: JSON.stringify(newPost),
        headers: { 'Content-Type': 'application/json' },
      });
      return res.json();
    },
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['posts'] }), // refetch the list
  });
}
```

### 9.4 `AsyncStorage` — Simple Key-Value Persistence **[B]**

**What it is:** a small, asynchronous, **unencrypted** key-value store (think `localStorage`, but async and native). **For:** non-sensitive data — preferences, last-viewed tab, cached lists, onboarding-seen flags. **Not for:** tokens or anything secret (use SecureStore, §9.5).

```bash
npx expo install @react-native-async-storage/async-storage
```

```tsx
import AsyncStorage from '@react-native-async-storage/async-storage';
await AsyncStorage.setItem('user_prefs', JSON.stringify({ theme: 'dark' })); // values must be strings
const raw = await AsyncStorage.getItem('user_prefs');
const prefs = raw ? JSON.parse(raw) : null;
await AsyncStorage.removeItem('user_prefs');
// Reusable typed hook:
import { useState, useEffect } from 'react';
function useStoredValue<T>(key: string, defaultValue: T) {
  const [value, setValue] = useState<T>(defaultValue);
  useEffect(() => {
    AsyncStorage.getItem(key).then((raw) => { if (raw !== null) setValue(JSON.parse(raw)); });
  }, [key]);
  const store = async (next: T) => { setValue(next); await AsyncStorage.setItem(key, JSON.stringify(next)); };
  return [value, store] as const;
}
```

> **⚡ Performance note:** For high-volume or synchronous reads, **`react-native-mmkv`** is a much faster (synchronous, C++) alternative to AsyncStorage, and integrates well with Zustand persistence. Consider it when AsyncStorage's async API becomes awkward.

### 9.5 `expo-secure-store` — Encrypted Storage for Secrets **[I]**

**What it is:** stores small values in the **iOS Keychain / Android Keystore** (hardware-backed, encrypted). **For:** auth tokens, refresh tokens, sensitive keys.

```bash
npx expo install expo-secure-store
```

```tsx
import * as SecureStore from 'expo-secure-store';
await SecureStore.setItemAsync('auth_token', 'eyJhbGci...');
const token = await SecureStore.getItemAsync('auth_token'); // null if absent
await SecureStore.deleteItemAsync('auth_token');
// Optional: require device unlock / biometrics to read a value:
await SecureStore.setItemAsync('secret', 'value', {
  keychainAccessible: SecureStore.WHEN_UNLOCKED, // only readable when device is unlocked
});
```

> **Gotcha:** ~2048-byte limit per value. Store the token, not a whole JSON blob. For larger secrets, store an encryption key in SecureStore and the (encrypted) data in the file system.

---

## 10. Expo SDK — Device & Native APIs

The Expo SDK is a large library of device capabilities that "just work" without writing native code. **Always install SDK packages with `npx expo install`** (not `npm install`) — it picks the version that matches your SDK, avoiding native-incompatibility crashes.

### 10.1 The Permissions Model — Learn This First **[I]**

Most device APIs (camera, location, notifications, photos, contacts) require **runtime permission**. The universal flow is: **check → request if needed → handle denial gracefully.** Each Expo module ships a hook (`useXPermissions`) and/or imperative functions.

```tsx
import { useCameraPermissions, PermissionStatus } from 'expo-camera';
import { Pressable, Text } from 'react-native';
function CameraGate({ children }: { children: React.ReactNode }) {
  const [permission, requestPermission] = useCameraPermissions();
  if (!permission) return null; // permission object still loading
  if (permission.status !== PermissionStatus.GRANTED) {
    return (
      <Pressable onPress={requestPermission}>
        <Text>Grant Camera Access</Text>
      </Pressable>
    );
  }
  return <>{children}</>;
}
```

> **Key fields on a permission object:** `granted` (boolean), `status` (`'granted' | 'denied' | 'undetermined'`), `canAskAgain` (if `false`, the OS won't show the dialog again — you must deep-link to Settings via `Linking.openSettings()`).
>
> **Best practice:** Always declare a *usage description string* in `app.json` (iOS requires `NSCameraUsageDescription` etc., usually set by the module's config plugin — §13). Without it, iOS crashes on first request. Explain *why* you need the permission, both in that string and ideally in a pre-prompt screen.

### 10.2 `expo-camera` **[I]**

```bash
npx expo install expo-camera
```

```tsx
import { CameraView, CameraType, useCameraPermissions } from 'expo-camera';
import { useRef, useState } from 'react';
import { Pressable, Text, View } from 'react-native';
export default function Camera() {
  const [facing, setFacing] = useState<CameraType>('back');
  const [permission, requestPermission] = useCameraPermissions();
  const cameraRef = useRef<CameraView>(null);
  if (!permission?.granted) {
    return (
      <View style={{ flex: 1, justifyContent: 'center' }}>
        <Text>Camera permission required</Text>
        <Pressable onPress={requestPermission}><Text>Grant</Text></Pressable>
      </View>
    );
  }
  const takePicture = async () => {
    const photo = await cameraRef.current?.takePictureAsync({
      quality: 0.8,  // 0–1 JPEG quality
      base64: false, // include a base64 string (large — avoid unless needed)
      exif: false,
    });
    console.log(photo?.uri); // a file:// URI you can upload or display
  };
  return (
    <CameraView style={{ flex: 1 }} facing={facing} ref={cameraRef}>
      <Pressable onPress={() => setFacing((f) => (f === 'back' ? 'front' : 'back'))}>
        <Text>Flip</Text>
      </Pressable>
      <Pressable onPress={takePicture}><Text>Capture</Text></Pressable>
    </CameraView>
  );
}
```

> **⚡ Version note:** Barcode/QR scanning is built into `CameraView` (`barcodeScannerSettings` + `onBarcodeScanned`) and generally requires a **development build** (not Expo Go) in recent SDKs.

### 10.3 `expo-image-picker` — Pick or Capture Media **[B]**

The easiest way to let users choose a photo/video from their library or take a new one.

```bash
npx expo install expo-image-picker
```

```tsx
import * as ImagePicker from 'expo-image-picker';
async function pickImage() {
  const { status } = await ImagePicker.requestMediaLibraryPermissionsAsync();
  if (status !== 'granted') return;
  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ['images'],  // 'images' | 'videos' | ['images','videos']
    allowsEditing: true,     // show the crop UI after selection
    aspect: [4, 3],          // crop ratio (only honored when allowsEditing)
    quality: 0.8,
  });
  if (!result.canceled) {
    const uri = result.assets[0].uri; // file:// URI
    console.log('Selected:', uri);
  }
}
async function takePhoto() {
  const { status } = await ImagePicker.requestCameraPermissionsAsync();
  if (status !== 'granted') return;
  const result = await ImagePicker.launchCameraAsync({ quality: 0.8 });
  if (!result.canceled) console.log(result.assets[0].uri);
}
```

### 10.4 `expo-location` **[I]**

```bash
npx expo install expo-location
```

```tsx
import * as Location from 'expo-location';
import { useEffect, useState } from 'react';
function useCurrentLocation() {
  const [location, setLocation] = useState<Location.LocationObject | null>(null);
  const [error, setError] = useState<string | null>(null);
  useEffect(() => {
    (async () => {
      const { status } = await Location.requestForegroundPermissionsAsync();
      if (status !== 'granted') { setError('Permission denied'); return; }
      const loc = await Location.getCurrentPositionAsync({ accuracy: Location.Accuracy.High });
      setLocation(loc); // loc.coords = { latitude, longitude, altitude, speed, ... }
    })();
  }, []);
  return { location, error };
}
// Continuously watch position (e.g. live tracking). Remember to remove() the sub.
async function watch() {
  const sub = await Location.watchPositionAsync(
    { accuracy: Location.Accuracy.High, timeInterval: 1000, distanceInterval: 10 },
    (loc) => console.log(loc.coords)
  );
  // sub.remove(); when done
}
// Reverse geocoding: coords → human-readable address.
async function whereAmI() {
  const [addr] = await Location.reverseGeocodeAsync({ latitude: 37.78, longitude: -122.42 });
  console.log(addr.city, addr.country);
}
```

> **Gotcha:** *Background* location (tracking while the app is closed) needs `requestBackgroundPermissionsAsync()`, extra `app.json` config, and triggers heavy store-review scrutiny. Only request it if you truly need it.

### 10.5 `expo-notifications` — Local & Push **[I]**

Two distinct things: **local notifications** (your app schedules them on-device) and **push notifications** (a server sends them via Apple/Google, routed through Expo's push service).

```bash
npx expo install expo-notifications expo-device expo-constants
```

```tsx
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import Constants from 'expo-constants';
import { useEffect, useRef } from 'react';
// Decide how notifications appear while the app is in the FOREGROUND.
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowBanner: true,
    shouldShowList: true,
    shouldPlaySound: true,
    shouldSetBadge: false,
  }),
});
// Register the device and obtain an Expo push token (the address servers send to).
async function registerForPushNotifications(): Promise<string | null> {
  if (!Device.isDevice) return null; // simulators/emulators can't receive push
  const { status: existing } = await Notifications.getPermissionsAsync();
  let finalStatus = existing;
  if (existing !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }
  if (finalStatus !== 'granted') return null;
  const projectId = Constants.expoConfig?.extra?.eas?.projectId;
  const token = await Notifications.getExpoPushTokenAsync({ projectId });
  return token.data; // "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]"
}
// Schedule a LOCAL notification (no server needed):
async function remindMe() {
  await Notifications.scheduleNotificationAsync({
    content: { title: 'Reminder', body: 'You have a task due!', data: { taskId: '123' } },
    trigger: { seconds: 60, repeats: false }, // fire in 60s
  });
}
// React to notifications (received in-app, or tapped):
function useNotifications() {
  const receivedRef = useRef<Notifications.EventSubscription>();
  const responseRef = useRef<Notifications.EventSubscription>();
  useEffect(() => {
    receivedRef.current = Notifications.addNotificationReceivedListener((n) =>
      console.log('Received in foreground:', n)
    );
    responseRef.current = Notifications.addNotificationResponseReceivedListener((r) => {
      const data = r.notification.request.content.data; // navigate based on this
      console.log('User tapped, data:', data);
    });
    return () => { receivedRef.current?.remove(); responseRef.current?.remove(); };
  }, []);
}
```

> **How push actually flows:** your app gets an Expo push token → you send it to *your* server → your server POSTs to Expo's push API with that token → Expo forwards to APNs (Apple) / FCM (Google) → the phone shows it. You don't deal with APNs/FCM certificates directly; EAS + Expo handle that plumbing.

### 10.6 `expo-file-system` — Read/Write Files **[I]**

```bash
npx expo install expo-file-system
```

```tsx
import * as FileSystem from 'expo-file-system';
const docDir = FileSystem.documentDirectory; // persistent, backed up, app-private
const cacheDir = FileSystem.cacheDirectory;   // OS may purge under storage pressure
await FileSystem.writeAsStringAsync(`${docDir}data.json`, JSON.stringify({ key: 'val' }));
const content = await FileSystem.readAsStringAsync(`${docDir}data.json`);
// Resumable download (survives interruptions):
const dl = FileSystem.createDownloadResumable('https://example.com/big.pdf', `${docDir}big.pdf`);
const { uri } = (await dl.downloadAsync()) ?? {};
console.log('Saved to', uri);
const info = await FileSystem.getInfoAsync(`${docDir}data.json`);
console.log(info.exists, info.size);
await FileSystem.deleteAsync(`${docDir}data.json`, { idempotent: true }); // no throw if absent
```

> **Gotcha:** files live in an app-private sandbox; other apps can't see them. `documentDirectory` persists and is backed up; `cacheDirectory` can vanish — never store anything you can't re-create there.

### 10.7 `expo-audio` / `expo-video` (replacing `expo-av`) **[I]**

> **⚡ Version note:** In SDK 53, `expo-av` is being split into the new **`expo-audio`** and **`expo-video`** packages. Use these for new projects.

```bash
npx expo install expo-audio expo-video
```

```tsx
// Audio:
import { useAudioPlayer } from 'expo-audio';
import { Pressable, Text } from 'react-native';
function AudioToggle() {
  const player = useAudioPlayer(require('../assets/sound.mp3'));
  return (
    <Pressable onPress={() => (player.playing ? player.pause() : player.play())}>
      <Text>{player.playing ? 'Pause' : 'Play'}</Text>
    </Pressable>
  );
}
// Video:
import { VideoView, useVideoPlayer } from 'expo-video';
function Player() {
  const player = useVideoPlayer('https://example.com/video.mp4', (p) => { p.loop = true; p.play(); });
  return (
    <VideoView player={player} style={{ width: 320, height: 180 }} allowsFullscreen allowsPictureInPicture />
  );
}
```

### 10.8 `expo-haptics` — Tactile Feedback **[B]**

Cheap, high-impact polish. iOS especially has crisp haptics.

```bash
npx expo install expo-haptics
```

```tsx
import * as Haptics from 'expo-haptics';
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);   // button tap
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy);
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success); // also Warning/Error
await Haptics.selectionAsync(); // light tick for picker/scroll selection changes
```

### 10.9 `expo-sensors` **[I]**

```bash
npx expo install expo-sensors
```

```tsx
import { Accelerometer } from 'expo-sensors';
import { useEffect, useState } from 'react';
function useAccelerometer() {
  const [data, setData] = useState({ x: 0, y: 0, z: 0 });
  useEffect(() => {
    Accelerometer.setUpdateInterval(100); // ms between samples
    const sub = Accelerometer.addListener(setData); // x/y/z roughly in g (-1..1)
    return () => sub.remove();
  }, []);
  return data;
}
```

> Also available: `Gyroscope`, `Magnetometer`, `Barometer`, `DeviceMotion`, `Pedometer`.

### 10.10 Other High-Value SDK Modules **[I]**

| Package | What it does |
|---|---|
| `expo-linking` | Deep links + open URLs/settings (`Linking.openSettings()`) |
| `expo-web-browser` | In-app browser (OAuth flows) |
| `expo-auth-session` | OAuth/OIDC sign-in (Google, Apple, etc.) |
| `expo-clipboard` | Copy/paste |
| `expo-sharing` | Native share sheet |
| `expo-contacts` | Read/write contacts (permission-gated) |
| `expo-local-authentication` | Face ID / Touch ID / biometric unlock |
| `expo-network` | Connectivity status |
| `expo-battery` | Battery level/state |
| `expo-blur` | Native blur views (iOS frosted glass) |
| `expo-linear-gradient` | Gradients (no CSS gradients in RN) |
| `expo-system-ui` | Background color, nav bar color |

> Cross-reference: for auth specifically, see `BETTERAUTH_GUIDE.md` / `SUPABASE_GUIDE.md` for backend patterns; `expo-auth-session` + `expo-secure-store` is the typical native client side.

---

## 11. Navigation Patterns & Params

This section assembles the §4 primitives into the real navigation *architectures* you'll actually ship.

### 11.1 Stack + Tabs Nested (the Most Common App Shape) **[I]**

Almost every consumer app is: a tab bar at the bottom, and tapping an item pushes a full-screen detail (hiding the tabs), with the occasional modal.

```
app/
├── _layout.tsx         ← Root Stack (wraps everything)
├── (tabs)/
│   ├── _layout.tsx     ← Tab navigator
│   ├── index.tsx       ← Tab 1: Home
│   └── profile.tsx     ← Tab 2: Profile
├── post/
│   └── [id].tsx        ← Pushed ON TOP of the tabs (full screen, no tab bar)
└── modal.tsx           ← Presented as a sheet
```

```tsx
// app/_layout.tsx — the root Stack ties it together.
import { Stack } from 'expo-router';
export default function RootLayout() {
  return (
    <Stack>
      {/* Tabs render with no Stack header (tabs manage their own UI). */}
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      {/* A detail page pushed over the tabs — tabs disappear, back button appears. */}
      <Stack.Screen name="post/[id]" options={{ title: 'Post' }} />
      {/* A modal sheet. */}
      <Stack.Screen name="modal" options={{ presentation: 'modal', title: 'Settings' }} />
    </Stack>
  );
}
```

### 11.2 Passing & Reading Params **[I]**

```tsx
// Navigate WITH params — three equivalent styles:
import { Link, router } from 'expo-router';
// 1. <Link> (object form = type-safe with typed routes):
<Link href={{ pathname: '/post/[id]', params: { id: item.id, title: item.title } }}>
  <Text>{item.title}</Text>
</Link>;
// 2. router.push:
router.push({ pathname: '/post/[id]', params: { id: '42', title: 'Hello' } });
// 3. Raw URL string (less type-safe; query string for extra params):
router.push('/post/42?title=Hello');
```

```tsx
// Read params in the destination:
import { useLocalSearchParams, useGlobalSearchParams } from 'expo-router';
export default function PostScreen() {
  // useLocalSearchParams: params for THIS route only. Use this 99% of the time.
  const { id, title } = useLocalSearchParams<{ id: string; title: string }>();
  // useGlobalSearchParams: params from the CURRENT URL, including parents. Re-renders on
  // ANY URL change — can cause extra renders. Use rarely.
  const all = useGlobalSearchParams();
  return <Text>Post {id}: {title}</Text>;
}
```

### 11.3 Passing Complex Data (Don't Stuff It in the URL) **[I]**

URL params are strings. Serializing an object into the URL is fragile and leaks data into navigation state.

```tsx
// ❌ Bad — serializing an object into params:
router.push(`/detail?data=${encodeURIComponent(JSON.stringify(bigObject))}`);
// ✅ Good — pass an ID; fetch or look up the full object at the destination:
router.push({ pathname: '/detail/[id]', params: { id: item.id } });
// Then in DetailScreen: useQuery(['item', id]) OR read from a Zustand store.
```

> **Best practice:** Treat params as *addresses*, not *payloads*. Put identifiers in the URL; resolve the data via TanStack Query (cache-friendly) or a shared store (Zustand).

### 11.4 Drawer Navigator **[I]**

A slide-out side menu. Requires gesture-handler + reanimated (already in most projects).

```bash
npx expo install @react-navigation/drawer react-native-gesture-handler react-native-reanimated
```

```tsx
// app/(drawer)/_layout.tsx
import { Drawer } from 'expo-router/drawer';
import { GestureHandlerRootView } from 'react-native-gesture-handler';
export default function DrawerLayout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <Drawer>
        <Drawer.Screen name="index" options={{ drawerLabel: 'Home', title: 'Home' }} />
        <Drawer.Screen name="profile" options={{ drawerLabel: 'Profile', title: 'Profile' }} />
      </Drawer>
    </GestureHandlerRootView>
  );
}
```

### 11.5 Auth Guard Pattern **[I]**

Gate the app behind login. The clean Expo Router approach: check auth in a layout and `<Redirect>` based on state, ideally using protected route groups.

```tsx
// app/_layout.tsx — redirect unauthenticated users to /login.
import { Slot, Redirect } from 'expo-router';
import { useAuth } from '../hooks/useAuth';
export default function RootLayout() {
  const { isLoggedIn, isLoading } = useAuth();
  if (isLoading) return null; // keep splash until we know the auth state
  // Send logged-out users to the auth group; logged-in users see the app.
  if (!isLoggedIn) return <Redirect href="/login" />;
  return <Slot />; // <Slot/> renders whichever child route matched
}
```

> **⚡ Version note:** Recent Expo Router supports **protected routes** via `<Stack.Protected guard={isLoggedIn}>`, declaratively hiding routes when a guard is false — cleaner than scattered redirects. Pair with `(auth)`/`(app)` groups (§4.6) so each side has its own layout.

> **Best practice:** Persist the auth token in `expo-secure-store` (§9.5), load it on app start, and keep `isLoading` true until that check completes — otherwise you flash the login screen for a frame on every cold start.

---

## 12. Animations & Gestures

Smooth animation separates a real app from a "web app in a wrapper." The key concept: run animations on the **UI thread**, not the JS thread, so they hold 60/120fps even while JS is busy. **Reanimated** makes this possible; **gesture-handler** provides native-quality touch. Cross-reference: `MOTION_ANIMATION_GUIDE.md` covers web animation — the concepts (springs, timing, easing) transfer, the API here is RN-specific.

### 12.1 `react-native-reanimated` — UI-Thread Animations **[A]**

**What it is:** the standard animation library. **The logic:** it introduces *shared values* (state that lives on the UI thread) and *worklets* (small functions that run on the UI thread). Because they don't round-trip to JS, animations don't stutter when your JS thread is doing work (fetching, rendering lists).

```bash
npx expo install react-native-reanimated
```

> **Setup:** add the Reanimated Babel plugin (it must be **last** in the list). With `babel-preset-expo` in recent SDKs this is wired automatically, but verify `babel.config.js` includes it if animations silently fail.

```tsx
import Animated, {
  useSharedValue, useAnimatedStyle, withTiming, withSpring, withRepeat, withSequence,
  Easing, FadeIn, FadeOut,
} from 'react-native-reanimated';
import { Pressable } from 'react-native';
import { useEffect } from 'react';
// 1) Fade toggle with timing.
function FadeBox() {
  const opacity = useSharedValue(1); // lives on the UI thread
  // useAnimatedStyle is a WORKLET — runs on the UI thread and reads .value.
  const animatedStyle = useAnimatedStyle(() => ({ opacity: opacity.value }));
  return (
    <Pressable onPress={() => { opacity.value = withTiming(opacity.value === 1 ? 0 : 1, { duration: 300 }); }}>
      <Animated.View style={[{ width: 100, height: 100, backgroundColor: 'blue' }, animatedStyle]} />
    </Pressable>
  );
}
// 2) Spring (physics-based bounce) — feels natural for press feedback.
function SpringBox() {
  const scale = useSharedValue(1);
  const style = useAnimatedStyle(() => ({ transform: [{ scale: scale.value }] }));
  return (
    <Pressable
      onPressIn={() => { scale.value = withSpring(1.2, { damping: 4, stiffness: 120 }); }}
      onPressOut={() => { scale.value = withSpring(1); }}
    >
      <Animated.View style={[{ width: 100, height: 100, backgroundColor: 'green' }, style]} />
    </Pressable>
  );
}
// 3) Declarative enter/exit animations — zero shared-value boilerplate.
function AnimatedItem({ visible }: { visible: boolean }) {
  return visible ? (
    <Animated.View entering={FadeIn.duration(300)} exiting={FadeOut.duration(200)} />
  ) : null;
}
// 4) Infinite rotation (loading spinner) with withRepeat.
function Spinner() {
  const rotation = useSharedValue(0);
  useEffect(() => {
    rotation.value = withRepeat(
      withTiming(360, { duration: 1000, easing: Easing.linear }),
      -1,    // -1 = repeat forever
      false  // don't reverse on each iteration
    );
  }, []);
  const style = useAnimatedStyle(() => ({ transform: [{ rotate: `${rotation.value}deg` }] }));
  return <Animated.View style={[{ width: 40, height: 40, borderWidth: 3, borderColor: '#007AFF', borderTopColor: 'transparent', borderRadius: 20 }, style]} />;
}
```

> **Key helpers:** `withTiming` (duration + easing), `withSpring` (physics: `damping`, `stiffness`, `mass`), `withDecay` (momentum), `withDelay`, `withSequence` (chain), `withRepeat` (loop). To call JS from a worklet (e.g. update React state at animation end), use `runOnJS(fn)(args)`.

### 12.2 `react-native-gesture-handler` **[A]**

**What it is:** native-thread gesture recognition (pan, pinch, tap, long-press, fling) that composes with Reanimated. **Why not RN's built-in touch system:** built-in touches go through the JS thread; gesture-handler runs on the UI thread, so drags track your finger perfectly even under JS load. It's also required by Drawer and bottom-sheet libraries.

```bash
npx expo install react-native-gesture-handler
```

```tsx
// You MUST wrap the app root once (usually the root _layout.tsx):
import { GestureHandlerRootView } from 'react-native-gesture-handler';
export default function RootLayout() {
  return <GestureHandlerRootView style={{ flex: 1 }}>{/* app */}</GestureHandlerRootView>;
}
```

```tsx
// A draggable box: Pan gesture driving Reanimated shared values.
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, { useSharedValue, useAnimatedStyle, withSpring } from 'react-native-reanimated';
function DraggableBox() {
  const x = useSharedValue(0);
  const y = useSharedValue(0);
  const startX = useSharedValue(0);
  const startY = useSharedValue(0);
  const pan = Gesture.Pan()
    .onBegin(() => { startX.value = x.value; startY.value = y.value; }) // remember origin
    .onUpdate((e) => { x.value = startX.value + e.translationX; y.value = startY.value + e.translationY; })
    .onEnd(() => { x.value = withSpring(0); y.value = withSpring(0); }); // spring back
  const style = useAnimatedStyle(() => ({ transform: [{ translateX: x.value }, { translateY: y.value }] }));
  return (
    <GestureDetector gesture={pan}>
      <Animated.View style={[{ width: 100, height: 100, backgroundColor: 'coral', borderRadius: 8 }, style]} />
    </GestureDetector>
  );
}
```

> **Composing gestures:** `Gesture.Simultaneous(pinch, pan)` (zoom + drag together), `Gesture.Race(tap, longPress)` (whichever wins), `Gesture.Exclusive(...)`. This composition model is the modern (v2) API — older `PanGestureHandler` JSX components are legacy.

### 12.3 Built-In `Animated` API (Simpler, Older) **[I]**

For trivial cases without adding Reanimated, RN's built-in `Animated` works. With `useNativeDriver: true` it runs transforms/opacity on the native side; without it, animations run on the JS thread.

```tsx
import { Animated, Pressable } from 'react-native';
import { useRef } from 'react';
function PulseButton() {
  const anim = useRef(new Animated.Value(1)).current;
  const pulse = () => {
    Animated.sequence([
      Animated.timing(anim, { toValue: 0.9, duration: 100, useNativeDriver: true }),
      Animated.timing(anim, { toValue: 1, duration: 100, useNativeDriver: true }),
    ]).start();
  };
  return (
    <Pressable onPress={pulse}>
      <Animated.View style={{ transform: [{ scale: anim }], padding: 16, backgroundColor: '#007AFF', borderRadius: 8 }} />
    </Pressable>
  );
}
```

> **Gotcha:** `useNativeDriver: true` only supports non-layout properties (`opacity`, `transform`). You cannot natively animate `width`/`height`/`flex` this way — animate `scale` instead, or use Reanimated. For anything beyond a quick pulse, prefer Reanimated.

---

## 13. Configuration: app.json / app.config.ts

This file is the single source of truth for *what your app is* to the OS and stores: its name, icon, bundle identifiers, permissions strings, and which native capabilities (config plugins) it uses.

### 13.1 `app.json` — Static Config **[I]**

```json
{
  "expo": {
    "name": "My App",
    "slug": "my-app",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/images/icon.png",
    "scheme": "myapp",
    "userInterfaceStyle": "automatic",
    "newArchEnabled": true,
    "splash": {
      "image": "./assets/images/splash-icon.png",
      "resizeMode": "contain",
      "backgroundColor": "#ffffff"
    },
    "ios": {
      "supportsTablet": true,
      "bundleIdentifier": "com.yourcompany.myapp",
      "buildNumber": "1",
      "infoPlist": {
        "NSCameraUsageDescription": "Used for profile photos",
        "NSLocationWhenInUseUsageDescription": "Used to show nearby locations"
      }
    },
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./assets/images/adaptive-icon.png",
        "backgroundColor": "#ffffff"
      },
      "package": "com.yourcompany.myapp",
      "versionCode": 1,
      "permissions": ["CAMERA", "ACCESS_FINE_LOCATION"]
    },
    "web": { "bundler": "metro", "output": "static", "favicon": "./assets/images/favicon.png" },
    "plugins": [
      "expo-router",
      "expo-font",
      ["expo-camera", { "cameraPermission": "Allow $(PRODUCT_NAME) to access your camera." }]
    ],
    "experiments": { "typedRoutes": true },
    "extra": { "eas": { "projectId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" } }
  }
}
```

**Fields that matter most:**
- `bundleIdentifier` (iOS) / `package` (Android) — your app's globally unique ID. **You cannot change these after first store release.** Pick `com.yourcompany.appname` carefully.
- `version` — the human-facing version (`1.2.0`). `buildNumber`/`versionCode` — the integer build counter the stores require to *increase* every upload (EAS can auto-increment, §14).
- `scheme` — your custom URL scheme (`myapp://`) for deep links.
- `infoPlist` permission strings — iOS crashes without the right `NS…UsageDescription` for each permission you request.

### 13.2 `app.config.ts` — Dynamic Config **[I]**

Use a `.ts` config when you need environment-driven values (dev vs prod bundle IDs, API URLs, secrets). It can read `process.env` and compute fields.

```ts
// app.config.ts — overrides/extends app.json at evaluation time.
import { ExpoConfig, ConfigContext } from 'expo/config';
export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: process.env.APP_ENV === 'production' ? 'My App' : 'My App (Dev)',
  slug: 'my-app',
  extra: {
    apiUrl: process.env.API_URL ?? 'https://api.example.com',
    eas: { projectId: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' },
  },
  ios: {
    ...config.ios,
    // Separate bundle IDs so dev + prod can coexist on one device:
    bundleIdentifier: process.env.APP_ENV === 'production' ? 'com.example.myapp' : 'com.example.myapp.dev',
  },
});
```

```tsx
// Read config values at runtime:
import Constants from 'expo-constants';
const apiUrl = Constants.expoConfig?.extra?.apiUrl;
```

### 13.3 Config Plugins — Native Edits Without Xcode **[A]**

**What they are:** functions that modify the generated native projects (`Info.plist`, `AndroidManifest.xml`, Gradle, entitlements) at build/prebuild time. **Why they exist:** so you can add native capabilities declaratively from `app.json` instead of hand-editing native files (which the managed workflow regenerates).

```json
"plugins": [
  "expo-router",
  ["expo-camera", { "cameraPermission": "Custom camera permission text." }],
  ["expo-location", { "locationAlwaysAndWhenInUsePermission": "Allow $(PRODUCT_NAME) to use your location." }],
  ["expo-build-properties", {
    "android": { "compileSdkVersion": 35, "targetSdkVersion": 35 },
    "ios": { "deploymentTarget": "15.1" }
  }]
]
```

> **Best practice:** When you add a native library, check whether it ships an Expo config plugin. If it does, add it here and run `npx expo prebuild --clean` (dev build) — no manual native editing. `expo-build-properties` is the escape hatch for tweaking SDK versions, deployment targets, and Gradle/Pod settings.

### 13.4 Environment Variables **[I]**

```bash
# .env  (committed — PUBLIC values only, since they're bundled into the JS)
EXPO_PUBLIC_API_URL=https://api.example.com
# .env.local  (NOT committed — local overrides)
EXPO_PUBLIC_API_URL=http://localhost:3000
```

```tsx
// Only variables prefixed EXPO_PUBLIC_ are inlined into the bundle:
const apiUrl = process.env.EXPO_PUBLIC_API_URL;
```

> **Gotcha (security):** anything in `EXPO_PUBLIC_*` is **shipped in the JS bundle and readable by anyone** who unzips your app. Never put real secrets there. Server-side secrets stay on your server; native-build secrets go in EAS secrets (`eas secret:create`), referenced in `eas.json`.

### 13.5 App Icons & Splash Screen **[B]**

- **Icon:** a single `1024×1024` PNG at `assets/images/icon.png`. Expo generates every required size. iOS icons must have **no alpha/transparency**.
- **Android adaptive icon:** a `1024×1024` foreground PNG (logo on transparent background) + a background color; Android masks it into circle/squircle/etc.
- **Splash screen:** via `expo-splash-screen` — keep it up until your app is ready.

```tsx
// app/_layout.tsx
import * as SplashScreen from 'expo-splash-screen';
SplashScreen.preventAutoHideAsync(); // hold the splash
export default function RootLayout() {
  const [ready, setReady] = useState(false);
  useEffect(() => {
    (async () => {
      await loadFonts();
      await preloadData();
      setReady(true);
      await SplashScreen.hideAsync(); // reveal the app
    })();
  }, []);
  if (!ready) return null;
  return <Stack />;
}
```

---

## 14. Building & Shipping with EAS

**EAS (Expo Application Services)** is the cloud platform that turns your project into installable binaries, submits them to the stores, and pushes JS updates over the air. The headline benefit: **you can build and ship an iOS app from Windows** (no Mac required), because EAS runs the Mac build in the cloud.

```bash
npm install -g eas-cli
eas login
eas init   # links the project to Expo's servers, writes a projectId into your config
```

### 14.1 `eas.json` — Build/Submit/Update Profiles **[I]**

```json
{
  "cli": { "version": ">= 12.0.0" },
  "build": {
    "development": {
      "developmentClient": true,        // builds a DEV CLIENT (your custom Expo Go)
      "distribution": "internal",       // installable via link, not the store
      "env": { "APP_ENV": "development" }
    },
    "preview": {
      "distribution": "internal",
      "env": { "APP_ENV": "preview" },
      "android": { "buildType": "apk" } // APK = easy sideload for testers
    },
    "production": {
      "autoIncrement": true,            // bump buildNumber/versionCode automatically
      "env": { "APP_ENV": "production" }
    }
  },
  "submit": {
    "production": {
      "ios": { "appleId": "your@apple.com", "ascAppId": "1234567890", "appleTeamId": "ABCDE12345" },
      "android": { "serviceAccountKeyPath": "./service-account.json", "track": "production" }
    }
  },
  "update": { "channel": "production" }
}
```

### 14.2 EAS Build **[A]**

```bash
# Development client — replaces Expo Go for YOUR project (includes your native modules).
eas build --profile development --platform ios
eas build --profile development --platform android
# Preview build — share with testers via a link/QR (Android APK, iOS ad-hoc/TestFlight).
eas build --profile preview --platform android
# Production builds for both stores (signed AAB / IPA):
eas build --profile production --platform all
# Build locally instead of the cloud (needs Xcode / Android Studio installed):
eas build --profile production --platform android --local
```

**Build profiles, in prose:**

| Profile | Purpose | `distribution` | Output |
|---|---|---|---|
| `development` | Daily dev with custom native modules | `internal` | Dev client (custom Expo Go) |
| `preview` | Hand to testers / QA | `internal` | APK (Android) or ad-hoc IPA (iOS) |
| `production` | App Store / Play Store | `store` | Signed AAB / IPA |

> **The dev-client loop:** build the `development` profile once, install the resulting app on your device, then `npx expo start --dev-client` and scan the QR. Now you have Fast Refresh *with* all your custom native code — the best of both worlds. Rebuild only when you add/upgrade a native dependency.

> **Credentials:** EAS manages signing certificates and provisioning profiles for you (`eas credentials`). You generally never touch Xcode's signing UI.

### 14.3 EAS Submit **[A]**

```bash
eas submit --platform ios --profile production       # → App Store Connect / TestFlight
eas submit --platform android --profile production   # → Google Play
```

**Accounts you need:**
- **iOS:** Apple Developer Program ($99/yr). EAS handles provisioning; you provide your Apple ID and an App Store Connect app record.
- **Android:** Google Play Console ($25 one-time) + a **service account JSON** key so EAS can upload on your behalf.

### 14.4 EAS Update — Over-the-Air (OTA) Updates **[A]**

**What it is:** push new **JavaScript and assets** to installed apps **without** a store review. **The logic:** your compiled binary is a native shell + a JS bundle; EAS Update swaps the JS bundle. **The hard limit:** you can change JS/assets only. **Native changes (new native modules, permissions, SDK upgrades) require a new build** and store submission.

```bash
# Push a JS-only fix to everyone on the production channel:
eas update --channel production --message "Fix checkout bug"
# Push to a feature branch (for QA):
eas update --branch my-feature --message "Test this"
```

```json
// app.json — control update behavior.
{
  "expo": {
    "updates": { "enabled": true, "fallbackToCacheTimeout": 0, "checkAutomatically": "ON_LOAD" },
    "runtimeVersion": { "policy": "appVersion" }
  }
}
```

```tsx
// Optionally check/apply updates manually (e.g. on resume):
import * as Updates from 'expo-updates';
async function checkForUpdate() {
  try {
    const update = await Updates.checkForUpdateAsync();
    if (update.isAvailable) {
      await Updates.fetchUpdateAsync();
      await Updates.reloadAsync(); // restart with the new bundle
    }
  } catch (e) {
    console.error('Update check failed:', e);
  }
}
```

> **⚡ Critical concept — `runtimeVersion`:** an OTA update is only delivered to binaries whose **runtime version matches** the update. This prevents shipping JS that calls native code the installed binary doesn't have. When you change native code, bump the runtime version and ship a new build; updates for the *old* runtime won't reach the *new* binary and vice versa. The `appVersion` policy ties runtime to your app version automatically.

### 14.5 Store Submission Checklist **[I]**

**iOS (App Store):**
- [ ] Icon `1024×1024`, no alpha channel
- [ ] Screenshots for required device sizes (6.7"/6.5" iPhone at minimum)
- [ ] Privacy "Nutrition Labels" completed in App Store Connect
- [ ] `expo-tracking-transparency` prompt if you access the IDFA
- [ ] Tested on a real iOS device (not just the simulator)

**Android (Play Store):**
- [ ] High-res icon `512×512`
- [ ] Feature graphic `1024×500`
- [ ] Phone screenshots
- [ ] Target API level meets current Play requirements (rises yearly; API 35 in 2026)
- [ ] Data Safety form completed
- [ ] Tested on a real Android device

> **Best practice:** Ship to **TestFlight** (iOS) / **Internal Testing** (Android) first. Catch crashes with real users on real devices before the public release. Then promote the *same* build to production — don't rebuild.

---

## 15. Debugging

### 15.1 React Native DevTools **[I]**

> **⚡ Version note:** Since RN 0.76+, **React Native DevTools** (Chrome DevTools-based, built into Metro) replaced Flipper as the primary debugger.

```bash
npx expo start
# Press 'j' in the Metro terminal to open React Native DevTools.
# Or open the in-app Dev Menu (below) → "Open DevTools".
```

It gives you:
- **Console** — JS logs, warnings, errors.
- **Sources** — set breakpoints in your TypeScript (source maps make this work on the original code).
- **Components** — inspect the React tree, props, and state (the React DevTools panel).
- **Network** — inspect HTTP requests/responses (⚡ improved in recent RN).
- **Memory / Performance** — profiling.

### 15.2 The In-App Developer Menu **[B]**

| Platform | Open with |
|---|---|
| iOS Simulator | `Cmd + D` (or `Cmd + Ctrl + Z` to simulate a shake) |
| Android Emulator | `Cmd + M` (Mac) / `Ctrl + M` (Windows/Linux) |
| Physical device | Shake the device |

Menu options: Reload, Open DevTools, Toggle Performance Monitor (live FPS overlay), Toggle Element Inspector (tap a view to see its styles/box model).

### 15.3 Console Logging & `__DEV__` **[B]**

```tsx
console.log('Debug:', { user, token });
console.warn('Deprecated API used');
console.error('Broke', error);
// __DEV__ is a global: true in development, false in production builds.
const log = (tag: string, data?: unknown) => {
  if (__DEV__) console.log(`[${tag}]`, data ?? '');
};
```

> **Best practice:** Strip noisy logs from production by guarding with `__DEV__`, or use a logger (e.g. a thin wrapper) you can silence. Excessive `console.log` in hot paths (list rows, animations) hurts performance even in dev.

### 15.4 Error Boundaries **[I]**

A class component that catches render-time errors in its subtree so one broken screen doesn't white-screen the whole app. (Expo Router also provides per-route `ErrorBoundary` exports.)

```tsx
import { Component, ReactNode, ErrorInfo } from 'react';
import { View, Text, Pressable } from 'react-native';
type Props = { children: ReactNode; fallback?: ReactNode };
type State = { hasError: boolean; error?: Error };
class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error }; // switch to fallback UI
  }
  componentDidCatch(error: Error, info: ErrorInfo) {
    // Report to Sentry/Bugsnag here.
    console.error('Caught by boundary:', error, info);
  }
  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
          <Text>Something went wrong.</Text>
          <Pressable onPress={() => this.setState({ hasError: false })}>
            <Text>Try again</Text>
          </Pressable>
        </View>
      );
    }
    return this.props.children;
  }
}
```

> **Best practice (production):** wire a crash-reporting service (Sentry has `sentry-expo`). It captures JS errors *and* native crashes with source-mapped stack traces — invaluable since you can't reproduce every user's device.

### 15.5 Native Logs **[I]**

```bash
npx expo start          # Metro shows JS logs
adb logcat              # raw Android native logs (verbose)
adb logcat | grep -i "ReactNative\|MyApp"   # filter to what matters
# iOS native logs: Xcode → Window → Devices and Simulators → Console, or `idevicesyslog`.
```

### 15.6 Common Errors & Fixes **[B]**

| Error | Likely cause | Fix |
|---|---|---|
| `Text strings must be rendered within a <Text> component` | A raw string sits directly in a `<View>` | Wrap it in `<Text>` |
| `VirtualizedList: ... slow to update` | A `FlatList` nested inside a `ScrollView` | Use `ListHeaderComponent`/`ListFooterComponent` instead of nesting (§6.5) |
| `undefined is not an object (evaluating 'x.y')` | Accessing data before it loads | Add a loading guard / optional chaining |
| Code changes not appearing | Stale Metro cache | `npx expo start --clear` |
| `Unable to resolve module X` | Missing/mismatched install | `npx expo install X`, then restart Metro |
| White screen on launch | JS error before first render | Open DevTools (`Cmd+D`) and read the stack trace |
| App works in Expo Go but crashes in a build | Used a native module not in Expo Go | Make a dev build (§14.2) |

---

## 16. Testing

Testing native apps spans three layers: **unit** (pure functions/hooks), **component** (does a component render and respond correctly), and **end-to-end** (drive the real app on a device/simulator). For React fundamentals of testing see `REACT_19_GUIDE.md`.

### 16.1 Unit & Component Tests — Jest + React Native Testing Library **[I]**

Expo ships a Jest preset (`jest-expo`). **React Native Testing Library (RNTL)** lets you render components and query them the way a user would (by text, role, accessibility label) rather than by implementation details.

```bash
npx expo install jest jest-expo @testing-library/react-native
npm install --save-dev @types/jest
```

```js
// package.json
{
  "scripts": { "test": "jest --watchAll" },
  "jest": { "preset": "jest-expo" }
}
```

```tsx
// components/Counter.tsx
import { useState } from 'react';
import { Pressable, Text, View } from 'react-native';
export function Counter() {
  const [n, setN] = useState(0);
  return (
    <View>
      <Text>Count: {n}</Text>
      <Pressable accessibilityRole="button" onPress={() => setN((c) => c + 1)}>
        <Text>Increment</Text>
      </Pressable>
    </View>
  );
}
```

```tsx
// components/Counter.test.tsx
import { render, screen, fireEvent } from '@testing-library/react-native';
import { Counter } from './Counter';
test('increments the count when the button is pressed', () => {
  render(<Counter />);
  // Query by what the USER sees, not by component internals:
  expect(screen.getByText('Count: 0')).toBeTruthy();
  fireEvent.press(screen.getByText('Increment'));
  expect(screen.getByText('Count: 1')).toBeTruthy();
});
```

> **Best practice:** Query by accessible text/role/label (`getByText`, `getByRole`, `getByLabelText`). This couples tests to behavior, not structure, so refactors don't break them — and it nudges you toward accessible UIs.

### 16.2 Mocking Native Modules **[I]**

Native modules (camera, secure store) don't exist in the Node test environment, so mock them.

```tsx
// Mock expo-secure-store in a test (or a jest setup file):
jest.mock('expo-secure-store', () => ({
  getItemAsync: jest.fn(async () => 'fake-token'),
  setItemAsync: jest.fn(async () => {}),
  deleteItemAsync: jest.fn(async () => {}),
}));
```

> **Tip:** put shared mocks in a `jest.setup.js` referenced via `setupFilesAfterEnv` in the Jest config, so every test gets them.

### 16.3 End-to-End Testing — Maestro **[A]**

**What it is:** Maestro drives the *real* app on a simulator/emulator/device with simple YAML flows. **Why it's popular for Expo:** far less brittle and lower-setup than Detox, and works well with EAS builds.

```yaml
# .maestro/login.yaml — a readable end-to-end flow.
appId: com.yourcompany.myapp
---
- launchApp
- tapOn: "Email"
- inputText: "user@example.com"
- tapOn: "Password"
- inputText: "secret123"
- tapOn: "Login"
- assertVisible: "Welcome back"
```

```bash
# Install Maestro (one-time), then run a flow against a running build:
maestro test .maestro/login.yaml
```

> **Best practice:** keep a tiny smoke-test suite (launch → log in → reach home) that runs against every `preview` build in CI. Catching "the app won't even start" before release is worth more than exhaustive coverage.

---

## 17. Secure Storage & Encrypted Caching (Banking-Grade)

Storing data on a phone is the single most common place a "secure" mobile app silently leaks money. A login screen can be flawless and the TLS pinning immaculate, yet if the app drops a JWT into `AsyncStorage`, or lets TanStack Query persist a cache full of account balances to plain disk, then anyone who later gets the device — or pulls an unencrypted OS backup — owns the data. This section is about the disk: where each kind of value is *allowed* to live, how the OS hardware-backed keystores actually protect it, and how to encrypt the bulk data and caches that don't fit in the keystore. It assumes the SecureStore basics from §9.5, biometrics from §18, and the broader hardening (jailbreak/root detection, tamper response) from §20.

### 17.1 The mobile storage threat model & storage tiers **[A]**

Before choosing an API, decide *what you are defending against*. On mobile the realistic threats are: (1) a **lost or stolen device** that is later powered on and attacked at the lock screen; (2) a **jailbroken/rooted device** where the attacker has root and can read app sandboxes and dump process memory; (3) **OS backups** (iCloud/iTunes, Android Auto Backup) that copy app data off-device to a place you don't control; and (4) **other apps / malware** on the same device probing shared storage. Banking apps must assume all four can happen to *some* of their users.

Those threats map onto a tier list. Pick the *lowest-privilege* tier that still works:

| Tier | Mechanism | Protects against | Use for |
|------|-----------|------------------|---------|
| In-memory only | JS variable / React state, never written | Disk theft, backups (but not a live root memory dump) | Short-lived secrets: the plaintext access token, a decrypted DEK, a one-time PIN |
| SecureStore | iOS Keychain / Android Keystore (hardware-backed) | Lost device, backups (with `_THIS_DEVICE_ONLY`), other apps | Small secrets ≤ ~2 KB: refresh tokens, a key-encryption key, device id |
| Encrypted DB / MMKV | SQLCipher / MMKV `encryptionKey`, key in SecureStore | Lost device, backups; root if biometric-gated key | Bulk sensitive records: offline transactions, statements, contacts |
| Envelope-encrypted file | AES-256-GCM blob on disk, DEK in SecureStore | Lost device, backups | Large blobs/documents that don't fit a DB row (PDF statements, images) |
| Plain AsyncStorage / disk | Unencrypted key-value | **Nothing** sensitive | UI prefs, feature flags, non-PII cache keys only |

The rule that prevents most incidents: **`AsyncStorage` (and any plain disk cache) is NOT for secrets.** It is a convenience store for theme choices and "has the user seen the onboarding" booleans. The moment a value is a token, PII, a balance, or anything an attacker would want, it moves up a tier. Everything else in this section is the detail of the upper tiers.

### 17.2 `expo-secure-store` in depth **[A]**

`expo-secure-store` is the front door to the platform keystores. On **iOS** it stores items in the **Keychain**; on **Android** it stores an entry whose value is encrypted with a key held in the **Android Keystore**. The Keystore key is **hardware-backed** wherever the device supports it — a dedicated Secure Enclave (iOS) or a TEE / **StrongBox** secure element (Android). The practical consequence: the raw key material never enters your app's memory and cannot be exported, even on a rooted device. An attacker with root can ask the OS to *use* the key (so a stolen-but-unlocked device is still at risk — that's what `requireAuthentication` and `_THIS_DEVICE_ONLY` address), but they cannot copy it off the phone.

**`keychainAccessible` — when the value is readable.** This is the most important banking knob. It controls the lock-state required to decrypt, and whether the item is eligible for backup/sync:

- `WHEN_UNLOCKED` (default) — readable only while the device is unlocked; eligible for backup. Foreground-only secrets.
- `AFTER_FIRST_UNLOCK` — readable after the first unlock since boot, including while subsequently locked; needed for background work (silent token refresh, push handling).
- `WHEN_UNLOCKED_THIS_DEVICE_ONLY` / `AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY` — same as above **but never leaves the device**: excluded from iCloud Keychain sync and from encrypted backups. **For banking, always use a `_THIS_DEVICE_ONLY` variant** so a refresh token can never resurface on a restored or synced second device.

**`requireAuthentication`** gates the value behind a fresh biometric / passcode check at *read* time (see §18). Reading throws if the user fails or cancels. Use it for the highest-value items (the key-encryption key, an offline-payment key) so that even an unlocked, stolen phone yields nothing without the user's face or PIN. Note the platform caveat: a value written with `requireAuthentication` may become unreadable if the user changes their biometric enrollment — treat that as a logout/re-enroll signal (§17.6), not a crash.

**Size limit:** SecureStore targets small secrets (~2 KB per item on Android via `EncryptedSharedPreferences`/Keystore; Keychain is more generous but treat 2 KB as the design ceiling). Bulk data does **not** go here — it goes through envelope encryption (§17.3) or an encrypted DB (§17.4). Keep **keys namespaced** (`auth.refreshToken`, not `token`) so logout can wipe a known set, and **delete on logout**.

```ts
import * as SecureStore from 'expo-secure-store';

// Hardware-backed, device-only, biometric-gated refresh token.
export async function saveRefreshToken(token: string): Promise<void> {
  await SecureStore.setItemAsync('auth.refreshToken', token, {
    keychainAccessible: SecureStore.AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY,
    requireAuthentication: true,          // face/touch/passcode at read time (§18)
    authenticationPrompt: 'Unlock your account',
  });
}

export async function getRefreshToken(): Promise<string | null> {
  try {
    return await SecureStore.getItemAsync('auth.refreshToken', {
      requireAuthentication: true,
      authenticationPrompt: 'Unlock your account',
    });
  } catch {
    // User cancelled biometrics, or enrollment changed -> treat as no token.
    return null;
  }
}

// Logout / tamper: wipe every known secret key (namespaced -> enumerable).
const SECRET_KEYS = ['auth.refreshToken', 'crypto.dek', 'db.key'] as const;
export async function wipeSecureStore(): Promise<void> {
  await Promise.all(SECRET_KEYS.map((k) => SecureStore.deleteItemAsync(k)));
}
```

> **Why not store the *access* token in SecureStore?** It's short-lived and re-fetchable from the refresh token. Keep it **in memory only** (§17.1). The fewer long-lived secrets on disk, the smaller the blast radius — see the "what NOT to store" rule in §17.6.

### 17.3 Envelope encryption for bulk data **[A]**

SecureStore can't hold a 4 MB statement PDF or a 50 KB JSON blob of cached transactions. The standard pattern for "encrypt a lot of data with a key the OS protects" is **envelope encryption**: generate a random **data encryption key (DEK)**, use the DEK to encrypt the payload with **authenticated** symmetric crypto (AES-256-GCM), write the ciphertext to the filesystem, and store *only the small DEK* in SecureStore. The hardware-backed keystore guards the tiny key; the key guards the big blob.

Non-negotiable crypto rules, because this is where homegrown code goes wrong:

- **Never roll your own crypto.** Use a vetted, audited primitive. Do not invent a cipher, a mode, or a padding scheme.
- **Authenticated encryption only** — AES-**GCM** (or XChaCha20-Poly1305). The auth tag detects tampering; plain AES-CBC does not and is exploitable.
- **A fresh random nonce/IV per message.** Never reuse a nonce under the same key — GCM nonce reuse is catastrophic. Generate it from a CSPRNG (`expo-crypto`'s `getRandomBytesAsync`, which is OS-backed), not `Math.random`.
- **Store nonce + tag alongside ciphertext** (they're not secret); store the **DEK** in SecureStore.

```ts
import * as Crypto from 'expo-crypto';
import * as FileSystem from 'expo-file-system';
import * as SecureStore from 'expo-secure-store';
import { gcm } from '@noble/ciphers/aes';                  // vetted, audited
import { bytesToHex, hexToBytes } from '@noble/ciphers/utils';

const DEK_KEY = 'crypto.dek';

// 256-bit DEK from an OS CSPRNG; persisted device-only in the keystore.
async function getOrCreateDek(): Promise<Uint8Array> {
  const existing = await SecureStore.getItemAsync(DEK_KEY);
  if (existing) return hexToBytes(existing);
  const dek = Crypto.getRandomBytes(32);                  // 32 bytes = AES-256
  await SecureStore.setItemAsync(DEK_KEY, bytesToHex(dek), {
    keychainAccessible: SecureStore.AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY,
  });
  return dek;
}

export async function encryptToFile(name: string, plaintext: string): Promise<void> {
  const dek = await getOrCreateDek();
  const nonce = Crypto.getRandomBytes(12);                // fresh per message
  const ct = gcm(dek, nonce).encrypt(new TextEncoder().encode(plaintext));
  // Envelope on disk: nonce ‖ ciphertext+tag, hex-encoded.
  const blob = bytesToHex(nonce) + ':' + bytesToHex(ct);
  await FileSystem.writeAsStringAsync(FileSystem.documentDirectory + name, blob);
}

export async function decryptFromFile(name: string): Promise<string> {
  const dek = await getOrCreateDek();
  const blob = await FileSystem.readAsStringAsync(FileSystem.documentDirectory + name);
  const [nHex, cHex] = blob.split(':');
  const pt = gcm(dek, hexToBytes(nHex)).decrypt(hexToBytes(cHex));   // throws on tamper
  return new TextDecoder().decode(pt);
}
```

The same envelope idea — wrap a bulk key with a key the keystore protects — is exactly the encryption-at-rest pattern documented for servers in `GO_JWT_ARGON2_GUIDE.md`; the difference here is that SecureStore plays the role the KMS/master key plays server-side. Decryption *throws* on a bad auth tag: treat that exception as a possible tamper signal (§20), not a parse error to swallow.

### 17.4 Encrypted local database & MMKV **[A]**

For an **offline-first** banking app that holds many sensitive rows — pending transfers, a statement ledger, beneficiary lists — you want queryable storage, not a pile of encrypted files. Use an **encrypted SQLite**. The strongest option in 2026 is **`@op-engineering/op-sqlite` built with SQLCipher**, which transparently encrypts the whole database file with AES-256; you supply the key. Keep that DB key in SecureStore (optionally biometric-gated, §18) and pass it when opening the connection. The plaintext key lives in memory only for the session.

```ts
import { open } from '@op-engineering/op-sqlite';
import * as SecureStore from 'expo-secure-store';
import * as Crypto from 'expo-crypto';
import { bytesToHex } from '@noble/ciphers/utils';

async function getDbKey(): Promise<string> {
  let key = await SecureStore.getItemAsync('db.key', { requireAuthentication: true });
  if (!key) {
    key = bytesToHex(Crypto.getRandomBytes(32));
    await SecureStore.setItemAsync('db.key', key, {
      keychainAccessible: SecureStore.AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY,
      requireAuthentication: true,
    });
  }
  return key;
}

export async function openSecureDb() {
  const db = open({ name: 'ledger.db', encryptionKey: await getDbKey() });   // SQLCipher
  await db.execute(
    'CREATE TABLE IF NOT EXISTS txns (id TEXT PRIMARY KEY, amount_cents INTEGER, memo TEXT)'
  );
  return db;
}
```

Alternatively, `expo-sqlite` can be paired with an application-level encryption layer (encrypt sensitive columns with the §17.3 DEK before insert), but a SQLCipher-backed file is simpler and encrypts indexes and the WAL too — prefer it when you can.

For fast encrypted **key-value** (settings that *are* sensitive, small per-user state, a secure read-through cache index), **`react-native-mmkv`** is excellent: it's an order of magnitude faster than AsyncStorage and supports an `encryptionKey`. Store that key in SecureStore, same pattern:

```ts
import { MMKV } from 'react-native-mmkv';
import * as SecureStore from 'expo-secure-store';

const key = await SecureStore.getItemAsync('mmkv.key');   // created like db.key above
export const secureKV = new MMKV({ id: 'secure', encryptionKey: key ?? undefined });
secureKV.set('lastSyncedBalanceCents', 1042311);          // encrypted at rest
```

MMKV is a great backing store for the *encrypted cache persister* in the next subsection, because writes are synchronous and fast — but remember MMKV's own encryption is only as strong as where the key lives, which is why the key always comes from the keystore.

### 17.5 Secure caching of data (encrypted persistence + eviction) **[A]**

This is the requirement most teams miss. **TanStack Query** (§9.3), Zustand stores, image caches, and ad-hoc API caches are wonderful for UX — and by default they serialize to **plain disk** when you add a persister. If those caches contain balances, transactions, names, or (worst of all) tokens, you've just undone SecureStore. Two principles fix it: **encrypt anything sensitive before it touches disk**, and **evict aggressively** on the events that matter (logout, app backgrounding, integrity failure).

**(a) An encrypted persister for TanStack Query.** Wrap a fast store (MMKV) and run every write through the §17.3 DEK so the dehydrated cache is ciphertext on disk:

```ts
import { QueryClient } from '@tanstack/react-query';
import { persistQueryClient } from '@tanstack/react-query-persist-client';
import { gcm } from '@noble/ciphers/aes';
import { bytesToHex, hexToBytes } from '@noble/ciphers/utils';
import * as Crypto from 'expo-crypto';
import { secureKV } from './mmkv';        // §17.4

async function encryptedPersister(dek: Uint8Array) {
  const STORE = 'rq.cache';
  return {
    persistClient: (client: unknown) => {
      const nonce = Crypto.getRandomBytes(12);
      const ct = gcm(dek, nonce).encrypt(
        new TextEncoder().encode(JSON.stringify(client))
      );
      secureKV.set(STORE, bytesToHex(nonce) + ':' + bytesToHex(ct));
    },
    restoreClient: () => {
      const blob = secureKV.getString(STORE);
      if (!blob) return undefined;
      try {
        const [n, c] = blob.split(':');
        const pt = gcm(dek, hexToBytes(n)).decrypt(hexToBytes(c));
        return JSON.parse(new TextDecoder().decode(pt));
      } catch {
        secureKV.delete(STORE);          // tampered/corrupt -> drop, don't trust
        return undefined;
      }
    },
    removeClient: () => secureKV.delete(STORE),
  };
}

export async function setupPersistence(client: QueryClient, dek: Uint8Array) {
  persistQueryClient({
    queryClient: client,
    persister: await encryptedPersister(dek),
    maxAge: 1000 * 60 * 30,              // (b) hard TTL: 30 min max-age on the cache
  });
}
```

**(b) TTL / max-age and eviction.** Give the persisted cache a hard `maxAge` so stale PII can't linger for days. Pair it with `staleTime`/`gcTime` tuning on the queries themselves: short `gcTime` for sensitive queries so they're garbage-collected from memory quickly. Crucially, **evict on app-background** — when the user swipes the app away, a balance shouldn't sit decrypted in memory or rendered behind the app switcher:

```ts
import { AppState } from 'react-native';
AppState.addEventListener('change', (s) => {
  if (s === 'background' || s === 'inactive') {
    queryClient.removeQueries();          // drop sensitive in-memory cache
    // and blur/blank the screen for the app switcher (see §20)
  }
});
```

**(c) Never cache auth tokens in the query cache.** Tokens are not query data. The access token lives in memory (§17.1); the refresh token lives in SecureStore (§17.2). Filter them out — don't `useQuery` your way into persisting a credential, and don't let an interceptor stash a token in a Zustand store that's persisted to disk.

**(d) `gcTime` tuning.** Treat `gcTime` (formerly `cacheTime`) as a data-minimization control: the lower it is, the sooner sensitive results leave memory. For account/transaction queries, a few minutes is plenty; never set `gcTime: Infinity` on PII.

**(e) Wipe everything on logout or integrity failure.** Logout is not "clear the token" — it's **scorched earth**: clear the query cache, delete the persisted encrypted cache, close and (where required by policy) delete the encrypted DB, and wipe SecureStore. If §20 detects a jailbreak/root or a tamper signal, run the same teardown before exiting.

```ts
export async function secureLogout() {
  queryClient.clear();                    // in-memory
  secureKV.delete('rq.cache');            // persisted encrypted cache
  secureKV.clearAll();                    // any other encrypted KV
  await wipeSecureStore();                // §17.2: tokens, DEK, db.key
  // optionally: delete ledger.db / encrypted files for a clean device
}
```

For server-side caches that mirror this data (e.g. a Redis layer fronting the API), the same TTL-and-eviction discipline applies on the backend — see `REDIS_GUIDE.md` for short-TTL, no-PII cache key design so the device cache and the server cache agree on what's safe to keep and for how long.

### 17.6 Lifecycle hygiene & the "what NOT to store" rule **[A]**

Encryption is necessary but not sufficient; *lifecycle* is what keeps the encrypted footprint from accumulating risk. Wire teardown to the events that change the trust assumptions:

- **On logout** — run `secureLogout()` (§17.5): in-memory, persisted cache, DB, and SecureStore all cleared.
- **On biometric-enrollment change** — if a `requireAuthentication` read starts failing because the user added/removed a face or fingerprint, treat it as "the device's trusted user may have changed": force re-auth and re-provision keys rather than silently retrying.
- **On detected tamper / jailbreak / root** (§20) — wipe before continuing; assume the sandbox is readable.

Defensive habits that cost nothing:

- **Never log secrets.** No `console.log(token)`, no full request/response bodies in production, no secrets in crash-report breadcrumbs (scrub them). Logs are exfiltrated constantly.
- **Disable autofill & keyboard caching on secret fields.** For PINs/passwords use `secureTextEntry`, `autoComplete="off"`, `textContentType="oneTimeCode"` (to dodge strong-password autofill where unwanted), and `keyboardType` choices that avoid predictive/learned text. The keyboard can otherwise cache typed secrets.
- **Blank the app-switcher snapshot** (§20) so balances don't leak into the OS task preview.
- **The "what NOT to store" rule:** *don't persist a long-lived secret you can re-fetch.* Access tokens are re-derived from the refresh token; derived display data can be re-queried. Every avoided persisted secret is one fewer thing an attacker can steal. Persist the minimum, encrypt what you must persist, and put the keys in hardware.

> **Banking-grade storage checklist:**
> - [ ] `AsyncStorage`/plain disk holds **no** secrets, tokens, or PII — UI prefs only.
> - [ ] Secrets in SecureStore use a **`_THIS_DEVICE_ONLY`** accessibility class (no iCloud/backup sync).
> - [ ] Highest-value secrets (KEK/DEK, db key) are **`requireAuthentication`** biometric-gated (§18).
> - [ ] Access token is **in-memory only**; refresh token in SecureStore; **no token in any disk cache**.
> - [ ] Bulk/sensitive data uses **envelope encryption** (AES-256-GCM, fresh nonce, auth tag) or **SQLCipher / MMKV `encryptionKey`**, with the key from the keystore.
> - [ ] **Only vetted crypto** (`@noble/ciphers`, SQLCipher); no homegrown ciphers; CSPRNG (`expo-crypto`) for keys/nonces.
> - [ ] TanStack Query / Zustand / image caches are **encrypted before disk** and have a hard **`maxAge` / `gcTime`**.
> - [ ] Caches **evict on app-background** and on logout; decryption failures **drop** the cache, never trust it.
> - [ ] **Logout = scorched earth**: clear memory, persisted cache, encrypted DB, and SecureStore.
> - [ ] Same teardown runs on **tamper/jailbreak/root** (§20) and on **biometric-enrollment change**.
> - [ ] **No secrets in logs**; secret fields disable autofill/keyboard caching; app-switcher snapshot blanked.

---

## 18. Biometric Authentication (Face ID / Touch ID / Fingerprint)

Biometrics feel like the natural "lock" for a banking app: the user glances at the phone or taps the sensor and the vault opens. But there is a deep and dangerous misconception baked into how most tutorials use them, and getting it wrong is the single most common security failure in mobile finance apps. This section is written for **Expo SDK 54 / RN 0.81 / React 19 (June 2026)**, and it assumes you have already read §17 (secure storage with Keychain/Keystore) — biometrics on their own are nearly worthless without it.

The mental model to hold throughout: a biometric prompt is a question the operating system answers with a **boolean** — "did a face/finger that the OS trusts just appear?" That boolean travels into your JavaScript. JavaScript is the least trustworthy place in the entire system: it can be patched on a jailbroken device, intercepted by a Frida hook, or simply skipped by an attacker who recompiles your app. So a biometric check is only as strong as **what it gates**, and the only thing strong enough to gate banking secrets is the OS-enforced key release described in §18.4 onward.

### 18.1 Presentation-only vs key-bound biometrics — the #1 mistake **[A]**

When people first reach for biometrics they write something like "if `authenticateAsync()` returns success, show the account screen." This is **presentation-only** biometrics: the biometric event is observed by your code, and your code *decides* to reveal data it already had in memory or in plaintext storage. Nothing about the data itself depended on the biometric. An attacker who can manipulate the JS runtime, or who pulls the app's storage off a rooted device, never needs to pass the prompt at all — the secret was sitting there the whole time. For a to-do app this is fine. For anything holding money, a session token, or PII, it is a critical vulnerability.

**Key-bound** biometrics are the opposite. Here the sensitive value (a refresh token, an encryption key, a session key) is stored inside the hardware-backed Keychain (iOS) or Keystore (Android) **with an access-control flag that tells the OS: do not release these bytes to any process until a biometric or device passcode succeeds.** The biometric check is no longer advisory — it is a *precondition the OS itself enforces* before the ciphertext can be decrypted. There is no boolean for an attacker to forge, because the boolean lives inside the secure hardware, not in your JS. This is the only acceptable pattern for banking, and the rest of this section builds toward it.

| Property | Presentation-only | Key-bound (Keychain/Keystore) |
| --- | --- | --- |
| What enforces the gate | Your JS `if` statement | The OS / secure hardware |
| Bypass on rooted/jailbroken device | Trivial | Requires breaking the secure element |
| Protects data at rest | No | Yes — value is encrypted, key locked behind biometric |
| Survives the app being recompiled | No | Yes |
| Acceptable for banking | **No** | **Yes** |

> The rule: never treat `authenticateAsync()`'s return value as authorization for anything valuable. Treat it as UX. Real authorization comes from §18.4 (OS-released keys) and §18.8 (server-side re-auth).

### 18.2 Capability probing with `expo-local-authentication` **[A]**

Before you ever show a biometric button you must know what the device can actually do. `expo-local-authentication` exposes the device's biometric state through three probes, and you should call all of them — hardware presence, enrollment, and the *type* of biometric — because the UX copy ("Sign in with Face ID" vs "with your fingerprint") and even whether you offer the feature depend on the answers.

- `hasHardwareAsync()` — does the device have a sensor at all? A budget Android phone or an old iPad may not.
- `isEnrolledAsync()` — has the user actually registered a face/finger? Hardware can exist with nothing enrolled, in which case prompting will fail.
- `supportedAuthenticationTypesAsync()` — returns an array of `AuthenticationType` values (`FACIAL_RECOGNITION`, `FINGERPRINT`, `IRIS`) so you can label the button correctly and pick the right icon.

```ts
// biometrics/capabilities.ts
import * as LocalAuthentication from 'expo-local-authentication';

export type BiometricKind = 'face' | 'fingerprint' | 'iris' | 'none';

export interface BiometricCapability {
  hasHardware: boolean;
  isEnrolled: boolean;
  kinds: BiometricKind[];
  /** Safe to actually prompt the user right now. */
  usable: boolean;
}

export async function getBiometricCapability(): Promise<BiometricCapability> {
  const [hasHardware, isEnrolled, types] = await Promise.all([
    LocalAuthentication.hasHardwareAsync(),
    LocalAuthentication.isEnrolledAsync(),
    LocalAuthentication.supportedAuthenticationTypesAsync(),
  ]);

  const kinds: BiometricKind[] = types.map((t) => {
    switch (t) {
      case LocalAuthentication.AuthenticationType.FACIAL_RECOGNITION:
        return 'face';
      case LocalAuthentication.AuthenticationType.FINGERPRINT:
        return 'fingerprint';
      case LocalAuthentication.AuthenticationType.IRIS:
        return 'iris';
      default:
        return 'none';
    }
  });

  return {
    hasHardware,
    isEnrolled,
    kinds: kinds.length ? kinds : ['none'],
    usable: hasHardware && isEnrolled,
  };
}
```

Note that `supportedAuthenticationTypesAsync()` reports what the *hardware supports*, not necessarily what is enrolled — always combine it with `isEnrolledAsync()` before deciding the feature is `usable`.

### 18.3 A robust `authenticate()` wrapper — handling every result **[A]**

`authenticateAsync(options)` is where the prompt actually appears. The naive call ignores the rich set of failure modes the OS reports, and in a banking app each one needs different handling: a *user cancel* should silently fall back to the PIN screen, a *lockout* must tell the user to wait or use the passcode, and *not enrolled* should route them to settings rather than spinning forever. The options also matter for security and UX:

- `promptMessage` — the explanation shown in the system sheet (required for good UX; iOS shows it under the Face ID icon).
- `cancelLabel` — text for the cancel button.
- `fallbackLabel` — text for the "use passcode" button on iOS (empty string hides it).
- `disableDeviceFallback` — when `true`, the OS will **not** offer the device passcode as a fallback. For pure biometric step-up you sometimes want `true`, but be careful: if biometrics lock out and there is no device fallback, the user is stuck unless *you* provide an app-level PIN (see §18.7).
- `requireConfirmation` (Android) — for face unlock, require an explicit confirm tap after recognition; leave `true` for high-value actions so a glance alone can't authorize a transfer.

```ts
// biometrics/authenticate.ts
import * as LocalAuthentication from 'expo-local-authentication';
import { getBiometricCapability } from './capabilities';

export type AuthOutcome =
  | { ok: true }
  | { ok: false; reason: 'not_available' | 'not_enrolled' | 'lockout'
        | 'user_cancel' | 'system_cancel' | 'fallback' | 'unknown'; message: string };

export async function authenticate(opts: {
  prompt: string;
  cancelLabel?: string;
  allowDeviceFallback?: boolean; // false => biometric only
  requireConfirmation?: boolean;
}): Promise<AuthOutcome> {
  const cap = await getBiometricCapability();
  if (!cap.hasHardware) return { ok: false, reason: 'not_available', message: 'No biometric hardware.' };
  if (!cap.isEnrolled) return { ok: false, reason: 'not_enrolled', message: 'No biometrics enrolled.' };

  const res = await LocalAuthentication.authenticateAsync({
    promptMessage: opts.prompt,
    cancelLabel: opts.cancelLabel ?? 'Cancel',
    fallbackLabel: opts.allowDeviceFallback === false ? '' : 'Use passcode',
    disableDeviceFallback: opts.allowDeviceFallback === false,
    requireConfirmation: opts.requireConfirmation ?? true,
  });

  if (res.success) return { ok: true };

  // res.error is a string code from the OS; map the important ones.
  switch (res.error) {
    case 'user_cancel':
    case 'app_cancel':
      return { ok: false, reason: 'user_cancel', message: 'Cancelled by user.' };
    case 'system_cancel':
      return { ok: false, reason: 'system_cancel', message: 'Interrupted by the system.' };
    case 'user_fallback':
      return { ok: false, reason: 'fallback', message: 'User chose the passcode fallback.' };
    case 'lockout':
    case 'lockout_permanent':
      return { ok: false, reason: 'lockout', message: 'Too many attempts. Locked out.' };
    case 'not_enrolled':
      return { ok: false, reason: 'not_enrolled', message: 'No biometrics enrolled.' };
    case 'not_available':
      return { ok: false, reason: 'not_available', message: 'Biometrics unavailable.' };
    default:
      return { ok: false, reason: 'unknown', message: res.error ?? 'Unknown error.' };
  }
}
```

Treat `lockout` specially: after several failures iOS/Android temporarily (or permanently, until passcode) disables biometrics. Your only recovery there is the device passcode (if you allowed fallback) or your own app PIN — never leave the user with a dead button.

### 18.4 iOS config: `NSFaceIDUsageDescription` (or Face ID crashes) **[A]**

On iOS, the very first time your app triggers Face ID the OS reads the `NSFaceIDUsageDescription` string from `Info.plist`. **If it is missing, the app does not show a friendly error — it crashes.** In Expo's managed workflow you set it through `app.config` (the `expo-local-authentication` config plugin will inject it, but providing the string explicitly is the reliable, reviewable way). The string is shown to the user in the permission context, so write it like a human, referencing your bank's brand.

```json
{
  "expo": {
    "ios": {
      "infoPlist": {
        "NSFaceIDUsageDescription": "Acme Bank uses Face ID to unlock the app and approve transactions securely."
      }
    },
    "plugins": [
      ["expo-local-authentication", {
        "faceIDPermission": "Acme Bank uses Face ID to unlock the app and approve transactions securely."
      }]
    ]
  }
}
```

Android needs no analogous usage string for the biometric prompt itself, but the `USE_BIOMETRIC` permission is added for you by the library. Always run a real EAS build (not just Expo Go, which bundles its own Info.plist) to verify the crash-on-missing-string case is handled — see permissions handling in §19.

### 18.5 The secure pattern: key-bound secrets via `expo-secure-store` **[A]**

This is the heart of banking-grade biometrics and the payoff of §18.1. Instead of storing a token and *then* gating it with a JS boolean, you store it in `expo-secure-store` with `requireAuthentication: true`. That flag wires the Keychain/Keystore item to the device's biometric/passcode gate at the OS level: the bytes are encrypted by a key that the secure hardware will only use after a successful local authentication. Pair it with `keychainAccessible: WHEN_UNLOCKED_THIS_DEVICE_ONLY` (or `AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY`) so the item is never written to an iCloud/Keystore backup and never leaves this physical device — essential so a restored backup on an attacker's phone can't carry the secret.

```ts
// biometrics/vault.ts — the OS-enforced unlock pattern (combine with §17)
import * as SecureStore from 'expo-secure-store';

const DATA_KEY = 'acme.session.refreshToken';

// Write once at login. The OS, not our JS, guards future reads.
export async function storeProtectedSecret(value: string): Promise<void> {
  await SecureStore.setItemAsync(DATA_KEY, value, {
    requireAuthentication: true, // <-- biometric/passcode required to READ
    keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
    authenticationPrompt: 'Unlock Acme Bank',
  });
}

// Reading TRIGGERS the system biometric prompt. If it fails, no bytes come back.
export async function readProtectedSecret(): Promise<string | null> {
  try {
    return await SecureStore.getItemAsync(DATA_KEY, {
      requireAuthentication: true,
      authenticationPrompt: 'Unlock Acme Bank',
    });
  } catch {
    // Auth failed / cancelled / item invalidated by enrollment change (§18.7).
    return null;
  }
}
```

The crucial difference from §18.3: here you do **not** call `authenticateAsync()` yourself and then read a plaintext token. The act of *reading the secret* is what raises the prompt, and a failed prompt means the decryption simply does not happen. There is no window in which the plaintext exists without a successful biometric. That is OS-enforced protection, and it is the line between a real banking vault and a UX gate.

### 18.6 Banking login, step-up, and inactivity auto-lock **[A]**

With the key-bound vault in place, three banking flows fall out naturally. The common thread: **never store the user's password.** Store a long-lived, server-revocable **refresh token** (or session key), protected by the vault, and exchange it for short-lived access tokens — see token/session concepts in `GO_JWT_ARGON2_GUIDE.md`.

**Biometric login** — on first login the user authenticates with username/password to your server; the server returns a refresh token; you stash it via `storeProtectedSecret`. On every later launch, reading it raises Face ID and, on success, you silently exchange it for an access token. The password is never persisted and never re-entered.

**Step-up / transaction signing** — listing balances might only need a valid session, but moving money must require a *fresh* biometric immediately before the action. Do not reuse the login prompt's result; demand a new one whose success releases a signing key or simply proves liveness right then, and have the server treat that as elevated authorization (§18.8).

```tsx
// flows.tsx — login bootstrap + per-transaction step-up
import { readProtectedSecret, storeProtectedSecret } from './vault';
import { authenticate } from './authenticate';
import { exchangeRefreshToken, postTransfer } from '../api';

export async function biometricLogin(): Promise<boolean> {
  const refresh = await readProtectedSecret(); // raises biometric prompt
  if (!refresh) return false;
  const access = await exchangeRefreshToken(refresh); // server re-validates
  return Boolean(access);
}

export async function approveTransfer(payeeId: string, amount: number) {
  const res = await authenticate({
    prompt: `Approve transfer of $${amount}`,
    allowDeviceFallback: false, // force a real, fresh biometric
    requireConfirmation: true,
  });
  if (!res.ok) throw new Error('Step-up authentication failed.');
  // The server still independently verifies the elevated session — §18.8.
  return postTransfer({ payeeId, amount, stepUp: true });
}
```

**Inactivity auto-lock** — banking apps must re-lock after a short idle period and on backgrounding, so a phone left on a table or snatched mid-session can't be used. Track the last-active timestamp, listen to `AppState`, and on resume (or after N minutes) drop any in-memory access token and force biometric re-auth, which re-reads the vault. See the broader hardening and step-up guidance in §20.

```tsx
import { AppState } from 'react-native';
import { useEffect, useRef } from 'react';

export function useInactivityLock(onLock: () => void, timeoutMs = 2 * 60_000) {
  const backgroundedAt = useRef<number | null>(null);
  useEffect(() => {
    const sub = AppState.addEventListener('change', (state) => {
      if (state === 'background') {
        backgroundedAt.current = Date.now();
      } else if (state === 'active' && backgroundedAt.current) {
        if (Date.now() - backgroundedAt.current >= timeoutMs) onLock();
        backgroundedAt.current = null;
      }
    });
    return () => sub.remove();
  }, [onLock, timeoutMs]);
}
```

### 18.7 Enrollment-change invalidation (critical for banking) **[A]**

Here is the attack that ruins a careless implementation: an attacker who briefly holds an unlocked phone enrolls *their own* fingerprint or face. Now the device's biometric set includes them. If your vault's secret survives that change, the attacker re-opens your app and their finger unlocks the victim's banking session. Defeating this requires telling the secure hardware to **invalidate the key whenever the enrolled biometric set changes.**

- **iOS:** the Keychain access control must use *current-set* semantics (conceptually `biometryCurrentSet` / `kSecAccessControlBiometryCurrentSet`). Adding or removing a face/fingerprint changes the "current set," and the key is automatically destroyed — the secret becomes permanently unreadable, which is exactly what you want.
- **Android:** the Keystore key must be generated with `setUserAuthenticationRequired(true)` **and** `setInvalidatedByBiometricEnrollment(true)`. Any new fingerprint enrollment then invalidates the key, throwing `KeyPermanentlyInvalidatedException` on next use.

`expo-secure-store`'s `requireAuthentication` gives you the user-authentication requirement, but as of SDK 54 it does **not** expose fine-grained `biometryCurrentSet` / `setInvalidatedByBiometricEnrollment` control. For genuine banking-grade enrollment-change invalidation you should drop to a native module or use **`react-native-keychain`** with `accessControl: BIOMETRY_CURRENT_SET` (which maps to the platform-specific current-set semantics above) under a development build. Detect the invalidated state the same way regardless: a read/decrypt that previously worked now throws — treat that as "vault invalidated," wipe any local state, and force a full server re-login rather than silently re-prompting.

```ts
// react-native-keychain (dev build) — current-set, this-device-only
import * as Keychain from 'react-native-keychain';

await Keychain.setGenericPassword('session', refreshToken, {
  accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_CURRENT_SET, // invalidate on enroll change
  accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
  securityLevel: Keychain.SECURITY_LEVEL.SECURE_HARDWARE,
});

export async function readSessionOrInvalidate() {
  try {
    const creds = await Keychain.getGenericPassword({
      authenticationPrompt: { title: 'Unlock Acme Bank' },
    });
    return creds ? creds.password : null;
  } catch (e) {
    // Enrollment changed or biometrics removed -> key destroyed.
    await Keychain.resetGenericPassword();
    return null; // force full re-login
  }
}
```

### 18.8 UX, fallbacks, and "never trust the client boolean" **[A]**

Two principles close out the section. First, **always provide a non-biometric path that is itself secure.** A user with no enrolled biometrics, a locked-out sensor, or a permanently invalidated key must still get in — but the fallback can't be a plaintext bypass. The right pattern is an **app PIN/passcode** that the user sets, from which you derive an encryption key (Argon2id/PBKDF2 — see `GO_JWT_ARGON2_GUIDE.md` for KDF parameters and rationale) that unlocks the same vault data. Biometrics become the fast path; the PIN-derived key is the recovery path; neither stores the secret in the clear. Handle "no biometrics enrolled" by routing to PIN setup, degrade gracefully on older hardware, and respect accessibility (screen readers, sufficient timeouts). Never lock a paying customer out with no recovery — but never make recovery a backdoor either.

Second, and most important: **the client-side biometric boolean is not authorization for server-side actions.** The OS pass/fail lives on a device you do not control. For anything that moves money or changes account state, the server must independently verify a real, fresh credential — a valid short-lived access token bound to the session, ideally elevated by a step-up signal the server itself can validate (a freshly minted token, a signed nonce, or a server-driven re-auth). Treat the on-device biometric purely as the gate that *releases* a server-trusted secret (§18.5) and as a liveness signal; the actual trust decision happens server-side every time. A device that lies about its biometric result must still be unable to forge a server-accepted transfer. See §20 for the full hardening picture and `GO_JWT_ARGON2_GUIDE.md` for how the server validates and rotates these tokens.

> **Banking-grade biometric checklist:**
> - **Key-bound, not presentation-only** — secrets live in Keychain/Keystore behind `requireAuthentication: true`; the OS releases bytes, your JS never gates plaintext (§18.1, §18.5).
> - **`NSFaceIDUsageDescription` set** in `app.config`/Info.plist or iOS Face ID crashes (§18.4).
> - **`*_THIS_DEVICE_ONLY` accessibility** so secrets never ride into a backup or another device (§18.5).
> - **Enrollment-change invalidation** — iOS `biometryCurrentSet`, Android `setInvalidatedByBiometricEnrollment(true)`; drop to `react-native-keychain`/native when needed (§18.7).
> - **Never store the password** — store a revocable refresh/session token instead (§18.6).
> - **Step-up + inactivity auto-lock** — fresh biometric before money moves; re-lock on idle/background (§18.6, §20).
> - **Secure fallback** — app PIN deriving a KDF key unlocks the same vault; no plaintext bypass, no lockout with zero recovery (§18.8).
> - **Server-side re-auth always** — the client biometric boolean is UX, not trust; the server validates fresh credentials for every sensitive action (§18.8, `GO_JWT_ARGON2_GUIDE.md`).

---

## 19. Device APIs & Permissions — The Complete Reference

§10 introduced the device-API surface and the **check → request → handle** permission flow (§10.1) and documented the everyday modules — camera, image-picker, location, notifications, file-system, audio/video, haptics, and sensors. This section finishes the job. It covers the remaining high-value native APIs you'll reach for in real apps, and — more importantly — it gives you the **master permissions reference** that ties every capability to the exact iOS usage-description key and Android `<uses-permission>` entry you must declare. Get permissions wrong and you don't get a bug; you get a hard crash on iOS or an App Store rejection. So we start there.

> **⚡ Version note:** Verified against **Expo SDK 54 / React Native 0.81 (June 2026)**. Always install with `npx expo install <pkg>` so the native module version matches your SDK; `npm install` can pull an incompatible build that crashes at startup.

### 19.1 The Permission Lifecycle, Properly **[I]**

§10.1 showed the happy path. Production code has to handle four states, not two. Every Expo permission object exposes `status` (`'granted' | 'denied' | 'undetermined'`), the convenience boolean `granted`, and the decisive field **`canAskAgain`**. The combination tells you what to do:

- **`undetermined`** — the user has never been asked. You may call `requestPermission()`; the OS will show its one prompt.
- **`granted`** — proceed.
- **`denied` + `canAskAgain: true`** — they said no once (or it lapsed). You can ask again, but only with a good reason — show a rationale first.
- **`denied` + `canAskAgain: false`** — the OS will **never show the dialog again**. Requesting does nothing. Your *only* recovery is to deep-link the user to the system Settings page with `Linking.openSettings()` and let them flip the toggle manually.

```tsx
import { useCameraPermissions } from 'expo-camera';
import { Linking, Pressable, Text, View } from 'react-native';

function CameraPermissionGate({ children }: { children: React.ReactNode }) {
  const [permission, requestPermission] = useCameraPermissions();
  if (!permission) return null;               // still loading
  if (permission.granted) return <>{children}</>;

  // Denied. The CTA depends on whether the OS will still show its dialog.
  const canPrompt = permission.canAskAgain;
  return (
    <View style={{ flex: 1, justifyContent: 'center', padding: 24, gap: 12 }}>
      <Text>We use the camera so you can scan a document. We never upload without your tap.</Text>
      <Pressable onPress={canPrompt ? requestPermission : () => Linking.openSettings()}>
        <Text>{canPrompt ? 'Allow camera' : 'Open Settings'}</Text>
      </Pressable>
    </View>
  );
}
```

**iOS vs Android differ in a way you must design around.** On **iOS** each permission gets exactly **one** native prompt for the app's lifetime — deny it and you're permanently in the `canAskAgain: false` world (Settings is the only path back). That makes the *first* prompt precious: never fire it cold. On **Android** the runtime model is more forgiving — the OS may re-show the dialog, and after two dismissals it enters a "don't ask again" state similar to iOS. Android also exposes a "Allow only this time" option for sensitive permissions (location, camera, mic), so a grant you saw last session may be gone today — **always re-check, never cache "granted" forever.**

The cardinal rule that follows from all of this: **request a permission only at the moment you actually need it (just-in-time), and precede the OS prompt with your own rationale screen** explaining *why*. A pre-prompt screen has two payoffs: dramatically higher grant rates (users who tap "continue" on your screen almost always tap "allow" on the OS one), and App Store compliance — Apple's reviewers reject apps that demand permissions on launch with no context.

**Re-check on focus.** Because the user may revoke a permission in Settings while your app is backgrounded, re-read permission state whenever a screen regains focus rather than trusting a value captured at mount:

```tsx
import { useFocusEffect } from 'expo-router';
import { useCallback } from 'react';
import { getForegroundPermissionsAsync } from 'expo-location';

function useRecheckLocation(onState: (granted: boolean) => void) {
  useFocusEffect(
    useCallback(() => {
      getForegroundPermissionsAsync().then(p => onState(p.granted));
    }, [onState]),
  );
}
```

**Foreground vs background.** Most permissions are *foreground* — they apply only while your app is on screen. **Location is the dangerous exception.** `expo-location` distinguishes `requestForegroundPermissionsAsync()` (when-in-use) from `requestBackgroundPermissionsAsync()` (always). Background location requires **extra app-config declarations**, a separate "Always Allow" grant the user must reach through Settings on iOS, and a written justification at App Store review — Apple scrutinizes "always" location heavily and will reject apps that request it without a clear, user-visible feature (turn-by-turn navigation, geofenced reminders). **Request foreground first; only escalate to background if a real feature demands it, and tell review exactly why.**

### 19.2 The Config-Plugin / App-Config Model — Where Permission Strings Live **[I]**

You never hand-edit `Info.plist` or `AndroidManifest.xml` in an Expo project — those files are generated during `prebuild`. Instead, each module ships a **config plugin** that injects the right entries at build time, and you supply the human-readable strings through `app.config.ts` (or `app.json`). On **iOS** the plugin writes `NS*UsageDescription` strings into `Info.plist`; on **Android** it adds `<uses-permission>` lines to the manifest. (See §13 for the broader config-plugin / prebuild story.)

**A missing iOS usage string is not a warning — it is a hard crash** the instant you call `request*PermissionsAsync()`, and the build will be rejected by App Store Connect. Conversely, declaring permissions you don't actually use *also* hurts you: it expands your App Store **Data Safety / privacy nutrition label**, and reviewers flag permissions with no corresponding feature. **Declare the minimum.**

```ts
// app.config.ts — permission strings supplied to each module's config plugin.
import { ExpoConfig } from 'expo/config';

const config: ExpoConfig = {
  name: 'Acme',
  slug: 'acme',
  ios: {
    infoPlist: {
      // These appear verbatim in the OS prompt — write them for the user, not the lawyer.
      NSCameraUsageDescription: 'Scan documents and receipts.',
      NSMicrophoneUsageDescription: 'Record voice notes attached to a ticket.',
      NSPhotoLibraryUsageDescription: 'Attach photos to a report.',
      NSContactsUsageDescription: 'Invite people you already know.',
      NSLocationWhenInUseUsageDescription: 'Tag a report with where it happened.',
      NSFaceIDUsageDescription: 'Unlock the app with Face ID.',
      NSUserTrackingUsageDescription: 'Used to measure ad performance.',
    },
  },
  android: {
    permissions: [
      'android.permission.CAMERA',
      'android.permission.RECORD_AUDIO',
      'android.permission.ACCESS_FINE_LOCATION',
      'android.permission.READ_CONTACTS',
    ],
  },
  plugins: [
    // Many modules accept their strings here too; the plugin merges them into the native files.
    ['expo-camera', { cameraPermission: 'Scan documents and receipts.' }],
    ['expo-contacts', { contactsPermission: 'Invite people you already know.' }],
    [
      'expo-location',
      {
        locationAlwaysAndWhenInUsePermission: 'Tag reports and send geofenced reminders.',
        isAndroidBackgroundLocationEnabled: false, // flip true ONLY if you truly need background
      },
    ],
  ],
};
export default config;
```

After changing any of this you must regenerate native code: `npx expo prebuild --clean` (bare/dev-build) or rely on EAS Build to run prebuild for you. Editing the string at runtime is impossible — it's baked into the binary.

### 19.3 Master Permission Reference Table **[I]**

This is the table to bookmark. Each row maps an Expo capability to the iOS usage-description key you must set and the Android permission(s) the plugin adds. "Limited / partial" entries are the modern OS feature where the user grants access to *some* data, not all.

| Capability (module) | iOS usage-description key(s) | Android permission(s) | Notes |
|---|---|---|---|
| Camera (`expo-camera`) | `NSCameraUsageDescription` | `CAMERA` | §10.2. Also needed for in-camera QR scanning (§19.8). |
| Microphone (`expo-camera`/`expo-audio`) | `NSMicrophoneUsageDescription` | `RECORD_AUDIO` | Required for video recording even if you don't keep audio. |
| Photo library — read (`expo-image-picker`) | `NSPhotoLibraryUsageDescription` | `READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO` (Android 13+) | iOS offers **Limited** access — user picks specific photos (see §19.10). §10.3. |
| Photo library — add only | `NSPhotoLibraryAddUsageDescription` | `WRITE_EXTERNAL_STORAGE` (legacy) | "Add to library" without read; lighter prompt, better for save-only flows. |
| Media library (`expo-media-library`) | `NSPhotoLibraryUsageDescription` | `READ_MEDIA_*` / `ACCESS_MEDIA_LOCATION` | Full album access + write. Heavier than image-picker. |
| Location — when in use (`expo-location`) | `NSLocationWhenInUseUsageDescription` | `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION` | §10. Request this before "always". |
| Location — always/background | `NSLocationAlwaysAndWhenInUseUsageDescription` | + `ACCESS_BACKGROUND_LOCATION` | Extra config + review justification (§19.1). |
| Notifications (`expo-notifications`) | (no plist string; prompt is requested at runtime) | `POST_NOTIFICATIONS` (Android 13+) | §10. iOS prompt is fired by `requestPermissionsAsync()`. |
| Contacts (`expo-contacts`) | `NSContactsUsageDescription` | `READ_CONTACTS`, `WRITE_CONTACTS` | §19.4. |
| Calendar/Reminders (`expo-calendar`) | `NSCalendarsUsageDescription`, `NSRemindersUsageDescription` | `READ_CALENDAR`, `WRITE_CALENDAR` | iOS 17+ adds "write-only" calendar access. §19.5. |
| Motion / sensors (`expo-sensors`) | `NSMotionUsageDescription` | `HIGH_SAMPLING_RATE_SENSORS`, `ACTIVITY_RECOGNITION` (pedometer) | §10 sensors; pedometer/step count is the gated case. |
| Bluetooth (`expo-bluetooth`/BLE libs) | `NSBluetoothAlwaysUsageDescription` | `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT` (Android 12+) | Android 12+ split BT into runtime permissions. |
| Local network (iOS) | `NSLocalNetworkUsageDescription` (+ `NSBonjourServices`) | — | iOS-only prompt; needed for LAN discovery/casting. |
| Face ID / biometrics (`expo-local-authentication`) | `NSFaceIDUsageDescription` | `USE_BIOMETRIC` | §18. Touch ID/fingerprint needs no plist string. |
| App Tracking Transparency (`expo-tracking-transparency`) | `NSUserTrackingUsageDescription` | `com.google.android.gms.permission.AD_ID` | Required before reading IDFA / cross-app tracking. §19.13. |
| Clipboard (`expo-clipboard`) | — | — | No permission, but a **privacy surface** (§19.7). |

### 19.4 `expo-contacts` — Read & Write the Address Book **[A]**

Contacts power "invite a friend" and autocomplete. It's a sensitive permission — reviewers expect a clear feature behind it. Read with a field mask so you don't pull more than you need; you can also create contacts.

```ts
import * as Contacts from 'expo-contacts';

async function loadEmails() {
  const { status } = await Contacts.requestPermissionsAsync();
  if (status !== 'granted') return [];
  const { data } = await Contacts.getContactsAsync({
    fields: [Contacts.Fields.Name, Contacts.Fields.Emails], // least-privilege field mask
    pageSize: 100,
  });
  return data.flatMap(c => c.emails?.map(e => e.email) ?? []);
}

async function addContact() {
  await Contacts.addContactAsync({
    [Contacts.Fields.FirstName]: 'Ada',
    [Contacts.Fields.Emails]: [{ email: 'ada@example.com', label: 'work' }],
  });
}
```

### 19.5 `expo-calendar` — Events & Reminders **[A]**

Useful for "add to calendar" and reminder features. Calendars and reminders are **separate** permissions on iOS, each with its own plist string. Find a writable calendar, then create the event.

```ts
import * as Calendar from 'expo-calendar';

async function addEvent(title: string, start: Date, end: Date) {
  const { status } = await Calendar.requestCalendarPermissionsAsync();
  if (status !== 'granted') return;
  const calendars = await Calendar.getCalendarsAsync(Calendar.EntityTypes.EVENT);
  const writable = calendars.find(c => c.allowsModifications);
  if (!writable) return;
  return Calendar.createEventAsync(writable.id, {
    title,
    startDate: start,
    endDate: end,
    timeZone: 'UTC',
    alarms: [{ relativeOffset: -30 }], // remind 30 min before
  });
}
```

### 19.6 `expo-device`, `expo-application` & `expo-network` — Identity, Versioning & Connectivity **[I]**

These three need **no permission** and are invaluable for diagnostics, feature gating, and **security risk signals** (see §20). `expo-device` tells you the hardware and whether you're on a real device or a simulator; `expo-application` gives the app version/build and a vendor-scoped install ID; `expo-network` reports connectivity.

```ts
import * as Device from 'expo-device';
import * as Application from 'expo-application';
import * as Network from 'expo-network';

const riskSignal = {
  model: Device.modelName,                 // "iPhone 16 Pro"
  os: `${Device.osName} ${Device.osVersion}`,
  isPhysical: Device.isDevice,             // false in a simulator — a fraud/test signal
  appVersion: Application.nativeApplicationVersion, // "1.4.2"
  build: Application.nativeBuildVersion,            // "142"
};

async function connectivity() {
  const state = await Network.getNetworkStateAsync();
  return {
    online: state.isConnected && state.isInternetReachable,
    type: state.type, // WIFI | CELLULAR | NONE ...
  };
}
```

**Offline-first implications.** Treat `online === false` as a normal state, not an error: queue mutations locally (e.g. via your data layer's mutation queue or a persisted store), show optimistic UI, and flush when connectivity returns. Combine `Network` with a subscription so a screen reacts the moment the device comes back online rather than failing a request and stranding the user.

### 19.7 `expo-clipboard` — and the Security Caveat **[I]**

Reading and writing the clipboard is trivial; the danger is that **the clipboard is a shared, OS-monitored surface.** Any app — and on iOS, the system itself — can read it, and **iOS shows a paste banner** ("App pasted from Notes") that alarms users if you read the clipboard unprompted.

```ts
import * as Clipboard from 'expo-clipboard';

await Clipboard.setStringAsync('non-secret value');
const text = await Clipboard.getStringAsync(); // only on an explicit user "paste" action
```

> **Security rules (forwarded to §20):** never *auto*-copy secrets (tokens, full card numbers, recovery phrases); if you must copy a one-time code, **clear it after a timeout** (`Clipboard.setStringAsync('')`); never *read* the clipboard except in direct response to a user paste — silent reads trip the iOS paste notification and look like spyware. Persist real secrets in secure storage (§17), never on the clipboard.

### 19.8 Barcode / QR Scanning with `CameraView` **[I]**

The modern scanner lives *inside* `CameraView` (the standalone `expo-barcode-scanner` is retired). Configure the symbologies and handle scans on the camera component directly — this drives payment QR flows, ticket check-in, and onboarding-by-code. It needs the camera permission and generally a **development build** (not Expo Go).

```tsx
import { CameraView, useCameraPermissions } from 'expo-camera';
import { useState } from 'react';

function Scanner({ onCode }: { onCode: (value: string) => void }) {
  const [permission] = useCameraPermissions();
  const [locked, setLocked] = useState(false); // debounce duplicate scans
  if (!permission?.granted) return null;
  return (
    <CameraView
      style={{ flex: 1 }}
      barcodeScannerSettings={{ barcodeTypes: ['qr', 'ean13', 'code128'] }}
      onBarcodeScanned={locked ? undefined : ({ data }) => { setLocked(true); onCode(data); }}
    />
  );
}
```

### 19.9 Screen & System APIs — Brightness, Battery, Orientation, Keep-Awake **[I]**

A cluster of small, permission-free modules that polish UX. `expo-brightness` raises screen brightness for a scan or boarding-pass screen (and restores it after). `expo-battery` reports level/charging state for "low-power" tweaks. `expo-screen-orientation` locks or unlocks rotation per screen (lock the app to portrait but allow a video player to rotate). `expo-keep-awake` prevents the screen from sleeping during a download, a recipe, or a presentation.

```ts
import * as Brightness from 'expo-brightness';
import * as Battery from 'expo-battery';
import * as ScreenOrientation from 'expo-screen-orientation';
import { useKeepAwake } from 'expo-keep-awake';

async function showBoardingPass() {
  const previous = await Brightness.getBrightnessAsync();
  await Brightness.setBrightnessAsync(1);            // max for the scanner
  return () => Brightness.setBrightnessAsync(previous); // restore on unmount
}

const level = await Battery.getBatteryLevelAsync();    // 0–1
await ScreenOrientation.lockAsync(ScreenOrientation.OrientationLock.PORTRAIT_UP);

function VideoScreen() {
  useKeepAwake();           // screen stays on while this component is mounted
  return null;
}
```

> `expo-brightness` does require `WRITE_SETTINGS` on Android (the plugin adds it); the others are permission-free.

### 19.10 Handling "Limited" Photo Access **[A]**

Modern iOS (and Android 14+) lets the user grant access to **specific photos** instead of the whole library. `expo-media-library` surfaces this via an `accessPrivileges` field of `'all' | 'limited' | 'none'`. **Design for `limited` as a first-class state** — don't treat it as denial. Offer a "Select more photos" affordance that re-presents the OS picker so the user can widen the set without going to Settings.

```ts
import * as MediaLibrary from 'expo-media-library';

const perm = await MediaLibrary.requestPermissionsAsync();
if (perm.accessPrivileges === 'limited') {
  // User chose specific photos. Let them add more on demand:
  await MediaLibrary.presentPermissionsPickerAsync();
}
```

### 19.11 `expo-linking` — Deep Links & Universal Links **[I]**

`expo-linking` opens external URLs, deep-links into your own app (`acme://post/42`), and — crucially — opens the OS Settings page that rescues a `canAskAgain: false` permission. Pair it with **universal/app links** (verified `https://` URLs that open your app directly) for the trustworthy, App-Store-preferred experience.

```ts
import * as Linking from 'expo-linking';

await Linking.openSettings();                   // rescue a permanently-denied permission
const url = Linking.createURL('post/42');       // -> acme://post/42 (custom scheme)
Linking.addEventListener('url', ({ url }) => {  // handle an inbound deep link
  const { path, queryParams } = Linking.parse(url);
});
```

> **Security:** deep-link parameters are **untrusted user input** — never use an inbound link to silently perform a privileged action (auto-login, payment, account change) without confirmation. Universal-link verification and the full threat model are covered in §20.

### 19.12 `AppState` — Foreground/Background Transitions **[I]**

`AppState` tells you when the app moves between `active`, `inactive`, and `background`. It is the hook behind the **privacy blur and auto-lock** patterns in §20: blur sensitive screens when the app enters the app-switcher, and require re-authentication (biometrics, §18) on return.

```tsx
import { AppState, AppStateStatus } from 'react-native';
import { useEffect, useRef } from 'react';

function useAutoLock(onBackground: () => void, onForeground: () => void) {
  const prev = useRef<AppStateStatus>(AppState.currentState);
  useEffect(() => {
    const sub = AppState.addEventListener('change', next => {
      if (prev.current === 'active' && next.match(/inactive|background/)) onBackground();
      if (prev.current.match(/inactive|background/) && next === 'active') onForeground();
      prev.current = next;
    });
    return () => sub.remove();
  }, [onBackground, onForeground]);
}
```

### 19.13 `expo-tracking-transparency` (ATT) **[A]**

Before you read the IDFA or do **any cross-app tracking** on iOS, Apple requires you to call the App Tracking Transparency prompt — and you must set `NSUserTrackingUsageDescription`. Without the granted status, the advertising identifier is zeroed out. Request it just-in-time (after a value-explaining screen), not on launch.

```ts
import { requestTrackingPermissionsAsync, getAdvertisingId } from 'expo-tracking-transparency';

const { granted } = await requestTrackingPermissionsAsync();
const idfa = granted ? getAdvertisingId() : null; // null/zeros if not granted
```

### 19.14 Briefly: Print, Sharing & Document Picker **[B]**

Three convenience modules with native-driven UIs and no runtime permission: `expo-print` (render HTML/a view to PDF and hit AirPrint), `expo-sharing` (the native share sheet to hand a file off to another app), and `expo-document-picker` (let the user pick a file from Files/Drive). The system file UIs grant access only to the chosen file, so there's nothing to declare.

```ts
import * as Print from 'expo-print';
import * as Sharing from 'expo-sharing';
import * as DocumentPicker from 'expo-document-picker';

const { uri } = await Print.printToFileAsync({ html: '<h1>Invoice</h1>' });
if (await Sharing.isAvailableAsync()) await Sharing.shareAsync(uri);
const picked = await DocumentPicker.getDocumentAsync({ type: 'application/pdf' });
```

### 19.15 Best Practices **[I]**

- **Least privilege.** Declare and request the minimum permissions; each extra one enlarges your Data Safety label and invites a rejection. Prefer "add-only" photo and "when-in-use" location over their broader siblings.
- **Just-in-time + rationale.** Never prompt on launch. Show a short pre-prompt screen explaining the benefit, *then* fire the OS dialog — higher grants and App Store compliance.
- **Graceful denial paths.** Every permission-gated feature needs a sensible no-permission fallback. Don't dead-end the user; route a permanently-denied permission to `Linking.openSettings()`.
- **Re-check on focus.** Permissions can be revoked in Settings while you're backgrounded, and Android "allow once" grants expire. Re-read state on screen focus; never cache "granted" indefinitely.
- **Handle "limited" photo access** as a first-class state with a "select more" affordance — not as denial.
- **Justify background work for review.** Background location and ATT need a clear, user-visible feature and a written rationale in App Store Connect, or you'll be rejected.
- **Don't block core flows on optional permissions.** If notifications or contacts are nice-to-have, let the user proceed without them and offer the prompt later from a relevant screen.

> **Permissions & device-API checklist:**
> - Every requested permission has a matching iOS `NS*UsageDescription` and Android `<uses-permission>` declared via the config plugin in `app.config.ts`, and `npx expo prebuild --clean` has been re-run.
> - Strings are written for the user (the *why*), and no permission is declared without a real feature behind it.
> - Each permission is requested just-in-time, behind a rationale screen — never on launch.
> - `canAskAgain: false` is handled by deep-linking to `Linking.openSettings()`.
> - Permission state is re-checked on screen focus; "limited" photo access and Android "allow once" are handled, not treated as denial.
> - Background location / ATT are only requested when a real feature needs them, with a review justification ready.
> - Clipboard never auto-copies secrets and is cleared after timeout; secrets live in secure storage (§17), not on the clipboard.
> - Deep-link params are treated as untrusted; `AppState` drives the privacy blur / auto-lock (§18, §20).

---

## 20. Mobile Security Hardening (Banking-Grade)

Everything up to this point made your app *work*. This section makes it *defensible* against a motivated, well-funded attacker — the standard a banking, payments, or healthcare app is held to. The governing reality, which colours every paragraph below, is simple and uncomfortable: **the device is hostile and the client is breakable.** You do not control the phone your code runs on. A determined attacker has root, a debugger, a patched OS, an instrumented runtime (Frida), and unlimited time. So mobile hardening is not about making the client *impenetrable* — that is impossible — it is about **defence in depth**: raising the cost of each attack, detecting compromise to feed risk-scoring, and above all **enforcing every real authorization decision on the server**, where you *do* control the environment. Treat the app as an untrusted UI in front of a trusted backend.

We map each subsection to the **OWASP MASVS** (Mobile Application Security Verification Standard) control groups so a fintech team can audit against an industry framework rather than a personal opinion. The closing checklist gathers it all into a scannable form.

Honesty about Expo up front: a fair amount of banking-grade hardening (root detection, attestation, screenshot-blocking native flags, certificate pinning at the native layer) requires either a **config plugin**, a **prebuild / dev client**, or a **native module**. None of it works in *Expo Go* (Expo Go is for prototyping only — never ship or security-test in it). Where a control needs to leave managed Expo, this section says so explicitly.

### 20.1 The Mobile Threat Model & OWASP MASVS **[A]**

Before writing a single defensive line, write down *who* you are defending against and *what* you are protecting. A bank's threat model is not a blog's. The MASVS organises controls into groups — the ones that matter most here are **MASVS-STORAGE** (data at rest, §17), **MASVS-CRYPTO** (correct key/crypto use), **MASVS-AUTH** (auth & session, §18 + this section), **MASVS-NETWORK** (TLS & pinning), **MASVS-PLATFORM** (IPC, deep links, WebViews, screenshots), **MASVS-CODE** (build hardening, dependencies), and **MASVS-RESILIENCE** (anti-tamper, root/jailbreak, attestation, anti-reverse-engineering). MASTG is the companion *testing guide* that tells a pentester how to break each one — read it as your own red-team checklist.

The single most important principle: **you cannot guarantee anything on a compromised device.** On a rooted phone the attacker can read your "secure" storage, hook your "jailbreak detection" function to return `false`, strip your certificate pinning, and replay your traffic. Therefore client-side controls are **signals and friction**, never the *enforcement boundary*. The enforcement boundary lives on the server (rate limits, fraud scoring, server-verified attestation, transaction signing, idempotency, authorization checks). Design so that even a fully owned client can do *nothing the server would not independently authorize.* See `GO_JWT_ARGON2_GUIDE.md` for how that server-side enforcement (token rotation, reuse detection, Argon2 password hashing, payload encryption) is actually built.

```text
THREAT MODEL (write your own version of this)
  Assets:        access/refresh tokens, PII, account balances, payment instruments, session
  Adversaries:   malware on device, rooted-device fraudster, MITM proxy, repackaged-app phisher,
                 shoulder-surfer, lost/stolen device holder, malicious 3rd-party SDK
  Assumption:    attacker may have ROOT + DEBUGGER + FRIDA + patched OS  → client is breakable
  Strategy:      defence in depth + detect-and-risk-score + ENFORCE ON SERVER
```

### 20.2 Network Security — TLS & Certificate/Public-Key Pinning **[A]**

**MASVS-NETWORK.** All traffic must be HTTPS over **TLS 1.2 or 1.3** — no cleartext, no exceptions, no "just for this one analytics call." But TLS alone trusts the device's CA store, and that store is the weak point: a corporate MITM proxy, malware, or a mis-issued certificate from a compromised CA can present a *valid* certificate the OS happily accepts, letting an attacker read and modify your API traffic. **Certificate pinning** closes this: the app ships knowing exactly which certificate (or, better, which public key) your API is allowed to present, and refuses any connection that does not match — regardless of what the device CA store says.

Pin to the **public key / SPKI (Subject Public Key Info) hash**, *not* to the leaf certificate. Leaf certs expire and rotate frequently (Let's Encrypt every 90 days; many CAs are moving toward ~47-day lifetimes); the underlying *key pair* can be kept stable across renewals, so SPKI pins survive routine cert rotation. Always ship **at least one backup pin** (a second key you control, kept offline) so you can rotate the primary key without an emergency app release. The operational danger is real and must be respected: **a wrong or stale pin bricks the app for every user** the moment the server cert changes, and you cannot hotfix it for users who can no longer reach your (pinned) update server. Pinning is therefore a *process*, not a config line: stage the next pin in the app *before* you rotate the key on the server.

For full TLS theory (handshake, cipher suites, OCSP, HSTS) defer to `NETWORKING_GUIDE.md`; for terminating TLS and configuring strong ciphers at the edge see `NGINX_GUIDE.md`. In RN/Expo you have a few implementation paths:

```ts
// pinning.ts — pinning with `react-native-ssl-pinning` (needs a dev client / prebuild;
// does NOT work in Expo Go). The library swaps fetch for a pinned native networking stack.
import { fetch as pinnedFetch } from 'react-native-ssl-pinning';

// Compute pins as: openssl x509 -in cert.pem -pubkey -noout |
//   openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | openssl enc -base64
// Pin the PUBLIC KEY (SPKI), and include a BACKUP pin you control offline.
const PINS = [
  'sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=', // current SPKI
  'sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=', // backup SPKI (rotation)
];

export async function securePost(url: string, body: unknown, accessToken: string) {
  return pinnedFetch(url, {
    method: 'POST',
    timeoutInterval: 10_000,
    sslPinning: { certs: PINS },                 // SPKI pins enforced natively
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${accessToken}`,    // short-lived token, §20.7
    },
    body: JSON.stringify(body),
  });
  // A pin mismatch throws here — treat it as "possible MITM": abort, do NOT retry unpinned.
}
```

Alternatively, pin declaratively at the OS layer via an Expo **config plugin** that injects Android Network Security Config and iOS settings — survives reinstalls and applies to *all* native traffic, but still requires prebuild:

```json
// Android res/xml/network_security_config.xml (injected by a config plugin)
{
  "_comment": "conceptual mapping; the plugin writes the real XML",
  "domain-config": {
    "domain": "api.yourbank.example",
    "pin-set": {
      "expiration": "2026-12-31",
      "pins": [
        { "digest": "SHA-256", "value": "AAAA...current" },
        { "digest": "SHA-256", "value": "BBBB...backup" }
      ]
    }
  }
}
```

### 20.3 Secrets in the App — There Are None **[A]**

**MASVS-CODE / MASVS-CRYPTO.** Internalise this as a law of physics: **there are no secrets in a mobile binary.** Anything you ship — API keys, signing secrets, "obfuscated" strings, encryption keys baked into the bundle — is *extractable* by anyone with the IPA/APK and an afternoon. Hermes bytecode and minification slow this down; they do not stop it. A `strings` dump, a Frida hook on your crypto calls, or a proxy on first launch surfaces it.

Practical consequences for a banking app:

- **Never embed private keys or high-privilege secrets** (payment-provider secret keys, admin tokens, master encryption keys, third-party server-side API keys). If leaking it would let an attacker act *as your server*, it does not belong in the app.
- **Only ship public / publishable keys** — Stripe *publishable* key, public OAuth client IDs, public pinning keys. These are designed to be public.
- **`EXPO_PUBLIC_` env vars are literally public.** Expo inlines them into the JS bundle at build time. They are a convenience for non-secret config (API base URL, public keys), nothing more. A secret in `EXPO_PUBLIC_STRIPE_SECRET` is a secret in your APK.
- **Proxy privileged calls through your backend.** The app calls *your* server with the user's session; your server holds the real provider secret and makes the privileged call. This also gives you a single place to enforce authorization and rate limits.
- **Fetch sensitive runtime config from the server after auth**, scoped to the authenticated user, never bundled.

```ts
// config.ts
// OK to inline — these are public by design:
export const API_BASE = process.env.EXPO_PUBLIC_API_BASE!;          // public URL
export const STRIPE_PUBLISHABLE = process.env.EXPO_PUBLIC_STRIPE_PK!; // public key

// NOT here, NOT EXPO_PUBLIC_, NOT in the bundle:
//   - Stripe SECRET key            -> backend only, proxied
//   - JWT signing secret           -> backend only (see GO_JWT_ARGON2_GUIDE.md)
//   - DB credentials / 3p secrets  -> backend only
//   - any key whose leak == game over
```

### 20.4 Jailbreak / Root / Emulator / Tamper Detection & Attestation **[A]**

**MASVS-RESILIENCE.** A rooted or jailbroken device, an emulator, or a Frida-instrumented runtime are all environments where your client-side defences can be neutralised. You cannot *prevent* them, but you can *detect* them and feed that into server-side **risk scoring** — the same way banks decide whether to allow, step-up-challenge, or soft-block a transaction. The hard rule: **every one of these signals must be verified or consumed server-side; a client that says "I'm not rooted" is untrusted by definition.**

Two tiers exist, and a banking app uses both:

1. **Heuristic detection (cheap, bypassable, still useful):** `jail-monkey` and native checks look for su binaries, writable system paths, suspicious packages (Magisk, Cydia), debugger attachment, hooking frameworks (Frida/Xposed), and emulator fingerprints. An attacker can hook these to lie — so treat a *positive* as high-confidence ("definitely compromised") and a *negative* as merely "no obvious compromise."
2. **Hardware-backed integrity attestation (strong, the real anti-fraud signal):** **iOS App Attest / DeviceCheck** and **Android Play Integrity API** ask the OS+hardware to produce a cryptographically signed statement that *this is a genuine, unmodified app on a genuine, non-rooted device, installed from the official store*. The token is **verified on your server against Apple/Google**, so a hooked client cannot forge it. This is what actually moves the needle against fraud. In Expo this needs native modules/config plugins (`expo-device` for basic info; App Attest / Play Integrity via community modules or a custom native module + a dev client) — **not available in Expo Go.**

The pattern is **risk-based "soft block / step-up,"** never a hard client-side `if (rooted) crash()` (which is trivially patched and annoys legitimate rooted-but-honest users):

```ts
// risk.ts — gather client signals, but let the SERVER decide.
import JailMonkey from 'jail-monkey';
import * as Device from 'expo-device';
import { getPlayIntegrityToken, getAppAttestToken } from './nativeAttest'; // native module

export async function buildRiskContext() {
  const heuristics = {
    rooted: JailMonkey.isJailBroken(),
    onExternalStorage: JailMonkey.isOnExternalStorage(),
    debugged: JailMonkey.isDebuggedMode?.() ?? false,
    hookDetected: JailMonkey.hookDetected?.() ?? false,
    isEmulator: !Device.isDevice,           // expo-device: false on simulators/emulators
  };
  // Hardware attestation: opaque token, ONLY meaningful after server verification.
  const attestation = Device.osName === 'iOS'
    ? await getAppAttestToken()             // Apple App Attest
    : await getPlayIntegrityToken();        // Android Play Integrity
  return { heuristics, attestation };
}

// On a sensitive action, send the context; the server scores it.
async function transfer(amount: number, to: string, token: string) {
  const risk = await buildRiskContext();
  const res = await securePost(`${API_BASE}/transfers`, { amount, to, risk }, token);
  // Server verifies attestation w/ Apple/Google, combines with heuristics + behavioural
  // signals, then responds: 200 OK | 401 step-up (re-auth/biometric, §20.7) | 403 soft-block.
  return res;
}
```

```ts
// SERVER-SIDE (conceptual — see GO_JWT_ARGON2_GUIDE.md for the real Go implementation)
// 1. Verify Play Integrity / App Attest token signature against Google/Apple.  <-- the real gate
// 2. Score: failed attestation + rooted + new device + large amount => deny or step-up.
// 3. NEVER trust risk.heuristics alone; they're advisory. Attestation is the trust anchor.
```

### 20.5 Anti-Reverse-Engineering **[A]**

**MASVS-RESILIENCE / MASVS-CODE.** Your JS ships as a bundle inside the app; anyone can unzip the IPA/APK and read it. You cannot make it unreadable, but you can make it *expensive* to read and remove easy footholds:

- **Ship Hermes bytecode, not plain JS.** Hermes (the default engine in modern Expo/RN, §1) compiles your JS to bytecode at build time. It is far less legible than minified JS and there is no human-readable source to grep — a meaningful speed bump for casual reversing.
- **Minify and remove dead code** in production (Metro does this by default for release builds).
- **Strip source maps from the shipped binary.** Upload them privately to your crash reporter (Sentry/Bugsnag) for symbolication, but **never bundle them** — a source map turns your bytecode back into readable, commented source.
- **Strip `console.*` and debug logs in production** (e.g. `babel-plugin-transform-remove-console`), both for performance and to avoid leaking internal logic, endpoints, or values.
- **For high-value targets, add native obfuscation / RASP** (commercial tooling, control-flow obfuscation, string encryption, anti-debug). Know the limit: obfuscation raises cost, it does not provide secrecy or authorization — it buys time, nothing more.

```json
// app.json (excerpt) — confirm Hermes (default in SDK 54) + production hardening
{
  "expo": {
    "jsEngine": "hermes",
    "extra": { "router": {} },
    "_comment": "Release builds: minified + Hermes bytecode; source maps uploaded to Sentry, NOT shipped"
  }
}
```

```bash
# Verify what actually ships before release (managed Expo / EAS):
npx expo export --platform android            # inspect dist/ — confirm no .map files leak into the app
# In babel.config.js (production only): plugins: ['transform-remove-console']
# Source maps -> Sentry via EAS (sentry-expo / @sentry/react-native upload), kept private.
```

### 20.6 Screen Privacy — Screenshots, Recording & App-Switcher Blur **[A]**

**MASVS-PLATFORM.** Sensitive screens (balances, statements, card PANs, OTPs) leak through three everyday channels: **screenshots**, **screen recording**, and the **app-switcher thumbnail** the OS captures when you background the app. Banking apps block all three.

- **Block screenshots/recording** with `expo-screen-capture`'s `preventScreenCaptureAsync()`. On **Android** this sets the native `FLAG_SECURE`, which genuinely prevents screenshots, blocks screen recording, and blanks the app-switcher thumbnail — a strong native control. On **iOS** the OS does *not* let apps hard-block screenshots; you instead **detect** screenshots/recording (`addScreenshotListener`, `usePreventScreenCapture`, and `isScreenRecording` / recording listeners) and respond — blur content, warn the user, or revoke the displayed secret.
- **Background privacy overlay:** when the app is backgrounded or inactive, render an opaque cover so the app-switcher thumbnail (and any screen recording mid-switch) shows nothing sensitive. Drive it off `AppState`.

```tsx
// SensitiveScreen.tsx — block capture while mounted (Android FLAG_SECURE; iOS detect+blur)
import { useEffect } from 'react';
import * as ScreenCapture from 'expo-screen-capture';

export function useCaptureGuard() {
  // Hook form handles add+remove automatically while this screen is mounted.
  ScreenCapture.usePreventScreenCapture('sensitive-screen');

  useEffect(() => {
    // iOS can't hard-block screenshots — detect and react (warn / re-auth / log to risk engine).
    const sub = ScreenCapture.addScreenshotListener(() => {
      // e.g. show a warning, scrub the secret from view, raise a security event server-side.
    });
    return () => sub.remove();
  }, []);
}
```

```tsx
// PrivacyOverlay.tsx — cover sensitive content in the app switcher (all platforms)
import { useEffect, useState } from 'react';
import { AppState, View, StyleSheet } from 'react-native';

export function PrivacyOverlay({ children }: { children: React.ReactNode }) {
  const [obscured, setObscured] = useState(false);
  useEffect(() => {
    const sub = AppState.addEventListener('change', (s) => {
      // 'inactive' fires on iOS the instant the switcher animates — cover BEFORE the snapshot.
      setObscured(s !== 'active');
    });
    return () => sub.remove();
  }, []);
  return (
    <View style={{ flex: 1 }}>
      {children}
      {obscured && <View style={[StyleSheet.absoluteFill, styles.cover]} />}
    </View>
  );
}
const styles = StyleSheet.create({ cover: { backgroundColor: '#0B1F3A' } }); // opaque brand cover
```

### 20.7 Session & Auth Hardening on the Device **[A]**

**MASVS-AUTH.** Session handling is where most "the app got hacked" incidents actually originate. The banking-grade pattern:

- **Short-lived access token in memory only** (minutes-long lifetime). Keep it in a module-scope variable / state, *not* on disk — memory is wiped when the process dies and is far harder to exfiltrate than storage.
- **Refresh token in biometric-gated SecureStore** (§17 for SecureStore/Keychain/Keystore, §18 for biometrics). Gate retrieval behind Face ID / fingerprint with `requireAuthentication`, so even a stolen unlocked-storage dump is useless without the live biometric.
- **Rotation & reuse detection on the server.** Each refresh issues a new refresh token and invalidates the old one; if an *old* (already-rotated) refresh token is ever presented, treat it as theft → revoke the whole token family and force re-auth. This is implemented server-side — see `GO_JWT_ARGON2_GUIDE.md` for rotation, reuse detection, and JWT signing.
- **Inactivity auto-lock:** lock the app (require biometric/PIN) after N minutes idle and on every cold start; never leave an authenticated session resumable indefinitely.
- **Step-up / re-auth for sensitive operations** (transfers, adding payees, changing limits): demand a fresh biometric or PIN even within an active session.
- **Logout wipes everything:** clear in-memory tokens, delete SecureStore entries, clear any cached PII (React Query cache, async storage, image cache), and call the server to revoke the session. Handle suspected token theft the same way (server-initiated family revocation).

```ts
// session.ts — access token in memory, refresh token biometric-gated on disk
import * as SecureStore from 'expo-secure-store';

let accessToken: string | null = null;            // MEMORY ONLY — never persisted
export const getAccess = () => accessToken;
export const setAccess = (t: string | null) => { accessToken = t; };

const REFRESH_KEY = 'refresh_token';

export async function saveRefresh(token: string) {
  await SecureStore.setItemAsync(REFRESH_KEY, token, {
    keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY, // not backed up / migrated
    requireAuthentication: true,                  // Face ID / fingerprint gate (§18)
    authenticationPrompt: 'Authenticate to continue',
  });
}

export async function getRefresh(): Promise<string | null> {
  return SecureStore.getItemAsync(REFRESH_KEY, { requireAuthentication: true });
}

export async function logout(revokeOnServer: (t: string) => Promise<void>) {
  if (accessToken) { try { await revokeOnServer(accessToken); } catch {} } // tell server to kill session
  accessToken = null;                              // wipe memory
  await SecureStore.deleteItemAsync(REFRESH_KEY);  // wipe disk
  // also: queryClient.clear(); clear image cache; drop any cached PII.
}
```

### 20.8 Clipboard, Keyboard & Logging Hygiene **[A]**

**MASVS-STORAGE / MASVS-PLATFORM.** Secrets leak through mundane OS conveniences:

- **Clipboard:** never auto-copy tokens/PANs/OTPs; if a "copy" affordance is genuinely needed, **clear the clipboard after a short timeout** and warn that other apps can read it. The system clipboard is shared and readable by any app (and, on Android, sometimes synced across devices).
- **Keyboard caching / autofill / autocorrect:** on secret fields (PIN, card number, password, OTP) disable predictive text and the keyboard's learning cache. Set `secureTextEntry`, `autoCorrect={false}`, `autoComplete="off"`, `spellCheck={false}`, `keyboardType` appropriately, and `textContentType="oneTimeCode"` only where you *want* OS OTP autofill.
- **Third-party keyboards** can keylog. Where the platform allows, prefer the system keyboard for secret entry (iOS lets you decline custom keyboards; on Android, advise users in-app).
- **Logging & crash reports:** **never log tokens, PII, card data, or request bodies.** Scrub them from crash reports and analytics before they leave the device (Sentry `beforeSend` redaction). Remember §20.5: strip `console.*` in production. A token in a Sentry breadcrumb is a token in a third party's database.

```tsx
// SecretField.tsx — keyboard hardening for a sensitive input
import { TextInput } from 'react-native';
import * as Clipboard from 'expo-clipboard';

export const SecretField = (props: React.ComponentProps<typeof TextInput>) => (
  <TextInput
    secureTextEntry                 // masks input; opts out of keyboard learning on most OSes
    autoCorrect={false}             // no predictive-text cache of the secret
    autoComplete="off"
    spellCheck={false}
    importantForAutofill="no"       // Android: keep out of autofill framework
    textContentType="none"         // iOS: don't offer to save/autofill as a known field
    {...props}
  />
);

// Clipboard with auto-clear (only if copy is truly required):
export async function copyEphemeral(value: string, ms = 30_000) {
  await Clipboard.setStringAsync(value);
  setTimeout(async () => {
    if ((await Clipboard.getStringAsync()) === value) await Clipboard.setStringAsync('');
  }, ms);
}
```

### 20.9 Deep-Link & Universal-Link Security **[A]**

**MASVS-PLATFORM.** Deep links are an attack surface: any app (or a malicious web page) can fire a link at yours. The rules:

- **Validate and authorize every deep-link action server-side; never perform a privileged action straight from a link.** A link should *navigate*, presenting a screen that then runs the app's normal authenticated, authorized flow — it must not, by itself, transfer money, change a payee, or confirm anything.
- **Prefer verified App Links (Android) / Universal Links (iOS) over custom schemes.** Custom schemes (`yourbank://`) can be **hijacked** — another app can register the same scheme and intercept the link. Verified links are cryptographically bound to your domain (`assetlinks.json` / `apple-app-site-association`) and cannot be claimed by an impostor.
- **Link-based auth (magic links / OTP links) must be one-time, short-lived, and bound** to the originating device/session; consume server-side and reject reuse. Never embed long-lived tokens in a link (they leak via history, referrers, and shoulder-surfing).
- **Guard against link injection / open-redirect:** treat every parameter as hostile, allow-list redirect targets, and never `eval`/route to arbitrary URLs from a link.

```ts
// linking.ts — sanitize and gate deep links (Expo Router)
import * as Linking from 'expo-linking';

const ALLOWED = new Set(['account', 'statements', 'payee']); // navigation targets only

export function handleDeepLink(url: string, isAuthenticated: boolean, navigate: (p: string) => void) {
  const { hostname, path, queryParams } = Linking.parse(url);
  const target = (path ?? '').split('/')[0];
  if (!ALLOWED.has(target)) return;               // reject unknown / injected routes

  // Privileged intents NEVER auto-execute. They route into the normal authed+authorized flow,
  // which re-verifies on the server and may demand step-up (§20.7).
  if (!isAuthenticated) { navigate(`/login?next=${encodeURIComponent(target)}`); return; }
  navigate(`/${target}`);                          // params re-validated server-side on the screen
}
```

### 20.10 Supply Chain & Runtime Integrity **[A]**

**MASVS-CODE / MASVS-RESILIENCE.** Your app is only as trustworthy as everything it pulls in and everything it can be updated with:

- **Pin and audit dependencies.** Lockfile committed, `npm audit` / a tool like Socket in CI, watch for typosquats and compromised packages. A malicious transitive dependency runs with your app's full privileges and can exfiltrate tokens. Minimise third-party SDKs in a banking app — every SDK is code you didn't write running next to your secrets.
- **Sign and verify OTA updates.** Expo **EAS Update** can be hijacked if unsigned: an attacker who can MITM or compromise the update channel could push malicious JS to every user. Enable **EAS Update code signing** so the client verifies the update bundle's signature against a key you control, and refuses unsigned/tampered updates. Combine with §20.2 pinning so the update channel itself is MITM-resistant. (See §14 for EAS Build/Update mechanics and shipping.)
- **Code-sign the binaries** (iOS provisioning/notarization, Android signing key in a secure keystore / EAS-managed credentials) and protect the signing keys — a leaked signing key lets an attacker ship a *trusted* malicious update.
- **All real authorization happens on the server.** This is the thread that ties the whole section together: pinning, attestation, risk scores, and tamper checks are *inputs* to a server decision. The server independently authorizes every transaction, re-checks limits and ownership, enforces idempotency, and treats the client as an untrusted petitioner. Build that side per `GO_JWT_ARGON2_GUIDE.md`.

```json
// app.json (excerpt) — EAS Update code signing so OTA can't be hijacked
{
  "expo": {
    "updates": {
      "url": "https://u.expo.dev/your-project-id",
      "codeSigningCertificate": "./code-signing/certificate.pem",
      "codeSigningMetadata": { "keyid": "main", "alg": "rsa-v1_5-sha256" }
    },
    "runtimeVersion": { "policy": "appVersion" }
  }
}
```

```bash
# Generate signing material and configure EAS Update signing (one-time):
npx eas-cli codesigning:configure        # creates key + cert, wires app.json
npm audit --omit=dev                     # audit production deps in CI; fail the build on highs
```

> **Banking-grade mobile security checklist:**
> *Map each item to its OWASP MASVS group; a fintech team should be able to mark every line ✅/❌/N-A.*
>
> **MASVS-RESILIENCE — threat model & anti-tamper**
> - [ ] Written threat model: assets, adversaries, "attacker has root + debugger + Frida" assumption documented.
> - [ ] Root/jailbreak detection (`jail-monkey` + native) feeding risk scoring — never a hard client-side crash.
> - [ ] Debugger / Frida / Xposed / hooking detection wired into risk context.
> - [ ] Emulator detection (`expo-device`).
> - [ ] Hardware attestation: **iOS App Attest/DeviceCheck** + **Android Play Integrity**, **verified server-side** (the real trust anchor; needs dev client, not Expo Go).
> - [ ] "Soft block / step-up on risk" pattern — server decides allow / step-up / deny.
>
> **MASVS-NETWORK — transport**
> - [ ] HTTPS everywhere, TLS 1.2/1.3 only, no cleartext (verify `NETWORKING_GUIDE.md` / `NGINX_GUIDE.md`).
> - [ ] Certificate/**public-key (SPKI)** pinning, with ≥1 offline backup pin and a documented rotation runbook.
> - [ ] Pin rollout precedes server key rotation (no app-bricking).
>
> **MASVS-CODE / MASVS-CRYPTO — secrets & build**
> - [ ] No private/high-privilege secrets in the binary; privileged calls proxied through the backend.
> - [ ] Only public/publishable keys shipped; `EXPO_PUBLIC_` used only for non-secret config.
> - [ ] Sensitive runtime config fetched server-side after auth, scoped to the user.
> - [ ] Hermes bytecode enabled; release minified; **source maps stripped from the app** (uploaded privately to crash tooling).
> - [ ] `console.*` / debug logs removed in production; native obfuscation/RASP for high-value targets.
> - [ ] Dependencies pinned + audited (CI), third-party SDK surface minimised.
>
> **MASVS-AUTH — session**
> - [ ] Access token in memory only, short-lived; refresh token in **biometric-gated SecureStore** (§17/§18).
> - [ ] Server-side refresh-token rotation + reuse detection (family revocation) — see `GO_JWT_ARGON2_GUIDE.md`.
> - [ ] Inactivity auto-lock + lock on cold start; step-up/re-auth for sensitive ops.
> - [ ] Logout wipes memory + SecureStore + cached PII and revokes the session server-side.
>
> **MASVS-PLATFORM — platform interaction**
> - [ ] Screenshot/recording blocked (Android `FLAG_SECURE` via `expo-screen-capture`; iOS detect + blur).
> - [ ] App-switcher / background privacy overlay (`AppState` → opaque cover before the OS snapshot).
> - [ ] Deep links validated/authorized; no privileged action straight from a link.
> - [ ] Verified App Links / Universal Links instead of hijackable custom schemes; link-auth one-time, short-lived, device-bound.
> - [ ] Clipboard never holds secrets (auto-clear if unavoidable); keyboard caching/autofill disabled on secret fields; third-party keyboards discouraged on secret entry.
> - [ ] No PII/tokens in logs, analytics, or crash reports (scrub before send).
>
> **MASVS-RESILIENCE — update integrity**
> - [ ] EAS Update **code signing** enabled; client rejects unsigned/tampered OTA bundles (§14).
> - [ ] Binaries code-signed; signing keys protected (EAS-managed credentials / secure keystore).
>
> **The overriding rule**
> - [ ] **Every real authorization decision is enforced on the server.** Assume the client is fully compromised and confirm the app still cannot do anything the server wouldn't independently authorize.

---

## 21. Production Release: App Store & Play Store

Shipping a banking-grade React Native + Expo app is mostly a *compliance and process* exercise, not a coding one. The build mechanics — `eas.json`, EAS Build/Submit/Update — are covered in §14; this section assumes you can produce binaries and focuses on what the **stores** demand: identifiers and signing, privacy declarations that must match reality, listing assets, the extra scrutiny fintech apps attract, and a safe rollout/rollback strategy. Get any of the privacy or signing pieces wrong and you do not ship — Apple and Google reject before users ever see the app.

Treat this as a checklist-driven chapter. Read it alongside §19 (permission/privacy *strings*), §17 (secure storage of tokens/keys), and §20 (runtime security hardening) — the store reviewers will probe exactly those areas in a banking app.

### 21.1 Structuring an app for production (dev / staging / prod) **[A]**

A production app is never one monolith. You want **three logical environments** — development, staging, and production — that can be built independently and, ideally, installed *side by side* on a tester's device. The cleanest pattern in Expo is a dynamic `app.config.ts` that reads a single environment variable (set per EAS build profile) and derives the bundle identifier, app name, icon, and API base URL from it.

Non-secret runtime config travels as `EXPO_PUBLIC_*` env vars (inlined into the JS bundle — never put secrets here). Real secrets (signing material is handled by EAS itself; API server secrets stay server-side) use **EAS secrets** / environment variables on the build, read at build time. See §14 for how profiles and secrets are wired into `eas.json`.

```ts
// app.config.ts — one config, three environments
import { ExpoConfig, ConfigContext } from 'expo/config';

const ENV = process.env.APP_ENV ?? 'development'; // set per EAS build profile

const variants = {
  development: { suffix: '.dev',     name: 'Acme Bank (Dev)',     scheme: 'acmebank-dev' },
  staging:     { suffix: '.staging', name: 'Acme Bank (Staging)', scheme: 'acmebank-stg' },
  production:  { suffix: '',         name: 'Acme Bank',           scheme: 'acmebank' },
} as const;

export default ({ config }: ConfigContext): ExpoConfig => {
  const v = variants[ENV as keyof typeof variants];
  return {
    ...config,
    name: v.name,
    slug: 'acme-bank',
    scheme: v.scheme,
    ios:     { bundleIdentifier: `com.acme.bank${v.suffix}` },
    android: { package:          `com.acme.bank${v.suffix}` },
    extra: {
      env: ENV,
      apiUrl: process.env.EXPO_PUBLIC_API_URL, // public, build-time
      eas: { projectId: '...' },
    },
  };
};
```

Distinct bundle IDs / packages per variant are what let a tester keep **staging and prod installed at once** — the OS keys apps by identifier, so `com.acme.bank.staging` and `com.acme.bank` are different apps. Give each variant its own icon tint so nobody demos the wrong build to a reviewer.

**Versioning** is the part people get wrong. There are two numbers and they mean different things:

| Field | iOS | Android | Visible to user? | Must increase per upload? |
|---|---|---|---|---|
| Marketing version | `version` (e.g. `2.4.0`) | `version` → `versionName` | Yes | No (can resubmit same `2.4.0`) |
| Build number | `buildNumber` | `versionCode` (integer) | No | **Yes** — every store upload must be higher |

The cleanest approach is to let EAS own the build counter. Set `"appVersionSource": "remote"` and `"autoIncrement": true` in the relevant build profile so EAS tracks and bumps `buildNumber`/`versionCode` server-side — no more "build already exists" rejections from forgetting to increment (mechanics in §14). Bump the marketing `version` by hand per release using semver.

```jsonc
// eas.json (excerpt) — remote version management; full file in §14
{
  "cli": { "appVersionSource": "remote" },
  "build": {
    "production": { "autoIncrement": true, "channel": "production" },
    "staging":    { "autoIncrement": true, "channel": "staging",
                    "env": { "APP_ENV": "staging" } }
  }
}
```

Other production-structure notes:

- **Release branching:** cut a `release/2.4.0` branch, build staging from it, fix forward, then tag and build production from the same commit. Keep `main` deployable.
- **Feature flags:** ship risky features dark and toggle server-side (e.g. LaunchDarkly / your own config endpoint). Lets you disable a broken feature without a store resubmit and lets reviewers see a stable surface.
- **Monorepo:** Expo supports monorepos; pin the app to a workspace, ensure Metro `watchFolders` and hoisting are configured, and keep one EAS project per app. Build profiles still drive the env.

### 21.2 iOS code signing & identifiers **[A]**

iOS signing is a chain: an **Apple Developer Program** membership ($99/yr, organization enrollment with a D-U-N-S number for a bank) → an **App ID / bundle identifier** registered in the developer portal → **capabilities/entitlements** enabled on that App ID → a **distribution certificate** (proves who you are) → a **provisioning profile** (binds certificate + App ID + entitlements). The signed binary must match all of these or App Store Connect rejects it at upload.

For a banking app you typically enable these capabilities/entitlements — each must be declared on the App ID *and* in the entitlements file:

| Capability / entitlement | Why a banking app needs it |
|---|---|
| Push Notifications | Transaction alerts, OTP/step-up prompts |
| Associated Domains | Universal Links + **passkey / WebAuthn** domain association (`webcredentials:`) |
| App Attest (DeviceCheck) | Attest the app is genuine/unmodified before trusting a device (anti-fraud) |
| Keychain Sharing | Share credentials/tokens across an app group if you have an extension |
| Sign in with Apple | Mandatory if you offer third-party social login (see §21.4) |
| Face ID usage | Biometric unlock (`NSFaceIDUsageDescription` string — §19) |

**EAS-managed signing vs manual.** By default `eas build` *manages signing for you*: it can create the distribution certificate and provisioning profile in your Apple account and store them securely, regenerating profiles when capabilities change. This is the recommended path. Banks with a security team that insists on holding their own certificates can instead use **local credentials** (`credentials.json`) and supply the `.p12` + `.mobileprovision`; you then own renewal. Manage either way with `eas credentials`. Whatever you choose, the **distribution certificate is shared infrastructure** — store it in a vault, not on one engineer's laptop.

**App Store Connect flow:** create the *app record* (matching the bundle ID), upload the build (EAS Submit, §14), wait for **build processing** (minutes to ~an hour), then distribute via **TestFlight** before App Store review:

| TestFlight track | Audience | Review needed? | Use for a bank |
|---|---|---|---|
| Internal testing | Up to 100 team members (App Store Connect users) | No | Daily QA, security team, demo prep |
| External testing | Up to 10,000 invited/public testers | **Yes** — a lighter Beta App Review | UAT, pilot customers, beta cohort |

### 21.3 Android signing & build format **[A]**

Android's modern signing model is **Google Play App Signing**, and you should opt in. You generate an **upload key**; Google holds the real **app signing key** and re-signs your uploads for distribution. The win: if your upload key is ever lost or compromised you can rotate it with Google's help, and you never risk the master signing identity that users' devices trust. EAS generates and manages the upload keystore for you (or you can supply your own via `eas credentials`); back up the keystore credentials in your vault regardless.

```bash
# Build a Play-ready AAB (Submit/credentials mechanics in §14)
eas build --platform android --profile production   # produces an .aab
```

Key Android facts for production:

- **Ship an AAB, not an APK.** The Play Store requires the **Android App Bundle** (`.aab`); Google generates per-device optimized APKs from it. Build APKs only for sideloaded internal QA.
- **`versionCode`** is an integer that must strictly increase on every upload — let EAS `autoIncrement` it (§21.1).
- **Target API level:** as of 2026, new apps and updates must target **API level 35 (Android 15)** or the console blocks the upload. Expo SDK 54 targets this by default.

Play testing tracks mirror TestFlight but are percentage-based:

| Track | Audience | Notes |
|---|---|---|
| Internal testing | Up to 100 testers, fast (~minutes) | Day-to-day QA |
| Closed testing | Named lists / Google Groups | UAT, pilot; **required** to graduate a brand-new personal-account app |
| Open testing | Public opt-in beta | Wider beta before production |
| Production | Everyone, supports **staged % rollout** | See §21.8 |

### 21.4 iOS privacy & review compliance (critical in 2026) **[A]**

This is where banking apps get rejected. Apple enforces several privacy mechanisms; **declarations must match what the app actually does**, and several are hard requirements.

- **Privacy Manifest (`PrivacyInfo.xcprivacy`)** — *mandatory*. An XML file declaring (a) every category of data your app and its SDKs collect, and (b) every **"required-reason API"** you call (e.g. `UserDefaults`, file timestamps, disk space, system boot time) with an approved reason code. Apple now **rejects** apps missing or mis-declaring this. Expo SDK 54 auto-generates a baseline manifest and many config plugins contribute their entries; you must still audit it and add reasons for any custom native code. Third-party SDKs that traffic in financial data must also ship their own manifest.

```xml
<!-- PrivacyInfo.xcprivacy (excerpt) — Expo generates a base; audit & extend -->
<dict>
  <key>NSPrivacyTracking</key><false/>
  <key>NSPrivacyCollectedDataTypes</key>
  <array>
    <dict>
      <key>NSPrivacyCollectedDataType</key>
      <string>NSPrivacyCollectedDataTypePaymentInfo</string>
      <key>NSPrivacyCollectedDataTypeLinkedToUser</key><true/>
      <key>NSPrivacyCollectedDataTypeUsedForTracking</key><false/>
    </dict>
  </array>
  <key>NSPrivacyAccessedAPITypes</key>
  <array>
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array><string>CA92.1</string></array> <!-- app's own storage -->
    </dict>
  </array>
</dict>
```

- **App Privacy "Nutrition Labels"** — a questionnaire in App Store Connect that produces the public privacy summary on your store page. Must agree with the Privacy Manifest and the Data Safety form (§21.5). A bank declares Financial Info, Contact Info, Identifiers, etc., and whether each is linked to the user / used for tracking.
- **App Tracking Transparency (ATT):** if you track users across other apps/sites (advertising attribution), you must call `requestTrackingAuthorization` and show the prompt with `NSUserTrackingUsageDescription` (§19). Most banks should simply **not track** and declare so — fewer rejections, cleaner labels.
- **Sign in with Apple:** if your app offers any third-party/social login (Google, Facebook), you **must** also offer Sign in with Apple. A bank-issued-credentials-only login is exempt; the moment you add social login, add Apple.
- **In-app account deletion:** if users can create an account, the app must let them **initiate deletion from within the app** (not just a web/support route). For regulated finance you may *retain* records for legal reasons — that's fine, but the user-facing deletion request flow must exist in-app.
- **Export compliance (`ITSAppUsesNonExemptEncryption`):** every banking app uses encryption (HTTPS, keychain). Standard/exempt cryptography qualifies for the simplified exemption — set the flag so you skip the upload-time question every build:

```jsonc
// app.config.ts → ios.infoPlist  (declares standard-crypto exemption)
"ios": {
  "infoPlist": { "ITSAppUsesNonExemptEncryption": false }
}
```
Use `false` only if you rely solely on standard/exempt encryption; proprietary crypto requires a French export declaration and self-classification (CCATS), so confirm with legal.

### 21.5 Android privacy & policy compliance **[A]**

- **Data Safety form** (Play Console) — Android's analogue to Apple's labels. You declare what data is **collected** vs **shared**, whether it's encrypted in transit, and whether users can request deletion. It must match the app's real behavior and your privacy policy; mismatches are a top rejection/enforcement cause.
- **Target API level:** must target **API 35** (§21.3) or the upload is blocked.
- **Sensitive-permission justification:** permissions like background location, SMS/Call Log, `QUERY_ALL_PACKAGES`, or `MANAGE_EXTERNAL_STORAGE` require a written, in-console justification and often a demo video. A bank rarely needs these — request only what you use (§19) and remove permissions pulled in transitively by SDKs.
- **Financial-services policy:** Google has a dedicated financial-products policy. Personal-loan/credit apps face extra disclosure rules; crypto, lending, and "regulated financial products" may require regional **declarations and proof of licensing** in the console.
- **In-app account deletion** + a deletion-request URL: Play also requires an account/data deletion path, including one reachable **outside** the app.
- **Families policy:** only relevant if your app targets children — almost never for a bank; do not opt into the Designed for Families program.

### 21.6 Store listing & assets **[I]**

Both stores block submission until the listing metadata is complete. Prepare these up front; localized variants multiply the work.

| Asset / field | iOS (App Store Connect) | Android (Play Console) |
|---|---|---|
| App icon | 1024×1024 (no alpha) | 512×512 + adaptive icon in build |
| Screenshots | Per device class: 6.9"/6.5" iPhone, 13" iPad if supported | Phone + 7"/10" tablet; 2–8 each |
| Promo media | App Preview video (optional) | Feature graphic 1024×500 (required), video optional |
| Text | Name, subtitle, description, **keywords** (100 chars), promo text | Title (30), short desc (80), full desc (4000) |
| **Privacy policy URL** | **Required** | **Required** |
| Support URL / contact | Required | Email required, website optional |
| Age rating | Apple questionnaire | IARC questionnaire |
| Category | Finance | Finance |

Localization: at minimum localize the listing for each market you operate in; regulators may require local-language disclosures. Keep screenshots free of real customer data — use seeded demo data.

### 21.7 Banking / fintech-specific review notes **[A]**

Finance apps get **manual, heightened scrutiny** on both stores. Plan for a slower, pickier review.

- **Provide a reviewer demo account.** Apple's App Review and Google's reviewers cannot pass your KYC/2FA, so give working test credentials (and any OTP bypass for the demo account) in the review notes. Missing/blocked demo access is the #1 banking-app rejection. If you use App Attest/Play Integrity, ensure the demo device/build passes.
- **Justify data handling clearly.** Explain in review notes why you collect financial data and how it's secured (TLS, keychain/Keystore via §17, no plaintext logging via §20). Reviewers may ask.
- **No misleading financial claims.** Avoid guaranteed returns, unverifiable "FDIC/regulator-backed" wording, or implying you're a bank if you're a fintech front-end. Show required disclosures/disclaimers.
- **Regional licensing.** Some regions require proof you're authorized to offer financial services; both consoles have fields/processes for this. Engage compliance early — these can add weeks.
- **Common rejection reasons & avoidance:**

| Rejection reason | How to avoid |
|---|---|
| Reviewer can't log in | Supply demo account + OTP path in notes |
| Privacy labels ≠ Data Safety ≠ manifest | Single source of truth; reconcile all three |
| Missing in-app account deletion | Build the in-app deletion flow (§21.4/21.5) |
| Social login without Sign in with Apple | Add Sign in with Apple |
| Undeclared/over-broad permissions | Request minimum; justify sensitive ones |
| Misleading financial claims | Legal review of all copy |
| Crash on reviewer device/region | Test on real devices, multiple locales/timezones |

### 21.8 Release process, OTA updates & post-launch **[A]**

Promote builds through a funnel, never straight to 100%:

1. **Internal/TestFlight + Play internal** → QA, security, demo rehearsal.
2. **External beta / closed track** → UAT and a pilot cohort.
3. **Submit to review** with reviewer notes + demo account (§21.7).
4. **Staged/phased rollout** to production (below).
5. **Monitor**, then ramp to 100% or halt.

**Staged rollout** is your safety valve and differs per platform:

| Aspect | iOS (Phased Release) | Android (Staged Rollout) |
|---|---|---|
| Granularity | Fixed 7-day curve (1→2→5→10→20→50→100%) | You choose % (e.g. 1/5/10/20/50/100) |
| Pause | Yes | Yes |
| **Halt + fix** | Can pause; can't reduce % already served | Can **halt** rollout at current % |
| Trigger | Auto over 7 days | Manual or rules |

**EAS Update (OTA) vs store resubmit.** This is the most important post-launch lever. EAS Update (§14) ships **JS/asset-only** changes over the air to a channel — instant hotfixes for bugs or content with no review wait. But anything touching **native code** requires a fresh store build and review:

| Change type | Ship via EAS Update (OTA)? | Resubmit to stores? |
|---|---|---|
| JS bug fix, copy/UI tweak, config | Yes | No |
| New JS feature (no native deps) | Yes | No |
| Add/upgrade a native module, new permission/entitlement | No | **Yes** |
| SDK upgrade / changed app.config native fields | No | **Yes** |
| `runtimeVersion` change | No (incompatible) | **Yes** |

OTA payloads only apply when the **`runtimeVersion` matches** the installed binary — that's the guardrail preventing JS that needs new native code from reaching an old binary. For a bank, gate sensitive OTA hotfixes and keep them auditable; never push unreviewed code that changes money movement without your normal change control.

**Rollback strategy:**
- *OTA:* `eas update --branch production` republishing the previous known-good update, or roll back the channel pointer — users recover on next launch.
- *Native:* you cannot un-ship a binary. **Halt the Android staged rollout** and/or pause iOS phased release the moment crash-free rate dips, then submit a fixed build. This is why you stage.

**Monitoring & post-launch:**
- Track **crash-free users/sessions** (Sentry/Crashlytics) and **adoption** per version; halt if a new version regresses.
- Watch auth-success and transaction-success rates per build — for a bank these matter more than generic crash metrics.
- After full rollout, bump the marketing `version` for the next cycle (build numbers auto-increment, §21.1), and keep the release branch tagged for hotfix forks.
- **Handling rejections:** read the resolution center message, fix or clarify in the reply, and resubmit; for policy disputes use Apple's App Review Board / Google's appeal flow with evidence.

> **Production release checklist (iOS + Android):**
> **Structure & versioning** — [ ] dev/staging/prod via `app.config.ts` + EAS profiles; [ ] distinct bundle IDs/packages per variant (staging installable beside prod); [ ] `appVersionSource: remote` + `autoIncrement`; [ ] marketing `version` bumped; [ ] release branch tagged; [ ] secrets via EAS, not `EXPO_PUBLIC_*`.
> **iOS signing** — [ ] Apple Developer org enrolled; [ ] App ID + capabilities (push, associated domains/passkeys, App Attest, Sign in with Apple if social); [ ] EAS-managed (or vaulted manual) distribution cert + profile; [ ] App Store Connect record; [ ] TestFlight tested.
> **Android signing** — [ ] Google Play App Signing enrolled; [ ] upload keystore vaulted; [ ] **AAB** built; [ ] target **API 35**; [ ] internal/closed track tested.
> **iOS compliance** — [ ] `PrivacyInfo.xcprivacy` audited (data types + required-reason APIs); [ ] App Privacy labels filled; [ ] ATT handled or "no tracking" declared; [ ] Sign in with Apple if applicable; [ ] in-app account deletion; [ ] `ITSAppUsesNonExemptEncryption=false` (standard crypto).
> **Android compliance** — [ ] Data Safety form matches reality + privacy policy; [ ] sensitive permissions justified; [ ] financial-products policy/licensing declared; [ ] in-app + external account deletion.
> **Listing** — [ ] icon, screenshots per device size, feature graphic (Android); [ ] description/keywords; [ ] **privacy-policy URL**; [ ] support URL; [ ] age rating; [ ] Finance category; [ ] localized where required; [ ] no real customer data in assets.
> **Fintech review** — [ ] reviewer demo account + OTP path in notes; [ ] data-handling explanation; [ ] no misleading financial claims; [ ] regional licensing evidence; [ ] App Attest/Play Integrity passes for the demo build.
> **Rollout & post-launch** — [ ] phased (iOS) / staged % (Android) rollout; [ ] crash-free + auth/txn success monitored; [ ] OTA (§14) reserved for JS-only fixes with `runtimeVersion` match; [ ] native fixes require resubmit; [ ] rollback plan (republish OTA / halt staged rollout) documented.

---

## 22. Tips, Tricks & Gotchas

### 17.1 Platform Differences: iOS vs Android **[I]**

| Behavior | iOS | Android |
|---|---|---|
| Shadows | `shadowColor/Offset/Opacity/Radius` | `elevation` (number) |
| Status bar | Overlaps content (use safe-area) | Usually pushes content down |
| System back | Edge-swipe gesture | Hardware/gesture back button (handle `onRequestClose`/`BackHandler`) |
| Keyboard handling | `KeyboardAvoidingView` `padding` | `windowSoftInputMode` often suffices |
| Date/time picker | Wheel spinner | Calendar/clock dialog |
| Permission dialog | "Allow Once / While Using / Don't Allow" | "Allow / Deny" |
| Ripple feedback | None (use opacity) | Material ripple (`android_ripple`) |
| Elevation/overflow | `overflow: hidden` reliable | `overflow: hidden` can be inconsistent |

```tsx
import { Platform } from 'react-native';
const hitSlop = Platform.select({
  ios: { top: 10, bottom: 10, left: 10, right: 10 },
  android: { top: 5, bottom: 5, left: 5, right: 5 },
  default: {},
});
```

> Cross-reference: when a native behavior differs, `ANDROID_STUDIO_GUIDE.md` shows what Android does *under the hood* (e.g. `windowSoftInputMode`, ripples, elevation) — useful for understanding *why* RN behaves as it does.

### 17.2 Expo Go Limitations — When You Need a Dev Build **[I]**

Switch from Expo Go to a development build the moment any of these is true:
- You install an npm package with native code that's **not** in the Expo SDK.
- You use `react-native-maps`, `react-native-firebase`, Stripe, or any third-party native module.
- You need push tokens on a real device (token generation requires your own bundle ID).
- You need to test custom config plugins or your own native module.
- You use a feature gated behind a dev build (e.g. some camera/barcode features in recent SDKs).

```bash
eas build --profile development --platform ios   # build once
# install the .ipa/.apk, then:
npx expo start --dev-client
```

### 17.3 New Architecture (Fabric + TurboModules) **[A]**

> **⚡ Version note:** Enabled by default in SDK 52+. Recap of what each piece does (full explanation in §1.3):
- **Fabric** — synchronous C++ renderer; smoother UI, Concurrent React on native.
- **TurboModules** — lazily-loaded native modules; faster startup.
- **JSI** — direct C++ binding; no JSON bridge serialization.
- **Concurrent React (React 19)** — `useTransition`, `useDeferredValue`, Suspense work on native (see `REACT_19_GUIDE.md`).

```json
// app.json — only disable as a last resort for an incompatible legacy library:
{ "expo": { "newArchEnabled": false } }
```

> Check library compatibility at `reactnative.directory` (filter "New Architecture"). Prefer replacing an incompatible library over disabling the New Architecture.

### 17.4 Safe Areas — Always **[B]**

```tsx
import { useSafeAreaInsets } from 'react-native-safe-area-context';
function Header() {
  const insets = useSafeAreaInsets();
  return <View style={{ paddingTop: insets.top, backgroundColor: '#fff' }}>{/* title */}</View>;
}
```

> Never hardcode `paddingTop: 44`. Notch/Dynamic Island/home-indicator sizes vary by device; the insets adapt automatically.

### 17.5 Images — Common Gotchas **[B]**

```tsx
// 1) Local images need require() — Metro must SEE the require to bundle the asset:
<Image source={require('./assets/photo.png')} />;     // ✅
<Image source={{ uri: './assets/photo.png' }} />;     // ❌ silently broken
// 2) Remote images have no intrinsic size — always give width/height:
<Image source={{ uri: 'https://…' }} style={{ width: 200, height: 200 }} />;
// 3) HTTP (not HTTPS) remote images need cleartext config on Android — just use HTTPS.
// 4) Use expo-image for remote images (caching, placeholders, transitions) — §5.4.
```

### 17.6 TypeScript Tips for RN **[I]**

```tsx
import { ViewStyle, TextStyle, StyleProp } from 'react-native';
// Style props: accept arrays/conditionals via StyleProp<…>:
type ButtonProps = {
  title: string;
  onPress: () => void;
  style?: StyleProp<ViewStyle>;
  textStyle?: StyleProp<TextStyle>;
  disabled?: boolean;
};
// Generic FlatList for typed items:
type ListItem = { id: string; name: string };
<FlatList<ListItem>
  data={items}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <Text>{item.name}</Text>}
/>;
```

> Enable typed routes (§4.7) so navigation is type-checked too. React 19's `ref`-as-a-prop change means many `forwardRef` wrappers are no longer needed — see `REACT_19_GUIDE.md`.

### 17.7 Performance Checklist **[A]**

1. Use `FlashList` for big lists; memoize rows; stable `keyExtractor` (§6).
2. Run animations on the UI thread with **Reanimated** (§12), not JS-thread `setState` loops.
3. Use `expo-image` with `cachePolicy` for remote images.
4. Avoid creating new objects/functions inline in hot render paths (defeats memoization).
5. Defer non-critical work after first paint; preload behind the splash screen (§13.5).
6. Profile with the Performance monitor (Dev Menu) and React DevTools Profiler.
7. Keep the JS thread free — heavy computation can move to a worklet or be chunked.

### 17.8 Common Gotchas Checklist **[I]**

- **`flex: 1` not working?** The parent must also be sized (often `flex: 1`). Flex bubbles up from a sized root.
- **Styles not updating?** Clear Metro cache: `npx expo start --clear`.
- **Android text cut off?** Don't set a fixed `height` on `<Text>`; use container padding and `lineHeight`.
- **Tap target too small?** Add `hitSlop` to `Pressable`.
- **`overflow: hidden` won't clip on Android?** Put `borderRadius` on the child too; Android clipping is inconsistent.
- **Images blurry on high-DPI?** Provide `@2x`/`@3x` variants for local images (Metro auto-selects).
- **`onPress` fires twice?** You nested two touchables (e.g. `Pressable` inside `TouchableOpacity`).
- **Crashes on Android but not iOS?** Check `elevation`, `useNativeDriver`, and New-Architecture compatibility of native libs.
- **Gestures/animations do nothing?** Missing `GestureHandlerRootView` wrapper, or the Reanimated Babel plugin isn't last.

---

## 23. Study Path & Build-to-Learn Projects

An ordered path from zero to shipping a real app. The best learning is *building* — each stage ends with a concrete project.

### Stage 1 — Foundations (Week 1–2) **[B]**
**Learn:** TypeScript basics; React fundamentals (components, props, state, hooks, context — `REACT_19_GUIDE.md`); how mobile differs from web (no DOM, native widgets, sandboxed, offline-first, the JSI/New-Architecture model from §1).
**Build:** a counter app and a static "about me" profile screen. No navigation yet.

### Stage 2 — Expo Basics (Week 2–3) **[B]**
**Learn:** `npx create-expo-app`, project structure (§3), Metro; core components (View, Text, StyleSheet, Flexbox column-first!, Image, Pressable, ScrollView); Expo Go for quick testing; FlatList basics.
**Build:** a recipe browser with a static list, images, and a detail screen via `router.push`.

### Stage 3 — Expo Router & Navigation (Week 3–4) **[B/I]**
**Learn:** file conventions (`_layout.tsx`, `[id].tsx`, `(groups)`); Stack push/pop + header options; Tabs with `@expo/vector-icons`; params via `useLocalSearchParams`; the auth-guard pattern.
**Build:** a multi-tab notes app — home (list), add (form), tapping a note pushes to detail.

### Stage 4 — Data & APIs (Week 4–5) **[I]**
**Learn:** `fetch` + async/await + error/loading states (§9.2); TanStack Query (`TANSTACK_QUERY_GUIDE.md`); AsyncStorage; SecureStore for tokens.
**Build:** connect the notes app to a real backend (Supabase is free — `SUPABASE_GUIDE.md`). Add login/logout with token storage and an auth guard.

### Stage 5 — Native Device Features (Week 5–6) **[I]**
**Learn:** the permissions flow (check → request → handle denial); `expo-image-picker`, `expo-location`, `expo-notifications` (local + push), `expo-haptics`.
**Build:** let users attach photos to notes (image picker); schedule a reminder local notification.

### Stage 6 — Styling & UX Polish (Week 6–7) **[I/A]**
**Learn:** `useWindowDimensions` responsiveness; `Platform.select`; safe-area insets everywhere; **Reanimated** + **gesture-handler** (§12).
**Build:** swipe-to-delete on the notes list with a fade-out animation; a spring scale on the "Add" button.

### Stage 7 — Build & Ship (Week 7–8) **[A]**
**Learn:** EAS Build profiles (dev/preview/production); creating a dev build; EAS Update (OTA) and `runtimeVersion`; store submission.
**Build:** ship to TestFlight (iOS) and/or Play internal testing; push a JS-only fix via EAS Update.

### Projects to Build (Progressive Complexity)

| Project | Key concepts practiced |
|---|---|
| Counter / flashcards | State, StyleSheet, Flexbox |
| Weather app | fetch, loading/error states, FlatList, Dimensions |
| Recipe browser | Expo Router, Stack, image, dynamic routes |
| Notes app (local) | CRUD, AsyncStorage, forms, React Hook Form |
| Notes app (backend) | TanStack Query, auth, SecureStore, mutations |
| Photo journal | Image picker, expo-image, camera, file system |
| Fitness tracker | Accelerometer, location, notifications, charts |
| Chat app | WebSockets / Supabase Realtime, inverted FlatList |
| Full SaaS mobile app | Everything above + EAS Build/Update + store submission |

### Recommended Resources (for when you have internet)

- **Expo Docs:** docs.expo.dev (authoritative)
- **React Native Docs:** reactnative.dev/docs
- **Expo Router:** docs.expo.dev/router
- **EAS:** docs.expo.dev/eas
- **React Native Directory:** reactnative.directory (filter by New Architecture)
- **TanStack Query:** tanstack.com/query/latest
- **Reanimated:** docs.swmansion.com/react-native-reanimated
- **NativeWind:** nativewind.dev
- **Maestro:** maestro.mobile.dev

### Quick Reference: Most-Used Commands

```bash
# Create & run
npx create-expo-app@latest MyApp
npx expo start
npx expo start --clear                 # clear Metro cache (fixes most weirdness)
npx expo start --dev-client            # run against a development build
npx expo run:ios                       # build & run on iOS simulator
npx expo run:android                   # build & run on Android emulator
# Install native packages (ALWAYS expo install, not npm install, for SDK packages)
npx expo install expo-image expo-camera expo-location
# EAS
eas login
eas init
eas build --profile development --platform ios
eas build --profile production --platform all
eas submit --platform ios --profile production
eas update --channel production --message "Bug fix"
# Debugging
# Press 'j' in Metro → React Native DevTools
# Cmd+D (iOS sim) / Cmd+M (Android emu) / shake device → Dev Menu
```

---

*This guide targets Expo SDK 54 (52/53 nearly identical), React Native 0.81+ (New Architecture enabled by default), and React 19, current in 2026. Compare against `ANDROID_STUDIO_GUIDE.md` (native Kotlin/Compose) and `REACT_19_GUIDE.md` (the React layer underneath). For the latest API changes always consult docs.expo.dev.*
