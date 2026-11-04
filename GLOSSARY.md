# GLOSSARY.md - Banca Antifrode Domain Dictionary

This document defines the core domain terminology, technical concepts, and architectural patterns used across the **Banca Antifrode** project to ensure clear communication and shared understanding.

---

## 1. Domain Entities & Microservices

- **Conti (Accounts):** The microservice responsible for managing bank accounts, account holders, IBANs, and balances.
- **Transazioni (Transactions):** The microservice responsible for recording, validating, routing, and finalizing financial transactions and transfers.
- **Antifrode (Anti-Fraud):** The microservice responsible for consuming transaction events, evaluating risk scores, and detecting fraudulent activities.
- **Conto (Account):** Entity representing a bank account with an IBAN, owner details, current balance, and status (Active, Suspended, Closed).
- **Transazione (Transaction):** Entity representing a financial movement between a sender account and a recipient account, along with an amount, timestamp, and status (Pending, Approved, FraudRejected).

---

## 2. Communication & Integration Patterns

- **Chiamata Sincrona (Synchronous HTTP Call):** Real-time HTTP request made by one microservice to another (e.g., `Transazioni` querying `Conti`) expecting an immediate response. Used strictly for initial account validation.
- **Fail-fast:** Design approach where an operation is terminated immediately upon encountering an error or invalid state (e.g., rejecting a transaction if account validation fails or `Conti` is unreachable).
- **Controllo Soglia (Amount Threshold Rule):** Preliminary rule evaluated within the `Transazioni` service to determine whether a transaction requires anti-fraud screening based on its monetary amount.
- **Transactional Outbox Pattern:** Architectural pattern ensuring reliable message publishing by writing business data and outgoing Kafka messages into the database within the same local transaction. A background worker subsequently dispatches the messages to Kafka.
- **Evento di Richiesta Trasferimento (Transfer Requested Event):** Asynchronous Kafka message published by `Transazioni` instructing `Conti` to execute the actual balance debit and credit.
- **Valutazione Preventiva (Evaluate-Before-Transfer / Pre-check):** Flow design where financial transactions are validated and cleared by anti-fraud *before* any money movement occurs, completely eliminating the need for complex refund or reversal logic.
