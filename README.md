# MusicHub — Spotify Music Dashboard

A production-ready full-stack web application that connects your Spotify Music account and displays your music library in a beautiful, modern interface.

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | React 18 · TypeScript 5 · Vite · Tailwind CSS · Framer Motion |
| State | Zustand · TanStack Query v5 |
| Backend | Spring Boot 3.2 · Java 17 · Spring Security |
| Auth | Yandex OAuth2 · JWT (access + refresh) |
| Database | PostgreSQL 16 · Flyway migrations |
| Cache | Caffeine (5-min TTL) |
| Infra | Docker · Docker Compose · nginx |
| Docs | Springdoc OpenAPI / Swagger UI |

## Architecture

```
yandex-music-app/
├── backend/                        # Spring Boot application
│   └── src/main/java/com/musicdashboard/
│       ├── config/                 # Security, WebClient, Cache, OpenAPI
│       ├── controller/             # REST endpoints (Auth, User, Music)
│       ├── service/                # Business logic + Yandex API client
│       ├── repository/             # JPA repositories
│       ├── model/                  # JPA entities (User, OAuthToken)
│       ├── dto/                    # Request / Response DTOs
│       ├── security/               # JWT filter + UserDetails
│       └── exception/              # Global exception handler
│
└── frontend/                       # React SPA
    └── src/
        ├── app/                    # Router · Providers
        ├── features/auth/          # Zustand store · API · hooks · ProtectedRoute
        ├── features/music/         # TanStack Query hooks + API
        ├── pages/                  # Dashboard · Favorites · Artists · Albums · Playlists
        ├── widgets/                # Sidebar · Header
        └── shared/                 # API client · UI components · types · hooks
```

## Prerequisites

- Docker & Docker Compose (for containerized setup)
- OR: Java 17, Node 20, PostgreSQL 16 (for local dev)
- A **Yandex OAuth application** — create one at [oauth.yandex.ru](https://oauth.yandex.ru/)
  - Set the callback URL to: `http://localhost:8080/api/auth/yandex/callback`
  - Required scopes: `login:info`, `login:email`

## Quick Start (Docker)

```bash
# 1. Clone and enter the project
git clone <repo-url>
cd yandex-music-app

# 2. Copy and fill in environment variables
cp .env.example .env
# Edit .env: set YANDEX_CLIENT_ID, YANDEX_CLIENT_SECRET, JWT_SECRET

# 3. Generate a JWT secret (must be Base64-encoded, ≥ 256 bits)
openssl rand -base64 64

# 4. Start all services
docker compose up --build

# 5. Open the app
#    Frontend:   http://localhost:3000
#    API docs:   http://localhost:8080/swagger-ui.html
#    Health:     http://localhost:8080/actuator/health
```

## Local Development (without Docker)

### Backend

```bash
cd backend

# Create a PostgreSQL database
createdb musicdashboard

# Set environment variables (or use .env file with spring-dotenv)
export JWT_SECRET=<base64-secret>
export YANDEX_CLIENT_ID=<your-client-id>
export YANDEX_CLIENT_SECRET=<your-client-secret>
export CORS_ALLOWED_ORIGINS=http://localhost:3000
export FRONTEND_URL=http://localhost:3000

# Run
./gradlew bootRun
# → Available at http://localhost:8080
```

### Frontend

```bash
cd frontend
npm install

# Copy and edit the API URL
echo "VITE_API_URL=http://localhost:8080" > .env.local

npm run dev
# → Available at http://localhost:3000
```

## Authentication Flow

```
User clicks "Login"
  → Frontend calls GET /api/auth/yandex/authorize
  → Backend returns Yandex OAuth URL
  → Frontend redirects to Yandex
  → User authenticates, Yandex redirects to backend callback
  → Backend exchanges code → Yandex token → fetches user info
  → Backend creates/updates user in DB, stores Yandex token
  → Backend issues JWT (access + refresh), redirects to /auth/callback
  → Frontend reads tokens from URL, stores in Zustand (persisted)
  → All subsequent API calls use JWT Bearer token
```

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/auth/yandex/authorize` | Get Yandex OAuth URL |
| GET | `/api/auth/yandex/callback` | OAuth callback (called by Yandex) |
| POST | `/api/auth/refresh` | Refresh JWT tokens |
| POST | `/api/auth/logout` | Logout |
| GET | `/api/users/me` | Current user profile |
| GET | `/api/music/dashboard` | Dashboard statistics |
| GET | `/api/music/tracks/liked` | Liked tracks |
| GET | `/api/music/albums/liked` | Liked albums |
| GET | `/api/music/artists/liked` | Liked artists |
| GET | `/api/music/playlists` | User playlists |

Full interactive docs: `http://localhost:8080/swagger-ui.html`

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `JWT_SECRET` | ✅ | Base64 HMAC-SHA256 key (≥256 bits) |
| `YANDEX_CLIENT_ID` | ✅ | Yandex OAuth app client ID |
| `YANDEX_CLIENT_SECRET` | ✅ | Yandex OAuth app client secret |
| `POSTGRES_PASSWORD` | ✅ | Database password |
| `JWT_EXPIRATION` | ❌ | Access token TTL in ms (default: 900000 = 15 min) |
| `JWT_REFRESH_EXPIRATION` | ❌ | Refresh token TTL in ms (default: 604800000 = 7 days) |
| `CORS_ALLOWED_ORIGINS` | ❌ | Frontend origin (default: http://localhost:3000) |
| `FRONTEND_URL` | ❌ | Frontend URL for OAuth redirect (default: http://localhost:3000) |
| `VITE_API_URL` | ❌ | Backend API URL seen by browser (default: empty = same origin) |

## Key Architectural Decisions

**JWT over session cookies** — stateless, works across domains, easier for SPA integration.

**Refresh token rotation** — short-lived access tokens (15 min) + long-lived refresh tokens (7 days) stored in Zustand `persist` middleware. Axios interceptor auto-refreshes on 401.

**Caffeine cache** — Yandex Music API calls are cached for 5 minutes per user. This dramatically reduces latency on navigation since library data changes infrequently.

**WebFlux WebClient** — used for all outbound HTTP (Yandex OAuth + Music API) with configured timeouts and codec size limits. Reactive client blocking call: acceptable here since we have a Servlet-based backend.

**Feature-Sliced Design** — frontend organized into `features/`, `pages/`, `widgets/`, `shared/` layers for scalable, team-friendly structure.

**Flyway migrations** — database schema versioning; `spring.jpa.hibernate.ddl-auto: validate` ensures schema matches entities at startup.
