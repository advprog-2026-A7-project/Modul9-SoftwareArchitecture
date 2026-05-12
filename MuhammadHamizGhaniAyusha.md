# Individual Works

- Nama: Muhammad Hamiz Ghani Ayusha
- NPM: 2406360413

## MySawit: Payroll Service & Frontend Integration

> **Tutorial B: Visualizing and Architectural Risk**
> Berdasarkan Container Diagram kelompok, dokumen ini memperluas diagram tersebut dengan Component Diagram dan empat Code Diagram yang berkaitan dengan kontribusi individual saya pada domain Payroll.

---

## 1. Component Diagram - Payroll Service

Component diagram ini memperbesar tampilan ke dalam container `mysawit-payroll-service`, memperlihatkan komponen internal untuk manajemen employee, payroll, konfigurasi upah, konsumsi event RabbitMQ, serta integrasi dengan container eksternal.

```mermaid
flowchart TD
    subgraph Client["External"]
        FE["mysawit-web\n(Next.js Frontend)"]
        IdentitySvc["Identity Service\n(REST / user detail)"]
        HarvestSvc["Harvest Service\n(publishes payroll event)"]
        ShipmentSvc["Shipment Service\n(publishes shipment.completed)"]
    end

    subgraph PayrollService["mysawit-payroll-service (Spring Boot, port 8085)"]
        direction TB

        subgraph Web["Web Layer"]
            EmployeeController["EmployeeController\n/api/employees"]
            PayrollController["PayrollController\n/api/payrolls"]
            WageConfigController["WageConfigController\n/api/admin/wage-configs"]
        end

        subgraph Domain["Domain / Business Logic"]
            EmployeeService["EmployeeService\nEmployee CRUD + user registration mapping"]
            PayrollServiceComp["PayrollService\nPayroll lifecycle + event idempotency"]
            WageConfigService["WageConfigService\nRole wage validation + active config lookup"]
        end

        subgraph Persistence["Persistence Layer"]
            EmployeeRepo["EmployeeRepository\nJpaRepository"]
            PayrollRepo["PayrollRepository\nJpaRepository"]
            WageConfigRepo["WageConfigRepository\nJpaRepository + active config query"]
            EmployeeEntity["Employee\n@Entity"]
            PayrollEntity["Payroll\n@Entity"]
            WageConfigEntity["WageConfig\n@Entity"]
        end

        subgraph Messaging["Event Consumption"]
            RabbitConfig["RabbitMqConfig\nqueues + JSON converter"]
            Consumer["PayrollEventConsumerImpl\n@RabbitListener"]
            PayrollEvent["HarvestEvent / ShipmentEvent\nPayrollEvent payload"]
            UserEvent["UserRegisteredEvent\nIdentity payload"]
        end

        subgraph ExternalClient["External Service Client"]
            IdentityClient["IdentityClient\nOpenFeign client"]
        end
    end

    subgraph Infra["Infrastructure"]
        DB[("Supabase PostgreSQL")]
        MQ[("RabbitMQ / CloudAMQP")]
    end

    FE -->|"HTTP REST"| EmployeeController
    FE -->|"HTTP REST"| PayrollController
    FE -->|"HTTP REST"| WageConfigController

    EmployeeController --> EmployeeService
    PayrollController --> PayrollServiceComp
    WageConfigController --> WageConfigService

    EmployeeService --> EmployeeRepo
    PayrollServiceComp --> PayrollRepo
    PayrollServiceComp --> IdentityClient
    WageConfigService --> WageConfigRepo

    EmployeeRepo --> EmployeeEntity
    PayrollRepo --> PayrollEntity
    WageConfigRepo --> WageConfigEntity

    EmployeeRepo -->|"JDBC"| DB
    PayrollRepo -->|"JDBC"| DB
    WageConfigRepo -->|"JDBC"| DB

    HarvestSvc -->|"payroll_queue"| MQ
    ShipmentSvc -->|"shipment.completed"| MQ
    IdentitySvc -->|"user.registered.queue"| MQ
    MQ --> Consumer
    RabbitConfig --> Consumer
    Consumer --> PayrollEvent
    Consumer --> UserEvent
    Consumer --> PayrollServiceComp
    Consumer --> EmployeeService

    IdentityClient -->|"GET /api/user/{id}"| IdentitySvc
```

