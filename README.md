# DBAP Platform Infrastructure

[![CI](https://github.com/pancakebaker/docker-dbap-platform/actions/workflows/validation.yml/badge.svg)](https://github.com/pancakebaker/docker-dbap-platform/actions/workflows/validation.yml)

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

### Prerequisites

Install Docker Desktop with Compose v2, or Docker Engine with the Compose
plugin. Start Docker before running the commands below.

### Create local configuration

From this repository root, create the local environment file before any Compose
command:

Unix/macOS:

```bash
cp .env.example .env
```

Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

The template contains development-only placeholders. Review the values if
another local service uses different ports or credentials, but do not put real
secrets in this file or commit it.

### Validate configuration

Validate the resolved Compose model before starting containers:

```powershell
docker compose config
```

This checks the Compose file after environment substitution and reports the
services, mounts, ports, and other resolved settings that Docker will use.

### Start infrastructure

Start the three infrastructure services:

```powershell
docker compose up -d postgres rabbitmq redis
docker compose ps
```

`docker compose up -d` is also valid and starts the same three services because
this Compose file contains no application services. The stack starts
infrastructure only. It does not apply EF migrations, create application
tables, seed tenants/auctions/bids/users/credentials, or start any application
workers.

### Verify infrastructure

Wait until `docker compose ps` reports `healthy` for all three services. The
Compose health checks are the authoritative readiness checks:

```powershell
docker compose ps
docker compose exec -T postgres pg_isready -U auction_app -d auction_demo
docker compose exec -T rabbitmq rabbitmq-diagnostics -q ping
docker compose exec -T redis redis-cli ping
```

The final commands should report PostgreSQL accepting connections, RabbitMQ
responding to `ping`, and `PONG` from Redis. The host endpoints are:

| Service | Endpoint | Purpose |
| --- | --- | --- |
| PostgreSQL | `127.0.0.1:55432` | Shared PostgreSQL server |
| RabbitMQ AMQP | `127.0.0.1:5672` | Application event transport |
| RabbitMQ management | <http://localhost:15672> | Broker management UI |
| Redis | `127.0.0.1:6379` | Runtime key-value service |

The development RabbitMQ username is `auction` and the password is the
placeholder from `.env.example`. PostgreSQL uses the configured development
database/user/password values. Redis has no password configured by this
Compose file.

### Initialize application databases

The PostgreSQL container initializes logical databases only when its data
volume is first created. The checked-in SQL creates `auction_operations` in
addition to the `POSTGRES_DB` database (normally `auction_demo`). It creates no
application tables.

After the infrastructure is healthy, initialize each application from its own
repository:

1. In [`dotnet-bidding-service`](https://github.com/pancakebaker/dotnet-bidding-service), apply the Bidding EF migrations and follow its documented local/demo seeding workflow. Bidding owns the `auction_demo` schema and all demo tenants, auctions, bids, credentials, and related data.
2. In [`dotnet-blazor-operations-portal`](https://github.com/pancakebaker/dotnet-blazor-operations-portal), apply the Operations EF migrations. Operations owns the `auction_operations` schema and its local system-administrator account behavior.
3. If Live Feed history is enabled, apply the Live Feed repository's own history setup. That schema remains Live Feed-owned.
4. Laravel owns its own SQLite/local database setup and migrations in [`laravel-react-auction-web`](https://github.com/pancakebaker/laravel-react-auction-web).

Do not add application migrations or demo seed data to this infrastructure
repository.

### Fresh local setup

For a complete local platform starting from zero:

1. Clone this repository and the application repositories as siblings.
2. Copy `.env.example` to `.env` using the command above.
3. Run `docker compose config`.
4. Start `postgres`, `rabbitmq`, and `redis`.
5. Wait for all three health checks to report `healthy`.
6. Follow the Bidding Service README to migrate and seed its local/demo database, then start the API, Scheduler, and Outbox Publisher.
7. Follow the Operations Portal README to migrate `auction_operations`, configure its local administrator material, and start the portal.
8. Start Live Feed and Laravel/React using their own repository instructions when those features are required.

### Stop infrastructure

Stop the containers while retaining named volumes and persisted development
data:

```powershell
docker compose down
```

Restart later with `docker compose up -d postgres rabbitmq redis`; the existing
volumes will be reused.

### Reset local infrastructure

To intentionally remove this Compose project's containers and named volumes:

```powershell
docker compose down -v
```

This removes `postgres_data`, `rabbitmq_data`, and `redis_data`, including the
development data stored in them. It does not remove bind-mounted files on the
host and only targets resources associated with the current Compose project.
The next startup will run the PostgreSQL init SQL again, creating the logical
databases but still not creating application tables or seed data.

### Troubleshooting old Compose projects

If containers remain after `docker compose down -v`, they may have been created
from the historical monorepo or another Compose project name. Inspect before
removing anything:

```powershell
docker compose ls
docker ps -a
docker volume ls
```

Stop and reset the old project from the directory and Compose file that created
it. Do not use broad commands such as `docker system prune --volumes` as the
normal DBAP reset procedure.

The localized PowerShell helper is a startup convenience only:

```powershell
.\scripts\start-infrastructure.ps1
```

It changes to this repository's directory and runs `docker compose up -d`. It
does not copy `.env.example`, validate the resolved configuration, wait for
health, apply application migrations, or seed demo data.

## Related repositories

- [Laravel React Auction Web](https://github.com/pancakebaker/laravel-react-auction-web)
- [Node.js Live Feed](https://github.com/pancakebaker/nodejs-live-feed)
- [.NET Operations Portal](https://github.com/pancakebaker/dotnet-blazor-operations-portal)
- [.NET Bidding Service](https://github.com/pancakebaker/dotnet-bidding-service)
- [Historical integrated monorepo](https://github.com/pancakebaker/distributed-bidding-auction-platform)

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
