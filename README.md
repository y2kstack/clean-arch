Okay, let's dive deeper into Clean Architecture and how it applies to the Go TCP watchdog server we designed.

**The Core Idea of Clean Architecture (Uncle Bob Martin)**

Imagine a series of concentric circles, like an onion:

```
+-----------------------------------------------------+
|                Frameworks & Drivers                 |  <-- Outermost: Web, DB, UI, Devices, net pkg
|  +-------------------------------------------------+  |
|  |               Interface Adapters                |  |
|  |  (Controllers, Gateways, Presenters)          |  |  <-- Adapts data for use cases & external world
|  |  +-------------------------------------------+  |  |
|  |  |          Application Business Rules       |  |  |
|  |  |          (Use Cases)                      |  |  |  <-- Application-specific logic
|  |  |  +-------------------------------------+  |  |  |
|  |  |  |         Enterprise Business Rules   |  |  |  |
|  |  |  |         (Entities)                  |  |  |  |  <-- Core business objects & rules
|  |  |  +-------------------------------------+  |  |  |
|  |  +-------------------------------------------+  |  |
|  +-------------------------------------------------+  |
+-----------------------------------------------------+
```

**The Overarching Rule: The Dependency Rule**

*   **Source code dependencies can only point inwards.**
*   Nothing in an inner circle can know anything at all about something in an outer circle.
*   Specifically, code in an inner circle cannot refer to the name of something (function, class, variable, data structure) declared in an outer circle.

**Let's break down each layer in the context of our Go TCP Watchdog:**

**1. Entities (Domain Layer)**

*   **What it is:** This is the absolute core. It contains enterprise-wide business objects and the most general, high-level business rules. These objects are the least likely to change when something external (like the database, UI, or framework) changes. They should be pure and have no dependencies on other layers in your application.
*   **Our Example:**
    *   `domain/client.go`:
        *   `Client` struct: Represents the core concept of a connected client, its ID, status (`StatusActive`, `StatusUnhealthy`), last heartbeat time.
        *   Methods like `UpdateHeartbeat()`, `IsHealthy()`, `MarkUnhealthy()`: These are fundamental business rules about what it means for a client to be healthy or how its state changes.
    *   **Key Characteristics:**
        *   No imports from `usecase`, `interfaces`, or `cmd`.
        *   Could theoretically be used by a completely different application if that application also dealt with "clients" in a similar way.
        *   The `net.Conn` within `domain.Client` is a slight concession to practicality for this specific TCP server. In a purer form, the `Client` might only hold `RemoteAddr` string, and the connection management would be entirely outside. However, for a server that *must* interact with the connection (e.g., to close it), embedding it here (or a wrapper around it) is common. The important part is that the *logic* within the domain methods doesn't depend on specific `net.Conn` behaviors beyond what's abstracted.

**2. Use Cases (Application Business Rules Layer)**

*   **What it is:** This layer contains application-specific business rules. It orchestrates the flow of data to and from the Entities and directs the Entities to use their enterprise-wide business rules to achieve the goals of the application.
*   It defines *what the application does*.
*   It depends on Entities but has no knowledge of databases, UI, or frameworks.
*   Crucially, this layer defines **interfaces** that outer layers (Interface Adapters) must implement (e.g., repository interfaces). This is Dependency Inversion.
*   **Our Example:**
    *   `usecase/client_uc.go`:
        *   `ClientUsecase` struct: Contains methods like `RegisterNewClient`, `ProcessHeartbeat`, `UnregisterClient`, `CheckAllClientsHealth`. These are the specific actions our watchdog application performs.
        *   It uses `ClientRepository` (an interface) to interact with data storage, without knowing *how* that storage is implemented.
        *   It operates on `domain.Client` objects.
    *   `usecase/client_repo.go`:
        *   `ClientRepository` interface: Defines the contract for how client data is saved, retrieved, and deleted (`Save`, `FindByID`, `FindAll`, `Delete`).
    *   **Key Characteristics:**
        *   Imports `domain`.
        *   Does *not* import `interfaces` or `cmd`.
        *   The `ClientUsecase` knows about `domain.Client` but doesn't know if it's talking to a TCP client, a WebSocket client, or if data is stored in memory or PostgreSQL.

**3. Interface Adapters Layer**

*   **What it is:** This layer is a set of adapters that convert data from the format most convenient for Use Cases and Entities, to the format most convenient for some external agency such as the Database or the Web.
*   It includes:
    *   **Controllers/Handlers:** Take input from the UI/Web/Network, process it into a form suitable for the Use Cases, and pass it to them.
    *   **Gateways/Repositories:** Implement the repository interfaces defined by the Use Cases. They know how to talk to specific databases or external services.
    *   **Presenters (less relevant for a simple TCP server, more for UI):** Take data from Use Cases and format it for display on the UI.
