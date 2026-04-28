# ADR-004 — SDK Multi-Plataforma como Librería Publicada (npm / pub.dev)

**Estado:** Aceptado
**Fecha:** 2026-04-28
**Área:** forza-pay / fpg-sdk-*

---

## Contexto

Forza tiene tres plataformas frontend activas: **Angular** (empresarial), **React** (microapps y portales) y **Flutter** (mobile). Cada una debe poder integrar el checkout de pagos con branding white-label del tenant correspondiente. Sin un SDK compartido, cada plataforma duplicaría la lógica de comunicación con la API, el sistema de theming y el manejo de errores.

## Decisión

Desarrollar y publicar **tres librerías de SDK** independientes con el mismo contrato de API:

| Librería | Tecnología | Distribución |
|---|---|---|
| `@forza/fpg-sdk-angular` | Angular Library (ng-packagr) | npm privado (GitHub Packages) |
| `@forza/fpg-sdk-react` | React + TypeScript (Rollup) | npm privado (GitHub Packages) |
| `fpg_sdk_flutter` | Flutter Package | pub.dev privado / Git dependency |

Cada SDK:
1. Expone un componente de checkout (`FpgCheckout`) con `tenantSlug`, `amount`, `currency`, `externalReference` como props/inputs obligatorios.
2. Consulta `GET /tenants/{slug}/config` al inicializarse y aplica el tema automáticamente.
3. Integra el SDK JS/native de la pasarela activa del tenant para la captura tokenizada.
4. Emite eventos tipados (`paymentSuccess`, `paymentError`, `paymentCancelled`).

## Consecuencias

**Positivo:**
- Una sola actualización de lógica de API se propaga a todas las plataformas actualizando la versión del paquete.
- El sistema de white-label es centralizado; cambiar el logo de Forza Delivery requiere solo actualizar la config del tenant en la API.
- Contrato TypeScript/Dart compartido reduce errores de integración.

**Negativo:**
- Requiere proceso de publicación y versionado semántico (`MAJOR.MINOR.PATCH`) en el pipeline CI/CD de cada SDK.
- Los equipos consumidores deben gestionar actualizaciones de versión del paquete.

## Alternativas Descartadas

- **Componente web (Web Components):** Compatible con Angular y React pero no con Flutter; además limita las capacidades de theming nativo por plataforma.
- **Copy/paste de código por equipo:** Genera inconsistencias y duplicación de bugs.
