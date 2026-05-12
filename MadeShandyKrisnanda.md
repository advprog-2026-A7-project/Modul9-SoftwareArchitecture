# Individual Works

- Nama: Made Shandy Krisnanda
- NPM: 2406495615

## MySawit: Shipment Service & Frontend (mysawit-web)

> **Tutorial B: Visualizing and Architectural Risk**
> Berdasarkan Container Diagram kelompok, dokumen ini memperluas diagram tersebut dengan Component Diagram dan empat Code Diagram yang berkaitan dengan kontribusi individual saya.

---

## 1. Component Diagram — Shipment Service

Component diagram ini memperbesar tampilan ke dalam container `mysawit-shipment-service`, memperlihatkan komponen-komponen internal dan cara mereka berinteraksi satu sama lain serta dengan container eksternal.

```mermaid
flowchart TD
    subgraph Client["External"]
        FE["mysawit-web\n(Next.js API Gateway)"]
        HarvestSvc["Harvest Service\n(REST)"]
    end

    subgraph ShipmentService["mysawit-shipment-service (Spring Boot, port 8084)"]
        direction TB

        subgraph Security["Security Layer"]
            Filter["ShipmentAccessFilter\n(OncePerRequestFilter)\nValidates JWT & Role"]
            JWT["JwtTokenProvider\nParses & validates JWT claims"]
            Roles["ShipmentRoles\nALLOWED_ROLES constant"]
        end

        subgraph Web["Web Layer"]
            Controller["ShipmentController\n(@RestController)\nGET, POST, PATCH endpoints"]
        end

        subgraph Domain["Domain / Business Logic"]
            Service["ShipmentService\n(@Service)\nOrchestrates all business rules"]
            Policy["ShipmentStatusTransitionPolicy\n(final class)\nImmutable transition map"]
        end

        subgraph Persistence["Persistence Layer"]
            Repo["ShipmentRepository\n(JpaRepository)"]
            ShipmentEntity["Shipment\n(@Entity)"]
            ItemEntity["ShipmentItem\n(@Entity)"]
        end

        subgraph Messaging["Event Publishing"]
            Publisher["ShipmentEventPublisher\n(@Service)\nPublishes to shipment.exchange"]
            Event["ShipmentCompletedEvent\n(record)"]
        end

        subgraph ExternalClient["External Service Client"]
            ClientIF["HarvestServiceClient\n(interface)"]
            ClientImpl["RestTemplateHarvestServiceClient\n(implementation)"]
        end
    end

    subgraph Infra["Infrastructure"]
        DB[("Supabase PostgreSQL")]
        MQ[("RabbitMQ / CloudAMQP")]
    end

    FE -->|"HTTP + Bearer JWT"| Filter
    Filter --> JWT
    Filter --> Roles
    Filter -->|"inject role & userId\ninto request attributes"| Controller

    Controller --> Service
    Service --> Policy
    Service --> Repo
    Service --> ClientIF
    Service --> Publisher

    ClientIF --> ClientImpl
    ClientImpl -->|"GET /harvests/{id}"| HarvestSvc

    Publisher --> Event
    Publisher -->|"shipment.completed\n(shipment.exchange)"| MQ

    Repo --> ShipmentEntity
    ShipmentEntity --> ItemEntity
    Repo -->|"JDBC"| DB
```

---

## 2. Code Diagram 1 — Class Diagram: Domain Model

Model data inti dari shipment service, menampilkan entitas-entitas, atribut, relasi antar entitas, dan enum status pengiriman.

