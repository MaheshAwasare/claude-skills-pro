---
name: react-native-expo
description: Build a production React Native app with Expo — EAS builds, OTA updates via expo-updates, native modules without ejecting, push notifications, deep links, app store submission, and the Expo SDK upgrade discipline. Use when starting a new mobile app or when an existing RN-vanilla project should switch to Expo.
---

# React Native + Expo

Modern Expo (SDK 53+) is the *default* React Native experience now. Bare RN is for cases that genuinely need it; everyone else gains by starting with Expo.

## When to use

- New mobile app, iOS + Android.
- Cross-platform with web (Expo Router supports React Native Web).
- You want EAS Build (cloud-built iOS without a Mac).
- You want OTA updates (push JS-only fixes without an app store review).

## When NOT to use

- Game engine — use Unity / Unreal.
- Heavy native UI customizations that fight Expo defaults.
- App must work fully offline-first with custom native sync layers — possible but bare RN is simpler.

## Decisions made for you

| Decision | Choice | Why |
|---|---|---|
| Framework | Expo SDK 53+ | Workflow > Bare unless blocked |
| Routing | Expo Router (file-based) | Replaces React Navigation boilerplate |
| State | Zustand or TanStack Query (server) | Redux is overkill for most apps |
| Auth | Clerk Expo SDK / Supabase / your backend | Don't roll auth on mobile |
| Builds | EAS Build (cloud) | No Mac required; CI integration |
| Updates | expo-updates (OTA) | Bug fixes without store review |
| Push | expo-notifications | One API for FCM + APNs |
| Deep links | expo-linking + Expo Router | Universal links, scheme handling |
| Native modules | Expo Modules API | Don't eject; write custom modules with Expo Modules |
| Testing | Jest + Detox (E2E) or Maestro | Maestro is simpler for most |

## Project layout (Expo Router)

```
my-app/
  app/
    _layout.tsx                # root layout
    (tabs)/
      _layout.tsx              # tab nav
      index.tsx                # home
      profile.tsx
    auth/
      sign-in.tsx
      sign-up.tsx
    [...not-found].tsx
  components/
  hooks/
  lib/
    api.ts
    auth.ts
  assets/
  app.json                     # expo config
  eas.json                     # EAS build profiles
  package.json
  tsconfig.json
```

`app.json` controls the entire app config — bundle ID, icons, splash, plugins. **Never edit native iOS/Android code directly** in Expo workflow; configure via plugins.

## EAS Build setup

```bash
npx expo install
eas init                       # creates project on EAS
eas build:configure            # creates eas.json
```

```json
// eas.json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal",
      "ios":     { "simulator": true },
      "android": { "buildType": "apk" }
    },
    "production": {
      "autoIncrement": true,
      "channel": "production"
    }
  },
  "submit": {
    "production": {}
  }
}
```

Build commands:
```bash
eas build --profile development --platform ios       # for sim/device dev
eas build --profile production --platform all        # store-bound builds
eas submit --platform ios --latest                   # uploads to App Store Connect
```

No Mac required for iOS builds. GH Actions integrates via `eas build` triggered from CI.

## OTA updates (the killer feature)

```bash
eas update --channel production --message "fix login bug"
```

Pushes the JS bundle to all installs of the matching native build. **Limitation:** OTA can only update JS — native deps (libraries with native code, Expo SDK upgrades) require a new build.

Track which native version supports which OTA bundle via `runtimeVersion`:
```json
// app.json
{ "expo": { "runtimeVersion": { "policy": "appVersion" } } }
```

Each store build has a runtimeVersion; OTA only delivers to matching builds.

## Push notifications (Expo's wrapper)

```ts
import * as Notifications from "expo-notifications";

// Request permission
const { status } = await Notifications.requestPermissionsAsync();
if (status !== "granted") return;

// Get Expo push token (one token works for both iOS and Android)
const token = (await Notifications.getExpoPushTokenAsync()).data;
// Send to your backend for storage

// Backend sends push via Expo Push API
fetch("https://exp.host/--/api/v2/push/send", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    to: token,
    title: "Hi",
    body: "Hello from server",
    data: { url: "myapp://chat/123" },
  }),
});
```

Expo's relay handles APNs (Apple) and FCM (Google) for you. For high-volume, switch to direct APNs/FCM later — `expo-server-sdk` makes it interchangeable.

## Deep links

`app.json`:
```json
{
  "expo": {
    "scheme": "myapp",
    "ios":     { "associatedDomains": ["applinks:example.com"] },
    "android": { "intentFilters": [{ "data": [{ "scheme": "https", "host": "example.com" }], "autoVerify": true }] }
  }
}
```

Universal links route automatically with Expo Router file-based routing — `/chat/123` opens `app/chat/[id].tsx`. No manual route registration.

## State + server data

```ts
// Zustand for client state
import { create } from "zustand";
const useAppStore = create((set) => ({
  theme: "light",
  setTheme: (t) => set({ theme: t }),
}));

// TanStack Query for server data
import { QueryClient, QueryClientProvider, useQuery } from "@tanstack/react-query";
const qc = new QueryClient();
function App() {
  return <QueryClientProvider client={qc}>...</QueryClientProvider>;
}
function ProjectList() {
  const { data } = useQuery({ queryKey: ["projects"], queryFn: api.listProjects });
  return <FlatList data={data} ... />;
}
```

Avoid Redux unless you have specific needs (time-travel debugging, complex undo). Zustand + TanStack Query handles 95% of cases with 1/10 the boilerplate.

## Anti-patterns

- **Ejecting from Expo because you "need a native module"** — Expo Modules API + config plugins handle most cases. Eject only after exhausting alternatives.
- **Editing `ios/` or `android/` folders directly** — gets clobbered on prebuild. Use config plugins.
- **OTA-pushing native changes** — silently fails because runtimeVersion mismatches. Build, then update.
- **One Expo project for prod + staging** — same Push token namespace, same OTA channel risk. Use separate slugs and EAS projects.
- **Storing tokens in AsyncStorage in plain text** — use `expo-secure-store` for credentials.
- **`useEffect` for navigation side effects** — use `expo-router`'s typed navigation helpers.
- **No splash + icon variants** — store rejection. Generate via `expo-splash-screen` and `expo-app-icon-utils`.
- **Skipping `runtimeVersion`** — OTA mismatches cause crashes; users panic.
- **No error boundaries** — RN white screen of death. Wrap top-level routes.
- **Pushing without internal preview build** — store rejection rate skyrockets without TestFlight/Internal Testing.
- **Hardcoded API URLs without `.env`** — staging vs prod mix-up. Use `expo-constants` + EAS env vars.

## Verify it worked

- [ ] `npx expo start` opens dev menu; QR code launches the app on phone via Expo Go (or dev client).
- [ ] `eas build --profile preview --platform ios` produces a `.ipa` you can sideload via TestFlight.
- [ ] Tapping a deep link `myapp://chat/123` opens the right screen.
- [ ] Push notification with `data: { url: ... }` opens the right screen on tap.
- [ ] OTA update via `eas update` reaches a real device within ~1 min after force-close + reopen.
- [ ] `runtimeVersion` is set; OTA to a build with mismatched runtime is correctly skipped (no crash).
- [ ] Auth tokens stored in `expo-secure-store`, not `AsyncStorage`.
- [ ] Production submission to App Store + Google Play reviews in standard time (no rejections from missing icons/splash/permissions strings).
- [ ] App opens on cold start under 3s on mid-tier Android.
- [ ] CI runs `eas build --non-interactive` on tag push.
