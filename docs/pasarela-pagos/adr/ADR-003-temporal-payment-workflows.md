# ADR-003 — Temporal.io para Orquestación de Flujos de Pago

**Estado:** Aceptado
**Fecha:** 2026-04-28
**Área:** forza-pay / fpg-worker

---

## Contexto

Los flujos de pago involucran múltiples pasos que pueden fallar de forma independiente: cargo a pasarela, actualización de base de datos, publicación de eventos NATS, reintento ante rechazo bancario, cobros recurrentes en fechas futuras. Un fallo en cualquier paso puede dejar el sistema en estado inconsistente si no hay un mecanismo de orquestación durable.

## Decisión

Usar **Temporal.io** como motor de orquestación de workflows para los siguientes procesos:

| Workflow | Descripción |
|---|---|
| `PaymentWorkflow` | Carga la tarjeta, actualiza estado en DB, publica evento NATS. Reintento automático ante timeout de pasarela. |
| `SubscriptionWorkflow` | Durable timer hasta `next_charge_at`. Al despertar, ejecuta `PaymentWorkflow`. Gestiona reintentos con backoff ante fallo. |
| `RefundWorkflow` | Solicita reembolso a pasarela, actualiza payment y refund en DB, publica evento. |
| `CashConfirmationWorkflow` | Espera señal de webhook de pago en efectivo confirmado. Expira el pago si no llega en el plazo configurado. |

## Consecuencias

**Positivo:**
- Historial de ejecución persistido; recuperación automática ante crash del worker.
- Reintento configurable por actividad sin lógica extra.
- `SubscriptionWorkflow` elimina la necesidad de un cron job frágil.
- Versionado de workflows facilita deploys sin pérdida de ejecuciones en curso.

**Negativo:**
- Requiere infraestructura adicional (Temporal Server en cluster).
- Curva de aprendizaje del modelo de programación determinista.
- El `fpg-worker` debe desplegarse como servicio separado de `fpg-api`.

## Alternativas Descartadas

- **Polly solo:** Resuelve reintentos simples pero no workflows multi-paso ni timers durables de suscripción.
- **Hangfire / Quartz.NET:** Adecuados para jobs periódicos simples, sin soporte de workflows con estado y señales externas (webhooks de confirmación de efectivo).
