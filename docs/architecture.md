# Architecture and transaction flow

[← README](../README.md) · [Contracts](api-contracts.md) · [Testing](testing-and-runbook.md) · [Learning](what-i-learned.md)

## Boundaries

Each boundary owns one concern, keeping transport code separate from business logic and persistence.

```mermaid
flowchart LR
    K[Kafka consumer<br/>Message boundary] --> S[Transaction service<br/>Business rules]
    B[Balance controller<br/>HTTP boundary] --> S
    S --> R[UserRepository<br/>Data boundary]
    S --> I[Incentive client<br/>External boundary]
    R --> H[(H2)]
    I --> A[Incentive API]
```

```mermaid
flowchart TB
    K[Consumer] -->|Receives| E[Transaction event]
    S[Service] -->|Owns| L[Validation and balance logic]
    R[Repository] -->|Owns| D[Database access]
    I[Client] -->|Owns| X[External API call]
    C[Controller] -->|Owns| Q[HTTP balance query]
```

## Processing sequence

Transactions move asynchronously from the producer to Kafka before validation and persistence begin.

```mermaid
sequenceDiagram
    autonumber
    participant P as Producer
    participant K as Kafka
    participant M as Midas Core
    participant R as UserRepository
    participant I as Incentive API
    participant H as H2

    P->>K: Transaction
    K->>M: Deserialize event
    M->>R: Find both users
    R->>H: SELECT users
    H-->>M: User records
    M->>M: Validate
    alt Valid
        M->>I: POST incentive
        I-->>M: Incentive amount
        M->>M: Calculate balances
        M->>R: Save both users
        R->>H: UPDATE records
    else Invalid
        M->>M: Reject transaction
    end
```

## Validation gate

Every check must pass before the incentive is requested or either user balance is changed.

```mermaid
flowchart TD
    A[Transaction] --> B{Sender exists?}
    B -- No --> X[Reject]
    B -- Yes --> C{Recipient exists?}
    C -- No --> X
    C -- Yes --> D{Amount positive?}
    D -- No --> X
    D -- Yes --> E{Enough funds?}
    E -- No --> X
    E -- Yes --> F[Request incentive]
    F --> G[Debit sender]
    G --> H[Credit recipient and incentive]
    H --> I[(Save both users)]
```

## Domain model

The core models represent incoming transfers, stored users, balance responses, and repository access.

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
    Transaction ..> UserRecord : identifies
    Balance ..> UserRecord : represents balance
```

## Complete application example

The example uses an amount of `256` so the bundled Incentive API returns a visible reward of `8`.

```mermaid
sequenceDiagram
    participant K as Kafka
    participant M as Midas Core
    participant D as H2
    participant I as Incentive API
    participant C as Client

    Note over D: ID 1 Rahul 1000<br/>ID 2 Aman 500
    K->>M: senderId 1, recipientId 2, amount 256
    M->>D: Load users 1 and 2
    D-->>M: Rahul 1000 and Aman 500
    M->>M: All validation checks pass
    M->>I: POST incentive with amount 256
    I-->>M: amount 8
    M->>D: Save Rahul 744 and Aman 764
    C->>M: GET balance with userId 2
    M-->>C: amount 764
```

```text
Sender:    1000 - 256     = 744
Recipient:  500 + 256 + 8 = 764
```

> `200` receives no reward from the bundled API. `256` produces an incentive of `8`, so the diagram shows the complete path accurately.
