# What I learned

[← README](../README.md) · [Architecture](architecture.md) · [Contracts](api-contracts.md) · [Testing](testing-and-runbook.md)

The project connected event-driven processing, relational persistence, REST integration, and testing in one practical workflow.

```mermaid
flowchart TB
    P((Project learning))
    P --> E[Event-driven systems]
    E --> K[Kafka topics]
    E --> S[Serialization]
    E --> C[Configurable consumers]
    P --> D[Data integrity]
    D --> V[Validation gates]
    D --> J[JPA repositories]
    D --> H[H2 testing]
    P --> I[Integration]
    I --> R[REST contracts]
    I --> RT[RestTemplate]
    P --> Q[Quality]
    Q --> EK[Embedded Kafka]
    Q --> M[Maven tests]
    Q --> DB[Debugger inspection]
```

## Workflow lessons

The order of operations protects data integrity and makes the transaction lifecycle easier to reason about.

```mermaid
flowchart LR
    A[Receive] --> B[Deserialize]
    B --> C[Validate]
    C --> D[Call dependency]
    D --> E[Calculate]
    E --> F[Persist]
    F --> G[Expose result]
```

```mermaid
flowchart TB
    C[Clean boundaries] --> K[Consumer<br/>Messaging only]
    C --> S[Service<br/>Business rules]
    C --> R[Repository<br/>Persistence]
    C --> I[Client<br/>External API]
    C --> B[Controller<br/>HTTP response]
```

## Key takeaways

| Learned | Why it matters |
|---|---|
| Validate before mutation | Prevents invalid or partial balance changes |
| Match contracts exactly | `recipientId` and `amount` must deserialize correctly |
| Externalize configuration | Topics and URLs can change by environment |
| Test real boundaries | Embedded Kafka verifies asynchronous ingestion |
| Debug asynchronous state | Events run outside the producer call stack |

## Production next steps

Moving from an exercise to a financial production service would require stronger consistency, resilience, security, and observability.

```mermaid
flowchart TB
    P[Production readiness]
    P --> M[Money<br/>BigDecimal or minor units]
    P --> A[Atomicity<br/>Database transaction]
    P --> I[Idempotency<br/>Transaction ID]
    P --> C[Concurrency<br/>Locking or versioning]
    P --> F[Failures<br/>Timeouts and retries]
    P --> O[Observability<br/>Logs, metrics, traces]
    P --> S[Security<br/>Authentication and data protection]
```
