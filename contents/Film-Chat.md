# Exam Project – MQTT-Based Film Chat Extension

## Description

**Film-Chat** is a full-stack web application that lets multiple reviewers collaborate on shared public films and chat with one another in **real time**. It extends a classic Film Manager platform with a live, non-persistent messaging layer built on **MQTT over WebSockets**, plus a presence system that shows which reviewers are currently online.

The **frontend** is a single-page application built with **React 18** and **Vite**, using **React Router** for navigation, **React-Bootstrap / Bootstrap 5** and **Semantic UI React** for the UI, and **MQTT.js** to publish and subscribe to per-film chat topics (`/chats/<filmId>`) directly from the browser over a WebSocket connection to a **Mosquitto** broker. Messages use **QoS 0** with `retain: false`, so nothing is stored — reviewers only receive messages while online and subscribed.

The **backend** is a **Node.js / Express** REST API defined with **OpenAPI (Swagger)** via `oas3-tools`. It uses **Passport.js** (local strategy) with **express-session** cookie sessions for authentication, **bcrypt** for password hashing, **SQLite3** for persistence, and **AJV** JSON-Schema validation on request bodies. Alongside the REST layer, the server runs a raw **`ws` WebSocket server** for broadcasting user login/logout presence events, and an **MQTT client** that publishes retained per-film reviewer-status messages to the broker.

**Tech stack:** React 18, Vite, React Router, React-Bootstrap, Bootstrap 5, Semantic UI React, MQTT.js, Node.js, Express, OpenAPI/Swagger, Passport.js, express-session, bcrypt, SQLite3, AJV JSON Schema, `ws` (WebSocket), Mosquitto (MQTT broker), MQTT over WebSockets.

---

# Full Technology Stack & Architecture

> This section is an exhaustive, self-contained description of every technology, library, protocol, port, file, and convention used in the project. It is intended to be complete enough to answer questions about the project without reading the source.

## Repository layout

| Path | Purpose |
| --- | --- |
| `Client/` | React + Vite single-page application (the frontend). |
| `Server/` | Node.js + Express REST API, WebSocket presence server, and MQTT client (the backend). |
| `Mosquitto Configuration/` | Eclipse Mosquitto broker config files, one for `Linux/` and one for `Windows/`. |
| `REST APIs Design/` | Markdown documentation of the REST API surface (`README.md`). |
| `DSP_20250620.pdf` | The original exam specification ("Distributed Systems / Web Applications" exam, call of 2025‑06‑20). |
| `Server/database/` | SQLite database file(s) and a plaintext list of test passwords. |

The project has **no monorepo tooling**: `Client/` and `Server/` are two independent npm packages, each with its own `package.json` and `package-lock.json`. There is **no TypeScript** anywhere; the client uses ES modules, the server uses CommonJS (`require`).

## Origin / academic context

The codebase extends the **"Lab 5" / BigLab01** solution from the *Applicazioni Web I / Web Applications I* course at **Politecnico di Torino**. The exam task ("Exam Call 3 – MQTT") adds: (1) letting multiple reviewers select the same public film, and (2) real‑time chat between reviewers over **MQTT over WebSockets**, with **no message persistence**.

---

## Frontend — `Client/`

### Core framework & build

| Technology | Version | Notes |
| --- | --- | --- |
| **React** | 18.3.1 | Function components + hooks only. Rendered with `ReactDOM.createRoot` inside `<React.StrictMode>` (`src/index.jsx`). |
| **react-dom** | 18.3.1 | `react-dom/client` API. |
| **Vite** | 3.2.11 | Dev server (`npm start` → `vite`) and bundler (`npm run build` → `vite build`). |
| **@vitejs/plugin-react** | 2.2.0 | React Fast Refresh / JSX transform. |
| Build target | `es2020` | Set in `vite.config.js` for both `build` and `optimizeDeps.esbuildOptions`. |
| Dev server port | **5173** | Vite default; hard‑coded as the allowed CORS origin on the server. |

### Routing

