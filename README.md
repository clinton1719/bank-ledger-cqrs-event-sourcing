# Bank Ledger System: CQRS & Event Sourcing Study

## 🎯 Project Purpose
This project is a backend-intensive study of the **CQRS (Command Query Responsibility Segregation)** and **Event Sourcing** patterns. It simulates a high-scale bank ledger where the primary goal is auditability, compliance, and strict data integrity.

## 🏗️ System Architecture

### 1. Command Side (Write Path)
* **Entry Point:** AWS Application Load Balancer (ALB).
* **Service:** Spring Boot (Java 21) on AWS ECS.
* **Responsibilities:** * Validating business rules (e.g., "Does the account have enough money?").
    * Rehydrating state from the Event Store.
    * Enforcing sequential consistency using Versioning/Optimistic Locking.
* **Source of Truth:** **DynamoDB** (Event Store).
    * *Storage Pattern:* Append-only log of events. Updates/Deletes are strictly forbidden.

### 2. The Bridge (Asynchronous Pipe)
* **Mechanism:** DynamoDB Streams.
* **Processor:** AWS Lambda (The Projectionist).
* **Responsibility:** Transports events from the Event Store and "folds" them into the Read Model.

### 3. Query Side (Read Path)
* **Entry Point:** AWS API Gateway.
* **Service:** AWS Lambda (Query Service).
* **Read Model:** **AWS RDS (PostgreSQL)**.
    * *Storage Pattern:* Denormalized, flat tables optimized for rapid retrieval without complex joins or math.

---

## 🛠️ Technical Constraints for AI Implementation

> [!IMPORTANT]
> To maintain the integrity of the Event Sourcing pattern, follow these rules strictly:

* **Immutability:** Events are facts. Once written to DynamoDB, they are never modified.
* **Optimistic Locking:** Every write to DynamoDB must include a `ExpectedVersion == CurrentVersion` check to prevent race conditions.
* **Idempotency:** Every incoming command must carry a `client_request_id`. The Command Service must check for duplicates before processing.
* **Eventual Consistency:** Acknowledge that the RDS Read Model may lag behind the DynamoDB Event Store. The system is designed for **High Availability** over Immediate Consistency.

---

## 📊 Data Modeling

### Event Store (DynamoDB)
| Attribute | Role | Description |
| :--- | :--- | :--- |
| `account_id` | **Partition Key** | Groups all events for a specific bank account. |
| `sequence_id` | **Sort Key** | Monotonically increasing integer (1, 2, 3...) for ordering. |
| `event_type` | Attribute | e.g., `MONEY_DEPOSITED`, `ACCOUNT_FROZEN`. |
| `payload` | Attribute | JSON blob containing transaction details. |
| `idempotency_key`| **GSI** | Used to ensure a command is only processed once. |

### Read Model (RDS)
| Column | Role | Description |
| :--- | :--- | :--- |
| `account_id` | Primary Key | Unique identifier. |
| `current_balance`| Value | Calculated sum of all previous events. |
| `last_sequence` | Metadata | Tracks the last event ID processed by the projectionist. |

---

## 📡 Observability (OpenTelemetry)
* **Tracing:** Use OTel `trace_id` to correlate a single transaction from the ALB -> ECS -> DynamoDB -> Lambda -> RDS.
* **Metrics:** Track **"Projection Lag"** (the time gap between DynamoDB write and RDS update).
* **Infrastructure:** ADOT (AWS Distro for OpenTelemetry) Collector running as a sidecar in ECS.

---

## 🤖 AI Context Instructions
1. **Focus:** Backend logic, data integrity, and AWS integration.
2. **Coding Style:** Clean architecture. Separate the **Domain Aggregate** (the logic) from the **Infrastructure** (DynamoDB/RDS mappers).
3. **Validation:** Use Spring Boot Validation for incoming DTOs.
4. **Error Handling:** Use custom exceptions (e.g., `InsufficientFundsException`) that map to appropriate HTTP status codes.