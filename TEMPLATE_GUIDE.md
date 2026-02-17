# Creating Custom Coolify Service Templates

This guide explains how to create Docker Compose templates compatible with [Coolify](https://coolify.io) for this repository. It is intended for both human contributors and AI agents.

> **Sources:** This guide is based on the official [Coolify Service Contribution Docs](https://github.com/coollabsio/coolify-docs/blob/next/docs/get-started/contribute/service.md) and the [Coolify Docker Compose Docs](https://github.com/coollabsio/coolify-docs/blob/next/docs/knowledge-base/docker/compose.md).

---

## Table of Contents

1. [Overview](#overview)
2. [Template File Structure](#template-file-structure)
3. [Header Metadata](#header-metadata)
4. [Services Section](#services-section)
5. [Environment Variables](#environment-variables)
6. [Magic Environment Variables](#magic-environment-variables)
7. [Storage and Volumes](#storage-and-volumes)
8. [Health Checks](#health-checks)
9. [Networking](#networking)
10. [Testing Your Template](#testing-your-template)
11. [Submitting Your Template](#submitting-your-template)
12. [Complete Example](#complete-example)
13. [Reference: Existing Templates](#reference-existing-templates)

---

## Overview

Coolify service templates are standard Docker Compose files enhanced with Coolify-specific features. The Docker Compose file is **the single source of truth** — configuration that would normally be done through the Coolify UI must be defined directly in the compose file.

Templates in this repository live under `templates/compose/` and follow a consistent naming convention:

- `<service-name>.yaml` — primary template
- `<service-name>-<variant>.yaml` — variants (e.g., `pocketid-pg.yaml`, `pocketid-sqlite.yaml`)

---

## Template File Structure

Every template follows this structure:

```yaml
# documentation: <url>
# slogan: <one-line description>
# tags: <comma-separated tags>
# port: <primary service port>
# author: <github handle>
# repository: <repo url>

services:
  <service-name>:
    image: <image>:<tag>
    environment:
      - SERVICE_FQDN_<NAME>_<PORT>
      - VAR=${VAR:-default}
    volumes:
      - <volume>:<path>
    depends_on:
      <dependency>:
        condition: service_healthy
    healthcheck:
      test: [...]
      interval: <duration>
      timeout: <duration>
      retries: <number>

  # Supporting services (databases, caches, etc.)
  <db-service>:
    image: <image>:<tag>
    ...

volumes:
  <volume-name>:
```

---

## Header Metadata

Every template **must** begin with comment-based metadata:

```yaml
# documentation: https://docs.example.com/
# slogan: A brief description of your service.
# tags: tag1,tag2,tag3
# port: 1234
# author: @yourgithubhandle
# repository: https://github.com/oregapam/awesome-coolify-service-templates
```

| Field           | Required | Description                                         |
|-----------------|----------|-----------------------------------------------------|
| `documentation` | Yes      | URL to the service's official documentation         |
| `slogan`        | Yes      | Short one-line description of the service           |
| `tags`          | Yes      | Comma-separated keywords for search/categorization  |
| `port`          | Yes      | The main port users access the service on           |
| `author`        | Yes      | GitHub handle of the template author                |
| `repository`    | Yes      | URL to this repository                              |

**Important:** Always specify a port. Coolify's Caddy proxy cannot automatically determine the service's port.

---

## Services Section

Define all services under the `services:` key. A template typically includes:

1. **Primary service** — the main application
2. **Supporting services** — databases (PostgreSQL, Redis), proxies, workers, etc.

### Primary Service Pattern

```yaml
services:
  myapp:
    image: vendor/myapp:latest
    environment:
      - SERVICE_FQDN_MYAPP_3000
      - DATABASE_URL=postgres://$SERVICE_USER_POSTGRES:$SERVICE_PASSWORD_POSTGRES@postgres:5432/mydb
    volumes:
      - myapp-data:/app/data
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### Database Service Patterns

**PostgreSQL:**

```yaml
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: $SERVICE_USER_POSTGRES
      POSTGRES_PASSWORD: $SERVICE_PASSWORD_POSTGRES
      POSTGRES_DB: mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "$SERVICE_USER_POSTGRES", "-d", "mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
```

**Redis:**

```yaml
  redis:
    image: redis:6-alpine
    environment:
      REDIS_PASSWORD: $SERVICE_PASSWORD_REDIS
    volumes:
      - redis-data:/data
    command: redis-server --requirepass "$SERVICE_PASSWORD_REDIS"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "$SERVICE_PASSWORD_REDIS", "ping"]
      interval: 5s
      timeout: 10s
      retries: 20
```

---

## Environment Variables

Coolify auto-detects environment variables in compose files and displays them in the UI.

### Syntax

| Syntax                        | Behavior                                              |
|-------------------------------|-------------------------------------------------------|
| `HARDCODED_VALUE=hello`       | Not visible in UI, hardcoded                          |
| `${VAR}`                      | Editable in UI, initially empty                       |
| `${VAR:-default}`             | Editable in UI, prefilled with `default`              |
| `${VAR:?}`                    | **Required** — deployment fails if not set            |
| `${VAR:?default}`             | Required with default value shown in UI               |

### Best Practices

- Use `${VAR:?}` for settings users **must** configure (API keys, domains, SMTP)
- Use `${VAR:-default}` for settings with sensible defaults
- Use hardcoded values for internal settings that should not change
- Group related variables with comments

```yaml
environment:
  # Required - user must provide
  - API_KEY=${API_KEY:?}
  - SMTP_HOST=${SMTP_HOST:?}

  # Optional with defaults
  - LOG_LEVEL=${LOG_LEVEL:-info}
  - CACHE_TTL=${CACHE_TTL:-3600}

  # Internal - not exposed in UI
  - INTERNAL_PORT=3000
```

---

## Magic Environment Variables

Coolify generates special dynamic variables using the naming pattern `SERVICE_<TYPE>_<IDENTIFIER>`. These are the core mechanism for automatic secret generation and service URL routing.

### Types

| Pattern                              | Purpose                                         | Example Value                            |
|--------------------------------------|-------------------------------------------------|------------------------------------------|
| `SERVICE_FQDN_<NAME>_<PORT>`        | Domain + port routing                           | Sets up domain proxy to container port   |
| `SERVICE_URL_<NAME>`                 | Full URL of the service                         | `http://myapp-abc123.example.com`        |
| `SERVICE_URL_<NAME>_<PORT>`         | Full URL with specific port                     | `http://myapp-abc123.example.com:3000`   |
| `SERVICE_USER_<NAME>`               | Random 16-char username                         | `xKq8mP2nL5vR9wYt`                      |
| `SERVICE_PASSWORD_<NAME>`           | Random password                                 | `aB3$kL9mN2pQ5rSt`                      |
| `SERVICE_PASSWORD_64_<NAME>`        | Random 64-char password                         | (64 characters)                          |
| `SERVICE_BASE64_<NAME>`             | Random Base64-encoded 32-char string            | `dGhpcyBpcyBhIHRlc3Q=`                  |
| `SERVICE_BASE64_64_<NAME>`          | Random Base64-encoded 64-char string            | (longer base64)                          |
| `SERVICE_BASE64_128_<NAME>`         | Random Base64-encoded 128-char string           | (even longer base64)                     |

### Usage Rules

1. **FQDN variables** are placed standalone in the environment list to register the service with Coolify's proxy:

   ```yaml
   environment:
     - SERVICE_FQDN_MYAPP_3000
   ```

2. **Password/User variables** are referenced with `$` prefix in other variables:

   ```yaml
   environment:
     POSTGRES_USER: $SERVICE_USER_POSTGRES
     POSTGRES_PASSWORD: $SERVICE_PASSWORD_POSTGRES
   ```

3. **Reusing generated values** across services — reference the same variable name:

   ```yaml
   services:
     app:
       environment:
         - DB_PASSWORD=$SERVICE_PASSWORD_POSTGRES
     postgres:
       environment:
         - POSTGRES_PASSWORD=$SERVICE_PASSWORD_POSTGRES
   ```

4. **Port identifiers** use hyphens (not underscores) when service names include ports:

   ```yaml
   # Correct
   - SERVICE_URL_APPWRITE-SERVICE_3000
   # Wrong
   - SERVICE_URL_APPWRITE_SERVICE_3000
   ```

---

## Storage and Volumes

### Named Volumes (Recommended)

Use named volumes for persistent data. Coolify manages these automatically:

```yaml
services:
  myapp:
    volumes:
      - app-data:/app/data

volumes:
  app-data:
```

### Creating Directories

Use Coolify's `is_directory` flag to create empty directories:

```yaml
volumes:
  - type: bind
    source: ./srv
    target: /srv
    is_directory: true
```

### Creating Files with Content

Use the `content` key to create configuration files inline:

```yaml
volumes:
  - type: bind
    source: ./config/app.conf
    target: /etc/app/app.conf
    read_only: true
    content: |
      [server]
      port = 3000
      host = 0.0.0.0
```

This is useful for injecting configuration files (nginx configs, squid proxy configs, init scripts, etc.) without requiring separate files in the repository.

### Docker Compose Configs (Alternative)

```yaml
services:
  myapp:
    configs:
      - source: init-script
        target: /docker-entrypoint-initdb.d/init.sql

configs:
  init-script:
    content: |
      CREATE TABLE IF NOT EXISTS users (id SERIAL PRIMARY KEY);
```

---

## Health Checks

Every service **should** include a health check. Health checks enable `depends_on` with `condition: service_healthy` and allow Coolify to monitor service status.

### Common Patterns

**HTTP endpoint:**

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

**PostgreSQL:**

```yaml
healthcheck:
  test: ["CMD", "pg_isready", "-U", "$SERVICE_USER_POSTGRES", "-d", "mydb"]
  interval: 10s
  timeout: 5s
  retries: 5
```

**Redis:**

```yaml
healthcheck:
  test: ["CMD", "redis-cli", "-a", "$SERVICE_PASSWORD_REDIS", "ping"]
  interval: 5s
  timeout: 10s
  retries: 20
```

**TCP port check:**

```yaml
healthcheck:
  test: ["CMD-SHELL", "bash -c ':> /dev/tcp/127.0.0.1/8080' || exit 1"]
  interval: 5s
  timeout: 5s
  retries: 10
```

**wget (when curl is not available):**

```yaml
healthcheck:
  test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000"]
  interval: 30s
  timeout: 10s
  retries: 3
```

### Excluding from Health Checks

For services that run once and exit (migrations, init containers):

```yaml
services:
  migration:
    exclude_from_hc: true
```

---

## Networking

### Default Behavior

- All services in a compose stack share a private network by default
- Services reference each other using service names as hostnames (e.g., `http://postgres:5432`)
- Services without a domain or port mapping remain private

### Domain Routing

The `SERVICE_FQDN_<NAME>_<PORT>` variable tells Coolify to route external traffic to the specified container port:

```yaml
environment:
  - SERVICE_FQDN_MYAPP_3000   # Routes domain traffic to container port 3000
```

### Direct Port Mapping

Use `ports` to expose a service directly on the host (bypasses the proxy):

```yaml
ports:
  - "3000:3000"        # Exposed on all interfaces
  - "127.0.0.1:3000:3000"  # Localhost only
```

**Warning:** Direct port mapping bypasses Coolify's proxy configuration and may unintentionally expose private services.

### Internal Networks

For security-sensitive services (e.g., SSRF proxies, sandboxes), create isolated networks:

```yaml
services:
  sandbox:
    networks:
      - internal_network
      - default

networks:
  internal_network:
    driver: bridge
    internal: true   # No external access
```

### Cross-Stack Communication

To connect services across different Coolify stacks:

1. Enable "Connect to Predefined Network" in Coolify service settings
2. Services are renamed to `<name>-<uuid>` to prevent collisions
3. Reference other services using their renamed identifier

---

## Testing Your Template

1. Open your **Coolify Dashboard**
2. Go to **Projects** and select or create a project
3. Click **+ Add New Resource**
4. Choose **Docker Compose Empty**
5. Paste your template YAML content
6. Click **Save**
7. Configure any required environment variables in the UI
8. Click **Deploy**

Verify that:

- All services start successfully
- Health checks pass
- The main service is accessible via its domain
- Environment variables appear correctly in the UI
- Required variables prevent deployment when empty

---

## Submitting Your Template

### To This Repository

1. Create your `<service-name>.yaml` file under `templates/compose/`
2. Follow the naming conventions:
   - Lowercase, hyphens as separators
   - Use suffixes for variants: `-pg`, `-sqlite`, `-basic`, `-production`
3. Open a Pull Request with:
   - The new template file
   - An update to `README.md` adding the service to the list

### To Coolify Upstream (Optional)

If you want to submit directly to Coolify:

1. Add your template to `/templates/compose` in the [Coolify repository](https://github.com/coollabsio/coolify)
2. Include an SVG logo in the `svgs` folder (filename must match the service name)
3. Submit a PR to the main repository
4. After merge, add documentation to the [coolify-docs](https://github.com/coollabsio/coolify-docs) repository

---

## Complete Example

Here is a complete template for a hypothetical service with PostgreSQL and Redis:

```yaml
# documentation: https://docs.example.com/
# slogan: An example service with PostgreSQL and Redis
# tags: example,web,api,postgres,redis
# port: 3000
# author: @yourgithubhandle
# repository: https://github.com/oregapam/awesome-coolify-service-templates

services:
  myapp:
    image: vendor/myapp:latest
    environment:
      # Coolify FQDN routing
      - SERVICE_FQDN_MYAPP_3000
      # Database
      - DATABASE_URL=postgres://$SERVICE_USER_POSTGRES:$SERVICE_PASSWORD_POSTGRES@postgres:5432/myapp
      # Redis
      - REDIS_URL=redis://:$SERVICE_PASSWORD_REDIS@redis:6379/0
      # Required settings
      - SECRET_KEY=$SERVICE_BASE64_64_SECRET
      - ADMIN_EMAIL=${ADMIN_EMAIL:?}
      # Optional settings
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - MAX_UPLOAD_SIZE=${MAX_UPLOAD_SIZE:-10M}
    volumes:
      - myapp-data:/app/data
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: $SERVICE_USER_POSTGRES
      POSTGRES_PASSWORD: $SERVICE_PASSWORD_POSTGRES
      POSTGRES_DB: myapp
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "$SERVICE_USER_POSTGRES", "-d", "myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:6-alpine
    environment:
      REDIS_PASSWORD: $SERVICE_PASSWORD_REDIS
    volumes:
      - redis-data:/data
    command: redis-server --requirepass "$SERVICE_PASSWORD_REDIS"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "$SERVICE_PASSWORD_REDIS", "ping"]
      interval: 5s
      timeout: 10s
      retries: 20

volumes:
  myapp-data:
  postgres-data:
  redis-data:
```

---

## Reference: Existing Templates

Study these existing templates for patterns and best practices:

| Template | Complexity | Notable Patterns |
|----------|-----------|-----------------|
| `chartdb.yaml` | Simple | Single service, no dependencies |
| `corsproxy.yaml` | Simple | Single service, minimal config |
| `pocketid-sqlite.yaml` | Simple | SQLite-based, minimal setup |
| `vaultwarden-pg.yaml` | Medium | App + PostgreSQL |
| `planka.yaml` | Medium | App + PostgreSQL, SMTP, S3 config |
| `discourse.yaml` | Complex | App + worker + PostgreSQL + Redis, SMTP |
| `lightrag-production.yaml` | Complex | App + Redis + Qdrant + Memgraph, YAML anchors |
| `dify.yaml` | Complex | Multi-service AI platform with SSRF proxy, sandbox, nginx |

---

## AI Agent Instructions

When creating a new Coolify service template, follow this checklist:

1. **Research the service:** Read the official documentation to understand required services, ports, environment variables, and dependencies.

2. **Determine the architecture:** Identify what database(s), cache(s), and auxiliary services are needed.

3. **Create the header metadata:** Include `documentation`, `slogan`, `tags`, `port`, `author`, and `repository`.

4. **Define services:**
   - Use official Docker images with specific version tags (not `latest` for databases)
   - Set up the FQDN variable on the primary service: `SERVICE_FQDN_<NAME>_<PORT>`
   - Use `SERVICE_USER_*` and `SERVICE_PASSWORD_*` for auto-generated credentials
   - Use `${VAR:?}` for required user inputs and `${VAR:-default}` for optional ones
   - Add health checks to every service
   - Use `depends_on` with `condition: service_healthy` for startup ordering

5. **Define volumes:** Use named volumes for all persistent data.

6. **Test the template:** Deploy via Coolify's "Docker Compose Empty" option.

7. **Update the README:** Add the new service to the list in `README.md`.
