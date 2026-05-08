# Centralized Traefik + Portainer — Domain Control Guide

## Overview

| Service | URL | Auth |
|---|---|---|
| Traefik dashboard | `https://domain-control.easydev.in/dashboard/` | Basic auth (admin) |
| Portainer | `https://portainer.easydev.in` | Portainer login |

**Server:** `155.248.244.237` (Oracle Linux ARM64)  
**Traefik version:** v2.11  
**Shared Docker network:** `edge`

---

## Quick Start — How to Use This Repo

### Step 1 — Clone and configure locally

```bash
git clone https://github.com/gostgmaer/infra.git
cd infra
cp .env.example .env
```

Edit `.env` with your real values (domain, email, password hash).

### Step 2 — Generate a dashboard password

```bash
docker run --rm httpd:2 htpasswd -nb admin yourpassword | sed 's/\$/\$\$/g'
```

Paste the output as `TRAEFIK_DASHBOARD_USERS` in `.env`.

### Step 3 — Add GitHub Secrets

Go to **GitHub → repo → Settings → Secrets and variables → Actions** and add:

| Secret | Value |
|---|---|
| `SSH_HOST` | `155.248.244.237` |
| `SSH_USER` | `opc` |
| `SSH_PORT` | `22` |
| `SSH_PRIVATE_KEY` | Contents of your `~/.ssh/id_rsa` |
| `GIT_REPO_URL` | `https://github.com/gostgmaer/infra.git` |
| `SERVER_ENV_FILE_B64` | Base64 of your `.env` (see below) |

**Generate `SERVER_ENV_FILE_B64` on Windows (PowerShell):**
```powershell
[Convert]::ToBase64String([System.IO.File]::ReadAllBytes("$(Get-Location)\.env"))
```

**On Linux/Mac:**
```bash
base64 -w 0 .env
```

### Step 4 — Deploy

Push to `main` — GitHub Actions will SSH into the server, clone/pull the repo, create `.env`, and run `docker compose up -d` automatically.

Or trigger manually: **GitHub → Actions → Deploy → Run workflow**

### Step 5 — Add a new app domain

In your app's `docker-compose.yml`, attach it to the shared network and add labels:

```yaml
services:
  myapp:
    image: myimage:latest
    networks:
      - edge
    labels:
      - traefik.enable=true
      - traefik.http.routers.myapp.rule=Host(`myapp.easydev.in`)
      - traefik.http.routers.myapp.entrypoints=websecure
      - traefik.http.routers.myapp.tls=true
      - traefik.http.routers.myapp.tls.certresolver=le
      - traefik.http.routers.myapp.middlewares=security-headers@file
      - traefik.http.services.myapp.loadbalancer.server.port=3000

networks:
  edge:
    external: true
    name: edge
```

Make sure a DNS `A` record for `myapp.easydev.in` points to `155.248.244.237`, then `docker compose up -d`. TLS is auto-issued. No changes needed here.

---

## 1. Initial Setup

