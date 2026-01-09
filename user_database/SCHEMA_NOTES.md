# User Database schema notes (JWT architecture)

## Port normalization
This MySQL container is standardized to **port 5001** (see `startup.sh` and `db_connection.txt`).

To connect locally (canonical command):
- `$(cat db_connection.txt)`

## Auth persistence responsibility (important)
- **API Gateway** owns **registration/login** and persists users in MySQL (`users` table).
- **Application Server (user_backend)** trusts `Authorization: Bearer <JWT>` and does **not** manage user credentials.

Both services may still connect to the same MySQL DB for other data (business tables).

## Required table: `users`
Fields:
- `id` (auto-increment primary key)
- `email` (unique)
- `password_hash`
- `created_at`
- `updated_at`

The table is created using one SQL statement (already applied in this environment):
- `CREATE TABLE IF NOT EXISTS users (...)`

Indexes/constraints expected:
- `PRIMARY KEY (id)`
- `UNIQUE KEY uq_users_email (email)`
- `KEY idx_users_created_at (created_at)`
- `KEY idx_users_updated_at (updated_at)`

## Minimal seed / test user creation (one-statement-at-a-time)
If you need a test user, create a bcrypt hash in Node and insert it.

1) Generate a bcrypt hash (example):
- `node -e "const b=require('bcryptjs'); b.hash('password123',12).then(h=>console.log(h))"`

2) Insert user (single statement):
- `mysql -u appuser -p*** -h localhost -P 5001 myapp -e "INSERT INTO users (email, password_hash) VALUES ('test@example.com', '<paste_hash_here>');"`

Notes:
- Do not commit real passwords or hashes.
- Email is unique; repeated inserts will fail with duplicate-key error.
