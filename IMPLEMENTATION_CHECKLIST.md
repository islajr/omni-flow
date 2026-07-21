# 📋 Implementation Checklist & Comprehensive Practice Guide: ForgePay
### Step-by-Step Execution Blueprint for Internalizing Spring Boot Mastery
*Version 2.5 — July 2026*

---

## 🎯 How to Approach This Master Checklist

This checklist is designed as a **deliberate practice curriculum**. Building ForgePay is not about speed or completing items as quickly as possible. It is about **deep comprehension of architectural mechanics, security principles, and distributed state management**.

For every phase:
1. **Read the corresponding Vault Study Guides & Reference Links** before writing code.
2. **Execute the tasks** step by step. Write code manually to build muscle memory.
3. **Execute the Phase Verification Commands**. Never declare a phase complete until tests pass cleanly.

---

## Phase 0: Environment & Project Bootstrap

### 0.1 Pedagogical Focus & Study References
Understand Spring Boot auto-configuration, starter dependency selection, and dockerized local infrastructure setups.
- 📖 **Vault Study Guide**: [[spring-boot-comprehensive-guide#11-spring-boot-auto-configuration--starters|Spring Boot Auto-Configuration §11]], [[spring-boot-comprehensive-guide#1-why-spring-exists-and-why-spring-boot|Why Spring Boot §1]]
- 📚 **External Reading & Material**: *Spring Boot in Action* (Craig Walls), Docker Compose Specification Documentation

- [ ] **0.1.1** Generate a Spring Boot 3.3+ project with Java 21 using Spring Initializr or Maven CLI. Include starters:
  - `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-data-redis`, `spring-boot-starter-security`, `spring-boot-starter-aop`, `spring-boot-starter-actuator`, `spring-boot-starter-validation`, `flyway-core`, `flyway-database-postgresql`, `postgresql`, `lombok`.
- [ ] **0.1.2** Initialize Git repository and commit initial bootstrap files.
- [ ] **0.1.3** Create `docker-compose.yml` in project root with PostgreSQL 16 (`5432:5432`) and Redis 7 Alpine (`6379:6379`) services with health checks.
- [ ] **0.1.4** Generate 2048-bit RS256 RSA private and public key pairs (`app.key` and `app.pub`) using OpenSSL and place them in `src/main/resources/certs/`.

*Verification:* Execute `docker compose up -d` and `./mvnw spring-boot:run`. The application must start cleanly and expose `/actuator/health` with `status: UP`.

---

## Phase 1: Database Schema & Core Domain Entities

### 1.1 Pedagogical Focus & Study References
Master Flyway database migrations, JPA entity mappings, optimistic locking (`@Version`), and relational constraints.
- 📖 **Vault Study Guide**: [[spring-boot-comprehensive-guide#8-core-annotations-reference|Spring Annotations Reference §8]], [[spring-security#3-user-entity--repository|User Entity & Repositories §3]]
- 📚 **External Reading & Material**: *High-Performance Java Persistence* (Vlad Mihalcea), Local Workspace PDFs: `postgresql-17-A4.pdf`, `sql.pdf`

- [ ] **1.1.1** Create Flyway migration script `src/main/resources/db/migration/V1__init_schema.sql`:
  - Tables: `users`, `refresh_tokens`, `wallets`, `transactions`, `idempotency_records`.
  - Enforce explicit `UNIQUE` index on `idempotency_records(idempotency_key)`.
- [ ] **1.1.2** Implement JPA Entities: `User`, `Wallet`, `Transaction`, `IdempotencyRecord`, `RefreshToken`.
  - Include `@Version Long version` on `Wallet` to enforce optimistic locking.
- [ ] **1.1.3** Implement Spring Data JPA Repositories (`UserRepository`, `WalletRepository`, etc.).
- [ ] **1.1.4** Write Unit Test (`WalletTest.java`) verifying that updating an outdated version throws `OptimisticLockingFailureException`.

*Verification:* `./mvnw test -Dtest=WalletTest`. Flyway should run migrations automatically on startup without SQL errors.

---

## Phase 2: Security & RS256 JWT Authentication Pipeline

### 2.1 Pedagogical Focus & Study References
Internalize Spring Security's Servlet Filter Chain, asymmetric cryptography, custom JWT authentication filters, and Redis token revocation lists.
- 📖 **Vault Study Guide**: [[spring-security#1-how-spring-security-really-works|Spring Security Filter Chain §1]], [[spring-security#7-jwt-request-filter|JWT Filter §7]], [[spring-security#8-security-configuration|Security Config §8]], [[spring-security#14-rs256--asymmetric-key-signing|RS256 Signing §14]], [[spring-security#17-token-revocation-with-redis|Token Revocation §17]]
- 📚 **External Reading & Material**: *Spring Security in Action* (Laurentiu Spilca, Ch. 6–11), [RFC 7519 (JSON Web Token Standard)](https://datatracker.ietf.org/doc/html/rfc7519)

- [ ] **2.1.1** Create `RsaKeyProperties.java` using `@ConfigurationProperties(prefix = "rsa")` to bind RSA public and private keys from classpath resources.
- [ ] **2.2.2** Build `JwtTokenProvider.java`:
  - Method `generateAccessToken(UserDetails userDetails)` signing JWTs with RS256 Private Key (Encoding claims: `sub`, `roles`, `authorities`, `jti`, `exp`).
  - Method `validateToken(String token)` verifying signature with RS256 Public Key.
- [ ] **2.2.3** Implement `JwtAuthenticationFilter.java` extending `OncePerRequestFilter`:
  - Extract Bearer token from `Authorization` header.
  - Query Redis to verify token `jti` is not in blacklist (`blacklist:jti:<JTI>`).
  - Construct `UsernamePasswordAuthenticationToken` and set into `SecurityContextHolder`.
- [ ] **2.2.4** Implement `CustomAuthenticationEntryPoint` (HTTP 401 JSON) and `CustomAccessDeniedHandler` (HTTP 403 JSON).
- [ ] **2.2.5** Configure `SecurityConfig.java` bean `SecurityFilterChain`:
  - Disable stateful session creation (`SessionCreationPolicy.STATELESS`).
  - Permit `/api/v1/auth/**`, `/actuator/health`.
  - Enforce `.hasAuthority("wallet:charge")` on charge endpoints.
- [ ] **2.2.6** Create `AuthController.java` implementing `/register`, `/login`, `/refresh`, and `/logout`.
  - Logout extracts `jti` and saves a Redis key (`blacklist:jti:<JTI>`) with TTL equal to remaining JWT lifetime.

*Verification:* Write `SecurityAuthTest.java` using MockMvc:
- Register user -> Receive JWT.
- Query protected endpoint without JWT -> 401 Unauthorized.
- Logout -> Query protected endpoint using old JWT -> 401 Unauthorized (Blacklisted in Redis).

---

## Phase 3: Redis Distributed Idempotency Aspect (`@Idempotent`)

### 3.1 Pedagogical Focus & Study References
Master Spring AOP, distributed locking mechanics with Redis `SET NX PX`, atomic state transitions, and caching response replays.
- 📖 **Vault Study Guide**: [[redis-springboot-guide#3-part-i-idempotency|Redis Guide Part I: Idempotency §3]], [[idempotency|Idempotency Concept Note]], [[spring-boot-comprehensive-guide#9-aspect-oriented-programming-aop|Spring AOP §9]]
- 📚 **External Reading & Material**: *Designing Data-Intensive Applications* (Martin Kleppmann, Ch. 11), Local Workspace PDF: `redis.pdf`

- [ ] **3.1.1** Create custom annotation `@Idempotent`:
  - Parameters: `keyPrefix`, `expireSeconds` (default: 120), `headerName` (default: "Idempotency-Key").
- [ ] **3.1.2** Implement `IdempotencyAspect.java` using `@Around("@annotation(idempotent)")`:
  - Extract `Idempotency-Key` from HTTP Request header.
  - Query Redis `idempotency:<KEY>`.
  - If status `IN_PROGRESS` -> Throw `IdempotencyException("Concurrent request in flight")` (HTTP 409).
  - If status `COMPLETED` -> Return cached response body & HTTP status directly from Redis.
  - If key missing -> Execute Redis `SET key "IN_PROGRESS" NX PX expireMs`.
  - Proceed with controller execution.
  - On success -> Update Redis key to `COMPLETED` with JSON response body and persist `IdempotencyRecord` to DB.
  - On business exception -> Evict Redis key to allow client retry.
- [ ] **3.1.3** Annotate `POST /api/v1/wallets/topup` and `POST /api/v1/wallets/charge` with `@Idempotent`.

*Verification:* Send `POST /api/v1/wallets/charge` twice with identical `Idempotency-Key`. The second request must return the cached response instantly without executing `WalletService` logic twice.

---

## Phase 4: Redis Rate Limiting Aspect (`@RateLimit`)

### 4.1 Pedagogical Focus & Study References
Master atomic Redis Lua scripting, Sliding Window Log algorithms, Spring AOP proxy interception, and RFC rate limit HTTP headers.
- 📖 **Vault Study Guide**: [[redis-springboot-guide#4-part-ii-rate-limiting|Redis Guide Part II: Rate Limiting §4]], [[rate-limiting|Rate Limiting Overview]]
- 📚 **External Reading & Material**: *System Design Interview* (Alex Xu, Ch. 4: Rate Limiter), Redis Lua Scripting Documentation (`EVAL`)

- [ ] **4.1.1** Write Lua script `src/main/resources/scripts/rate_limiter.lua` for Sliding Window Log algorithm.
- [ ] **4.1.2** Create custom annotation `@RateLimit`:
  - Parameters: `limit` (default: 10), `windowSeconds` (default: 60), `keyType` (USER, IP, TIER).
- [ ] **4.1.3** Implement `RateLimitingAspect.java` using `@Around`:
  - Determine key based on user ID or tier.
  - Execute Redis Lua Script via `StringRedisTemplate.execute(...)`.
  - Append HTTP Response Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.
  - If script returns `0` -> Throw `RateLimitExceededException` -> `GlobalExceptionHandler` returns `HTTP 429 Too Many Requests`.
- [ ] **4.1.4** Annotate `POST /api/v1/gateway/metered-execute` with `@RateLimit`.

*Verification:* Fire HTTP requests in a loop exceeding tier quota. The N+1 request must return HTTP 429 with `X-RateLimit-Remaining: 0`.

---

## Phase 5: Caching & Eviction Layer

### 5.1 Pedagogical Focus & Study References
Master distributed caching topologies, Jackson JSON serialization, `@Cacheable` and `@CacheEvict` lifecycles, and anti-penetration null-value caching rules.
- 📖 **Vault Study Guide**: [[redis-springboot-guide#5-part-iii-caching|Redis Guide Part III: Caching §5]], [[caching|Caching Notes]]
- 📚 **External Reading & Material**: *High-Performance Java Persistence* (Vlad Mihalcea), Local Workspace PDF: `postgresql-17-A4.pdf`

- [ ] **5.1.1** Configure `RedisCacheManager` in `RedisConfig.java` with Jackson JSON serializer and null-value caching disabled.
- [ ] **5.1.2** Annotate `PlanService.getBillingPlans()` with `@Cacheable(value = "plans")`.
- [ ] **5.1.3** Annotate `WalletService.getWalletBalance(userId)` with `@Cacheable(value = "wallets", key = "#userId")`.
- [ ] **5.1.4** Annotate `WalletService.topUpWallet(...)` and `chargeWallet(...)` with `@CacheEvict(value = "wallets", key = "#userId")`.

*Verification:* Execute `GET /api/v1/wallets/balance`. Inspect Redis via `redis-cli keys "*"`. Verify `wallets::<userId>` exists. Execute top-up, verify key is evicted immediately.

---

## Phase 6: Multi-Stage Containerization & Monitoring

### 6.1 Pedagogical Focus & Study References
Master multi-stage Docker builds, Alpine distroless JRE runtimes, non-root container security, JVM RAM tuning, and Prometheus Actuator telemetry.
- 📖 **Vault Study Guide**: [[spring-security#21-security-hardening-checklist|Security Hardening Checklist §21]]
- 📚 **External Reading & Material**: *Docker Deep Dive* (Nigel Poulton), OWASP Container Security Cheat Sheet

- [ ] **6.1.1** Create multi-stage `Dockerfile` with JDK 21 builder stage and Alpine JRE non-root execution stage.
- [ ] **6.2.2** Update `docker-compose.yml` to include `forge-pay-app` container linked to Postgres and Redis containers.
- [ ] **6.2.3** Enable Spring Boot Actuator endpoints (`/actuator/health`, `/actuator/prometheus`).

*Verification:* Run `docker compose up --build`. Access `http://localhost:8080/actuator/health`. Status should be `UP`.

---

## Phase 7: Testcontainers & Concurrency Testing (The Crucible)

### 7.1 Pedagogical Focus & Study References
Master integration testing with real containerized dependencies (Testcontainers) and multi-threaded race condition verification (`ExecutorService` + `CountDownLatch`).
- 📖 **Vault Study Guide**: [[spring-security#19-testing-your-auth-system|Testing Auth System §19]], [[spring-boot-comprehensive-guide#9-aspect-oriented-programming-aop|Spring AOP §9]]
- 📚 **External Reading & Material**: *Practical Unit Testing with JUnit 5 and AssertJ* (Tomek Kaczanowski), Testcontainers Java Official Documentation

- [ ] **7.1.1** Create `BaseIntegrationTest.java` declaring `@Testcontainers` with PostgreSQL 16 and Redis 7 containers.
- [ ] **7.1.2** Implement `IdempotencyConcurrencyTest.java`:
  - Spin up 50 threads via `ExecutorService`.
  - Synchronize start via `CountDownLatch(1)`.
  - Fire 50 concurrent `POST /api/v1/wallets/charge` requests simultaneously with identical `Idempotency-Key`.
  - Assert: Exactly 1 thread executes; remaining 49 receive cached response or HTTP 409.
  - Assert: Wallet balance is debited **exactly once** in PostgreSQL.

*Verification:* Run `./mvnw verify`. All Testcontainers tests must pass 100%.

---

## Phase 8: Outbox Pattern Event Hook (EDA Base)

- 📖 **Vault Study Guide**: [[redis-springboot-guide#6-part-iv-putting-it-all-together|Redis Messaging §6]]
- [ ] **8.1** Create `OutboxEvent` JPA Entity and table `outbox_events`.
- [ ] **8.2** In `WalletService.chargeWallet()`, save a `WalletChargedEvent` to `outbox_events` in the **same DB transaction**.
- [ ] **8.3** Build `@Scheduled` worker reading pending outbox events and logging them.

---

## Phase 9: Advanced Extensibility Practice Modules

Complete these elective modules after mastering Phases 0–8:
- [ ] **9.1 Read/Write Replica Routing**: Implement custom `AbstractRoutingDataSource` separating Primary (writes) and Replica (reads).
  - 📖 **Vault Study Guide**: [[spring-boot-comprehensive-guide#12-configuration--profiles|Profiles & Configuration §12]]
- [ ] **9.2 Resilience4j Circuit Breaker**: Wrap payment client calls with `@CircuitBreaker` and `@Bulkhead`.
  - 📖 **Vault Study Guide**: [[spring-security#16-rate-limiting--brute-force-protection|Resilience §16]]
- [ ] **9.3 OpenTelemetry Distributed Tracing**: Export traces to Zipkin/Tempo propagating unified `traceId`.
  - 📖 **Vault Study Guide**: [[redis-springboot-guide#9-observability--monitoring|Observability §9]]
- [ ] **9.4 Webhook Engine**: Implement `WebhookDeliveryService` calculating HMAC-SHA256 signature with `@Retryable` backoff.
- [ ] **9.5 gRPC API Interface**: Implement `GrpcWalletService` on HTTP/2 port `9090`.
- [ ] **9.6 Redlock Lock Consensus**: Implement Redisson `RLock` for distributed Redis clusters.
- [ ] **9.7 Zero-Downtime DB Migrations**: Execute Flyway Expand-Contract schema refactoring.
