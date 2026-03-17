---
title: Django, RDS, and AWS Lambda – Connection Pooling Explained
date: 2026-03-15 00:00:00 +/-0000
categories: [Databases, Connections, Django]
tags: [Concept]     # TAG names should always be lowercase
---

## Connection Pooling with Django and AWS RDS

Both **Django** and **Amazon RDS** do not provide native connection pooling. Instead, Django uses **persistent connections per thread**.

### How it Works

1. Every **uWSGI/Gunicorn process** creates a database connection and uses it until:
   - `CONN_MAX_AGE` is reached
   - The worker dies
   - The network drops

2. Connections are **created lazily** (only when the first request requiring DB access arrives).

3. Example connection calculation:
4 servers × 4 processes × 2 threads = 32 concurrent DB connections

4. Recommended `CONN_MAX_AGE` in production:
60 – 300 seconds

5. When scaling, if **maximum concurrent connections exceed the DB limit**, an **external pooling layer** becomes necessary.

6. For **PostgreSQL**, the industry standard pooling solution is:

- **PgBouncer**

7. Creating a new DB connection is an **expensive operation**, and excessive connection churn can become a **performance bottleneck**.

8. If:

```python
CONN_MAX_AGE = 0
```
Django **closes the database connection after every request**.
# Using the Same RDS Setup with AWS Lambda

AWS Lambda **does not provide connection pooling**.

Each **Lambda execution environment** creates its **own database connection**.

Example:

500 concurrent lambdas → 500 direct DB connections

This can become dangerous for **performance and scalability**.

## Recommended Solution

### RDS Proxy

**Amazon RDS Proxy** is the officially recommended approach when using **Lambda with relational databases**.

It works by:

-   Maintaining a **connection pool**
    
-   **Multiplexing thousands of Lambda connections**
    
-   Using **tens or hundreds of actual DB connections**
    

This protects the database from **connection storms**.

----------

# Common Misconception

## “My queries are fast, so connections don't matter”

In reality:

Operation

Typical Time

Query Execution

~2 ms

Connection Setup

~30 ms

This means **~90% of database latency may come from connection overhead**, not the query itself.

----------

# Why NoSQL is Popular with Serverless

NoSQL systems **do not rely on the traditional connection model** used by relational databases.

Instead, they operate on a **throughput-based model**.

Scaling is based on:

-   Requests per second
    
-   Read/write throughput
    
-   Capacity units
    

### Key Characteristics

-   Connections are **managed internally**
    
-   Requests are **stateless**
    
-   Scaling happens **horizontally**
    
-   Communication often happens through **HTTP APIs**
    

This is why NoSQL databases are widely preferred in **serverless architectures like AWS Lambda**.

----------

# AWS Lambda Startup Time

Lambda execution time depends on whether the invocation is a:

-   **Warm start**
    
-   **Cold start**
    

## Warm Start

Most invocations are warm starts.

Typical latency:

1 – 5 ms

This occurs when the Lambda execution environment was recently used.

## Cold Start

Cold starts occur when a **new execution environment** must be created.

Typical latency:

100 ms – 2 seconds

Cold starts happen when:

1.  The function has not been invoked recently
    
2.  There is a sudden spike in concurrency
    
3.  A new version of the function is deployed
    
4.  Memory or architecture configuration changes
    

----------

# Case Study: Using Lambda for Real-Time Traffic

## When Lambda Works Well

Lambda is preferred when:

-   Traffic patterns are **unpredictable**
    
-   **Burst traffic** is expected
    
-   Low-latency processing is required
    

Typical request flow:

User → Lambda Invocation → Process Request → Return Response

Many applications use Lambda to serve **real-time user requests**.

## When Lambda May Not Be Ideal

If:

-   Traffic is **steady and consistently high**
    
-   Workloads are **long-running**
    
-   Business logic is **complex**
    

Then it may be better to use:

-   **EC2 with Auto Scaling**
    
-   **Kubernetes / EKS**
    

Lambda also has limitations such as:

-   Execution time limits
    
-   Cold start delays