### 1.1 Configure `.env`

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env
```

| Variable | Description | Example |
|---|---|---|
| `BASE_DOMAIN` | Root domain | `easydev.in` |
| `PORTAINER_SUBDOMAIN` | Subdomain for Portainer | `portainer` |
| `TRAEFIK_DASHBOARD_SUBDOMAIN` | Subdomain for Traefik dashboard | `domain-control` |
| `LETSENCRYPT_EMAIL` | Email for TLS cert notifications | `infra@easydev.in` |
| `TRAEFIK_NETWORK_NAME` | Docker network shared with all apps | `edge` |
| `TRAEFIK_DASHBOARD_USERS` | Basic auth credentials (htpasswd format) | see below |

### 1.2 Generate Dashboard Password

```bash
docker run --rm httpd:2 htpasswd -nb admin yourpassword | sed 's/\$/\$\$/g'
```

Copy the output into `TRAEFIK_DASHBOARD_USERS` in `.env`. The `$$` escaping is required for Docker Compose.

### 1.3 DNS Records

In Cloudflare (or your DNS provider), create `A` records pointing to `155.248.244.237`:

| Name | Type | Value |
|---|---|---|
| `portainer` | A | `155.248.244.237` |
| `domain-control` | A | `155.248.244.237` |
| `*` (wildcard) | A | `155.248.244.237` |

> The wildcard `*` record covers all future subdomains automatically — no new DNS record needed per app.

### 1.4 Firewall / Ports

Ensure these ports are open on the server:

| Port | Protocol | Purpose |
|---|---|---|
| 80 | TCP | Let's Encrypt HTTP challenge + HTTP→HTTPS redirect |
| 443 | TCP | HTTPS traffic |

### 1.5 Start the Stack

```bash
docker compose --env-file .env up -d
```

---

## 2. Adding a New Domain / Service

There are two ways depending on where your app lives.

---

### Option A — App inside this `docker-compose.yml`

Add a new service block with Traefik labels. Replace `myapp`, `3000`, and the subdomain with your values:

```yaml
  myapp:
    image: myimage:latest
    container_name: infra-myapp
    restart: unless-stopped
    networks:
      - edge
    labels:
      - traefik.enable=true
      - traefik.http.routers.myapp.rule=Host(`myapp.easydev.in`)
      - traefik.http.routers.myapp.entrypoints=websecure
      - traefik.http.routers.myapp.tls=true
      - traefik.http.routers.myapp.tls.certresolver=le
      - traefik.http.routers.myapp.middlewares=security-headers@file
      - traefik.http.services.myapp.loadbalancer.server.port=3000
```

Then apply:

```bash
docker compose --env-file .env up -d
```

---

### Option B — App in a separate project (recommended)

This is the preferred approach for other repos/teams. The app joins the shared `edge` network externally — no changes needed in this infra repo.

In the **other app's** `docker-compose.yml`:

```yaml
services:
  myapp:
    image: myimage:latest
    restart: unless-stopped
    networks:
      - edge
    labels:
      - traefik.enable=true
      - traefik.http.routers.myapp.rule=Host(`myapp.easydev.in`)
      - traefik.http.routers.myapp.entrypoints=websecure
      - traefik.http.routers.myapp.tls=true
      - traefik.http.routers.myapp.tls.certresolver=le
      - traefik.http.routers.myapp.middlewares=security-headers@file
      - traefik.http.services.myapp.loadbalancer.server.port=3000

networks:
  edge:
    external: true   # joins the existing Traefik network
    name: edge
```

Then deploy that app:

```bash
docker compose up -d
```

Traefik auto-discovers the container and issues a TLS cert. No restart of infra required.

---

### Checklist per new service

- [ ] DNS `A` record created (or wildcard `*` already covers it)
- [ ] `traefik.enable=true` label present
- [ ] Router name is unique (e.g. `myapp` — must not conflict with other routers)
- [ ] Correct internal port in `loadbalancer.server.port`
- [ ] Service connected to `edge` network

---

## 3. Available Middlewares

Defined in `traefik/dynamic/middlewares.yml`. Reference them in labels like:

```
- traefik.http.routers.myapp.middlewares=security-headers@file
```

| Middleware | Provider | Effect |
|---|---|---|
| `security-headers@file` | file | Adds HSTS, X-Frame-Options, CSP headers |
| `gzip@file` | file | Enables gzip compression |
| `dashboard-auth@docker` | docker | Basic auth (Traefik dashboard only) |

You can chain multiple:

```
- traefik.http.routers.myapp.middlewares=security-headers@file,gzip@file
```

---

## 4. Updating Dashboard Password

```bash
# 1. Generate new hash
docker run --rm httpd:2 htpasswd -nb admin newpassword | sed 's/\$/\$\$/g'

# 2. Update on server
sed -i 's|^TRAEFIK_DASHBOARD_USERS=.*|TRAEFIK_DASHBOARD_USERS=<paste hash here>|' ~/infra/.env

# 3. Recreate Traefik to pick up new env
cd ~/infra && docker compose --env-file .env up -d --force-recreate traefik
```

---

## 5. Notes

- **Cloudflare SSL mode** must be set to **Full (strict)** for HTTPS to work end-to-end.
- **HTTP challenge** is used for Let's Encrypt — port 80 must be reachable from the internet during cert issuance.
- Traefik watches the Docker socket for label changes — no restart needed when adding/removing services.
- The `$$` in `TRAEFIK_DASHBOARD_USERS` is required by Docker Compose to represent a literal `$` in the htpasswd hash.
