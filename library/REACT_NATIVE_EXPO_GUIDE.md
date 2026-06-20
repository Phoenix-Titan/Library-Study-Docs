# React Native + Expo Complete Guide (2026 Edition)

A full, study-friendly offline reference for **React Native with Expo** — covering project setup, Expo Router (file-based routing), core components, native APIs, EAS Build/Submit/Update, animations, and everything you need to ship real iOS and Android apps in 2026.

> **What it is / Why it matters:** React Native lets you write mobile apps in JavaScript/TypeScript using React. Expo is the toolchain and SDK layer on top — it handles native build complexity, provides a rich SDK of device APIs, and ships **Expo Router** (the file-based routing system modeled on Next.js). In 2026 Expo is the **recommended starting point** for almost every new React Native project because it gives you managed builds (EAS), over-the-air updates, and the full New Architecture (Fabric + TurboModules) by default.

> **⚡ Version note:** This guide targets **Expo SDK 53** (released early 2026) and React Native **0.77+** with the New Architecture enabled by default. Where an API or behavior is version-specific it is flagged with ⚡.

---

## Table of Contents
1. [React Native + Expo — The Big Picture](#1-react-native--expo--the-big-picture)
2. [Setup: Create, Run & Tooling](#2-setup-create-run--tooling)
3. [Project Structure & the `app/` Directory](#3-project-structure--the-app-directory)
4. [Expo Router — File-Based Routing](#4-expo-router--file-based-routing)
5. [Core Components](#5-core-components)
6. [Lists & Performance: FlatList vs FlashList](#6-lists--performance-flatlist-vs-flashlist)
7. [Styling: StyleSheet, Flexbox, Responsive & NativeWind](#7-styling-stylesheet-flexbox-responsive--nativewind)
8. [Input, Forms & Keyboard Handling](#8-input-forms--keyboard-handling)
9. [State & Data Fetching](#9-state--data-fetching)
10. [Expo SDK — Device & Native APIs](#10-expo-sdk--device--native-apis)
11. [Navigation Patterns & Params](#11-navigation-patterns--params)
12. [Animations & Gestures](#12-animations--gestures)
13. [Configuration: app.json / app.config.ts](#13-configuration-appjson--appconfigts)
14. [Building & Shipping with EAS](#14-building--shipping-with-eas)
15. [Debugging](#15-debugging)
16. [Tips, Tricks & Gotchas](#16-tips-tricks--gotchas)
17. [Study Path](#17-study-path)

---

## 1. React Native + Expo — The Big Picture

### What React Native Is

React Native compiles your TypeScript/JavaScript components into **real native UI elements** — not a WebView. A `<View>` becomes a `UIView` on iOS and an `android.view.View` on Android. Layout is handled by Yoga (a cross-platform Flexbox engine). Business logic runs in a JavaScript engine (Hermes by default).

```
Your TypeScript/TSX Code
        │
        ▼
   Metro Bundler (JS bundle)
        │
        ▼
  Hermes JS Engine (runs on device)
        │  ←── JSI Bridge (New Architecture: direct C++ calls, no JSON serialization)
        ▼
  Native Modules / Fabric Renderer
        │
        ▼
  iOS UIKit / Android Views (real native UI)
```

### Expo vs Bare React Native

| | Managed Expo (default) | Dev Build (Expo + custom native) | Bare React Native |
|---|---|---|---|
| Setup | `npx create-expo-app` | `npx expo prebuild` | `npx react-native init` |
| Native code visible | No | Yes (after prebuild) | Yes |
| Add native modules | Via Expo SDK + config plugins | Full freedom | Full freedom |
| EAS Build | Yes | Yes | Yes (with config) |
| Expo Go testing | Yes (limited) | No — needs dev build | No |
| Recommended in 2026 | **Yes (start here)** | When you need custom native | Rarely |

> **Why Expo in 2026:** Config plugins let you modify native code without ever opening Xcode or Android Studio for most use-cases. EAS handles cloud builds. Expo Router gives you a Next.js-like DX. The New Architecture ships enabled by default.

### Expo Go vs Development Build

- **Expo Go** — a pre-built shell app on the App Store / Play Store. Scan a QR code and your app runs inside it. Great for learning, limited to Expo SDK APIs only. No custom native modules.
- **Development Build** — a custom version of Expo Go that includes your specific native dependencies. This is the modern recommended workflow. Built with EAS Build or locally via `npx expo run:ios`.

> **⚡ Version note:** In SDK 53, `npx create-expo-app` creates a project that uses Expo Router by default. The `app/` directory is the entry point — no `App.tsx` at the root anymore.

---

## 2. Setup: Create, Run & Tooling

### Prerequisites

```bash
# Node.js 20+ (LTS)
node --version   # should be 20.x or 22.x

# Install Expo CLI globally (optional — npx works fine)
npm install -g expo-cli eas-cli

# iOS: Xcode 16+ (Mac only) with Command Line Tools
# Android: Android Studio with Android SDK, JAVA_HOME set
```

### Create a New Project

```bash
# Default — creates an Expo Router project with TypeScript
npx create-expo-app@latest MyApp

# With a specific template
npx create-expo-app@latest MyApp --template blank-typescript   # minimal, no Router
npx create-expo-app@latest MyApp --template tabs               # tabs + Expo Router (default)
```

### Running the App

```bash
cd MyApp

# Start Metro bundler
npx expo start

# Press 'i' for iOS simulator, 'a' for Android emulator, scan QR for Expo Go on device
# Press 's' to switch between Expo Go and dev build modes

# Run directly on a simulator (triggers a local build if needed)
npx expo run:ios
npx expo run:android

# Clear Metro cache (fixes most mysterious bugs)
npx expo start --clear
```

### Metro Bundler

Metro is the JavaScript bundler for React Native (like Webpack but for mobile). It:
- Watches your files and sends updates to the device via **Fast Refresh** (hot reload)
- Bundles JS, images, and assets
- Resolves platform-specific files (`Button.ios.tsx` over `Button.tsx` on iOS)

```
metro.config.js  — extend Metro behavior (custom resolvers, transforms)
```

```js
// metro.config.js — example: enable CSS modules or custom asset extensions
const { getDefaultConfig } = require('expo/metro-config');
const config = getDefaultConfig(__dirname);

// Add SVG support
config.transformer.babelTransformerPath = require.resolve('react-native-svg-transformer');
config.resolver.assetExts = config.resolver.assetExts.filter(ext => ext !== 'svg');
config.resolver.sourceExts = [...config.resolver.sourceExts, 'svg'];

module.exports = config;
```

### iOS Simulator (Mac Only)

```bash
# List available simulators
xcrun simctl list devices

# Open a specific simulator, then run your app
npx expo run:ios --device "iPhone 16 Pro"
```

### Android Emulator

```bash
# Start an AVD from Android Studio's Device Manager, then:
npx expo run:android

# Or specify a device ID
adb devices
npx expo run:android --device emulator-5554
```

---

## 3. Project Structure & the `app/` Directory

```
MyApp/
├── app/                        # ←★ Expo Router file-based routes
│   ├── _layout.tsx             # Root layout (required) — wraps everything
│   ├── index.tsx               # "/" home screen
│   ├── +not-found.tsx          # 404 / unmatched routes
│   │
│   ├── (tabs)/                 # Route GROUP for tab navigator
│   │   ├── _layout.tsx         # Defines the <Tabs> navigator
│   │   ├── index.tsx           # First tab
│   │   └── explore.tsx         # Second tab
│   │
│   ├── details/
│   │   └── [id].tsx            # Dynamic route → /details/42
│   │
│   └── modal.tsx               # Can be presented as a modal
│
├── components/                 # Reusable components (not routes)
├── hooks/                      # Custom hooks
├── constants/                  # Colors, spacing tokens, etc.
├── assets/                     # Images, fonts, icons
│   ├── images/
│   └── fonts/
│
├── app.json                    # Expo project config
├── app.config.ts               # Dynamic config (optional, overrides app.json)
├── eas.json                    # EAS Build / Submit / Update config
├── tsconfig.json
├── metro.config.js
└── package.json
```

**Key rules for the `app/` directory:**
- Every `.tsx` / `.jsx` file in `app/` becomes a **route** (the filename = URL segment).
- `_layout.tsx` files define navigators (Stack, Tabs, Drawer).
- Files/folders starting with `_` or `+` are **not** treated as routes (`+not-found.tsx`, `+html.tsx`).
- Folders wrapped in `(parentheses)` are **route groups** — they organize files but are excluded from the URL path.

---

## 4. Expo Router — File-Based Routing

Expo Router brings Next.js-style file-based routing to React Native. It's built on top of React Navigation under the hood.

### Root Layout (`app/_layout.tsx`)

Every app needs a root layout. It sets up global providers, fonts, and the root navigator.

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';
import { useFonts } from 'expo-font';
import * as SplashScreen from 'expo-splash-screen';
import { useEffect } from 'react';
import { StatusBar } from 'expo-status-bar';

// Prevent splash screen from auto-hiding before fonts load
SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const [loaded] = useFonts({
    SpaceMono: require('../assets/fonts/SpaceMono-Regular.ttf'),
  });

  useEffect(() => {
    if (loaded) SplashScreen.hideAsync();
  }, [loaded]);

  if (!loaded) return null;

  return (
    <>
      <StatusBar style="auto" />
      {/* Stack navigator — each screen slides in from the right by default */}
      <Stack>
        <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
        <Stack.Screen name="+not-found" />
        {/* Present modal.tsx as a modal */}
        <Stack.Screen name="modal" options={{ presentation: 'modal' }} />
      </Stack>
    </>
  );
}
```

### Stack Navigator (`<Stack>`)

```tsx
// A Stack screen with custom header options
// app/details/[id].tsx
import { Stack, useLocalSearchParams } from 'expo-router';

export default function DetailsScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();

  return (
    <>
      {/* Override header for this screen specifically */}
      <Stack.Screen options={{ title: `Item #${id}` }} />
      {/* ... screen content */}
    </>
  );
}
```

### Tab Navigator (`<Tabs>`)

```tsx
// app/(tabs)/_layout.tsx
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function TabLayout() {
  return (
    <Tabs
      screenOptions={{
        tabBarActiveTintColor: '#007AFF',
        headerShown: false,
      }}
    >
      <Tabs.Screen
        name="index"
        options={{
          title: 'Home',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="home" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="explore"
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

### Dynamic Routes `[id]`

```
app/
└── posts/
    ├── index.tsx        →  /posts
    └── [id].tsx         →  /posts/123  (id = "123")

app/
└── docs/
    └── [...slug].tsx    →  /docs/a/b/c  (slug = ["a", "b", "c"])
```

```tsx
// app/posts/[id].tsx
import { useLocalSearchParams } from 'expo-router';
import { Text, View } from 'react-native';

export default function PostScreen() {
  // useLocalSearchParams reads the URL segment AND any query params
  const { id } = useLocalSearchParams<{ id: string }>();

  return (
    <View>
      <Text>Post ID: {id}</Text>
    </View>
  );
}
```

### Navigation: `<Link>` and `router`

```tsx
import { Link, router } from 'expo-router';
import { Pressable, Text } from 'react-native';

// Declarative — like <a href>
<Link href="/posts/42">
  <Text>Go to Post 42</Text>
</Link>

// With query params
<Link href={{ pathname: '/posts/[id]', params: { id: '42' } }}>
  <Text>Post 42</Text>
</Link>

// Imperative — use inside event handlers
function handlePress() {
  router.push('/posts/42');                          // push onto stack
  router.replace('/login');                          // replace current screen (no back)
  router.back();                                     // go back
  router.push({ pathname: '/posts/[id]', params: { id: '42' } });
}

// Present as modal
router.push('/modal');
```

### Route Groups

Route groups use `(groupName)` folder syntax. The group name is **not** part of the URL.

```
app/
├── (auth)/
│   ├── _layout.tsx     # Auth-specific layout (no tabs/header)
│   ├── login.tsx       # → /login
│   └── register.tsx    # → /register
│
└── (app)/
    ├── _layout.tsx     # Main app layout (with tabs)
    └── (tabs)/
        └── home.tsx    # → /home
```

This pattern lets you have completely different navigators/layouts for authenticated vs. unauthenticated users without affecting URLs.

### Typed Routes

> **⚡ Version note:** Enable typed routes in `app.json` for TypeScript autocompletion on `href` values — prevents typos in route strings.

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
// Now <Link href="/posts/42"> gives a TS error if /posts/[id] doesn't exist
import { Link } from 'expo-router';
// href is typed — autocomplete works!
<Link href="/settings">Settings</Link>
```

---

## 5. Core Components

### View

The fundamental building block — equivalent to a `<div>`. Always use `View` for layout containers.

```tsx
import { View, StyleSheet } from 'react-native';

export default function Card() {
  return (
    <View style={styles.card}>
      {/* children */}
    </View>
  );
}

const styles = StyleSheet.create({
  card: {
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 16,
    // Flexbox is the only layout system — see Section 7
    flexDirection: 'row',
    alignItems: 'center',
    // Shadow — platform-specific!
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,   // Android shadow
  },
});
```

### Text

All text **must** be inside `<Text>`. You cannot have raw strings directly in `<View>`.

```tsx
import { Text, StyleSheet } from 'react-native';

<Text
  style={styles.heading}
  numberOfLines={2}          // truncate after 2 lines
  ellipsizeMode="tail"       // add "..." at end
  selectable                 // allow user to copy text
  onPress={() => {}}         // text can be tappable
>
  Hello World
</Text>

// Nested Text for mixed styles (like <span> inside <p>)
<Text style={styles.body}>
  This is <Text style={{ fontWeight: 'bold' }}>bold</Text> text.
</Text>
```

### ScrollView

For scrollable content that fits in memory (small lists). For long lists use FlatList.

```tsx
import { ScrollView, View, Text } from 'react-native';

<ScrollView
  style={{ flex: 1 }}
  contentContainerStyle={{ padding: 16 }}   // padding for scrollable content
  showsVerticalScrollIndicator={false}
  horizontal={false}
  keyboardShouldPersistTaps="handled"        // taps dismiss keyboard
  refreshControl={                            // pull-to-refresh
    <RefreshControl refreshing={refreshing} onRefresh={onRefresh} />
  }
>
  {items.map(item => <Text key={item.id}>{item.name}</Text>)}
</ScrollView>
```

### Image (expo-image)

> **⚡ Version note:** Use `expo-image` instead of the built-in `<Image>` — it has better caching, blurhash placeholders, and performance.

```bash
npx expo install expo-image
```

```tsx
import { Image } from 'expo-image';

<Image
  source={{ uri: 'https://example.com/photo.jpg' }}
  // OR local: source={require('../assets/photo.png')}
  style={{ width: 200, height: 200, borderRadius: 100 }}
  contentFit="cover"                          // like CSS object-fit
  transition={300}                            // fade-in animation (ms)
  placeholder={{ blurhash: 'L6PZfSi_.AyE_3t7t7R**0o#DgR4' }}
  cachePolicy="memory-disk"                   // aggressive caching
  priority="high"
/>
```

### Pressable

The modern replacement for `TouchableOpacity`. Gives you fine-grained press state control.

```tsx
import { Pressable, Text, StyleSheet } from 'react-native';

<Pressable
  onPress={() => console.log('pressed')}
  onLongPress={() => console.log('long press')}
  // Dynamic style based on press state
  style={({ pressed }) => [
    styles.button,
    pressed && styles.buttonPressed,
  ]}
  // Expand the hit area beyond the visual bounds
  hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
>
  {({ pressed }) => (
    <Text style={[styles.label, pressed && { opacity: 0.7 }]}>
      Press Me
    </Text>
  )}
</Pressable>

const styles = StyleSheet.create({
  button: { backgroundColor: '#007AFF', borderRadius: 8, padding: 12 },
  buttonPressed: { backgroundColor: '#005EC4' },
  label: { color: '#fff', fontWeight: '600', textAlign: 'center' },
});
```

### TouchableOpacity

Simpler than Pressable — dims on press. Still widely used.

```tsx
import { TouchableOpacity, Text } from 'react-native';

<TouchableOpacity
  onPress={handlePress}
  activeOpacity={0.7}   // 0 = fully transparent when pressed
>
  <Text>Tap me</Text>
</TouchableOpacity>
```

### TextInput

```tsx
import { TextInput, StyleSheet } from 'react-native';
import { useState } from 'react';

function SearchBar() {
  const [value, setValue] = useState('');

  return (
    <TextInput
      value={value}
      onChangeText={setValue}                   // called on every keystroke
      placeholder="Search..."
      placeholderTextColor="#999"
      style={styles.input}
      // Keyboard config
      keyboardType="email-address"              // or "numeric", "phone-pad", "url"
      returnKeyType="search"                    // keyboard return button label
      autoCapitalize="none"
      autoCorrect={false}
      secureTextEntry={false}                   // true for passwords
      // Events
      onSubmitEditing={({ nativeEvent }) => handleSearch(nativeEvent.text)}
      onFocus={() => console.log('focused')}
      onBlur={() => console.log('blurred')}
      // Multi-line
      multiline
      numberOfLines={4}
      // Control focus imperatically
      ref={inputRef}
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
    color: '#000',
  },
});
```

### SafeAreaView

Prevents content from going under the notch, status bar, or home indicator.

```tsx
import { SafeAreaView, SafeAreaProvider } from 'react-native-safe-area-context';

// In root layout — wrap your whole app
import { SafeAreaProvider } from 'react-native-safe-area-context';

export default function RootLayout() {
  return (
    <SafeAreaProvider>
      {/* navigator / screens */}
    </SafeAreaProvider>
  );
}

// In individual screens
import { SafeAreaView } from 'react-native-safe-area-context';

export default function HomeScreen() {
  return (
    <SafeAreaView style={{ flex: 1 }} edges={['top', 'bottom']}>
      {/* content is safe */}
    </SafeAreaView>
  );
}
```

> **Gotcha:** Never use the built-in `SafeAreaView` from `react-native` — it doesn't handle all devices correctly. Always use `react-native-safe-area-context`.

### ActivityIndicator

```tsx
import { ActivityIndicator, View } from 'react-native';

// Show a loading spinner
{isLoading && (
  <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
    <ActivityIndicator size="large" color="#007AFF" />
  </View>
)}
```

### Modal

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
        transparent                         // background shows through
        animationType="slide"               // "fade" | "slide" | "none"
        onRequestClose={() => setVisible(false)}   // Android back button
        statusBarTranslucent                // Android: modal over status bar
      >
        <View style={{ flex: 1, justifyContent: 'flex-end' }}>
          <View style={{ backgroundColor: '#fff', borderRadius: 20, padding: 20 }}>
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

> **Tip:** For sheets and modals, prefer Expo Router's `presentation: 'modal'` or a library like `@gorhom/bottom-sheet` over bare `<Modal>`.

---

## 6. Lists & Performance: FlatList vs FlashList

### FlatList

Built into React Native. Virtualizes the list (only renders visible items). Use for most lists.

```tsx
import { FlatList, Text, View, StyleSheet } from 'react-native';

type Item = { id: string; title: string; subtitle: string };

const DATA: Item[] = [
  { id: '1', title: 'First', subtitle: 'subtitle one' },
  { id: '2', title: 'Second', subtitle: 'subtitle two' },
  // ...
];

function ItemCard({ title, subtitle }: Omit<Item, 'id'>) {
  return (
    <View style={styles.card}>
      <Text style={styles.title}>{title}</Text>
      <Text style={styles.subtitle}>{subtitle}</Text>
    </View>
  );
}

export default function MyList() {
  return (
    <FlatList
      data={DATA}
      keyExtractor={(item) => item.id}              // must return a unique string
      renderItem={({ item, index }) => (
        <ItemCard title={item.title} subtitle={item.subtitle} />
      )}
      // Performance props
      initialNumToRender={10}                        // items rendered on first paint
      maxToRenderPerBatch={10}                       // items rendered per JS frame
      windowSize={5}                                 // render window = 5x visible area
      removeClippedSubviews                          // unmount off-screen (Android)
      getItemLayout={(data, index) => ({             // skip measurement — BIG perf win
        length: 80,                                  // item height (px)
        offset: 80 * index,
        index,
      })}
      // UX
      ListHeaderComponent={<Text style={styles.header}>My Items</Text>}
      ListFooterComponent={isLoading ? <ActivityIndicator /> : null}
      ListEmptyComponent={<Text>No items found.</Text>}
      ItemSeparatorComponent={() => <View style={styles.separator} />}
      onEndReached={fetchMoreData}                   // infinite scroll
      onEndReachedThreshold={0.5}                   // trigger when 50% from bottom
      refreshControl={
        <RefreshControl refreshing={refreshing} onRefresh={onRefresh} />
      }
    />
  );
}

const styles = StyleSheet.create({
  card: { padding: 16, backgroundColor: '#fff' },
  title: { fontSize: 16, fontWeight: '600' },
  subtitle: { fontSize: 14, color: '#666', marginTop: 4 },
  header: { fontSize: 20, fontWeight: 'bold', padding: 16 },
  separator: { height: 1, backgroundColor: '#eee' },
});
```

### SectionList

Like FlatList but with grouped sections.

```tsx
import { SectionList } from 'react-native';

const SECTIONS = [
  { title: 'Fruits', data: ['Apple', 'Banana'] },
  { title: 'Veggies', data: ['Carrot', 'Broccoli'] },
];

<SectionList
  sections={SECTIONS}
  keyExtractor={(item, index) => item + index}
  renderItem={({ item }) => <Text>{item}</Text>}
  renderSectionHeader={({ section: { title } }) => (
    <Text style={{ fontWeight: 'bold', backgroundColor: '#f0f0f0' }}>{title}</Text>
  )}
/>
```

### FlashList (Shopify)

A drop-in FlatList replacement that is significantly faster — uses a recycling mechanism instead of virtualization.

```bash
npx expo install @shopify/flash-list
```

```tsx
import { FlashList } from '@shopify/flash-list';

// Replace FlatList with FlashList — same API, just add estimatedItemSize
<FlashList
  data={DATA}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <ItemCard {...item} />}
  estimatedItemSize={80}     // REQUIRED — your best estimate of item height
  // All FlatList props work here
  onEndReached={fetchMore}
  onEndReachedThreshold={0.5}
/>
```

### When to Use Which

| Scenario | Use |
|---|---|
| < 50 items | `ScrollView` (simplest) |
| 50–1000 items, uniform height | `FlashList` (fastest) |
| 50–1000 items, variable height | `FlatList` with no `getItemLayout` |
| Grouped data | `SectionList` |
| Horizontal carousel | `FlatList` with `horizontal` prop |
| 10,000+ items | `FlashList` (must use) |

---

## 7. Styling: StyleSheet, Flexbox, Responsive & NativeWind

### StyleSheet.create

Always use `StyleSheet.create` — it validates styles in dev and optimizes them in production by sending IDs instead of objects across the bridge.

```tsx
import { StyleSheet, View, Text } from 'react-native';

export default function Card() {
  return (
    <View style={styles.container}>
      <Text style={[styles.text, styles.bold]}>Hello</Text>
    </View>
  );
}

// Defined outside the component so it's not re-created on each render
const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
    padding: 16,
  },
  text: {
    fontSize: 16,
    color: '#333',
  },
  bold: {
    fontWeight: '700',
  },
});
```

### Flexbox in React Native — CRITICAL DIFFERENCES FROM WEB

| Property | Web Default | React Native Default |
|---|---|---|
| `flexDirection` | `row` | **`column`** |
| `alignContent` | `stretch` | `flex-start` |
| `flexShrink` | `1` | **`0`** |
| `position` | `static` | `relative` |

```tsx
// Centering a box — the most common pattern
<View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
  <Text>Centered</Text>
</View>

// Row layout
<View style={{ flexDirection: 'row', gap: 8 }}>   {/* gap works in RN! */}
  <View style={{ flex: 1, backgroundColor: 'red' }} />
  <View style={{ flex: 2, backgroundColor: 'blue' }} />
</View>

// Common layout patterns
const styles = StyleSheet.create({
  // Fill the screen
  screen: { flex: 1 },

  // Row with space between items
  row: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' },

  // Absolute positioning (like CSS position: absolute)
  overlay: {
    position: 'absolute',
    top: 0, left: 0, right: 0, bottom: 0,
    backgroundColor: 'rgba(0,0,0,0.5)',
  },
});
```

### Responsive Design: Dimensions & useWindowDimensions

```tsx
import { Dimensions, useWindowDimensions, StyleSheet } from 'react-native';

// Static (captured once at app start — does NOT update on rotation)
const { width, height } = Dimensions.get('window');  // viewport
const { width: screenW } = Dimensions.get('screen'); // physical screen

// Dynamic hook (updates on rotation, window resize on iPad)
function ResponsiveCard() {
  const { width, height, fontScale } = useWindowDimensions();

  const isLandscape = width > height;
  const numColumns = width > 600 ? 3 : 2;

  return (
    <View style={{ width: width * 0.9, padding: 16 }}>
      <Text style={{ fontSize: 16 * fontScale }}>Scaled Text</Text>
    </View>
  );
}
```

### Platform-Specific Code

```tsx
import { Platform, StyleSheet } from 'react-native';

// Inline check
const shadowStyle = Platform.select({
  ios: {
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.15,
    shadowRadius: 4,
  },
  android: {
    elevation: 4,
  },
  default: {},
});

// File-based platform split
// Button.ios.tsx  ← used on iOS
// Button.android.tsx  ← used on Android
// Button.tsx  ← fallback
import Button from './Button';  // Metro picks the right file automatically
```

### NativeWind (Tailwind CSS for React Native)

> **⚡ Version note:** NativeWind v4 is stable in 2026 and works with Expo SDK 53. It compiles Tailwind class names to React Native StyleSheets.

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
// babel.config.js
module.exports = {
  presets: [
    ['babel-preset-expo', { jsxImportSource: 'nativewind' }],
    'nativewind/babel',
  ],
};
```

```tsx
// Usage — className prop works just like Tailwind on the web
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

---

## 8. Input, Forms & Keyboard Handling

### KeyboardAvoidingView

Moves your form up when the keyboard appears, so inputs are not hidden.

```tsx
import {
  KeyboardAvoidingView,
  Platform,
  ScrollView,
  TextInput,
  View,
  Text,
  Pressable,
  StyleSheet,
} from 'react-native';

export default function LoginForm() {
  return (
    <KeyboardAvoidingView
      style={{ flex: 1 }}
      // iOS: 'padding' pushes content up by keyboard height
      // Android: 'height' or nothing (usually windowSoftInputMode handles it)
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      keyboardVerticalOffset={90}   // adjust for header height
    >
      <ScrollView
        contentContainerStyle={styles.container}
        keyboardShouldPersistTaps="handled"   // taps on buttons work while keyboard is up
      >
        <TextInput style={styles.input} placeholder="Email" keyboardType="email-address" />
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

### Managing Focus Between Inputs

```tsx
import { useRef } from 'react';
import { TextInput } from 'react-native';

function LoginForm() {
  const passwordRef = useRef<TextInput>(null);

  return (
    <>
      <TextInput
        placeholder="Email"
        returnKeyType="next"
        onSubmitEditing={() => passwordRef.current?.focus()}  // jump to next field
        blurOnSubmit={false}
      />
      <TextInput
        ref={passwordRef}
        placeholder="Password"
        secureTextEntry
        returnKeyType="done"
        onSubmitEditing={handleSubmit}
      />
    </>
  );
}
```

### Forms with React Hook Form

```bash
npm install react-hook-form
```

```tsx
import { useForm, Controller } from 'react-hook-form';
import { TextInput, Text, Pressable, View } from 'react-native';

type FormData = { email: string; password: string };

export default function SignUpForm() {
  const { control, handleSubmit, formState: { errors } } = useForm<FormData>();

  const onSubmit = (data: FormData) => {
    console.log(data);
  };

  return (
    <View>
      <Controller
        control={control}
        name="email"
        rules={{ required: 'Email is required', pattern: { value: /\S+@\S+\.\S+/, message: 'Invalid email' } }}
        render={({ field: { onChange, value, onBlur } }) => (
          <TextInput
            value={value}
            onChangeText={onChange}
            onBlur={onBlur}
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
          <TextInput value={value} onChangeText={onChange} onBlur={onBlur}
            placeholder="Password" secureTextEntry />
        )}
      />
      {errors.password && <Text style={{ color: 'red' }}>{errors.password.message}</Text>}

      <Pressable onPress={handleSubmit(onSubmit)}>
        <Text>Sign Up</Text>
      </Pressable>
    </View>
  );
}
```

### Dismissing the Keyboard

```tsx
import { Keyboard, TouchableWithoutFeedback, View } from 'react-native';

// Wrap content so tapping outside an input dismisses the keyboard
<TouchableWithoutFeedback onPress={Keyboard.dismiss}>
  <View style={{ flex: 1 }}>
    {/* form content */}
  </View>
</TouchableWithoutFeedback>

// Imperatively
Keyboard.dismiss();

// Listen to keyboard events
import { useEffect } from 'react';
useEffect(() => {
  const show = Keyboard.addListener('keyboardDidShow', (e) => {
    console.log('Keyboard height:', e.endCoordinates.height);
  });
  const hide = Keyboard.addListener('keyboardDidHide', () => {});
  return () => { show.remove(); hide.remove(); };
}, []);
```

---

## 9. State & Data Fetching

### Local State & Context

Standard React hooks (`useState`, `useReducer`, `useContext`) work exactly as in web React.

```tsx
import { useState, useContext, createContext, ReactNode } from 'react';

// Theme context example
type Theme = 'light' | 'dark';
const ThemeContext = createContext<{ theme: Theme; toggle: () => void } | null>(null);

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light');
  const toggle = () => setTheme(t => t === 'light' ? 'dark' : 'light');

  return (
    <ThemeContext.Provider value={{ theme, toggle }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be inside ThemeProvider');
  return ctx;
}
```

### Native `fetch` API

```tsx
import { useState, useEffect } from 'react';

type Post = { id: number; title: string; body: string };

function usePosts() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;

    async function load() {
      try {
        const res = await fetch('https://jsonplaceholder.typicode.com/posts?_limit=20');
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data: Post[] = await res.json();
        if (!cancelled) setPosts(data);
      } catch (e) {
        if (!cancelled) setError((e as Error).message);
      } finally {
        if (!cancelled) setLoading(false);
      }
    }

    load();
    return () => { cancelled = true; };
  }, []);

  return { posts, loading, error };
}
```

### TanStack Query (React Query)

The best way to handle server state — caching, refetching, background updates.

```bash
npm install @tanstack/react-query
```

```tsx
// app/_layout.tsx — set up the QueryClient provider
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,   // 5 minutes
      retry: 2,
    },
  },
});

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      {/* ... rest of layout */}
    </QueryClientProvider>
  );
}
```

```tsx
// In a screen component
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

type Post = { id: number; title: string };

const fetchPosts = async (): Promise<Post[]> => {
  const res = await fetch('https://jsonplaceholder.typicode.com/posts?_limit=10');
  if (!res.ok) throw new Error('Network error');
  return res.json();
};

export default function PostsScreen() {
  const { data: posts, isLoading, isError, error, refetch } = useQuery({
    queryKey: ['posts'],         // cache key — must be unique
    queryFn: fetchPosts,
    // staleTime, gcTime, refetchInterval, etc.
  });

  if (isLoading) return <ActivityIndicator />;
  if (isError) return <Text>Error: {error.message}</Text>;

  return (
    <FlatList
      data={posts}
      keyExtractor={(p) => String(p.id)}
      renderItem={({ item }) => <Text>{item.title}</Text>}
      refreshControl={<RefreshControl refreshing={false} onRefresh={refetch} />}
    />
  );
}

// Mutation example (POST / PUT / DELETE)
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
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['posts'] });  // refetch
    },
  });
}
```

### AsyncStorage — Simple Persistence

For non-sensitive key-value data (user preferences, cached data, app state).

```bash
npx expo install @react-native-async-storage/async-storage
```

```tsx
import AsyncStorage from '@react-native-async-storage/async-storage';

// Store
await AsyncStorage.setItem('user_prefs', JSON.stringify({ theme: 'dark' }));

// Read
const raw = await AsyncStorage.getItem('user_prefs');
const prefs = raw ? JSON.parse(raw) : null;

// Remove
await AsyncStorage.removeItem('user_prefs');

// All operations are async — always await or .then()
// Good pattern: custom hook
import { useState, useEffect } from 'react';

function useStoredValue<T>(key: string, defaultValue: T) {
  const [value, setValue] = useState<T>(defaultValue);

  useEffect(() => {
    AsyncStorage.getItem(key).then(raw => {
      if (raw !== null) setValue(JSON.parse(raw));
    });
  }, [key]);

  const store = async (newValue: T) => {
    setValue(newValue);
    await AsyncStorage.setItem(key, JSON.stringify(newValue));
  };

  return [value, store] as const;
}
```

### expo-secure-store — Sensitive Data

For tokens, passwords, and sensitive keys — stored in iOS Keychain / Android Keystore.

```bash
npx expo install expo-secure-store
```

```tsx
import * as SecureStore from 'expo-secure-store';

// Store
await SecureStore.setItemAsync('auth_token', 'eyJhbGci...');

// Read
const token = await SecureStore.getItemAsync('auth_token');

// Delete
await SecureStore.deleteItemAsync('auth_token');

// Size limit: 2048 bytes per value — store a token, not a whole JSON blob
// For larger sensitive data: store the key in SecureStore, encrypt the data with it
```

---

## 10. Expo SDK — Device & Native APIs

All Expo SDK packages are installed with `npx expo install` (not `npm install`) — this ensures the version matches your SDK.

### Permissions (Required for Most Device APIs)

```tsx
import { useCameraPermissions, PermissionStatus } from 'expo-camera';

function CameraButton() {
  const [permission, requestPermission] = useCameraPermissions();

  if (!permission) return null;  // still loading

  if (permission.status !== PermissionStatus.GRANTED) {
    return (
      <Pressable onPress={requestPermission}>
        <Text>Grant Camera Access</Text>
      </Pressable>
    );
  }

  return <CameraView />;
}
```

### expo-camera

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
      quality: 0.8,       // 0-1
      base64: false,      // include base64 data?
      exif: false,
    });
    console.log(photo?.uri);  // file:// URI
  };

  return (
    <CameraView style={{ flex: 1 }} facing={facing} ref={cameraRef}>
      <Pressable onPress={() => setFacing(f => f === 'back' ? 'front' : 'back')}>
        <Text>Flip</Text>
      </Pressable>
      <Pressable onPress={takePicture}>
        <Text>Capture</Text>
      </Pressable>
    </CameraView>
  );
}
```

### expo-image-picker

Lets users pick photos/videos from their library or take a new one.

```bash
npx expo install expo-image-picker
```

```tsx
import * as ImagePicker from 'expo-image-picker';

async function pickImage() {
  // Request permission
  const { status } = await ImagePicker.requestMediaLibraryPermissionsAsync();
  if (status !== 'granted') {
    alert('Permission denied');
    return;
  }

  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ['images'],       // 'images' | 'videos' | ['images', 'videos']
    allowsEditing: true,
    aspect: [4, 3],               // crop ratio (only when allowsEditing: true)
    quality: 0.8,
  });

  if (!result.canceled) {
    const uri = result.assets[0].uri;  // file:// URI
    console.log('Selected:', uri);
  }
}

// Take a photo with the camera
async function takePhoto() {
  const { status } = await ImagePicker.requestCameraPermissionsAsync();
  if (status !== 'granted') return;

  const result = await ImagePicker.launchCameraAsync({ quality: 0.8 });
  if (!result.canceled) {
    console.log(result.assets[0].uri);
  }
}
```

### expo-location

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
      if (status !== 'granted') {
        setError('Permission denied');
        return;
      }

      const loc = await Location.getCurrentPositionAsync({
        accuracy: Location.Accuracy.High,
      });
      setLocation(loc);
    })();
  }, []);

  return { location, error };
}

// Watch position continuously
const subscription = await Location.watchPositionAsync(
  { accuracy: Location.Accuracy.High, timeInterval: 1000, distanceInterval: 10 },
  (location) => console.log(location.coords)
);
// Cleanup
subscription.remove();

// Geocoding
const [address] = await Location.reverseGeocodeAsync({ latitude: 37.78, longitude: -122.42 });
console.log(address.city, address.country);
```

### expo-notifications

```bash
npx expo install expo-notifications
```

```tsx
import * as Notifications from 'expo-notifications';
import { useEffect, useRef } from 'react';
import * as Device from 'expo-device';
import Constants from 'expo-constants';
import { Platform } from 'react-native';

// Set how notifications appear when app is in foreground
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: false,
  }),
});

// Register for push notifications — returns an Expo Push Token
async function registerForPushNotifications(): Promise<string | null> {
  if (!Device.isDevice) return null;  // simulators can't receive push notifications

  const { status: existingStatus } = await Notifications.getPermissionsAsync();
  let finalStatus = existingStatus;

  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }

  if (finalStatus !== 'granted') return null;

  // Get Expo push token (use this to send notifications via Expo's service)
  const projectId = Constants.expoConfig?.extra?.eas?.projectId;
  const token = await Notifications.getExpoPushTokenAsync({ projectId });
  return token.data;  // "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]"
}

// Schedule a local notification
await Notifications.scheduleNotificationAsync({
  content: {
    title: 'Reminder',
    body: 'You have a task due!',
    data: { taskId: '123' },
  },
  trigger: {
    seconds: 60,          // fire in 60 seconds
    repeats: false,
  },
});

// Listen for notifications
function useNotifications() {
  const notificationListener = useRef<Notifications.EventSubscription>();
  const responseListener = useRef<Notifications.EventSubscription>();

  useEffect(() => {
    // Fires when notification is received while app is open
    notificationListener.current = Notifications.addNotificationReceivedListener(n => {
      console.log('Received:', n);
    });

    // Fires when user taps the notification
    responseListener.current = Notifications.addNotificationResponseReceivedListener(r => {
      const data = r.notification.request.content.data;
      console.log('Tapped, data:', data);
    });

    return () => {
      notificationListener.current?.remove();
      responseListener.current?.remove();
    };
  }, []);
}
```

### expo-file-system

```bash
npx expo install expo-file-system
```

```tsx
import * as FileSystem from 'expo-file-system';

// Directories (sandboxed, not accessible by other apps)
const docDir = FileSystem.documentDirectory;   // persistent, backed up
const cacheDir = FileSystem.cacheDirectory;     // may be cleared by OS

// Read / Write
await FileSystem.writeAsStringAsync(`${docDir}data.json`, JSON.stringify({ key: 'val' }));
const content = await FileSystem.readAsStringAsync(`${docDir}data.json`);

// Download a file
const download = FileSystem.createDownloadResumable(
  'https://example.com/large-file.pdf',
  `${docDir}large-file.pdf`
);
const { uri } = await download.downloadAsync();
console.log('Downloaded to:', uri);

// File info
const info = await FileSystem.getInfoAsync(`${docDir}data.json`);
console.log(info.exists, info.size);

// Delete
await FileSystem.deleteAsync(`${docDir}data.json`, { idempotent: true });
```

### expo-av / expo-audio / expo-video

> **⚡ Version note:** In SDK 53, `expo-av` is being split into `expo-audio` (new) and `expo-video` (new). Use the new packages for new projects.

```bash
npx expo install expo-audio expo-video
```

```tsx
// Audio playback
import { useAudioPlayer } from 'expo-audio';

function AudioPlayer() {
  const player = useAudioPlayer(require('../assets/sound.mp3'));

  return (
    <Pressable onPress={() => player.playing ? player.pause() : player.play()}>
      <Text>{player.playing ? 'Pause' : 'Play'}</Text>
    </Pressable>
  );
}

// Video playback
import { VideoView, useVideoPlayer } from 'expo-video';

function VideoPlayer() {
  const player = useVideoPlayer('https://example.com/video.mp4', p => {
    p.loop = true;
    p.play();
  });

  return (
    <VideoView
      player={player}
      style={{ width: 320, height: 180 }}
      allowsFullscreen
      allowsPictureInPicture
    />
  );
}
```

### expo-haptics

Trigger haptic feedback (vibration patterns) — iOS feels especially good.

```bash
npx expo install expo-haptics
```

```tsx
import * as Haptics from 'expo-haptics';

// Light tap (button press)
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);

// Medium impact
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);

// Heavy impact
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy);

// Success / warning / error notification pattern
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Warning);
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);

// Selection changed (scroll picker)
await Haptics.selectionAsync();
```

### expo-sensors

```bash
npx expo install expo-sensors
```

```tsx
import { Accelerometer, Gyroscope, Magnetometer } from 'expo-sensors';
import { useEffect, useState } from 'react';

function useAccelerometer() {
  const [data, setData] = useState({ x: 0, y: 0, z: 0 });

  useEffect(() => {
    Accelerometer.setUpdateInterval(100);  // ms between updates
    const subscription = Accelerometer.addListener(setData);
    return () => subscription.remove();
  }, []);

  return data;
}

// In a component
const { x, y, z } = useAccelerometer();
// x, y, z in g-force units (-1 to 1 approximately)
```

---

## 11. Navigation Patterns & Params

### Stack + Tabs Nested (Most Common App Pattern)

```
app/
├── _layout.tsx         ← Root Stack (wraps everything)
├── (tabs)/
│   ├── _layout.tsx     ← Tab Navigator
│   ├── index.tsx       ← Tab 1: Home
│   └── profile.tsx     ← Tab 2: Profile
├── post/
│   └── [id].tsx        ← Pushed on top of any tab (no tabs visible)
└── modal.tsx           ← Presented as sheet
```

```tsx
// app/_layout.tsx — Root Stack
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      {/* Tabs group — headerShown false so tabs handle their own headers */}
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      {/* Full-screen detail page */}
      <Stack.Screen name="post/[id]" options={{ title: 'Post' }} />
      {/* Bottom sheet modal */}
      <Stack.Screen name="modal" options={{ presentation: 'modal', title: 'Settings' }} />
    </Stack>
  );
}
```

### Passing & Reading Params

```tsx
// Navigating WITH params (three ways)

// 1. Via <Link>
<Link href={{ pathname: '/post/[id]', params: { id: item.id, title: item.title } }}>
  <Text>{item.title}</Text>
</Link>

// 2. Via router.push
router.push({ pathname: '/post/[id]', params: { id: '42', title: 'Hello' } });

// 3. In the URL string (less type-safe)
router.push('/post/42?title=Hello');
```

```tsx
// Reading params in the destination screen
import { useLocalSearchParams, useGlobalSearchParams } from 'expo-router';

export default function PostScreen() {
  // useLocalSearchParams: params from THIS specific route
  const { id, title } = useLocalSearchParams<{ id: string; title: string }>();

  // useGlobalSearchParams: params from the CURRENT URL (including parent routes)
  // Use this rarely — prefer useLocalSearchParams
  const globalParams = useGlobalSearchParams();

  return <Text>Post {id}: {title}</Text>;
}
```

### Passing Complex Data

URL params are strings only. For complex data, use a store (Context, Zustand, etc.) or a local cache.

```tsx
// Bad: serializing objects into URL params
router.push(`/detail?data=${JSON.stringify(complexObj)}`); // don't do this

// Good: store ID in URL, fetch/look up data in the destination screen
router.push({ pathname: '/detail/[id]', params: { id: item.id } });
// Then in DetailScreen, fetch by ID or look up in a shared store
```

### Drawer Navigator

```bash
npx expo install expo-router @react-navigation/drawer react-native-gesture-handler react-native-reanimated
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

### Auth Guard Pattern

```tsx
// app/_layout.tsx — redirect based on auth state
import { Slot, Redirect } from 'expo-router';
import { useAuth } from '../hooks/useAuth';

export default function RootLayout() {
  const { isLoggedIn, isLoading } = useAuth();

  if (isLoading) return <SplashScreen />;

  if (!isLoggedIn) {
    return <Redirect href="/login" />;
  }

  return <Slot />;   // render whatever child route matches
}
```

---

## 12. Animations & Gestures

### react-native-reanimated

The standard animation library for React Native. Runs animations on the **UI thread** (not the JS thread) so they're smooth even during JS work.

```bash
npx expo install react-native-reanimated
```

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSpring,
  withSequence,
  withRepeat,
  Easing,
  FadeIn,
  FadeOut,
  SlideInRight,
} from 'react-native-reanimated';
import { Pressable, View } from 'react-native';

// 1. Basic fade animation
function FadeBox() {
  const opacity = useSharedValue(1);    // shared value lives on UI thread

  const animatedStyle = useAnimatedStyle(() => ({
    opacity: opacity.value,             // "worklet" — runs on UI thread
  }));

  return (
    <Pressable onPress={() => {
      opacity.value = withTiming(opacity.value === 1 ? 0 : 1, { duration: 300 });
    }}>
      <Animated.View style={[{ width: 100, height: 100, backgroundColor: 'blue' }, animatedStyle]} />
    </Pressable>
  );
}

// 2. Spring animation (bouncy)
function SpringBox() {
  const scale = useSharedValue(1);

  const style = useAnimatedStyle(() => ({ transform: [{ scale: scale.value }] }));

  return (
    <Pressable
      onPress={() => { scale.value = withSpring(1.2, { damping: 4, stiffness: 100 }); }}
      onPressOut={() => { scale.value = withSpring(1); }}
    >
      <Animated.View style={[{ width: 100, height: 100, backgroundColor: 'green' }, style]} />
    </Pressable>
  );
}

// 3. Layout animations (enter/exit)
function AnimatedItem({ visible }: { visible: boolean }) {
  return visible ? (
    <Animated.View entering={FadeIn.duration(300)} exiting={FadeOut.duration(200)}>
      {/* content */}
    </Animated.View>
  ) : null;
}

// 4. Infinite rotation (loading spinner)
function Spinner() {
  const rotation = useSharedValue(0);

  useEffect(() => {
    rotation.value = withRepeat(
      withTiming(360, { duration: 1000, easing: Easing.linear }),
      -1,    // repeat forever
      false  // do not reverse
    );
  }, []);

  const style = useAnimatedStyle(() => ({
    transform: [{ rotate: `${rotation.value}deg` }],
  }));

  return <Animated.View style={[{ width: 40, height: 40, borderWidth: 3, borderColor: '#007AFF', borderRadius: 20, borderTopColor: 'transparent' }, style]} />;
}
```

### react-native-gesture-handler

Required for Drawer and bottom sheets; also gives you better gesture primitives than built-in.

```bash
npx expo install react-native-gesture-handler
```

```tsx
// MUST wrap your app root with this (usually in _layout.tsx)
import { GestureHandlerRootView } from 'react-native-gesture-handler';

export default function RootLayout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      {/* everything else */}
    </GestureHandlerRootView>
  );
}
```

```tsx
// Pan gesture — drag a box
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, { useSharedValue, useAnimatedStyle } from 'react-native-reanimated';

function DraggableBox() {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const startX = useSharedValue(0);
  const startY = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onBegin(() => {
      startX.value = translateX.value;
      startY.value = translateY.value;
    })
    .onUpdate((e) => {
      translateX.value = startX.value + e.translationX;
      translateY.value = startY.value + e.translationY;
    });

  const style = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }, { translateY: translateY.value }],
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[{ width: 100, height: 100, backgroundColor: 'coral', borderRadius: 8 }, style]} />
    </GestureDetector>
  );
}
```

### Animated API (Built-in, Simpler)

For simple cases, the built-in `Animated` API is fine (but runs on JS thread by default).

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
      <Animated.View style={{ transform: [{ scale: anim }], padding: 16, backgroundColor: '#007AFF', borderRadius: 8 }}>
        {/* content */}
      </Animated.View>
    </Pressable>
  );
}
```

---

## 13. Configuration: app.json / app.config.ts

### app.json

The static configuration for your Expo project.

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

    "web": {
      "bundler": "metro",
      "output": "static",
      "favicon": "./assets/images/favicon.png"
    },

    "plugins": [
      "expo-router",
      "expo-font",
      [
        "expo-camera",
        { "cameraPermission": "Allow $(PRODUCT_NAME) to access your camera." }
      ]
    ],

    "experiments": {
      "typedRoutes": true
    },

    "extra": {
      "eas": {
        "projectId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
      }
    }
  }
}
```

### app.config.ts (Dynamic Config)

Use when you need environment variables, conditional logic, or computed values.

```ts
// app.config.ts
import { ExpoConfig, ConfigContext } from 'expo/config';

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: process.env.APP_ENV === 'production' ? 'My App' : 'My App (Dev)',
  slug: 'my-app',
  extra: {
    apiUrl: process.env.API_URL ?? 'https://api.example.com',
    eas: { projectId: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' },
  },
  // Override anything based on env
  ios: {
    ...config.ios,
    bundleIdentifier:
      process.env.APP_ENV === 'production'
        ? 'com.example.myapp'
        : 'com.example.myapp.dev',
  },
});
```

### Accessing Config at Runtime

```tsx
import Constants from 'expo-constants';

const apiUrl = Constants.expoConfig?.extra?.apiUrl;  // from app.config.ts extra
```

### Config Plugins

Config plugins modify native code (Info.plist, AndroidManifest.xml, etc.) at build time — without manually editing native files.

```json
// app.json plugins array
"plugins": [
  "expo-router",
  ["expo-camera", { "cameraPermission": "Custom camera permission text." }],
  ["expo-location", {
    "locationAlwaysAndWhenInUsePermission": "Allow $(PRODUCT_NAME) to use your location."
  }],
  [
    "expo-build-properties",
    {
      "android": { "compileSdkVersion": 34, "targetSdkVersion": 34 },
      "ios": { "deploymentTarget": "15.1" }
    }
  ]
]
```

### Environment Variables

```bash
# .env (committed — public vars only)
EXPO_PUBLIC_API_URL=https://api.example.com

# .env.local (NOT committed — local overrides)
EXPO_PUBLIC_API_URL=http://localhost:3000
```

```tsx
// Only EXPO_PUBLIC_ prefix is exposed to the JS bundle
const apiUrl = process.env.EXPO_PUBLIC_API_URL;

// For true secrets (API keys for native modules) — use app.config.ts + eas.json secrets
// NEVER put secret keys in EXPO_PUBLIC_ vars — they are bundled and readable
```

### App Icons & Splash Screen

**Icon:** 1024×1024 PNG at `assets/images/icon.png`. Expo generates all required sizes.

**Android Adaptive Icon:** 1024×1024 foreground PNG on transparent background.

**Splash Screen:**

```bash
npx expo install expo-splash-screen
```

```tsx
// app/_layout.tsx
import * as SplashScreen from 'expo-splash-screen';

SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    async function prepare() {
      await loadFonts();
      await preloadData();
      setReady(true);
      await SplashScreen.hideAsync();   // hides the native splash screen
    }
    prepare();
  }, []);

  if (!ready) return null;
  return <Stack />;
}
```

---

## 14. Building & Shipping with EAS

EAS (Expo Application Services) is the cloud build and delivery platform.

```bash
npm install -g eas-cli
eas login
eas init   # links your project to Expo's servers and creates a projectId
```

### eas.json

Controls build, submit, and update configurations.

```json
{
  "cli": {
    "version": ">= 12.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "env": {
        "APP_ENV": "development"
      }
    },
    "preview": {
      "distribution": "internal",
      "env": {
        "APP_ENV": "preview"
      },
      "android": {
        "buildType": "apk"
      }
    },
    "production": {
      "autoIncrement": true,
      "env": {
        "APP_ENV": "production"
      }
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "your@apple.com",
        "ascAppId": "1234567890",
        "appleTeamId": "ABCDE12345"
      },
      "android": {
        "serviceAccountKeyPath": "./service-account.json",
        "track": "production"
      }
    }
  },
  "update": {
    "channel": "production"
  }
}
```

### EAS Build

```bash
# Build a development client (replaces Expo Go for your project)
eas build --profile development --platform ios
eas build --profile development --platform android

# Build a preview APK (share with testers via link)
eas build --profile preview --platform android

# Build for both stores (production)
eas build --profile production --platform all

# Build locally (no cloud — requires Xcode/Android Studio)
eas build --profile production --platform ios --local
```

**Development Client:** After building, install it on your device. Then `npx expo start` and scan the QR code — your app runs inside your own native shell that includes all your custom native modules.

### Build Profiles Explained

| Profile | Purpose | `distribution` | Output |
|---|---|---|---|
| `development` | Day-to-day development | `internal` | Custom Expo Go with your native modules |
| `preview` | Share with testers | `internal` | APK (Android) or IPA for ad-hoc (iOS) |
| `production` | App Store / Play Store | `store` | Signed IPA / AAB |

### EAS Submit

```bash
# Submit to App Store Connect (iOS)
eas submit --platform ios --profile production

# Submit to Google Play (Android)
eas submit --platform android --profile production
```

You need:
- **iOS:** Apple Developer account ($99/yr), Xcode provisioning profiles (EAS handles automatically)
- **Android:** Google Play Console account ($25 one-time), a service account JSON key

### EAS Update (OTA Updates)

Push JavaScript + asset changes to users **without** going through the App Store review.

> **Important:** OTA updates can only change JS code and assets. Native code changes require a new build.

```bash
# Push an update to the production channel
eas update --channel production --message "Fix checkout bug"

# Update a specific branch
eas update --branch my-feature --message "Test this feature"

# Preview: show what would be updated
eas update --channel production --message "Release notes" --dry-run
```

```json
// app.json — configure update behavior
{
  "expo": {
    "updates": {
      "enabled": true,
      "fallbackToCacheTimeout": 0,
      "checkAutomatically": "ON_LOAD"
    },
    "runtimeVersion": {
      "policy": "appVersion"
    }
  }
}
```

```tsx
// Manually check for and apply updates
import * as Updates from 'expo-updates';

async function checkForUpdate() {
  if (!Updates.isEmbeddedLaunch) return;   // not in Expo Go

  try {
    const update = await Updates.checkForUpdateAsync();
    if (update.isAvailable) {
      await Updates.fetchUpdateAsync();
      await Updates.reloadAsync();    // restart app with new bundle
    }
  } catch (e) {
    console.error('Update check failed:', e);
  }
}
```

### Store Checklist Before Submitting

**iOS (App Store):**
- [ ] Icon at 1024×1024, no alpha channel
- [ ] Screenshots for iPhone 6.5" and 5.5" (minimum)
- [ ] Privacy Nutrition Labels filled in App Store Connect
- [ ] App uses `expo-tracking-transparency` if IDFA accessed
- [ ] Tested on a real iOS device

**Android (Play Store):**
- [ ] High-res icon (512×512 PNG)
- [ ] Feature graphic (1024×500 PNG)
- [ ] Screenshots for phone (minimum)
- [ ] Target API level meets Play Store requirements (API 34+ in 2024+)
- [ ] Tested on a real Android device

---

## 15. Debugging

### React Native DevTools

> **⚡ Version note:** Starting with React Native 0.76+, the new **React Native DevTools** replaces Flipper as the primary debugging tool. It's built into Metro.

```bash
# Start Metro with DevTools
npx expo start

# Press 'j' in Metro terminal to open React Native DevTools in Chrome
# Or shake device / press Cmd+D (iOS sim) / Cmd+M (Android emu) → "Open DevTools"
```

React Native DevTools gives you:
- **Console** — JS logs, errors, warnings
- **Sources** — breakpoints in your TypeScript source (with source maps)
- **React Components inspector** — view component tree and props
- **Network** — HTTP requests and responses (⚡ New in RN 0.76+)
- **Performance** — JS frame time, renders

### In-App Developer Menu

| Platform | How to open |
|---|---|
| iOS Simulator | `Cmd + D` |
| Android Emulator | `Cmd + M` (Mac) or `Ctrl + M` (Win/Linux) |
| Physical device | Shake the device |

Options: Reload, Open DevTools, Toggle Performance Monitor, Enable Element Inspector.

### Console Logging

```tsx
console.log('Debug info:', { user, token });
console.warn('Deprecated API used');
console.error('Something broke', error);

// Structured logging helper
const log = (tag: string, data?: unknown) => {
  if (__DEV__) {  // __DEV__ is true in dev, false in production
    console.log(`[${tag}]`, data ?? '');
  }
};
```

### Error Boundary

```tsx
import { Component, ReactNode, ErrorInfo } from 'react';
import { View, Text, Pressable } from 'react-native';

type Props = { children: ReactNode; fallback?: ReactNode };
type State = { hasError: boolean; error?: Error };

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    // Log to Sentry, Bugsnag, etc.
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

### Expo Logs

```bash
# View native logs in terminal
npx expo start   # Metro shows JS logs
adb logcat       # Android native logs
# iOS: use Xcode's Console or `idevicesyslog`

# Filter to your app
adb logcat | grep -i "ReactNative\|MyApp"
```

### Common Error Messages & Fixes

| Error | Likely Cause | Fix |
|---|---|---|
| `Invariant Violation: Text strings must be rendered within a <Text> component` | JSX has a string outside `<Text>` | Wrap the string in `<Text>` |
| `VirtualizedList: You have a large list that is slow to update` | FlatList inside ScrollView | Never nest FlatList inside ScrollView — use `ListHeaderComponent` instead |
| `Error: [TypeError: undefined is not an object]` | Accessing a property before data loads | Add a loading state check |
| Metro bundler not updating | Stale cache | `npx expo start --clear` |
| `Unable to resolve module X` | Missing install or wrong package name | `npx expo install X` then restart Metro |
| White screen on iOS simulator | JS error before first render | Open DevTools (Cmd+D) to see the stack trace |

---

## 16. Tips, Tricks & Gotchas

### Platform Differences: iOS vs Android

| Behavior | iOS | Android |
|---|---|---|
| Shadow | `shadowColor`, `shadowOffset`, `shadowOpacity`, `shadowRadius` | `elevation` (number) |
| Font rendering | Subpixel antialiasing | Outline/bitmap fonts |
| Status bar | Overlaps content | Pushes content down (usually) |
| Back navigation | Swipe from left edge | Hardware back button |
| Keyboard behavior | `KeyboardAvoidingView` with `padding` | Usually handled by `windowSoftInputMode` |
| Date/Time picker | Native wheel spinner | Native calendar/clock dialogs |
| Permission dialogs | "Allow Once / Allow While Using / Don't Allow" | "Allow / Deny" |
| Text capitalization | System setting can auto-capitalize | Same |

```tsx
// Always use Platform.select or Platform.OS for divergent behavior
import { Platform } from 'react-native';

const hitSlop = Platform.select({
  ios: { top: 10, bottom: 10, left: 10, right: 10 },
  android: { top: 5, bottom: 5, left: 5, right: 5 },
  default: {},
});
```

### Expo Go Limitations — When You Need a Dev Build

If any of these apply, stop using Expo Go and create a development build:
- You install any npm package with native code that's NOT in the Expo SDK
- You use `react-native-maps`, `react-native-firebase`, `Stripe`, or any other third-party native module
- You need push notifications on a real device (token generation requires your bundle ID)
- You need to test custom native modules or config plugins
- Your app uses `expo-camera` barcode scanning (requires dev build in SDK 50+)

```bash
# Create and install a dev build
eas build --profile development --platform ios
# Install the resulting .ipa/.apk on your device, then:
npx expo start --dev-client
```

### New Architecture (Fabric + TurboModules)

> **⚡ Version note:** In Expo SDK 52+, the New Architecture is **enabled by default** for new projects.

- **Fabric** — new synchronous renderer; removes the async bridge for UI operations
- **TurboModules** — lazy-loaded native modules; faster startup
- **JSI (JavaScript Interface)** — direct C++ binding; eliminates JSON serialization
- **Concurrent React** — enables `useTransition`, `useDeferredValue`, Suspense on native

```json
// app.json — disable new architecture if a package is not yet compatible
{
  "expo": {
    "newArchEnabled": false   // only do this as a last resort
  }
}
```

Check package compatibility at: https://reactnative.directory (filter "New Architecture")

### Safe Areas — Always

Never hardcode `paddingTop: 44` or `paddingBottom: 34`. Device notch/island/home indicator sizes vary.

```tsx
// Always use react-native-safe-area-context
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function Header() {
  const insets = useSafeAreaInsets();
  return (
    <View style={{ paddingTop: insets.top, backgroundColor: '#fff' }}>
      <Text>Title</Text>
    </View>
  );
}
```

### List Performance Rules

1. Always use `keyExtractor` returning a **string** (not a number)
2. Use `getItemLayout` when items have fixed height — eliminates measurement overhead
3. Use `FlashList` for lists with 100+ items
4. Memoize `renderItem` with `useCallback` and item components with `React.memo`
5. Never put a `FlatList` inside a `ScrollView`
6. Avoid `map()` for long arrays — use `FlatList` instead

```tsx
// Optimized FlatList pattern
const renderItem = useCallback(({ item }: { item: MyItem }) => (
  <MemoizedItemCard item={item} />
), []);

// Memoized item component
const MemoizedItemCard = React.memo(({ item }: { item: MyItem }) => (
  <View>
    <Text>{item.title}</Text>
  </View>
));
```

### Images — Common Gotchas

```tsx
// 1. Local images need require() — not string paths
<Image source={require('./assets/photo.png')} />  // correct
<Image source={{ uri: './assets/photo.png' }} />  // WRONG — won't work

// 2. Remote images need explicit dimensions (no intrinsic size)
<Image source={{ uri: 'https://...' }} style={{ width: 200, height: 200 }} />

// 3. Network images on Android need android:usesCleartextTraffic="true"
//    if using HTTP (not HTTPS) — or just always use HTTPS

// 4. Use expo-image for network images — it caches aggressively
```

### TypeScript Tips for RN

```tsx
// Type your navigation params with Expo Router's typed routes
// Enable in app.json:  "experiments": { "typedRoutes": true }

// Type component props properly
import { ViewStyle, TextStyle, StyleProp } from 'react-native';

type ButtonProps = {
  title: string;
  onPress: () => void;
  style?: StyleProp<ViewStyle>;
  textStyle?: StyleProp<TextStyle>;
  disabled?: boolean;
};

// Type your FlatList data
type ListItem = { id: string; name: string };
<FlatList<ListItem>
  data={items}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <Text>{item.name}</Text>}
/>
```

### Common Gotchas Checklist

- **`flex: 1` not working?** Make sure the parent also has `flex: 1` — flex bubbles up through the tree.
- **Styles not updating?** Clear Metro cache: `npx expo start --clear`.
- **Android text cut off?** Don't set `height` on `<Text>` — use padding on the container.
- **Tap target too small?** Use `hitSlop` on `<Pressable>` to expand the touchable area.
- **`overflow: hidden` doesn't clip on Android?** Works on iOS, inconsistent on Android. Use `borderRadius` on the item instead.
- **Images blurry on high-DPI screens?** Use `@2x` and `@3x` suffix variants for local images (Metro picks the right one automatically).
- **`onPress` fires twice?** Caused by nesting `<Pressable>` inside `<TouchableOpacity>` or similar.
- **App crashes on Android but not iOS?** Check `elevation` (Android shadow), `useNativeDriver: true` animations, and New Architecture compatibility.

---

## 17. Study Path

This is an ordered learning path for someone going from zero to shipping a real app.

### Stage 1 — Foundations (Week 1–2)

**Learn these concepts:**
- JavaScript / TypeScript fundamentals (if shaky: do TypeScript basics first)
- React fundamentals: components, props, state, hooks, context
- How mobile apps differ from web apps (no DOM, native UI, sandboxed, offline-first)

**Build:** A simple counter app and a static "about me" profile screen. No navigation yet.

### Stage 2 — Expo Basics (Week 2–3)

**Learn these concepts:**
- `npx create-expo-app`, project structure, Metro bundler
- Core components: View, Text, StyleSheet, Flexbox (column-first!), Image, Pressable
- Expo Go for quick testing
- ScrollView and FlatList basics

**Build:** A recipe browser with a static list of recipes, images, and a detail screen pushed via `router.push`.

### Stage 3 — Expo Router & Navigation (Week 3–4)

**Learn these concepts:**
- Expo Router file conventions: `_layout.tsx`, `[id].tsx`, `(groups)`
- Stack navigator: push, pop, header customization
- Tab navigator: icons with `@expo/vector-icons`
- Passing params with `useLocalSearchParams`
- Auth guard pattern with `<Redirect />`

**Build:** A multi-tab notes app with a home tab (list), add tab (form), and tapping a note pushes to a detail screen.

### Stage 4 — Data & APIs (Week 4–5)

**Learn these concepts:**
- `fetch` with async/await, error handling, loading states
- TanStack Query: `useQuery`, `useMutation`, `QueryClient`
- AsyncStorage for simple persistence
- expo-secure-store for tokens

**Build:** Connect your notes app to a real backend (supabase.com is free and easy). Implement login/logout with token storage.

### Stage 5 — Native Device Features (Week 5–6)

**Learn these concepts:**
- Permissions flow (request → check → use)
- expo-image-picker for photo selection
- expo-location for coordinates
- expo-notifications for local and push notifications
- expo-haptics for feedback polish

**Build:** Extend the notes app — let users attach photos to notes (image picker). Add a reminder notification (local notification after a set time).

### Stage 6 — Styling & UX Polish (Week 6–7)

**Learn these concepts:**
- `useWindowDimensions` for responsive layouts
- Platform.select for iOS/Android differences
- Safe area insets everywhere
- `react-native-reanimated` for smooth animations
- `react-native-gesture-handler` for swipe-to-delete

**Build:** Add swipe-to-delete on the notes list with a fade-out animation. Add a scale animation on the "Add" button.

### Stage 7 — Build & Ship (Week 7–8)

**Learn these concepts:**
- EAS Build: dev / preview / production profiles
- Creating a dev build (replace Expo Go)
- EAS Update for OTA JS updates
- App Store / Play Store submission process

**Build:** Ship a real app to TestFlight (iOS) or the Play Store internal testing track.

### Projects to Build (Progressive Complexity)

| Project | Key Concepts Practiced |
|---|---|
| Counter / Flashcard app | State, StyleSheet, Flexbox |
| Weather app | fetch, loading states, FlatList, Dimensions |
| Recipe browser | Expo Router, Stack, image, dynamic routes |
| Notes app (local) | CRUD, AsyncStorage, forms, React Hook Form |
| Notes app (with backend) | TanStack Query, auth, SecureStore, mutations |
| Photo journal | Image picker, expo-image, camera, file system |
| Fitness tracker | Accelerometer, location, notifications, charts |
| Chat app | WebSockets or Supabase Realtime, FlatList inverted |
| Full SaaS mobile app | Everything above + EAS Build + Store submission |

### Recommended Resources (for when you have internet)

- **Expo Docs:** docs.expo.dev — the authoritative source
- **React Native Docs:** reactnative.dev/docs
- **Expo Router Docs:** docs.expo.dev/router
- **EAS Docs:** docs.expo.dev/eas
- **React Native Directory:** reactnative.directory — find packages, filter by New Architecture support
- **TanStack Query:** tanstack.com/query/latest
- **Reanimated:** docs.swmansion.com/react-native-reanimated
- **NativeWind:** nativewind.dev

### Quick Reference: Most-Used Commands

```bash
# Create & Run
npx create-expo-app@latest MyApp
npx expo start
npx expo start --clear                   # clear Metro cache
npx expo run:ios                         # build & run on iOS simulator
npx expo run:android                     # build & run on Android emulator

# Install packages (always use expo install, not npm install for native packages)
npx expo install expo-image expo-camera expo-location

# EAS
eas login
eas build --profile development --platform ios
eas build --profile production --platform all
eas submit --platform ios --profile production
eas update --channel production --message "Bug fix"

# Debugging
# Press 'j' in Metro terminal → React Native DevTools
# Cmd+D (iOS sim) / Cmd+M (Android emu) → Dev Menu
# Shake device → Dev Menu
```

---

*This guide was written for Expo SDK 53 / React Native 0.77+ with New Architecture enabled by default. For the latest API changes always consult docs.expo.dev.*
