# ADR-001 — Strategy Pattern para Adaptadores de Pasarela de Pago

**Estado:** Aceptado
**Fecha:** 2026-04-28
**Área:** forza-pay / ForzaPay.Infrastructure

---

## Contexto

El sistema debe integrarse con **dos pasarelas de pago** (BAC Credomatic y Neonet). Cada tenant del sistema puede tener preferencia o contratos con una pasarela diferente. Ambas pasarelas tienen APIs, esquemas de autenticación y estructuras de respuesta distintas. En el futuro podría sumarse una tercera pasarela.

## Decisión

Implementar el **Strategy Pattern** a través de la interfaz `IPaymentGateway` con un `GatewayResolver` que selecciona el adaptador correcto en tiempo de ejecución, según la configuración del tenant y disponibilidad de la pasarela primaria.

```csharp
public interface IPaymentGateway
{
    GatewayType Type { get; }
    Task<ChargeResult> ChargeAsync(ChargeRequest request, CancellationToken ct);
    Task<RefundResult> RefundAsync(RefundRequest request, CancellationToken ct);
    Task<CashReferenceResult> GenerateCashReferenceAsync(CashReferenceRequest request, CancellationToken ct);
}
```

## Consecuencias

**Positivo:**
- Agregar una nueva pasarela requiere solo implementar `IPaymentGateway` y registrarla en DI, sin modificar lógica de negocio.
- Los workflows de Temporal llaman a `IPaymentGateway` sin conocer la pasarela concreta.
- Facilita pruebas unitarias con mocks de la interfaz.

**Negativo:**
- Algunas funcionalidades muy específicas de una pasarela (ej. cobro en cuotas de BAC) necesitan extensión de la interfaz o métodos opcionales.

## Alternativas Descartadas

- **Condicional directo (`if gateway == BAC`):** Viola Open/Closed Principle. Imposible escalar a más pasarelas.
- **Un servicio por pasarela sin abstracción:** Duplica toda la lógica de orquestación de Temporal.
