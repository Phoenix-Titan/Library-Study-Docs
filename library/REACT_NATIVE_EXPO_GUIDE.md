# React Native + Expo — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I know some React on the web" (or even "I'm new to React") to "I can design, build, debug, and ship a production iOS + Android app with Expo" — without an internet connection. Every concept comes with prose that explains *what it is*, *the logic / why it works this way*, *what it's for and when you reach for it*, *how to use it*, the *key props/parameters*, *best practices*, and *gotchas* — followed by runnable, heavily-commented code. Read top-to-bottom the first time; afterward use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Expo SDK 52 / 53** (current in 2026), **React Native 0.77+** with the **New Architecture** (Fabric + TurboModules + JSI) enabled by default, and **React 19**. Things worth knowing about the modern stack:
> - **New Architecture is the default** in SDK 52+. The legacy bridge is gone for new projects; UI and native calls go through JSI (direct C++ binding). Covered in §1 and §18.
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
17. [Tips, Tricks & Gotchas](#17-tips-tricks--gotchas) **[I/A]**
18. [Study Path & Build-to-Learn Projects](#18-study-path--build-to-learn-projects)

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

**Why you care:** mostly you don't have to *do* anything — it's on by default. Two practical consequences: (1) some old native libraries aren't yet New-Architecture-compatible — check `reactnative.directory`, filter "New Architecture" (disabling is a last resort, §17); (2) Reanimated UI-thread animations (§12) stay smooth even when JS is busy, since they no longer cross a JSON bridge.

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

> **⚡ Version note:** **NativeWind v4** is stable in 2026 and works with Expo SDK 52/53. It lets you use Tailwind utility classes via a `className` prop; at build time it compiles them to RN `StyleSheet` objects (no runtime CSS). If you know Tailwind on the web (`TAILWIND_CHEATSHEET.md`), the classes are the same.

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

## 17. Tips, Tricks & Gotchas

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

## 18. Study Path & Build-to-Learn Projects

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

*This guide targets Expo SDK 52/53, React Native 0.77+ (New Architecture enabled by default), and React 19, current in 2026. Compare against `ANDROID_STUDIO_GUIDE.md` (native Kotlin/Compose) and `REACT_19_GUIDE.md` (the React layer underneath). For the latest API changes always consult docs.expo.dev.*
