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
# postgres://my_app_4242:password@host:port/my_app_4242
#             ^^^^^^^^^^^^ this is your PostgreSQL username
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

Used to verify JWT tokens issued by an external authentication provider ([Keycloak](https://scalingo.com/blog/guide-to-deploy-keycloak-on-scalingo),...).

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

## Quick Example

Connect to your database via the Scalingo console (replace `my_app_4242` with your actual user):

```bash
scalingo --app my-postgrest-api pgsql-console
```

```sql
CREATE TABLE todos (
  id    SERIAL PRIMARY KEY,
  task  TEXT NOT NULL,
  done  BOOLEAN DEFAULT false
);

INSERT INTO todos (task, done) VALUES
  ('Buy groceries', false),
  ('Deploy PostgREST', true);
```

> The Scalingo user already has full access to the schema - no `GRANT` needed.

**Get all todos:**

```bash
curl https://my-postgrest-api.osc-fr1.scalingo.io/todos
```

```json
[
  { "id": 1, "task": "Buy groceries", "done": false },
  { "id": 2, "task": "Deploy PostgREST", "done": true }
]
```

**Filter - only incomplete todos:**

```bash
curl 'https://my-postgrest-api.osc-fr1.scalingo.io/todos?done=eq.false'
```

```json
[
  { "id": 1, "task": "Buy groceries", "done": false }
]
```

### Authenticated Request with JWT

Once `PGRST_JWT_SECRET` is set, PostgREST accepts requests authenticated with a JWT. The token is obtained from your auth provider (e.g., Keycloak) after a user logs in.

A typical JWT payload for PostgREST looks like:

```json
{
  "role": "my_app_4242",
  "sub":  "alice",
  "exp":  1999999999
}
```

Pass it in the `Authorization` header:

```bash
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.\
eyJyb2xlIjoibXlfYXBwXzQyNDIiLCJzdWIiOiJhbGljZSIsImV4cCI6MTk5OTk5OTk5OX0.\
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"

curl https://my-postgrest-api.osc-fr1.scalingo.io/todos \
  -H "Authorization: Bearer $TOKEN"
```

```json
[
  { "id": 1, "task": "Buy groceries", "done": false },
  { "id": 2, "task": "Deploy PostgREST", "done": true }
]
```

> The token above is illustrative. In practice, your auth provider (Keycloak, Auth0, etc.) generates and signs it using the same secret as `PGRST_JWT_SECRET`.

For all available query operators (ordering, pagination, insertion, updates, and more), refer to the official [PostgREST documentation](https://postgrest.org/en/stable/references/api.html).

---

## Links

- [PostgREST Documentation](https://postgrest.org/)
- [PostgREST GitHub](https://github.com/PostgREST/postgrest)
- [Scalingo Documentation](https://doc.scalingo.com/)
- [Scalingo Custom Buildpacks](https://doc.scalingo.com/platform/deployment/buildpacks/custom)
