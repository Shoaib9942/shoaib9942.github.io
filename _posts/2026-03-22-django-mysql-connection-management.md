---
title: Django & MySQL – Connection Management and Pooling
date: 2026-03-15 00:00:00 +/-0000
categories: [Databases, Connections, Django]
tags: [django, mysql, connection-pooling, rds, lambda, architecture]
---


## How Django Manages MySQL Connections

A common misconception is that Django provides connection pooling out of the box. It doesn't and neither does the `mysqlclient` or `PyMySQL` drivers Django relies on. What Django offers instead is **persistent connections per worker thread**, a simpler but meaningfully different mechanism.

Understanding this distinction matters at scale.

---

## The Default Lifecycle: Per-Request Connections

By default, Django's database connection lifecycle is tied to each request. The relevant setting is `CONN_MAX_AGE`, which defaults to `0`:
```python
# settings.py (default behavior)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'CONN_MAX_AGE': 0,  # close connection after every request
    }
}
```

With `CONN_MAX_AGE = 0`:
- A new TCP connection to MySQL is established on the first DB operation per request.
- Django closes the connection in the `request_finished` signal handler after the response is sent.
- No connection is shared between requests or threads.

This is safe and predictable but carries real cost. Establishing a new MySQL connection involves a TCP handshake, TLS negotiation (if applicable), authentication, and session initialization. This overhead is typically **20–100ms**, dwarfing typical query execution times of **1–5ms**. In a 500 RPS application, this translates to hundreds of wasted connection setups per second.

---

## Persistent Connections: `CONN_MAX_AGE`

Setting `CONN_MAX_AGE` to a positive integer tells Django to keep the connection open and reuse it across subsequent requests handled by the same thread:
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'CONN_MAX_AGE': 60,  # reuse connection for up to 60 seconds
    }
}
```

Internally, Django stores open connections in a thread-local registry (`django.db.connections`). On each request, it checks whether a usable connection exists for the current thread. If yes, it reuses it; if the connection is stale or exceeded `CONN_MAX_AGE`, it closes and re-establishes it.

You can also set `CONN_MAX_AGE = None` for indefinitely persistent connections — though this requires careful handling of connection errors and reconnects, since MySQL's `wait_timeout` will silently drop idle connections server-side (default: 8 hours).

**When Django closes a persistent connection:**
- The `CONN_MAX_AGE` duration has elapsed.
- The connection raises an `OperationalError` (e.g., `MySQL server has gone away`).
- The process/thread is recycled by the WSGI server.

---

## Why This Is Not Connection Pooling

True connection pooling involves a **shared pool** managed independently of application threads: connections are checked out, used, and returned to the pool for reuse by *any* thread. PgBouncer (PostgreSQL) and ProxySQL (MySQL) work this way.

Django's mechanism is strictly **per-thread**. There is no shared pool. Under Gunicorn or uWSGI, each worker process and each thread within it holds its own connection:
```
4 instances × 4 workers × 2 threads = 32 concurrent MySQL connections
```

With `CONN_MAX_AGE > 0`, those connections are long-lived. With `CONN_MAX_AGE = 0`, each of those 32 threads opens and closes a connection per request.

This has a critical implication: **your MySQL `max_connections` must accommodate your full worker concurrency**, not just your query concurrency. A spike from 32 to 64 instances doubles your connection count directly, regardless of how many of those threads are actually querying the DB at any moment.

> **Practical limit:** MySQL's default `max_connections` is 151. A 10-instance Django deployment with 4 workers × 4 threads would consume 160 connections — already over the default limit.

---

## Application-Level Pooling with Django

For scenarios where you want true pooling within the application layer, there are two common approaches:

### 1. `django-db-geventpool`
Useful for gevent-based async setups. Maintains a pool per database alias.

### 2. `SQLAlchemy` connection pool (via custom backend)
SQLAlchemy's `QueuePool` can be wired into a custom Django database backend. This gives you pool sizing, overflow limits, and connection health checks:
```python
from sqlalchemy import pool
from sqlalchemy import create_engine

