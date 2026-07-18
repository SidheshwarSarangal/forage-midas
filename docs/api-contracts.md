# Message and API contracts

[← README](../README.md) · [Architecture](architecture.md) · [Testing](testing-and-runbook.md) · [Learning](what-i-learned.md)

## Contract flow

```mermaid
sequenceDiagram
    participant P as Producer
    participant K as Kafka
    participant M as Midas Core
    participant I as Incentive API
    participant C as Balance client

    P->>K: Transaction JSON
    K->>M: Transaction object
    M->>I: POST incentive
    I-->>M: amount
    C->>M: GET balance with userId
    M-->>C: amount
```

## Kafka message

```json
{
  "senderId": 1,
  "recipientId": 2,
  "amount": 256
}
```

```mermaid
classDiagram
    class Transaction {
        long senderId
        long recipientId
        float amount
    }
```

> The model uses `recipientId`, not `receiverId`.

## Incentive endpoint

```mermaid
flowchart LR
    M[Midas Core] -->|POST localhost 33433 incentive| I[Incentive API]
    I -->|JSON amount| M
```

```json
{
  "amount": 8
}
```

```mermaid
flowchart LR
    A[Amount] --> B{Whole-number<br/>power of two?}
    B -- Yes --> C[Incentive equals log2 amount]
    B -- No --> D[Incentive equals 0]
```

| Amount | Incentive |
|---:|---:|
| `128` | `7` |
| `200` | `0` |
| `256` | `8` |

> The response field is `amount`, not `incentive`.

## Balance endpoint

```mermaid
sequenceDiagram
    participant C as Client
    participant B as Balance controller
    participant R as UserRepository
    C->>B: GET balance with userId 2
    B->>R: Find user 2
    R-->>B: Balance 764
    B-->>C: JSON amount 764
```

```json
{
  "amount": 764
}
```

> `Balance` contains only `amount`.

## Ports

```mermaid
flowchart LR
    K[9092<br/>Embedded Kafka] --> M[33400<br/>Midas Core]
    M --> I[33433<br/>Incentive API]
```
