# 📄 Product Requirement Document (PRD): ForgePay
### Production Sandbox for Spring Boot Backend Engineering Mastery
*Version 2.5 — July 2026*

---

## 1. Executive Summary & Core Learning Purpose

### 1.1 The Engineering Problem
Most software engineers learning Spring Boot fall into a common trap: they build basic CRUD applications (Create, Read, Update, Delete) where controllers directly delegate to repositories, security is copy-pasted from internet tutorials without an understanding of the underlying Servlet Filter Chain, database access is unoptimized, and distributed state problems—such as race conditions, duplicate API requests, and unthrottled traffic spikes—are completely ignored.

When these engineers enter high-throughput production environments, their systems fail under load. Duplicate requests double-charge user credit accounts, unthrottled clients crash database connection pools, stale cached data causes financial inconsistencies, and unhandled JWT expiration edge cases expose security vulnerabilities.

### 1.2 The ForgePay Solution
**ForgePay** is a production-grade, high-throughput metered billing and transaction security engine built in **Java 21 and Spring Boot 3**. 

While ForgePay operates as a realistic multi-tenant API billing platform—where software developers register, generate API keys, purchase credit balances, and execute metered API transactions—its true purpose is to serve as a **rigorous training ground for internalizing 6 core backend engineering pillars**:

