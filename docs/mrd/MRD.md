# MRD — Market Requirements Document

**Caso**: FTGO (Food To Go)
**Versión**: 1.0
**Autor**: David (Equipo de Arquitectura, en colaboración con Producto)
**Fecha**: 21 mayo 2026
**Estado**: Aprobado para alimentar el PRD

> Documento centrado en el **mercado y los usuarios**: qué necesidades existen, qué expectativas tiene cada actor del marketplace y qué requisitos de mercado deben traducirse luego en producto (PRD) y función (FSD). Hereda los objetivos del [BRD](../brd/BRD.md) y se restringe al [Brief FTGO](../brief/brief.md).

---

## 1. Posicionamiento

FTGO es un **marketplace de tres lados** (consumidor ↔ restaurante ↔ courier) que intermedia la operación de delivery de comida. Compite por **conveniencia, transparencia y rapidez de entrega**. La arquitectura actual (monolito) no permite acompañar el crecimiento, en particular en horarios pico (almuerzo/cena), lo que pone en riesgo la propuesta de valor.

## 2. Segmentos de usuarios y necesidades

Las necesidades se derivan estrictamente de los stakeholders declarados en el [Brief §A.2](../brief/brief.md) y se ordenan por prioridad para producto.

### 2.1 Consumidor (lado demanda)

- **Necesidad N-01**: Hacer un pedido en pocos pasos sin esperas perceptibles en la app.
- **Necesidad N-02**: Ver en tiempo real el estado del pedido (aceptado, en preparación, en ruta, entregado).
- **Necesidad N-03**: Confiar en que su método de pago se procesa con seguridad (PCI-DSS delegado a Stripe, [Brief §A.4]).
- **Necesidad N-04**: Poder seguir pidiendo aunque alguna integración externa esté degradada (la app sigue tomando pedidos aunque Stripe sufra latencia, [Brief §A.4 Tolerancia a fallos]).

### 2.2 Restaurante (oferta de comida)

- **Necesidad N-05**: Recibir tickets de pedido en su dashboard sin saturar la cocina.
- **Necesidad N-06**: Poder aceptar/rechazar tickets con un tiempo de preparación realista (US-02).
- **Necesidad N-07**: Controlar disponibilidad del menú y horarios.

### 2.3 Courier (oferta de entrega)

- **Necesidad N-08**: Recibir asignaciones cercanas a su ubicación actual con bajo tiempo de propuesta (timeout ≈ 30 s, US-03).
- **Necesidad N-09**: Ver una ruta optimizada al restaurante y al consumidor (depende de Google Maps, [Brief §A.2]).
- **Necesidad N-10**: Pagos confiables y trazables por entrega cerrada (US-03 + cap 4 Richardson sobre pagos a partners).

### 2.4 Empleado FTGO — back office

- **Necesidad N-11**: Visibilidad end-to-end de pedidos para soporte e incidencias ([Brief §A.2], NFR-07 trazabilidad).
- **Necesidad N-12**: Reportes financieros (cobros, comisiones, payouts) — capacidad **Billing & Accounting** del [Brief §A.3].

### 2.5 Sistemas externos como “usuarios” indirectos

- Stripe, Google Maps, SendGrid/Twilio requieren contratos de integración estables (timeouts, retries, idempotencia) — alimenta NFR-04 (tolerancia a fallos).

## 3. Drivers de mercado y oportunidad

| ID | Driver de mercado | Origen | Implicación en producto |
|---|---|---|---|
| MD-01 | Tráfico **5x** en horarios de almuerzo (12–14) y cena (19–22) | [Brief §A.4 Carga] | Escalado horizontal por capability (X-axis Scale Cube, cap 1 Richardson) |
| MD-02 | Expectativa móvil: UX percibida < 200 ms p95 | [Brief §A.4 Latencia UX] | Lecturas prioritariamente síncronas con cache; gateway móvil dedicado |
| MD-03 | Caídas de pasarela externa no deben cancelar pedidos | [Brief §A.4 Tolerancia a fallos] | Persistir intención de pago + cola de retry (cap 4–5 Richardson) |
| MD-04 | Crecimiento incremental sin big-bang | [Brief §A.4 Migración incremental] | Strangler Fig por capability; UI gateway dirige tráfico viejo↔nuevo |
| MD-05 | Regulación: PCI-DSS + GDPR | [Brief §A.4 Cumplimiento] | Aislar datos sensibles; pagos solo en Stripe; consent + retention en Consumer Mgmt |

