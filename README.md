# Teckne

An AI-powered image generation and social sharing platform. Users can generate images from text prompts, remix existing posts, and interact with a community feed through reactions, comments, and tags.

This projects constists of three repositories:

- **Teckne/** -- Flutter client (iOS, Android, Web)
- **Teckne-server/** -- Node.js/Fastify REST API and WebSocket proxy
- **ai-server/** -- Python/FastAPI GPU inference server (FLUX diffusion model)

---

## What It Does

- **Generate** images from natural language prompts via a real-time WebSocket pipeline with live progress feedback
- **Remix** existing community posts, loads the original image into the generation flow as a starting point
- **Browse** a filterable, paginated community feed (by time range, content type, and tags)
- **React and comment** on posts; tag posts with searchable labels
- **Watermark** downloaded images optionally via in-app settings
- **Authenticate** with OAuth providers (Google, Apple, etc.) via Supabase Auth

---

## System Architecture

```
┌──────────────┐     REST / WebSocket      ┌─────────────────────┐
│  Flutter App │ ─────────────────────────► │   Teckne-server     │
│  (Dart)      │                            │   (Node.js/Fastify) │
└──────────────┘                            └────────┬────────────┘
                                                     │
                                    ┌────────────────┼────────────────┐
                                    │                │                │
                                    ▼                ▼                ▼
                             ┌────────────┐  ┌────────────┐  ┌────────────────┐
                             │  Supabase  │  │  Supabase  │  │   ai-server    │
                             │  Postgres  │  │  Storage   │  │ (Python/FLUX)  │
                             └────────────┘  └────────────┘  └────────────────┘
```

The Flutter app never connects to the GPU server directly. Teckne-server owns the shared secret and acts as the WebSocket proxy, so the AI server is never exposed to clients. Auth is enforced at the API layer by verifying Supabase JWTs before any database or storage operation.

---

## Tech Stack

### Flutter Client (`Teckne/`)

| Layer | Technology |
|---|---|
| Framework | Flutter (Material 3, dark theme) |
| State management | Riverpod 2 with code generation (`riverpod_annotation`) |
| Routing | GoRouter with `ShellRoute` and auth-driven redirects |
| HTTP client | Dio with a custom auth interceptor and 401-only retry |
| Real-time | WebSocket (`web_socket_channel`) for the image generation pipeline |
| Auth and identity | Supabase Auth (PKCE flow, OAuth via `supabase_auth_ui`) |
| Data models | Freezed + `json_serializable` |
| Local storage | `shared_preferences` for settings; `flutter_secure_storage` for credentials |
| Image loading | `cached_network_image` |
| Fonts and icons | Google Fonts, Font Awesome Flutter |

### API Server (`Teckne-server/`)

| Layer | Technology |
|---|---|
| Runtime | Node.js (ES modules) |
| Framework | Fastify v5 |
| Auth | JWT verification via `jsonwebtoken` against Supabase JWT secret |
| Database and storage | Supabase (Postgres RPC + Storage), user JWT forwarded per request |
| Content moderation | TensorFlow.js toxicity model, loaded on startup |
| WebSocket proxy | `ws` client to ai-server; `@fastify/websocket` for client connections |
| Security | `@fastify/helmet`, `@fastify/cors`, `@fastify/rate-limit` (100 req / 15 min) |
| File uploads | `@fastify/multipart`, 50MB cap |

### AI Inference Server (`ai-server/`)

| Layer | Technology |
|---|---|
| Runtime | Python 3.10+ |
| Framework | FastAPI + Uvicorn |
| Model | `black-forest-labs/FLUX.2-klein-4B` via Hugging Face `diffusers` |
| Compute | PyTorch with CUDA (bfloat16); CPU offload if VRAM < 12GB |
| Prompt generation | `succinctly/text2image-prompt-generator` for empty prompts |
| Concurrency | `asyncio` event loop + `run_in_executor` for blocking inference |

---

## Flutter Client Architecture

The client follows a **feature-first structure** with a shared services and widgets layer:

```
lib/
├── config/           # Compile-time configuration (AppConfig via --dart-define)
├── features/
│   ├── create/       # Image generation and remix (UI, WebSocket repo, Riverpod controllers)
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

## API Server Architecture

```
src/
├── server.js          # Process entry, graceful shutdown, toxicity model init
├── app.js             # Plugin registration, routes, global error handler
├── config/
│   └── constants.js
├── middleware/
│   ├── setup.js       # CORS, Helmet, rate limiting
│   └── auth.js        # JWT verification, Supabase client per request
├── routes/
│   ├── posts.js       # Feed queries, post creation and deletion
│   ├── comments.js    # Comment creation and pagination
│   ├── images.js      # Multipart upload and deletion
│   ├── reactions.js   # Toggle reactions via Supabase RPC
│   ├── tags.js        # Popular tags and tag search
│   ├── users.js       # Username to user ID resolution
│   └── websocket.js   # WebSocket proxy to ai-server
└── services/
    └── toxicityCheck.js
```

---

## Notable Implementations

### Auth-Driven Routing (Flutter)

`AppAuthState` is an enum that encodes **which path patterns are allowed** and **where to redirect** for each auth state. A `ValueNotifier<AppAuthState>` is passed to `GoRouter.refreshListenable`, so route guards re-evaluate automatically whenever auth changes. No manual navigation logic in UI code.

### Startup Session Safety (Flutter)

Before the app renders, `main.dart` waits for Supabase to either refresh an expired token or emit `signedOut` (with a timeout). This prevents the first outbound API calls from using a stale JWT without blocking the startup path longer than necessary.

### WebSocket Image Generation Pipeline (All Three)

The generation flow spans all three services. The Flutter client sends a JSON init message over its connection to Teckne-server; Teckne-server proxies the traffic to ai-server after injecting the shared secret. The ai-server streams back JSON status frames (queued, progress with step count, errors) and a final binary JPEG payload. The Flutter `GenerateImageController` distinguishes binary vs. string frames by payload type and updates UI state accordingly.

The `WebSocketService` (Flutter) and the ai-server both use a **30-second inactivity timeout**, closing the connection if no messages are exchanged. Teckne-server validates that client-sent message `type` fields are within an allowed set before forwarding, so arbitrary messages cannot reach the inference server.

### Serialized Inference with Cancellation (ai-server)

The ai-server allows only one generation at a time per process. Additional connections receive a `status: queued` response and wait on `_generation_lock`. Cancellation sets `pipe._interrupt` on the diffusers pipeline, which is checked at each diffusion step boundary. After cancellation, the server runs `torch.cuda.synchronize` and `empty_cache` to reclaim VRAM before accepting the next job.

Progress callbacks use `asyncio.run_coroutine_threadsafe` to push step updates from the executor thread back to the async WebSocket handler without blocking inference.

### Content Moderation (Teckne-server)

Automatic content moderation on titles, descriptions, tags, meme text, and comment text to filter out inappropriate content before any write reaches Supabase. A toxicity check model is loaded asynchronously after the HTTP server starts. On model load failure, `checkToxicity` fails closed, blocking the write rather than silently skipping the check.

### Storage Security (Teckne-server)

Post deletion fetches the `content_url` from the database server-side before removing the storage object, so clients cannot supply an arbitrary path to delete. Image deletion on orphaned uploads verifies that the storage path begins with the requesting user's `user_id` prefix before the storage operation runs.

### Feed Pagination (Flutter)

The home feed uses page-scoped Riverpod providers keyed by page number. A sentinel row at the bottom of the list triggers the next page load via `setState`. Pull-to-refresh invalidates all loaded page providers, causing Riverpod to refetch cleanly without manual list management.

### Platform-Aware Downloads (Flutter)

`DownloadController` branches on `kIsWeb` vs. mobile, resolves the correct save path via `path_provider`, and optionally composites a watermark onto the image before saving.

---

## Configuration

### Flutter (`Teckne/dart_defines.json`)

```json
{
  "SUPABASE_URL": "https://your-project.supabase.co",
  "SUPABASE_ANON_KEY": "your-anon-key",
  "API_BASE_URL": "https://your-api.example.com",
  "WS_BASE_URL": "wss://your-api.example.com"
}
```

### API Server (`Teckne-server/.env`)

```
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key
SUPABASE_JWT_SECRET=your-jwt-secret
CORS_ORIGIN=http://localhost:PORT
WEBSOCKET_URL=ws://localhost:8000/ws/generate-image
AI_SERVER_SECRET=your-shared-secret
PORT=3000
```

### AI Server (`ai-server/.env`)

```
AI_SERVER_SECRET=your-shared-secret
```

---

## Running the Project

### AI Server

```bash
cd ai-server
pip install -r requirements.txt
python verify_setup.py   # checks CUDA, torch, and dependencies
python flux_server.py
```

### API Server

```bash
cd Teckne-server
npm install
node src/server.js
```

### Flutter Client

```bash
cd Teckne
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter run --dart-define-from-file=dart_defines.json
```

---

## What I'd Do Next

- Expand post types (video, music)
- Abstract the WebSocket protocol behind a typed event interface to make it easier to add new message types
- Create logic for a suggested content feed based on user's interests and favorited tags
