# Video Platform — Design & Spec

## 1. Overview

A front/back-separated video hosting & playback platform (B站风格), built from scratch.

- **Backend**: Spring Boot 3.3 (Java 17), Gradle build, MyBatis-Plus, MySQL (default) or SQLite.
- **Frontend**: React 18 + TypeScript + Vite, custom dark theme, video.js/hls.js player.
- **Auth**: JWT (access + refresh), BCrypt password hashing (per-user salt / 盐加密), image captcha login & register.
- **Video**: chunked/resumable 4K upload, HTTP Range streaming for direct 4K playback, optional ffmpeg → HLS adaptive transcode.
- **Storage**: pluggable `StorageService` — LocalDisk or MinIO (S3-compatible), toggled by config.
- **Ops**: unified `R<T>` response, global exception handler, configurable log level via `data/config/application.yml`.

## 2. Architecture

```
video-platform/
├── backend/                Spring Boot app (Gradle)
│   ├── data/config/application.yml     ← single unified external config file
│   └── src/main/java/com/videoplatform/
│       ├── VideoPlatformApplication.java
│       ├── common/                R<T>, ResultCode, BusinessException, GlobalExceptionHandler
│       ├── config/                MybatisPlusConfig, CorsConfig, AppProperties, SecurityConfig
│       ├── security/              JwtUtil, JwtAuthenticationFilter, UserDetailsServiceImpl, CaptchaService
│       ├── module/auth/           AuthController, AuthService, DTOs
│       ├── module/user/           User entity/mapper/service/controller
│       ├── module/video/          Video & UploadTask entity/mapper/service/controller
│       └── storage/               StorageService, LocalStorageService, MinioStorageService, StorageFactory
├── frontend/               React SPA (Vite)
│   └── src/  api/ pages/ components/ styles/
└── docs/
```

## 3. Backend Design

### 3.1 Config (`data/config/application.yml`)

A single external YAML that holds **everything**: db, storage, jwt, logging, video/ffmpeg.

| Key | Purpose |
|---|---|
| `platform.db.type` | `mysql` or `sqlite` — must match `spring.datasource` |
| `spring.datasource.*` | JDBC url / username / password / driver |
| `storage.type` | `local` or `minio` |
| `storage.local.base-path` | directory for local storage |
| `storage.minio.*` | endpoint, bucket, access/secret key |
| `jwt.secret`, `jwt.access-expire-min`, `jwt.refresh-expire-day` | token settings |
| `logging.level.*` | log level (read via logback `${LOG_LEVEL:-INFO}`) |
| `video.hls.enabled`, `video.hls.ffmpeg-path`, `video.hls.workDir` | optional transcode |

Logging level is wired into `logback-spring.xml` through `logging.level.root` → property, so a single edit in the config file changes it.

### 3.2 Auth flow

- **Register**: `username + password (+ captcha)`. Password hashed with BCrypt (per-user salt). Username unique.
- **Captcha**: `GET /api/auth/captcha` returns `{ captchaId, image(base64) }`; stored in-memory cache with TTL (~2 min), one-time use. `POST /api/auth/login` validates & consumes it.
- **Login**: validates captcha + credentials → returns `{ accessToken, refreshToken, user }`.
- **Refresh**: `POST /api/auth/refresh` with refresh token → new access token.
- **JWT filter** reads `Authorization: Bearer <token>`, validates, populates `SecurityContext`; invalid tokens are ignored (public endpoints work).

### 3.3 Storage

`StorageService` interface: `upload / getObject / getObjectRange / getSize / delete / exists / getPublicUrl`.
`StorageFactory` picks `LocalStorageService` or `MinioStorageService` from `storage.type`.

### 3.4 Video upload & streaming

- **Chunked upload**: browser slices the 4K file; `POST /api/videos/upload/init` → returns `taskId`; `POST /api/videos/upload/{taskId}/{index}` uploads each chunk; `POST /api/videos/upload/{taskId}/complete` merges chunks + writes metadata.
- **Range streaming**: `GET /api/videos/{id}/stream` supports `Range` → `206 Partial Content` with `Content-Range`, enabling native 4K playback & seeking.
- **HLS (optional)**: if `video.hls.enabled`, a background worker runs ffmpeg to produce `*_hls/index.m3u8` + segments, served statically; frontend uses hls.js when present.
- Public: `list / detail / stream / cover`. Authenticated: `upload / delete`.

### 3.5 Global exception handling

`@RestControllerAdvice` maps `BusinessException`, `MethodArgumentNotValidException`, `AccessDeniedException`, `BadCredentialsException`, and fallback `Exception` → unified `R<T>{ code, message, ... }`, with stack trace logged at the configured level.

## 4. Frontend Design

- **Routing** (react-router): `/login`, `/register`, `/` (home grid), `/video/:id` (player), `/upload` (protected).
- **API client**: axios base URL from `VITE_API_BASE_URL` (dev proxy `/api` → `:8080`); request interceptor adds JWT; response interceptor auto-refreshes on 401.
- **Player**: `video.js` + `hls.js` for HLS; native `<video>` with Range-streaming MP4 otherwise. Controls + 4K-aware UI.
- **Upload**: chunked, progress bar, resumable-friendly.
- **Theme**: B站-dark; hand-written CSS (no heavy UI lib).

## 5. Data Model

**users**: id, username (unique), password (BCrypt), email, created_at, updated_at.
**videos**: id, title, description, object_key, file_name, file_size, mime_type, width, height, duration_seconds, cover_url, status, uploader_id, created_at, updated_at.
**upload_task**: id, task_id (unique), file_name, file_size, chunk_size, total_chunks, uploaded_chunks, status, object_key, owner_id, created_at.

SQLite uses `INTEGER PRIMARY KEY AUTOINCREMENT`; MySQL uses `BIGINT AUTO_INCREMENT` — two schema files, selected via the config.

## 6. Build & Run

- Backend: `cd backend && ./gradlew bootRun` (config at `backend/data/config/application.yml`).
- Frontend: `cd frontend && npm install && npm run dev` (dev server `:5173`, proxies `/api` to `:8080`).
- Default login: a seed user is created on startup (`admin` / `admin123`) when `platform.seed.enabled=true` (for dev convenience).

## 7. Verification

1. `./gradlew build` compiles the backend.
2. `vite build` builds the frontend.
3. Smoke-test the auth + video endpoints if the servers can run in this environment.