## 4. Casos de uso de mercado (alto nivel)

Estos son los flujos críticos por los que el mercado valora a FTGO. Cada uno aterriza en el FSD como **UC formal** con Given/When/Then.

| MUC | Flujo de mercado | UCs derivados en el FSD | Trazabilidad |
|---|---|---|---|
| MUC-1 | Un consumidor descubre, pide y recibe comida | UC-01, UC-04, UC-05, UC-07 | US-01 |
| MUC-2 | Un restaurante recibe el ticket y lo gestiona | UC-02, UC-06 | US-02 |
| MUC-3 | Un courier acepta una entrega y la completa | UC-03, UC-07 | US-03 |
| MUC-4 | Back office investiga una incidencia | UC-07 + NFR-07 trazabilidad | Brief §A.2 Empleado FTGO |

## 5. Requisitos de mercado priorizados (MoSCoW)

| Categoría | Must (24 m) | Should (24 m) | Could (24 m+) |
|---|---|---|---|
| Toma de pedidos | UX < 200 ms p95 (MD-02) | Pagos one-click | Promociones dinámicas |
| Restaurante | Dashboard de tickets en tiempo real | Sugerencias de tiempo de prep | Forecast de demanda |
| Courier | Asignación cercana < 30 s | Ruta optimizada | Bonificaciones por zona |
| Back office | Trazabilidad end-to-end | Reportes auto-servicio | BI avanzado |
| Externos | Resiliencia ante caídas (MD-03) | Multi-proveedor de mapas | Multi-pasarela |

## 6. Métricas de mercado (qué mediremos en producción)

| ID | Métrica de mercado | Origen / racional | Meta inicial |
|---|---|---|---|
| MM-01 | Tasa de abandono en checkout | MD-02 latencia UX | < 5 % |
| MM-02 | % de pedidos completados sin intervención del back office | MD-03 tolerancia a fallos | > 99 % |
| MM-03 | Tiempo medio de asignación de courier | MD-01 (carga pico) + US-03 | < 30 s |
| MM-04 | Tiempo de respuesta del dashboard del restaurante | N-05 | < 1 s p95 |
| MM-05 | % de pedidos que sobreviven una caída de Stripe simulada | MD-03 | 100 % (con cola de retry) |

## 7. Exclusiones (no son requisitos de mercado en esta fase)

- Verticales fuera de food delivery (groceries, farmacia, retail).
- Voice/chatbot ordering como canal principal.
- Loyalty / suscripciones (rotaciones aparte del scope de migración).
- Marketplace B2B (sólo C2C consumidor ↔ restaurante).

## 8. Trazabilidad downstream

| Elemento MRD | Aterriza en |
|---|---|
| MD-01 (carga 5x) | PRD §NFR-03 escalabilidad horizontal |
| MD-02 (latencia UX < 200 ms) | PRD §NFR-01, ADR 0002 IPC (síncrono eficiente para reads) |
| MD-03 (tolerancia Stripe) | PRD §NFR-04, ADR 0002 (async + retry), FSD UC-04 |
| MD-04 (Strangler Fig) | ADR 0001 + diagramas C4 (sistema legacy aparece como `System_Ext`) |
| MD-05 (compliance) | PRD §NFR-05, FSD UC-04 (pago) |
| N-02 (tracking real-time) | FSD UC-05 + PRD §NFR-01 (degradación a 99.5 %, [Brief §A.4 Disponibilidad]) |
| N-08 / N-09 (courier) | FSD UC-03 + ADR 0002 (push / WebSocket o polling) |

## 9. Referencias

- [BRD](../brd/BRD.md) (objetivos de negocio y restricciones)
- [Brief FTGO](../brief/brief.md) (única fuente del dominio)
- Richardson, C. (2019). *Microservices Patterns*. Manning. Capítulos 1–2.
