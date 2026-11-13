# POTKO API server

Standalone extraction of the Express, PostgreSQL, and WebSocket backend from [`Jonathanvinu2121/potko`](https://github.com/Jonathanvinu2121/potko), source revision `9a97aa54ec035e0609e9482b998ce5307288264a` (2026-11-13 checkout). It preserves the backend's existing routes, database schema, migrations, authorization rules, and realtime actions. The Expo client and its build artifacts are not included. The source checkout did not include a top-level license file; this extraction does not assign a license to upstream code.

This is a real data server. It does not silently seed production with sample records. The source repository's SQL seed/setup scripts remain available for a fresh development database.

## Requirements and start

- Node.js 20 or newer
- PostgreSQL 14 or newer (Neon works; local PostgreSQL is also supported)

```powershell
cd path\to\potko-api
npm install
Copy-Item .env.example .env
# Edit .env: set DATABASE_URL and a private JWT_SECRET
npm run db:setup
npm run dev
```

`SETUP_CODE` is used to authorize the first-run admin/restaurant setup at `POST /auth/setup`; keep it private and configure it before initializing a new database.

The API listens at `http://localhost:4000` by default. Check `GET /health` for a database-independent process health response. To use the bundled PGlite local database, open a second terminal and run `npm run db:local`; use the commented local `DATABASE_URL` in `.env`. Run the setup command once for a new database. Use `npm run db:migrate` to apply new migrations to an existing database.

Available database scripts: `db:local`, `db:schema`, `db:setup`, `db:migrate`, `db:reset`, `db:clear`, and `restaurant:add`. Read each script before using destructive commands such as reset/clear. `npm start` starts the server without file watching.

## Connect the React/Vite dashboard

The API is browser-accessible. Set `CORS_ORIGINS` to the exact origins used by Vite, the deployed web app, and any desktop webview origin. For local development, the example allows Vite's `localhost:5173`, `127.0.0.1:5173`, and preview `localhost:4173`. Set a comma-separated list for deployment; `*` is supported for local-only testing. CORS is not authentication: all protected requests still need a bearer token.

```ts
const API_URL = import.meta.env.VITE_API_URL ?? "http://localhost:4000";

const response = await fetch(`${API_URL}/auth/login`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ login_id: "...", password: "..." }),
});
const session = await response.json();
// Keep the token in the app's session/auth store, then:
const me = await fetch(`${API_URL}/me`, {
  headers: { Authorization: `Bearer ${session.token}` },
});
```

The precise login request fields and response are defined in `src/routes/auth.js`; do not copy this abbreviated illustration as a schema. A successful login token is required on every route except `/health` and `/auth/*`. Authenticated REST calls use `Authorization: Bearer <token>`. JSON errors have `{ error, message, ... }` shape. Numeric SQL values may be JSON numbers. The server scopes reads and writes by the authenticated user's restaurant and role permissions; client-side route hiding does not replace those checks.

For React Query, make the token part of the session lifecycle, add it to the shared request function, and clear query caches on logout or account switch. Never put a database URL or JWT signing secret in a `VITE_*` variable; Vite variables ship to browsers.

### Concrete response and message shapes

Successful `POST /auth/login` returns `{ token, user, restaurant }`. The login JSON body is `{ loginId, password, role?, otp? }` (camelCase). `user` includes `id`, `name`, `login_id`, `role`, `restaurant_id`, `permissions`, `access`, `terminals`, and account flags. Use the returned token in the Authorization header; do not treat the role selector as authorization.

`GET /sync` returns an object with `{ floors, areas, tables, terminals, orders, approvals, readyCount, serverTime }`. Use the response as a bootstrap snapshot, then apply authorized websocket events. `GET /menu?all=1` returns `{ categories, items }` scoped to that restaurant; category and item fields are the source server's snake_case shape (for example category `item_count`, `active`, `sort`, and item `category_id`, `price`, `tax_rate`, `available`, `terminal_ids`). The route's `menuFor` / `itemFor` serializers can omit fields according to permissions.

Example menu read:

```ts
const menu = await apiGet<{ categories: Category[]; items: MenuItem[] }>("/menu?all=1");
```

Example realtime order creation (waiter role only; a dine-in table must first be locked by that user):

```ts
socket.send(JSON.stringify({
  type: "order.create",
  id: crypto.randomUUID(),
  payload: {
    type: "takeaway",
    customerName: "Priya Sharma",
    items: [{ menuItemId: "<menu-item-id>", qty: 2, size: "regular", addons: [], note: "Less spicy" }],
  },
}));
```

The server computes prices and totals from database menu/settings data; do not submit a client total as the source of truth. Order creation payload uses lowercase values (`dine_in`, `delivery`, `takeaway`) even though the dashboard preview fixtures use values such as `DINE_IN`. Treat the websocket ack as the command result and the subsequent scoped events/snapshot as shared state.

## Contract fit with `outputs/potko-pos`

The extracted dashboard is also React + Vite, but its current `lib/api.ts` is an in-memory demo adapter. It uses a different legacy contract: paths such as `/dashboard/summary`, `/inventory/ingredients`, `/attendance`, `/customers`, `/reservations`, `/accounting/*`, and `/operations/*`, plus branch-oriented DTOs. This server's source contract uses restaurant-scoped `/overview`, `/reports/*`, `/menu`, `/users`, `/floors`, `/areas`, `/tables`, `/terminals`, `/orders`, and WebSocket commands. Similar names do not guarantee equivalent fields or behavior.

So the React app can connect at the HTTP/WebSocket transport level, but changing its base URL alone will not make every page work against this API. Build a typed adapter in the React app (or add explicitly versioned compatibility routes here) that maps each page's view model to the source API, and use the source WebSocket protocol for writes and live updates. Do not map a feature to a fake empty list or success response. The current backend has no persisted inventory, attendance, reservations, payroll, accounting, delivery management, customer CRM, marketing, or operations equipment/document modules. Those require domain/schema/API work before those dashboard sections can operate on backend data.

The strongest existing overlaps are menu/categories, orders and bill/payment flow, kitchen tickets, floor/areas/tables, terminals, staff/users, approvals, restaurant settings, and reports. The React dashboard currently includes sections beyond this list. Preserve demo mode as the default until an integration adapter is ready; configure `VITE_API_URL` only after adding that adapter.

## REST API inventory

All routes listed below except `/health` and `/auth/*` require a valid bearer token. Permissions and role restrictions are enforced in the source route handlers.

| Area | Routes | Purpose |
| --- | --- | --- |
| Health | `GET /health` | Process/version status; does not check database readiness |
| Authentication | `POST /auth/login`, `GET/POST /auth/setup`, `GET /auth/dev`, `POST /auth/dev-login` | Login, first-run setup, development helpers (review their environment gating before deployment) |
| Current user/session | `GET /me`, `/me/getting-started`, `POST /me/pin`, `/me/password`, `GET /me/sessions`, `POST /me/sessions/:id/logout`, `/me/logout`, `/me/logout-all` | Identity, setup progress, credentials, and session controls |
| Snapshot | `GET /sync` | Role-filtered initial POS state for reconnect/bootstrap |
| Menu | `GET /menu`, `/menu/search`, `/menu/popular`; item/category CRUD and ordering under `/menu/items` and `/menu/categories` | Menu data, availability, category management |
| Menu import | `GET /menu/import/template`, `POST /menu/import/preview`, `POST /menu/import/:id/commit`, `GET /menu/import/:id/errors.csv` | Spreadsheet import preview/commit/error report |
| Images | `POST /images`, `GET /images/:id` | Upload/retrieve image records |
| Orders | `GET /orders/recent-customers`, `POST /orders/:id/receipt`; manager order history via `GET /orders`, `/orders/:id` | Receipts and historical/order detail reads |
| Users | `GET /users`, `/users/:id`, `/users/:id/orders`; `POST /users`; `PATCH /users/:id`; `POST /users/:id/logout`, `POST /users/:id/otp`, `DELETE /users/:id` | Staff listing, management, and user order history |
| Reports | `GET /reports/overview`, `/sales`, `/menu`, `/orders`, `/staff`, `/staff/:id`, `/daily` | Restaurant-scoped operating reports |
| Restaurant/settings | `PATCH /restaurant`; `GET /overview` | Restaurant profile/settings and manager overview |
| Approvals/audit | `GET /approvals`, `/approvals/:id`, `/audit` | Discount/cancel/refund approvals and audit history |
| Floor | `/floors`, `/areas`, `/tables`, `/terminals` | GET/POST/PATCH/DELETE management; route-specific move, layout, ordering, chef/item assignment actions are in `src/routes/floor.js` |

The route files under `src/routes/` are authoritative for request query parameters, JSON bodies, validation, permissions, and response fields. This table is an index, not a replacement for the route schemas. The source API does not expose a generic CRUD endpoint for every entity in the dashboard.

## Realtime WebSocket API

Connect to `ws://localhost:4000/ws?token=<JWT>` (use `wss://` in production). The server sends `hello` on a valid connection. Client messages are JSON `{ "type": "action.name", "payload": {}, "id": "unique-action-id" }`; responses are `{ "type": "ack", "id": "...", "ok": true, "payload": ... }` or an ack with `ok:false`, `error`, and `message`. Use unique action IDs and handle errors. Repeated IDs are idempotent for a short server cache. The server emits scoped update events to connected clients; see `src/ws/publish.js` and `src/ws/events.js` for exact broadcast envelopes and filters. Send `{ "type": "ping" }` periodically (the mobile client uses 25 seconds) and handle `{ "type": "pong" }`.

Commands registered in the source:

- Orders/items: `order.create`, `order.addItems`, `item.edit`, `item.remove`, `waiter.deliver`
- Tables: `table.lock`, `table.release`, `table.split`, `table.merge`, `table.unjoin`, `table.transfer`
- Kitchen: `chef.accept`, `chef.deny`, `chef.start`, `chef.itemReady`, `chef.allReady`
- Billing/payment/approval: `discount.apply`, `approval.request`, `approval.decide`, `bill.request`, legacy alias `table.close`, `payment.add`, `payment.mark`, `order.close`, `order.handover`, `order.reopen`, `payment.refund`

Use `/sync` after initial login or reconnect, then apply realtime events to cached state. Do not invent event payloads: payload validation and role requirements are implemented beside each handler in `src/ws/*.js`. Waiter area/table scopes and chef terminal assignments are enforced in the server; sensitive prices/bill totals are filtered from replies where the user's permissions do not allow them.

## Data model and migrations

PostgreSQL schema source is `sql/schema.sql`; seed data is in `sql/seed.sql`; incremental migrations are `sql/migrations/*.sql`. Tables cover restaurants and users, sessions, floors/areas/tables/terminals and staff assignments, menu categories/items, orders/order items/KOTs, payments, approvals, audit events, and supporting settings/import records. Inspect the DDL before creating an integration DTO: names and snake_case database columns are not automatically the JSON API shape. Route modules assemble API responses and may transform fields.

When evolving the contract:

1. Add a forward-only SQL migration with the next numbered filename.
2. Add/update the relevant route or WebSocket handler, request validation, permission checks, restaurant scoping, transaction, and audit entry as appropriate.
3. Keep API response fields documented and coordinate a typed frontend adapter.
4. Update `/sync` if a new entity is required during POS bootstrap, and add scoped broadcast events if live screens need them.
5. Keep secrets, auth tokens, password/PIN hashes, and cross-restaurant records out of client responses.

Money and status fields have domain-specific semantics (discount approvals, payment closing, order lifecycle, role scoping). Reuse the existing helpers in `src/` instead of implementing parallel calculations in React.

## Source layout

- `src/index.js`: Express app, CORS, body limits, REST mount points, WebSocket attachment, migration/startup
- `src/routes/`: authenticated REST APIs
- `src/ws/`: authenticated realtime actions, event publication, and role-scoped updates
- `src/auth.js`, `src/permissions.js`, `src/scope.js`: identity and authorization
- `src/db.js`, `src/migrate.js`, `src/scripts/`: PostgreSQL connection and database lifecycle
- `src/orders.js`, `src/bill.js`, `src/tables.js`, `src/settings.js`: domain operations and rules
- `sql/`: full schema, seed, and migrations
- `test/`: source backend test suite

## Deployment notes

Set `DATABASE_URL`, a private random `JWT_SECRET`, `PORT` (usually supplied by the host), and production `CORS_ORIGINS`. Terminate TLS at the host or reverse proxy and use HTTPS/WSS from browsers. Keep `.env` out of version control. Do not expose development login/setup endpoints in a public production deployment; inspect `src/routes/auth.js` and deployment configuration before enabling those flows. The health endpoint indicates process liveness only, not database readiness.

The copied `src/`, `sql/`, and `test/` directories preserve upstream implementation behavior, with configurable CORS and restrictive local development defaults as the noted extraction change. See the upstream repository history for subsequent changes.
