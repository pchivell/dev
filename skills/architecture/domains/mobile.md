# Domain: Mobile Applications

Applies to: native iOS (Swift/SwiftUI), native Android (Kotlin/Jetpack),
and cross-platform (React Native / Expo, Flutter) mobile apps.

---

## Domain Characteristics

- **Platform constraints**: App Store / Play Store review; OS version fragmentation; sandbox permissions model
- **Connectivity**: intermittent; offline-first is a quality expectation, not optional
- **UX contract**: 60 fps animations; instant feedback; background task budget is limited
- **State persistence**: local DB (SQLite/Realm/CoreData) survives app restarts; remote state via API
- **Updates**: users may run old versions for months; API versioning and graceful degradation are mandatory
- **Push notifications**: first-class feature requiring APNs / FCM integration

---

## Recommended Architecture

### Layer Model

```
┌─────────────────────────────────────────┐
│  Presentation (Views / Screens)         │  ← UI components, navigation
├─────────────────────────────────────────┤
│  ViewModel / Presenter                  │  ← UI state, input handling
├─────────────────────────────────────────┤
│  Domain (Use Cases / Interactors)       │  ← business rules, pure functions
├─────────────────────────────────────────┤
│  Data Layer (Repositories)             │  ← abstract data sources
├─────────────────────────────────────────┤
│  Infrastructure                         │  ← API clients, local DB, device APIs
└─────────────────────────────────────────┘
```

**Dependency rule**: Presentation → ViewModel → Domain ← Repository ← Infrastructure.
Domain layer has zero imports from UI or infrastructure frameworks.

### State Management Patterns

| Pattern | Platform | When |
|---------|----------|------|
| **Redux / Zustand** | React Native | predictable global state; time-travel debugging |
| **Context + useReducer** | React Native | small-to-medium apps; avoid prop drilling |
| **ViewModel + StateFlow** | Android/Kotlin | official Jetpack recommendation |
| **ObservableObject / @State** | iOS/SwiftUI | built-in; fine for simple screens |
| **Riverpod / Bloc** | Flutter | complex state; testable |

### Offline-First Pattern

```
User action
  → optimistic UI update (immediate feedback)
  → write to local queue / DB
  → sync worker pushes queue to API when online
  → on success: confirm; on conflict: resolve or surface to user
```

Never make a network call on the main thread. Every data operation must have a
local-first path.

---

## Technology Selection Framework

| Decision | Default | Alternatives |
|----------|---------|-------------|
| Cross-platform | **React Native + Expo** | Flutter (Dart; better animations); Native (max performance/fidelity) |
| Local DB | **SQLite via Drizzle/Expo SQLite** | Realm (sync), WatermelonDB (reactive) |
| Navigation | **Expo Router** (file-based) | React Navigation |
| API client | **TanStack Query + fetch** | Apollo (GraphQL), SWR |
| Auth | **Clerk / Supabase Auth** | Firebase Auth, Auth0 |
| Push | **Expo Notifications** | FCM direct, OneSignal |
| State | **Zustand** | Redux Toolkit, Jotai |

---

## Key Decision Checkpoints

1. **Cross-platform vs native?** Time-to-market vs platform fidelity vs long-term maintenance cost.
2. **Offline-first depth?** Simple caching vs full sync engine (Realm, PowerSync, Electric).
3. **Backend coupling?** Tight (custom API) vs BaaS (Supabase, Firebase) vs headless CMS.
4. **Auth requirements?** Social login, biometrics, enterprise SSO?
5. **App size budget?** Cross-platform frameworks add 15–30 MB baseline; matters for emerging markets.
6. **Live updates?** Expo EAS Update / CodePush for JS-layer hotfixes without store review.

---

## Common Pitfalls

- **Fetching on every mount** — cache aggressively; network is slow and unreliable on mobile.
- **No error boundaries** — an unhandled error crashing the whole app is unacceptable; use error boundaries and Sentry.
- **Hardcoded API base URL** — use environment config (`.env.development`, `.env.production`).
- **Blocking the JS thread** — heavy computation must run on a background thread or Web Worker.
- **No deep-link / universal link support** — add it from the start; retrofitting routing is painful.
- **Ignoring accessibility** — VoiceOver / TalkBack support is a legal requirement in many markets.
- **No analytics from day one** — instrument screens and key actions immediately; retro-fitting loses historical data.
