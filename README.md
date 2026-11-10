# zulip-compose

Production-ready Docker Compose deployment for [Zulip](https://zulip.com/) team chat, tailored for GitOps deployment via Portainer behind a Traefik reverse proxy.

Based on the official [zulip/docker-zulip](https://github.com/zulip/docker-zulip) `compose.yaml` (Zulip 12.x):
- `zulip` (`ghcr.io/zulip/zulip-server:12.3-0`)
- `database` (`zulip/zulip-postgresql:14`)
- `memcached` (`memcached:1.6-alpine`)
- `rabbitmq` (`rabbitmq:4.2`)
- `redis` (`redis:8-alpine`)

All service versions are pinned and all persistent state is retained in named Docker volumes.

---

## Key Differences from Upstream

- **Traefik Ingress**: No published container ports on the host. The Zulip application container joins an external Docker proxy network (`proxy` by default) and is exposed via Traefik router labels (`websecure` entrypoint, Let's Encrypt TLS cert resolver).
- **TLS Termination at Proxy**: `CERTIFICATES` remains unset so Zulip's internal Nginx serves HTTP over port 80 while Traefik handles external SSL/TLS termination. `LOADBALANCER_IPS` is configured (default `172.16.0.0/12`) so Zulip trusts `X-Forwarded-For` and `X-Forwarded-Proto` headers from Traefik.
- **Strict Environment Configuration**: Configuration parameters map to `SETTING_*` environment variables and secrets map to Docker Compose secrets sourced from `ZULIP__*` environment variables. No plain secrets are committed to git.
- **GitOps CI/CD**: Includes a GitHub Actions workflow that synchronizes commits from `master` to a `deploy` branch and triggers Portainer's redeploy webhook.

---

## Prerequisites

1. Docker Engine with Docker Compose v2.
2. A running Traefik reverse proxy attached to an external network (e.g. `proxy` or `traefik_default`).
3. DNS record for your domain (e.g. `zulip.example.com`) pointing to your server.
4. An SMTP server with a verified sending domain for transactional emails.
5. (Optional) Google OAuth 2.0 Web Application credentials for Google SSO.

---

## Environment Variables

Copy `.env.example` to `.env` or populate these in Portainer:

| Variable | Required | Default | Description |
|---|---|---|---|
| `ZULIP_HOST` | **Yes** | - | Public domain name for your Zulip instance (e.g. `zulip.example.com`) |
| `SETTING_ZULIP_ADMINISTRATOR` | **Yes** | - | Support and administrator email address |
| `ZULIP__POSTGRES_PASSWORD` | **Yes** | - | Database password (`openssl rand -hex 32`, read on first initialization) |
| `ZULIP__MEMCACHED_PASSWORD` | **Yes** | - | Memcached SASL password (`openssl rand -hex 32`) |
| `ZULIP__RABBITMQ_PASSWORD` | **Yes** | - | RabbitMQ user password (`openssl rand -hex 32`) |
| `ZULIP__REDIS_PASSWORD` | **Yes** | - | Redis password (`openssl rand -hex 32`) |
| `ZULIP__SECRET_KEY` | **Yes** | - | Django secret key (`openssl rand -hex 32`) |
| `SETTING_EMAIL_HOST` | **Yes** | - | Outgoing SMTP host (e.g. `smtp.sendgrid.net`) |
| `SETTING_EMAIL_HOST_USER` | **Yes** | - | SMTP username |
| `ZULIP__EMAIL_PASSWORD` | **Yes** | - | SMTP password |
| `SETTING_EMAIL_PORT` | No | `587` | SMTP port |
| `SETTING_EMAIL_USE_TLS` | No | `True` | Whether to use STARTTLS |
| `SETTING_NOREPLY_EMAIL_ADDRESS` | **Yes** | - | From-address for notifications (e.g. `noreply@example.com`) |
| `TOKENIZED_NOREPLY_EMAIL_ADDRESS` | **Yes** | - | Pattern for reply tokens (e.g. `noreply-{token}@example.com`) |
| `TRAEFIK_NETWORK_NAME` | No | `proxy` | Name of the external Traefik bridge network |
| `TRAEFIK_CERTRESOLVER` | No | `le` | Traefik certificate resolver name |
| `LOADBALANCER_IPS` | No | `172.16.0.0/12` | Trusted IP/CIDR ranges for proxy headers |
| `ZULIP_AUTH_BACKENDS` | No | `EmailAuthBackend` | Authentication backends (e.g. `EmailAuthBackend,GoogleAuthBackend`) |
| `SOCIAL_AUTH_GOOGLE_KEY` | No | - | Google OAuth Client ID |
| `ZULIP__SOCIAL_AUTH_GOOGLE_SECRET` | No | - | Google OAuth Client Secret |
| `SETTING_ZULIP_SERVICE_PUSH_NOTIFICATIONS` | No | `False` | Enable Zulip push notification service |
| `SETTING_ZULIP_SERVICE_SUBMIT_USAGE_STATISTICS` | No | `False` | Submit anonymous usage stats to Zulip |
| `QUEUE_WORKERS_MULTIPROCESS` | No | `False` | `False` uses multi-threaded workers (saves RAM); `True` uses multiprocess |

---

## Deployment (Portainer GitOps)

1. In Portainer, navigate to **Stacks** → **Add stack** → **Repository**.
2. Configure stack repository settings:
   - **Name**: `zulip`
   - **Repository URL**: `https://github.com/<your-username>/zulip-compose`
   - **Repository reference**: `refs/heads/deploy` *(or `master` if managing directly)*
   - **Compose path**: `docker-compose.yml`
3. In **Environment variables**, paste and fill in the values from `.env.example`.
4. Deploy the stack.

### Automated CI/CD with GitHub Actions

This repository includes a deployment workflow (`.github/workflows/deploy.yml`):
1. In Portainer stack settings, enable **Webhook** and copy the webhook URL.
2. In your GitHub repository, navigate to **Settings** → **Secrets and variables** → **Actions**.
3. Create a repository secret named `PORTAINER_REDEPLOY_HOOK` and paste the Portainer webhook URL.
4. Any push to `master` will update the `deploy` branch and trigger Portainer to pull and redeploy the stack.

---

## First Run & Organization Creation

Zulip does not create a default administrator on initial boot. Once the containers are running and healthy, generate a single-use realm creation URL:

```bash
docker exec -u zulip zulip /home/zulip/deployments/current/manage.py generate_realm_creation_link
```

Open the printed URL in your browser to complete the organization setup and create your initial admin account.

---

## Google SSO Configuration

To enable Google Single Sign-On:

1. Go to the [Google Cloud Console](https://console.cloud.google.com/) → **APIs & Services** → **Credentials**.
2. Create an **OAuth 2.0 Client ID** with Application type **Web application**.
3. Add Authorized redirect URI:
   ```
   https://<ZULIP_HOST>/complete/google/
   ```
4. Set the following environment variables in your stack:
   ```env
   ZULIP_AUTH_BACKENDS=EmailAuthBackend,GoogleAuthBackend
   SOCIAL_AUTH_GOOGLE_KEY=<your-client-id>.apps.googleusercontent.com
   ZULIP__SOCIAL_AUTH_GOOGLE_SECRET=<your-client-secret>
   ```
5. Redeploy the stack.

---

## Mobile Push Notifications

Push notifications require server registration with the Zulip push notification service:

1. Set `SETTING_ZULIP_SERVICE_PUSH_NOTIFICATIONS=True` in your stack environment and redeploy.
2. Run the registration command on the host:
   ```bash
   docker exec -u zulip zulip /home/zulip/deployments/current/manage.py register_server --agree-to-terms-of-service
   ```
3. See [Zulip Push Notification Documentation](https://zulip.readthedocs.io/en/stable/production/mobile-push-notifications.html) for details.

---

## Backups & Maintenance

- **Persistent Volumes**:
  - `zulip`: uploads, system configuration, backups
  - `postgresql-14`: PostgreSQL database files
  - `rabbitmq`: message broker state
  - `redis`: key-value cache
- **Automated DB Dumps**: Daily database backups are saved to `/data/backups/` within the `zulip` volume.
- **Manual Full Backup**:
  ```bash
  docker exec -u zulip zulip /home/zulip/deployments/current/manage.py backup --output=/data/backups/zulip-backup.tar.gz
  ```
- **Secret Rotation**: Note that altering `ZULIP__POSTGRES_PASSWORD` after the initial boot requires running `ALTER ROLE` in PostgreSQL. See [docker-zulip password rotation docs](https://zulip.readthedocs.io/projects/docker/en/latest/how-to/compose-secrets.html#rotating-the-postgresql-password).

---

## License

This project is licensed under the [MIT License](LICENSE).