```mermaid
classDiagram
    class Shipment {
        +UUID id
        +UUID mandorUserId
        +UUID supirUserId
        +String destination
        +Double totalKg
        +ShipmentStatus status
        +OffsetDateTime createdAt
        +OffsetDateTime updatedAt
        +List~ShipmentItem~ items
        +addItem(ShipmentItem item)
        #onCreate()
        #onUpdate()
    }

    class ShipmentItem {
        +UUID id
        +UUID harvestId
        +Double weightKg
        +Shipment shipment
    }

    class ShipmentStatus {
        <<enumeration>>
        MEMUAT
        MENGIRIM
        TIBA
        ADMIN_APPROVED
        PARTIALLY_REJECTED
    }

    class ShipmentStatusTransitionPolicy {
        <<final class>>
        -Map~ShipmentStatus,ShipmentStatus~ NEXT_STATUS
        +canTransition(ShipmentStatus from, ShipmentStatus to) boolean$
    }

    class ShipmentCompletedEvent {
        <<record>>
        +UUID shipmentId
        +UUID driverId
        +UUID mandorId
        +Double totalKg
        +List~UUID~ harvestIds
        +OffsetDateTime completedAt
    }

    Shipment "1" --> "0..*" ShipmentItem : contains
    Shipment --> ShipmentStatus : has status
    ShipmentStatusTransitionPolicy ..> ShipmentStatus : validates transition
    ShipmentCompletedEvent ..> Shipment : created from
```

---

## 3. Code Diagram 2 — Class Diagram: Security Layer

Memperlihatkan bagaimana komponen-komponen keamanan berinteraksi untuk mengautentikasi dan mengotorisasi request yang masuk sebelum mencapai controller.

```mermaid
classDiagram
    class ShipmentAccessFilter {
        <<OncePerRequestFilter>>
        -JwtTokenProvider jwtTokenProvider
        +shouldNotFilter(HttpServletRequest) boolean
        +doFilterInternal(request, response, chain) void
    }

    class JwtTokenProvider {
        -String jwtSecret
        +parseClaimsOrNull(String token) Optional~Claims~
    }

    class ShipmentRoles {
        <<constants>>
        +SUPIR : String$
        +MANDOR : String$
        +ADMIN : String$
        +ALLOWED_ROLES : Set~String~$
    }

    class ShipmentSecurityAttributes {
        <<constants>>
        +JWT_ROLE : String$
        +JWT_USER_ID : String$
    }

    class ShipmentController {
        -ShipmentService shipmentService
        +getAllShipments(request) ResponseEntity
        +getShipmentById(id, request) ResponseEntity
        +createShipment(body, request) ResponseEntity
        +updateShipmentStatus(id, body, request) ResponseEntity
        +approveShipmentByAdmin(id, body, request) ResponseEntity
        -requireRole(request, role) void
        -extractRequesterRole(request) String
        -extractRequesterUserId(request) UUID
    }

    ShipmentAccessFilter --> JwtTokenProvider : uses
    ShipmentAccessFilter --> ShipmentRoles : checks against
    ShipmentAccessFilter --> ShipmentSecurityAttributes : writes attributes
    ShipmentController --> ShipmentSecurityAttributes : reads attributes
    ShipmentController --> ShipmentRoles : requireRole checks
```

---

## 4. Code Diagram 3 — Sequence Diagram: Create Shipment (MANDOR)

Menggambarkan alur lengkap ketika MANDOR membuat shipment baru, mencakup validasi JWT, validasi panen via REST, pengecekan batas berat, dan penyimpanan ke database.