engine = create_engine(
    "mysql+mysqlclient://user:pass@host/db",
    poolclass=pool.QueuePool,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,  # validates connections before use
)
```

`pool_pre_ping=True` is important: it issues a cheap `SELECT 1` before returning a connection from the pool, avoiding the `MySQL server has gone away` error on stale connections.

---

## ProxySQL: Pooling at the Infrastructure Layer

For production systems, an infrastructure-level proxy is often a better fit than application-level pooling. **ProxySQL** sits between Django and MySQL and maintains a shared connection pool:
```
Django Workers → ProxySQL → MySQL
(many short-lived)  (pool)   (few long-lived)
```

Benefits:
- Read/write splitting (route `SELECT` to replicas automatically)
- Connection multiplexing across backends
- Query mirroring, firewall rules, and slow query logging
- Transparent to the application — Django sees it as a regular MySQL host

---

## The AWS Lambda Problem

Lambda's execution model makes connection management significantly harder.

Each Lambda invocation runs in an isolated execution environment. If a new environment is created (cold start or scale-out), it opens a fresh database connection. There is no persistent worker model:
```
500 concurrent Lambdas → up to 500 simultaneous MySQL connections
```

Unlike a uWSGI worker that persists for hours and amortizes connection cost, Lambda environments can be created and destroyed in seconds. This causes **connection storms** — rapid simultaneous connection attempts that can overwhelm MySQL's `max_connections` and trigger authentication queue saturation.

A partial mitigation is reusing the connection across *warm* invocations by initializing it outside the handler:
```python
import pymysql

# Initialized once per execution environment, reused across warm invocations
conn = pymysql.connect(host=HOST, user=USER, passwd=PASS, db=DB)

def handler(event, context):
    with conn.cursor() as cursor:
        cursor.execute("SELECT ...")
```

But this only helps for warm invocations. At high concurrency with rapid scale-out, it provides limited relief.

### RDS Proxy: The Proper Solution

**Amazon RDS Proxy** sits between Lambda and RDS, maintaining a persistent, sized connection pool to the database:
```
Lambda (thousands of envs)
        ↓
   RDS Proxy (pool: e.g., 20–100 connections)
        ↓
   RDS MySQL / Aurora
```

RDS Proxy multiplexes Lambda connections onto a far smaller set of actual database connections using **connection pinning** (for transactions) and **multiplexing** (for idle connections between statements). It also:
- Handles failover transparently during Multi-AZ events
- Integrates with IAM for credential management (via AWS Secrets Manager)
- Reduces failover time from ~30s to ~5s for Aurora

> **Note on connection pinning:** RDS Proxy cannot multiplex a connection that has active transactions, session-level state (e.g., `SET` variables), or stored procedures with `OUT` parameters. These pin the connection for the lifetime of the Lambda invocation, reducing pooling efficiency. Keep transactions short and avoid session state in Lambda handlers.

---

## The Cost of Connection Overhead

To put numbers on this concretely:

| Operation | Typical Latency |
|---|---|
| MySQL query (indexed, simple) | 1–5ms |
| MySQL connection establishment | 20–100ms |
| Connection via RDS Proxy (warm) | ~2ms overhead |

In a naive Lambda setup without proxy, **60–95% of database latency can come from connection setup**, not the query. This is why the cold-start problem in Lambda is primarily a *connection* problem, not a *compute* problem.

---

## Why NoSQL Works Better in Serverless

Relational databases were designed for the **persistent connection model**: authentication, session state, and transaction semantics are all tied to a connection. This creates fundamental friction with stateless, ephemeral compute.

NoSQL systems like DynamoDB, Cassandra, and MongoDB (Atlas) sidestep this by using a **stateless request model**:
- Each operation is a self-contained HTTP/gRPC request
- No connection handshake is required per request
- Authentication is handled at the request level (e.g., SigV4 for DynamoDB)
- Scaling is based on throughput (RCUs/WCUs, capacity units) rather than connection limits

For DynamoDB specifically, the AWS SDK maintains a small internal HTTP connection pool, but this is managed transparently and never becomes a bottleneck at Lambda scale.

This is the primary reason NoSQL is the default recommendation for serverless workloads — not schema flexibility, but the **fundamental impedance mismatch** between stateful connections and stateless execution environments.

---

## Summary

| Scenario | Recommendation |
|---|---|
| Django + Gunicorn/uWSGI, moderate scale | `CONN_MAX_AGE = 60`, monitor `max_connections` |
| Django + high concurrency, many workers | ProxySQL or PgBouncer-equivalent in front of MySQL |
| Django + AWS Lambda | RDS Proxy (mandatory above trivial scale) |
| Greenfield serverless | DynamoDB or another HTTP-native datastore |


---

## Acknowledgements

Thanks for reading! If you found this useful or have corrections/additions, feel free to reach out or drop a comment.
