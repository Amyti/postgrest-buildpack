# PostgREST Buildpack for Scalingo

A custom buildpack that deploys [PostgREST](https://postgrest.org/) on [Scalingo](https://scalingo.com/), turning your PostgreSQL database into a RESTful API instantly.

## Quick Start

### 1. Create a Scalingo App

```bash
scalingo create my-postgrest-api
```

### 2. Set the Buildpack

```bash
scalingo --app my-postgrest-api env-set BUILDPACK_URL=https://github.com/Scalingo/postgrest-buildpack
```

### 3. Add a PostgreSQL Addon

```bash
scalingo --app my-postgrest-api addons-add postgresql postgresql-starter-512
```

### 4. Find Your PostgreSQL Username

Scalingo automatically creates a PostgreSQL user when you add the addon. Retrieve its name from the connection URL:

```bash
scalingo --app my-postgrest-api env | grep SCALINGO_POSTGRESQL_URL

#Output
postgres://my_app_4242:password@host:port/my_app_4242
#             ^^^^^ this is your PostgreSQL username
```

### 5. Configure Environment Variables

```bash
scalingo --app my-postgrest-api env-set \
  PGRST_DB_URI='$SCALINGO_POSTGRESQL_URL' \
  PGRST_DB_ANON_ROLE='my_app_4242' \
  PGRST_JWT_SECRET='your-secret-jwt-at-least-32-chars' \
  PGRST_SERVER_PORT='$PORT'
```

> **Note:** On Scalingo, use `$PORT` for `PGRST_SERVER_PORT` - the platform dynamically assigns the port via the `PORT` environment variable.

### 6. Deploy

Create an empty commit and push (the buildpack needs no application code):

```bash
git init && git commit --allow-empty -m "Deploy PostgREST"
git remote add scalingo git@ssh.osc-fr1.scalingo.com:my-postgrest-api.git
git push scalingo main
```

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `BUILDPACK_URL` | Yes | URL of this buildpack repository. |
| `PGRST_DB_URI` | Yes | PostgreSQL connection string. |
| `PGRST_DB_ANON_ROLE` | Yes | The database user used for unauthenticated (anonymous) requests. |
| `PGRST_SERVER_PORT` | Yes | The port PostgREST listens on. Must be set to `$PORT` on Scalingo. |
| `PGRST_JWT_SECRET` | No | Secret key used to verify JWT tokens for authenticated requests. Must be at least 32 characters. |
| `POSTGREST_VERSION` | No | Override the PostgREST version to install. Defaults to the latest version. |

### Variable Details

#### `BUILDPACK_URL`

Points Scalingo to this custom buildpack:

```bash
scalingo --app my-app env-set BUILDPACK_URL=https://github.com/Scalingo/postgrest-buildpack
```

#### `PGRST_DB_URI`

The full PostgreSQL connection string. Use the `SCALINGO_POSTGRESQL_URL` variable provided by the addon:

```bash
scalingo --app my-app env-set PGRST_DB_URI='$SCALINGO_POSTGRESQL_URL'
```

#### `PGRST_DB_ANON_ROLE`

The PostgreSQL user PostgREST uses for unauthenticated requests.

> **Important:** On Scalingo, creating new PostgreSQL roles is not supported. You must use the user automatically provided by the PostgreSQL addon - its name is the username part of `SCALINGO_POSTGRESQL_URL`.

```bash
# Retrieve the username from the connection URL
scalingo --app my-app env | grep SCALINGO_POSTGRESQL_URL
# postgres://my_app_4242:password@host:port/my_app_4242

scalingo --app my-app env-set PGRST_DB_ANON_ROLE='my_app_4242'
```


#### `PGRST_SERVER_PORT`

Scalingo assigns a dynamic port via the `PORT` environment variable. PostgREST must bind to it:

```bash
scalingo --app my-app env-set PGRST_SERVER_PORT='$PORT'
```

#### `PGRST_JWT_SECRET`

is the secret key PostgREST uses to verify and decode JWT tokens for authenticating API requests. ([Keycloak](https://scalingo.com/blog/guide-to-deploy-keycloak-on-scalingo),...).

**How it works:**

1. A user authenticates against your auth provider and receives a signed JWT.
2. The client sends the JWT in the `Authorization: Bearer <token>` header.
3. PostgREST verifies the token's signature using `PGRST_JWT_SECRET`.
4. If valid, the request is processed. If the JWT contains a `role` claim, PostgREST uses it to switch the PostgreSQL role for that request.

> **Note on Scalingo:** Since you cannot create new PostgreSQL roles, the `role` claim in the JWT should match your Scalingo-provided user (e.g., `my_app_4242`). Alternatively, omit the `role` claim - PostgREST will fall back to `PGRST_DB_ANON_ROLE`.

The secret must be **at least 32 characters**. For HS256 tokens, configure your auth provider to sign JWTs with this same shared secret. For RS256, use the RSA public key instead.

```bash
scalingo --app my-app env-set PGRST_JWT_SECRET='super-secret-jwt-key-at-least-32chars'
```

---

## Customizing PostgREST Version

By default, the buildpack installs PostgREST **v14.9**. To use a different version:

```bash
scalingo --app my-app env-set POSTGREST_VERSION='custom version'
```

Check [PostgREST releases](https://github.com/PostgREST/postgrest/releases) for available versions.

---

## How the Buildpack Works

1. **Detect** - The buildpack identifies itself as a PostgREST buildpack.
2. **Compile** - Downloads the PostgREST static binary from GitHub releases and places it in `/app/bin/`.
3. **Release** - Configures the default web process to run `/app/bin/postgrest`.

---

## End-to-End Example

This walkthrough assumes your app is deployed and running at `https://my-postgrest-api.osc-fr1.scalingo.io`. Replace all occurrences of `my_app_4242` with your actual Scalingo PostgreSQL username.

### Step 1 — Create a table with Row Level Security

Open a console on your Scalingo database:

```bash
scalingo --app my-postgrest-api pgsql-console
```

Create a `todos` table where each row belongs to a user, and enable RLS so that each user only sees their own data:

```sql
CREATE TABLE todos (
  id      SERIAL PRIMARY KEY,
  user_id TEXT    NOT NULL,
  task    TEXT    NOT NULL,
  done    BOOLEAN DEFAULT false
);

-- Enable Row Level Security
ALTER TABLE todos ENABLE ROW LEVEL SECURITY;

-- Each user can only read their own rows
-- PostgREST exposes the JWT claims via request.jwt.claims
CREATE POLICY todos_isolation ON todos
  USING (user_id = current_setting('request.jwt.claims', true)::json->>'sub');

-- Insert data for two different users
INSERT INTO todos (user_id, task, done) VALUES
  ('alice', 'Buy groceries',    false),
  ('alice', 'Read the docs',    false),
  ('bob',   'Deploy PostgREST', true),
  ('bob',   'Write tests',      false);
```

### Step 2 — Anonymous request (no JWT)

Without a token, `request.jwt.claims` is empty — the RLS policy filters out all rows:

```bash
curl https://my-postgrest-api.osc-fr1.scalingo.io/todos
```

```json
[]
```

### Step 3 — Authenticated requests (with JWT)

In production, the JWT is issued by your auth provider ([Keycloak](https://scalingo.com/blog/guide-to-deploy-keycloak-on-scalingo), etc.) after the user logs in. The `sub` claim identifies the user.

For testing, generate a token locally (requires `pyjwt`):

```bash
pip install pyjwt

python3 - <<'EOF'
import jwt

# Token for alice
print(jwt.encode(
    {"role": "my_app_4242", "sub": "alice", "exp": 9999999999},
    "super-secret-jwt-key-at-least-32chars",
    algorithm="HS256"
))

# Token for bob
print(jwt.encode(
    {"role": "my_app_4242", "sub": "bob", "exp": 9999999999},
    "super-secret-jwt-key-at-least-32chars",
    algorithm="HS256"
))
EOF
```

> Alternatively, use [jwt.io](https://jwt.io): algorithm `HS256`, secret `super-secret-jwt-key-at-least-32chars`, and adjust the `sub` field in the payload.

**Request as alice — sees only her todos:**

```bash
TOKEN_ALICE="<token generated for alice>"

curl https://my-postgrest-api.osc-fr1.scalingo.io/todos \
  -H "Authorization: Bearer $TOKEN_ALICE"
```

```json
[
  { "id": 1, "user_id": "alice", "task": "Buy groceries", "done": false },
  { "id": 2, "user_id": "alice", "task": "Read the docs", "done": false }
]
```

**Request as bob — sees only his todos:**

```bash
TOKEN_BOB="<token generated for bob>"

curl https://my-postgrest-api.osc-fr1.scalingo.io/todos \
  -H "Authorization: Bearer $TOKEN_BOB"
```

```json
[
  { "id": 3, "user_id": "bob", "task": "Deploy PostgREST", "done": true  },
  { "id": 4, "user_id": "bob", "task": "Write tests",      "done": false }
]
```

For all available query operators (filtering, ordering, pagination, inserting, updating...), refer to the official [PostgREST documentation](https://postgrest.org/en/stable/references/api.html).

---

## Links

- [PostgREST Documentation](https://postgrest.org/)
- [PostgREST GitHub](https://github.com/PostgREST/postgrest)
- [Scalingo Documentation](https://doc.scalingo.com/)
- [Scalingo Custom Buildpacks](https://doc.scalingo.com/platform/deployment/buildpacks/custom)
