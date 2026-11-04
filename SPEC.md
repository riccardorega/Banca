# Specification: Banca Antifrode Microservices System

## Problem Statement

Banking systems require real-time processing of financial transactions while ensuring robust fraud detection and prevention. Without a distributed, decoupled microservices architecture, processing transactions and evaluating risk synchronously can lead to high latency, single points of failure, and tightly coupled systems that are difficult to scale. Furthermore, handling fraud detection without a pre-check mechanism introduces complex refund and reversal flows that degrade data consistency and operational efficiency.

## Solution

A distributed microservices-based banking anti-fraud platform (**Banca Antifrode**) consisting of three autonomous microservices (`Conti`, `Transazioni`, `Antifrode`) communicating via synchronous HTTP calls (for initial account validation) and asynchronous Apache Kafka event streams (for anti-fraud screening and balance settlement). The system implements an **Evaluate-Before-Transfer (Pre-check)** flow where financial movements occur only after risk evaluation, eliminating the need for complex refund logic, and utilizes the **Transactional Outbox Pattern** to ensure reliable message publishing.

## User Stories

1. As a bank customer, I want to create and manage my bank accounts (`Conti`), so that I can hold funds and execute financial operations.
2. As a bank customer, I want to initiate money transfers and payments (`Transazioni`), so that I can send funds to other accounts securely.
3. As a system operator, I want incoming transactions to be validated synchronously against account existence and status, so that invalid or closed accounts are rejected immediately (**Fail-fast**).
4. As a system operator, I want transactions exceeding a predefined amount threshold to be flagged as suspicious and routed to the anti-fraud service (`Antifrode`), so that high-risk transfers are scrutinized before execution.
5. As a risk analyst, I want suspicious transactions to be evaluated asynchronously via Kafka events, so that risk scoring does not block normal transaction processing.
6. As a system operator, I want approved transactions (normal or anti-fraud cleared) to trigger asynchronous balance settlement on recipient and sender accounts, so that funds are transferred reliably.
7. As a bank customer, I want fraudulent transactions to be rejected instantly without moving funds, so that no manual refund or reversal process is required.

## Implementation Decisions

- **Architecture:** 3 autonomous microservices (`Conti`, `Transazioni`, `Antifrode`), each adhering to the standard 5-project .NET structure (`*.Api`, `*.Business`, `*.ClientHttp`, `*.Repository`, `*.Shared`).
- **Synchronous Communication:** Typed HTTP Clients (`ClientHttp` libraries distributed as NuGet packages) used by `Transazioni` to query `Conti` for account existence and status validation.
- **Asynchronous Communication:** Apache Kafka utilizing the reusable `Utility.Kafka` shared library package.
- **Threshold Routing:** `Transazioni` evaluates transaction amounts against a configurable threshold to determine if anti-fraud screening is required.
- **Zero-Refund Flow:** Financial transfers occur strictly *after* successful validation and anti-fraud approval (Evaluate-Before-Transfer).
- **Reliability:** Implementation of the **Transactional Outbox Pattern** in EF Core repositories to ensure atomic database writes and Kafka message dispatching.
- **Data Persistence:** Entity Framework Core with explicit SQL initialization scripts (`*.sql`).

## Testing Decisions

- **Testing Philosophy:** Tests must focus on verifying external observable behavior rather than internal implementation details.
- **Unit Testing:** Business logic, threshold rules, and validation handlers tested in isolation using mocking frameworks.
- **Integration Testing:** End-to-end API testing via Swagger UI and Postman collections; Kafka messaging verified through local integration test containers or Docker Compose test rigs.

## Out of Scope

- User interface or frontend application development (API exposure via Swagger/Postman is sufficient).
- Multi-currency conversion logic.
- Complex machine learning models for anti-fraud (heuristic risk scoring and amount threshold rules are sufficient).
- Native Kubernetes deployment manifests (Docker Compose orchestration is the target deployment mechanism).

## Further Notes

- All shared components (`Utility.Kafka`, `ClientHttp` libraries) must be packaged and distributed via NuGet packages (local feed or GitHub Package Registry).
- CI/CD pipelines must be automated using GitHub Actions workflows (`nuget.yml`, `docker.yml`).
