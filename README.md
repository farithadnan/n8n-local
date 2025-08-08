# n8n Docker Compose Setup

This setup allows you to run **n8n** in two modes using Docker Compose:

1. **Normal Mode** – Local access only (e.g., http://localhost:5678 or LAN access)
2. **Tunnel Mode** – Exposes n8n via a secure HTTPS tunnel (required for certain integrations like Telegram)


Workflows, credentials, and settings are stored in a shared Docker volume so you can switch modes without losing data.

---
## How to Use It

### Start normal mode

```powershell
docker compose up -d n8n
```

### Start tunnel mode

```powershell
docker compose up -d n8n_tunnel
```

If you want to see the logs (like the interactive `-it` flags in your Docker command), run:

```powershell
docker compose logs -f n8n_tunnel
```

#### Important: Tunnel Timeout Considerations

When using tunnel mode, you may encounter a **504 Gateway Time-out** error if the container runs idle for extended periods. This happens because:

- n8n uses a built-in tunnel service (n8n.cloud) designed for temporary HTTPS exposure, not always-on operation
- If there's no activity (workflows, webhooks, etc.) for some time, the tunnel may become inactive
- When the tunnel times out, external services cannot reach your local container, resulting in 504 errors

**Recommendations:**

- Use tunnel mode primarily for development, testing, or temporary webhook integrations
- For production or always-on scenarios, consider using a reverse proxy or dedicated hosting
- If you encounter 504 errors, restart the tunnel service: `docker compose restart n8n_tunnel`

### Switch modes

Stop the running container, then start the other mode:

```powershell
docker compose down
docker compose up -d n8n_tunnel
```

### Updating n8n

This will fetch the latest image and restart with your existing data.

```powershell
docker compose pull
docker compose up -d
```

---

## How it works

The docker-compose.yml file defines two services:

- `n8n` → Runs n8n normally, bound to your local network.
- `n8n_tunnel` → Runs n8n with `--tunnel` enabled to provide a secure HTTPS public URL via n8n.cloud.

Both services share the same volume:

```yaml
volumes:
  - n8n_data:/home/node/.n8n
```

This ensures your workflows and credentials persist across restarts and when switching modes.

### Normal mode

```yaml
environment:
  - N8N_RUNNERS_ENABLED=true
```

- `N8N_RUNNERS_ENABLED=true` – Enables task runners to avoid future deprecation issues.

This mode is great if you only want to use n8n on the same machine or local network.

### Tunnel Mode

```yaml
environment:
  - N8N_SECURE_COOKIE=false
  - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
  - N8N_TUNNEL_SUBDOMAIN=farithadnan
  - N8N_HOST=farithadnan.n8n.cloud
  - N8N_RUNNERS_ENABLED=true
  - N8N_EXPRESS_TRUST_PROXY=true
  - N8N_EDITOR_BASE_URL=https://farithadnan.n8n.cloud
command: [start --tunnel]
```

#### Tunnel & Networking Config
- `N8N_TUNNEL_SUBDOMAIN=yourname` – Sets your tunnel URL to `https://yourname.n8n.cloud`
- `N8N_HOST=yourname.n8n.cloud` – Public hostname n8n binds to (needed for webhook/tunnel routing)
- `N8N_EDITOR_BASE_URL=https://yourname.n8n.cloud` – Public-facing URL for your n8n editor (used for webhooks, OAuth2 flows, etc.)
- `start --tunnel` – Runs n8n in tunnel mode, automatically creating a secure HTTPS endpoint

#### Security & Stability

- `N8N_SECURE_COOKIE=false` – Allows cookies to work over HTTP (development/testing only)

- `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true` – Enforces correct file permissions

- `N8N_RUNNERS_ENABLED=true` – Enables task runners (future-proof)

- `N8N_EXPRESS_TRUST_PROXY=true` – Allows Express to trust proxy headers (needed in tunnel mode)

This mode is required when services like **Telegram** need an **HTTPS** webhook URL.

### Environment Variable

Variables for both services are stored in a .`.env` file but you can refer `.env.sample` file for guidance:

```env
# Shared
N8N_PORT=5678
N8N_RUNNERS_ENABLED=true

# Tunnel-specific
N8N_SECURE_COOKIE=false
N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
N8N_TUNNEL_SUBDOMAIN=yourname
N8N_HOST=yourname.n8n.cloud
N8N_EXPRESS_TRUST_PROXY=true
N8N_EDITOR_BASE_URL=https://yourname.n8n.cloud
```