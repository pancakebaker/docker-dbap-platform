# DBAP Platform Infrastructure

This repository owns the shared local Docker runtime for the Distributed
Bidding Auction Platform. It provides PostgreSQL, RabbitMQ, and Redis without
containing any application implementation.

This is development/demo orchestration, not a hardened production deployment.
Images are pinned to the platform's current major versions and credentials in
`.env.example` are development-only placeholders.

## Ownership

The platform stack contains:

| Service | Image | Host ports | Role |
| --- | --- | --- | --- |
| PostgreSQL | `postgres:17` | `55432` | Shared database server |
| RabbitMQ | `rabbitmq:4-management` | `5672`, `15672` | AMQP event transport and management UI |
| Redis | `redis:8-alpine` | `6379` | Shared runtime key-value service |

PostgreSQL owns runtime and logical database provisioning only. Bidding owns
its EF migrations and authoritative auction database schema. Operations owns
its migrations and activity database schema. Live Feed's optional history
schema remains Live Feed-owned. The initialization SQL creates the
`auction_operations` logical database; it does not create application tables.

Bidding, Scheduler, Outbox Publisher, Live Feed, Operations, and Laravel/React
remain in their own repositories:

- `dotnet-bidding-service`
- `nodejs-live-feed`
- `dotnet-blazor-operations-portal`
- `laravel-react-auction-web`

No application source, Dockerfiles, event contracts, migrations, or business
logic are owned here. RabbitMQ exchanges, queues, bindings, and Redis keys are
declared and owned by the applications that use them. This repository only
provides the broker and data-store processes.

## Architecture

```text
Laravel/React --HTTP--> Bidding API --transaction--> PostgreSQL + outbox
                                                     |
                                      Outbox Publisher + Scheduler
                                                     |
                                                  RabbitMQ
                                             /                  \
                                      Live Feed              Operations
                                      + Redis                 + PostgreSQL

Bidding client-assertion replay protection -> Redis
Live Feed projection/idempotency state     -> Redis
```

The Bidding API does not synchronously depend on RabbitMQ for its request and
outbox transaction. Scheduler and Outbox Publisher are separate processes in
the Bidding repository. Service-to-service URLs and application credentials
are configured by those service repositories, not by this Compose stack.

## Usage

Requirements: Docker Desktop with Compose v2, or Docker Engine with the
Compose plugin. Copy `.env.example` to `.env` and adjust only for local needs.

Start shared infrastructure:

```powershell
docker compose up -d postgres rabbitmq redis
docker compose ps
```

The services expose PostgreSQL on `127.0.0.1:55432`, RabbitMQ AMQP on
`127.0.0.1:5672`, RabbitMQ management on `127.0.0.1:15672`, and Redis on
`127.0.0.1:6379`. Health checks use `pg_isready`, RabbitMQ diagnostics, and
`redis-cli ping`.

Stop containers while retaining named volumes:

```powershell
docker compose down
```

The named volumes are `postgres_data`, `rabbitmq_data`, and `redis_data`.
`docker compose down -v` is destructive: it removes this stack's persisted
development data and should be used only when an intentional reset is needed.

The localized PowerShell helper is equivalent to `docker compose up -d`:

```powershell
./scripts/start-infrastructure.ps1
```

## Application repositories

For full local development, clone the application repositories as siblings of
this repository. The infrastructure stack itself does not require those
directories and does not use hardcoded host paths. Applications should be
started with their own documented commands and configured with host URLs when
running outside Docker, or Docker DNS names when an application container is
introduced later.

Application migrations remain application-owned. No application images are
claimed here because the current extracted repositories do not provide
service-specific Dockerfiles or published images. Full-platform container
startup is therefore not currently supported; this repository supports
infra-only startup.

## Validation and CI

Validate the resolved Compose model with:

```powershell
docker compose config
```

The repository workflow runs this validation and an infrastructure-only smoke
test for all three services. Application test suites belong to their
respective repositories and are intentionally not duplicated here.

Do not commit `.env`, passwords other than harmless local placeholders, JWT
keys, certificates, or service credentials. Laravel owns its client private
key, Live Feed owns its service private key, and Bidding/Operations consume
the public material they require through their own configuration.
