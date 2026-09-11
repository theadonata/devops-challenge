# Problem 1: Building Castle In The Cloud

A design for a highly available, scalable, cost-effective trading system on AWS, covering the three deliverables asked for:
1. An overview diagram of the services used and the role each plays.
2. Why each service was chosen and what alternatives were considered.
3. Plans for scaling beyond the current setup.

**Features covered**:
1. **Wallet** — deposit, check balance, withdraw.
2. **Orders** — place a spot order, cancel a spot order.
3. **Matching** — match buy and sell orders against each other.

---

### Role of each component

**Loading the app:** the browser's first request resolves through **Route 53** to **CloudFront**, which serves the trading app's static files from an **S3** bucket (the frontend build) — no backend service involved yet.

**Placing and matching an order:**
1. Once loaded, the app calls the API directly: `POST /orders` over HTTPS, resolved by **Route 53** and filtered by **WAF**, then the **Application Load Balancer** picks a healthy container.
2. **Order Service** (ECS Fargate) verifies the request's Cognito token, writes the order to **DynamoDB** as `status: PENDING`, and publishes a "new order" event to the **`orders` Kafka topic**
3. The **Matching Engine** for that pair — a single process holding that pair's order book in memory for speed — reads events **in the exact order they arrived** and checks if a new order crosses with one already on the book.
   - If it matches: a trade is created and published to the **`trades` topic**.
   - If not: it's added to the in-memory book and mirrored into **Redis**, so a replacement engine can rebuild the book instantly if this one crashes.
4. **Wallet Service** consumes the `trades` topic and moves funds between the buyer's and seller's balances in **Aurora**, inside a database transaction (money must never be double-spent or lost).

**Cancelling an order:** `DELETE /orders/{id}` follows the same path into Order Service, which checks the order's current status in DynamoDB, then publishes a "cancel" event to the **same** `orders` topic partition as the original order — not a side channel — so the Matching Engine sees the cancellation in the correct sequence relative to any orders that arrived around the same time, instead of racing a match that might already be in flight.

**Depositing, checking, or withdrawing:** these go straight to **Wallet Service**, which reads or writes the balance in **Aurora** directly — no Kafka hop needed, since there's no matching or multi-party settlement involved, just a single account's balance.

Because the order→match→settle path hands off through Kafka instead of one service calling the next and waiting, a slow or restarting service doesn't freeze the chain — events just queue up for a few milliseconds and get processed as soon as it's back.

---

## Deliverable 2: Services used, why, and alternatives considered

| Layer | AWS Service | What it does here | Why this one |
|---|---|---|---|
| DNS & failover | **Route 53** | Routes users to the app, runs health checks | Can redirect traffic to another region automatically if AWS has a regional outage |
| Frontend hosting | **S3** | Stores the built frontend (JS/CSS/HTML) as static files | Cheapest way to host a static single-page app; no servers needed to serve files that don't change per-request — see "Why S3 + CloudFront instead of AWS Amplify Hosting" below |
| CDN / edge | **CloudFront** | Serves the frontend from S3, cached close to users worldwide | Cuts page load time; also absorbs large traffic spikes (like a DDoS) before they reach the S3 origin — see "Why S3 + CloudFront instead of AWS Amplify Hosting" below |
| Web protection | **AWS WAF** | Filters SQL injection, bot traffic, applies rate limits | Standard first line of defense for anything public-facing and money-related |
| Load balancing | **Application Load Balancer (ALB)** | Spreads incoming traffic across containers in 3 AZs | Simple, managed, and integrates directly with ECS Fargate's health checks — see "Why ALB instead of API Gateway" below |
| Compute | **ECS Fargate** | Runs Order, Wallet, and Matching Engine as containers, auto-scales, no servers to patch | ECS over EKS: three services, no multi-cloud need, and no existing Kubernetes expertise to leverage. Fargate over EC2/Lambda: no host patching, and no cold starts for a matching engine that must hold its book in memory between requests |
| Order matching | *(custom service, on ECS Fargate)* | The application logic that holds each trading pair's order book in memory and matches buy/sell orders | Not an AWS product — matching logic is domain-specific code this design owns; Fargate just keeps that process running, restarts it on failure, and scales it |
| Identity | **Amazon Cognito** | Issues a JWT on login; Order and Wallet services verify it locally on every request | No dedicated Auth microservice needed — verifying a signed token is a local, sub-millisecond check against Cognito's public keys, not a network hop |
| Event backbone | **Amazon MSK (managed Kafka)** | Carries order/cancel/trade events between services, **in order**, per trading pair | Ordering guarantees are non-negotiable for a matching engine — SQS can't guarantee cross-consumer ordering, and Kafka's replayable log lets a crashed Matching Engine rebuild its state exactly |
| Balances | **Aurora PostgreSQL (Multi-AZ)** | Stores wallet balances | Moving money needs real ACID transactions (all-or-nothing); a relational DB is the right tool here |
| Orders | **DynamoDB** | Stores each order and its current status (open/filled/cancelled) | Keeps the system's highest-volume write path off Aurora entirely, so order traffic can never contend with wallet traffic for write capacity. Orders need no cross-row transactions, just fast lookups by ID — exactly DynamoDB's strength |
| Order book cache | **ElastiCache (Redis)** | Keeps a fast, shared snapshot of each order book | Lets a replacement Matching Engine instance recover in milliseconds instead of replaying the entire day's orders — see "Why ElastiCache Redis instead of Memcached or DAX" below |
| Monitoring | **CloudWatch + X-Ray** | Metrics, logs, alarms, and traces one request across all services | Needed to actually prove the p99 < 100ms target is being met, and to find which service is slow when it isn't |
| Secrets | **Secrets Manager + KMS** | Stores DB passwords/API keys, encrypts data at rest | Nothing sensitive (DB creds, private keys) should live in code or plain config — see "Why Secrets Manager instead of SSM Parameter Store" below |
| Networking | **VPC** with public + private subnets | Public subnets only hold the load balancer; everything else (containers, databases) is private | Databases and internal services should never be reachable directly from the internet |

