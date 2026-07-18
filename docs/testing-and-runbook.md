# Testing and local runbook

[← README](../README.md) · [Architecture](architecture.md) · [Contracts](api-contracts.md) · [Learning](what-i-learned.md)

## Task journey

```mermaid
flowchart LR
    T1[Task 1<br/>Boot] --> T2[Task 2<br/>Kafka]
    T2 --> T3[Task 3<br/>Persistence]
    T3 --> T4[Task 4<br/>Incentives]
    T4 --> T5[Task 5<br/>Balance API]
```

```mermaid
flowchart TB
    T1[TaskOneTests] --> O1[Verify application context]
    T2[TaskTwoTests] --> O2[Inspect consumed events]
    T3[TaskThreeTests] --> O3[Inspect Waldorf balance]
    T4[TaskFourTests] --> O4[Inspect Wilbur balance]
    T5[TaskFiveTests] --> O5[Query balance JSON]
```

## Embedded Kafka flow

```mermaid
sequenceDiagram
    participant J as JUnit task
    participant F as FileLoader
    participant U as UserPopulator
    participant P as KafkaProducer
    participant K as Embedded Kafka
    participant M as Midas Core

    J->>F: Load user fixture
    F-->>U: User lines
    U->>M: Save users
    J->>F: Load transactions
    F-->>P: Transaction lines
    P->>K: Send Transaction objects
    K->>M: Deliver events
    J->>M: Inspect or query result
```

> Tasks 2–4 stay alive for debugger inspection. Stop them after recording the requested state.

## Run

```bash
# Terminal 1
java -jar services/transaction-incentive-api.jar

# Terminal 2 — select one task
./mvnw test -Dtest=TaskOneTests
./mvnw test -Dtest=TaskTwoTests
./mvnw test -Dtest=TaskThreeTests
./mvnw test -Dtest=TaskFourTests
./mvnw test -Dtest=TaskFiveTests
```

```mermaid
flowchart LR
    J[Start Incentive API] --> M[Run Maven task]
    M --> K[Embedded Kafka starts]
    K --> V[Verify output or debugger state]
```

> [!NOTE]
> The current scaffold has an empty dependency list and configuration file. The complete suite needs Spring Web, Kafka, Data JPA, H2, test dependencies, and runtime settings.

## Repository map

```text
forage-midas/
├── docs/                  # Linked documentation
├── services/              # Incentive API JAR
├── src/main/
│   └── java/.../midascore
│       ├── component/     # Database conduit
│       ├── entity/        # UserRecord
│       ├── foundation/    # Transaction and Balance
│       └── repository/    # UserRepository
├── src/test/              # Five tasks, helpers, fixtures
├── application.yml
└── pom.xml
```
