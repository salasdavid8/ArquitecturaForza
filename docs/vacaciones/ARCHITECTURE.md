# ARCHITECTURE.md — Gestión de Vacaciones (Forza Vacaciones)

> Generado con GitHub Copilot usando el stack ForzaTech v3.
> Actualizar este archivo en cada cambio arquitectónico significativo.

---

## Descripción del Servicio

**Nombre:** `forza-vacaciones`
**Dominio:** `Recursos Humanos / Operaciones`
**Track:** Innovación
**Responsable:** David Salas — Gerente de Desarrollo y Tecnología
**Última actualización:** 2026-04-27

Sistema web para que los colaboradores de Forza Logistics Group soliciten períodos de vacaciones. El flujo de aprobación es **multi-nivel** y se orquesta con Temporal.io, garantizando que ningún paso se pierda aunque ocurran fallos de infraestructura. Los managers aprueban o rechazan desde la misma aplicación web; en cada transición se envía notificación en tiempo real vía NATS.

---

## Arquitectura General

```mermaid
flowchart LR
    subgraph Clients
        EMP["Colaborador\n(Angular Web)"]
        MGR["Manager / RRHH\n(Angular Web)"]
    end

    APISIX["Apache APISIX\nAPI Gateway"]
    KC["Keycloak\nIdentity Provider"]

    subgraph Backend ["forza-vacaciones (.NET Core)"]
        API[".NET Core API\nClean Architecture"]
        WORKER[".NET Worker\nConsumidor de eventos"]
    end

    PG[("PostgreSQL\nforza_vacaciones")]
    NATS[["NATS JetStream\nrrhh.*"]]
    Temporal["Temporal.io\nVacationApprovalWorkflow"]
    NOTIF["Notification Service\n(NATS subscriber)"]

    EMP & MGR -->|"HTTPS"| APISIX
    APISIX -->|"openid-connect plugin"| KC
    APISIX -->|"JWT validado + roles"| API
    API -->|"EF Core"| PG
    API -->|"Inicia / señaliza workflow"| Temporal
    Temporal -->|"Activities"| API
    API -->|"Publica eventos"| NATS
    NATS -->|"Subscribe"| WORKER
    NATS -->|"Subscribe"| NOTIF
    WORKER -->|"Actualiza read-models"| PG
```

---

## Estados de una Solicitud de Vacaciones

```mermaid
stateDiagram-v2
    [*] --> Borrador : Colaborador crea borrador
    Borrador --> PendienteNivel1 : Colaborador envía solicitud
    PendienteNivel1 --> PendienteNivel2 : Manager directo aprueba
    PendienteNivel1 --> Rechazada : Manager directo rechaza
    PendienteNivel2 --> PendienteNivel3 : RRHH aprueba
    PendienteNivel2 --> Rechazada : RRHH rechaza
    PendienteNivel3 --> Aprobada : Gerente Regional aprueba\n(solo si duración > 5 días)
    PendienteNivel3 --> Rechazada : Gerente Regional rechaza
    PendienteNivel2 --> Aprobada : RRHH aprueba\n(si duración ≤ 5 días)
    Aprobada --> Cancelada : Colaborador cancela\n(hasta 48h antes)
    Rechazada --> [*]
    Aprobada --> [*]
    Cancelada --> [*]
```

---

## Flujo de Autenticación y Roles

```mermaid
sequenceDiagram
    participant C as Angular (PKCE)
    participant A as APISIX Gateway
    participant K as Keycloak
    participant S as .NET Core API

    C->>K: Authorization Code Flow + PKCE
    K-->>C: JWT (roles: employee | manager | hr | regional_manager | admin)
    C->>A: HTTP Request + Bearer JWT
    A->>K: Validar token (openid-connect plugin)
    K-->>A: Claims validados + roles
    A->>S: Request con claims inyectados
    S->>S: Autorización basada en roles (ICurrentUser)
    S-->>A: Response
    A-->>C: Response HTTP
```

**Roles en Keycloak:**

| Rol | Descripción |
|-----|-------------|
| `employee` | Puede crear y cancelar sus propias solicitudes |
| `manager` | Aprobación Nivel 1 — equipo directo a cargo |
| `hr` | Aprobación Nivel 2 — todas las solicitudes |
| `regional_manager` | Aprobación Nivel 3 — solicitudes > 5 días |
| `admin` | Gestión del sistema, saldos, períodos bloqueados |

