## Collaborative Text Editor with AI Assistant – Architecture Overview

### 1. Goals & Non-Goals
- Deliver a production-ready baseline for a multi-user collaborative editor with AI-assisted writing.
- Focus on reliable real-time sync, secure document access, and extensible AI integrations.
- Non-goals: offline-first editing, exhaustive formatting features, or native mobile clients (future scope).

### 2. High-Level Architecture
- **Client (React + Vite)**
  - Rich text editor (Quill.js) with custom collaboration layer.
  - Context-aware AI assistant panel for grammar, enhancement, summarization, auto-complete, suggestions.
  - Auth flows (register/login/logout, role selection) with protected routes.
- **API Server (Express + TypeScript)**
  - REST endpoints for auth, documents, AI services.
  - Socket.io server for collaborative editing events.
  - Business services modularized (auth, documents, AI, notifications).
- **Database (MongoDB Atlas/replica on EC2)**
  - Collections: `users`, `documents`, `documentVersions`, `permissions`, `sessions`, `aiLogs`.
- **AI Integration (Google Gemini)**
  - Service wrapper with request throttling, caching recent prompts, and transparent fallbacks (e.g., degrade gracefully).
- **Infrastructure (AWS EC2 + Docker Compose)**
  - Reverse proxy (Nginx) terminating TLS, forwarding to Node/Express and Vite build.
  - PM2 or systemd for process supervision inside container if needed; prefer Docker orchestration.
  - GitHub Actions pipeline for CI lint/test/build and redeploy (stretch goal for automation).

### 3. Detailed Component Design
#### 3.1 Backend Modules (`/server`)
- `config/`
  - `env.ts`: typed environment variables, schema validation (zod).
  - `database.ts`: Mongo connection via Mongoose.
  - `jwt.ts`: sign/verify helpers with refresh/access token support.
  - `gemini.ts`: API client initialization, rate limits.
- `models/`
  - `User`: fields (email, passwordHash, name, roles, status, createdAt, updatedAt).
  - `Document`: title, content, owner, collaborators, version, timestamps, isShared.
  - `Permission`: user, document, accessLevel (`owner|editor|viewer`), invitationToken, expiresAt.
  - `DocumentVersion`: version number, snapshot, createdAt, performedBy.
- `routes/`
  - `auth.route.ts`: register/login/logout/me/refresh.
  - `document.route.ts`: CRUD, share, permissions.
  - `ai.route.ts`: grammar-check/enhance/summarize/complete/suggestions.
- `middleware/`
  - `authenticate`: validates JWT, loads user.
  - `authorize`: checks roles/permissions.
  - `rateLimiter`: per IP + per user.
  - `validate`: request schema validation using Zod/Joi.
  - `errorHandler`: centralized error response with logging.
- `services/`
  - `auth.service.ts`: registration, hashing (bcrypt), token issuing, refresh flow.
  - `document.service.ts`: CRUD, versioning, collaborator management, autosave.
  - `ai.service.ts`: orchestrates Gemini requests, caching, fallback messaging.
- `websockets/`
  - `document.socket.ts`: handles joining rooms, broadcasting `text-change`, `cursor-move`, presence, persisted snapshots every N ops.
  - `ai.socket.ts`: optional streaming suggestions for real-time AI features.
- `utils/`
  - Logging (pino/winston), response helpers, permission checks, sanitization.

#### 3.2 Frontend Structure (`/client/src`)
- `components/`
  - `Editor`: wraps Quill with custom modules for presence + AI suggestions overlay.
  - `ChatPanel`: displays AI interactions, suggestions, status.
  - `DocumentList`, `AuthForms`, `Navbar`, `ShareDialog`.
- `hooks/`
  - `useAuth`, `useDocuments`, `useSocket`, `useAI` for modular state.
- `services/`
  - `apiClient` (axios with interceptors for JWT refresh), `socketClient`.
  - `aiService` calling backend endpoints.
