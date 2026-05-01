# Vouched

A backend API for collecting verified client reviews through single-use tokens. Only clients who receive a private link can submit a review — making every testimonial cryptographically unforgeable.

## How it works

```
Project completed → generate a unique token tied to the client
                  → send the client a private link containing that token
                  → client opens the link and submits a review
                  → system validates the token (exists, not yet used)
                  → review is saved, token is permanently burned
                  → review appears in the portfolio with a "verified" badge
```

A UUID v4 token has 2¹²² possible values. Guessing a valid token by brute force is computationally infeasible. Even if someone generated a random UUID, the system would reject it — only explicitly created tokens exist in the database.

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
| `GET` | `/reviews` | List all published reviews |
| `POST` | `/reviews` | Submit a review using a valid token |

### Admin (JWT required)

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/auth/login` | Obtain a JWT |
| `POST` | `/tokens` | Generate a token for a client |
| `GET` | `/tokens` | List all tokens (active and used) |
| `DELETE` | `/tokens/:id` | Delete a token |

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

JWT_SECRET=your-secret-key

ADMIN_EMAIL=your@email.com
ADMIN_PASSWORD=yourpassword

RESEND_API_KEY=re_...
FRONTEND_URL=https://yourportfolio.com
```

## Security

- Admin endpoints are protected by JWT bearer authentication
- Review submission requires a valid single-use token
- Tokens are burned immediately upon use — replay attacks are not possible
- Public review responses never expose internal token data
- Input validation enforced on all endpoints via `ValidationPipe` with `whitelist: true`

## License

MIT © [EmaLica](https://github.com/EmaLica)
