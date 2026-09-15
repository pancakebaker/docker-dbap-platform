# Platform Runtime Boundary

`docker-dbap-platform` supplies shared development infrastructure only:
PostgreSQL 17, RabbitMQ 4 with management, and Redis 8. It does not own
application processes or application schemas.

```text
Laravel/React -> Bidding API -> Bidding PostgreSQL/outbox
                                      -> Outbox Publisher -> RabbitMQ
                                      -> Auction Scheduler -> Bidding PostgreSQL
RabbitMQ -> Live Feed -> Redis projection state -> browser clients
RabbitMQ -> Operations -> Operations PostgreSQL + SignalR clients
Bidding client assertions -> Redis replay protection
```

The Bidding API is authoritative for auction, bid, tenant, and client
application state. Live Feed Redis data is a projection; Operations PostgreSQL
data is an activity/history projection. Each application repository owns its
own migrations, consumers, producers, authentication, and business rules.
