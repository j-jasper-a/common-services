# common-services

A docker-compose stack of backend services used as shared dev infrastructure across local projects.

## Prerequisites

Docker Engine with Compose v2 (`docker compose`, not `docker-compose`).

## Services

| Service       | Image                                                 | Ports           |
| ------------- | ----------------------------------------------------- | --------------- |
| PostgreSQL    | Local build from `postgres:17-trixie` with pg_cron    | `5432`          |
| Redis         | `redis:8.8.0-alpine`                                  | `6379`          |
| Elasticsearch | `docker.elastic.co/elasticsearch/elasticsearch:9.4.3` | `9200`, `9300`  |
| Consul        | `hashicorp/consul:2.0`                                | `8500`          |
| RabbitMQ      | `rabbitmq:4.3.2-alpine`                               | `5672`, `15672` |

All services include healthchecks and persist data to named Docker volumes.

## Connection details

- **PostgreSQL**: `localhost:5432`, database `pathao_enterprise`, user/password `postgres`/`postgres`; connection URL `postgresql://postgres:postgres@localhost:5432/pathao_enterprise`
- **Redis**: `localhost:6379`
- **Elasticsearch**: `http://elastic:elastic@localhost:9200`
- **Consul**: UI & HTTP API at `http://localhost:8500`
- **RabbitMQ**: AMQP at `localhost:5672` (default `guest`/`guest`); management UI at `http://localhost:15672`

The Elasticsearch password is set in `docker-compose.yml` — change `ELASTIC_PASSWORD` if `elastic` doesn't suit.

PostgreSQL preloads pg_cron and initializes the extension in `pathao_enterprise`. Cron jobs use background workers so they do not need separate connection credentials. Scripts in `scripts/postgres-init` run only when the PostgreSQL data volume is first initialized. PostgreSQL data persists in the `postgres_data` named volume.

If the volume already exists without the extension, enable it manually:

```sh
docker compose exec postgres psql -U postgres -d pathao_enterprise -c 'CREATE EXTENSION IF NOT EXISTS pg_cron;'
```

## Usage

Start the stack:

```sh
docker compose up -d
```

Build and start only PostgreSQL:

```sh
docker compose up -d --build postgres
```

Port `5432` must be available; stop any other local PostgreSQL container using it before starting this service.

Check status:

```sh
docker compose ps
```

View logs (all services, or one):

```sh
docker compose logs -f
docker compose logs -f elasticsearch
```

Stop and remove containers (volumes are preserved):

```sh
docker compose down
```

Wipe all data and start fresh:

```sh
docker compose down -v
```

## Using these services from another project

The stack exposes a shared network named `common-services`. Attach your project's `docker-compose.yml` to it so containers can reach services by name (`postgres`, `redis`, `elasticsearch`, `consul`, `rabbitmq`).

Example `docker-compose.yml` for a consuming project:

```yaml
services:
  api:
    build: .
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/pathao_enterprise
      REDIS_URL: redis://redis:6379
      ELASTICSEARCH_URL: http://elastic:elastic@elasticsearch:9200
    networks:
      - common

networks:
  common:
    external: true
    name: common-services
```

From your host, your app still talks to `localhost:6379`, `localhost:9200`, etc. From inside containers, use the service names.
