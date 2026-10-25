# RYVON backend (Node.js + Express + PostgreSQL + Prisma)

Adds a real backend next to the existing RYVON frontend. **No frontend file was changed.**
Express also serves the existing `index.html`, `pages/` and `assets/` at the same URLs as before.

```
Ryvon Bike Rental/
├── index.html, pages/, assets/     ← your frontend, untouched
└── server/                         ← NEW
    ├── prisma/  schema.prisma, migrations/, seed.js, data/
    ├── src/     app.js, server.js, routes/, services/, middleware/, schemas/, utils/
    ├── tests/   unit + integration tests
    └── .env.example, .gitignore, package.json
```

## 1. Prerequisites
Node.js 20.10+ and PostgreSQL 13+ (`btree_gist` is required; it ships with PostgreSQL).

## 2. Create the database
```bash
psql -U postgres
```
```sql
CREATE USER ryvon WITH PASSWORD 'choose-a-strong-password';
CREATE DATABASE ryvon OWNER ryvon;
\q
```

## 3. Install and configure
```bash
cd server
npm install
cp .env.example .env            # Windows (cmd): copy .env.example .env
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"   # copy output into JWT_SECRET
```
Edit `.env`: set `DATABASE_URL` (your password) and `JWT_SECRET`. The server refuses to start if `JWT_SECRET` is missing or shorter than 32 characters.

## 4. Migrate, seed, run
```bash
npx prisma generate
npx prisma migrate deploy       # creates tables + overlap constraint
npm run seed                    # loads the 34 bikes and 7 extras from your frontend data (safe to re-run)
npm start                       # or: npm run dev
```
Open http://localhost:3000 (frontend) and http://localhost:3000/api/health (backend + DB check).

If `migrate deploy` fails on `CREATE EXTENSION btree_gist`, run once as a superuser:
`psql -U postgres -d ryvon -c "CREATE EXTENSION IF NOT EXISTS btree_gist;"`

## 5. Tests
Use a **separate** empty database whose name contains `test` (the tests delete users/bookings and refuse to run otherwise).
```bash
psql -U postgres -c "CREATE DATABASE ryvon_test OWNER ryvon;"
DATABASE_URL=postgresql://ryvon:PASSWORD@localhost:5432/ryvon_test npx prisma migrate deploy
TEST_DATABASE_URL=postgresql://ryvon:PASSWORD@localhost:5432/ryvon_test npm test
```
PowerShell: `$env:DATABASE_URL="..."; npx prisma migrate deploy; $env:TEST_DATABASE_URL="..."; npm test`.
Without `TEST_DATABASE_URL`, only the unit tests run.

## API (all responses: `{ "data": ... }` or `{ "error": { "code", "message", "details?" } }`)
Authenticated routes need `Authorization: Bearer <token>` (from register/login).

| Method & path | Auth | Purpose |
|---|---|---|
| `POST /api/auth/register` | – | `{name,email,password,phone?,licenseNumber?}` → 201 `{user,token}` |
| `POST /api/auth/login` | – | `{email,password}` → `{user,token}` |
| `GET /api/auth/me` | ✔ | current user |
| `PATCH /api/users/me` | ✔ | update `name, phone, licenseNumber, avatar(https URL)` only |
| `GET /api/bikes` | – | filters: `type, brand, search, available, sort(price_asc\|price_desc\|rating), limit, offset`; same field names as the frontend's `bikes[]` |
| `GET /api/bikes/:id` | – | bike details |
| `GET /api/bikes/:id/availability?start=&end=` | – | `{available, reason}` |
| `GET /api/extras` | – | rental extras |
| `POST /api/bookings/quote` | – | server-side price + availability, writes nothing |
| `POST /api/bookings` | ✔ | create booking (`bikeId,startDate,endDate,pickupHub,pickupTime?,paymentMode?,extras?[],couponCode?,destination?,source?`) |
| `GET /api/bookings?status=&limit=&offset=` | ✔ | own history, newest first; `status` = Upcoming / Active / Completed / Cancelled |
| `GET /api/bookings/:id` | ✔ | own booking only (others' → 404) |
| `PATCH /api/bookings/:id/cancel` | ✔ | cancel own booking before its start date |

Dates are `YYYY-MM-DD`. Common errors: 400 validation, 401 auth, 404 not found, 409 conflict (`BIKE_NOT_AVAILABLE`, `BIKE_UNAVAILABLE`, `EMAIL_TAKEN`, `ALREADY_CANCELLED`, `CANNOT_CANCEL`), 429 rate limit.

## How the important parts work
- **Price** is computed only on the server (same formula the frontend shows: days × rate + extras×days + ₹149 service fee, 18% GST; `WANDER20` = 20% off the rental). Client-sent prices are ignored.
- **Days**: same-day rental bills 1 day, otherwise `end − start` (matches the frontend).
- **No double bookings**: PostgreSQL exclusion constraint `bookings_no_overlap` on `(bikeId, daterange(start,end,'[]'))` for non-cancelled bookings. Start **and end days are inclusive**, so a bike returned on the 15th can be picked up again from the 16th. Inserts are also serialized per bike with an advisory lock to avoid deadlocks. Cancelling frees the dates.
- **Status**: only `CONFIRMED`/`CANCELLED` are stored; Upcoming/Active/Completed are derived from dates in `BUSINESS_TIMEZONE`.
- **Security**: bcrypt (cost 12, via `bcryptjs`), passwords limited to 72 bytes, HS256 JWT with pinned algorithm, generic login errors + constant-time compare for unknown emails, auth rate limiting, Helmet on `/api`, zod validation with unknown fields stripped, owner-scoped booking queries, no stack traces in responses, only `index.html`/`pages`/`assets` are served statically (never `server/`).

## Verified vs not verified
**Verified** (27 automated tests, run against a real PostgreSQL 16, 3 consecutive passes): registration/login/JWT rejection cases, validation, pricing, every overlap shape, 15 simultaneous identical bookings → exactly 1 succeeds, 20 simultaneous overlapping ranges → no overlapping rows, cancel + rebook race, authorization between users, history filters, DB constraints with raw SQL, error bodies, startup failure modes, seed idempotency, static frontend serving.

**Not verified — please check on your machine:**
- `npx prisma migrate deploy` / `migrate dev`: my sandbox could not download Prisma's CLI engines, so I applied the identical `migration.sql` with `psql` instead. If `npx prisma migrate dev` ever proposes a new migration, run `npx prisma migrate diff --from-migrations prisma/migrations --to-schema-datamodel prisma/schema.prisma --shadow-database-url <scratch db url>` and tell me what it reports.
- Tested only on Linux; Windows/macOS commands above are standard but unrun.
- CORS settings, `TRUST_PROXY`, and the rate-limit rejection (429) path were not exercised.

## Not included / limitations
- **The frontend still uses localStorage.** Nothing in `assets/js/app.js` calls this API yet (you asked to preserve it). Wiring is a separate step.
- **Payments are not integrated.** Bookings store `paymentStatus = PENDING`; nothing is charged.
- No email verification, password reset, refresh tokens, admin role, wishlist or contact-form endpoints.
- Registering with an existing email returns 409, which reveals that the email has an account.
- The token is a Bearer token the frontend must store (e.g. localStorage); XSS on the site could read it.
- No idempotency key: a client retrying a timed-out `POST /api/bookings` could get a 409 against its own first booking.
