# AI Agent Memory

This file stores durable learnings so future AI agents can reliably author and update Coolify templates in this repo. Keep entries actionable and stable across sessions.

## Learned User Preferences

- Document all learnings from coolify-enhanced docs and session work in AI agent files (AGENTS.md, .cursor/rules) so future AI work is more reliable.
- This repo targets both vanilla Coolify and coolify-enhanced: use TEMPLATE_GUIDE.md for base Coolify conventions and the Coolify Enhanced section / custom-templates.md for enhanced metadata and database classification.

## Learned Workspace Facts

### Coolify Enhanced (custom-templates)

- **Source:** [coolify-enhanced custom-templates.md](https://github.com/amirhmoradi/coolify-enhanced/blob/main/docs/custom-templates.md). When editing templates, follow both TEMPLATE_GUIDE.md and this doc.
- **Metadata headers** (comment-based, top of YAML): `documentation`, `slogan`, `tags`, `category`, `logo`, `port`, `env_file`, `type`, `ignore`, `minversion`. Category examples: `ai`, `monitoring`, `cms`, `development`. Logo: relative path from repo root (e.g. `svgs/myservice.svg`) or absolute URL; SVG preferred.
- **Magic variables:** `SERVICE_URL_*`, `SERVICE_FQDN_*`, `SERVICE_USER_*`, `SERVICE_PASSWORD_*`, `SERVICE_PASSWORD_64_*`, `SERVICE_BASE64_*`, `SERVICE_BASE64_64_*`, `SERVICE_BASE64_128_*`. For identifiers that include a port, use **hyphens** in the name: `SERVICE_URL_MY-APP_8080` not `SERVICE_URL_MY_APP_8080`.
- **Shared env from Coolify:** `{{environment.SHARED_VARIABLE}}` — use for values provided in Coolify’s shared environment section.
- **Bind mounts:** Use `is_directory: true` for directory creation; use `content: |` in a bind mount to generate config files inline (e.g. nginx, squid, init scripts). No separate repo file needed.
- **Health:** Standard healthchecks on all long-running services. Use `exclude_from_hc: true` for one-off or init containers (migrations, init_permissions).
- **Database classification:** Template-level `# type: database` or per-service `labels: coolify.database: "true"` / `"false"`. Multi-port DBs: `coolify.proxyPorts: "7687:bolt,7444:log-viewer"` (label name is case-sensitive: `coolify.proxyPorts`). Per-service labels override template-level `# type:`.
- **Repo layout:** Templates in `templates/compose/`; filenames `.yaml` or `.yml`; template name = filename without extension. Logos in `svgs/` or `logos/`; relative paths resolved from repo root.

### This repository

- **Templates path:** `templates/compose/<service-name>.yaml` (variants: `<service-name>-<variant>.yaml`, e.g. `pocketid-pg.yaml`).
- **Logos:** Store under `svgs/<service>.svg`. Prefer official project SVG (e.g. Dify: `https://raw.githubusercontent.com/langgenius/dify/main/web/public/logo/logo.svg`).
- **Complex templates:** Use YAML anchors (e.g. `x-shared-env`) for shared environment blocks; use internal networks for security-sensitive services (ssrf_proxy, sandbox); use inline bind mounts with `content:` for nginx/squid configs and large uploads (e.g. `client_max_body_size`).
- **Coolify-generated secrets:** For apps that need multiple long random values, use distinct magic variables (e.g. `SERVICE_PASSWORD_64_SECRETKEY`, `SERVICE_PASSWORD_64_INNERAPIKEY`, `SERVICE_PASSWORD_64_PLUGINDAEMONKEY`, `SERVICE_PASSWORD_64_SANDBOXKEY`) so Coolify generates and injects them.

### Where to look

- **Base Coolify template format:** `TEMPLATE_GUIDE.md` in this repo.
- **Coolify Enhanced format (metadata, DB classification, bind mounts, magic vars):** [custom-templates.md](https://github.com/amirhmoradi/coolify-enhanced/blob/main/docs/custom-templates.md) and the “Coolify Enhanced Conventions” section in TEMPLATE_GUIDE.md.
- **Existing patterns:** See “Reference: Existing Templates” in TEMPLATE_GUIDE.md; for complex multi-service + proxy + sandbox, use `dify.yaml` as reference.
