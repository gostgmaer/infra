# GitHub CI/CD for Infra Stack

This project now includes a GitHub Actions workflow:
- [ .github/workflows/deploy.yml ](.github/workflows/deploy.yml)

It deploys on:
- push to `main`
- manual trigger (`workflow_dispatch`)

## 1) Server Requirements

Install on the target server:
- Docker Engine
- Docker Compose plugin
- git

Oracle Linux ARM quick setup:

```bash
sudo dnf -y update
sudo dnf -y install git
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

Verify ARM64 host:

```bash
uname -m
# expected: aarch64
```

The deployment workflow auto-detects host architecture and sets the correct Docker platform:
- aarch64/arm64 -> linux/arm64
- x86_64/amd64 -> linux/amd64

Create deploy folder and initial env file once:

```bash
mkdir -p /opt/infra
cd /opt/infra
# first-time clone optional; workflow can clone automatically
# git clone <repo-url> .
cp .env.example .env
nano .env
```

## 2) GitHub Repository Secrets

Add these in GitHub:
- `SSH_HOST`: public IP or hostname of your server
- `SSH_USER`: SSH user (for example `ubuntu`)
- `SSH_PRIVATE_KEY`: private key content (PEM) used by Actions
- `SSH_PORT`: SSH port, usually `22`
- `DEPLOY_PATH`: absolute server path (optional, defaults to `/opt/infra`)
- `DEPLOY_BRANCH`: branch to deploy (optional, auto-selects `main` then `master`)
- `GIT_REPO_URL`: clone URL reachable by server (SSH or HTTPS)
- `SERVER_ENV_FILE_B64`: optional base64-encoded full `.env` content (used to auto-create `.env` on first deploy)

Notes:
- If `GIT_REPO_URL` is private via SSH, ensure the server can authenticate to GitHub (deploy key or SSH key).
- Keep production values in server-side `.env`; do not commit them.

## 2.1) How to set env variables on server

Option A: set directly on server (recommended)

```bash
mkdir -p /opt/infra
cat > /opt/infra/.env << 'EOF'
BASE_DOMAIN=easydev.in
PORTAINER_SUBDOMAIN=portainer
TRAEFIK_DASHBOARD_SUBDOMAIN=domain-control
LETSENCRYPT_EMAIL=infra@easydev.in
TRAEFIK_NETWORK_NAME=edge
EOF
chmod 600 /opt/infra/.env
```

Option B: let GitHub Actions create .env automatically

1. Create base64 from your local `.env`:

```bash
base64 -w 0 .env
```

2. Save that output as GitHub Secret `SERVER_ENV_FILE_B64`.

3. On deploy, workflow creates `DEPLOY_PATH/.env` if missing.

For Oracle Linux with older base64 where `-w` is unsupported:

```bash
base64 .env | tr -d '\n'
```

## 3) Recommended Git Auth Pattern

Use SSH deploy key on the server:
1. Generate key on server:
   - `ssh-keygen -t ed25519 -C "infra-deploy"`
2. Add public key as a Deploy Key (read-only) in your GitHub repo.
3. Put SSH clone URL in `GIT_REPO_URL` secret:
   - `git@github.com:<owner>/<repo>.git`

## 4) Deploy Flow

On each push to `main`, workflow does:
1. SSH to server
2. Clone repo if first run
3. Uses branch from `DEPLOY_BRANCH`, or auto-selects `main`/`master`
4. `git fetch` + `git pull --ff-only`
5. `docker compose --env-file .env pull`
6. `docker compose --env-file .env up -d --remove-orphans`
7. prune dangling images

Default behavior when optional secrets are missing:
1. `DEPLOY_PATH` -> `/opt/infra`
2. `DEPLOY_BRANCH` -> `main` if present, else `master`
3. `SERVER_ENV_FILE_B64` -> skipped (workflow expects existing server `.env`)

It also validates:
1. git exists on server
2. docker exists on server
3. compose command is available (`docker compose` or `docker-compose`)

## 5) First-Time Bootstrap Checklist

1. Confirm DNS for your domains points to server IP.
2. Open firewall ports 22, 80, 443.
3. Configure server `.env` in deploy path.
4. Add GitHub Secrets.
5. Push to `main` and check Actions logs.
