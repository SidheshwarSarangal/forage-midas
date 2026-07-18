# Architecture and transaction flow

[← Back to README](../README.md) · [API contracts](api-contracts.md) · [Testing](testing-and-runbook.md) · [What I learned](what-i-learned.md)

## Architectural boundaries

```mermaid
flowchart LR
    K[Kafka consumer<br/>Message boundary] --> S[Transaction service<br/>Business rules]
    B[Balance controller<br/>HTTP boundary] --> S
    S --> R[UserRepository<br/>Persistence boundary]
    S --> I[Incentive client<br/>External-service boundary]
    R --> H[(H2)]
    I --> A[Incentive API]
```

| Layer | Owns | Does not own |
|---|---|---|
| Kafka consumer | Message receipt and deserialization | Balance calculations or SQL |
| Transaction service | Validation, incentive orchestration, and balance updates | Kafka configuration or HTTP routing |
| Repository | Loading and saving users | Business validation |
| Incentive client | External HTTP request/response | Database updates |
| Balance controller | Query parameters and JSON responses | Transaction processing |

This separation keeps transport, business rules, persistence, and integrations independently understandable and testable.

## End-to-end processing

```mermaid
sequenceDiagram
    autonumber
    participant P as Producer
    participant K as Kafka topic
    participant M as Midas Core
    participant R as UserRepository
    participant I as Incentive API :33433
    participant H as H2

    P->>K: Transaction(senderId, recipientId, amount)
    K->>M: Deserialize event
    M->>R: Find sender and recipient
    R->>H: SELECT users
    H-->>M: User records
    M->>M: Validate transaction

    alt valid
        M->>I: POST /incentive
        I-->>M: {"amount": incentive}
        M->>M: Calculate both balances
        M->>R: Save sender and recipient
        R->>H: UPDATE user records
    else invalid
        M->>M: Reject transaction without balance changes
    end
```

## Validation path

```mermaid
flowchart TD
    A[Transaction received] --> B{Sender exists?}
    B -- No --> X[Reject]
    B -- Yes --> C{Recipient exists?}
    C -- No --> X
    C -- Yes --> D{Amount > 0?}
    D -- No --> X
    D -- Yes --> E{Sender balance >= amount?}
    E -- No --> X
    E -- Yes --> F[Request incentive]
    F --> G[Debit sender]
    G --> H[Credit recipient + incentive]
    H --> I[(Persist both users)]
```

## Domain model

```mermaid
classDiagram
    class Transaction {
        long senderId
        long recipientId
        float amount
    }

    class UserRecord {
        long id
        String name
        float balance
        setBalance(float)
    }

    class Balance {
        float amount
    }

    class UserRepository {
        findById(long) UserRecord
        save(UserRecord)
    }

    class DatabaseConduit {
        save(UserRecord)
    }

    DatabaseConduit --> UserRepository
    UserRepository --> UserRecord
    Transaction ..> UserRecord : identifies users
    Balance ..> UserRecord : exposes balance
```

| `UserRecord` field | Type | Database rule |
|---|---:|---|
| `id` | `long` | Generated primary key |
| `name` | `String` | Not null |
| `balance` | `float` | Not null |

## Complete application example

### 1. Initial database

| ID | User | Balance |
|---:|---|---:|
| `1` | Rahul | ₹1,000 |
| `2` | Aman | ₹500 |

### 2. Kafka receives

```json
{
  "senderId": 1,
  "recipientId": 2,
  "amount": 256
}
```

The JSON is deserialized into a `Transaction`. The service loads both `UserRecord` objects and checks:

```text
Rahul exists                 ✓
Aman exists                  ✓
Amount is positive           ✓
Rahul has sufficient funds   ✓
```

### 3. Incentive API responds

```json
{
  "amount": 8
}
```

### 4. Balances are calculated

```text
Rahul: 1,000 − 256     = 744
Aman:    500 + 256 + 8 = 764
```

Both records are saved. A later request to `GET /balance?userId=2` returns:

```json
{
  "amount": 764
}
```

> The original ₹200 example would receive an incentive of `0` from the bundled API. `256` produces `8`, so this example demonstrates the complete incentive path accurately.

## Complete system design

```mermaid
flowchart TB
    subgraph Input[Transaction ingestion]
        TP[Test KafkaProducer / upstream producer]
        KT[(Configurable Kafka topic<br/>Embedded broker in tests :9092)]
        TP -->|serialized Transaction| KT
    end

    subgraph Core[Midas Core · Spring Boot :33400]
        KC[Kafka consumer<br/>deserialization boundary]
        VS[Transaction validation<br/>and workflow service]
        IC[RestTemplate<br/>incentive client]
        DC[DatabaseConduit]
        UR[UserRepository<br/>Spring Data JPA]
        BC[Balance controller<br/>GET /balance]
        BM[Balance response model]

        KC --> VS
        VS --> IC
        VS --> DC
        DC --> UR
        BC --> UR
        BC --> BM
    end

    subgraph Data[Persistence]
        H2[(H2 SQL database<br/>USER_RECORD)]
    end

    subgraph External[External integration]
        IA[Transaction Incentive API :33433<br/>POST /incentive]
    end

    subgraph Query[Balance query]
        Client[HTTP client / BalanceQuerier]
    end

    KT -->|Transaction event| KC
    IC -->|transaction JSON| IA
    IA -->|incentive amount| IC
    UR <-->|find / save UserRecord| H2
    Client -->|userId| BC
    BM -->|JSON amount| Client

    subgraph Verification[Test boundaries]
        MVN[Maven + JUnit task suites]
        DBG[Debugger checkpoints]
        MVN -. boots and drives .-> Core
        MVN -. provisions .-> KT
        DBG -. inspects events and balances .-> VS
    end
```
