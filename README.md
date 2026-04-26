# payment-platform

A production-grade Maven multi-module monorepo for real-time payment processing with integrated fraud detection.

---

## Architecture overview

```
Client → LB / Gateway → Payment Service ──► Main Ledger DB (PostgreSQL)
                              │
                              ▼
                      Fraud Risk Service
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                GeoLite2    Redis     Sift API
               (IP/VPN)  (Velocity)  (ML Score)
```

---

## Modules

| Module | Port | Description |
|---|---|---|
| `payment-service` | `8080` | Payment order processing, card vault integration, outbox pattern |
| `fraud-risk-service` | `8082` | Real-time fraud evaluation — IP, device fingerprint, velocity, Sift Science |

---

## Tech stack

| Layer | Technology |
|---|---|
| Language | Java 21 (virtual threads) |
| Framework | Spring Boot 3.4.4 |
| Persistence | Spring Data JPA + PostgreSQL 15 |
| Caching / Velocity | Redis 7 |
| Reactive HTTP client | Spring WebFlux (WebClient) |
| Resilience | Resilience4j circuit breaker |
| IP Intelligence | MaxMind GeoLite2-City + GeoLite2-ASN |
| ML Fraud Scoring | Sift Science API |
| Object mapping | MapStruct 1.5.5 |
| Boilerplate reduction | Lombok 1.18.36 |
| JSONB support | Hypersistence Utils 3.7.3 |
| Observability | Micrometer + Datadog |
| Build | Maven 3.9 (multi-module) |

---

## Prerequisites

| Tool | Version |
|---|---|
| Java (Temurin recommended) | 21 LTS |
| Maven | 3.9+ |
| PostgreSQL | 15+ |
| Redis | 7+ |
| Git | Any recent version |

> **Important:** Java 21 LTS is required. JDK 22+ and JDK 26 are not supported by
> Spring Boot 3.4.x, Lombok, or MapStruct at this time.

---

## Project structure

```
payment-platform/
├── pom.xml                          ← root aggregator pom (packaging=pom)
├── .gitignore
├── README.md
│
├── payment-service/
│   ├── pom.xml
│   └── src/main/java/com/example/payment/
│       ├── controller/              PaymentController, CardVaultController
│       ├── service/impl/            PaymentServiceImpl
│       ├── client/                  FraudRiskClient, FraudRiskClientFallback
│       ├── entity/                  ledger/ (Merchant, PaymentOrder, PaymentEvent, OutboxEvent)
│       │                            vault/  (CardVault)
│       ├── repository/
│       ├── dto/                     request/ + response/
│       ├── enums/                   PaymentStatus, FraudDecision, RiskLevel …
│       ├── mapper/                  PaymentMapper (MapStruct)
│       ├── config/                  WebClientConfig, DataSourceConfig
│       └── exception/               GlobalExceptionHandler
│
└── fraud-risk-service/
    ├── pom.xml
    └── src/main/java/com/example/fraud/
        ├── controller/              FraudRiskController
        ├── service/impl/            FraudRiskServiceImpl
        ├── evaluator/               IpRiskEvaluator, DeviceFingerprintEvaluator, VelocityEvaluator
        ├── geo/                     GeoIpService, GeoIpResult, GeoRiskResult
        ├── sift/                    SiftClient, SiftScoreResult, SiftEventBuilder
        ├── entity/                  FraudEvaluation, VelocityRecord
        ├── repository/
        ├── dto/                     request/ + response/
        ├── enums/                   RiskLevel, FraudDecision
        └── config/                  FraudRiskConfig, RedisConfig
```

---

## Getting started

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/payment-platform.git
cd payment-platform
```

### 2. Set up PostgreSQL databases

```sql
-- Main Ledger DB
CREATE DATABASE ledger_db;

