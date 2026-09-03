# Exam #2: "Social Budget"

## Student: s308461 KHALILINEJAD SEYED ALIREZA

## Technology Stack

This project is a full-stack single-page web application. It is split into two independent Node.js
packages that run as separate processes and talk to each other over HTTP:

- `client/` — the front-end, a React application bundled and served by Vite. In development it runs
  on `http://localhost:5173`.
- `server/` — the back-end, an Express REST API. It runs on `http://localhost:3001` and exposes all
  routes under the `/api` prefix.

The two processes communicate exclusively through JSON over HTTP. Cross-origin requests are allowed
between the two ports by CORS, and the browser sends the session cookie on every request so the
server can identify the logged-in user.

### Front-end (the `client/` package)

- **JavaScript (ES Modules), no TypeScript.** The whole client is written in modern JavaScript.
  Files use the `.jsx` extension when they contain React (JSX) markup and `.mjs` when they are plain
  ES module JavaScript. `package.json` sets `"type": "module"`, so every file is an ES module and
  uses `import` / `export` rather than `require`.
- **React 18 (`react`, `react-dom`).** The UI library. The app is built entirely from function
  components and React hooks (`useState`, `useEffect`). The root of the app is mounted with the
  React 18 `createRoot` API in `src/main.jsx`, and the whole tree is wrapped in `<React.StrictMode>`
  during development to surface unsafe patterns.
- **Vite 5 (`vite`, `@vitejs/plugin-react`).** The build tool and development server. Vite provides
  an instant dev server with Hot Module Replacement (the page updates in the browser as you edit),
  and produces an optimized static bundle for production with `vite build`. `@vitejs/plugin-react`
  adds React Fast Refresh and JSX transformation. Configuration lives in `client/vite.config.js`.
  The available npm scripts are `dev` (start the dev server), `build` (production bundle),
  `preview` (serve the built bundle locally), and `lint`.
- **React Router 6 (`react-router-dom`).** Client-side routing. `<BrowserRouter>` wraps the app in
  `src/main.jsx`, and `src/App.jsx` declares the route table with `<Routes>` / `<Route>`. Navigation
  is done imperatively with the `useNavigate` hook (for example, the app redirects the user to the
  page that matches the current phase) and declaratively with `<Navigate>`.
- **Axios (`axios`).** The HTTP client used to call the back-end API. A single pre-configured
  instance is created in `src/API/API.mjs` with `baseURL: "http://localhost:3001/api"` and
  `withCredentials: true` so the browser always attaches the session cookie. Every network call the
  UI makes goes through the thin wrapper functions in this file (login, logout, fetch session,
  fetch/change phase, set budget, CRUD on proposals, submit scores, reset process).
- **React-Bootstrap 2 (`react-bootstrap`) + Bootstrap 5 (`bootstrap`).** The component and styling
  system. Bootstrap 5 supplies the CSS design system (grid, spacing, forms, tables, buttons);
  React-Bootstrap provides those same widgets as ready-made React components so no jQuery is needed.
  The Bootstrap stylesheet and JS bundle are imported once in `src/main.jsx`.
- **Bootstrap Icons (`bootstrap-icons`).** The icon set, imported as a font/CSS in `src/main.jsx`
  and used via `<i className="bi bi-...">` elements.
- **Day.js (`dayjs`).** A small (~2 KB) library for parsing, formatting, and comparing dates,
  used wherever the UI needs to display or manipulate dates.
- **prop-types (`prop-types`).** Runtime type-checking of the props passed to React components,
  used as lightweight documentation and validation in place of TypeScript.
- **ESLint 8 (`eslint` + `eslint-plugin-react`, `eslint-plugin-react-hooks`,
  `eslint-plugin-react-refresh`).** Static analysis / linting for the client. The configuration is
  in `client/.eslintrc.cjs` and is run with `npm run lint`. The React Hooks plugin enforces the
  rules of hooks; the React Refresh plugin warns about patterns that break Fast Refresh.

### Back-end (the `server/` package)

- **Node.js with ES Modules.** The server is plain Node.js. Every source file uses the `.mjs`
  extension and native `import` / `export`. The entry point is `server/index.mjs`
  (`"main": "index.mjs"` in `package.json`).
