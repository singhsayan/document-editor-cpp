# C++ Document Editor System

## Overview
The **C++ Document Editor System** is a modular, extensible text and media editor implemented using **Object-Oriented Design (OOD)** principles.  
It demonstrates **High-Level Design (HLD)** and **Low-Level Design (LLD)** concepts along with real-world application of **design patterns** to ensure scalability, maintainability, and clarity.

The system supports:
- Text elements  
- Image elements   
- Line breaks and tab spaces  
- Rendering and saving documents to persistent storage (file or database)

---
# High-Level Design (HLD)

## Overview
The **High-Level Design (HLD)** of the **C++ Document Editor System** describes the system’s major components, data flow, and architectural decisions.  
The system enables **real-time collaborative document editing** using **Cassandra**, **WebSockets**, and **event-driven services** for high scalability and low latency.

---

## Functional Requirements

| Type | Requirement |
|------|--------------|
| **Core** | Single-user CRUD (Create, Read, Update, Delete) operations. |
| | Multi-user real-time collaborative editing. |
| | Live updates via WebSockets. |
| **Out of Scope** | Authentication, document versioning, sharing/permissions, offline access. |

---

## Non-Functional Requirements

| Requirement | Target |
|--------------|---------|
| **High Availability** | 99.9% uptime using distributed services. |
| **Low Latency** | Sub-200 ms update propagation. |
| **Consistency** | Eventual consistency across replicas and clients. |
| **Scalability** | 1M DAU, 5× peak traffic support. |
| **Concurrency** | Up to 100 concurrent editors per document. |

---

## Core Components

| Component | Description |
|------------|--------------|
| **Client Application** | Web or desktop interface where users edit documents in real time. |
| **WebSocket Gateway** | Manages persistent bidirectional connections; receives edits and broadcasts updates. |
| **Message Queue (Kafka / Redis Streams)** | Decouples edit ingestion from persistence; ensures durability and async processing. |
| **Document Service** | Consumes operations, validates edits, applies conflict resolution (OT or LWW), and persists data. |
| **Cassandra Database** | Stores document content and metadata in a distributed, fault-tolerant cluster. |
| **Object Storage (e.g., S3 / MinIO)** | Stores large binary assets such as images. |
| **Cache Layer (Redis / Memcached)** | Speeds up frequent reads and reduces Cassandra load. |

---

## Data Flow

# Synchronization and Conflict Resolution

### Synchronization Protocol
For real-time collaboration, **WebSocket** is chosen as the synchronization protocol.

| Protocol | Suitability | Notes |
|-----------|--------------|-------|
| **HTTP Polling** |  Not suitable | Causes high latency and server overhead due to constant polling. |
| **Server-Sent Events (SSE)** | Partially suitable | Supports one-way communication (server → client) but not ideal for bidirectional edits. |
| **WebSocket** |  Best choice | Enables full-duplex communication, allowing instant edit synchronization across users. |

---

A **dedicated WebSocket Service** handles all real-time connections. It operates independently from the **Document Service** to keep the core system stateless and scalable.  
A **Message Queue (Kafka / Redis Streams)** sits between these two services to decouple ingestion from persistence, ensuring durability and resilience.

---

### Real-Time Edit Flow
1. **Client → WebSocket Service:**  
   The client sends an edit operation (insert, delete, update) over the WebSocket connection.  
   Only the *delta* (change) is sent — not the entire document — to save bandwidth.  

2. **WebSocket → Message Queue:**  
   The WebSocket service publishes the operation into a message queue for asynchronous processing.

3. **Message Queue → Document Service:**  
   The Document Service consumes the message, applies validation and conflict resolution logic, and persists the change.

4. **Document Service → WebSocket:**  
   Once the operation is saved, the Document Service pushes the confirmed update (or merged result) back to the WebSocket layer.

5. **WebSocket → All Clients:**  
   The WebSocket Service broadcasts the update to all connected clients of that document, maintaining real-time consistency.

---

### Conflict Resolution Strategies

#### 1. Last-Write-Wins (LWW)
Each operation is timestamped.  
If multiple edits conflict, the one with the most recent timestamp prevails.

