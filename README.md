<p align="center">
  <img src="vouched_logo.svg" alt="Vouched Logo" width="320" />
</p>

# Vouched

A backend API for collecting verified client reviews through single-use tokens. Only clients who receive a private link can submit a review — making every testimonial cryptographically unforgeable.

## How it works

```
Project completed → generate a unique token tied to the client
                  → send the client a private link containing that token
                  → client opens the link and submits a review
                  → system validates the token (exists, not expired, not yet used)
                  → review is saved as pending, token is permanently burned
                  → you review it and approve (or reject) it
                  → once approved, the review appears in the portfolio with a "verified" badge
```

The system rests on two guarantees:

- **Authenticity** comes from the token. A UUID v4 has 2¹²² possible values, so guessing a valid one by brute force is computationally infeasible. Even a randomly generated UUID is rejected — only explicitly created tokens exist in the database, and each one is single-use and time-limited.
- **Editorial control** comes from manual moderation. No review goes public automatically. Every submission lands in a `pending` state and appears on the portfolio only after you approve it — so even a leaked token can't push spam live without your sign-off.

## Stack

| Layer | Technology |
|---|---|
| Framework | [NestJS](https://nestjs.com) |
| Language | TypeScript |
| ORM | TypeORM |
| Database | SQLite (single file, no server) — Postgres-ready |
| Auth | JWT via Passport.js |
| Validation | class-validator |
| Email | Manual for v1 — [Resend](https://resend.com) integration planned for v2 |

## Design decisions

Some choices here are driven by the real scale of the problem; others are
deliberate portfolio showcases. Being explicit about which is which:

- **SQLite, not PostgreSQL.** The workload is tiny — one freelancer, a handful
  of tokens and reviews per month, a single writer. SQLite is a single file
  with no server to run, no Docker, no connection config, and nothing to "keep
  running". TypeORM keeps the code database-agnostic, so moving to Postgres
  later is a one-line driver change. Relational modelling (entities, foreign
  keys, one-to-one relations, atomic transactions) is demonstrated exactly the
  same way.
- **JWT + Passport.js for auth — a showcase, not a necessity.** With a single
  admin, a static bearer key in an environment variable would be enough. Full
  JWT authentication (login endpoint, bcrypt-hashed password, signed
  short-lived tokens, Passport strategy) is implemented because it is a
  standard backend skill worth demonstrating, and because it makes the
  security reasoning behind the project concrete.
- **Manual review-link delivery for v1.** `POST /tokens` returns the private
  link; you send it to the client however you already talk to them. Automated
  email (Resend) adds a third-party account, an API key, DNS records for
  deliverability, and send-failure handling — disproportionate for a few
  clients a year. It is planned as a v2 feature.
- **Manual moderation.** Not overhead — it is the second half of the trust
  model. Even a leaked token cannot publish anything without an explicit
  approval.

## API

### Public

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/reviews` | List approved reviews only |
| `POST` | `/reviews` | Submit a review using a valid token |

### Admin (JWT required)

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/auth/login` | Obtain a JWT |
| `POST` | `/tokens` | Generate a token for a client and return the private review link |
| `GET` | `/tokens` | List all tokens (active and used) |
| `DELETE` | `/tokens/:id` | Delete a token |
| `GET` | `/reviews/pending` | List reviews awaiting approval |
| `PATCH` | `/reviews/:id/approve` | Approve a pending review |
| `DELETE` | `/reviews/:id` | Reject and remove a review |

## Project structure

```
src/
├── app.module.ts
├── main.ts
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── jwt.strategy.ts
│   └── jwt-auth.guard.ts
├── tokens/
│   ├── tokens.module.ts
│   ├── tokens.controller.ts
│   ├── tokens.service.ts
│   ├── token.entity.ts
│   └── dto/
│       └── create-token.dto.ts
└── reviews/
    ├── reviews.module.ts
    ├── reviews.controller.ts
    ├── reviews.service.ts
    ├── review.entity.ts
    └── dto/
        └── create-review.dto.ts
```

## Getting started

```bash
# Install dependencies
npm install

# Copy and configure environment variables
cp .env.example .env

# Start in development mode
npm run start:dev
```

### Environment variables

```env
NODE_ENV=development

# SQLite database file. In production, point this at a path on a persistent
# volume, or switch the driver to Postgres — the code stays database-agnostic.
DB_FILE=vouched.sqlite

# Generate with: openssl rand -hex 32
JWT_SECRET=
JWT_EXPIRES_IN=8h

ADMIN_EMAIL=your@email.com
# bcrypt hash of the password, not the plaintext
ADMIN_PASSWORD_HASH=$2b$12$...

# How long a token stays valid after it's issued
TOKEN_TTL_HOURS=168

# Base URL of the portfolio page that opens the review form. Fixed value —
# never derived from the request Host header.
FRONTEND_URL=https://yourportfolio.com
```

> The admin password is never stored in plaintext — only its bcrypt hash. Generate the hash once and put it in `ADMIN_PASSWORD_HASH`. The `JWT_SECRET` must be long and random; anything guessable lets an attacker forge admin tokens.

## Security

- Admin endpoints are protected by JWT bearer authentication, with short-lived tokens signed by a high-entropy secret
- Admin credentials are never stored in plaintext — only a bcrypt hash of the password
- The login endpoint is rate-limited to blunt brute-force attempts
- Review submission requires a valid token that is single-use and time-limited; tokens are burned immediately upon use, so replay attacks are not possible
- Token validation and burn happen in a single transaction, so a token can never be consumed twice under concurrent requests
- Reviews are moderated: nothing is publicly visible until approved, and the public endpoint filters by approval status at the query level
- Public review responses never expose internal data (token UUID, client email)
- Input validation enforced via `ValidationPipe` with `whitelist: true` and `forbidNonWhitelisted: true`, blocking mass-assignment of fields like the approval flag
- Every string field has an explicit maximum length, so a valid token cannot be used to store an oversized payload
- The JWT strategy pins the signing algorithm (`HS256`) and the token is only ever read from the `Authorization` header, never a cookie — so CSRF does not apply
- The review link is single-use and short-lived; the portfolio page sends `Referrer-Policy: no-referrer` and strips the token from the URL on load, so it does not leak through logs or the `Referer` header
- Review text is stored raw and escaped by the portfolio at render time (never `innerHTML`); incoming HTML is also stripped as defense in depth
- CORS is restricted to the portfolio origin, and `helmet` sets baseline security headers including HSTS

The reasoning behind each of these — the attack it stops and where the defense
lives in the request pipeline — is documented alongside the implementation.

## License

MIT © [EmaLica](https://github.com/EmaLica)
