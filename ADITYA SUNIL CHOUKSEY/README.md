# Car Rental & Fleet Booking System API

A RESTful API for managing a rental car fleet and customer bookings, built with
**Node.js + Express** and **Supabase (PostgreSQL + Auth)**.

It handles vehicle inventory, authenticated users, and the core booking logic:
server-side cost calculation and double-booking (date-collision) prevention.

---

## Tech Stack

- **Node.js + Express.js** — HTTP server & routing
- **@supabase/supabase-js** — PostgreSQL data access + Auth
- **dotenv** — environment configuration
- **cors** — cross-origin support
- **nodemon** (dev) — auto-reload

---

## Project Structure

```
assignment-10-car-rental-api/
├── config/
│   └── supabase.js            # Supabase client (warns, does not crash, if creds missing)
├── controllers/
│   ├── authController.js      # register / login via Supabase Auth
│   ├── rentalController.js    # booking, collision check, cancel, complete
│   └── vehicleController.js   # fleet CRUD + filters
├── middleware/
│   ├── auth.js                # Bearer token verification via supabase.auth.getUser
│   └── errorHandler.js        # centralized JSON error shape + success/error helpers
├── routes/
│   ├── authRoutes.js
│   ├── rentalRoutes.js
│   └── vehicleRoutes.js
├── utils/
│   ├── rentalUtils.js         # pure cost-calc + overlap helpers
│   └── rentalUtils.test.js    # zero-dependency unit tests
├── migrations/
│   └── schema.sql             # paste into the Supabase SQL editor
├── postman/
│   └── car-rental-api.postman_collection.json
├── .env.example
├── .gitignore
├── package.json
├── server.js
└── README.md
```

---

## Supabase Setup

1. **Create a project** at [supabase.com](https://supabase.com) (free tier is fine).
2. **Run the schema.** Open your project → **SQL Editor** → paste the entire
   contents of [`migrations/schema.sql`](migrations/schema.sql) → **Run**. This
   creates the `vehicles` and `rentals` tables plus helpful indexes.
   - You do **not** create a users table. `auth.users` is created and managed
     automatically by **Supabase Auth**. `rentals.user_id` references it.
3. **Copy your credentials.** Project → **Settings → API**:
   - **Project URL** → `SUPABASE_URL`
   - **Project API keys → anon public** → `SUPABASE_ANON_KEY`
4. **Configure `.env`.** Copy `.env.example` to `.env` and paste the values:
   ```
   cp .env.example .env
   ```
5. *(Optional)* For easy local testing, disable email confirmation in
   **Authentication → Providers → Email** so `register` returns a usable session
   immediately. Otherwise, confirm the email before `login`.

> **No live Supabase project?** The server still boots. `config/supabase.js`
> prints a clear startup **WARNING** (it does not crash) when
> `SUPABASE_URL`/`SUPABASE_ANON_KEY` are missing. Requests that hit the database
> will then fail gracefully with a clean JSON error until real credentials are
> provided.

---

## Environment Variables

| Variable            | Required | Description                                  |
| ------------------- | -------- | -------------------------------------------- |
| `SUPABASE_URL`      | yes      | Supabase project URL                         |
| `SUPABASE_ANON_KEY` | yes      | Supabase anon/public API key                 |
| `PORT`              | no       | Port for the Express server (default `3000`) |

---

## How to Run

```bash
# 1. Install dependencies
npm install

# 2. Run the unit tests (no DB needed)
npm test

# 3. Start in development (auto-reload)
npm run dev

# ...or production
npm start
```

The API listens on `http://localhost:3000` by default.

---

## API Endpoints

Base URL: `http://localhost:3000`

All responses share the shape `{ success, message, data? }`.

### Auth

| Method | Path                 | Auth | Description                                             |
| ------ | -------------------- | ---- | ------------------------------------------------------- |
| POST   | `/api/auth/register` | no   | Sign up `{ name, email, password }`. 201 / 400.         |
| POST   | `/api/auth/login`    | no   | Sign in `{ email, password }`. Returns `access_token`. 401 on bad creds. |

Send the returned token on protected routes as `Authorization: Bearer <access_token>`.

### Vehicles

| Method | Path                 | Auth | Description                                                   |
| ------ | -------------------- | ---- | ------------------------------------------------------------ |
| GET    | `/api/vehicles`      | no   | List vehicles. Filters: `?category=` and `?status=` (combinable). |
| GET    | `/api/vehicles/:id`  | no   | Single vehicle **including its past rental records**. 404 if missing. |
| POST   | `/api/vehicles`      | yes  | Create a vehicle. Validates required fields + `daily_rate > 0`. 201 / 400. |
| PUT    | `/api/vehicles/:id`  | yes  | Partial update (`daily_rate`, `status`, …). 404 if missing.  |
| DELETE | `/api/vehicles/:id`  | yes  | Delete. 400 `"Has Active Bookings"` if any rental is `booked`/`active`. |

### Rentals

| Method | Path                          | Auth | Description                                                        |
| ------ | ----------------------------- | ---- | ----------------------------------------------------------------- |
| POST   | `/api/rentals`                | yes  | Create a booking. Validates dates, checks collisions, computes cost. 201 / 400. |
| GET    | `/api/rentals/my-bookings`    | yes  | Only the authenticated user's rentals.                            |
| PATCH  | `/api/rentals/:id/cancel`     | yes  | Owner only; only if `booked` **and** `start_date` in the future. Else 400 `"Cannot Cancel"`. |
| PATCH  | `/api/rentals/:id/complete`   | yes  | Owner only; marks rental `completed` and vehicle `available`.     |

**Booking request body:**
```json
{
  "vehicle_id": 1,
  "start_date": "2026-05-01",
  "end_date": "2026-05-05",
  "customer_name": "Ada Lovelace",
  "customer_email": "ada@example.com"
}
```

---

## Postman Collection

Import [`postman/car-rental-api.postman_collection.json`](postman/car-rental-api.postman_collection.json).
It runs the full flow in order and auto-captures variables between steps:

1. Register
2. Login (sets `access_token`)
3. Create Vehicle (sets `vehicle_id`)
4. List Vehicles (filtered)
5. Get Vehicle by id (includes rentals)
6. **Book Vehicle** `2026-05-01 → 2026-05-05` (asserts `total_cost = 300`)
7. **Colliding Booking** `2026-05-03 → 2026-05-07` — **asserts HTTP 400** and the
   message `"Vehicle already reserved during this timeframe"`
8. Book Vehicle #2 (future dates, to complete)
9. Cancel Booking
10. Complete Booking
11. My Bookings

> Set the `baseUrl` collection variable if you are not on `http://localhost:3000`.
> For step 2 (Login), use the email you actually registered (Supabase may require
> email confirmation depending on your project settings).

---

## Booking Logic (the interesting part)

Extracted into pure, unit-tested helpers in [`utils/rentalUtils.js`](utils/rentalUtils.js):

- **Collision check** — a new booking conflicts with any existing `booked`/`active`
  rental for the same vehicle when the date ranges overlap, i.e.
  `NOT (new.end < existing.start OR new.start > existing.end)`. This correctly
  catches partial overlaps (existing `05-01→05-05` vs new `05-03→05-07`).
- **Cost calculation** — always computed **server-side** as
  `days(end − start) × vehicles.daily_rate`. The client's total is never trusted.

Run `npm test` to execute the assertions (no database required).

---

## AUTHOR

## Aditya Sunil Chouksey

## 150096725070

## SAM ALTMAN