**Pros:**
- Simple and fast to implement  
- Low overhead  

**Cons:**
- Risk of overwriting concurrent user changes  

---

#### 2. Operational Transformation (OT)
Used for advanced real-time editing (like Google Docs).  
Instead of discarding edits, **OT transforms** conflicting operations so that all user intents are preserved.

**Example:**
- User A inserts `" beautiful"` at position 6.  
- User B inserts `" amazing"` at position 6.  
- OT shifts B’s edit to position 16, resulting in → `"Hello beautiful amazing World"`  

**Pros:**
- Preserves all user intents  
- Ensures consistency across all clients  

**Cons:**
- More complex to implement  
- Can become computationally expensive with many concurrent edits  

---

1. **User Edit:** The client sends an edit operation (e.g., insert/delete text) through WebSocket.  
2. **WebSocket Gateway:** Forwards the edit into a **message queue** for async processing.  
3. **Document Service:**  
   - Reads the edit from the queue.  
   - Validates and merges it using **Operational Transformation (OT)** or **Last-Write-Wins (LWW)**.  
   - Persists the updated content in **Cassandra**.  
4. **Broadcast:**  
   - The service publishes the updated operation back to the WebSocket Gateway.  
   - The gateway broadcasts the change to all connected clients of the same document.  
5. **Client Update:**  
   - Clients apply the operation locally to maintain synchronized document state.  

---

## Data Storage Design

- **Cassandra** is used due to its high write throughput, horizontal scalability, and tunable consistency.  
- Each document is stored as a collection of **segments** or **text runs** to support granular edits.  
- Partitioning by `document_id` ensures that all edits for a document are co-located for fast access.

**Example Schema (Cassandra):**
```sql
CREATE TABLE documents (
  document_id UUID,
  version INT,
  content TEXT,
  last_updated TIMESTAMP,
  PRIMARY KEY (document_id, version)
);
```
| Mechanism                           | Description                                                               |
| ----------------------------------- | ------------------------------------------------------------------------- |
| **WebSocket Protocol**              | Enables full-duplex communication for instant edit propagation.           |
| **Operational Transformation (OT)** | Adjusts concurrent edits to maintain document consistency across clients. |
| **Last-Write-Wins (LWW)**           | Fallback conflict resolution based on timestamps.                         |
| **Eventual Consistency**            | All clients eventually converge to the same document state.               |


---
###  HLD Diagram
![High-Level Design Diagram](./screenshots/hld.svg)

---

##  Low-Level Design (LLD)

###  Overview
The **Low-Level Design (LLD)** focuses on how the internal components of the document editor interact at the class level.  
It highlights class responsibilities, relationships, and the design patterns used to achieve modularity and extensibility.

---

### Core Classes and Responsibilities

| Class | Responsibility |
|-------|----------------|
| **DocumentElement (Abstract)** | Base class for all document components. Defines the `render()` interface for all subclasses. |
| **TextElement** | Represents and renders text content in the document. |
| **ImageElement** | Represents and renders an image placeholder with a path reference. |
| **NewLineElement** | Represents a line break (`\n`) in the document. |
| **TabSpaceElement** | Represents a tab space (`\t`) in the document. |
| **Document** | Maintains a collection of `DocumentElement` objects and provides the overall document rendering logic. |
| **Persistence (Abstract)** | Defines the `save(string data)` interface for implementing various storage mechanisms. |
| **FileStorage** | Implements `Persistence` to save the document to a file (local storage). |
| **DBStorage** | Implements `Persistence` for database storage (placeholder). |
| **DocumentEditor** | Acts as the controller — manages document composition, rendering, and persistence operations. |
| **Client** | Represents the end user interacting with the editor. |

---

### Design Principles Followed
- **Single Responsibility Principle (SRP):** Each class has a single, clear purpose.  
- **Open/Closed Principle (OCP):** New element or storage types can be added without modifying existing code.  
- **Dependency Inversion Principle (DIP):** The editor depends on abstractions (`Persistence`), not concrete implementations.  
- **Interface Segregation:** Each class only exposes necessary methods relevant to its purpose.  

---


### UML Class Diagram
![Class Diagram](./screenshots/class_diagram.svg)