- **react-router-dom** 6.30.1 — `BrowserRouter`, `Routes`, `Route`, `Navigate`, `useLocation`.
- Route table (in `src/App.jsx`, nested under a guarded `/` route that redirects to `/login` when logged out):
  - `/` and `/private` → `PrivateLayout`
  - `/private/add` → `AddPrivateLayout`, `/private/edit/:filmId` → `EditPrivateLayout`
  - `/public` → `PublicLayout`, `/public/add` → `AddPublicLayout`, `/public/edit/:filmId` → `EditPublicLayout`
  - `/public/:filmId/reviews` → `ReviewLayout`, `/public/:filmId/reviews/complete` → `EditReviewLayout`, `/public/:filmId/issue` → `IssueLayout`
  - `/public/to_review` → `PublicToReviewLayout` (hosts the chat + per‑film reviewer status)
  - `/online` → `OnlineLayout`
  - `/login` → `LoginLayout`
  - `*` → `NotFoundLayout`

### UI libraries

| Technology | Version | Notes |
| --- | --- | --- |
| **react-bootstrap** | 2.10.10 | Primary component library (`Container`, `Toast`, `Card`, `Form`, `Button`, `Alert`, …). |
| **bootstrap** | 5.3.6 | CSS imported in `App.jsx` (`bootstrap/dist/css/bootstrap.min.css`). |
| **bootstrap-icons** | 1.13.1 | Icon font imported in `App.jsx`. |
| **semantic-ui-react** | 2.1.5 | Secondary component library used in parts of the UI. |
| **react-select** | 5.10.1 | Multi-select dropdowns (e.g. choosing reviewers to invite). |
| **react-js-pagination** | 3.0.3 | Pagination controls for film / review lists. |
| **dayjs** | 1.11.13 | Date parsing/formatting; used inside the `Film` and `Review` models and for `watchDate` / `reviewDate` (`YYYY-MM-DD` on the wire). |
| Custom CSS | — | `src/App.css`, `src/index.css`. |

### State management

- **No Redux / MobX / Zustand.** State is local component state via `useState` / `useEffect` / `useRef` / `useContext`.
- **`MessageContext`** (`src/messageCtx.jsx`) — a React context exposing `handleErrors`; errors surface as an auto-hiding Bootstrap `Toast` in `App.jsx`.
- **`window.sessionStorage`** is used as an ad‑hoc store for: `user`, `userId`, `username`, `email`, `filmManager` (the HATEOAS root), and pagination bookkeeping (`totalPages`, `currentPage`, `totalItems`, `filmsType`).

### Client → server communication

- **`src/API.js`** — thin wrapper over the `fetch` API. Base URL `http://localhost:3001`. Every call uses `credentials: "include"` so the `express-session` cookie is sent. A `getJson` helper normalizes success/error JSON.
- The API is **HATEOAS-style**: the client first `GET`s `/api` to obtain a `filmManager` resource full of hyperlinks (`films`, `privateFilms`, `publicFilms`, `invitedPublicFilms`, `users`, `usersAuthenticator`, …) and follows those links, plus per-entity `self` links.
- **Models** (`src/models/`): `Film`, `Review`, `User` — plain constructor functions that normalize server payloads (e.g. `private` → `privateFilm`, dates → `dayjs`).

### Real-time on the client

Two independent real-time channels are opened once, at `App` mount:

1. **Native `WebSocket`** to `ws://localhost:5000` — global presence. Handles messages of `typeMessage`:
   - `login` → add user to `onlineList`
   - `logout` → remove user
   - `update` → replace user entry (used when a reviewer changes their selected film)
2. **MQTT.js client** (`mqtt` 5.13.1) to `ws://<window.location.hostname>:8080` (MQTT over WebSockets). Client options: `keepalive: 30`, `clean: true`, `reconnectPeriod: 1000`, `connectTimeout: 30000`, random `clientId` `mqttjs_<hex>`, Last-Will topic `WillMsg`. Used for:
   - **Per-film reviewer status** — subscribes to topic `<filmId>` (numeric string); server publishes **retained** `{ typeMessage: "active"|"inactive", status, reviewers: [{userId,userName}] }` messages. On `typeMessage: "deleted"` the client unsubscribes.
   - **Chat** — see `FilmChat.jsx` below.

### Chat component — `src/components/FilmChat.jsx`

