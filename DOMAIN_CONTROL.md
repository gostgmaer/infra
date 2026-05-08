# Centralized Traefik + Portainer Domain Control

This setup gives you:
- `https://portainer.easydev.in` for Portainer
- `https://traefik.easydev.in` for Traefik dashboard
- One central file for domain values: `.env`

## 1) Set Domain Values

1. Copy `.env.example` to `.env`
2. Edit values in `.env`:
   - `BASE_DOMAIN`
   - `PORTAINER_SUBDOMAIN`
   - `TRAEFIK_DASHBOARD_SUBDOMAIN`
   - `LETSENCRYPT_EMAIL`

Example result:
- Portainer URL: `https://${PORTAINER_SUBDOMAIN}.${BASE_DOMAIN}`
- Traefik Dashboard URL: `https://${TRAEFIK_DASHBOARD_SUBDOMAIN}.${BASE_DOMAIN}`

## 2) DNS Requirements (Domain Control)

Create DNS records that point to your server public IP:
- `A` record: `portainer` -> `<YOUR_SERVER_PUBLIC_IP>`
- `A` record: `traefik` -> `<YOUR_SERVER_PUBLIC_IP>`

Optional:
- If you will expose more apps later, use wildcard DNS:
  - `A` record: `*` -> `<YOUR_SERVER_PUBLIC_IP>`

## 3) Network / Firewall Requirements

Open inbound ports to this host:
- `80/tcp` (required for Let's Encrypt HTTP challenge)
- `443/tcp` (HTTPS traffic)

## 4) Start the stack

```bash
docker compose --env-file .env up -d
```

## 5) Add more domains later (where needed)

For any new service you add behind Traefik:
1. Connect the service to network `${TRAEFIK_NETWORK_NAME}`
2. Add Traefik labels with host rule:

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.myapp.rule=Host(`myapp.${BASE_DOMAIN}`)
  - traefik.http.routers.myapp.entrypoints=websecure
  - traefik.http.routers.myapp.tls=true
  - traefik.http.routers.myapp.tls.certresolver=le
  - traefik.http.services.myapp.loadbalancer.server.port=<APP_INTERNAL_PORT>
```

3. Add DNS record:
- `A` record: `myapp` -> `<YOUR_SERVER_PUBLIC_IP>`

## 6) Notes

- Keep Cloudflare in **DNS only** mode while issuing certificates with HTTP challenge.
- If you want proxy mode enabled permanently, switch Traefik ACME to DNS challenge.
- Docker socket access grants control over Docker; limit host access to trusted admins.
