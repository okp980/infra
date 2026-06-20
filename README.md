# Pine Infra

Docker Compose stack for running the [Pine API](https://hub.docker.com/r/okp980/pine-api) in production. Nginx terminates TLS and serves static/media files; the API, Celery worker, Postgres, and Redis run as linked services.

## Services


| Service   | Role                                     |
| --------- | ---------------------------------------- |
| `nginx`   | Reverse proxy, TLS, static/media files   |
| `api`     | Django app (Gunicorn)                    |
| `celery`  | Background task worker                   |
| `migrate` | One-shot database migrations             |
| `db`      | PostgreSQL 16                            |
| `redis`   | Cache, Celery broker, and result backend |


## Production setup

### 1. Prerequisites

- Docker and Docker Compose on the host
- Domain DNS pointing to the server (`NGINX_HOST`)
- TLS certificates under `/etc/letsencrypt` (e.g. via [Certbot](https://certbot.eff.org/))
- `/var/www/certbot` available for ACME HTTP challenges

### 2. Configure environment

```bash
cp .env.sample .env
```

Fill in all required values in `.env`, including:

- `DJANGO_SECRET_KEY` — generate with `python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"`
- `DJANGO_ALLOWED_HOSTS`, `DJANGO_CORS_ALLOWED_ORIGINS`
- Database credentials (`DATABASE_*`)
- Paystack, Resend, and email settings
- `NGINX_HOST` — must match your TLS certificate paths
- `PINE_API_TAG` — Docker image tag to deploy (e.g. `latest`, `v1.0.0`)

### 3. Pull the API image

```bash
export PINE_API_TAG=latest   # or a pinned tag

docker pull okp980/pine-api:${PINE_API_TAG}
```

Pin a specific tag in production rather than relying on `latest`.

### 4. Start the stack

```bash
docker compose up -d
```

Migrations run automatically via the `migrate` service before `celery` starts.

### 5. Verify

```bash
docker compose ps
curl -I https://${NGINX_HOST}/health-check/
```

## Inspecting containers

List all services and their status:

```bash
docker compose ps
```

Follow logs for every service:

```bash
docker compose logs -f
```

Logs and shell access per service:

```bash
# nginx
docker compose logs -f nginx
docker compose exec nginx nginx -t

# api
docker compose logs -f api
docker compose exec api sh

# celery
docker compose logs -f celery
docker compose exec celery celery -A config inspect ping

# db
docker compose logs -f db
docker compose exec db pg_isready -U ${DATABASE_USERNAME:-postgres}
docker compose exec db psql -U ${DATABASE_USERNAME:-postgres} -d ${DATABASE_NAME:-pine}

# redis
docker compose logs -f redis
docker compose exec redis redis-cli ping

# migrate (one-shot; may already be stopped)
docker compose logs migrate
```

Inspect resource usage and metadata:

```bash
docker compose top
docker inspect $(docker compose ps -q api)
```

Restart or recreate a single service after config changes:

```bash
docker compose up -d --no-deps api
docker compose restart nginx
```