- **Express 4 (`express`).** The web framework. `index.mjs` creates the app, registers middleware,
  and mounts the API router (`server/routes.mjs`) under `/api`. `express.json()` parses incoming
  JSON request bodies. The router uses `express.Router()` and `async` route handlers that call the
  data-access layer and translate errors into `500` responses.
- **CORS (`cors`).** Middleware that allows the browser running the client on
  `http://localhost:5173` to call the API on port `3001`. It is configured with
  `origin: "http://localhost:5173"` and `credentials: true` so cookies are accepted cross-origin.
- **express-session (`express-session`).** Server-side session management. On successful login a
  session is created and its id is stored in a cookie on the browser; subsequent requests are
  authenticated by that cookie. Configured with `resave: false` and `saveUninitialized: false`
  (sessions are stored in memory, which is fine for this exam project).
- **Passport (`passport`) with the Local strategy (`passport-local`).** Authentication. The strategy
  is defined in `server/passport/passport.mjs`: it takes a username and password, looks the user up,
  and verifies the password. `passport.serializeUser` / `deserializeUser` store the whole user
  object in the session. `passport.initialize()`, `passport.session()`, and
  `passport.authenticate("session")` are wired into the middleware chain in `index.mjs`.
- **Node.js `crypto` (built-in) — `scrypt`.** Password hashing. `server/dao/authDao.mjs` hashes the
  submitted password with `crypto.scrypt` using the per-user `salt` stored in the database, then
  compares it to the stored `password_hash` with `crypto.timingSafeEqual` to avoid timing attacks.
  Passwords are never stored in plain text.
- **SQLite via `sqlite3` and `sqlite`.** The database. `sqlite3` is the low-level driver;
  `server/db/db.mjs` opens the file `server/db/database.db` and exports a shared connection. The
  data-access objects (`server/dao/dao.mjs`, `server/dao/authDao.mjs`) contain all the SQL and wrap
  the callback-style driver calls in Promises so the route handlers can `await` them. The schema is
  four tables: `users`, `app_phases`, `proposals`, `app_score` (described below).
- **Morgan (`morgan`).** HTTP request logging middleware, used for development visibility into
  incoming requests.
- **Day.js (`dayjs`).** Same date library as the client, available on the server for any
  date handling.

### Architecture summary

| Layer            | Technology                                             | Responsibility                                    |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------- |
| UI               | React 18, React-Bootstrap, Bootstrap 5, Bootstrap Icons | Render pages and forms, handle user interaction   |
| Client routing   | React Router 6                                          | Map URLs to page components, redirect by phase    |
| Client build     | Vite 5 + `@vitejs/plugin-react`                         | Dev server, HMR, production bundle                |
| Client ↔ Server  | Axios over HTTP/JSON, cookies (`withCredentials`)       | Call the REST API, carry the session cookie       |
| API              | Express 4, CORS, Morgan                                 | REST endpoints under `/api`, request logging      |
| Auth             | Passport (local strategy), express-session, `crypto.scrypt` | Login, session cookies, salted password hashing |
| Data access      | `sqlite` / `sqlite3`, Promise-wrapped DAO modules       | SQL queries, map rows to objects                  |
| Storage          | SQLite (`server/db/database.db`)                        | Persist users, phases/budget, proposals, scores   |
| Tooling          | ESLint 8, npm scripts                                   | Linting and project scripts                       |

### How to run

```
# Terminal 1 — back-end
cd server
npm install
node index.mjs          # starts the API on http://localhost:3001

# Terminal 2 — front-end
cd client
npm install
npm run dev             # starts Vite on http://localhost:5173
```

Open `http://localhost:5173` in the browser. The client will call the API on port `3001`.

## React Client Application Routes

- Route `/`: Home page for users without logging in that show the phase number for 0 to 2 and for phase 3 shows approved proposals.
- Route `/login`: use for logging in
- Route `/phase-0`: Defining the budget by admin
- Route `/phase-1`: propose proposals based on the budget for every logged in users
- Route `/phase-2`: User can rate others proposals (admin and members)
- Route `/phase-3`: show result of the voting and proposals

## Main React Components

