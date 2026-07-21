# 🏗️ Technical Design Document (TDD): ForgePay
### Architecture, Security Filters, Redis AOP Aspects, Testing & Advanced Extensibility
*Version 2.5 — July 2026*

---

## 1. Architectural Overview & Request Execution Lifecycle

ForgePay is engineered around a **Layered Domain Architecture** backed by Spring Boot 3 (Java 21), PostgreSQL 16, and Redis 7. 

- 📖 **Vault Study Guide**: [[spring-boot-comprehensive-guide#10-spring-mvc-request-lifecycle|Spring MVC Lifecycle §10]], [[spring-boot-comprehensive-guide#13-putting-it-all-together-a-mental-architecture-diagram|Architecture Diagram §13]]
- 📚 **External Reading**: *Spring in Action* (Craig Walls, Ch. 2: Developing Web Applications), Local Workspace PDF: `backend.pdf`

```
                  ┌─────────────────────────────────────────┐
                  │          Client HTTP Request            │
                  └────────────────────┬────────────────────┘
                                       │
                                       ▼
                  ┌─────────────────────────────────────────┐
                  │       Servlet Filter Chain              │
                  │  (CorsFilter -> JwtAuthFilter)          │
                  └────────────────────┬────────────────────┘
                                       │ (Authenticated SecurityContext)
                                       ▼
                  ┌─────────────────────────────────────────┐
                  │       Spring MVC Controller             │
                  └────────────────────┬────────────────────┘
                                       │
                ┌──────────────────────┴──────────────────────┐
                ▼                                             ▼
  ┌───────────────────────────┐                 ┌───────────────────────────┐
  │  @RateLimit Aspect (AOP)  │                 │  @Idempotent Aspect (AOP) │
  │   (Redis Lua Script)      │                 │   (Redis SET NX + DB)     │
  └─────────────┬─────────────┘                 └─────────────┬─────────────┘
                │ (Quota OK)                                  │ (Lock Acquired)
                └──────────────────────┬──────────────────────┘
                                       │
                                       ▼
                  ┌─────────────────────────────────────────┐
                  │            Service Layer                │
                  │   (@Cacheable / @CacheEvict enabled)    │
                  └────────────────────┬────────────────────┘
                                       │
                      ┌────────────────┴────────────────┐
                      ▼                                 ▼
         ┌───────────────────────────┐    ┌───────────────────────────┐
         │     PostgreSQL 16 DB      │    │       Redis 7 Cache       │
         │  (JPA / Flyway / Locks)   │    │  (Blacklist, Keys, Locks) │
         └───────────────────────────┘    └───────────────────────────┘
```

---

## 2. Spring Security Architecture & Cryptographic Plumbing

### 2.1 Why Asymmetric RS256 Signing Over Symmetric HS256?
- 📖 **Vault Reference**: [[spring-security#14-rs256--asymmetric-key-signing|Spring Security Guide §14: RS256 Signing]], [[spring-security#6-jwt-fundamentals--hs256|JWT Fundamentals §6]]
- 📚 **External Reading**: *Spring Security in Action* (Ch. 11: Authorization Servers and JWT), [RFC 7519 §4 (JWT Claims)](https://datatracker.ietf.org/doc/html/rfc7519#section-4)

```bash
# Key Generation Workflow
openssl genrsa -out app.key 2048
openssl rsa -in app.key -pubout -out app.pub
```

### 2.2 Servlet Filter Chain Execution Mechanics
- 📖 **Vault Reference**: [[spring-security#1-how-spring-security-really-works|Spring Security Guide §1: Servlet Filter Chain]], [[spring-security#7-jwt-request-filter|JWT Request Filter §7]], [[spring-security#8-security-configuration|Security Configuration §8]]
- 📚 **External Reading**: *Spring Security in Action* (Ch. 9: Implementing Filters)

---

## 3. Pillar II: Redis Distributed Idempotency Aspect (`@Idempotent`)

### 3.1 Distributed State Machine & Race Condition Protection
- 📖 **Vault Reference**: [[redis-springboot-guide#3-part-i-idempotency|Redis Guide Part I: Idempotency §3]], [[spring-boot-comprehensive-guide#9-aspect-oriented-programming-aop|Spring AOP §9]]
- 📚 **External Reading**: *Designing Data-Intensive Applications* (Martin Kleppmann, Ch. 11: Stream Processing - Idempotence), Local Workspace PDF: `redis.pdf`

```
Client Request (Header: Idempotency-Key = "uuid-1234")
  │
  ▼
@Idempotent Aspect Intercepts Method Execution
  │
  ├─► Redis SET "idempotency:uuid-1234" "IN_PROGRESS" NX PX 120000
  │     │
  │     ├─► Lock Acquisition FAILS (Key already exists):
  │     │     ├─► Query Redis status for key "idempotency:uuid-1234"
  │     │     ├─► If status == "IN_PROGRESS": Throw IdempotencyException("Request in flight") -> HTTP 409
  │     │     └─► If status == "COMPLETED": Return cached JSON Response & HTTP Status directly!
  │     │
  │     └─► Lock Acquisition SUCCEEDS (Key created atomically):
  │           ├─► Proceed with Controller / Service method execution
  │           ├─► On Success: Update Redis Key status to "COMPLETED" + JSON ResponseBody (TTL 24h)
  │           │              Persist IdempotencyRecord entity to PostgreSQL
  │           └─► On Business Exception: Evict Redis Key immediately to allow user retry
```

---

## 4. Pillar III: Redis Rate Limiting Aspect (Lua Scripting)

### 4.1 Atomic Lua Scripts & Sliding Window Log
- 📖 **Vault Reference**: [[redis-springboot-guide#4-part-ii-rate-limiting|Redis Guide Part II: Rate Limiting §4]], [[rate-limiting|Rate Limiting Overview]]
- 📚 **External Reading**: *System Design Interview* (Alex Xu, Ch. 4: Rate Limiter Algorithms), Redis Command Reference (`EVAL`, `ZREMRANGEBYSCORE`, `ZCARD`)

```lua
-- rate_limiter.lua
-- KEYS[1]: Rate limit Redis key (e.g. "rate_limit:user:123:endpoint")
-- ARGV[1]: Current Epoch Milliseconds
-- ARGV[2]: Window Duration in Milliseconds (e.g. 60000 for 1 minute)
-- ARGV[3]: Maximum Allowed Requests in Window

local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local clearBefore = now - window

-- 1. Remove timestamps outside the sliding window
redis.call('ZREMRANGEBYSCORE', key, '-inf', clearBefore)

-- 2. Count requests remaining in window
local currentRequests = redis.call('ZCARD', key)

if currentRequests < limit then
    -- 3. Add current timestamp to Sorted Set
    redis.call('ZADD', key, now, now)
    -- 4. Refresh key TTL
    redis.call('PEXPIRE', key, window)
    return {1, limit - currentRequests - 1, (now + window)} -- Allowed: 1 (True)
else
    return {0, 0, (now + window)} -- Allowed: 0 (False - Quota Exceeded)
end
```

---

## 5. Pillar IV: Caching Topology & Anti-Penetration Mechanics

### 5.1 Redis CacheManager Configuration
- 📖 **Vault Reference**: [[redis-springboot-guide#5-part-iii-caching|Redis Guide Part III: Caching §5]], [[caching|Caching Overview]]
- 📚 **External Reading**: *High-Performance Java Persistence* (Vlad Mihalcea), Local Workspace PDF: `postgresql-17-A4.pdf`

```java
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        ObjectMapper objectMapper = new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .activateDefaultTyping(LaissezFaireSubTypeValidator.instance, ObjectMapper.DefaultTyping.NON_FINAL);

        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(15))
                .disableCachingNullValues() // Anti-penetration configuration
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer(objectMapper)));

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(config)
                .withInitialCacheConfigurations(Map.of(
                    "plans", config.entryTtl(Duration.ofHours(1)),
                    "wallets", config.entryTtl(Duration.ofMinutes(5))
                ))
                .build();
    }
}
```

---

## 6. Pillar V: Multi-Stage Containerization Engineering

- 📖 **Vault Reference**: [[spring-security#21-security-hardening-checklist|Security Hardening Checklist §21]]
- 📚 **External Reading**: *Docker Deep Dive* (Nigel Poulton), Eclipse Temurin JDK 21 Alpine Container Best Practices

---

## 7. Pillar VI: Testcontainers & Concurrency Testing Methodology

### 7.1 Integration Testing with Real Containers & Multi-Threaded Harness
- 📖 **Vault Reference**: [[spring-security#19-testing-your-auth-system|Testing Auth System §19]], [[spring-boot-comprehensive-guide#9-aspect-oriented-programming-aop|Spring AOP §9]]
- 📚 **External Reading**: *Practical Unit Testing with JUnit 5 and AssertJ* (Tomek Kaczanowski), Testcontainers Java Official Guide

---

## 8. Advanced Extensibility Technical Blueprints (Beyond EDA)

- 📖 **Vault Reference Links**:
  1. **Replica Routing**: [[spring-boot-comprehensive-guide#12-configuration--profiles|Profiles & DataSources §12]]
  2. **Resilience4j**: [[spring-security#16-rate-limiting--brute-force-protection|Resilience §16]]
  3. **Tracing**: [[redis-springboot-guide#9-observability--monitoring|Observability §9]]
  4. **CDC & Debezium**: [[redis-springboot-guide#6-part-iv-putting-it-all-together|Redis PubSub & Streaming §6]]