-- Fraud DB
CREATE DATABASE fraud_db;
```

```bash
# Apply DDL scripts
psql -U postgres -d ledger_db  -f payment-service/src/main/resources/db/schema.sql
psql -U postgres -d fraud_db   -f fraud-risk-service/src/main/resources/db/schema.sql
```

### 3. Set up Redis

```bash
# macOS
brew install redis && brew services start redis

# Linux
sudo apt install redis-server && sudo systemctl start redis

# Docker (quickest)
docker run -d -p 6379:6379 redis:7-alpine
```

### 4. Add GeoLite2 database files

Download from [MaxMind](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data):

```bash
cp GeoLite2-City.mmdb fraud-risk-service/src/main/resources/geo/
cp GeoLite2-ASN.mmdb  fraud-risk-service/src/main/resources/geo/
```

> These files are excluded from git (`.gitignore`) due to their size (~70MB each).

### 5. Configure environment variables

Create `application-local.yml` in each service (also excluded from git):

**`payment-service/src/main/resources/application-local.yml`**

```yaml
spring:
  datasource:
    url:      jdbc:postgresql://localhost:5432/ledger_db
    username: postgres
    password: your_password
  data:
    redis:
      host: localhost
      port: 6379

fraud:
  service:
    base-url: http://localhost:8082
```

**`fraud-risk-service/src/main/resources/application-local.yml`**

```yaml
spring:
  datasource:
    url:      jdbc:postgresql://localhost:5432/fraud_db
    username: postgres
    password: your_password
  data:
    redis:
      host: localhost
      port: 6379

sift:
  api-key:  YOUR_SIFT_API_KEY
  enabled:  false   # set true when you have a Sift account

geo:
  ip:
    city-db-path: classpath:geo/GeoLite2-City.mmdb
    asn-db-path:  classpath:geo/GeoLite2-ASN.mmdb
```

### 6. Build

```bash
# Build all modules from root
mvn clean install -DskipTests

# Build a single module
mvn clean install -pl payment-service     -DskipTests
mvn clean install -pl fraud-risk-service  -DskipTests
```

### 7. Run

```bash
# Terminal 1 — fraud-risk-service first (payment-service depends on it)
mvn spring-boot:run -pl fraud-risk-service

# Terminal 2 — payment-service
mvn spring-boot:run -pl payment-service
```

Or run the fat JARs directly:

```bash
java -jar payment-service/target/payment-service-1.0.0-SNAPSHOT-exec.jar
java -jar fraud-risk-service/target/fraud-risk-service-1.0.0-SNAPSHOT-exec.jar
```

---

## API reference

### Payment Service — `localhost:8080`

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/v1/payments` | Create a payment order (triggers fraud check) |
| `GET` | `/api/v1/payments/{paymentId}` | Get payment by ID |
| `POST` | `/api/v1/merchants` | Register a merchant |
| `POST` | `/api/v1/vault/tokenize` | Tokenize a card (stores in Vault DB) |

#### Create payment — example request

```bash
curl -X POST http://localhost:8080/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{
    "merchantId":         "550e8400-e29b-41d4-a716-446655440000",
    "customerId":         "660e8400-e29b-41d4-a716-446655440001",
    "amount":             15000,
    "currency":           "USD",
    "idempotencyKey":     "order-20240426-001",
    "paymentMethodToken": "tok_visa_test_4242"
  }'
```

#### Response

```json
{
  "paymentId":   "770e8400-e29b-41d4-a716-446655440002",
  "status":      "INITIATED",
  "amount":      15000,
  "currency":    "USD",
  "createdAt":   "2024-04-26T10:30:00"
}
```

### Fraud Risk Service — `localhost:8082`

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/v1/fraud/evaluate` | Evaluate fraud risk for a payment |

#### Fraud check — example request

```bash
curl -X POST http://localhost:8082/api/v1/fraud/evaluate \
  -H "Content-Type: application/json" \
  -d '{
    "paymentId":         "770e8400-e29b-41d4-a716-446655440002",
    "merchantId":        "550e8400-e29b-41d4-a716-446655440000",
    "customerId":        "660e8400-e29b-41d4-a716-446655440001",
    "amount":            150.00,
    "currency":          "USD",
    "ipAddress":         "203.0.113.5",
    "deviceFingerprint": "fp-abc123",
    "billingCountry":    "US",
    "paymentMethodToken":"tok_visa_test_4242"
  }'