---

## 2. Code Diagram 1 - Class Diagram: Domain Model

Model data inti dari payroll service, mencakup entitas yang disimpan di database dan payload event yang dikonsumsi dari RabbitMQ.

```mermaid
classDiagram
    class BaseEntity {
        <<MappedSuperclass>>
        -LocalDateTime createdAt
        -LocalDateTime updatedAt
        #onCreate()
        #onUpdate()
    }

    class Employee {
        +Long id
        +String name
        +String employeeCode
        +String position
        +Long plantationId
        +String phoneNumber
        +String address
        +LocalDateTime hireDate
        +Double baseSalary
        +String status
    }

    class Payroll {
        +Long id
        +String eventId
        +Long employeeId
        +LocalDateTime periodStart
        +LocalDateTime periodEnd
        +Double baseAmount
        +Double bonusAmount
        +Double deductionAmount
        +Double totalAmount
        +String status
        +LocalDateTime paymentDate
        +String paymentMethod
        +String notes
        #onCreate()
        #onUpdate()
        -calculateTotal()
    }

    class WageConfig {
        +Long id
        +String roleType
        +Double ratePerKg
        +LocalDate effectiveDate
        +String description
        +String createdBy
    }

    class PayrollEvent {
        +String eventId
        +String employeeId
        +double amount
        +long timestamp
    }

    class HarvestEvent
    class ShipmentEvent

    class UserRegisteredEvent {
        +Long id
        +Long userId
        +String username
        +String email
        +String role
        +String name
        +String fullName
        +String employeeCode
        +String position
        +resolveUserId() Long
    }

    BaseEntity <|-- Employee
    BaseEntity <|-- WageConfig
    PayrollEvent <|-- HarvestEvent
    PayrollEvent <|-- ShipmentEvent
    Payroll ..> Employee : references employeeId
    WageConfig ..> Employee : configures wage by role
```

---

## 3. Code Diagram 2 - Class Diagram: REST, Service, Repository, and Messaging

Diagram ini memperlihatkan pembagian tanggung jawab antar controller, service, repository, RabbitMQ consumer, dan client eksternal.