```mermaid
sequenceDiagram
    actor Mandor
    participant FE as mysawit-web<br/>(Next.js Gateway)
    participant Filter as ShipmentAccessFilter
    participant Controller as ShipmentController
    participant Service as ShipmentService
    participant HarvestClient as RestTemplateHarvestServiceClient
    participant HarvestSvc as Harvest Service
    participant Repo as ShipmentRepository
    participant DB as Supabase PostgreSQL

    Mandor->>FE: POST /api/shipments (with JWT)
    FE->>Filter: Forward request
    Filter->>Filter: Extract & validate JWT
    alt Invalid / missing token
        Filter-->>FE: 401 Unauthorized
    else Role != MANDOR
        Filter-->>FE: 403 Forbidden
    else Valid, role = MANDOR
        Filter->>Controller: inject role & userId, pass request
    end

    Controller->>Service: createShipment(mandorUserId, request)

    Service->>Service: calculateTotalKg(request)
    alt totalKg > 400
        Service-->>Controller: ShipmentWeightExceededException
        Controller-->>FE: 400 Bad Request
    end

    Service->>Service: check duplicate harvestIds in request
    alt duplicates found
        Service-->>Controller: HarvestValidationException
        Controller-->>FE: 400 Bad Request
    end

    loop for each harvestId
        Service->>HarvestClient: getHarvestById(mandorUserId, harvestId)
        HarvestClient->>HarvestSvc: GET /harvests/{harvestId}
        HarvestSvc-->>HarvestClient: HarvestDetails {id, status}
        alt Harvest not found
            Service-->>Controller: HarvestValidationException (404)
            Controller-->>FE: 422 Unprocessable Entity
        else status != "Approved"
            Service-->>Controller: HarvestValidationException (status mismatch)
            Controller-->>FE: 422 Unprocessable Entity
        end
    end

    Service->>Service: buildShipment(mandorUserId, request, totalKg)
    Service->>Repo: save(shipment)
    Repo->>DB: INSERT shipments + shipment_items
    DB-->>Repo: saved Shipment
    Repo-->>Service: Shipment
    Service-->>Controller: Shipment
    Controller-->>FE: 201 Created + ShipmentResponse
    FE-->>Mandor: Shipment created
```

---

## 5. Code Diagram 4 — Sequence Diagram: Update Status & Event Publishing (SUPIR)

Menggambarkan alur ketika SUPIR memperbarui status pengiriman, mencakup pengecekan kepemilikan, validasi state machine, dan penerbitan event ke RabbitMQ saat shipment mencapai status `TIBA`.

```mermaid
sequenceDiagram
    actor Supir
    participant FE as mysawit-web<br/>(Next.js Gateway)
    participant Filter as ShipmentAccessFilter
    participant Controller as ShipmentController
    participant Service as ShipmentService
    participant Policy as ShipmentStatusTransitionPolicy
    participant Repo as ShipmentRepository
    participant Publisher as ShipmentEventPublisher
    participant MQ as RabbitMQ (CloudAMQP)
    participant Payroll as Payroll Service

    Supir->>FE: PATCH /api/shipments/{id}/status (with JWT)
    FE->>Filter: Forward request
    Filter->>Filter: Validate JWT
    Filter->>Filter: Check role in ALLOWED_ROLES
    alt role != SUPIR
        Filter-->>FE: 403 Forbidden
    else Valid, role = SUPIR
        Filter->>Controller: inject role & userId
    end

    Controller->>Controller: requireRole(SUPIR)
    Controller->>Service: updateShipmentStatus(id, supirUserId, targetStatus)

    Service->>Repo: findById(id)
    Repo-->>Service: Shipment
    alt not found
        Service-->>Controller: ShipmentNotFoundException
        Controller-->>FE: 404 Not Found
    end

    Service->>Service: ensureOwnedByRequester(shipment, supirUserId)
    alt supirUserId != shipment.supirUserId
        Service-->>Controller: ShipmentForbiddenException
        Controller-->>FE: 403 Forbidden
    end

    Service->>Policy: canTransition(current, targetStatus)
    alt invalid transition
        Policy-->>Service: false
        Service-->>Controller: ShipmentInvalidTransitionException
        Controller-->>FE: 409 Conflict
    else valid
        Policy-->>Service: true
    end

    Service->>Service: shipment.setStatus(targetStatus)
    Service->>Repo: save(shipment)
    Repo-->>Service: saved Shipment

    alt targetStatus == TIBA
        Service->>Publisher: publishShipmentCompleted(shipment)
        Publisher->>Publisher: build ShipmentCompletedEvent
        Publisher->>MQ: convertAndSend("shipment.exchange", "shipment.completed", event)
        MQ-->>Payroll: deliver ShipmentCompletedEvent (async)
    end

    Service-->>Controller: Shipment
    Controller-->>FE: 200 OK + ShipmentResponse
    FE-->>Supir: Status updated
```