---

## Flujo de Aprobación Multi-Nivel (Temporal.io)

```mermaid
sequenceDiagram
    participant EMP as Colaborador
    participant API as .NET Core API
    participant T as Temporal Workflow
    participant MGR as Manager Directo
    participant HR as RRHH
    participant REG as Gerente Regional
    participant NATS as NATS JetStream

    EMP->>API: POST /vacation-requests
    API->>API: Valida saldo disponible
    API->>T: StartWorkflow(VacationApprovalWorkflow)
    T->>NATS: rrhh.vacacion.solicitada
    NATS-->>MGR: Notificación push (nivel 1)

    MGR->>API: PUT /vacation-requests/{id}/approve (nivel 1)
    API->>T: Signal(ApproveLevel1)
    T->>NATS: rrhh.vacacion.nivel1.aprobada

    alt Duración ≤ 5 días
        T->>NATS: rrhh.vacacion.nivel2.pendiente
        NATS-->>HR: Notificación push (nivel 2)
        HR->>API: PUT /vacation-requests/{id}/approve (nivel 2)
        API->>T: Signal(ApproveLevel2)
        T->>T: Completa workflow → Aprobada
    else Duración > 5 días
        T->>NATS: rrhh.vacacion.nivel2.pendiente
        NATS-->>HR: Notificación push (nivel 2)
        HR->>API: PUT /vacation-requests/{id}/approve (nivel 2)
        API->>T: Signal(ApproveLevel2)
        T->>NATS: rrhh.vacacion.nivel3.pendiente
        NATS-->>REG: Notificación push (nivel 3)
        REG->>API: PUT /vacation-requests/{id}/approve (nivel 3)
        API->>T: Signal(ApproveLevel3)
        T->>T: Completa workflow → Aprobada
    end

    T->>NATS: rrhh.vacacion.aprobada
    NATS-->>EMP: Notificación final al colaborador
```

---

## Flujo de Eventos con Outbox Pattern

```mermaid
sequenceDiagram
    participant API as .NET Core API
    participant PG as PostgreSQL
    participant Outbox as outbox_events table
    participant NATS as NATS JetStream
    participant Worker as .NET Worker

    API->>PG: UPDATE vacation_requests (misma transacción)
    API->>Outbox: INSERT evento (misma transacción)
    Worker->>Outbox: Poll eventos no publicados
    Worker->>NATS: Publish evento
    Worker->>Outbox: Marcar como publicado
    NATS-->>Worker: Subscribe rrhh.vacacion.*
    Worker->>PG: Actualiza read-models / saldos
```

---

## Modelo de Datos

```mermaid
erDiagram
    EMPLOYEES {
        uuid employee_id PK
        string keycloak_user_id UK
        string full_name
        string email
        uuid manager_id FK
        string department
        string country
        timestamp created_at
    }

    VACATION_BALANCES {
        uuid balance_id PK
        uuid employee_id FK
        int year
        int total_days
        int used_days
        int pending_days
        int available_days
        timestamp updated_at
    }

    VACATION_REQUESTS {
        uuid request_id PK
        uuid employee_id FK
        date start_date
        date end_date
        int total_days
        string status
        string rejection_reason
        string temporal_workflow_id
        timestamp created_at
        timestamp updated_at
    }

    APPROVAL_STEPS {
        uuid step_id PK
        uuid request_id FK
        int level
        uuid approver_id FK
        string action
        string comments
        timestamp actioned_at
    }

    BLOCKED_PERIODS {
        uuid period_id PK
        string country
        date start_date
        date end_date
        string reason
    }

    OUTBOX_EVENTS {
        uuid event_id PK
        string subject
        jsonb payload
        boolean published
        timestamp created_at
        timestamp published_at
    }

    EMPLOYEES ||--o{ VACATION_REQUESTS : "solicita"
    EMPLOYEES ||--o{ VACATION_BALANCES : "tiene"
    VACATION_REQUESTS ||--o{ APPROVAL_STEPS : "pasa por"
    EMPLOYEES ||--o{ APPROVAL_STEPS : "aprueba/rechaza"
```

---

## Stack de Dependencias

### Runtime