- Topic per film: **`/chats/<filmId>`**.
- Subscribes with **QoS 0** on mount; unsubscribes + clears local messages on unmount.
- Outgoing message shape (validated server-side by `mqtt_chat_message_schema.json`):
  ```json
  { "reviewer": "Frank", "reviewerId": 2, "timestamp": "2025-06-04T15:45:00Z", "message": "..." }
  ```
- Published with `{ qos: 0, retain: false }` — **nothing is stored**; only currently-subscribed reviewers receive it.
- `timestamp` is `new Date().toISOString()`.
- Own messages render right-aligned (blue bubble, `✓`), others left-aligned (gray bubble, sender name shown).
- Auto-scrolls to newest message via `bottomRef.scrollIntoView({ behavior: "smooth" })` in a `useEffect` keyed on `messages`.
- Shows a warning `Alert`: *"Only users currently in this chat will receive your message."*

### Components inventory (`src/components/`)

`Auth.jsx`, `Cards.jsx`, `FilmChat.jsx`, `FilmReviewLibrary.jsx`, `FilmToReviewLibrary.jsx`, `Filters.jsx`, `IssueReviewLibrary.jsx`, `MiniOnlineList.jsx`, `Navigation.jsx`, `OnlineList.jsx`, `PageLayout.jsx` (exports all the `*Layout` route elements), `PrivateFilmForm.jsx`, `PrivateFilmLibrary.jsx`, `PublicFilmForm.jsx`, `PublicFilmLibrary.jsx`, `ReviewForm.jsx`, `ws_message.js` (client-side WS message class).

### Frontend testing & misc

- `web-vitals` 2.1.4 + `src/reportWebVitals.jsx` (Create-React-App leftover).
- `src/setupTests.jsx` imports `@testing-library/jest-dom`; `src/App.test.jsx` uses `@testing-library/react` (Jest-style). *(These testing packages are referenced but not listed in `package.json` dependencies.)*
- ESLint config: `eslintConfig.extends = ["react-app", "react-app/jest"]`.
- `public/` holds `favicon.ico`, `logo192.png`, `logo512.png`, `manifest.json`, `robots.txt` (CRA-style PWA scaffolding).
- npm scripts: `start` → `vite`, `build` → `vite build`.

---

## Backend — `Server/`

### Core framework

| Technology | Version | Notes |
| --- | --- | --- |
| **Node.js** | — | CommonJS modules. |
| **Express** | 4.21.2 | HTTP framework (pulled in transitively via `oas3-tools`; the app instance comes from `oas3Tools.expressAppConfig(...).getApp()`). |
| **oas3-tools** | 2.0.2 | Wires the **OpenAPI 3.0.1** spec (`api/openapi.yaml`, also `api/openapi.json`) to controllers and serves **Swagger UI at `http://localhost:3001/docs`**. |
| **connect** | 3.7.0 | Middleware framework (dependency). |
| **cors** | 2.8.5 | `origin: http://localhost:5173`, `credentials: true`. |
| Server port | **3001** | `http.createServer(app).listen(3001)` in `index.js`. |
| `js-yaml` | 3.14.1 | Reads the YAML OpenAPI spec. |

The server was **scaffolded by swagger-codegen** — see `.swagger-codegen/VERSION` and `.swagger-codegen-ignore`.

### Authentication & sessions

| Technology | Version | Notes |
| --- | --- | --- |
| **passport** | 0.6.0 | Auth middleware. `serializeUser` / `deserializeUser` are pass-through (the whole user object is stored in the session). |
| **passport-local** | 1.0.0 | `LocalStrategy` with `usernameField: 'email'`, `passwordField: 'password'`. |
| **express-session** | 1.18.1 | `secret: "shhhhh... it's a secret!"`, `resave: false`, `saveUninitialized: false`, `cookie.maxAge: 300000` (**5 minutes**). |
| **bcrypt** | 6.0.0 | `bcrypt.compareSync` to verify passwords against the stored `hash`. |

