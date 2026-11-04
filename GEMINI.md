# Banca Antifrode Project - Architectural & Instructional Context

This document serves as the primary instructional and architectural guide for the **Banca Antifrode** microservices project. It defines the technical stack, architectural patterns, 3-microservice domain layout, shared libraries, Kafka messaging, CI/CD pipelines, and deployment expectations.

---

## 1. Project Overview & Domain Structure

The **Banca Antifrode** system is a distributed microservices banking platform comprising **3 core microservices**:
1. **Conti (Accounts)** - Manages bank accounts, customer balances, and account statuses.
2. **Transazioni (Transactions)** - Processes financial transactions, validates account existence via synchronous HTTP calls, applies amount threshold rules, dispatches events via Kafka, and updates transaction states.
3. **Antifrode (Anti-Fraud)** - Consumes transaction events via Kafka, analyzes risk metrics, flags suspicious activities, and returns verification verdicts.

### Core Architecture & Deliverables
- **3 Autonomous Microservices:** `Conti`, `Transazioni`, `Antifrode`.
- **5-Project Layered Structure per Microservice:** Every microservice solution strictly follows the standard 5-project layout:
  1. `*.Api` (Web API / Presentation)
  2. `*.Business` (Domain & Business Logic, Kafka Message Handlers)
  3. `*.ClientHttp` (Typed HTTP Clients for synchronous inter-service communication)
  4. `*.Repository` (EF Core Data Access, DbContext, Models, Transactional Outbox)
  5. `*.Shared` (DTOs, Contracts, Shared DTOs)
- **Shared Infrastructure Libraries:**
  - `Utility.Kafka` (Shared NuGet package for Kafka producers, consumers, and administrator clients).
- **Communication Patterns:**
  - *Synchronous:* Typed HTTP Clients (`ClientHttp` libraries distributed as NuGet packages) used for account existence validation (Fail-fast).
  - *Asynchronous:* Apache Kafka event streaming using Transactional Outbox pattern for anti-fraud routing and account balance settlement.
- **Data Persistence:** Entity Framework Core with SQL initialization scripts (`*.sql`).
- **Containerization & Deployment:** Docker Compose orchestration referencing images stored in GitHub Container Registry (GHCR).
- **CI/CD (GitHub Actions):** Automated workflows for building and publishing NuGet packages and Docker images.

---

## 2. Detailed Operational Workflow

1. **Initial Validation (Synchronous):**
   - When a transaction request is received by `Transazioni.Api`, it verifies synchronously via `Conti.ClientHttp` that both sender and recipient accounts exist and are active. If any account is invalid or unreachable, the transaction is rejected immediately (**Fail-fast**).
2. **Threshold Filtering:**
   - **Normal Transaction ($\le$ threshold):** Bypasses anti-fraud analysis. `Transazioni` immediately emits a `TransferRequestedEvent` to Kafka.
   - **Suspicious Transaction ($>$ threshold):** Marked as `InSospesoAntifrode` and an event is published to Kafka for the `Antifrode` service.
3. **Anti-Fraud Analysis & Verdict:**
   - `Antifrode` consumes the event, evaluates risk, and publishes the verdict back via Kafka.
   - If **Fraud** $\rightarrow$ Transaction status is set to `RifiutataFrode` (no funds moved, zero refunds).
   - If **Safe** $\rightarrow$ Emits a `TransferRequestedEvent` to Kafka.
4. **Asynchronous Money Transfer:**
   - The `Conti` microservice consumes the `TransferRequestedEvent` via `Utility.Kafka`, updates sender and recipient balances in its local database, and `Transazioni` marks the transaction as `Approvata`.

---

## 3. Shared Libraries & NuGet Packaging

- **`Utility.Kafka` Library:** Reusable .NET library providing robust abstractions (`IProducerClient`, `IConsumerClient`, `IMessageHandler`) wrapped into clean dependency injection extensions (`KafkaExtensions`).
- **ClientHttp Libraries:** Each microservice provides a `*.ClientHttp` project packaged as a NuGet package for type-safe inter-service HTTP communication.
- **NuGet Configuration (`NuGet.config`):** Configured to pull packages from GitHub Package Registry or local feeds.

---

## 4. Development Standards & Conventions

- **Strict Type Safety:** No compiler warning suppressions or type-casting workarounds.
- **Transactional Outbox Pattern:** Use outbox tables in EF Core repositories to reliably publish Kafka messages alongside database transactions.
- **Asynchronous Flow:** Full utilization of `async`/`await` across EF Core, HTTP calls, and Kafka event loops.
- **API Testing:** Endpoints documented and testable via Swagger UI and Postman collections (no frontend required).
