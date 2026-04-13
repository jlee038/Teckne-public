# Teckne

An AI-powered image generation and social sharing platform built with Flutter. Users can generate images from text prompts, remix existing posts, and interact with a community feed through reactions, comments, and tags.

---

## What It Does

- **Generate** images from natural language prompts via a real-time WebSocket pipeline with live progress feedback
- **Remix** existing community posts, loads the original image into the generation flow as a starting point
- **Browse** a filterable, paginated community feed (by time range, content type, and tags)
- **React and comment** on posts; tag posts with searchable labels
- **Watermark** downloaded images optionally via in-app settings
- **Authenticate** with OAuth providers (Google, Apple, etc.) via Supabase Auth

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Flutter (Material 3, dark theme) |
| State management | Riverpod 2 with code generation (`riverpod_annotation`) |
| Routing | GoRouter with `ShellRoute` and auth-driven redirects |
| HTTP client | Dio with a custom auth interceptor and 401-only retry |
| Real-time | WebSocket (`web_socket_channel`) for the image generation pipeline |
| Auth & identity | Supabase Auth (PKCE flow, OAuth via `supabase_auth_ui`) |
| Data models | Freezed + `json_serializable` |
| Local storage | `shared_preferences` for settings; `flutter_secure_storage` for credentials |
| Image loading | `cached_network_image` |
| Fonts & icons | Google Fonts, Font Awesome Flutter |

---

## Architecture

The project follows a **feature-first structure** with a shared services and widgets layer:

```
lib/
├── config/           # Compile-time configuration (AppConfig via --dart-define)
├── features/
│   ├── create/       # Image generation & remix (UI, WebSocket repo, Riverpod controllers)
│   ├── login/        # Auth state, OAuth login screen, AppUser model
│   ├── posts/        # Feed, profiles, post/comment/reaction repos and providers
│   └── settings/     # User preferences (watermark toggle)
├── services/
│   ├── api/          # Shared Dio instance, auth interceptor, retry policy
│   ├── router.dart   # GoRouter config, shell scaffold, auth redirects
│   └── websocket/    # Long-lived WebSocketService (Riverpod keepAlive)
├── utils/            # Theme, responsive helpers, alerts
└── widgets/          # Shared UI components (post shell, filters, comments, tags)
```

Each feature contains its own `models/`, `repos/`, `providers/`, `screens/`, and `widgets/` directories. Repositories handle all data access; Riverpod `AsyncNotifier` and `Notifier` classes own state; screens and widgets only read and dispatch.

---

## Notable Implementations

### Auth-Driven Routing

`AppAuthState` is an enum that encodes **which path patterns are allowed** and **where to redirect** for each auth state. A `ValueNotifier<AppAuthState>` is passed to `GoRouter.refreshListenable`, so route guards re-evaluate automatically whenever auth changes. No manual navigation logic in UI code.

### Startup Session Safety

Before the app renders, `main.dart` waits for Supabase to either refresh an expired token or emit `signedOut` (with a timeout). This prevents the first outbound API calls from using a stale JWT without blocking the startup path longer than necessary.

### WebSocket Image Generation Pipeline

The `WebSocketService` is a long-lived Riverpod provider (`keepAlive`) that manages a single connection with 30-second inactivity close and automatic reconnect on next send. The generation flow sends a JSON init message, then streams back string JSON frames (status, progress, errors) and a final binary frame containing the image — handled in `GenerateImageController` and distinguished by payload type.

### Feed Pagination

The home feed uses page-scoped Riverpod providers. A sentinel row at the bottom of the list triggers the next page load via `setState`. Pull-to-refresh invalidates all loaded page providers, causing Riverpod to refetch cleanly without manual list management.

### Platform-Aware Downloads

`DownloadController` branches on `kIsWeb` vs. mobile, resolves the correct save path via `path_provider`, and optionally composites a watermark onto the image before saving using the `image` package.

---

## Configuration

The app reads configuration at compile time via `--dart-define-from-file`. Create a `dart_defines.json` with:

```json
{
  "SUPABASE_URL": "https://your-project.supabase.co",
  "SUPABASE_ANON_KEY": "your-anon-key",
  "API_BASE_URL": "https://your-api.example.com",
  "WS_BASE_URL": "wss://your-api.example.com"
}
```

Then run:

```bash
flutter run --dart-define-from-file=dart_defines.json
```

`AppConfig.isConfigured` will throw at startup if the required keys are missing, failing fast rather than producing cryptic runtime errors.

---

## Running the Project

```bash
# Install dependencies
flutter pub get

# Generate Freezed models and Riverpod providers
dart run build_runner build --delete-conflicting-outputs

# Run (requires dart_defines.json and a running backend)
flutter run --dart-define-from-file=dart_defines.json
```

---

## What I'd Do Next

- Expand post types (video, music)
- Abstract the WebSocket protocol behind a typed event interface to make it easier to add new message types
- Create logic for a suggested content feed based on user's interests and favorited tags. 
