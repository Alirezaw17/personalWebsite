# DevBoard

A full-stack project & task management board. Users register, create colour-coded
projects, break them down into prioritised tasks with due dates, and every change
is recorded in a per-user activity log that feeds a dashboard.

Built as a hands-on study of a **real production-shaped architecture**: session
auth with a Redis store, a PostgreSQL relational schema, database transactions,
ownership / role checks, and a layered `route → service → dao` backend, paired
with a typed React 19 single-page app.

---

## Table of contents

- [Tech stack](#tech-stack)
- [Full stack breakdown](#full-stack-breakdown)
- [Features](#features)
- [Architecture](#architecture)
- [Data model](#data-model)
- [API reference](#api-reference)
- [Project structure](#project-structure)
- [Getting started](#getting-started)
- [Environment variables](#environment-variables)
- [Available scripts](#available-scripts)
- [Security notes](#security-notes)
- [Known limitations & roadmap](#known-limitations--roadmap)
- [LinkedIn project blurb](#linkedin-project-blurb)

---

## Tech stack

### Frontend (`/client`)
| Concern | Choice |
| --- | --- |
| Framework | React 19 + the React Compiler (`babel-plugin-react-compiler`) |
| Language | TypeScript |
| Build tool | Vite 8 |
| Routing | React Router 7 |
| UI kit | React-Bootstrap 5 + `react-bootstrap-icons` |
| Data layer | `fetch` wrappers in `src/api.ts`, cookie sessions (`credentials: 'include'`) |

### Backend (`/server`)
| Concern | Choice |
| --- | --- |
| Runtime | Node.js + Express 5 |
| Auth | `express-session` with an httpOnly session cookie |
| Session store | Redis via `connect-redis` |
| Database | PostgreSQL via `pg` (connection pool) |
| Password hashing | `bcrypt` (12 salt rounds) |
| Config | `dotenv`, `cors` (credentialed, single origin) |

---

## Full stack breakdown

Every moving part of the project, layer by layer, with the version range pinned in
`package.json`.

### Language & tooling
| Part | Version | Role |
| --- | --- | --- |
| Node.js | 18+ | JavaScript runtime for the API |
| TypeScript | `~6.0.2` | Static typing for the entire client |
| npm workspaces | – | Two independent packages: `client/` and `server/` |

### Frontend — `client/` (runtime dependencies)
| Package | Version | Role |
| --- | --- | --- |
| `react` / `react-dom` | `^19.2.6` | UI library (React 19) |
| `react-router-dom` | `^7.15.1` | Client-side routing, public vs. guarded routes |
| `bootstrap` | `^5.3.8` | Base CSS framework (Bootstrap 5) |
| `react-bootstrap` | `^2.10.10` | Bootstrap components as React components |
| `react-bootstrap-icons` | `^1.11.6` | SVG icon set |

### Frontend — `client/` (build & dev tooling)
| Package | Version | Role |
| --- | --- | --- |
| `vite` | `^8.0.12` | Dev server + production bundler |
| `@vitejs/plugin-react` | `^6.0.1` | React fast-refresh / JSX transform for Vite |
| `babel-plugin-react-compiler` | `^1.0.0` | React Compiler — auto-memoisation at build time |
| `@babel/core` / `@rolldown/plugin-babel` | `^7.29.0` / `^0.2.3` | Babel pass that runs the React Compiler plugin |
| `typescript` / `typescript-eslint` | `~6.0.2` / `^8.59.2` | Type-checking (`tsc -b`) and typed lint rules |
| `eslint` | `^10.3.0` | Linting, with `eslint-plugin-react-hooks` + `eslint-plugin-react-refresh` |
| `@types/react`, `@types/react-dom`, `@types/node` | matching majors | Ambient type definitions |

### Backend — `server/` (runtime dependencies)
| Package | Version | Role |
| --- | --- | --- |
| `express` | `^5.2.1` | HTTP framework (Express 5) |
| `express-session` | `^1.19.0` | Session middleware, `httpOnly` cookie |
| `connect-redis` | `^9.0.0` | Adapter that stores sessions in Redis |
| `redis` | `^5.12.1` | Redis client |
| `pg` | `^8.20.0` | PostgreSQL client with a connection pool |
| `bcrypt` | `^6.0.0` | Password hashing (12 salt rounds) |
| `cors` | `^2.8.6` | CORS, locked to one credentialed origin |
| `helmet` | `^8.1.0` | Security headers (dependency present, currently commented out) |
| `dotenv` | `^17.4.2` | Loads `server/.env` |

### Backend — `server/` (dev tooling)
| Package | Version | Role |
| --- | --- | --- |
| `nodemon` | `^3.1.14` | Auto-restart the API on file change |

### Data & infrastructure
| Part | Version | Role |
| --- | --- | --- |
| PostgreSQL | 14+ | Relational store: `users`, `projects`, `tasks`, `activity_log` |
| Redis | 6+ | Session store backing `express-session` |

### At a glance

```
Frontend   React 19 · TypeScript · Vite 8 · React Router 7 · React-Bootstrap · React Compiler
Backend    Node.js · Express 5 · express-session · bcrypt · cors · helmet · dotenv
Data       PostgreSQL (pg) · Redis (connect-redis)
Tooling    TypeScript · ESLint · nodemon
```

---

## Features

**Authentication & roles**
- Register / login / logout with hashed passwords.
- Session ID is regenerated on login to prevent session fixation.
- The **first user to register becomes `admin`**; everyone after is a `member`.
- `GET /auth/me` re-validates the cookie and returns the current user.
- `requireAuth` middleware guards every protected route.

**Projects**
- Full CRUD, each project scoped to its owner (`user_id`).
- Colour tag (defaults to `#6366f1`) and a `status` (`active` / `archived`).
- Ownership is enforced on read-by-id, update and delete.
- `admin` can delete any project; a `member` can only delete their own.

**Tasks**
- CRUD nested under a project (`/projects/:id/tasks`).
- `priority` (`low` / `medium` / `high`) and `status` (`todo` / `in_progress` / `done`), validated server-side.
- Optional `due_date`; partial updates fall back to the existing values.
- Viewing a project's tasks requires ownership of that project.

**Activity log & dashboard**
- Creating, updating or deleting a project writes an `activity_log` row **inside the same transaction** as the change itself, so the log can never drift from the data.
- The log intentionally stores the project name as plain text (no FK), so history survives project deletion.
- `GET /dashboard` returns the current user's activity feed, rendered on the dashboard alongside a quick-overview panel.

**Data integrity**
- Multi-statement writes (`createProjects`, `updateProjects`, `deleteProjects`) run in `BEGIN / COMMIT / ROLLBACK` transactions with a dedicated pooled client that is always released.
- `ON DELETE CASCADE` on foreign keys keeps projects/tasks/logs consistent when a parent row is removed.

---

## Architecture

```
Browser (React 19 SPA)
   │  fetch + session cookie (credentials: 'include')
   ▼
Express 5 API  ── requireAuth middleware ──┐
   │                                       │
   ├── routes  (index.js)                  │  session cookie
   ├── service (service.js)  ── ownership / not-found rules
   └── dao     (dao.js)      ── SQL, transactions, bcrypt
        │                                  │
        ▼                                  ▼
   PostgreSQL                          Redis (session store)
```

The backend is deliberately layered:

- **route** (`index.js`) – HTTP concerns: parse the request, map errors to status codes.
- **service** (`service.js`) – business rules such as "does this project exist and does it belong to the caller?".
- **dao** (`dao.js`) – all database access, SQL, transactions and password hashing.

`getProjectById` is the reference example that flows through all three layers; the
rest of the codebase is progressively being refactored toward the same shape.

---

## Data model

PostgreSQL schema (see [`server/DEVBOARD .session.sql`](server/DEVBOARD%20.session.sql)):

```sql
users
  id            SERIAL PK
  email         VARCHAR(255) UNIQUE NOT NULL
  password_hash TEXT NOT NULL
  display_name  VARCHAR(100) NOT NULL
  role          VARCHAR(20) NOT NULL DEFAULT 'member'   -- 'admin' | 'member'
  created_at    TIMESTAMPTZ DEFAULT NOW()

projects
  id          SERIAL PK
  user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE
  name        VARCHAR(150) NOT NULL
  description TEXT
  color       CHAR(7) DEFAULT '#6366f1'
  status      VARCHAR(20) DEFAULT 'active'
  created_at  TIMESTAMPTZ DEFAULT NOW()

tasks
  id          SERIAL PK
  project_id  INTEGER NOT NULL REFERENCES projects(id) ON DELETE CASCADE
  title       VARCHAR(255) NOT NULL
  description TEXT
  priority    VARCHAR(10) DEFAULT 'medium'
  status      VARCHAR(20) DEFAULT 'todo'
  due_date    DATE
  created_at  TIMESTAMPTZ DEFAULT NOW()

activity_log
  id           SERIAL PK
  user_id      INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE
  action       VARCHAR(100) NOT NULL
  entity_type  VARCHAR(50)
  entity_name  VARCHAR(255)
  project_name VARCHAR(255)   -- denormalised on purpose: no FK to projects
  created_at   TIMESTAMPTZ DEFAULT NOW()
```

---

## API reference

Base URL: `http://localhost:3000`
All protected routes require the session cookie issued at login.

### Auth
| Method | Path | Auth | Description |
| --- | --- | --- | --- |
| `POST` | `/auth/register` | – | Create a user, start a session. First ever user → `admin`. |
| `POST` | `/auth/login` | – | Verify credentials, regenerate the session. |
| `POST` | `/auth/logout` | – | Destroy the session and clear the cookie. |
| `GET`  | `/auth/me` | ✅ | Return the currently authenticated user. |
| `GET`  | `/session` | – | Debug helper: dump the raw session object. |

### Projects
| Method | Path | Auth | Description |
| --- | --- | --- | --- |
| `GET`    | `/projects` | ✅ | List the caller's projects. |
| `GET`    | `/projects/:id` | ✅ | Get one project (404 if missing, 403 if not owner). |
| `POST`   | `/projects` | ✅ | Create a project (+ activity log, transactional). |
| `PATCH`  | `/projects/:id` | ✅ | Partial update (owner only, + activity log). |
| `DELETE` | `/projects/:id` | ✅ | Delete (owner or `admin`, + activity log). |

### Tasks
| Method | Path | Auth | Description |
| --- | --- | --- | --- |
| `GET`    | `/projects/:id/tasks` | ✅ | List tasks for a project the caller owns. |
| `POST`   | `/projects/:id/tasks` | ✅ | Create a task (`title` + `description` required). |
| `PATCH`  | `/projects/:projectId/tasks/:taskId` | ✅ | Partial update, validates `priority` / `status`. |
| `DELETE` | `/projects/:projectId/tasks/:taskId` | ✅ | Delete (owner or `admin`). |

### Dashboard
| Method | Path | Auth | Description |
| --- | --- | --- | --- |
| `GET` | `/dashboard` | ✅ | Return the caller's activity-log feed. |

---

## Project structure

```
DevBoard/
├── client/                      # React 19 + Vite + TypeScript SPA
│   ├── src/
│   │   ├── api.ts               # typed fetch wrappers for every endpoint
│   │   ├── types.ts             # User / Projectt / Taskk / ActivityLog interfaces
│   │   ├── App.tsx              # routes (public: /login /register, guarded: /dashboard …)
│   │   ├── components/          # Layout, Header, Footer
│   │   └── pages/               # Login, Register, Dashboard, AllProjects, AllTasks, …
│   └── vite.config.ts           # React plugin + React Compiler preset
│
├── server/                      # Express 5 API
│   ├── index.js                 # app setup, middleware, all routes
│   ├── service.js               # business-rule layer
│   ├── dao.js                   # SQL, transactions, bcrypt
│   ├── db.js                    # pg Pool
│   ├── redis.js                 # Redis client for the session store
│   ├── requireAuth.js           # auth guard middleware
│   └── DEVBOARD .session.sql    # reference schema
│
└── README.md
```

---

## Getting started

### Prerequisites
- Node.js 18+
- PostgreSQL 14+
- Redis 6+

### 1. Clone & install
```bash
git clone <repo-url> DevBoard
cd DevBoard

# backend
cd server && npm install

# frontend
cd ../client && npm install
```

### 2. Create the database
```bash
createdb devboard
psql devboard -f "server/DEVBOARD .session.sql"
```

### 3. Configure the backend
Create `server/.env` (see [Environment variables](#environment-variables)).

### 4. Run
```bash
# terminal 1 – API (http://localhost:3000)
cd server && npx nodemon index.js

# terminal 2 – SPA (http://localhost:5173)
cd client && npm run dev
```

Open the SPA, register the first account (it becomes the admin), and start
creating projects.

---

## Environment variables

`server/.env`:

| Variable | Example | Purpose |
| --- | --- | --- |
| `DATABASE_URL` | `postgres://user:pass@localhost:5432/devboard` | PostgreSQL connection string |
| `REDIS_URL` | `redis://localhost:6379` | Redis connection for the session store |
| `SESSION_SECRET` | `a-long-random-string` | Signs the session cookie |
| `CLIENT_URL` | `http://localhost:5173` | Allowed CORS origin (credentialed) |

> The frontend currently points at `http://localhost:3000` via the `API_URL`
> constant in [`client/src/api.ts`](client/src/api.ts).

---

## Available scripts

### `client`
| Command | Description |
| --- | --- |
| `npm run dev` | Start Vite dev server |
| `npm run build` | Type-check (`tsc -b`) and build for production |
| `npm run preview` | Preview the production build |
| `npm run lint` | Run ESLint |

### `server`
Run with `npx nodemon index.js` (nodemon is a dependency) or `node index.js`.

---

## Security notes

Implemented:
- Passwords hashed with bcrypt (12 rounds); the hash is never returned to the client.
- Session ID regenerated on login; `httpOnly` cookie; 24-hour expiry.
- Parameterised SQL everywhere (no string interpolation of user input).
- Per-request ownership checks in the service/dao layers; role gate for destructive actions.
- CORS locked to a single credentialed origin.

For a real deployment you would still want:
- `cookie.secure = true` and `sameSite = 'none'` behind HTTPS.
- `helmet` (already a dependency, currently commented out).
- Rate limiting on the auth routes and input validation/schema (e.g. `zod`).
- A frontend route guard / auth context instead of relying on API 401s.

---

## Known limitations & roadmap

- [ ] Only `getProjectById` uses the full `route → service → dao` split; migrate the rest.
- [ ] `Dashboard` "Open Tasks" / "Completed this week" tiles are placeholder numbers.
- [ ] `Project.tsx` and `Task.tsx` pages are stubs; task detail view is unfinished.
- [ ] No global auth context — pages fetch `/auth/me` individually and unauthenticated users are redirected only after an API call fails.
- [ ] Task `status` option in one form sends `"in progress"` where the API expects `in_progress`.
- [ ] Move `API_URL` to a Vite env variable.
- [ ] Add automated tests (none yet) and CI.
- [ ] AI-assisted task suggestions (a `GEMINI_API_KEY` slot exists in `.env` for a planned integration).

---

## LinkedIn project blurb

**Subject:** Full-stack project & task management app with session auth, a
relational Postgres schema, and an auditable activity log.

> **DevBoard — Full-stack project & task manager**
> Designed and built a project management board where users organise work into
> colour-coded projects and prioritised tasks, with every change captured in a
> per-user activity log. Backend is a layered Express 5 API (route → service →
> DAO) over PostgreSQL with bcrypt auth, Redis-backed sessions, role-based access
> control, and database transactions that keep the audit log in sync with the
> data. Frontend is a TypeScript React 19 SPA (Vite, React Router, React-Bootstrap)
> with fully typed API clients.
> **Stack:** React 19 · TypeScript · Vite · Node.js · Express 5 · PostgreSQL · Redis · bcrypt

**One-liner version:**
> DevBoard — a full-stack (React 19 + TypeScript / Express 5 / PostgreSQL / Redis)
> project & task manager featuring session-based auth, role-based access control,
> transactional writes, and an auditable activity log.