```mermaid
classDiagram
    class EmployeeController {
        -EmployeeService employeeService
        +getAllEmployees() ResponseEntity
        +getEmployeeById(id) ResponseEntity
        +getEmployeeByCode(employeeCode) ResponseEntity
        +getEmployeesByPlantation(plantationId) ResponseEntity
        +getEmployeesByStatus(status) ResponseEntity
        +createEmployee(employee) ResponseEntity
        +updateEmployee(id, employee) ResponseEntity
        +deleteEmployee(id) ResponseEntity
    }

    class PayrollController {
        -PayrollService payrollService
        +getAllPayrolls() ResponseEntity
        +getPayrollById(id) ResponseEntity
        +getPayrollsByEmployee(employeeId) ResponseEntity
        +getPayrollsByStatus(status) ResponseEntity
        +createPayroll(payroll) ResponseEntity
        +updatePayroll(id, payroll) ResponseEntity
        +approvePayroll(id) ResponseEntity
        +acceptPayroll(id) ResponseEntity
        +rejectPayroll(id, request) ResponseEntity
        +markAsPaid(id, request) ResponseEntity
        +deletePayroll(id) ResponseEntity
    }

    class WageConfigController {
        -WageConfigService wageConfigService
        +getAllConfigs() ResponseEntity
        +getConfigById(id) ResponseEntity
        +getConfigsByRole(role) ResponseEntity
        +getActiveConfigForRole(role) ResponseEntity
        +createConfig(wageConfig) ResponseEntity
        +updateConfig(id, wageConfig) ResponseEntity
        +deleteConfig(id) ResponseEntity
    }

    class EmployeeService {
        -EmployeeRepository employeeRepository
        -Double defaultBaseSalary
        +createEmployeeFromUserRegistered(event) Employee
        +createEmployee(employee) Employee
        +updateEmployee(id, employeeDetails) Employee
        +deleteEmployee(id) void
    }

    class PayrollService {
        -PayrollRepository payrollRepository
        -IdentityClient identityClient
        -String identityServiceToken
        +createPayroll(payroll) Payroll
        +approvePayroll(id) Payroll
        +acceptPayroll(id) Payroll
        +rejectPayroll(id, reason) Payroll
        +markAsPaid(id, paymentMethod) Payroll
        +processHarvestPayroll(event) void
        +processShipmentPayroll(event) void
        -processPayrollEvent(eventId, employeeId, amount, eventType) void
    }

    class WageConfigService {
        -WageConfigRepository wageConfigRepository
        +getActiveConfigForRole(roleType) Optional~WageConfig~
        +createConfig(wageConfig) WageConfig
        +updateConfig(id, details) WageConfig
        -validateRoleType(roleType) void
    }

    class PayrollEventConsumerImpl {
        -PayrollService payrollService
        -EmployeeService employeeService
        +handleHarvestEvent(event) void
        +handleShipmentEvent(event) void
        +handleUserRegisteredEvent(event) void
    }

    class RabbitMqConfig {
        +harvestPayrollQueue(queueName) Queue
        +shipmentCompletedQueue(queueName) Queue
        +userRegisteredQueue(queueName) Queue
        +jsonMessageConverter() MessageConverter
        +rabbitListenerContainerFactory(...) SimpleRabbitListenerContainerFactory
    }

    class IdentityClient {
        <<FeignClient>>
        +getUserById(id, token) Map~String,Object~
    }

    class EmployeeRepository {
        <<JpaRepository>>
        +findByEmployeeCode(code) Optional~Employee~
        +findByPlantationId(plantationId) List~Employee~
        +findByStatus(status) List~Employee~
    }

    class PayrollRepository {
        <<JpaRepository>>
        +findByEmployeeId(employeeId) List~Payroll~
        +findByStatus(status) List~Payroll~
        +findByEventId(eventId) Payroll
    }

    class WageConfigRepository {
        <<JpaRepository>>
        +findByRoleTypeOrderByEffectiveDateDesc(roleType) List~WageConfig~
        +findActiveConfigForRole(roleType, today) List~WageConfig~
    }

    EmployeeController --> EmployeeService
    PayrollController --> PayrollService
    WageConfigController --> WageConfigService
    EmployeeService --> EmployeeRepository
    PayrollService --> PayrollRepository
    PayrollService --> IdentityClient
    WageConfigService --> WageConfigRepository
    PayrollEventConsumerImpl --> PayrollService
    PayrollEventConsumerImpl --> EmployeeService
    RabbitMqConfig ..> PayrollEventConsumerImpl : configures listener factory
```

---

## 4. Code Diagram 3 - Sequence Diagram: Manual Payroll Lifecycle

Menggambarkan alur saat admin atau frontend membuat payroll secara manual, lalu mengubah status payroll dari `PENDING` menjadi `APPROVED` dan akhirnya `PAID`.

