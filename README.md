[View completion certificate](https://www.theforage.com/completion-certificates/Sj7temL583QAYpHXD/E6McHJDKsQYh79moz_Sj7temL583QAYpHXD_694d2801f76d215bcf3b3a4b_1766844198237_completion_certificate.pdf)

# Midas Core

> A transaction-processing microservice from the JPMorgan Chase & Co. Advanced Software Engineering job simulation on Forage.

Think of Midas Core as a digital bank-transfer system: a transaction arrives through Kafka, the service validates it, requests an incentive, updates the sender and recipient records, and makes the resulting balance available through an HTTP endpoint.

```mermaid
flowchart LR
    P[Transaction producer] --> K[Kafka topic]
    K --> M[Midas Core]
    M <--> I[Incentive API]
    M <--> D[(H2 database)]
    C[API client] -->|GET /balance| M
```

## Core capabilities

| Area | Responsibility |
|---|---|
| Kafka | Consume and deserialize transaction messages from a configurable topic |
| Validation | Verify both users, a positive amount, and sufficient sender funds |
| Persistence | Model and update relational `UserRecord` data with Spring Data JPA and H2 |
| Integration | Call the external Incentive API with `RestTemplate` |
| REST | Return a user's balance as JSON through `GET /balance` |
| Verification | Exercise the workflow with Maven, JUnit, embedded Kafka, and debugger inspection |

## Documentation

| Guide | Contents |
|---|---|
| [Architecture and transaction flow](docs/architecture.md) | Components, boundaries, validation path, domain model, worked example, and complete system design |
| [Message and API contracts](docs/api-contracts.md) | Kafka payload, Incentive API, balance endpoint, ports, and verified field names |
| [Testing and local runbook](docs/testing-and-runbook.md) | Task suites, fixtures, embedded Kafka, commands, and repository structure |
| [What I learned](docs/what-i-learned.md) | New concepts and practical engineering lessons from the project |

## Technology stack

`Java 17` · `Spring Boot 3.2.5` · `Apache Kafka` · `Spring Data JPA` · `H2` · `Spring MVC` · `RestTemplate` · `Maven` · `JUnit 5`

## Transaction at a glance

```mermaid
flowchart TD
    A[Receive transaction] --> B[Deserialize JSON]
    B --> C[Load sender and recipient]
    C --> D{Valid?}
    D -- No --> X[Reject without balance changes]
    D -- Yes --> E[Request incentive]
    E --> F[Debit sender]
    F --> G[Credit recipient + incentive]
    G --> H[(Save both users)]
```

### Example result

| Stage | Rahul — ID 1 | Aman — ID 2 |
|---|---:|---:|
| Initial balance | ₹1,000 | ₹500 |
| Transfer | −₹256 | +₹256 |
| Incentive | — | +₹8 |
| Final balance | **₹744** | **₹764** |

The amount `256` is used because the bundled service returns `log₂(amount)` for whole-number powers of two. See the [complete walkthrough](docs/architecture.md#complete-application-example).

## Repository snapshot

> [!IMPORTANT]
> The current `flow` branch is the supplied task scaffold. It contains the application entry point, models, JPA repository and conduit, test fixtures and helpers, and the bundled Incentive API. The Kafka consumer, balance controller, incentive client, Maven dependencies, and runtime configuration described by the task architecture are not checked into this branch. The documentation distinguishes this intended completed flow from the code currently present.

<details>
<summary><strong>View complete system design</strong></summary>

```mermaid
flowchart TB
    subgraph Input[Transaction ingestion]
        TP[Test KafkaProducer or upstream producer]
        KT[(Configurable Kafka topic<br/>Embedded broker in tests on port 9092)]
        TP -->|Serialized Transaction| KT
    end

    subgraph Core[Midas Core Spring Boot service on port 33400]
        KC[Kafka consumer<br/>Deserialization boundary]
        VS[Transaction validation<br/>Workflow service]
        IC[RestTemplate<br/>Incentive client]
        DC[DatabaseConduit]
        UR[UserRepository<br/>Spring Data JPA]
        BC[Balance controller<br/>GET balance]
        BM[Balance response model]

        KC --> VS
        VS --> IC
        VS --> DC
        DC --> UR
        BC --> UR
        BC --> BM
    end

    subgraph Data[Persistence]
        H2[(H2 SQL database<br/>USER_RECORD table)]
    end

    subgraph External[External integration]
        IA[Transaction Incentive API on port 33433<br/>POST incentive]
    end

    subgraph Query[Balance query]
        Client[HTTP client or BalanceQuerier]
    end

    KT -->|Transaction event| KC
    IC -->|Transaction JSON| IA
    IA -->|Incentive amount| IC
    UR -->|Find and save UserRecord| H2
    H2 -->|User data| UR
    Client -->|userId| BC
    BM -->|JSON amount| Client

    subgraph Verification[Test and verification]
        MVN[Maven and JUnit task suites]
        DBG[Debugger checkpoints]
        MVN -.-> KC
        MVN -.-> KT
        DBG -.-> VS
    end
```

</details>