*   **Our Example:**
    *   `interfaces/tcp_handler.go`:
        *   `TCPServer` and its `handleConnection` method: This is like a "Controller." It listens for TCP connections, reads data from the network, and translates network events (new connection, incoming message) into calls to the `ClientUsecase` (e.g., `clientUsecase.RegisterNewClient`, `clientUsecase.ProcessHeartbeat`).
        *   It knows about `net.Conn` and how to parse incoming byte streams.
    *   `interfaces/inmem_client_repo.go`:
        *   `InMemoryClientRepository`: This is a "Gateway" or "Repository Implementation." It implements the `usecase.ClientRepository` interface using an in-memory map. It's responsible for the actual storage mechanism.
    *   **Key Characteristics:**
        *   Imports `usecase` and `domain`.
        *   Knows about specific technologies (like Go's `net` package for TCP handling, or the specifics of an in-memory map).
        *   Converts data between the "external world" format and the "use case" format.

**4. Frameworks & Drivers Layer (External World)**

*   **What it is:** This is the outermost layer. It generally consists of frameworks, tools, and drivers: the Web Framework, the Database, the UI framework, device drivers, Go's standard library components like `net`, `os`, `log`.
*   You generally don't write much code here, other than "glue" code that connects to the Interface Adapters. The `main` function often lives here.
*   **Our Example:**
    *   `cmd/server/main.go`:
        *   The `main` function wires everything together (Dependency Injection): creates the repository, then the use case (injecting the repo), then the TCP server (injecting the use case).
        *   It uses `net.Listen` (implicitly via `TCPServer.Start`), `os.Signal` for graceful shutdown.
        *   The `log` package.
        *   The `github.com/google/uuid` package.
    *   Go's `net` package itself.
    *   **Key Characteristics:**
        *   This is where the "details" live.
        *   It's the most volatile layer in terms of changes. Changing your web framework shouldn't force changes in your business rules.

**How the Dependency Rule is Enforced (Dependency Inversion)**

If Use Cases need to talk to a database, they can't directly call database-specific code (because that would violate the rule: inner layers can't know outer layers).
Instead:
1.  The **Use Case layer defines an interface** (e.g., `ClientRepository`).
2.  The **Interface Adapters layer provides a concrete implementation** of that interface (e.g., `InMemoryClientRepository` or `PostgresClientRepository`).
3.  At runtime (often in `main.go`), the concrete implementation is **injected** into the Use Case.

The Use Case only knows about the interface, not the concrete class. This inverts the typical flow of control and dependency.

**Benefits of Clean Architecture for our Watchdog (and in general):**

1.  **Testability:**
    *   **Entities:** Can be tested in complete isolation as they have no external dependencies.
    *   **Use Cases:** Can be tested by mocking the repository interfaces. You don't need a real database or network connection to test `ClientUsecase`.
    *   **Interface Adapters:** `InMemoryClientRepository` can be tested directly. `TCPHandler` can be tested by providing a mock `ClientUsecase`.
2.  **Framework Independence:** The core business logic (Entities, Use Cases) doesn't depend on the `net` package. If you wanted to expose your watchdog via WebSockets instead of raw TCP, you'd primarily change the `interfaces.TCPHandler` (or add a `WebSocketHandler`) without touching the `usecase` or `domain` layers.
3.  **Database Independence:** If you decide to move from `InMemoryClientRepository` to a persistent store like Redis or PostgreSQL, you just need to write a new implementation of `usecase.ClientRepository` in the `interfaces` layer. The `ClientUsecase` remains unchanged.
4.  **Maintainability & Understandability:**
    *   Clear separation of concerns makes it easier to understand where specific logic resides.
    *   Changes in one layer are less likely to ripple through the entire system.
5.  **Independent Development:** Different teams could potentially work on different layers with less friction, as long as the interfaces between layers are well-defined.

**Back to your VmRSS increase problem:**

With Clean Architecture, if you see a memory leak:
*   `pprof` might show `domain.Client` objects accumulating. This points you to:
    *   Is the `ClientRepository` (e.g., `InMemoryClientRepository`) not `Delete`-ing them?
    *   Is the `ClientUsecase` (e.g., `UnregisterClient`, `CheckAllClientsHealth`) failing to call the repository's delete method correctly?
*   If goroutines are leaking, it's likely in the `interfaces.TCPHandler` (failing to close connections or exit handler goroutines) or potentially in long-running background tasks initiated by a use case but not managed properly.

The structure helps you narrow down where the problem might be by understanding the responsibilities of each layer.

While Clean Architecture can introduce some boilerplate for very small projects, its benefits in terms of testability, maintainability, and flexibility often outweigh the initial setup cost for anything non-trivial or expected to grow.