```

#### Response

```json
{
  "evaluationId":   "880e8400-e29b-41d4-a716-446655440003",
  "paymentId":      "770e8400-e29b-41d4-a716-446655440002",
  "decision":       "APPROVE",
  "riskLevel":      "LOW",
  "riskScore":      12,
  "triggeredRules": [],
  "decisionReason": "No fraud signals detected",
  "evaluatedAt":    "2024-04-26T10:30:00.123"
}
```

---

## Fraud evaluation — risk signals

| Signal | Evaluator | Data source | Weight |
|---|---|---|---|
| VPN / Tor / Proxy detection | `GeoIpService` | MaxMind GeoLite2 | 40–60 pts |
| IP country vs billing mismatch | `IpRiskEvaluator` | MaxMind GeoLite2 | 20 pts |
| Customer transaction velocity 1h | `VelocityEvaluator` | Redis | 25 pts |
| Customer transaction velocity 24h | `VelocityEvaluator` | Redis | 15 pts |
| Device used by multiple customers | `VelocityEvaluator` | Redis SET | 35 pts |
| Card testing detection | `VelocityEvaluator` | Redis | 30 pts |
| Device previously declined | `DeviceFingerprintEvaluator` | PostgreSQL | 35 pts |
| ML fraud score | `SiftClient` | Sift Science API | up to 25 pts |

### Risk score → decision mapping

| Score | Risk level | Decision |
|---|---|---|
| 0–24 | LOW | APPROVE |
| 25–49 | MEDIUM | REVIEW |
| 50–74 | HIGH | DECLINE |
| 75–100 | CRITICAL | DECLINE |

---

## Databases

### Main Ledger DB

| Table | Description |
|---|---|
| `merchant` | Registered merchants |
| `payment_order` | Payment orders with status lifecycle |
| `payment_event` | Immutable event log per payment |
| `outbox_event` | Transactional outbox for Kafka publishing |

### Vault DB (separate datasource)

| Table | Description |
|---|---|
| `card_vault` | Encrypted PAN storage, referenced by token only |

### Fraud DB

| Table | Description |
|---|---|
| `fraud_evaluation` | Immutable audit log of every fraud decision |
| `velocity_record` | Rolling transaction counters (backed by Redis) |

---

## Running with Docker (optional)

```bash
# Start all infrastructure dependencies
docker compose up -d

# docker-compose.yml included in repo
# starts: postgres, redis
```

```yaml
# docker-compose.yml
version: '3.9'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

---

## Useful Maven commands

```bash
# Build all
mvn clean install -DskipTests

# Run tests
mvn test

# Run tests for one module
mvn test -pl fraud-risk-service

# Dependency tree
mvn dependency:tree -pl payment-service

# Effective POM (shows resolved versions)
mvn help:effective-pom -pl payment-service

# Check for dependency updates
mvn versions:display-dependency-updates
```

---

## Contributing

```bash
# Create a feature branch
git checkout -b feat/your-feature-name

# Commit with conventional commits format
git commit -m "feat(fraud): add behavioral biometrics evaluator"
git commit -m "fix(payment): handle idempotency key collision"
git commit -m "chore(deps): bump mapstruct to 1.5.5.Final"

# Push and open a Pull Request
git push -u origin feat/your-feature-name
```

### Conventional commit prefixes

| Prefix | Use for |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `chore` | Build, deps, config |
| `docs` | Documentation |
| `test` | Adding or fixing tests |
| `refactor` | Code change with no behavior change |

---

## License

This project is private and proprietary.

---

## Author

Chidananda — Lead Software Engineer