| Componente | Tecnología | Versión | Notas |
|------------|-----------|---------|-------|
| Framework | .NET Core | 8.0 LTS | `net8.0` target |
| ORM | Entity Framework Core | 8.x | Migrations en `Infrastructure/Persistence/Migrations` |
| CQRS | MediatR | 12.x | Commands y Queries en `Application/` |
| Workflows | Temporalio SDK | 1.x | `VacationApprovalWorkflow` con signals |
| Mensajería | NATS.Net | 2.x | JetStream — subjects `rrhh.*` |
| Resiliencia | Polly | 8.x | Retry + Circuit Breaker en Activities de Temporal |
| Logging | Serilog | 3.x | JSON estructurado + correlationId por solicitud |
| Tests | xUnit + Moq | Latest | Cobertura mínima: 80% |
| Frontend | Angular 17+ | LTS | NgRx Signals, Standalone Components, Lazy Loading |

### Infraestructura

| Componente | Herramienta | Configuración |
|------------|------------|---------------|
| API Gateway | Apache APISIX | Routes por rol en `apisix/routes-vacaciones.yaml` |
| Identity | Keycloak | Realm: `forza-prod`, Client: `vacaciones-spa` |
| Workflows | Temporal.io | Namespace: `forza-vacaciones` |
| Orquestación | Kubernetes + Rancher | Namespace: `prod-vacaciones` |
| CI/CD | GitHub Actions | `.github/workflows/ci-vacaciones.yaml` |
| Calidad | Codacy | Gate: sin issues Críticos/Altos |
| Imágenes | GHCR | `ghcr.io/forzatech/forza-vacaciones:v1.0.0-sha` |

---

## Estructura del Proyecto (Clean Architecture)

```
src/
├── ForzaVacaciones.Domain/
│   ├── Entities/
│   │   ├── VacationRequest.cs
│   │   ├── ApprovalStep.cs
│   │   ├── VacationBalance.cs
│   │   └── BlockedPeriod.cs
│   ├── ValueObjects/
│   │   ├── VacationRequestId.cs
│   │   └── DateRange.cs
│   ├── Enums/
│   │   └── VacationRequestStatus.cs
│   └── Interfaces/
│       ├── IVacationRequestRepository.cs
│       └── IVacationBalanceRepository.cs
│
├── ForzaVacaciones.Application/
│   ├── Commands/
│   │   ├── CreateVacationRequest/
│   │   │   ├── CreateVacationRequestCommand.cs
│   │   │   └── CreateVacationRequestCommandHandler.cs
│   │   ├── ApproveVacationRequest/
│   │   │   ├── ApproveVacationRequestCommand.cs
│   │   │   └── ApproveVacationRequestCommandHandler.cs
│   │   └── RejectVacationRequest/
│   ├── Queries/
│   │   ├── GetMyVacationRequests/
│   │   ├── GetPendingApprovals/
│   │   └── GetVacationBalance/
│   ├── DTOs/
│   │   ├── VacationRequestDto.cs
│   │   └── ApprovalStepDto.cs
│   └── Interfaces/
│       └── ICurrentUser.cs
│
├── ForzaVacaciones.Infrastructure/
│   ├── Persistence/
│   │   ├── AppDbContext.cs
│   │   ├── Migrations/
│   │   ├── Repositories/
│   │   └── OutboxRelay/
│   ├── Temporal/
│   │   ├── Workflows/
│   │   │   └── VacationApprovalWorkflow.cs
│   │   └── Activities/
│   │       ├── NotifyApproverActivity.cs
│   │       └── UpdateRequestStatusActivity.cs
│   └── Messaging/
│       └── NatsPublisher.cs
│
└── ForzaVacaciones.Presentation/
    ├── Controllers/
    │   ├── VacationRequestsController.cs
    │   └── ApprovalsController.cs
    └── Middleware/
        └── CurrentUserMiddleware.cs

frontend/
└── forza-vacaciones-web/           # Angular 17+
    ├── src/app/
    │   ├── features/
    │   │   ├── my-vacations/       # Vista colaborador
    │   │   ├── approvals/          # Vista manager / RRHH
    │   │   └── admin/              # Vista admin
    │   ├── core/
    │   │   ├── auth/               # Keycloak PKCE
    │   │   └── interceptors/       # JWT + correlationId
    │   └── shared/

tests/
├── ForzaVacaciones.UnitTests/
└── ForzaVacaciones.IntegrationTests/
```

---

## Convenciones de Naming en este Servicio

