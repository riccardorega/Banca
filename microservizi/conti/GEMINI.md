# Conti Microservice - Architectural & Instructional Context

This document defines the architecture, design patterns, project layout, communication contracts, and operational guidelines for the **Conti (Accounts)** microservice within the **Banca Antifrode** distributed banking platform.

---

## 1. Microservice Overview & Domain Responsibilities

The **Conti** microservice is responsible for the foundational banking domain:
- **Account Management:** Creation, retrieval, updating, and status management of customer bank accounts.
- **Balance & Currency Operations:** Secure maintenance of customer balances and status checks (`Active`, `Suspended`, `Closed`).
- **Synchronous Validation Provider:** Exposes endpoints consumed by the `Transazioni` microservice via `Conti.ClientHttp` to validate account existence and active status before transaction routing (**Fail-fast**).
- **Asynchronous Settlement:** Listens for `TransferRequestedEvent` messages from Apache Kafka (produced after anti-fraud verification or for normal-tier transfers) and atomically updates sender and recipient account balances.

---

## 2. 5-Project Layered Architecture

The `Conti` solution strictly adheres to the mandated 5-project layout (placed directly in the microservice root without a `src` folder):

1. **`Conti.Api` (Presentation / Web API)**
   - ASP.NET Core Web API controllers.
   - Swagger / OpenAPI documentation.
   - Dependency injection wiring, middleware configuration, health checks, and logging.

2. **`Conti.Business` (Domain & Business Logic)**
   - Core domain models and business services (`AccountService`, balance calculation rules).
   - Kafka message handlers implementing `IMessageHandler` from `Utility.Kafka` (e.g., consuming `TransferRequestedEvent`).
   - Validation logic and domain exceptions.

3. **`Conti.ClientHttp` (Typed HTTP Client Library)**
   - Reusable client library packaged as a NuGet package (`ContiHttpClient`).
   - Provides typed, resilient HTTP wrappers (`IContiClient`) for inter-service communication (used primarily by `Transazioni`).
   - Handles Polly resilience policies (retries, circuit breakers) for synchronous calls.

4. **`Conti.Repository` (Data Access & Persistence)**
   - Entity Framework Core `ContiDbContext`.
   - Entity configurations, database migrations, and repository implementations (`AccountRepository`).
   - Implementation of the **Transactional Outbox Pattern** to ensure reliable publishing of domain events alongside database transactions.
   - Database initialization scripts (`*.sql`).

5. **`Conti.Shared` (DTOs, Contracts & Shared Models)**
   - Data Transfer Objects (DTOs), request/response contracts shared between API, ClientHttp, and consumers.
   - Shared domain enums (e.g., `AccountStatus`).

---

## 3. Communication Patterns

### Synchronous (HTTP / ClientHttp)
- `Transazioni` calls `Conti.Api` synchronously via `Conti.ClientHttp` to verify that both sender and recipient accounts exist and are active.
- If an account is missing, inactive, or the service is unreachable, `Transazioni` rejects the transaction immediately (**Fail-fast**).

### Asynchronous (Apache Kafka via `Utility.Kafka`)
- **Consumer:** Subscribes to transaction topics (e.g., `transfer-requested`).
- **Event Flow:** Upon receiving a `TransferRequestedEvent`, `Conti.Business` executes a database transaction to debit the sender and credit the recipient, ensuring atomicity and idempotency.

---

## 4. Building, Testing & Running

### Build & Compilation
```bash
dotnet restore
dotnet build --no-restore
```

### Running Tests
```bash
dotnet test
```

### Running the API
```bash
dotnet run --project Conti.Api
```

---

## 5. Development Standards

- **Strict Type Safety:** No compiler warning suppressions, unsafe casts, or hidden reflection tricks.
- **Asynchronous First:** Full use of `async`/`await` across EF Core queries, HTTP requests, and Kafka consumers.
- **Transactional Integrity:** All balance updates must occur within EF Core database transactions, utilizing the Outbox pattern for event publishing.
- **Code Documentation:** Public APIs and client interfaces must include XML documentation comments.
