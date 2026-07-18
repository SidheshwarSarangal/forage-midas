[View completion certificate](https://www.theforage.com/completion-certificates/Sj7temL583QAYpHXD/E6McHJDKsQYh79moz_Sj7temL583QAYpHXD_694d2801f76d215bcf3b3a4b_1766844198237_completion_certificate.pdf)

# Midas Core

> A Kafka-driven bank-transfer microservice from the JPMorgan Chase & Co. Advanced Software Engineering job simulation on Forage.

```mermaid
flowchart LR
    P[Transaction producer] --> K[Kafka topic]
    K --> M[Midas Core]
    M --> V{Valid transaction?}
    V -- No --> X[Reject]
    V -- Yes --> I[Request incentive]
    I --> D[(Update H2 balances)]
    C[Balance client] -->|GET balance| M
    M -->|JSON amount| C
```

## What the system covers

```mermaid
flowchart TB
    M((Midas Core))
    M --> K[Kafka<br/>Consume and deserialize]
    M --> V[Validation<br/>Users, amount, funds]
    M --> J[JPA and H2<br/>Load and save users]
    M --> R[REST integration<br/>Request incentive]
    M --> B[Balance API<br/>Return JSON]
    M --> T[Verification<br/>Maven, JUnit, debugger]
```

## Documentation map

```mermaid
flowchart LR
    R[README] --> A[Architecture<br/>Flows and boundaries]
    R --> C[Contracts<br/>Kafka and REST]
    R --> T[Testing<br/>Tasks and runbook]
    R --> L[Learning<br/>Skills and takeaways]
```

| Open | Focus |
|---|---|
| [Architecture](docs/architecture.md) | Processing, validation, domain model, example |
| [API contracts](docs/api-contracts.md) | Payloads, endpoints, response fields, ports |
| [Testing](docs/testing-and-runbook.md) | Embedded Kafka, task suites, commands |
| [What I learned](docs/what-i-learned.md) | New concepts and production lessons |

## One transaction

```mermaid
sequenceDiagram
    participant K as Kafka
    participant M as Midas Core
    participant I as Incentive API
    participant D as H2
    participant C as Client

    Note over D: Rahul 1000<br/>Aman 500
    K->>M: senderId 1, recipientId 2, amount 256
    M->>M: Validate users, amount, and funds
    M->>I: POST incentive
    I-->>M: amount 8
    M->>D: Rahul 744, Aman 764
    C->>M: GET balance for user 2
    M-->>C: amount 764
```

```text
Rahul: 1000 - 256     = 744
Aman:   500 + 256 + 8 = 764
```

`256` is used because the bundled API awards `log₂(amount)` for whole-number powers of two. [See the verified example →](docs/architecture.md#complete-application-example)

## Stack

```mermaid
flowchart LR
    J[Java 17] --> S[Spring Boot 3.2.5]
    S --> K[Kafka]
    S --> P[Spring Data JPA]
    P --> H[H2]
    S --> W[Spring MVC]
    S --> RT[RestTemplate]
    S --> M[Maven and JUnit 5]
```

> [!IMPORTANT]
> The current `flow` branch is the task scaffold. It contains the models, repository, database conduit, tests, fixtures, and Incentive API JAR. The consumer, controller, incentive client, dependencies, and runtime configuration shown in the intended architecture are not checked into this branch.

<details>
<summary><strong>View complete system design</strong></summary>

```mermaid
flowchart TB
    subgraph Input[Transaction ingestion]
        TP[Test producer or upstream producer]
        KT[(Configurable Kafka topic<br/>Embedded broker on port 9092)]
        TP -->|Transaction| KT
    end

    subgraph Core[Midas Core on port 33400]
        KC[Kafka consumer]
        VS[Validation and workflow]
        IC[RestTemplate client]
        DC[DatabaseConduit]
        UR[UserRepository]
        BC[Balance controller]
        BM[Balance model]

        KC --> VS
        VS --> IC
        VS --> DC
        DC --> UR
        BC --> UR
        BC --> BM
    end

    subgraph Data[Persistence]
        H2[(H2 USER_RECORD)]
    end

    subgraph External[External service]
        IA[Incentive API on port 33433]
    end

    subgraph Query[Balance query]
        Client[HTTP client]
    end

    KT --> KC
    IC -->|POST incentive| IA
    IA -->|Incentive amount| IC
    UR -->|Find and save| H2
    H2 -->|User data| UR
    Client -->|GET balance with userId| BC
    BM -->|JSON amount| Client

    subgraph Verification[Verification]
        MVN[Maven and JUnit]
        DBG[Debugger]
        MVN -.-> KC
        MVN -.-> KT
        DBG -.-> VS
    end
```

</details>