```csharp
// Commands y Queries
CreateVacationRequestCommand
ApproveVacationRequestCommand
RejectVacationRequestCommand
GetMyVacationRequestsQuery
GetPendingApprovalsQuery

// Entidades
VacationRequest
ApprovalStep
VacationBalance

// Value Objects
VacationRequestId
DateRange

// Temporal
VacationApprovalWorkflow
NotifyApproverActivity

// NATS Subjects (rrhh.entidad.accion)
rrhh.vacacion.solicitada
rrhh.vacacion.nivel1.aprobada
rrhh.vacacion.nivel1.rechazada
rrhh.vacacion.nivel2.aprobada
rrhh.vacacion.nivel2.rechazada
rrhh.vacacion.nivel3.aprobada
rrhh.vacacion.nivel3.rechazada
rrhh.vacacion.aprobada
rrhh.vacacion.rechazada
rrhh.vacacion.cancelada
```

---

## Endpoints Principales (REST)

| Método | Ruta | Rol requerido | Descripción |
|--------|------|---------------|-------------|
| `POST` | `/vacation-requests` | `employee` | Crear solicitud |
| `GET` | `/vacation-requests/mine` | `employee` | Mis solicitudes |
| `DELETE` | `/vacation-requests/{id}` | `employee` | Cancelar (≥ 48h antes) |
| `GET` | `/vacation-requests/pending-approvals` | `manager`, `hr`, `regional_manager` | Solicitudes pendientes por aprobar |
| `PUT` | `/vacation-requests/{id}/approve` | `manager`, `hr`, `regional_manager` | Aprobar en el nivel correspondiente |
| `PUT` | `/vacation-requests/{id}/reject` | `manager`, `hr`, `regional_manager` | Rechazar con motivo |
| `GET` | `/vacation-balances/mine` | `employee` | Mi saldo de vacaciones |
| `GET` | `/vacation-balances/{employeeId}` | `hr`, `admin` | Saldo de un colaborador |
| `POST` | `/blocked-periods` | `admin` | Definir período bloqueado |

---

## Variables de Entorno Requeridas

> Los valores NUNCA van en el código ni en Git. Usar Vault o K8s Secrets.

```bash
# Base de datos
ConnectionStrings__DefaultConnection=   # Vault: secret/vacaciones/pg-connection

# Keycloak
Keycloak__Authority=                    # https://keycloak.forzatech.com/realms/forza-prod
Keycloak__Audience=                     # forza-vacaciones

# NATS
Nats__Url=                              # nats://nats.forzatech.svc.cluster.local:4222
Nats__StreamName=                       # RRHH

# Temporal
Temporal__HostPort=                     # temporal.forzatech.svc.cluster.local:7233
Temporal__Namespace=                    # forza-vacaciones

# Notificaciones
Notifications__WebhookUrl=              # Vault: secret/vacaciones/notif-webhook
```

---

## Pipeline CI/CD

```
PR abierto
  └─> Codacy (gate: sin Críticos/Altos)
  └─> Build .NET (dotnet build)
  └─> Tests unitarios (cobertura >= 80%)
  └─> Build Angular (ng build --configuration production)
  └─> Trivy scan de imagen Docker

Merge a main
  └─> Build imagen Docker multi-stage (API + Angular en nginx)
  └─> Push a GHCR: ghcr.io/forzatech/forza-vacaciones:v{semver}-{sha}
  └─> Deploy automático a dev/staging (Rancher Fleet)
  └─> Smoke tests (POST solicitud de prueba → verificar estado)

Deploy a producción
  └─> Gate manual (GitHub Environments: producción)
  └─> Rancher Fleet sincroniza desde Git
  └─> Notificación al equipo vía NATS
```

---

## ADRs (Architecture Decision Records)

| Decisión | Fecha | Estado | Documento |
|----------|-------|--------|-----------|
| Temporal.io para el flujo de aprobación multi-nivel | 2026-04-27 | Aceptado | `docs/adr/001-temporal-aprobaciones.md` |
| Outbox Pattern para garantía de entrega de eventos | 2026-04-27 | Aceptado | `docs/adr/002-outbox-pattern.md` |
| Aprobación nivel 3 solo para solicitudes > 5 días | 2026-04-27 | Aceptado | `docs/adr/003-regla-nivel3.md` |

---

*Generado con GitHub Copilot usando `.github/copilot-instructions.md` del repositorio*
*Stack: ForzaTech v3 — Referencia: Política_Forza_Tech_v3.docx*
