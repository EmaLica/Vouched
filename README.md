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
| Database | PostgreSQL |
| Auth | JWT via Passport.js |
| Validation | class-validator |
| Email | Resend |

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
| `POST` | `/tokens` | Generate a token for a client |
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
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=yourpassword
DB_DATABASE=vouched

# Generate with: openssl rand -hex 32
JWT_SECRET=
JWT_EXPIRES_IN=8h

ADMIN_EMAIL=your@email.com
# bcrypt hash of the password, not the plaintext
ADMIN_PASSWORD_HASH=$2b$12$...

# How long a token stays valid after it's issued
TOKEN_TTL_HOURS=168

RESEND_API_KEY=re_...
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
- CORS is restricted to the portfolio origin, and `helmet` sets baseline security headers

## License

MIT © [EmaLica](https://github.com/EmaLica)