- `context/`
  - `AuthProvider`, `DocumentProvider`, `PresenceContext`.
- `routes/`
  - Protected routes using React Router, lazy loaded pages.
- `styles/`
  - TailwindCSS or CSS-in-JS (decide later). Default to Tailwind for speed.

### 4. Data Flow
1. **Authentication**
   - User registers/login → backend issues access + refresh tokens (HTTP-only cookie for refresh, short-lived access in memory).
   - Frontend stores access token in memory (React Query), refresh via silent endpoint when expired.
2. **Document Collaboration**
   - On selecting document, client fetches latest snapshot via REST.
   - Client connects to Socket.io room `document:<id>`; server authenticates handshake.
   - Text changes captured by Quill → delta diffs sent via `text-change`; server rebroadcasts after conflict resolution (operational transform via sharedb-like custom or use Quill delta + sequential operations). We'll implement using simple sequential queue with server applying delta to authoritative state (persisted every 30s or on manual save).
   - Cursor positions & presence broadcasted without persistence.
3. **AI Interactions**
   - Client sends selected text via REST `POST /api/ai/...`; server proxies to Gemini with request throttling + caching.
   - Optionally stream results via WebSocket events `ai-processing`, `ai-suggestions-ready`.

### 5. Security Considerations
- Input validation and sanitization using Zod.
- Escape/sanitize HTML content before rendering; rely on Quill delta to mitigate XSS, plus server sanitization (DOMPurify in backend).
- JWT stored securely: refresh token in HTTP-only cookie, access token in memory + Authorization header.
- Rate limiting: express-rate-limit + Redis (future) or in-memory for MVP.
- Helmet for HTTP headers, CORS restricted to known origins, CSRF protection for critical endpoints (share operations).
- WebSocket authentication via JWT handshake.

### 6. Deployment Strategy
- **Docker**
  - Multi-stage build for backend (Node 18) and frontend (Node 18 + build static) combined via docker-compose.
  - Services: `client`, `server`, `nginx`, `mongo` (for local dev). Production uses managed MongoDB (Atlas) or self-hosted with persistent volume.
- **AWS EC2**
  - Provision t3.small (or t2.micro for demo) with Ubuntu 22.04.
  - Install Docker & docker-compose plugin.
  - Setup security groups: allow 22 (restricted IP), 80/443, optionally 3000/4000 for debugging.
  - Use Let's Encrypt (Certbot) with Nginx container terminating SSL.
  - Environment secrets through `.env` files injected via AWS SSM Parameter Store (optional) or `.env.production` + Docker secrets.
- **CI/CD (Stretch)**
  - GitHub Actions: run lint/test/build on PR, deploy via SSH or AWS CodeDeploy.

### 7. Testing Approach
- **Backend**
  - Jest + Supertest for REST endpoints.
  - Socket.io-client for WebSocket tests (join, text-change, presence).
  - Mock Gemini API using nock.
- **Frontend**
  - React Testing Library for components (Auth forms, Editor). Integration tests for AI panel.
- **End-to-End (Stretch)**
  - Playwright/Cypress to validate collaborative session.

### 8. Roadmap (High-Level)
1. Scaffold repository: initialize backend & frontend, shared config, linting.
2. Implement auth (register/login/me/logout) with JWT.
3. Build document CRUD + permissions (owner/editor/viewer), autosave.
4. Add Socket.io-based collaboration (text-change, cursor, presence).
5. Integrate Gemini AI endpoints + frontend UI.
6. Harden security (rate limiting, sanitization, HTTPS).
7. Write unit/integration tests.
8. Containerize and deploy to AWS EC2 with Nginx + SSL.
9. Prepare README, learning log, demo video script.

### 9. Future Enhancements
- Offline editing with CRDTs (Yjs/Automerge).
- Document commenting, chat, and tracked changes.
- Rich role management (teams, organizations), audit logs.
- AI fine-tuning per-document context, prompt templates.
- Collaborative cursors with avatars and chat integration.

