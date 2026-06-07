# Auto setup migration times out on Render with external PostgreSQL

## Summary

When deploying Sub2API to Render using the official Docker image and environment-variable based auto setup, the service can fail during the initial database migration phase if PostgreSQL is hosted externally, such as on Aiven.

The database and Redis connectivity checks pass successfully, but `AUTO_SETUP=true` fails while applying migrations because the setup migration context appears to be limited to 60 seconds.

## Environment

- Deployment platform: Render Web Service
- Image: `ghcr.io/wei-shaw/sub2api:latest`
- Setup mode: `AUTO_SETUP=true`
- PostgreSQL: external managed PostgreSQL with TLS, e.g. Aiven
- Redis: Render Redis / Key Value
- Persistent disk: not attached

## Relevant environment variables

Sensitive values redacted:

```env
AUTO_SETUP=true
SERVER_HOST=0.0.0.0
SERVER_PORT=8080
SERVER_MODE=release

DATABASE_HOST=<external-postgres-host>
DATABASE_PORT=<external-postgres-port>
DATABASE_USER=<postgres-user>
DATABASE_PASSWORD=<redacted>
DATABASE_DBNAME=<database-name>
DATABASE_SSLMODE=require

REDIS_HOST=<redis-host>
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
REDIS_ENABLE_TLS=false

ADMIN_EMAIL=<redacted>
ADMIN_PASSWORD=<redacted>
JWT_SECRET=<redacted>
TOTP_ENCRYPTION_KEY=<redacted>
```

## Expected behavior

With `AUTO_SETUP=true`, Sub2API should:

1. Read database, Redis, and admin settings from environment variables.
2. Complete initial migrations.
3. Create the initial admin account.
4. Start the web server and show the login page.

## Actual behavior

Database and Redis connection tests pass, then setup fails during migration:

```text
Auto setup mode enabled...
Auto setup enabled, configuring from environment variables...
Data directory: /app/data
Testing database connection...
Database connection successful
Testing Redis connection...
Redis connection successful
Initializing database...
Auto setup failed: database initialization failed: check migration 091_add_group_messages_dispatch_model_config.sql: context deadline exceeded
```

Render also reports no open port, but this is a secondary symptom because the server has not started listening yet:

```text
No open ports detected, continuing to scan...
```

## Notes

This seems related to the install-time migration timeout in `backend/internal/setup/setup.go`:

```go
migrationCtx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
```

Normal startup migrations appear to use a much longer timeout, but the first-run auto setup path uses 60 seconds. With external PostgreSQL over TLS, running many migrations can exceed 60 seconds even though the database is reachable and functioning.

The frontend setup wizard also has a 30-second setup API timeout, which can surface as:

```text
timeout of 30000ms exceeded
```

## Suggested fix

Make the setup migration timeout configurable and/or increase the default timeout to match normal startup migration behavior.

For example:

```go
const defaultMigrationTimeoutSec = 600

func getMigrationTimeout() time.Duration {
    seconds := getEnvIntOrDefault("SETUP_MIGRATION_TIMEOUT_SECONDS", defaultMigrationTimeoutSec)
    if seconds <= 0 {
        seconds = defaultMigrationTimeoutSec
    }
    return time.Duration(seconds) * time.Second
}
```

Then use it in setup migration initialization:

```go
migrationCtx, cancel := context.WithTimeout(context.Background(), getMigrationTimeout())
```

Also consider increasing the frontend `/setup/install` request timeout, or making it configurable, because the browser request can time out before the backend finishes first-run migrations.

## Workaround

Building a custom image with a longer setup migration timeout, or using a deployment where PostgreSQL is in the same Docker network, avoids this issue.
