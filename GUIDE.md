# Explicit Kernel & Project Structure Guide (Framework-Agnostic)

This document outlines the architectural standards, directory structure, and naming conventions for the **Explicit Kernel** library and the **Applications** built using it. This approach uses **pure Kotlin** concepts to define boundaries, making it adaptable to any framework (Spring, Ktor, etc.) or a pure manual setup.

## 1\. Core Philosophy

The goal is to enforce a clear separation of concerns using a **Command-Driven** approach while avoiding specific framework dependencies.

* **Domain:** Pure business logic and state.
* **Application:** Execution flow (Commands & Handlers).
* **Infrastructure:** Implementation details (Persistence, External APIs).
* **Web/Presentation:** Communication boundary (HTTP, CLI, Messaging).

-----

## 2\. The Kernel Library Structure

This library contains only the reusable building blocks (interfaces and base classes). It is the source of truth for the architecture.

### Root Package: `com.melsardes.libraries.structuskotlin`

#### `domain/`

*Contains the base classes for business modeling.*

* **`AggregateRoot`**: The base class for entities that manage their own events.
* **`Entity`**: Base identity-based object.
* **`ValueObject`**: Marker for attribute-based objects.
* **`events/`**: Contains `DomainEvent` interface and `BaseEvent`.

#### `application/`

*Contains the interfaces for the execution model.*

* **`commands/`**:
    * `Command`: Marker interface for write-operations.
    * `CommandHandler`: Interface for logic that processes a command.
    * `CommandBus`: Interface for dispatching commands.
* **`queries/`** (Optional):
    * `Query` & `QueryHandler` for read-only operations.

-----

## 3\. The Concrete Application Structure

When implementing a specific application (microservice, module, etc.), use the following structure.

### General Rules

1.  **No "Ports" or "Adapters"**: Do not use these package names.
2.  **Implementation Suffix**: Implementations of interfaces must end in `Impl` (e.g., `UserRepositoryImpl`).
3.  **Feature-First**: Organize top-level packages by functional module (e.g., `billing`, `auth`, `catalog`), not by technical layer.

### Folder Layout Template

```text
src/main/kotlin/com/company/project/
│
├── feature_name/               # e.g., "user" or "checkout"
│   │
│   ├── domain/                 # PURE KOTLIN (Depends only on Kernel Domain)
│   │   ├── User.kt             # Extends AggregateRoot
│   │   ├── UserEvent.kt        # Extends BaseEvent
│   │   └── UserRepository.kt   # Interface only! (The "Contract")
│   │
│   ├── application/            # LOGIC FLOW (Depends on Kernel Application)
│   │   ├── RegisterUserCmd.kt  # Data class (Command)
│   │   └── RegisterUserHandler.kt # Implementation of CommandHandler (The logic)
│   │
│   ├── infrastructure/         # DETAILS (Database, File System, API Clients)
│   │   └── persistence/
│   │       ├── UserPersistenceModel.kt # The specific database schema/table representation
│   │       └── UserRepositoryImpl.kt   # Implementation of domain/UserRepository
│   │
│   └── web/                    # HTTP, CLI, or Messaging Interface (The boundary)
│       ├── RegisterRequestDto.kt # Data Transfer Object (DTO)
│       └── UserController.kt     # HTTP Controller Class
│
└── shared/                     # Shared components unique to this application
    └── wiring/                 # Dependency Setup/IoC Container Configuration
```

-----

## 4\. Layer Responsibilities & Dependency Rules

### A. The Domain Layer (Core)

* **Contains:** Aggregates, Entities, Value Objects, Domain Events, and **Repository Interfaces**.
* **Rules:**
    * MUST NOT depend on any framework, database, or I/O library.
    * MUST NOT depend on `application`, `web`, or `infrastructure`.
    * **Repository Interfaces** live here, defining methods like `suspend fun save(user: User)`.

### B. The Application Layer (Use Cases)

* **Contains:** `Command` data classes and `CommandHandler` implementations (logic).
* **Rules:**
    * Orchestrates the flow: Load Aggregate $\rightarrow$ Call Domain Method $\rightarrow$ Save Aggregate.
    * Calls the **Repository Interfaces** defined in the `domain` layer.
    * Does NOT contain HTTP/JSON parsing or SQL queries.

### C. The Infrastructure Layer (Implementation)

* **Contains:** Concrete implementations of **Interfaces** defined in the Domain and Application layers (e.g., `UserRepositoryImpl`).
* **Rules:**
    * This layer holds all framework-specific code (e.g., DB clients, HTTP libraries).
    * It implements the `UserRepository` interface and uses **mappers** to convert its `Persistence Models` into the clean `Domain Entities`.

### D. The Web (or Presentation) Layer (Boundary)

* **Contains:** The entry point classes (e.g., HTTP Controller, Message Queue Listener).
* **Rules:**
    * Accepts input (HTTP Request) $\rightarrow$ Creates Command $\rightarrow$ Dispatches Command via the **Command Bus**.
    * Never calls the Repository directly for write operations.

### The Dependency Check 🚦

The core rule of Explicit Architecture is the **Inward Dependency Rule**: Layers can only know about layers beneath them.

1.  **`web`** imports **`application`** (to dispatch commands)
2.  **`infrastructure`** imports **`domain`** (to fulfill contracts)
3.  **`application`** imports **`domain`** (to use Entities and Repo contracts)
4.  **`domain`** imports **`NOTHING`** (except standard Kotlin or your Kernel)

If you find a file in the `domain` package importing code from `infrastructure` or `web`, the architectural boundary has been violated.