- `Header` (in `Header.jsx`): Header of every page except login, It has login or logout button and show logged in username.
- `Home` (in `Home.mjs`): Component fetching current phase and displaying approved proposals in a table if phase is 3, or a welcome message otherwise.
- `Login` (in `Login.mjs`): Component for logging in users.
- `PhaseZero` (in `PhaseZero.mjs`): Component to set budget by admin. Others only see a text.
- `PhaseOne` (in `PhaseOne.mjs`): Component to set Proposals by users(admin and members)
- `PhaseTwo` (in `PhaseTwo.mjs`): Component to rate proposals by users(admin and members).
- `PhaseThree` (in `PhaseThree.mjs`): Component to show approved and unapproved proposals to everyone.

(only _main_ components, minor ones may be skipped)

## API Server

### Authentication Routes

- POST `/api/auth/login`: Authenticate a user using local strategy.
  - request parameters and request body content: `{ username: string, password: string }`
  - response body content: `{ user: object }` on success
  - Status Codes: `200 OK` on success, `401 Unauthorized` on failure
- GET `/api/auth/session`: Check the current authentication session.
  - Response: `{ user: object }` if authenticated, `{ error: "Not authenticated" }` if not
  - Status Codes: `200 OK` if authenticated, `401 Unauthorized` if not
- DELETE `/api/auth/logout`: Logout the current user.
  - Response: `{}`
  - Status Codes: `200 OK`

### Phase Management Routes

- GET `/api/currentPhase`: Retrieve the current phase.
  - Response: `{ phase: object }`
  - Status Codes: `200 OK`, `500 Internal Server Error` on failure
- PATCH `/api/changePhase`: Change to the next phase.
  - Response: `{ phase: object }`
  - Status Codes: `200 OK`, `500 Internal Server Error` on failure

### Budget Management Routes

- POST `/api/adminSetBudgets`: Set the budget amount.
  - Request Body: `{ amount: number }`
  - Response: `{ budget: object }`
  - Status Codes: `200 OK`, `500 Internal Server Error` on failure

### Proposal Management Routes

- GET `/api/userProposals`: Retrieve proposals submitted by the current user.
  - Response: `{ proposals: array }`
  - Status Codes: `200 OK`, `500 Internal Server Error` on failure
- GET `/api/allProposals`: Retrieve all proposals.
  - Response: `{ proposals: array }`
  - Status Codes: `200 OK`, `500 Internal Server Error` on failure
- GET `/api/userPreferences`: Retrieve proposals based on user preferences.
  - Response: `{ proposals: array }`
  - Status Codes: `200 OK`, `500 Internal Server Error` on failure
- POST `/api/proposals`: Create a new proposal.
  - Request Body: `{ estimated_cost: number, description: string }`
  - Response: `{ proposals: object }`
  - Status Codes: `200 OK`, `500 Internal Server Error` on failure
- DELETE `/api/proposals/:id`: Retrieve proposals based on user preferences.
  - Response: `{}`
  - Status Codes: `200 OK`, `500 Internal Server Error` on failure
- PATCH `/api/proposals/:id`: Update a proposal by ID.
  - Request Body: `{ estimated_cost: number, description: string }`
  - Response: `{}`
  - Status Codes: `200 OK`, `500 Internal Server Error` on failure
- POST `/api/proposals/:id/score`: Submit a score for a proposal.
  - Request Body: `{ score: number }`
  - Response: `{}`
  - Status Codes: `200 OK`, `500 Internal Server Error` on failure

### Restarting Route

- POST `/api/admin/resetProcess`: Reset the scoring process.
  - Response: `{}`
  - Status Codes: `200 OK`, `500 Internal Server Error` on failure

## Database Tables

- Table `users` - User details with usernam,email and role
- Table `app_phases` - Phase number and budget of the system
- Table `proposals` - Id of submitter, description and cost of proposal
- Table `app_score` - Id of submitter, Id of the proposal, score

## Screenshots

![Screenshot1](./screenshot1.png)
![Screenshot2](./screenshot2.png)

## Users Credentials

- username: admin, password: admin123
- username: "john_doe", password: password123
- username: "jane_doe", password: securepass
- username: "user_one", password: userpassword
