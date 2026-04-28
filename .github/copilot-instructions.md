# ForzaTech Copilot Skill — Stack & Estándares v3

> Este archivo es el "skill" de GitHub Copilot para todos los repositorios de ForzaTech.
> Copilot lo lee automáticamente en cada sesión de chat del workspace.
> Mantenerlo actualizado es responsabilidad del Tech Lead del proyecto.

---

## Stack Tecnológico Obligatorio

### Backend
- **Framework:** .NET Core (LTS vigente), C# con estilo Microsoft
- **Arquitectura:** Clean Architecture (Domain / Application / Infrastructure / Presentation)
- **Patrones:** CQRS con MediatR, Repository Pattern + Unit of Work, SOLID
- **ORM:** Entity Framework Core con migrations versionadas en Git
- **Mensajería ligera:** NATS JetStream (pub/sub en tiempo real, colas lightweight)
- **Workflows duraderos:** Temporal.io (procesos multi-paso, sagas, reconciliaciones)
- **Resiliencia:** Polly (retry con backoff exponencial, circuit breaker, timeout)
- **Logging:** Serilog con salida estructurada en JSON
- **Tests:** xUnit + Moq (unitarios), cobertura mínima 80%

### Frontend Web
- **Angular:** aplicaciones empresariales — NgRx/Signals, Smart/Dumb components, Lazy Loading
- **React:** componentes funcionales con Hooks, React Query, Zustand, estructura `features/`
- **No usar:** jQuery, class components, variables globales sin gestión de estado

### Aplicaciones Móviles
- **Framework:** Flutter (canal stable, última LTS), Dart
- **Arquitectura:** Clean Architecture con BLoC (flutter_bloc) o Riverpod
- **HTTP:** Dio con interceptores para tokens y logging

### Bases de Datos
- **Transaccional:** PostgreSQL — naming `snake_case`, migrations con EF Core o Flyway
- **Analítica:** ClickHouse — solo lectura desde aplicaciones, alimentado vía Kafka/Debezium
- **No crear** sistemas propios de autenticación; siempre usar Keycloak

### Infraestructura
- **Orquestación:** Kubernetes 1.30+ gestionado con Rancher
- **API Gateway:** Apache APISIX — único punto de entrada, configuración declarativa en YAML
- **Autenticación:** Keycloak — OIDC/OAuth 2.0, Authorization Code Flow + PKCE
- **CI/CD:** GitHub Actions + GitOps (Rancher Fleet sincroniza desde GitHub)
- **Calidad:** Codacy obligatorio en todos los PRs — sin issues Críticos ni Altos en merge
- **Contenedores:** Docker multi-stage, imágenes escaneadas con Trivy
- **Secretos:** HashiCorp Vault o Kubernetes Secrets — NUNCA en código fuente

---

## Convenciones de Nombramiento

```
// C# / .NET
PascalCase     → clases, interfaces, métodos, propiedades públicas
camelCase      → variables locales, parámetros
I-prefix       → interfaces (IUserRepository)
Async-suffix   → métodos async (GetUserAsync)
-Exception     → excepciones personalizadas (NotFoundException)

// Base de datos (PostgreSQL)
snake_case     → tablas, columnas, índices
plural         → nombres de tablas (users, transactions)
id             → PK estándar (user_id, transaction_id)

// Kafka topics
{entorno}.{dominio}.{entidad}   → prod.pagos.transacciones

// NATS subjects
{dominio}.{entidad}.{accion}    → pagos.transaccion.creada

// Git branches
feature/{ticket}-descripcion
fix/{ticket}-descripcion
release/{version}
```

---

## Reglas para Generación de Código con IA

1. **TODO código IA debe pasar revisión humana** antes del merge al repositorio.
2. **TODO código IA debe pasar Codacy** — sin issues Críticos ni Altos.
3. **NUNCA incluir** en los prompts: credenciales, tokens, datos de clientes, configuraciones de producción.
4. **El desarrollador es responsable** del código que hace merge, sea de IA o manual.
5. Usar **Claude** para: diseño de arquitectura, documentación técnica, ADRs, diagramas Mermaid, revisión de contratos de API.
6. Usar **Copilot** para: autocompletado en IDE, generación de boilerplate, tests unitarios, documentación de funciones.
7. Usar **Cursor** para: análisis de código legacy, refactoring multi-archivo, navegación de código existente.

---

## Seguridad — Reglas No Negociables

- OWASP Top 10 como referencia mínima en todo desarrollo
- Validación de inputs en TODOS los endpoints
- Autenticación y autorización SIEMPRE vía Keycloak + APISIX
- Secretos gestionados con Vault o K8s Secrets — NUNCA en código ni repositorios
- TLS 1.2+ obligatorio en comunicaciones externas
- mTLS entre servicios internos del cluster

---

## Patrones de Arquitectura Obligatorios

```
Clean Architecture (proyectos .NET):
  Domain/          → entidades, value objects, interfaces de repositorio
  Application/     → casos de uso, commands/queries (CQRS), DTOs
  Infrastructure/  → repositorios, BD, clientes HTTP, NATS, Temporal
  Presentation/    → controllers, middlewares, configuración de API

Event-Driven:
  NATS JetStream   → eventos ligeros entre microservicios
  Temporal.io      → flujos de negocio multi-paso con garantías
  Outbox Pattern   → garantía de entrega desde BD al bus de mensajería
  Debezium + Kafka → CDC y replicación hacia ClickHouse

Frontend Angular:
  Facade Pattern   → encapsular lógica de servicios
  Smart/Dumb       → containers manejan estado, presentacionales solo inputs
  Interceptors     → tokens Keycloak y correlación de trazas

Frontend React:
  Custom Hooks     → lógica reutilizable
  features/        → cada feature autocontenida
  React Query      → fetching, caching y sincronización
```

---

## CI/CD — Pipeline Estándar

Todo PR debe pasar en orden:
1. Análisis Codacy (bloqueante: sin Críticos ni Altos)
2. Build de la aplicación
3. Tests unitarios + cobertura ≥ 80%
4. Escaneo de imagen Docker con Trivy
5. Build imagen Docker multi-stage
6. Push a GHCR con tag semántico `v1.2.3-sha`
7. Deploy automático a dev/staging
8. Smoke tests post-deploy
9. Gate manual de aprobación para producción

---

## Documentación Estándar

Todo proyecto debe tener en la raíz del repositorio:
- `README.md` — propósito, setup local, comandos de desarrollo
- `ARCHITECTURE.md` — diagrama de arquitectura en Mermaid, stack, dependencias, naming
- `.github/copilot-instructions.md` — este archivo (skill del repo)
- `docs/adr/` — Architecture Decision Records para decisiones importantes

Usar **Mermaid** para todos los diagramas en `.md`:
- `flowchart LR` para arquitecturas de servicios
- `sequenceDiagram` para flujos de autenticación y APIs
- `erDiagram` para modelos de datos

---

*Versión: 3.0 — 2025 | Mantenido por: David Salas — Gerente de Desarrollo y Tecnología*
*Referencia completa: Política_Forza_Tech_v3.docx*