```mermaid
sequenceDiagram
    actor Admin
    participant FE as mysawit-web<br/>(Payroll Page)
    participant Controller as PayrollController
    participant Service as PayrollService
    participant Repo as PayrollRepository
    participant Entity as Payroll Entity
    participant DB as Supabase PostgreSQL

    Admin->>FE: Fill payroll form
    FE->>Controller: POST /api/payrolls
    Controller->>Service: createPayroll(payroll)
    Service->>Repo: save(payroll)
    Repo->>Entity: @PrePersist onCreate()
    Entity->>Entity: calculateTotal()
    Repo->>DB: INSERT payrolls
    DB-->>Repo: saved row
    Repo-->>Service: Payroll
    Service-->>Controller: Payroll
    Controller-->>FE: 201 Created + Payroll
    FE-->>Admin: Payroll displayed as PENDING

    Admin->>FE: Click Approve
    FE->>Controller: PATCH /api/payrolls/{id}/approve
    Controller->>Service: approvePayroll(id)
    Service->>Repo: findById(id)
    alt payroll not found
        Service-->>Controller: RuntimeException
        Controller-->>FE: 404 Not Found
    else payroll found
        Service->>Service: setStatus("APPROVED")
        Service->>Repo: save(payroll)
        Repo->>DB: UPDATE payrolls
        Repo-->>Service: approved Payroll
        Service-->>Controller: approved Payroll
        Controller-->>FE: 200 OK
    end

    Admin->>FE: Click Mark as Paid
    FE->>Controller: PATCH /api/payrolls/{id}/pay
    Controller->>Service: markAsPaid(id, paymentMethod)
    Service->>Repo: findById(id)
    alt payroll not found
        Service-->>Controller: RuntimeException
        Controller-->>FE: 404 Not Found
    else payroll found
        Service->>Service: setStatus("PAID")
        Service->>Service: setPaymentDate(now)
        Service->>Service: setPaymentMethod(paymentMethod)
        Service->>Repo: save(payroll)
        Repo->>DB: UPDATE payrolls
        Repo-->>Service: paid Payroll
        Service-->>Controller: paid Payroll
        Controller-->>FE: 200 OK
        FE-->>Admin: Payroll marked as PAID
    end
```

---

## 5. Code Diagram 4 - Sequence Diagram: Event-Driven Payroll Generation

Menggambarkan alur pembuatan payroll secara asinkron ketika Harvest Service atau Shipment Service mengirim event ke RabbitMQ. Payroll Service memastikan idempotency dengan `eventId` agar event yang sama tidak membuat payroll ganda.

```mermaid
sequenceDiagram
    participant Producer as Harvest/Shipment Service
    participant MQ as RabbitMQ
    participant Consumer as PayrollEventConsumerImpl
    participant Service as PayrollService
    participant Repo as PayrollRepository
    participant IdentityClient as IdentityClient
    participant Identity as Identity Service
    participant DB as Supabase PostgreSQL

    Producer->>MQ: Publish HarvestEvent / ShipmentEvent
    MQ->>Consumer: Deliver event from configured queue

    alt event from harvest queue
        Consumer->>Service: processHarvestPayroll(event)
    else event from shipment.completed queue
        Consumer->>Service: processShipmentPayroll(event)
    end

    Service->>Repo: findByEventId(event.eventId)
    alt duplicate event
        Repo-->>Service: existing Payroll
        Service-->>Consumer: return without save
    else new event
        Repo-->>Service: null
        Service->>Service: build Payroll from employeeId and amount
        Service->>Service: status = "PENDING"
        Service->>Service: notes = "Generated from event"

        alt identity client and service token configured
            Service->>IdentityClient: getUserById(employeeId, Bearer token)
            IdentityClient->>Identity: GET /api/user/{id}
            alt user detail found
                Identity-->>IdentityClient: user detail
                IdentityClient-->>Service: username map
                Service->>Service: append username to notes
            else identity lookup failed
                Identity-->>IdentityClient: error
                IdentityClient-->>Service: exception
                Service->>Service: append "user fetch failed" to notes
            end
        end

        Service->>Repo: save(payroll)
        Repo->>DB: INSERT payrolls with event_id
        DB-->>Repo: saved row
        Repo-->>Service: Payroll
        Service-->>Consumer: done
    end
```