### ALB vs. API Gateway

| ALB | API Gateway | Tradeoff |
|---|---|---|
| Routes directly to ECS with minimal latency; handles path-based routing and health checks across 3 AZs | Adds per-key throttling, usage plans, request validation, native Cognito authorizers | None of API Gateway's API-management features are needed — no external partners requiring quotas — and its extra hop adds latency and per-request cost directly against the p99 < 100ms budget. **ALB chosen.** |

### S3 + CloudFront vs. AWS Amplify Hosting

| S3 + CloudFront | Amplify Hosting | Tradeoff |
|---|---|---|
| Explicit, composed control over CDN and storage config | Git-push CI/CD, managed CloudFront distribution, and PR preview URLs bundled together | Amplify is faster to set up and nicer day-to-day, but trades away granular config control for a more opinionated build pipeline. **S3 + CloudFront chosen** to keep that control and match this design's pattern of composing primitives — Amplify is a reasonable pick for a team that values shipping speed over that control. |

### Secrets Manager vs. SSM Parameter Store

| Secrets Manager | Parameter Store | Tradeoff |
|---|---|---|
| Native automatic rotation for RDS/Aurora credentials | Free SecureString tier (~$0.40/secret/month cheaper), still KMS-encrypted | Parameter Store has no built-in rotation — rotating a credential means building your own Lambda + EventBridge schedule. **Secrets Manager chosen** for native rotation on the credential guarding wallet balances; the cheaper option just shifts the work onto the team for its most sensitive secret. |


### CloudWatch + X-Ray vs. Grafana

| CloudWatch + X-Ray | Grafana + Prometheus + Jaeger | Tradeoff |
|---|---|---|
| Native AWS integration, one IAM boundary, no extra system to run | Generally stronger dashboards and query experience, cloud-agnostic | Grafana costs either a second vendor bill (Grafana Cloud) or owning the monitoring stack's own uptime (self-hosted) — a real risk if it's down during the incident it's needed for. **CloudWatch + X-Ray chosen** to keep monitoring inside the same IAM boundary as everything else, with AWS operating it. |

### Secrets Manager vs. HashiCorp Vault

| Secrets Manager | HashiCorp Vault | Tradeoff |
|---|---|---|
| Native AWS integration, no extra system to run or secure | More powerful (dynamic secrets, finer-grained policies), cloud-agnostic | Vault is another system to run and secure — adding operational risk to the thing meant to reduce risk, unless using HCP Vault's managed offering. **Secrets Manager chosen** for the same reason as everywhere else in this design: one IAM boundary instead of a second system to operate. |

---

## Deliverable 3: Plans for scaling beyond the current setup

1. **More containers.** ECS auto-scaling adds Fargate tasks for Order Service and Wallet Service based on CPU/request count. This is close to "free" scaling since both are already stateless.
2. **More Matching Engine lanes.** Add more Kafka partitions and run more Matching Engine instances — one busy trading pair can even get dedicated compute while quiet pairs share a process.
3. **Database read replicas.** Aurora read replicas absorb read-heavy traffic (e.g. repeated balance checks) so the primary is reserved for writes (deposits, withdrawals, settlements). DynamoDB scales its throughput automatically (or via on-demand capacity mode) as order volume grows.
4. **Bigger cache.** ElastiCache can scale out to more shards/replicas so hot order books never become a bottleneck.
5. **Go multi-region.** For global users, deploy the same stack in a second region, use Route 53 latency-based routing to send each user to their nearest region, and replicate balances via Aurora Global Database.
7. **Protect the core from spikes.** Add stricter rate limiting at WAF and let Kafka's queue naturally absorb short bursts, rather than scaling the Matching Engine itself to match worst-case spikes.