1. **Security Hardening & Asymmetric Cryptography**: RS256 JWT authentication, Refresh Token Rotation, Redis instant revocation lists, and granular RBAC.
   - 📖 **Vault Study Guide**: [[spring-security#1-how-spring-security-really-works|Spring Security Guide §1]], [[spring-security#14-rs256--asymmetric-key-signing|RS256 Signing §14]], [[spring-security#17-token-revocation-with-redis|Token Revocation §17]]
   - 📚 **Books & Resources**: *Spring Security in Action* (Laurentiu Spilca, Ch. 6–11), [RFC 7519 (JSON Web Token Specification)](https://datatracker.ietf.org/doc/html/rfc7519)
2. **Strict Distributed Idempotency**: Zero double-charge guarantees using custom `@Idempotent` Spring AOP aspect, Redis distributed locks (`SET NX PX`), response body caching replay, and PostgreSQL unique constraints.
   - 📖 **Vault Study Guide**: [[redis-springboot-guide#3-part-i-idempotency|Redis Guide Part I: Idempotency §3]], [[caching|Caching Notes]]
   - 📚 **Books & Resources**: *Designing Data-Intensive Applications* (Martin Kleppmann, Ch. 11), *Redis in Action* (Josiah L. Carlson), Local Workspace PDF: `redis.pdf`
3. **High-Performance Sliding Window Rate Limiting**: Tiered quota enforcement (`FREE`, `PRO`, `ENTERPRISE`) using custom `@RateLimit` Spring AOP aspect executing atomic Redis Lua scripts (Sliding Window Log).
   - 📖 **Vault Study Guide**: [[redis-springboot-guide#4-part-ii-rate-limiting|Redis Guide Part II: Rate Limiting §4]], [[rate-limiting|Rate Limiting Overview]]
   - 📚 **Books & Resources**: *System Design Interview – An Insider's Guide* (Alex Xu, Ch. 4: Rate Limiter), Redis Documentation on Lua Scripting (`EVAL`)
4. **Resilient Read-Layer Caching**: Distributed caching with Lettuce and Spring Data Redis, custom Jackson JSON serialization, `@Cacheable` and `@CacheEvict` lifecycle management, anti-penetration null-caching, and cache stampede protection.
   - 📖 **Vault Study Guide**: [[redis-springboot-guide#5-part-iii-caching|Redis Guide Part III: Caching §5]], [[spring-boot-comprehensive-guide#11-spring-boot-auto-configuration--starters|Spring Boot Auto-Config §11]]
   - 📚 **Books & Resources**: *High-Performance Java Persistence* (Vlad Mihalcea), Local Workspace PDF: `postgresql-17-A4.pdf`
5. **Production Containerization & Security**: Multi-stage non-root Docker images utilizing distroless/Alpine JRE 21 runtimes and orchestrating full stack container environments via Docker Compose.
   - 📖 **Vault Study Guide**: [[spring-security#21-security-hardening-checklist|Security Hardening Checklist §21]]
   - 📚 **Books & Resources**: *Docker Deep Dive* (Nigel Poulton), OWASP Docker Security Cheat Sheet
6. **Integration & Concurrency Testing Architecture**: JUnit 5, **Testcontainers** (spinning up ephemeral PostgreSQL and Redis instances for true integration testing), MockMvc security verification, and multi-threaded race-condition tests using `ExecutorService` and `CountDownLatch`.
   - 📖 **Vault Study Guide**: [[spring-security#19-testing-your-auth-system|Testing Auth System §19]], [[spring-boot-comprehensive-guide#9-aspect-oriented-programming-aop|Spring AOP §9]]
   - 📚 **Books & Resources**: *Practical Unit Testing with JUnit 5 and AssertJ* (Tomek Kaczanowski), Testcontainers Java Official Documentation

---

## 2. Target Users, Tiers & Granular RBAC Matrix

### 2.1 User Roles & Tier Classifications
ForgePay models a multi-tenant platform where users belong to specific operational roles and subscription tiers.

| Role | Operational Description | Default Tier | Rate Limit Quota |
|---|---|---|---|
| `ROLE_ADMIN` | System administrators possessing full operational, management, and security audit access across all accounts. | `ENTERPRISE` | 10,000 req / minute |
| `ROLE_MERCHANT` | Business account holders managing commercial billing, customer wallets, and API key generation. | `PRO` | 500 req / minute |
| `ROLE_DEVELOPER` | Standard API consumers integration testing and executing metered transactions against the gateway. | `FREE` | 30 req / minute |
| `ROLE_AUDITOR` | Read-only compliance officers inspecting transaction audit logs and security revocation records. | `FREE` | 60 req / minute |

### 2.2 Granular Authorities vs. Plain Roles
ForgePay implements **Granular Authorities (Permissions)**. Roles are collections of fine-grained authorities encoded into the RS256 JWT access token (`authorities: ["wallet:charge", "wallet:read"]`).

- 📖 **Vault Reference**: [[spring-security#11-proper-rbac--roles--permissions|Spring Security Guide §11: Proper RBAC]], [[spring-security#3-user-entity--repository|User Entity & Authorities §3]]
- 📚 **External Reading**: *Spring Security in Action* (Ch. 3: Managing Users, Ch. 5: Authorization rules)

```
AUTHORITIES MAPPING:
  - auth:refresh        -> Permission to exchange a valid refresh token for a new access token
  - wallet:read         -> Permission to view balance, transaction history, and wallet details
  - wallet:topup        -> Permission to execute balance top-ups (Requires Idempotency-Key)
  - wallet:charge       -> Permission to execute metered charges (Requires Idempotency-Key)
  - apikey:manage       -> Permission to generate, rotate, or revoke developer API keys
  - plan:read           -> Permission to query cached billing plan catalogs
  - admin:audit         -> Permission to query security revocation logs and Actuator metrics
```

---

## 3. Core Functional Requirements & Deep Endpoint Lifecycles

### 3.1 Authentication & Token Lifecycle Engine
- **User Registration (`POST /api/v1/auth/register`)**: BCrypt hashing (cost factor 12) + User + Wallet initialization.
  - 📖 **Vault Reference**: [[spring-security#4-password-encoding|Password Encoding §4]], [[spring-security#9-auth-controller--login-register--logout|Auth Controller §9]]
- **User Login (`POST /api/v1/auth/login`)**: Returns RS256 JWT Access Token (15 mins) & UUID Refresh Token (7 days).
  - 📖 **Vault Reference**: [[spring-security#6-jwt-fundamentals--hs256|JWT Fundamentals §6]], [[spring-security#14-rs256--asymmetric-key-signing|RS256 Signing §14]]
- **Token Rotation (`POST /api/v1/auth/refresh`)**: Strict refresh token rotation & family revocation on reuse attempt.
  - 📖 **Vault Reference**: [[spring-security#10-refresh-tokens|Refresh Tokens §10]]
- **Logout & Instant Revocation (`POST /api/v1/auth/logout`)**: Places JWT `jti` into Redis revocation blacklist (`blacklist:jti:<JTI>`).
  - 📖 **Vault Reference**: [[spring-security#17-token-revocation-with-redis|Token Revocation §17]], [[redis-springboot-guide#7-redis-key-naming-conventions|Redis Naming §7]]

### 3.2 Idempotent Wallet Operations Engine
- **Wallet Top-Up (`POST /api/v1/wallets/topup`)** & **Metered Charge (`POST /api/v1/wallets/charge`)**:
  - Enforces `Idempotency-Key: <UUID>` header via `@Idempotent` AOP aspect.
  - 📖 **Vault Reference**: [[redis-springboot-guide#3-part-i-idempotency|Redis Guide Part I: Idempotency §3]], [[idempotency|Idempotency Concept Note]]
  - 📚 **External Reference**: Local Workspace PDF: `backend.pdf` (Idempotent API Design)

### 3.3 Rate-Limited Metered Gateway Engine
- **Gateway Execution (`POST /api/v1/gateway/metered-execute`)**:
  - `@RateLimit` AOP aspect executing atomic Redis Lua script (Sliding Window Log). Returns `X-RateLimit-*` headers or HTTP 429.
  - 📖 **Vault Reference**: [[redis-springboot-guide#4-part-ii-rate-limiting|Redis Guide Part II: Rate Limiting §4]], [[rate-limiting|Rate Limiting Overview]]

### 3.4 Cached Plan & Balance Query Engine
- **Billing Plans (`GET /api/v1/plans`)** & **Wallet Balance (`GET /api/v1/wallets/balance`)**:
  - Uses `@Cacheable` and `@CacheEvict` with Lettuce CacheManager.
  - 📖 **Vault Reference**: [[redis-springboot-guide#5-part-iii-caching|Redis Guide Part III: Caching §5]], [[caching|Caching Concept Note]]

---

## 4. Non-Functional Requirements & Performance SLAs

### 4.1 Latency Targets & High Availability Fallbacks
- **Failure Mode Fallbacks**: Redis outage triggers fallback to PostgreSQL `UNIQUE(idempotency_key)` constraint and fail-open rate limiting.
  - 📖 **Vault Reference**: [[redis-springboot-guide#1-introduction--mental-models|Redis Trade-offs & Fallbacks §1]], [[redis-springboot-guide#11-common-pitfalls--anti-patterns|Redis Pitfalls §11]]
  - 📚 **External Reading**: *Designing Data-Intensive Applications* (Ch. 8: The Trouble with Distributed Systems)

---

## 5. Domain Entity Specification & Relational Integrity

### 5.1 Optimistic Locking on Wallets (`@Version`)
- 📖 **Vault Reference**: [[spring-boot-comprehensive-guide#8-core-annotations-reference|Spring Annotations Reference §8]]
- 📚 **External Reading**: Local Workspace PDF: `postgresql-17-A4.pdf` (Concurrency Control & Transactions)

---

## 6. Advanced Extensibility Roadmap (Beyond EDA)

- 📖 **Vault Reference Links**:
  1. **Event-Driven & Outbox**: [[redis-springboot-guide#6-part-iv-putting-it-all-together|Redis PubSub & Messaging §6]]
  2. **Read/Write Replica Routing**: [[spring-boot-comprehensive-guide#12-configuration--profiles|Spring Profiles & Configuration §12]]
  3. **Resilience & Circuit Breakers**: [[spring-security#16-rate-limiting--brute-force-protection|Resilience §16]]
  4. **Distributed Tracing**: [[redis-springboot-guide#9-observability--monitoring|Observability & Monitoring §9]]