- Auth guard: `isLoggedIn` middleware returns `401 { error: 'Not authorized' }` when `req.isAuthenticated()` is false.
- **Server-side session timeout**: a per-`sessionID` `setTimeout` of 300000 ms; on expiry it fires a WebSocket `logout` broadcast (`handleSessionExpiration` → `WebSocketLogoutNotify`). `checkSessionTimeout` clears the timer on activity.
- Login flow: if already authenticated it first logs the user out (WS `logout` broadcast), then `passport.authenticate('local')`, then on success broadcasts a WS `login` message (including the user's currently active film, if any).

### Persistence

| Technology | Version | Notes |
| --- | --- | --- |
| **sqlite3** | 5.1.7 (`.verbose()`) | Single connection in `components/db.js` to `database/databaseV2.db`. Runs `PRAGMA foreign_keys = ON;` on open. |
| WAL files | — | `databaseV2.db-shm`, `databaseV2.db-wal` present (SQLite Write-Ahead Logging). |
| Legacy | — | `database/database.zip` and `database/passwords_databases.txt` (plaintext test credentials). |

- **No ORM.** Raw SQL strings in the `service/` layer, executed with `db.all` / `db.get` / `db.run`, wrapped in hand-rolled `Promise`s. `selectFilm` uses an explicit `BEGIN TRANSACTION` / `COMMIT` / `ROLLBACK`.
- Inferred schema:
  - **`users`** `(id, name, email, hash)`
  - **`films`** `(id, title, owner, private, watchDate, rating, favorite)`
  - **`reviews`** `(filmId, reviewerId, completed, reviewDate, rating, review, active)` — `active = 1` marks the reviewer's currently selected film.

### Request validation

| Technology | Version | Notes |
| --- | --- | --- |
| **express-json-validator-middleware** | 3.0.1 | `validate({ body: schema })` middleware on write routes. |
| **ajv** | 8.17.1 | JSON Schema validator (`new Validator({ allErrors: true })`). |
| **ajv-formats** | 2.1.1 | Adds `date`, `date-time`, `email`, … format checks. |

- Schemas in `Server/json_schemas/` (all JSON Schema **draft-07**):
  - `film_schema.json`, `user_schema.json`, `review_schema.json` — loaded in `index.js` and registered with AJV.
  - `mqtt_chat_message_schema.json` — `ChatMessage` `{ reviewer:string, reviewerId:number, timestamp:date-time, message:string }`, all required.
  - `ws_message_schema.json` — `WebSocket Message` `{ typeMessage: login|logout|update, userId:int, userName, filmId:int, taskName }`; conditional `required` per `typeMessage`.
- Error handling: `ValidationError` → `400`; `UnauthorizedError` → `401`.

### Real-time on the server

1. **Raw WebSocket server** — `ws` **8.18.2**, `new WebSocket.Server({ port: 5000 })` in `components/websocket.js`.
   - Keeps a `Map` (`loginMessagesMap`) of `userId → last login/update message`; replays all of them to every newly connected client so late joiners see who is already online.
   - `sendAllClients(message)` broadcasts JSON to every connected client.
   - Message types produced: `login`, `logout`, `update` (built via `components/ws_message.js`, a `WSMessage` class).
2. **MQTT client** — `mqtt` **5.13.1**, connects to `ws://127.0.0.1:8080` (MQTT over WebSockets) in `components/mqtt.js`.
   - Options: `keepalive: 30`, `clean: true`, `reconnectPeriod: 60000`, `connectTimeout: 30000`, random `clientId`, Last-Will topic `WillMsg`, `rejectUnauthorized: false`.
   - On `connect`: for every public film it publishes a **retained** reviewer-status message so subscribers get current state immediately.
   - `publishFilmMessage(filmId, message)` publishes to topic `String(filmId)` with **`{ qos: 0, retain: true }`**.
   - Message shape: `{ typeMessage: "active"|"inactive", status, reviewers: [{ userId, userName }] }` (see `components/mqtt_film_message.js`).
   - Triggered from `ReviewsService.selectFilm` whenever a reviewer selects/deselects a film (also emits a WS `update`).

### Backend structure (layered)

```
index.js                     # app wiring: CORS, session, passport, routes, error handlers, HTTP server
controllers/                 # thin HTTP handlers (generated by swagger-codegen)
  FilmsController.js
  ReviewsController.js
  UsersController.js          # also defines the passport LocalStrategy + session-timeout logic
service/                     # business logic + raw SQL
  FilmsService.js
  ReviewsService.js           # film selection, review issuing, MQTT + WS side-effects
  UsersService.js             # getUserByEmail/ById, checkPassword (bcrypt)
components/                   # infrastructure + domain objects
  db.js                       # sqlite3 connection
  film.js  review.js  user.js # domain classes
  FilmManager.js              # HATEOAS resource (hyperlink map)
  websocket.js                # ws server + online-users map
  mqtt.js                     # mqtt client + retained publishing
  ws_message.js  mqtt_film_message.js
utils/
  constants.js                # OFFSET = "10"  (pagination page size)
  writer.js                   # writeJson / ResponsePayload helpers
api/
  openapi.yaml / openapi.json # OpenAPI 3.0.1 spec
json_schemas/                 # AJV schemas
```

- **Pagination**: page size is the `OFFSET` constant (`"10"`) in `utils/constants.js`; requests take `?pageNo=`, SQL appends `LIMIT ?,?`; responses include `totalPages`, `currentPage`, `totalItems`.
- npm scripts: `start` → `node index.js`, `prestart` → `npm install`.
- `Server/.gitignore` ignores `node_modules/`.

### REST API surface

Base URL `http://localhost:3001/api`, JSON, session-cookie auth. Defined in `Server/index.js` and documented in `REST APIs Design/README.md` + `api/openapi.yaml`.

| Method & path | Auth | Purpose |
| --- | --- | --- |
| `GET /api` | no | HATEOAS entry point (`FilmManager` hyperlinks) |
| `POST /api/users/authenticator` | no | Log in (email + password), starts session |
| `GET /api/users` | yes | List all users |
| `GET /api/users/:userId` | yes | Single user |
| `PUT /api/users/:userId/selection` | yes | Select a public film as the reviewer's active film |
| `GET /api/films/public` | no | Public films (paginated) |
| `GET /api/films/public/:filmId` | no | Single public film |
| `GET /api/films/public/invited` | yes | Public films the user was invited to review |
| `POST /api/films` | yes | Create a film |
| `GET /api/films/private` / `GET /api/films/private/:filmId` | yes | Private films of the user |
| `PUT /api/films/private/:filmId` / `PUT /api/films/public/:filmId` | yes | Update film |
| `DELETE /api/films/private/:filmId` / `DELETE /api/films/public/:filmId` | yes | Delete film |
| `GET /api/films/public/:filmId/reviews` | no | Reviews for a film |
| `POST /api/films/public/:filmId/reviews` | yes (owner) | Assign one or more reviewers |
| `GET /api/films/public/:filmId/reviews/:reviewerId` | no | Single review |
| `PUT /api/films/public/:filmId/reviews/:reviewerId` | yes | Complete/update a review |
| `DELETE /api/films/public/:filmId/reviews/:reviewerId` | yes (owner) | Remove a review invitation |

Typical error codes: `400` validation, `401` not authorized, `403` not the owner, `404` not found / wrong visibility, `409` conflict (already assigned / completed), `500` server error.

---

## MQTT broker — Eclipse Mosquitto

- Config files under `Mosquitto Configuration/` (the folder must contain `mosquitto.conf`):
  - **Windows** (`Windows/mosquitto.conf`): `listener 1883 localhost` (protocol `mqtt`), `listener 8080 localhost` (protocol `websockets`), `allow_anonymous true`.
  - **Linux** (`Linux/mosquitto.conf`): same two listeners but bound to **all interfaces** (no `localhost` restriction).
- Both the browser client and the Node server connect over the **WebSockets listener on 8080**. No authentication (anonymous).

---

## Messaging channels & topic conventions (summary)

| Concern | Transport | Topic | Payload | QoS / retain | Direction |
| --- | --- | --- | --- | --- | --- |
| **Reviewer chat** | MQTT/WS (:8080) | `/chats/<filmId>` | `{reviewer, reviewerId, timestamp, message}` | QoS 0, **retain false** (ephemeral, never stored) | client ↔ client |
| **Per-film reviewer status** | MQTT/WS (:8080) | `<filmId>` (numeric string) | `{typeMessage, status, reviewers:[{userId,userName}]}` | QoS 0, **retain true** | server → clients |
| **Global presence** | native WebSocket (:5000) | n/a | `{typeMessage: login\|logout\|update, userId, userName, filmId, filmTitle}` | n/a | server → clients |

---

## Ports

| Port | Service |
| --- | --- |
| **5173** | Vite dev server (frontend) |
| **3001** | Express REST API + Swagger UI (`/docs`) |
| **5000** | `ws` WebSocket presence server |
| **1883** | Mosquitto MQTT (plain TCP) |
| **8080** | Mosquitto MQTT over WebSockets (used by browser + server) |

## Test credentials

- `user.dsp@polito.it` / `password` (from `Server/README.md`; more in `Server/database/passwords_databases.txt`).

## Tooling / meta

- **Package manager:** npm (committed `package-lock.json` in both `Client/` and `Server/`).
- **VCS:** git.
- **API contract:** OpenAPI 3.0.1 (`title: Film Manager`, `version: 1.0.0`), served via Swagger UI.
- **Code generation:** swagger-codegen (server stub).
- **Languages:** JavaScript only — JSX + ES modules on the client, CommonJS on the server. No TypeScript, no build step on the server.
- **Dev OS support:** Windows and Linux (separate Mosquitto configs; repo also contains macOS `.DS_Store` artifacts).

---

## Overview

This project extends the Lab 5 solution to allow **multiple reviewers** to select the same public film for review, and to enable **real-time chat between reviewers** using **MQTT over WebSockets**. No message persistence is implemented, as per the exam requirements.

## Design Choices

### MQTT Topics

- Each film has a **dedicated MQTT topic** for chat messages:

  ```
  /chats/<filmId>
  ```

- Example: If a film has ID `3`, the MQTT topic used is `/chats/3`.

### MQTT Messages

Messages are sent as JSON objects with the following schema:

```json
{
  "reviewer": "Frank",
  "reviewerId": 2,
  "timestamp": "2025-06-04T15:45:00Z",
  "message": "Let’s focus on the ending scene."
}
```

### Publisher

- A message is published to the topic `/chats/<filmId>` when the reviewer sends a chat message.
- The message is sent using:

  ```js
  client.publish(`/chats/${filmId}`, JSON.stringify(message), {
    qos: 0,
    retain: false,
  });
  ```

### Subscriber

- When a reviewer navigates to the "Public to Review" page, the client subscribes to `/chats/<filmId>` for every assigned film.

- The MQTT client is created using:

  ```js
  mqtt.connect("ws://localhost:8080", options);
  ```

- On receiving a message, the client appends it to the film-specific chat window.

### Retention Policy

- **`retain: false`**: No message is stored on the broker. Reviewers only receive messages **when they are online and subscribed**.
- This matches the assignment rule: messages must be exchanged **only via MQTT** and **not stored**.

### QoS

- **QoS 0** (at most once) was selected:

  - Messages are delivered only once if the recipient is connected.
  - No delivery guarantees.
  - Lightweight and minimal overhead — ideal for temporary, real-time messaging.

## UI & UX Features

### Chat Styling

- Messages from the current user appear **on the right** with a blue bubble.
- Messages from others appear **on the left** with a light gray bubble.
- If a user is the sender:

  - **✓** indicates the message was sent.

### Auto Scroll on New Message

- The chat panel **automatically scrolls to the bottom** on each new message using:

  ```js
  useEffect(() => {
    if (bottomRef.current) {
      bottomRef.current.scrollIntoView({ behavior: "smooth" });
    }
  }, [messages]);
  ```

### Chat Notification

- For clearity we show an alert:

  > Only users currently in this chat will receive your message.

## Why This Design

- Per-film MQTT topics ensure messages are scoped correctly.
- Using `retain: false` and `QoS 0` ensures no persistence, per requirements.
- WebSocket + MQTT allow full real-time experience without modifying the database schema.

## Result

- Multiple reviewers can select the same film.
- They can chat in real-time without any stored messages.
- The implementation strictly adheres to the exam constraints.

---

Prepared by: _Alireza Khalili_
Date: _18/06/2025_
