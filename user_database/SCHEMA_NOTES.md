# User Database schema notes (JWT architecture)

## Port normalization
This MySQL container is standardized to **port 5001** (see `startup.sh` and `db_connection.txt`).

To connect locally (canonical command):
- `$(cat db_connection.txt)`

## Auth persistence responsibility (important)
- **API Gateway** owns **registration/login** and persists users in MySQL (`users` table).
- **Application Server (user_backend)** trusts `Authorization: Bearer <JWT>` and does **not** manage user credentials.

Both services may still connect to the same MySQL DB for other data (business tables).

## Required table: `users` (final)
Fields (required):
- `id` (PK, auto-increment)
- `email` (VARCHAR(255) UNIQUE NOT NULL)
- `password_hash` (VARCHAR(255) NOT NULL)
- `created_at` (TIMESTAMP DEFAULT CURRENT_TIMESTAMP)

### One-statement-at-a-time CLI (canonical: uses `db_connection.txt`)
Run from the `user_database/` directory:

1) Create table (single statement):
- `$(cat db_connection.txt) -e "CREATE TABLE IF NOT EXISTS users (id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT, email VARCHAR(255) NOT NULL, password_hash VARCHAR(255) NOT NULL, created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (id));"`

2) Add unique index if missing (single statement):
- `$(cat db_connection.txt) -e "ALTER TABLE users ADD UNIQUE KEY uq_users_email (email);"`

Note: If the unique key already exists, step (2) will fail with an "already exists" error; that is expected.

### Minimal helper SQL (via CLI only; no .sql files)
- Fetch a user row for login verification:
  - `$(cat db_connection.txt) -e "SELECT id, email, password_hash FROM users WHERE email='test@example.com' LIMIT 1;"`

- Insert a new user (provide a bcrypt hash from Node):
  - `$(cat db_connection.txt) -e "INSERT INTO users (email, password_hash) VALUES ('test@example.com', '<PASTE_BCRYPT_HASH_HERE>');"`

## Minimal seed / test user creation (one-statement-at-a-time)
If you need a test user, create a bcrypt hash in Node and insert it.

1) Generate a bcrypt hash (example):
- `node -e "const b=require('bcryptjs'); b.hash('password123',12).then(h=>console.log(h))"`

2) Insert user (single statement):
- `mysql -u appuser -p*** -h localhost -P 5001 myapp -e "INSERT INTO users (email, password_hash) VALUES ('test@example.com', '<paste_hash_here>');"`

Notes:
- Do not commit real passwords or hashes.
- Email is unique; repeated inserts will fail with duplicate-key error.
