This repository is a small MERN-style app split into two apps: a backend API (server/) and a frontend SPA (client/).

Quick run (dev):
- Backend: cd server; create `.env` with MONGO_URI and JWT_SECRET; npm install; npm run dev (nodemon src/index.js)
- Frontend: cd client; npm install; npm run dev (vite)
- Backend base: http://localhost:4000
- Frontend base: http://localhost:5000
- API base path: /api (see `client/src/api.js` where axios baseURL is configured)

Big-picture architecture and important files:
- server/src/index.js -> connects DB and starts the server (ES modules)
- server/src/app.js -> express app, cors/morgan/json middleware, routes mounted at `/api/*`, centralized error handler
- server/src/routes/* -> route wiring (auth.js, perks.js)
- server/src/controllers/* -> controllers implement request handling and return JSON shapes (important: they return objects like `{ perks }`, `{ perk }`, `{ token, user }`)
- server/src/models/* -> Mongoose models; Perk has unique compound index on `{ title, merchant }` (see `Perk.js`)
- server/src/middleware/auth.js -> JWT-based requireAuth middleware (routes under `/api/perks` use it for protected endpoints)

API & data shapes (explicit examples agents should preserve):
- GET /api/perks/all -> public endpoint; returns { perks: [...] }
  Example: GET /api/perks/all?search=coffee&merchant=Cafe
- GET /api/perks (protected) -> returns { perks } for the logged-in user's perks
- GET /api/perks/:id -> returns { perk }
- POST /api/perks (protected) -> returns { perk } on create; duplicate index errors return code 11000
- Auth routes: POST /api/auth/login and POST /api/auth/register -> return { token, user }

Frontend patterns and conventions:
- Client is a Vite + React app in `client/` using ESM. Routes are defined in `client/src/App.jsx` (react-router v6).
- API client: `client/src/api.js` exports an axios instance with baseURL `http://localhost:4000/api` and an interceptor that adds `Authorization: Bearer <token>` using `localStorage.getItem('token')`.
- Auth: `client/src/context/AuthContext.jsx` bootstraps the logged-in user by calling GET /auth/me and stores token in localStorage. Use `useAuth()` (re-exported in `client/src/hooks/useAuth.js`).
- UI data shapes expect the controller responses above. Example: components call `api.get('/perks/all', { params: { search, merchant } })` and read `res.data.perks` (see `client/src/pages/AllPerks.jsx`).

Project-specific rules an agent must respect:
- Use existing JSON shapes from controllers: don't change `{ perks }` or `{ perk }` envelopes without updating the callers in `client/src/`.
- Server uses ES module syntax (import/export). Follow that style for new server files.
- Validation: controllers use Joi for input validation. Keep validation close to controller logic (see `server/src/controllers/perkController.js`).
- Authentication: handlers that require authentication rely on `requireAuth` middleware to populate `req.user`. Protected controllers expect `req.user.id` and will 401 if absent.
- Mongoose models may have unique indexes (Perk). Handle duplicate-key (11000) in server controller error handling if you add new DB writes.

Where to look for examples when implementing features:
- Search & filter implementation (backend): `server/src/controllers/perkController.js` already implements search/merchant filtering for `/api/perks/all`. Use its query shape (regex on title, merchant exact match).
- Frontend usage example: `client/src/pages/AllPerks.jsx` already contains a mostly-complete UI and a function `loadAllPerks()` that calls `/perks/all` with params. Implement missing input handlers and initial data loading there.
- Auth usage: `client/src/context/AuthContext.jsx` and `client/src/api.js` show how tokens are stored and attached.

Dev & test notes for agents:
- Backend dev: `npm run dev` in `server/` (nodemon). Node must support ES modules (the project uses "type": "module").
- Frontend dev: `npm run dev` in `client/` (vite). The client expects the backend at http://localhost:4000; CORS is configured in `server/src/app.js` to allow the client origin.
- Tests: client contains Vitest config; server lists jest but no test harness present. Add tests to `client/` using vitest (if adding server tests, follow ES module jest config or switch to vitest for parity).

Editing guidance for AI agents:
- Small changes only: prefer editing existing controllers/routes/components, don't re-architect. Cite and preserve response shapes.
- For new endpoints or UI behavior, include a quick manual test plan in the PR: commands to run both apps and a sample request (example above).
- When changing authentication behavior, update `client/src/api.js` and `client/src/context/AuthContext.jsx` together.

If you find an existing `.github/copilot-instructions.md` or AGENT.md in this repo, merge preserved content instead of replacing it; otherwise add concise facts like above.

If anything above is unclear or you'd like more examples (e.g., request/response logs, specific files to open), tell me which area to expand and I'll iterate.
