# Collaborative Text Editor with AI Assistant

This repository contains the implementation for the WorkRadius AI Technologies SDE Intern assignment: a full-stack, real-time collaborative text editor with integrated Google Gemini AI assistance and AWS deployment.

## Project Overview
- **Frontend:** React (Vite) with Quill.js rich text editor, Socket.io client, AI assistant panel.
- **Backend:** Node.js (Express, TypeScript) with JWT auth, Socket.io, Mongoose/MongoDB.
- **AI Integration:** Google Gemini API via backend service layer with rate limiting and caching.
- **Deployment:** Dockerized services orchestrated on AWS EC2 behind Nginx with HTTPS.

For detailed architecture, refer to `docs/architecture.md`.

## Roadmap
1. Scaffold backend and frontend applications, establish shared tooling (ESLint, Prettier, Husky).
2. Implement authentication & authorization with JWT and secure cookies.
3. Build document management APIs with autosave and sharing permissions.
4. Add real-time collaboration (Socket.io) for text changes, cursor presence, and save notifications.
5. Integrate Gemini AI endpoints for grammar, enhancement, summarization, auto-complete, and suggestions.
6. Harden security (rate limiting, sanitization, HTTPS) and add monitoring/logging.
7. Implement testing (unit, integration, WebSocket) and set up CI checks.
8. Containerize and deploy to AWS EC2 with SSL and environment management.
9. Produce learning log, documentation, and demo video.

## Repository Status
- Architecture plan drafted (`docs/architecture.md`).
- Implementation scaffolding in progress.

## Getting Started (to be expanded)
Instructions for local development, environment variables, and deployment will be added as the project scaffolding progresses.