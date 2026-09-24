# NRApp CV bullets — verified draft

Drafted against the production VPS and repository evidence checked on 23 Sep 2026. The English bullets are copy-ready; keep only the ones relevant to the role and available CV space.

## Recommended version

- **Implemented email/OTP registration and login with JWT access/refresh tokens, Google ID-token sign-in, and Gateway RBAC for `user`/`admin` roles. Stored OTP state in Redis with a 5-minute TTL, limited OTP requests to one per email every 60 seconds, and blocked verification after five failed attempts to reduce OTP abuse and email-quota exposure.**

- **Decoupled OTP email delivery from Auth through a durable RabbitMQ queue and a separate Mail consumer, with up to five retries at 5-second intervals before dead-lettering; this lets transient mail failures be retried without keeping SMTP work inside the login request.**

- **Built employee chat on Socket.IO bidirectional WebSocket connections, allowing clients to receive messages in real time without repeatedly polling the chat API.**

- **Deployed eight active Node.js 22/NestJS 11 services on an Ubuntu 22.04 VPS (2 vCPU, 3.9 GiB RAM) using Docker Compose and Nginx HTTPS. Per-service GitHub Actions deployments wait for container health checks (30-second interval) and restore the previous image after an unhealthy rollout; the VPS history contains 50 successful deployments.**

- **Configured Prometheus and node_exporter to collect VPS CPU, memory, and disk metrics every 60 seconds; combined container health checks with JSON request-ID logs to support service-health checks and cross-service troubleshooting. Both Prometheus targets were healthy during verification.**

- **Automated daily MongoDB Atlas and Redis backups at 02:30 ICT, encrypting archives with Age and verifying SHA-256; retained at least seven copies across a 14-day window and offsite GitHub Actions artifacts for 30 days. The latest archive completed in 19 seconds; an isolated restore verified 21 MongoDB collections/76 documents and four Redis keys in 4.97 seconds.**

## Optional performance bullet

Use this only if you are prepared to explain that the test missed its latency target:

- **Benchmarked three authenticated read APIs with k6 at 50 virtual users sharing one account: 26.45 RPS and 100% HTTP success during a 60-second steady-load window, with end-to-end p95/p99 of 1.40/1.70 seconds against a 500 ms target. A separate Auth profile measured credential-query p95 at 258 ms (1.25-second maximum) and credential-wait p95 at 532 ms, while Mongo connection-pool checkout p95 was 1.03 ms with no observed pool queue. Enabled a bounded 2-second in-process identity cache with invalidation on role/email changes and account deletion; post-change API latency still needs a fresh load test.**

This is a truthful characterization of a baseline, not a claim that the API met the target or that MongoDB alone caused the latency. The 50-VU test was not a three-minute hold, used one shared account, and had empty task/chat data.

## Claims to remove or qualify

- Do not say **“Redis caching for read APIs”** or quote a Redis cache-hit ratio. Redis is used for OTP/refresh-token state and Canteen undo/redo. Auth identity caching is in-process, configured to 2 seconds on the single VPS replica after the initial production verification; its performance benefit has not yet been measured.
- Do not claim **Grafana dashboards, resource alerts, or incident detection**. Grafana was running, but had zero dashboards and alert rules. Describe Prometheus/node_exporter collection only.
- Do not claim a **measured rollback duration** or zero downtime. The receiver has automatic rollback logic and 50 successful deployments are recorded, but no rollback duration was measured.
- Do not claim **26.5 RPS at 50 VUs for three minutes**. The verified test held 50 VUs for 60 seconds; p95/p99 were over the 500 ms target.
- Do not label request IDs as **distributed tracing**. They support log correlation; OpenTelemetry tracing was disabled on the verified VPS.
- Treat the 4.97-second restore as a **small isolated test**, not a production RTO: the sample held 76 MongoDB documents and four Redis keys.

## Shorter version for a one-page CV

- **Built JWT/Google sign-in and Gateway RBAC; used Redis-backed OTP state with one resend per email per minute and a five-attempt verification limit, and RabbitMQ retries/DLQ to decouple email delivery from Auth.**
- **Deployed eight Node.js/NestJS services with Docker Compose, HTTPS, per-service CI/CD, health-gated rollout and automatic rollback; recorded 50 successful deployments on a 2-vCPU/3.9-GiB VPS.**
- **Automated encrypted daily MongoDB/Redis backups with SHA-256 and offsite retention; verified an isolated restore of 21 collections/76 documents and four Redis keys in 4.97 seconds.**
- **Collected VPS CPU/RAM/disk metrics every 60 seconds with Prometheus/node_exporter and used container health checks plus request-ID JSON logs for operational diagnosis.**

Include the performance bullet only when the role values load testing and you can discuss its latency result candidly.
