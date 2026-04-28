# ADR-002 — Tokenización en SDK de Pasarela para Reducir Scope PCI DSS

**Estado:** Aceptado
**Fecha:** 2026-04-28
**Área:** forza-pay / fpg-sdk-* / Seguridad

---

## Contexto

El procesamiento de pagos con tarjeta de crédito/débito implica manejar datos del titular (PAN, CVV, fecha de expiración). Si el backend de Forza capturara estos datos directamente, el sistema quedaría bajo el alcance de **PCI DSS SAQ D** (el más estricto), con requisitos de auditoría extensos y costosos.

## Decisión

La captura de datos de tarjeta ocurre **exclusivamente dentro del SDK o iFrame provisto por la pasarela** (BAC JS SDK / Neonet Native SDK). El SDK de Forza (`fpg-sdk-angular`, `fpg-sdk-react`, `fpg-sdk-flutter`) integra estos SDKs y recibe como resultado un **payment_token** opaco. Dicho token es el único dato que viaja hacia el backend `fpg-api`.

- El PAN, CVV y datos de tarjeta **nunca transitan ni se almacenan en infraestructura Forza**.
- Los tokens de tarjeta para suscripciones se almacenan en **HashiCorp Vault**, nunca en PostgreSQL.
- Scope PCI resultante: **SAQ A** (sin acceso a datos de tarjeta).

## Consecuencias

**Positivo:**
- Reducción drástica de auditoría PCI DSS.
- Responsabilidad de custodia del PAN recae sobre la pasarela certificada.
- Simplifica los requisitos de seguridad de la base de datos y logs.

**Negativo:**
- Dependencia del SDK de cada pasarela en el frontend; actualizaciones del SDK deben gestionarse en los tres proyectos (`fpg-sdk-angular`, `fpg-sdk-react`, `fpg-sdk-flutter`).
- Funcionalidades de UI del checkout (estilos del formulario de tarjeta) están parcialmente controladas por la pasarela.

## Alternativas Descartadas

- **Captura directa de tarjeta en frontend Forza → backend Forza:** Requiere PCI DSS SAQ D. Inviable operativamente.
- **Hosted payment page (redirección total):** Aceptable para PCI, pero rompe la experiencia de marca (white-label) dentro de la app.
