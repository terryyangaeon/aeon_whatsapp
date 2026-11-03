# AEON WhatsApp — self-hosted WAHA gateway

Thin wrapper around [`devlikeapro/waha`](https://waha.devlike.pro/) that
gives the AEON apps (IoT Smart Toilet alerts, OneView notifications, etc.)
a shared WhatsApp gateway. One repo serves both:

- **Local docker** — run `docker compose up -d` for development.
- **Coolify** — point at this repo as a Docker Compose build pack.

Coolify's Traefik is the only reverse proxy on the host, so the compose
file carries its own Traefik labels (http → https redirect, `letsencrypt`
cert resolver, service port 3000). The container is *not* Coolify-managed
in the Sentinel sense when started standalone — we borrow Traefik without
paying the per-container polling cost.

## Local

```bash
cp .env.example .env
# fill in the real values in .env
docker compose up -d
docker compose logs -f waha
```

- Dashboard: <http://localhost:3000/dashboard>
- Swagger:   <http://localhost:3000/>
- Session is stored in the `sessions` named volume — survives restarts.

The Traefik labels only take effect when the external `coolify` network
exists. Locally that network does not exist, so either remove the
`networks:` block before running locally or create the network manually
(`docker network create coolify`).

## Deploy to a Coolify host

### A. Through Coolify (managed)

1. In Coolify → new resource → **Public Git Repository**
2. URL: `https://github.com/terryyangaeon/aeon_whatsapp`, branch `main`
3. Build Pack: Docker Compose. Coolify auto-detects `docker-compose.yml`
   and the 9 env vars.
4. Fill in the env vars with the real secrets, set the domain to
   `waha.aeondelightasia.online`, set Resource Limits (CPU 1.5, RAM 768M,
   reservation 256M — see **Resource limits** below), **Deploy**.

### B. Standalone via SSH (bypasses Coolify management)

SSH into the host, then:

```bash
cd /opt
git clone https://github.com/terryyangaeon/aeon_whatsapp.git
cd aeon_whatsapp

# Create .env (NOT committed) with the real secrets.
cat > .env <<'EOF'
WHATSAPP_API_KEY=<the shared AEON key>
WAHA_SESSION=default
WHATSAPP_API_HOSTNAME=0.0.0.0
WAHA_DASHBOARD_USERNAME=aeon
WAHA_DASHBOARD_PASSWORD=<password>
WHATSAPP_SWAGGER_USERNAME=aeon
WHATSAPP_SWAGGER_PASSWORD=<password>
WAHA_LOG_FORMAT=JSON
WHATSAPP_DEFAULT_ENGINE=WEBJS
EOF
chmod 600 .env

docker compose up -d
docker compose logs -f waha
```

Traefik auto-provisions the Let's Encrypt cert on first HTTPS request
(takes ~30 s). Verify with:

```bash
curl -I https://waha.aeondelightasia.online/
# expect HTTP/2 401 (dashboard needs auth) with a valid cert
```

Pair the WhatsApp number: open the dashboard, sign in with the AEON
credentials, start session `default`, scan the QR with the AEON phone.

## Update later

```bash
cd /opt/aeon_whatsapp
git pull
docker compose pull
docker compose up -d
```

## Resource limits

Baked into `docker-compose.yml`:

| | Value |
|---|---|
| Memory limit (hard) | 768 MB |
| Memory reservation (soft) | 256 MB |
| CPU limit | 1.5 cores |

If WAHA ever gets OOM-killed, raise `mem_limit` to 1 GB. If Chromium
spawns too many processes, lower `cpus` to 1.

## Notes

- WAHA Core is single-session. One AEON phone number per container.
- Session volume holds a Chromium profile; it does **not** transfer safely
  between hosts. On a move, re-pair by QR.
- The API key, dashboard password and Swagger password live only in
  `.env` on the host (or Coolify's Environment Variables UI) — never in
  git.
- The IoT dashboard's **WhatsApp Alerts → Connection** page can set the
  server URL / session / API key at runtime now (saved in
  `whatsapp_waha_config`), so you don't need to redeploy the dashboard
  to point it at this gateway.
